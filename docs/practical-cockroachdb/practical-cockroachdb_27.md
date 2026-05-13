# 第 8 章 测试

#### 使用应用代码进行测试

到目前为止，我们一直将 API 本身作为一个黑盒进行测试。如果我们想从我们应用程序自身的测试套件中，仅将数据库作为黑盒来测试，该怎么办呢？让我们看看如何通过 Docker Compose 将 CockroachDB 包含到你的本地测试环境中。

对于这个示例，我们需要几个文件：

- `docker-compose.yml` – 包含将作为我们组合的应用程序和数据库测试套件一部分运行的 Docker 服务。具体来说，是一个用于我们应用代码的服务和一个用于 CockroachDB 的服务。
- `Dockerfile` – 一个将我们的应用程序测试二进制文件打包成 Docker 镜像的文件。
- `main_test.go` – 我们的应用程序测试代码。在本例中，我使用的是 Go 语言。

让我们开始吧！我们将首先通过几个命令准备一个目录来存放我们的测试：

```
mkdir db_tests
cd db_tests
go mod init dbtests
```

接下来，我们将引入实现数据库交互所需的数据库依赖：

```
go get -u github.com/jackc/pgx
```

接着，我们将创建 Go 代码，该代码将连接到数据库并运行一些单元测试。由于此文件独立于任何其他 Go 代码存在，因此没有其他自管理 Go 模块的依赖。在这段代码中，我们将：

- 从命令行读取一个标志，指示是否应包含数据库测试（默认为 false）
- 如果标志指示要运行数据库测试：
  - 从环境变量中读取连接字符串。
  - 在数据库启动时，尝试通过一个简单的退避循环连接到数据库。
  - 所有测试完成后关闭数据库连接。
  - 运行单元测试并捕获一个状态码，指示测试是通过还是失败。
  - 使用从测试运行接收到的状态码退出应用程序。
- 请注意，`TestADatabaseInteraction`函数仅在提供了数据库标志时才会执行。

```go
package main

import (
    "context"
    "flag"
    "log"
    "os"
    "testing"
    "time"

    "github.com/jackc/pgx/v4/pgxpool"
)

var (
    dbTests *bool
    db      *pgxpool.Pool
)

func TestMain(m *testing.M) {
    dbTests = flag.Bool("db", false, "run database tests")
    flag.Parse()

    if *dbTests {
        connStr, ok := os.LookupEnv("CONN_STR")
        if !ok {
            log.Fatal("connection string env var not found")
        }

        if err := waitForDB(connStr); err != nil {
            log.Fatalf("error waiting for database: %v", err)
        }
    }

    code := m.Run()

    if *dbTests {
        db.Close()
    }

    os.Exit(code)
}

func waitForDB(connStr string) error {
    var err error
    for i := 1; i <= 10; i++ {
        db, err = pgxpool.Connect(context.Background(), connStr)
        if err != nil {
            log.Printf("error connecting to database: %v", err)
        }
    }
}
```

在响应体中（我为了可读性稍微调整了一下格式），我们可以看到我们的两个产品已被返回，并且过期的促销价和更贵的现价都没有出现。

接下来，让我们下一个订单，并通过下一个多语句事务调用来检查数据库行为。在这个场景中，我们将创建一个订单，然后使用其 ID 向`product_order`表中插入行：

```
curl -X POST 'localhost:3000/orders' \
-d '{
  "customer_id": "85f05ac3-931f-4417-a6ae-1403468a2c10",
  "delivery_instructions": "Please leave package behind the green bin.",
  "products": {
    "386e841e-4da8-4688-82a4-71d6fbb369fa": "cc71e211-abc6-42d5-8f59-d357042334b0",
    "d77bfea2-f0df-4da9-aeef-7c8d9c362e38": "cbdc9d9c-a987-4655-b3f8-6b9b2ec9c50e"
  }
}'
{"id":"3ad9469a-50ec-4773-bc50-1c92209078de","reference":"NIAXGBQP"}
```

让我们通过向返回客户订单的端点发起请求，来检查订单是否成功插入：

```
curl -X GET localhost:3000/customers/85f05ac3-931f-4417-a6ae-1403468a2c10/orders
[{"id":"93bff9af-5794-47a5-94e5-18e8a9832c41","reference":"GDCQONIR","customer_id":"85f05ac3-931f-4417-a6ae-1403468a2c10","date":"2022-01-20T18:32:45Z","delivery_instructions":"Please leave package behind the green bin.","products":{"Has Bean":10.0,"Where Have You Bean All My Life?":12.0}}]
```

在响应体中，我们可以看到包含了所有订单字段，以及正确的产品和价格。



```
time.Sleep(time.Second * time.Duration(i))

}

}

return err

}

func TestADatabaseInteraction(t *testing.T) {

if !*dbTests {

t.SkipNow()

}

t.Log("running database test")

}

func TestANonDatabaseInteraction(t *testing.T) {

t.Log("running non-database test")

}
```

请注意，此时我们可以使用以下命令运行所有**不需要**数据库连接的测试：

`go test ./... -v`

```
=== RUN TestADatabaseInteraction
--- SKIP: TestADatabaseInteraction (0.00s)
Chapter 8 testing
=== RUN TestANonDatabaseInteraction
main_test.go:65: running non-database test
--- PASS: TestANonDatabaseInteraction (0.00s)
PASS
ok
```

接下来，我们将应用程序打包成一个 Docker 镜像，以便能在 Docker Compose 环境中与 CockroachDB 一同运行。这会非常简单，因为我们将执行一个与 Linux 兼容的、位于`/app`目录下的可执行文件。

```
FROM alpine:latest
COPY . /app
WORKDIR /app
```

代码（以及这个 Dockerfile）准备就绪后，我们就可以构建一个可用于 Docker Compose 的 Docker 镜像。第一条命令将为名为 “crdb-test” 的 Linux 机器构建一个测试可执行文件。第二条命令将把我们当前的工作目录（包括测试可执行文件）打包到`/app`目录中：

```
GOOS=linux go test ./... -c -o crdb-test
docker build -t crdb-test .
```

我们要创建的最后一个文件是用于 Docker Compose 的。这个文件将创建两个服务：一个用于 CockroachDB，另一个用于我们的测试套件。在此文件中，我们

• 创建一个名为 `cockroach_test` 的服务，使用 v21.2.4 版本的
CockroachDB Docker 镜像，并使其
• 定制其启动命令以仅运行单个节点，这样它
就不会等待初始化命令
• 不暴露 CockroachDB 的默认端口 26257，因为无需
从主机访问 CockroachDB
• 禁用日志记录，这意味着我们测试套件的日志是
我们唯一能看到的输出

Chapter 8 testing

• 创建一个名为 `app_test` 的服务，使用我们上一步构建的
Docker 镜像：
• 向容器传递一个名为 `CONN_STR` 的环境变量，
其中包含连接数据库所需的信息。
注意，我们使用了 `cockroach_test` 服务的名称而非
“localhost”，这是由于 Docker 网络的工作方式所致。
• 传递 `-db` 命令行参数以确保数据库
测试得以运行。
• 传递 `-test.v` 命令行参数以确保我们
获得详细的日志记录。设置此项后，我们将看到运行的每个函数。
• 将容器的工作目录设置为测试可执行文件
所在的目录。
• 将容器标记为依赖于 `cockroach_`
test 服务。

我们将在名为 `app-network` 的 Docker 桥接网络中运行这两个容器，以便它们可以相互通信。

```
version: '3'
services:
  cockroach_test:
    image: cockroachdb/cockroach:v21.2.4
    command: start-single-node --insecure
    logging:
      driver: none
    networks:
      - app-network
  app_test:
    image: crdb-test:latest
    environment:
      CONN_STR: postgres://root@cockroach_test:26257/defaultdb?sslmode=disable
    command: ./crdb-test -db -test.v
    working_dir: /app
    depends_on:
      - cockroach_test
    networks:
      - app-network
networks:
  app-network:
    driver: bridge
```

一切准备就绪后，我们可以通过以下命令启动完整的数据库测试：

`docker-compose up --abort-on-container-exit --force-recreate`

```
Recreating app_cockroach_test_1 ... done
Recreating app_app_test_1 ... done
Attaching to app_app_test_1
app_test_1 | === RUN TestADatabaseInteraction
app_test_1 | main_test.go:61: running database test
app_test_1 | --- PASS: TestADatabaseInteraction (0.00s)
app_test_1 | === RUN TestANonDatabaseInteraction
app_test_1 | main_test.go:65: running non-database test
app_test_1 | --- PASS: TestANonDatabaseInteraction (0.00s)
app_test_1 | PASS
app_app_test_1 exited with code 0
Aborting on container exit...
Stopping app_cockroach_test_1 ... done
```

正如我们所见，数据库和非数据库测试都按预期运行。`app_test` 服务容器完成后，`app_test` 和 `cockroach_test` 容器都会退出。

`白盒测试`



##### 白盒测试与数据库验证

白盒测试涉及测试所有通常对用户不可见的内容，包括引用完整性检查、索引查询计划分析以及默认值赋值检查等。现在，让我们来看看其中的一些内容。

##### 引用完整性检查

引用完整性检查确保表间的依赖关系能够防止意外的破坏性操作。让我们执行一些破坏性操作，以验证数据库的引用完整性。

首先，我们将尝试删除 `retail.order`、`retail.product` 和 `retail.product_price` 表中的所有数据。如果不加注意，这可能会导致糟糕的一天：

```sql
DELETE FROM retail.order WHERE true;
```

```
ERROR: delete on table "order" violates foreign key constraint "fk_order_id_ref_order" on table "product_order"
SQLSTATE: 23503
DETAIL: Key (id)=('93bff9af-5794-47a5-94e5-18e8a9832c41') is still referenced from table "product_order".
CONSTRAINT: fk_order_id_ref_order
```

```sql
DELETE FROM retail.product_price WHERE true;
```

```
ERROR: delete on table "product_price" violates foreign key constraint "fk_product_price_id_ref_product_price" on table "product_order"
SQLSTATE: 23503
DETAIL: Key (id)=('cbdc9d9c-a987-4655-b3f8-6b9b2ec9c50e') is still referenced from table "product_order".
CONSTRAINT: fk_product_price_id_ref_product_price
```

```sql
DELETE FROM retail.product WHERE true;
```

```
ERROR: delete on table "product" violates foreign key constraint "fk_product_id_ref_product" on table "product_price"
SQLSTATE: 23503
DETAIL: Key (id)=('386e841e-4da8-4688-82a4-71d6fbb369fa') is still referenced from table "product_price".
CONSTRAINT: fk_product_id_ref_product
```

如果没有这些引用检查，我们将在引用表中创建孤立数据。需要指出的是，如果没有适当的权限，我们仍可能无意中从 `retail.customer` 和 `retail.product_order` 表中删除数据。不过，我们可以放心，其他表的检查机制是到位的。

##### 索引与查询计划

接下来是索引。让我们检查一下我们正在查询的表中的索引是否足够，以避免进行全表扫描。我们将遍历 API 代码中的每个 `SELECT` 查询，看看它们的优化程度，并确定我们可以采取哪些措施来改善其性能。

我们要看的第一个查询是通过 ID 获取客户信息：

```sql
EXPLAIN SELECT full_name, email, join_date FROM retail.customer
WHERE id = '85f05ac3-931f-4417-a6ae-1403468a2c10'
LIMIT 1;
```

```
info
distribution: local
vectorized: true
• scan
estimated row count: 1 (33% of the table; stats collected 28 minutes ago)
table: customer@primary
spans: [/'85f05ac3-931f-4417-a6ae-1403468a2c10' - /'85f05ac3-931f-4417-a6ae-1403468a2c10']
```

在表中的三个项目中，我们只扫描了感兴趣 ID 的那一行（占表的 33%）。`ID` 是表的主键，因此它已被索引，效率很高。请注意，虽然我不需要包含 `LIMIT 1` 表达式（因为表中只会有一个此 ID 的实例），但如果你希望进一步将查询限制为预期的返回行数，我认为这是一个安全且明智的习惯。

```sql
EXPLAIN WITH min_product_price AS (
    SELECT DISTINCT ON(product_id) product_id, id, min(amount) AS amount
    FROM retail.product_price
    WHERE start_date <= now()
      AND (end_date >= now() OR end_date IS NULL)
    GROUP BY id
    ORDER BY product_id, amount
)
SELECT p.id, p.name, p.sku, pp.id price_id, pp.amount FROM min_product_price pp
JOIN retail.product p ON pp.product_id = p.id;
```

```
info
distribution: full
vectorized: true
• lookup join
│ estimated row count: 2
│ table: product@primary
│ equality: (product_id) = (id)
│ equality cols are key
│
└── • render
   │ estimated row count: 3
   │
   └── • distinct
      │ estimated row count: 3
      │ distinct on: any_not_null
      │
      └── • sort
         │ estimated row count: 3
         │ order: +min
         │
         └── • group
            │ estimated row count: 3
            │ group by: id
            │ ordered: +id
            │
            └── • filter
               │ estimated row count: 3
               │ filter: (start_date <= '2022-01-24 19:14:54.499657+00:00') AND ((end_date >=
```



