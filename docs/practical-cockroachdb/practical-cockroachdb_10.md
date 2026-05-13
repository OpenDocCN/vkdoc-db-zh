# 第五章 与 CockroachDB 交互

#### 连接参数

```
# ...

# - 连接参数：

# (webui) http://127.0.0.1:8080/demologin?password=demo1892&username=demo

# (sql) postgres://demo:demo1892@127.0.0.1:26257?sslmode=require

#

# - 用户名: "demo", 密码: "demo1892"

# - 包含证书文件的目录（用于某些 SQL 驱动/工具）：

REDACTED

# ...
```

`demo@127.0.0.1:26257/defaultdb>`

现在是打开 DBeaver 并连接到数据库的时候了。请注意前面输出中的用户名和密码。图 5-1 展示了使用这些值连接到此集群的配置。

![](img/index-97_1.jpg)

**图 5-1.** DBeaver 中的连接设置

请注意，端口 `26257` 是 CockroachDB 用于数据库连接的默认端口。由于我没有更改默认端口，这里我使用 `26257` 作为连接端口号。

现在，让我们创建一个表，向其中插入一些数据，然后选择数据出来看看在 DBeaver 中的样子。图 5-2 展示了使用 DBeaver 处理数据时的样子。

![](img/index-98_1.jpg)

**图 5-2.** DBeaver 中的数据

我们连接到的数据库是基础版本，并未使用客户端证书。现在我将连接到一个 Cockroach Cloud 数据库，向你展示如何在 DBeaver 中完成此操作。

与本地演示数据库不同，免费层级的 Cockroach Cloud 数据库需要一个证书和一个参数来告诉 Cockroach Cloud 要连接到哪个集群（它是多租户的，意味着不同用户的集群是共同托管的）。

图 5-3 展示了 Cockroach Cloud 数据库所需的附加配置值。即，主机字段已更新为指向 Cockroach Cloud 实例，数据库字段现在包含了 Cockroach Cloud 定位你的集群所需的信息。

![](img/index-99_1.jpg)

**图 5-3.** 连接到 Cockroach Cloud 集群的基本设置

除了基本设置，你还需要帮助 DBeaver 找到将对你的连接进行身份验证的证书。图 5-4 展示了你需要设置的两个附加配置值。

![](img/index-100_1.jpg)

**图 5-4.** 连接到 Cockroach Cloud 集群的 SSL 设置

完成这些更改后，我们就可以连接到 Cockroach Cloud 数据库了。

### 通过编程方式连接

你可以从许多不同的编程语言连接到 CockroachDB。Cockroach Labs 在他们的网站<sup>1</sup>上列出了可用的驱动程序，并为每个驱动程序提供了示例应用程序。<sup>2</sup>

为了让你很好地了解如何以编程方式使用 CockroachDB，我现在将使用一些流行的驱动程序对数据库执行一些基本操作。

<sup>1</sup> [www.cockroachlabs.com/docs/stable/third-party-database-tools#drivers](http://www.cockroachlabs.com/docs/stable/third-party-database-tools#drivers)
<sup>2</sup> [www.cockroachlabs.com/docs/stable/example-apps](http://www.cockroachlabs.com/docs/stable/example-apps)

让我们创建一个要使用的表：

```sql
CREATE TABLE IF NOT EXISTS person (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL
);
```

接下来的示例展示了如何使用几种常见的编程语言连接到 CockroachDB 并获取结果。在每种情况下，为了简洁起见，我都选择了简洁性而非完整性。

#### Go 示例

在此示例中，我将连接到 `defaultdb` 数据库，并对 `person` 表执行 `INSERT` 和 `SELECT` 操作。我将使用一个流行的 Go Postgres 驱动程序，名为 `pgx`。

首先，初始化环境：

```bash
$ mkdir go_example
$ cd go_example
$ go mod init go_example
```

接下来，我们将在 `main.go` 文件中创建一个简单的应用程序。我将在此分享应用程序的关键构建块，因为完整的代码清单可在 GitHub 上找到。

首先，获取并导入 `pgx` 包：

```bash
$ go get github.com/jackc/pgx/v4
```

```go
import "github.com/jackc/pgx/v4/pgxpool"
```

接下来，连接到数据库并确保使用后关闭连接：

```go
db, err := pgxpool.Connect(
    context.Background(),
    "postgres://demo:demo22992@localhost:26257/defaultdb")
if err != nil {
```

