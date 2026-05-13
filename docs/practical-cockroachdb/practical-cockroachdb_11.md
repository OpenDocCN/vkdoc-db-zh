# 第五章 与 CockroachDB 交互

`log.Fatalf("error opening database connection: %v", err)`

}

defer db.Close()

为简洁起见，我传入了一个 `context.Context` 类型的值 `context.Background()`，对于像这样的简单示例来说没问题。然而，在与 actor 交互的场景中，您需要传递请求的上下文。

`pgx` 包在对 `*pgxpool.Pool` 结构体调用 `Close()` 时不返回错误，这是由于它管理的连接具有池化特性。如果您使用的是该函数的 `pgx.Connect()` 变体——我不推荐在多线程环境中使用——您将获得一个带有 `Close()` 函数的对象来调用。

假设我们不是连接到本地集群，而是想连接到一个基于云的集群；在这种情况下，您需要传递一个类似于以下的连接字符串：`postgresql://<USERNAME>:<PASSWORD>@<HOST>:<PORT>/<DB_NAME>?sslmode=verify-full&sslrootcert=<PATH_TO_CERT>&options=--cluster%3D<CLUSTER_NAME>`。现在我们准备好插入数据了。以下语句向 "people" 表插入三个名字：

```
stmt := `INSERT INTO person (name) VALUES ($1), ($2), ($3)`

_, err = db.Exec(
    context.Background(),
    stmt,
    "Annie Easley", "Valentina Tereshkova", "Wang Zhenyi",
)
```

数据在表中后，我们现在可以将其读出：

```
stmt = `SELECT id, name FROM person`
rows, err := db.Query(context.Background(), stmt)
if err != nil {
    log.Fatalf("error querying table: %v", err)
}

var id, name string
for rows.Next() {
    if err = rows.Scan(&id, &name); err != nil {
        log.Fatalf("error reading row: %v", err)
    }
    log.Printf("%s %s", id, name)
}
```

最后，我们将使用 Go 工具链运行该应用程序：

`$ go run main.go`

```
2021/11/11 09:54:46 2917ee7b-3e26-4999-806a-ba799bc515b3 Wang Zhenyi
2021/11/11 09:54:46 5339cd29-77d5-4e01-966f-38ac7c6f9fbc Annie Easley
2021/11/11 09:54:46 85288df9-25fe-439a-a670-a8e8ea70c7db Valentina Tereshkova
```

#### Python 示例

接下来，我们将创建一个 Python 应用程序来完成同样的事情：连接到 “defaultdb” 数据库，进行 INSERT 和 SELECT 操作。

首先，我们初始化环境：

`$ mkdir python_example`

`$ python3 -m pip install --no-binary :all: psycopg2`

安装好 `psycopg2` 依赖项后，我们就可以编写应用程序了。让我们导入依赖项并创建到数据库的连接：

```
import psycopg2

conn = psycopg2.connect(
    dbname="defaultdb",
    user="demo",
    password="demo9315",
    port=26257,
    host="localhost",
)
```

使用接受连接字符串/数据源名称（DSN）的 `connect` 函数，还是为每个连接元素使用单独的变量，这是一个偏好问题。

接下来，我们将插入一些数据：

```
with conn.cursor() as cur:
    cur.execute(
        "INSERT INTO person (name) VALUES (%s), (%s), (%s)",
        ('Chien-Shiung Wu', 'Lise Meitner', 'Rita Levi-Montalcini')
    )
conn.commit()
```

`psycopg2` 的 `cursor` 函数返回一个游标，用于对数据库执行语句。使用 `with` 语句可确保在引发异常时回滚事务，而在未引发异常时提交事务。

最后，我们将读回我们插入的数据：

```
with conn.cursor() as cur:
    cur.execute("SELECT id, name FROM person")
    rows = cur.fetchall()
    for row in rows:
        print(row[0], row[1])
```

`psycopg2` 的 `fetchall` 命令返回一个元组列表，每个元组对应返回的一行。我们可以通过索引访问每个元组的列值。

#### Ruby 示例

接下来看 Ruby！让我们初始化环境：

`$ mkdir ruby_example`

`$ cd ruby_example`

接下来，我们将安装 `pg` gem：

`$ gem install pg`

我们准备好创建一个应用程序了。首先，引入 `pg` gem 并创建到数据库的连接：

```
require 'pg'

conn = PG.connect(
    user: 'demo',
    password: 'demo9315',
    dbname: 'defaultdb',
    host: 'localhost',
    port: "26257",
    sslmode: 'require'
)
```

接下来，让我们插入一些数据。我们将重用 `person` 表：

```
conn.transaction do |tx|
    tx.exec_params(
        'INSERT INTO person (name) VALUES ($1), ($2), ($3)',
```

