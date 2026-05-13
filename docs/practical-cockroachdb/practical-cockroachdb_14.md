# 第五章 与 CockroachDB 交互

简而言之，在面向生产规模的系统中工作时，像这样创建表是不可取的。提供表的完全限定名称至关重要。在下面的例子中，我们消除了关于表应该存放位置的任何不确定性：

```
CREATE TABLE database_name.schema_name.table_name();
```

务必为你的表起描述性的名称。选择单数还是复数的表名，以及决定使用 `PascalCase`、`camelCase`、`snake_case` 还是 `SPoNgEBOb case`（海绵宝宝式大小写），都由你决定。然而，无论你如何决定，始终保持命名约定的一致性总是一个好主意。

#### 列数据类型

CockroachDB 拥有多种列类型，为了从数据库中获得最佳性能，你需要知道何时使用它们，何时不使用。

以下表为例。除了我刻意加入的两个基于角色的列、将 `age` 指定为整数而非时间戳，以及省略了数据库和模式之外，你还能发现多少有疑问的设计决策？

```
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name STRING,
    age INT,
    is_admin INT,
    primary_role STRING
);
```

以下是我能识别出的该表设计中潜在问题的列表：

-   除了 `ID` 列，所有列都可为空。
-   `ID` 列使用了 `SERIAL` 数据类型，这仅为兼容 Postgres 而存在。在大多数情况下，最好使用 `UUID` 数据类型，其值在范围内的分布更均匀。
-   `name` 和 `primary_role` 列的长度没有限制。
-   `age` 列可以存储从 -9,223,372,036,854,775,807 到 +9,223,372,036,854,775,807 的值，鉴于我们当前的科学进展，这似乎不太可能。
-   `is_admin` 列也可以存储一个很大的数字，但此列类型为 `INT` 的主要问题是，存在一种更好的数据类型来传达该列的布尔语义：`BOOL`。
-   考虑到我们只有有限的取值范围，使用 `ENUM` 数据类型作为 `primary_role` 可能更合适。

这是同一个表，但这次已解决了我发现的问题：

```
CREATE TYPE users_primary_role AS ENUM (
    'admin',
    'write',
    'read'
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING(50) NOT NULL,
    age INT2 NOT NULL,
    is_admin BOOL NOT NULL DEFAULT false,
    primary_role users_primary_role NOT NULL DEFAULT 'read'
);
```

总结一下：

-   通常，总有一种数据类型非常适合你想存储的数据。
-   如果你能对列中的数据量做出合理预测，请尽可能使用该数据类型的有限可变长度变体。
-   如果你需要某列有数据，请将其设为非空。
-   使用 `ENUM` 来表示有限字符串值集合之间的选择。
-   考虑使用 `DEFAULT` 值来实现最小权限语义。

## 索引

使用索引通过为 CockroachDB 提供在何处查找你请求的值的提示，来提升 `SELECT` 查询的性能。

CockroachDB 区分两种索引类型：

-   **主索引** – 每个表都会获得一个主索引。如果你为表提供了主键，这将成为该表的主索引。如果你没有提供索引，CockroachDB 将创建一个名为 "rowid" 的主索引，并为每一行分配一个唯一值。
-   **次级索引** – 次级索引用于提升 `SELECT` 查询的性能，建议为你将在查询中进行过滤或排序的任何列创建次级索引。

在以下示例中，我创建了一个没有主索引或次级索引的表：

```
CREATE TABLE person (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    date_of_birth TIMESTAMP NOT NULL,
    pets STRING[]
);
```

让我们运行一个查询，看看我们的表在 CockroachDB 中是什么样子的：

```
SELECT column_name, column_default, is_nullable, data_type;
```


```sql
FROM defaultdb.information_schema.columns
WHERE table_name = 'person';
```

`column_name` | `column_default` | `is_nullable` | `data_type`
----------------|-------------------|-------------|-----------------------
`id` | `gen_random_uuid()` | NO | `uuid`
`date_of_birth` | NULL | NO | `timestamp without time zone`
`pets` | NULL | YES | `ARRAY`
`rowid` | `unique_rowid()` | NO | `bigint`

## 第五章 与 CockroachDB 交互

如前所述，CockroachDB 生成了一个名为 `rowid` 的第四列，作为我们缺失的主键列的替代。该列的值是自动递增的，但不一定是连续的。通过调用 `unique_rowid()` 可以看出这一点：

```sql
SELECT abs(unique_rowid() - unique_rowid()) AS difference;
```

`difference`

要查看 CockroachDB 为我们生成了什么，我将运行另一个命令来生成该表的 `CREATE` 语句：

```sql
SHOW CREATE TABLE person;
```

`table_name` | `create_statement`
-------------|-----------------------------------------------------------
`person` | ```CREATE TABLE public.person (  id UUID NOT NULL DEFAULT gen_random_uuid(),  date_of_birth TIMESTAMP NOT NULL,  pets STRING[] NULL,  rowid INT8 NOT VISIBLE NOT NULL DEFAULT unique_rowid(),  CONSTRAINT "primary" PRIMARY KEY (rowid ASC),  FAMILY "primary" (id, date_of_birth, pets, rowid) )```

让我们再次创建表，但这次提供我们自己的主键：

```sql
CREATE TABLE person (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  date_of_birth TIMESTAMP NOT NULL,
  pets STRING[]
);
```

现在我们表中的 `ID` 列是主键，CockroachDB 没有为我们创建自己的主键：

```sql
SELECT column_name, column_default, is_nullable, data_type
FROM defaultdb.information_schema.columns
WHERE table_name = 'person';
```

## 第五章 与 CockroachDB 交互

`column_name` | `column_default` | `is_nullable` | `data_type`
----------------|-------------------|-------------|-----------------------
`id` | `gen_random_uuid()` | NO | `uuid`
`date_of_birth` | NULL | NO | `timestamp without time zone`
`pets` | NULL | YES | `ARRAY`

表就位后，让我们向其中插入一些数据，并查看查询时 CockroachDB 如何搜索表：

```sql
INSERT INTO person (date_of_birth, pets) VALUES
  ('1980-01-01', ARRAY['Mittens', 'Fluffems']),
  ('1979-02-02', NULL),
  ('1996-03-03', ARRAY['Mr. Dog']),
  ('2001-04-04', ARRAY['His Odiousness, Lord Dangleberry']),
  ('1954-05-05', ARRAY['Steve']);
```

```sql
EXPLAIN SELECT id, date_of_birth FROM person
WHERE date_of_birth BETWEEN '1979-01-01' and '1997-12-31'
AND pets <@ ARRAY['Mr. Dog']
ORDER BY date_of_birth;
```

```
distribution: full
vectorized: true
• sort
│ estimated row count: 0
│ order: +date_of_birth
│
└── • filter
    │ estimated row count: 0
    │ filter: ((date_of_birth >= '1979-01-01 00:00:00') AND (date_of_birth <= '1997-12-31 00:00:00')) AND (pets <@ ARRAY['Mr. Dog'])
    │
    └── • scan
```

## 第五章 与 CockroachDB 交互

```
estimated row count: 5 (100% of the table; stats collected 5 minutes ago)
table: person@primary
spans: FULL SCAN
```

糟糕！CockroachDB 不得不执行整个 `person` 表的完整扫描来完成我们的查询。随着向数据库添加的每一行新数据，查询速度会线性变慢。

二级索引来救援！让我们重新创建表，并在我们要过滤和排序的列上添加索引。在我们的例子中，就是 `date_of_birth` 和 `pets` 列：

```sql
CREATE TABLE person (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  date_of_birth TIMESTAMP NOT NULL,
  pets STRING[],
  INDEX (date_of_birth),
  INVERTED INDEX (pets)
);
```

```sql
EXPLAIN SELECT id, date_of_birth FROM person
WHERE date_of_birth BETWEEN '1979-01-01' and '1997-12-31'
AND pets <@ ARRAY['Mr. Dog']
ORDER BY date_of_birth;
```

```
distribution: local
vectorized: true
• filter
│ filter: pets <@ ARRAY['Mr. Dog']
│
└── • index join
   │ table: person@primary
   │
   └── • scan
       missing stats
       table: person@person_date_of_birth_idx
       spans: [/'1979-01-01 00:00:00' - /'1997-12-31 00:00:00']
```

## 第五章 与 CockroachDB 交互

成功！我们的查询不再需要全表扫描。但是，请注意如果我们按 `date_of_birth` 的降序排序结果会发生什么：

```sql
EXPLAIN SELECT id, date_of_birth FROM person
```

# 第 5 章 与 CockroachDB 交互

WHERE `date_of_birth` BETWEEN '1979-01-01' and '1997-12-31'

AND `pets` <@ ARRAY['Mr. Dog']

ORDER BY `date_of_birth` DESC;

`distribution: full`
`vectorized: true`

- `sort`
  - `order: -date_of_birth`
- `filter`
  - `filter: pets <@ ARRAY['Mr. Dog']`
- `index join`
  - `table: person@primary`
- `scan`
  - `missing stats`
  - `table: person@person_date_of_birth_idx`
  - `spans: [/'1979-01-01 00:00:00' - /'1997-12-31 00:00:00']`

CockroachDB 必须在返回结果前为我们手动排序，因为表上的索引默认按升序排列，而我们要求按降序排列。

假设你知道 CockroachDB 可能按某列以升序或降序返回查询结果。在这种情况下，值得考虑为这两种场景添加索引，如下所示：

```sql
CREATE TABLE person (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    date_of_birth TIMESTAMP NOT NULL,
    pets STRING[],
    INDEX person_date_of_birth_asc_idx (date_of_birth ASC),
    INDEX person_date_of_birth_desc_idx (date_of_birth DESC),
    INVERTED INDEX (pets)
);
```

有了这个，CockroachDB 就知道如何返回按 `date_of_birth` 升序或降序排列的查询结果，而无需手动排序。

如果你正在索引包含顺序数据的列，例如 `TIMESTAMP` 或递增的 `INT` 值，你可能需要考虑使用哈希分片索引.3

哈希分片索引确保索引数据均匀分布在各个范围（range）中，从而防止单范围热点。当查询多个相似值且这些值存在于同一个范围中时，就会发生单范围热点。

在撰写本文时，哈希分片索引仍处于实验阶段，因此需要按如下方式启用：

```sql
SET experimental_enable_hash_sharded_indexes = true;

CREATE TABLE device_event (
    device_id UUID NOT NULL,
    event_timestamp TIMESTAMP NOT NULL,
    INDEX (device_id, event_timestamp) USING HASH WITH bucket_count = 10
);
```

在底层，索引中的值将是你要索引的数据的 `FNV` 哈希表示。对输入数据的任何微小更改都可能导致不同的哈希输出和不同的索引存储桶（bucket）。现在，表可以在集群的节点之间更均匀地扩展，从而提高性能。

3 [www.cockroachlabs.com/docs/stable/hash-sharded-indexes](http://www.cockroachlabs.com/docs/stable/hash-sharded-indexes)

### 视图设计

视图只是一个你为其命名并让 CockroachDB 记住的 `SELECT` 查询。CockroachDB 支持三种类型的视图：

- `物化视图` – CockroachDB 存储结果数据，以提高后续查询的性能。我仅建议在无法为表中的数据创建索引以提升查询性能，且该表中的数据不经常更新的情况下使用物化视图。如果数据频繁更新，你的结果可能会过时。
- `非物化视图` – 视图的结果不被存储，这意味着数据始终与表保持一致。由于结果不被存储，CockroachDB 将为每次视图查询查询底层表。如果你已为表适当地建立了索引以提升性能，非物化视图是最佳选择，因为它们总是返回正确的数据。
- `临时视图` – 临时视图是临时性的视图，只能在创建它们的会话中访问，并在该会话结束时删除。

如果你需要将表数据限制给特定用户，或者有希望以更简单形式呈现的复杂查询，视图是一个不错的选择。让我们创建一些视图，看看它们能为我们提供什么。

#### 简化查询

假设你有一个希望隐藏其复杂性（或只是不想重复编写）的查询。你可以基于此数据创建一个视图，以呈现一个更简单的查询。


# 第五章 与 CockroachDB 交互

假设我们正在经营一家网店，并且希望有一个查询能返回每日的购买数量。诚然，这是一个直接的查询，但为了论证起见，我们将把它封装在一个视图中以演示这个概念。

#### 基本视图示例

首先，我们将创建表：

```sql
CREATE TABLE customer (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING(50) NOT NULL,
    INDEX(email)
);

CREATE TABLE product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku STRING(20) NOT NULL,
    INDEX(sku)
);

CREATE TABLE purchase (
    customer_id UUID NOT NULL REFERENCES customer(id),
    product_id UUID NOT NULL REFERENCES product(id),
    checkout_at TIMESTAMPTZ NOT NULL
);
```

接下来，我们将向表中插入一些数据以便进行操作：

```sql
INSERT INTO customer (email) VALUES
    ('m.curie@gmail.com'),
    ('l.meitner@gmail.com'),
    ('d.hodgkin@gmail.com');

INSERT INTO product (sku) VALUES
    ('E81ML1GA'),
    ('4UXK19D0'),
    ('L28C0XJ4');

INSERT INTO purchase (customer_id, product_id, checkout_at) VALUES
    ('2df28d7d-67a3-4e83-a713-96a68a196b0b', '486ecc02-6d8e-44fe-b8ef-b8d004658465', '2021-11-23T06:50:46Z'),
    ('5047d78c-12cc-48e3-a36a-7cac1018ba7e', '5b2756e6-cc88-47e6-849d-6291a76f3881', '2021-11-22T08:13:13+07:00'),
    ('792ab188-1bb2-4ac6-b1d2-f1adbcf9223e', '6cfd7bed-bf11-4c96-9300-82c8e9dd97f2', '2021-11-23T06:50:46-05:00');
```

最后，我们将创建视图。这个视图将是对 `purchase` 表的一个简单选择，并按从 `checkout_at` 列提取的星期几的值进行分组：

```sql
CREATE VIEW purchases_by_dow (dow, purchases) AS
    SELECT EXTRACT('ISODOW', pu.checkout_at), COUNT(*)
    FROM purchase pu
    GROUP BY EXTRACT('ISODOW', pu.checkout_at);
```

有了这个视图，我们就可以针对视图进行操作，它隐藏了日期处理的复杂性：

```sql
SELECT * FROM purchases_by_dow;
```

```
  dow | purchases
------+------------
    2 |         2
    1 |         1
```

#### 限制表数据

视图的另一个良好用例是限制对数据的访问。在下面的例子中，我们将创建一个表，并将其数据限制为用户组访问。

假设在前面的例子中，我们的商店是全球运营的。我们可能不希望在所有地方销售所有产品，因此限制在不同国家展示给用户的产品是合理的。

在开始之前，我们需要创建并连接到一个安全的集群。现在让我们创建一个单节点集群。

```bash
mkdir certs
mkdir keys

cockroach cert create-ca \
    --certs-dir=certs \
    --ca-key=keys/ca.key

cockroach cert create-node \
    --certs-dir=certs \
    --ca-key=keys/ca.key \
    localhost

cockroach start-single-node \
    --certs-dir=certs \
    --store=node1 \
    --listen-addr=localhost:26257 \
    --http-addr=localhost:8080

cockroach cert create-client \
    root \
    --certs-dir=certs \
    --ca-key=keys/ca.key

cockroach sql --certs-dir=certs
```

首先，我们将创建 `products` 表并向其中插入一些产品：

```sql
CREATE TABLE product (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_iso STRING(3) NOT NULL,
    sku STRING(20) NOT NULL
);

INSERT INTO product (country_iso, sku) VALUES
    ('BRA', 'E81ML1GA'),
    ('FIN', '4UXK19D0'),
    ('GBR', 'L28C0XJ4');
```

接下来，我们将创建一个视图，返回给英国客户的所有产品：

```sql
CREATE VIEW product_gbr (id, country_iso, sku) AS
    SELECT id, country_iso, sku FROM product
    WHERE country_iso = 'GBR';
```

最后，我们将创建一个用户并授予他们对该视图的访问权限。我们假设以下用户将服务英国客户：

```sql
CREATE USER user_gbr WITH LOGIN PASSWORD 'some_password';

GRANT SELECT ON product_gbr TO user_gbr;

REVOKE SELECT ON product FROM user_gbr;
```

让我们以新用户身份连接到数据库，看看我们能（和不能）访问什么：

```bash
cockroach sql \
    --certs-dir=certs \
    --url "postgres://user_gbr:some_password@localhost:26257/defaultdb"
```

```sql
user_gbr@localhost:26257/defaultdb> SELECT * FROM product;
ERROR: user user_gbr does not have SELECT privilege on relation product
SQLSTATE: 42501

user_gbr@localhost:26257/defaultdb> SELECT * FROM product_gbr;
```



# 第五章 与 CockroachDB 交互

### 移动数据

在本节中，我们将探讨一些涉及通过 `IMPORT` 和 `EXPORT` 语句以及 CDC（变更数据捕获）将数据移入和移出数据库的用例。

### 导出和导入数据

除非你正在创建一个新数据库来存储新数据，否则你很可能需要从另一个数据库迁移数据。本节将向你展示如何将数据从 CockroachDB 表中导出以及导入数据到其中。

首先，让我们创建一个单节点集群：

```bash
$ cockroach demo --no-example-database --insecure
```

集群建立后，我们将创建一个包含两张表的数据库：一张用于导出数据，另一张用于导入数据。我们还将插入一些要导出的数据：

```sql
CREATE DATABASE import_export;

USE import_export;

CREATE TABLE to_export (
    name TEXT NOT NULL
);

CREATE TABLE to_import (
    name TEXT NOT NULL
);

INSERT INTO to_export (name) VALUES ('A'), ('B'), ('C');
```

接下来，我们需要一个导出数据的地方。CockroachDB 支持云提供商目的地，如 Amazon S3、Azure Storage 和 Google Cloud Storage，并允许你导出到自托管目的地。

对于此示例，我创建了一个基本的 HTTP 服务器，托管在 [replit.com](http://replit.com) 上（[地址为 https://replit.com/@robreid/importexport](https://replit.com/@robreid/importexport)）。你可以注册一个免费的 Replit 账户并 fork 这个 Repl 来测试你自己的导出/导入。

服务器启动并运行后（地址为 `https://importexport.robreid.repl.co`），我可以按如下方式导出数据（你的 URL 将是 `https://importexport.YOUR_USERNAME.repl.co`）：

```sql
EXPORT INTO CSV 'https://importexport.robreid.repl.co'
FROM TABLE to_export;
```

```
filename                                                                   | rows | bytes
--------------------------------------------------------------------------+------+------
export16b4ae42664118980000000000000001-n707964976657694721.0.csv | 3    | 6
```

我们可以通过在运行的 Replit 地址后加上文件名来确认文件已被服务器成功接收：

```bash
$ curl https://importexport.robreid.repl.co/export16b4ae42664118980000000000000001-n707964976657694721.0.csv
A
B
C
```

现在让我们将这些数据导入到我们的 `to_import` 表中：

```sql
IMPORT INTO to_import (name)
    CSV DATA (
    'https://importexport.robreid.repl.co/export16b4ae42664118980000000000000001-n707964976657694721.0.csv'
);
```

```
job_id       | status    | fraction_completed | rows | index_entries | bytes
-------------+-----------+--------------------+------+---------------+-------
707965889435041793 | succeeded | 1                  | 3    | 0             | 60
```

CockroachDB 还支持从 Avro 文件、TSV 和其他分隔符文件，以及 CockroachDB、MySQL 和 Postgres 的 dump 文件中导入数据。我建议在以下场景中使用 `IMPORT` 语句：
- 你需要快速插入大量数据。
- 你需要在应用程序典型插入流程之外插入数据。在这种情况下，拥有一个从文件导入数据的流程可能是有意义的。

### 监控数据库变更

与 Postgres 类似，CockroachDB 支持变更数据捕获（CDC），它允许你将表数据的变更流式传输出 CockroachDB 并进入外部系统。

在本节中，我们将创建一个企业级变更数据流（Enterprise Changefeed）并将数据变更从一个表中流式传输出来。变更数据流在企业和核心集群中均可用，但创建语法和行为略有不同。

#### Kafka Sink

Kafka CDC sink 是 CockroachDB 中第一个出现的 sink，首次出现在 v2.1 版本。

---
d1c055ed-1a2e-4383-875e-3c53a2bf65cc | GBR | L28C0XJ4

我们的 `user_gbr` 用户没有直接查询 `product` 表的权限，但*确实*有权通过 `product_gbr` 视图访问其数据，该视图仅公开我们希望他们看到的产品。

---
*（注：此部分似乎是示例数据或无关注释，与主要章节内容不直接相关，因此单独列出。）*



# 第 5 章 与 CockroachDB 交互

在本示例中，我将使用 Redpanda，这是一个与 Kafka 线协议兼容的流平台（其方式类似于 CockroachDB 与 Postgres 的线协议兼容性）。让我们启动一个 Redpanda 实例，创建一个主题，并从中消费数据：

```bash
docker run \
--rm -it \
--name redpanda \
-p 9092:9092 \
-p 9644:9644 \
docker.vectorized.io/vectorized/redpanda:latest \
redpanda start \
--overprovisioned \
--smp 1 \
--memory 1G \
--reserve-memory 0M \
--node-id 0 \
--check=false
```

```bash
docker exec -it \
redpanda rpk topic create cdc_example \
--brokers=localhost:9092
```

```bash
docker exec -it \
redpanda rpk topic consume cdc_example \
--brokers=localhost:9092
```

回到 CockroachDB 这边，让我们创建一个简单的非安全演示集群，并带有一个用于流式传输数据的示例表：

```bash
cockroach demo --no-example-database --insecure
```

```sql
CREATE TABLE example (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    value STRING(10) NOT NULL
);
```

表建好后，让我们创建一个 Kafka 变更流来开始发布更改：

```sql
SET CLUSTER SETTING kv.rangefeed.enabled = true;

CREATE CHANGEFEED FOR TABLE example
INTO 'kafka://localhost:9092?topic_name=cdc_example'
WITH updated;
```

注意，要为多个表创建变更流，只需提供一个逗号分隔的字符串，其中包含你想要监控的表，如下所示：

```sql
CREATE CHANGEFEED FOR TABLE table_one, table_two, table_three
INTO 'kafka://localhost:9092?topic_name=cdc_example'
WITH updated;
```

现在，将数据送入 Kafka 唯一需要做的就是正常使用你的表。对我们示例表数据的所有更改都将自动发布到 Kafka。现在让我们`INSERT`、`UPDATE`和`DELETE`一些数据，看看 CockroachDB 向`cdc_example`主题发布了什么：

```sql
INSERT INTO example (value) VALUES ('a');

UPDATE example
SET value = 'b'
WHERE id = '4c0ebb98-7f34-436e-9f6b-7ea1888327d9';

DELETE FROM example
WHERE id = '4c0ebb98-7f34-436e-9f6b-7ea1888327d9';
```

此时，你将开始看到通过 Redpanda 消费者从 CockroachDB 接收到的事件。接收到的第一条消息代表`INSERT`语句引入的更改，第二条消息是在`UPDATE`语句后发布的，最后一条消息显示了因`DELETE`语句而导致的行被删除：

```json
{
    "topic": "cdc_example",
    "key": "[\"4c0ebb98-7f34-436e-9f6b-7ea1888327d9\"]",
    "value": "{\"after\": {\"id\": \"4c0ebb98-7f34-436e-9f6b-7ea1888327d9\", \"value\": \"a\"}, \"updated\": \"1637755793947605000.0000000000\"}",
    "timestamp": 1637755794706,
    "partition": 0,
    "offset": 44
}

{
    "topic": "cdc_example",
    "key": "[\"4c0ebb98-7f34-436e-9f6b-7ea1888327d9\"]",
    "value": "{\"after\": {\"id\": \"4c0ebb98-7f34-436e-9f6b-7ea1888327d9\", \"value\": \"b\"}, \"updated\": \"1637755826616641000.0000000000\"}",
    "timestamp": 1637755826791,
    "partition": 0,
    "offset": 45
}

{
    "topic": "cdc_example",
    "key": "[\"4c0ebb98-7f34-436e-9f6b-7ea1888327d9\"]",
    "value": "{\"after\": null, \"updated\": \"1637756011024955000.0000000000\"}",
    "timestamp": 1637756011195,
    "partition": 0,
    "offset": 46
}
```

要停止变更流，你需要定位并删除正在运行它的作业。现在让我们找到该作业并取消它：

```sql
root@127.0.0.1:26257/defaultdb> SELECT job_id, job_type, description FROM [SHOW JOBS];
```

```
  job_id             | job_type | description
---------------------+----------+--------------------------------------
  713311406086291457 | CHANGEFEED | CREATE CHANGEFEED FOR TABLE example INTO 'kafka://localhost:9092?topic_name=cdc_example' WITH updated
```

```sql
root@127.0.0.1:26257/defaultdb> CANCEL JOB 713311406086291457;
```

#### Webhook Sink

现在让我们将 CDC 事件重定向到一个 Web 服务器。鉴于你表中数据的潜在敏感性，CockroachDB 仅能将数据发送到启用 HTTPS 的服务器。如同之前的导出/导入示例，我将复用`replit.com`来创建并运行一个免费的 HTTPS 服务器。让我们创建并运行一个简单的 HTTPS 服务器：

```go
package main

import (
    "io/ioutil"
    "log"
    "net/http"
)

func main() {
```



# 第五章 与 CockroachDB 交互

```go
http.HandleFunc("/", cdc)

log.Fatal(http.ListenAndServe(":9090", nil))

}

func cdc(w http.ResponseWriter, r *http.Request) {

	defer func() {
		if err := r.Body.Close(); err != nil {
			log.Printf("error closing request body: %v", err)
		}
	}()

	event, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Printf("error reading request body: %v", err)
	}

	log.Println(string(event))

}
```

欢迎将我的 Repl 项目复制一份：[`replit.com/@robreid/cdcexample`](https://replit.com/@robreid/cdcexample)。

复制你正在运行的服务器的 URL（我的是`https://cdcexample.robreid.repl.co`），并将其配置到 CockroachDB 中，以开始发布 Webhook CDC 事件：

```sql
CREATE CHANGEFEED FOR TABLE example
INTO 'webhook-https://cdcexample.robreid.repl.co?insecure_tls_skip_verify=true'
WITH updated;
```

最后，让我们用`INSERT`、`UPDATE`和`DELETE`语句生成一些事件：

```sql
INSERT INTO example (value) VALUES ('b');

UPDATE example SET value = 'b'
WHERE id = 'aea98953-0a9b-4315-a55f-e95bc7dc8305';

DELETE FROM example
WHERE id = 'aea98953-0a9b-4315-a55f-e95bc7dc8305';
```

切回到 replit.com，我们的服务器现在正在接收 CockroachDB 变更的事件：

```
2021/11/24 12:41:13 {"payload":[{"after":{"id":"aea98953-0a9b-4315-a55f-e95bc7dc8305","value":"b"},"key":["aea98953-0a9b-4315-a55f-e95bc7dc8305"],"topic":"example","updated":"1637757672559012000.0000000000"}],"length":1}

2021/11/24 12:41:20 {"payload":[{"after":{"id":"aea98953-0a9b-4315-a55f-e95bc7dc8305","value":"b"},"key":["aea98953-0a9b-4315-a55f-e95bc7dc8305"],"topic":"example","updated":"1637757679279445000.0000000000"}],"length":1}

2021/11/24 12:41:20 {"payload":[{"after":null,"key":["aea98953-0a9b-4315-a55f-e95bc7dc8305"],"topic":"example","updated":"1637757680248432000.0000000000"}],"length":1}
```

# 第六章 数据隐私

数据隐私应该是无论身处世界何处、规模大小的企业都应关注的问题。虽然因安全措施实施不力而产生的处罚法律可能因国家而异，但保护用户是至关重要的。

在本主题中，我们将深入探讨 CockroachDB 如何帮助您为数据创建安全且合规的基础设施。

### 全球法规

没有任何一部数据隐私法可以管辖全世界。每个国家都有其自身的立法，并且可能随着时间的推移进行修订。这就形成了一个不断变化的格局，我们作为工程师需要跟上并保持合规。律师事务所 DLA Piper 维护了一个优秀的网站，按国家逐个概述了这一格局：`www.dlapiperdataprotection.com`。我将使用 DLA Piper 的研究作为本部分内容的起点。

我在本节中介绍的内容不应被视为法律建议。在为客户数据创建生产级 CockroachDB 集群之前，请咨询数据隐私专家。

以下是一些要点，让您了解这种合规挑战的复杂性：

• 没有公认的成文标准可以全球性地认定数据应受保护。在一些监管严格的国家，个人身份信息（`PII`）的定义很明确，而在其他一些监管严格的国家，则根本没有定义。

• 在美国，数据隐私立法权下放给各州，这意味着每个州都可能有不同的数据隐私法。

• 《加州消费者隐私法案》（`CCPA`）现已生效，成为首个全州范围的综合性隐私政策。虽然常与欧洲的《通用数据保护条例》（`GDPR`）相比较，但两者在细微但重要的方面存在差异：`GDPR`框架要求个人的“事先同意”，而`CCPA`框架赋予个人“选择退出”的权利。1

• `GDPR`将数据收集和处理的同意年龄定为 16 岁，但允许欧洲成员国根据需要降低这一年龄。因此，在欧洲，关于儿童数据保护没有统一的定义。2

• 脱欧后，英国制定了英国特定的`GDPR`，其内容与欧盟版本大体相同。然而，英国《2018 年数据保护法案》（`DPA`）的某些部分会影响`GDPR`所提供的规定。

• 在欧洲，对于何谓数据传输没有标准定义。因此，任何涉及数据的操作都被视为传输。

法规及其差异的列表超出了本书的范围。然而，我希望通过分享上述要点，强调了在您运营或从其用户处收集或处理数据的任何国家了解数据隐私法律的重要性。

您的责任和义务将根据您被视为数据的处理者、控制者还是联合控制者而变化，因此理解两者之间的区别至关重要。在英国，`信息专员办公室`（`ICO`）提供了一个有用的清单来帮助您确定自己的角色。3 简而言之，如果您决定收集数据并知晓处理数据的方式和原因，那么您就是控制者。

1. [`www.cookiebot.com/en/ccpa-vs-gdpr-compliance-with-cookiebot-cmp`](http://www.cookiebot.com/en/ccpa-vs-gdpr-compliance-with-cookiebot-cmp)
2. [`www.clarip.com/data-privacy/gdpr-child-consent`](http://www.clarip.com/data-privacy/gdpr-child-consent)
3. [`https://ico.org.uk/for-organisations/guide-to-data-protection/guide-to-the-general-data-protection-regulation-gdpr/key-definitions/controllers-and-processors/`](https://ico.org.uk/for-organisations/guide-to-data-protection/guide-to-the-general-data-protection-regulation-gdpr/key-definitions/controllers-and-processors/)

您的责任和义务也会根据您处理的数据类型而变化。如果您存储的是医疗保健数据（例如患者姓名和健康状况），您将受到比在线零售环境中存储客户数据（例如客户电子邮件和实体地址）更严格的审查。

根据经验，如果您在多个国家运营，明智的做法是遵守数据隐私法律最严格的国家的规定，并将其应用到所有国家。

此外，如果两个国家的规则存在冲突，您可能需要根据客户的所在国/州，为每个客户单独实施相应的规则。

尽管驾驭数据隐私的复杂水域仍然是所有数据库用户需要考虑的重要问题，但 CockroachDB 的传输中加密、静态加密和地理分区功能使得这种复杂性更易于管理。

### 特定地点的考量因素

在本节中，我们将更详细地探讨在跨地域运营以及面临影响您合规责任的不同情况时，您需要考虑的因素。

#### 拥有英国和欧洲客户的英国公司

如果您是一家总部位于英国的公司，并且拥有英国和欧盟的客户，您将受到`GDPR`和`UK-GDPR`的约束。脱欧后，英国采纳了原始`GDPR`框架的所有原则，这意味着如果您满足`GDPR`要求，您将在欧盟和英国都保持合规。

© Rob Reid 2022
R. Reid, *Practical CockroachDB*, [`doi.org/10.1007/978-1-4842-8224-3_6`](https://doi.org/10.1007/978-1-4842-8224-3_6#DOI)


# 第六章 数据隐私

2021 年 6 月 28 日，欧洲委员会承认欧洲公民的个人数据可以通过两项充分性决定.4 安全地存储在英国。因此，您可以自由选择将数据存储在英国或欧盟。由于个人数据可以自由流入和流出英国，您在集群架构设计上拥有灵活性：

-   **在英国处理所有客户数据** – 为降低基础设施成本和复杂性，您可以将集群部署在英国的单个区域内。鉴于英国与欧洲其他地区的邻近性，到中欧的延迟将在 12 毫秒至 33 毫秒之间（具体取决于您的云提供商）。请访问 `www.cloudping.co/` 查看 `AWS` 区域的延迟网格。
-   **在欧盟处理所有客户数据** – 与将所有数据存放在英国单个区域可以降低成本和简化基础设施的原因相同，将所有数据存放在欧盟的单个区域也是一个不错的选择。
-   **一个集群，节点靠近您的用户** – `CockroachDB` 的跨区域水平扩展能力在这种场景下会表现良好。您可以创建一个跨多个区域的数据库节点集群，并将节点部署在客户附近，无论他们身处英国还是欧洲。

#### 拥有欧洲和美国客户的欧洲公司

如果您是一家总部位于欧洲的公司，客户分布在欧洲和美国，那么您将同时受到 `GDPR` 和您在美国客户所在州的法规约束。

由于 `GDPR` 目前比任何美国数据隐私框架（包括 `CCPA`）都更严格，对您的欧洲和美国客户都采用 `GDPR` 标准可能是合理的。

关于存储数据的位置，您有三个主要选项：

-   **在欧洲处理所有客户数据** – Schrems II 裁决保护欧洲人免受美国政府监控，因此将美国客户数据存储在欧盟数据中心比将欧盟客户数据存储在美国数据中心更直接。这种方法明显的缺点是，如果美国客户的请求直接访问欧盟数据库节点，他们将面临延迟问题。
-   **一个具有地理分离节点的集群** – 如果您希望运行单个多区域集群，可以使用地理分区来分离欧盟和美国客户数据。
-   **多个独立集群** – 如果您更愿意将欧盟和美国客户数据完全隔离，可以运行两个集群：一个在欧盟，一个在美国。

这些选项各有利弊。在这种情况下，我建议运行一个地理分区集群或多个独立集群。

#### 拥有美国客户的美国公司

如果您是一家总部位于美国且只有美国客户的公司，您需要考虑美国各地不同的州法规。

在撰写本文时，美国使用了三个综合性框架：适用于加利福尼亚州、弗吉尼亚州和科罗拉多州客户的 `CCPA`、`VCDPA` 和 `ColoPA`。每个框架为客户提供不同级别的保护。例如，`VCDPA` 吸取了亚马逊的意见，因此更有利于企业而非客户。5

与欧盟/美国示例类似，与其试图遵守数量不断增长且差异巨大的数据隐私框架，不如以遵守最严格的框架为目标。我们已确定 `VCDPA` 不是以客户为中心的框架，所以 `CCPA` 将是更严格的框架，但 `ColoPA` 呢？`CCPA` 赋予客户 *某些* 就数据处理疏忽起诉公司的权利，但 `ColoPA` 没有提供此类权利。

对加利福尼亚州以外的客户适用 `CCPA` 是合理的，理由与对美国客户适用 `GDPR` 相同；它是更严格的数据隐私框架。

#### 拥有美国和欧洲客户的美国公司

2020 年 7 月 16 日，Schrems II 裁决生效。6 It 认定根据《外国情报监视法》(`FISA`) 进行的美国政府监控的可能性对欧盟公民的数据隐私构成了足够大的威胁，从而废除了之前适用的欧盟-美国数据保护盾协议。

因此，希望存储欧盟客户数据的美国公司必须遵守 `GDPR` 法规。鉴于此，在这种情况下分离美国和欧盟客户数据是明智之举。如果我们需要在美国对数据进行任何分析处理，可以获取欧盟客户数据的匿名副本并将其存储在美国是可以接受的。

我们可以设计多种合规的数据库基础设施来应对这种情况。让我们回顾两个选项：

-   **多个独立集群** – 这是在此场景下最安全的操作方式，因为集群信息、凭据和证书保持分离。
-   **一个具有地理分离节点的集群** – 除了分离节点，我们还可以实施一种机制来防止欧盟数据离开欧洲。这可以简单地是一个视图，该视图仅允许美国服务访问美国数据。

#### 拥有中国客户的非中国公司

2021 年 11 月 1 日，中国实施了其首个综合性数据隐私框架——《个人信息保护法》(`PIPL`)。与 `CCPA` 类似，`PIPL` 借鉴了 `GDPR`，但不同于关注个人权利和消费者权利的 `GDPR` 和 `CCPA`，`PIPL` 更侧重于国家安全.7

希望存储中国客户数据的公司必须在中国境内进行存储。对我们来说，这意味着在中国运行 `CockroachDB` 节点。任何需要在境外处理中国客户数据的公司，在处理前必须通过国家安全审查.8

除非您是中国企业，否则在中国托管基础设施并非易事。拥有 `AWS` 账户不足以将软件部署到中国。您需要在中国的 `AWS` 站点 `amazonaws.cn` 上创建一个单独的账户，并提供信息安全标识和商业登记证书。

为了在中国大陆实际托管站点，公司必须拥有一家注册的中国公司并完成 `ICP`（互联网内容提供商）商业许可证。9

### 个人身份信息

个人身份信息 (`PII`) 是向处理不当的公司开具巨额罚单的“黄金门票”。虽然 `PII` 的定义可能因国家/地区而异，但您可以将其视为任何可能识别某人身份的个人信息。例如，姓名、地址以及财务和地理信息都可用于识别某人。


以下是一份非穷尽的 PII 存储注意事项清单：

-   支付信息——如果您存储客户的财务信息，您对他们的责任将远大于合规要求。如果您有可用的支付处理方，请尽量将存储支付信息的责任委托给他们。

-   姓名——姓名或许是识别某人最明显的方式。您可能需要存储用户的姓名以便联系他们、处理支付或向其发货。如果您的网站无需存储他们的姓名（或全名），请尽量最小化存储。

-   联系客户——如果用户不允许您联系他们，请考虑对其电子邮件地址进行混淆处理，使其无法被发送邮件。这可以像对其电子邮件地址进行哈希处理一样简单，该过程仍允许您在登录时重新哈希并验证其电子邮件地址，而无需以明文形式存储。

### 加密

在考虑了全球法规和 PII 之后，数据的下一道防线是加密。如果实施得当，加密将使恶意行为者几乎无法理解您的数据并将其用于恶意目的。在本节中，我们将介绍 CockroachDB 中的数据加密。

从宏观层面看，使用加密时有两个重点领域：

-   **传输中**数据——这涉及在节点与终端用户之间或节点之间传输的数据。
-   **静态**数据——这涉及存储在物理磁盘上的数据。

CockroachDB 提供了两种实现方式，我将向您介绍这两种方式。

#### 传输中

默认情况下，CockroachDB 会尝试安全启动，并在启动时查找要使用的证书和密钥。如果您在未生成证书或密钥的情况下尝试运行以下命令，CockroachDB 将无法启动：

```bash
$ cockroach start-single-node
*
* ERROR: ERROR: cannot load certificates.
* Check your certificate settings, set --certs-dir, or use --insecure for insecure clusters.
*
* failed to start server: problem using security settings: no certificates found; does certs dir exist?
*
...
```

到目前为止，我的重点一直是演示 CockroachDB 的运行时功能，因此我一直在使用 `--insecure` 标志来绕过此检查，并以不安全模式运行数据库。对于除用于开发的本地 CockroachDB 安装以外的任何情况，您都应该通过使用证书保护集群安全。我们现在就来做这件事。

首先，创建两个目录：一个用于存储证书，一个用于存储密钥。请注意，一旦 CockroachDB 生成了证书和密钥，这两个目录都将包含敏感信息：

```bash
$ mkdir certs
$ mkdir keys
```

接下来，我们将要求 CockroachDB 生成一个证书颁发机构（CA）证书和一个私钥。我们将在后续步骤中使用这些文件为节点创建证书：

```bash
$ cockroach cert create-ca \
--certs-dir=certs \
--ca-key=keys/ca.key
```

```bash
$ ls certs
ca.crt
$ ls keys
ca.key
```

接下来，我们将要求 CockroachDB 为集群节点生成一个证书和一对公钥/私钥：

```bash
$ cockroach cert create-node \
--certs-dir=certs \
--ca-key=keys/ca.key \
localhost
```

```bash
$ ls certs
ca.crt node.crt node.key
$ ls keys
ca.key
```

前面的 `cockroach cert create-node` 命令接受一个可变数量的主机参数以帮助节点发现。在此示例中，我在本地运行所有内容，因此传递单个 `localhost` 值就足够了。

现在，让我们使用生成的证书和密钥来启动一些 CockroachDB 节点。之前，当我们尝试这样做时，CockroachDB 返回了一个错误，但现在我们拥有了安全启动所需的文件，因此它将毫无问题地启动：

```bash
$ cockroach start \
--certs-dir=certs \
--store=node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--join=localhost:26257,localhost:26258,localhost:26259
```

```bash
$ cockroach start \
--certs-dir=certs \
--store=node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257,localhost:26258,localhost:26259
```

#### 静态数据



# 第六章 数据隐私

### 静态加密

静态加密是 CockroachDB 的一项企业版功能。它会透明地加密磁盘上的数据，这意味着，即使是有数据访问权限的用户也永远不会察觉到加密过程的发生。

CockroachDB 使用高级加密标准算法来加密数据，支持 128、192 或 256 位三种密钥长度。

要生成加密密钥，只需调用 `cockroach` 二进制文件的 `gen` 命令。

以下命令生成一个 256 位 AES 密钥，本章后续内容将使用此密钥：

```
$ cockroach gen encryption-key -s 256 keys/enc.key
```

您必须安全管理此密钥。如果它落入不法之徒手中，任何能够访问您的 CockroachDB 节点的人都将能够获取您的数据。Cockroach Labs 对加密密钥管理提出以下建议：

*   只有运行 CockroachDB 进程的 UNIX 用户才应有权访问加密密钥。
*   不应将加密密钥与您的 CockroachDB 数据存储在同一位置。HashiCorp 的 Vault 之类的方案是一个不错的选择。
*   定期轮换加密密钥。每隔几个月到几个月进行一次。

#### 数据库加密

数据库加密涉及加密数据库中的所有表。如果您不确定某些特定表是否需要加密，这是一个明智的选择，因为它确保所有内容都被加密。

首先，让我们创建一些节点。这些命令与我们在“传输中加密”部分用于启动节点的命令相似，但有一个区别。在以下命令中，我们提供了一个额外的 `--enterprise-encryption` 参数，该参数告知 CockroachDB 我们将使用企业版加密功能。

`--enterprise-encryption` 参数使用键值对来配置加密特定的设置：

*   `path` – 将存储您的 CockroachDB 数据的目录。此值对应于提供给 `--store` 参数的值。
*   `key` – 当前加密密钥的绝对或相对路径。
*   `old-key` – 先前加密密钥的绝对或相对路径。使用 `plain` 告知 CockroachDB 回退到以明文（即未加密）形式存储数据。
*   `rotation-period` – 在此周期之后，CockroachDB 将自动轮换您的加密密钥。

以下命令创建三个启用加密的节点。为保持一致性，我将重用上一节中生成的证书：

```
$ cockroach start \
--certs-dir=certs \
--store=node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--enterprise-encryption=path=node1,key=keys/enc.key,old-key=plain
```

```
$ cockroach start \
--certs-dir=certs \
--store=node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--enterprise-encryption=path=node2,key=keys/enc.key,old-key=plain
```

```
$ cockroach start \
--certs-dir=certs \
--store=node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--enterprise-encryption=path=node3,key=keys/enc.key,old-key=plain
```

让我们连接到我们的集群。由于这是一个安全集群，我们需要生成并提供证书来验证用户身份。我们将为数据库的 root 用户生成客户端证书并使用它进行连接。首先，生成 root 客户端证书：

```
$ cockroach cert create-client \
root \
--certs-dir=certs \
--ca-key=keys/ca.key
```

现在我们可以初始化节点并完成集群设置：

```
$ cockroach init --certs-dir=certs --host=localhost:26257
```

最后，使用我们生成的客户端证书以 root 用户身份连接到集群：

```
$ cockroach sql --certs-dir=certs
```

如果您现在在浏览器中打开 CockroachDB 管理控制台 (`https://localhost:8080`)，系统会要求您使用用户名和密码登录。这些必须从 CockroachDB shell 中创建，如下所示：

```
CREATE USER rob WITH PASSWORD 'A_STRONG_PASSWORD';
```

CockroachDB 管理控制台要求登录用户拥有管理员权限才能查看某些元素（例如，节点地图）。让我们预先授予该用户管理员权限：

```
GRANT ADMIN TO rob WITH ADMIN OPTION;
```



--join=localhost:26257,localhost:26258,localhost:26259 \

--enterprise-encryption=path=node2,key=keys/enc.key,old-key=plain

```
$ cockroach start \
--certs-dir=certs \
--store=node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--enterprise-encryption=path=node3,key=keys/enc.key,old-key=plain
```

现在，使用以下命令初始化集群：

```
$ cockroach init --certs-dir=certs --host=localhost:26257
```

![](img/index-146_1.jpg)

## 第六章 数据隐私

如果你打开 CockroachDB 管理控制台并访问以下 URL，你会看到每个节点都有一个使用我们生成的密钥加密的数据存储，如图 6-1: 所示。

• **节点 1** – `https://localhost:8080/#/reports/stores/1`

• **节点 2** – `https://localhost:8080/#/reports/stores/2`

• **节点 3** – `https://localhost:8080/#/reports/stores/3`

##### 图 6-1. 节点 1 的数据存储视图

#### 表加密

CockroachDB 还允许你加密特定的表，而不是加密整个集群。例如，假设你的数据库中有一些不包含敏感信息的巨大表。在这种情况下，这可能是一个不错的选择，因为你无需承担解密加密数据带来的性能损失。

创建支持表加密的集群与创建加密集群类似。但是，你需要对启动命令进行两处小的更改：

• `--store` – 此参数现在提供一组键值对，而不是一个简单的字符串，用于告诉 CockroachDB 将加密和未加密的数据存储在何处。在下面的例子中，我告诉 CockroachDB 加密 "node1/encrypted" 目录，而不加密 "node1/unencrypted" 目录。

• `--enterprise-encryption` – 此参数需要更新，仅指向加密数据目录。

以下命令创建三个启用了部分加密的节点。为了保持一致性，我将继续保护我的集群：

```
$ cockroach start \
--certs-dir=certs \
--store=path=node1/encrypted,attrs=encrypted \
--store=path=node1/unencrypted,attrs=unencrypted \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--enterprise-encryption=path=node1/encrypted,key=keys/enc.key,old-key=plain
```

```
$ cockroach start \
--certs-dir=certs \
--store=path=node2/encrypted,attrs=encrypted \
--store=path=node2/unencrypted,attrs=unencrypted \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--enterprise-encryption=path=node2/encrypted,key=keys/enc.key,old-key=plain
```

```
$ cockroach start \
--certs-dir=certs \
--store=path=node3/encrypted,attrs=encrypted \
--store=path=node3/unencrypted,attrs=unencrypted \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--enterprise-encryption=path=node3/encrypted,key=keys/enc.key,old-key=plain
```

现在，使用以下命令初始化集群：

```
$ cockroach init --certs-dir=certs --host=localhost:26257
```

## 第六章 数据隐私

集群初始化后，我们现在可以使用先前创建的根用户客户端证书连接到它：

```
$ cockroach sql --certs-dir=certs
``

是时候创建一些加密和未加密的表了。我们通过首先确定哪些表需要加密、哪些不需要加密来实现。在下面的例子中，我创建了一个包含个人身份信息（PII）的客户表，这个表应该被加密；以及一个不包含 PII 的产品表，这个表不需要被加密。

现在让我们创建客户表和产品表：

```
CREATE TABLE customer (
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
first_name string NOT NULL,
last_name string NOT NULL
);

ALTER TABLE customer CONFIGURE ZONE USING CONSTRAINTS = '[+encrypted]';

CREATE TABLE product (
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
name string NOT NULL
);
```



