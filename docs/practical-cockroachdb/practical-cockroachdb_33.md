# 生产

在本书的最后一章，我们将介绍一些确保集群能够应对生产环境各种挑战的实践和工具：

• **最佳实践** – 学习新技术可能让人不知所措，而知道如何“正确”做事看起来也是一项艰巨的任务。本节将探讨在运行生产集群时需要考虑的一些最佳实践。
• **升级集群** – 我们需要保持节点更新，以利用最新的 CockroachDB 特性和修复。
• **备份和恢复数据** – 虽然 CockroachDB “多活”的理念消除了灾难恢复规划的必要性，但考虑采取数据库快照以便在发生意外数据库损坏（例如 `DELETE * FROM customer...`）时回退，仍然是良好的实践。
• **迁移集群** – 我曾在化妆品零售商 Lush 工作过，深知没有任何云服务商是永恒的。本节将向您展示如何在重大基础设施改革时，将 CockroachDB 集群从一个地方迁移到另一个地方。

1 [`cloud.google.com/customers/lush`](https://cloud.google.com/customers/lush)

© Rob Reid 2022
R. Reid, *Practical CockroachDB*, [`doi.org/10.1007/978-1-4842-8224-3_9`](https://doi.org/10.1007/978-1-4842-8224-3_9#DOI)

### 最佳实践

Cockroach Labs 网站上的最佳实践文档非常全面，是任何希望充分利用其 CockroachDB 集群的人的绝佳参考。

在本节中，我们将应用这些最佳实践，并观察如果不遵循它们会发生什么。

除非特别说明，本章后续的许多示例将使用我在第 8 章中创建的三节点、负载均衡的集群。

#### SELECT 性能

在本节中，我们将探讨一些关于 `SELECT` 语句的性能最佳实践，以及您可以调整哪些配置值以获得最佳性能。

如果您发现 `SELECT` 查询运行缓慢，并且已经排除了索引问题，那么您的数据库可能缺乏缓存内存。默认情况下，每个节点将拥有 128MiB（~134MB）的缓存大小，用于存储查询所需的信息。这个大小作为本地数据库开发的默认值效果良好，但您可能会发现增加它会带来更快的 `SELECT` 性能。Cockroach Labs 建议在生产部署中将缓存大小设置为机器可用内存的至少 25%。

要更新此设置，请在启动节点时向 `--cache` 参数传递一个值：

```
# 将缓存大小设置为节点总内存的 30%：
$ cockroach start --cache=.30 ...
$ cockroach start --cache=30% ...

# 将缓存大小设置为特定值：
$ cockroach start --cache=200MB ...
$ cockroach start --cache=200MiB ...
```

另一个需要考虑更改的设置是 `--max-sql-memory` 参数，它默认使用节点总内存的 25%。增加此值将允许 CockroachDB 为 `GROUP BY` 和 `ORDER BY` 等聚合操作分配更多内存，并增加可能的并发数据库连接数，从而缓解连接池争用。

2 [www.cockroachlabs.com/docs/stable/performance-best-practices-overview](http://www.cockroachlabs.com/docs/stable/performance-best-practices-overview)

要更新此设置，请在启动节点时向 `--max-sql-memory` 参数传递一个值：

```
# 将缓存大小设置为节点总内存的 30%：
```

#### INSERT 性能

在本节中，我们将探讨一些性能最佳实践及其重要性。我们将从执行一些 `INSERT` 语句开始，了解为何使用多行 DML 语句优于单行 DML 语句**至关重要**。

首先，创建一个数据库和一个表：

```sql
CREATE TABLE sensor_reading (
    sensor_id UUID PRIMARY KEY,
    reading DECIMAL NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

接下来，我们将编写一个脚本来模拟批量插入 1000 行数据，并比较通过单行和多行 DML 语句执行的性能差异。

我将使用 Go 进行这些测试，因此我的导入和主函数如下所示：

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/google/uuid"
    "github.com/jackc/pgx/v4/pgxpool"
)

func main() {
    connStr := "postgres://root@localhost:26257/best_practices?sslmode=disable"
    db, err := pgxpool.Connect(context.Background(), connStr)
    if err != nil {
        log.Fatalf("error connecting to db: %v", err)
    }
    defer db.Close()
    // ...
}
```

接下来，我们将实现单行 DML `INSERT` 代码。此代码将发出 1000 个独立请求（涉及 1000 次独立的网络往返），以将一行数据插入 `sensor_reading` 表，并记录整个操作所需的时间：

```go
func insertSingleRowDML(db *pgxpool.Pool, rows int) (time.Duration, error) {
    const stmt = `INSERT INTO sensor_reading (sensor_id, reading) VALUES ($1, $2)`
    start := time.Now()
    for i := 0; i < rows; i++ {
        if _, err := db.Exec(context.Background(), stmt, uuid.New(), 1); err != nil {
            return 0, fmt.Errorf("inserting row: %w", err)
        }
    }
    return time.Since(start), nil
}
```

接下来，我们将以批处理方式执行单行 `INSERT` 语句。这会将多个 `INSERT` 语句同时发送给 CockroachDB，并在单个事务中执行它们。在批量插入数据时，考虑将大型插入分解为较小的部分非常重要，因为插入的数据越多，锁定要插入的表的时间就越长。以下函数允许我以小的并发块执行批量插入：

```go
func insertSingleRowDMLBatched(db *pgxpool.Pool, rows, chunks int) (time.Duration, error) {
    const stmt = `INSERT INTO sensor_reading (sensor_id, reading) VALUES ($1, $2)`
    start := time.Now()
    eg, _ := errgroup.WithContext(context.Background())
    for c := 0; c < chunks; c++ {
        eg.Go(func() error {
            batch := &pgx.Batch{}
            for i := 0; i < rows/chunks; i++ {
                batch.Queue(stmt, uuid.New(), i)
            }
            res := db.SendBatch(context.Background(), batch)
            if _, err := res.Exec(); err != nil {
                return fmt.Errorf("running batch query insert: %w", err)
            }
            return nil
        })
    }
    if err := eg.Wait(); err != nil {
        return 0, fmt.Errorf("in batch worker: %w", err)
    }
    return time.Since(start), nil
}
```

请注意，此实现仍然是在数据库中作为单行 DML 执行的。然而，此实现与第一种实现的区别在于，现在的网络往返次数要少得多，从而加快了端到端查询的速度。

如果我们真想从查询中看到性能提升，就需要执行多行 DML 语句。以下代码正是这样做的，并借助了两个额外的函数。第一个辅助函数名为 `counter`，其工作是在每次调用时简单地返回一个递增的数字：

```go
func counter(start int) func() int {
    return func() int {
        // ...
    }
}
```


```go
func argPlaceholders(rows, columns, start int) string {
    builder := strings.Builder{}
    counter := counter(start)

    for r := 0; r < rows; r++ {
        builder.WriteString("(")

        for c := 0; c < columns; c++ {
            builder.WriteString("$")
            builder.WriteString(strconv.Itoa(counter()))

            if c < columns-1 {
                builder.WriteString(",")
            }
        }

        builder.WriteString(")")

        if r < rows-1 {
            builder.WriteString(",")
        }
    }

    return builder.String()
}
```

以下是一些示例，帮助你理解 `argPlaceholders` 的工作方式：

```go
fmt.Println(argPlaceholders(1, 1, 1)) // -> ($1)
fmt.Println(argPlaceholders(1, 2, 1)) // -> ($1,$2)
fmt.Println(argPlaceholders(2, 1, 1)) // -> ($1),($2)
fmt.Println(argPlaceholders(3, 2, 1)) // -> ($1,$2),($3,$4),($5,$6)
```

将所有部分组合起来，我们得到了多行 DML 插入函数：

```go
func insertMultiRowDML(db *pgxpool.Pool, rows, chunks int) (time.Duration, error) {
    const stmtFmt = `INSERT INTO sensor_reading (sensor_id, reading) VALUES %s`

    start := time.Now()
    eg, _ := errgroup.WithContext(context.Background())

    for c := 0; c < chunks; c++ {
        eg.Go(func() error {
            argPlaceholders := argPlaceholders(rows/chunks, 2, 1)
            stmt := fmt.Sprintf(stmtFmt, argPlaceholders)

            var args []interface{}
            for i := 0; i < rows/chunks; i++ {
                args = append(args, uuid.New(), i)
            }

            if _, err := db.Exec(context.Background(), stmt, args...); err != nil {
                return fmt.Errorf("inserting rows: %w", err)
            }
            return nil
        })
    }

    if err := eg.Wait(); err != nil {
        return 0, fmt.Errorf("in batch worker: %w", err)
    }

    return time.Since(start), nil
}
```

## 第九章 生产环境

那么，我们的查询速度有多快？我会将每个查询运行五次并取平均值：

| 测试名称 | 耗时 |
| :--- | :--- |
| `insertSinglerowdML (rows = 1000)` | 6.82s |
| `insertSinglerowdMLBatched (rows = 1000, chunks = 1)` | 4.89s |
| `insertSinglerowdMLBatched (rows = 1000, chunks = 2)` | 2.77s |
| `insertSinglerowdMLBatched (rows = 1000, chunks = 4)` | 1.87s |
| `insertMultirowdMLBatched (rows = 1000, chunks = 1)` | 49.35ms |
| `insertMultirowdML (rows = 1000, chunks = 2)` | 35.61ms |

可以清楚地看到，通过执行多行 DML，我们的执行时间只是单行 DML 等效操作的一小部分。对于批量操作，使用多行 DML 语句被认为是最佳实践，这是有充分理由的。

#### UPDATE 性能

如果你需要批量更新表中的数据，考虑以下几点非常重要：

-   **批处理** – 与 `INSERT` 语句类似，如果不小心执行，大型 `UPDATE` 语句会导致表锁定和性能低下。尽可能使用多行 DML 运行查询。
-   **过滤** – 与其尝试动态更新数据，不如考虑使用多遍技术，首先选择需要更新的行的主键列，然后在后续的 `UPDATE` 语句中使用它们。这样，CockroachDB 就不会被要求运行一个缓慢的、锁定的查询，而是先运行一个非锁定的读取查询，然后再运行一个快速的、有索引的锁定查询。

## 第九章 生产环境

让我们比较一下动态更新数据的 `UPDATE` 语句与使用多遍方法的 `UPDATE` 语句的性能。与 `INSERT` 示例一样，我将在运行它们之前提供两种场景的代码。

在我们开始之前，我们将再次向表中插入一些随机数据，这次借助 CockroachDB 的 `generate_series` 函数，跳过 Go 代码，直接进入 CockroachDB。该语句将为过去大约 1000 天向表中插入 1000 条随机条目，并将作为我们 `UPDATE` 语句的基础：

```sql
INSERT INTO sensor_reading (sensor_id, reading, timestamp)
SELECT
    gen_random_uuid(),
    CAST(random() AS DECIMAL),
    '2022-02-18' - CAST(s * 10000 AS INTERVAL)
FROM generate_series(1, 100000) AS s;
```

让我们假设，我们 1990 年代的传感器读数高了 0.001（没人说这不会变得极其牵强），需要修复。我们将使用 `UPDATE` 语句来修复数据。

首先，我们创建动态更新的解决方案：



func updateOnTheFly(db *pgxpool.Pool) (time.Duration, error) {
	const stmt = `UPDATE sensor_reading
	SET reading = reading - 0.001
	WHERE date_trunc('decade', timestamp) = '1990-01-01'`
	start := time.Now()
	if _, err := db.Exec(context.Background(), stmt); err != nil {
		return 0, fmt.Errorf("updating rows: %w", err)
	}
	return time.Since(start), nil
}

