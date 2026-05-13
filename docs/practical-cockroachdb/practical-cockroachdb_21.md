# 第 7 章 部署拓扑

如果数据未被固定且来自多个区域，你将产生区域间延迟。

### 写入

★★

在写入语句返回之前，数据会在各区域间复制。

### 延迟

#### 弹性

★★★★★ 如果你在各区域间复制数据，你将能抵御区域故障。如果你将数据固定到特定区域，并在该区域内的不同可用区部署节点，你将能抵御可用区故障。

#### 部署便捷度

★★★

设置一个全局表集群与设置一个区域表集群类似。

要启用全局表拓扑模式，我们只需更新表的配置使其变为全局表。在此示例中，我将把我们的 `example_regional_table` 更新为全局表：

```sql
ALTER TABLE example_regional_table SET LOCALITY GLOBAL;
```

你可能会注意到副本数量已从三变为五。这是全局表的默认副本数量，它允许 CockroachDB 将表扩展到现有 `us-central1` 区域之外：

```sql
SELECT lease_holder_locality, replicas, replica_localities
FROM [SHOW RANGES FROM TABLE example_regional_table];
```

```
region=us-east1,zone=us-east1a
{1,2,3,5,9}
{
  "region=us-east1,zone=us-east1a",
  "region=us-east1,zone=us-east1b",
  "region=us-east1,zone=us-east1c",
  "region=us-west1,zone=us-west1b",
  "region=us-central1,zone=us-central1b"
}
```

### 跟随者读取

跟随者读取拓扑非常适合读多写少的场景，其中可以容忍读取过时数据以及写入延迟。这种模式所提供的过期容忍度降低了对相同数据的读写争用，因为可以请求稍不同时间段的数据。读取过期的程度取决于你的要求，可以是以下之一：

• **精确过期** – 数据过期量是精确的（至少 4.8 秒），因此读取比有界过期读取更快。
• **有界过期** – CockroachDB 动态确定数据过期量，从而最小化过期。在发生网络分区的情况下，其可用性高于精确过期，因为读取可以从本地副本提供，而非必须从租赁者提供。

让我们看看使用此拓扑的集群的一些优缺点：

#### 读取延迟
★★★★★ 读取速度非常快，因为延迟优先于一致性。

#### 写入延迟
★★

在写入语句返回之前，数据会在各区域间复制。

#### 弹性
★★★★★ 集群可以容忍可用区和区域的故障。

#### 部署便捷度
★★★

跟随者读取的配置由客户端在读取时提供，因此此拓扑并不比其他任何多区域集群更困难。

要启用跟随者读取拓扑，我们只需执行允许数据过期的 `SELECT` 查询。以下是对 `example_regional_rows` 表进行跟随者读取的查询：

```sql
SELECT * FROM example_regional_rows
  AS OF SYSTEM TIME follower_read_timestamp();
```

```
  id                                   | city       | crdb_region
---------------------------------------+------------+--------------
  7b65d3fa-a0e1-4cd6-b125-05654a6fbce8 | Portland   | us-west1
  d39ec728-cfc0-47e8-914c-fff493064071 | Des Moines | us-central1
  e6a0c79c-2026-4877-b24f-6d4d9dd86848 | Charleston | us-east1
```

也可以使用以下语法在事务中启用跟随者读取：

```sql
BEGIN;
SET TRANSACTION AS OF SYSTEM TIME follower_read_timestamp();
SELECT * FROM example_regional_rows;
COMMIT;
```

使用 `follower_read_timestamp()` 函数不仅方便，而且返回的时间戳增加了查询从本地区域返回数据的可能性。

### 跟随工作负载

跟随工作负载拓扑是未指定表局部性（例如，按行分区的区域表）的集群的默认拓扑。对于当前最活跃区域的读取延迟必须很低的场景，它们是一个不错的选择。

在使用跟随工作负载拓扑的集群中，CockroachDB 将移动数据以确保最繁忙的读取区域获得所需的数据。



让我们来看看采用这种拓扑结构的集群的一些优点和缺点：

#### 读取延迟 ★★★★
对于当前最繁忙区域内的数据读取会很快，而来自该区域外的读取则会较慢。

#### 写入延迟 ★★
数据在写入语句返回前会在各区域间进行复制。

#### 弹性 ★★★★★
集群可以容忍可用区（AZ）和区域的故障。

#### 设置简易度 ★★★
此拓扑是多区域集群的默认拓扑，因此不比任何其他多区域集群更困难。

由于它是默认拓扑，我们无需做任何额外操作来显式启用它。

### 反模式
考虑反模式以及设计不良的集群对性能的影响，与考虑推荐的拓扑模式同样关键。在本节中，我们将使用 `demo` 命令创建一个集群，它*看起来*设计良好，但性能特性会很差。

## 第 7 章 部署拓扑
首先，我们将启动集群。该集群将是一个九节点的多区域部署，节点位于 `us-west1`、`us-east1` 和 `europe-west1`。

```
cockroach demo --nodes 9 --no-example-database
```

接下来，我们将创建一些数据库对象。以下语句将：
-   创建一个名为 `antipatterns_database` 的数据库，包含三个区域，反映由 `demo` 命令创建的区域。
-   创建一个区域性表，其行将在区域间进行地理分布。
-   为每个区域插入数据。
-   更新表的区域配置，以确保数据保留在这些区域内。

```
CREATE DATABASE antipattern_database;
USE antipattern_database;
ALTER DATABASE "antipattern_database" PRIMARY REGION "us-east1";
ALTER DATABASE "antipattern_database" ADD REGION "us-west1";
ALTER DATABASE "antipattern_database" ADD REGION "europe-west1";
ALTER DATABASE "antipattern_database" SURVIVE ZONE FAILURE;
CREATE TABLE antipattern_table (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    city TEXT NOT NULL,
    crdb_region crdb_internal_region NOT NULL AS (
        CASE
            WHEN city IN ('New York', 'Boston', 'Chicago') THEN 'us-east1'
            WHEN city IN ('San Francisco', 'Portland', 'Seattle') THEN 'us-west1'
            WHEN city IN ('Berlin', 'Paris', 'Madrid') THEN 'europe-west1'
            ELSE 'europe-west1'
        END
    ) STORED
);
ALTER TABLE antipattern_table SET LOCALITY REGIONAL BY ROW;
INSERT into antipattern_table (city)
VALUES
    ('New York'), ('Boston'), ('Chicago'),
    ('San Francisco'), ('Portland'), ('Seattle'),
    ('Berlin'), ('Paris'), ('Madrid');
SET override_multi_region_zone_config = true;
ALTER TABLE antipattern_table CONFIGURE ZONE USING num_voters = 3;
ALTER TABLE antipattern_table CONFIGURE ZONE USING num_replicas = 3;
```

我们通过这些命令本质上创建了一个全局分布的、跟随工作负载的集群。如果在此示例中最活跃的区域是欧洲，那么对 `antipattern_table` 的查询将不得不从美国获取数据，反之亦然。

稍等片刻后，运行以下语句以确保数据已均匀分布在各节点之间：

```
SELECT lease_holder_locality, replicas, replica_localities
FROM [SHOW RANGES FROM TABLE antipattern_table]
WHERE "start_key" NOT LIKE '%Prefix%';
```

```
region=europe-west1,az=c
{7,8,9}
{
  "region=europe-west1,az=b",
  "region=europe-west1,az=c",
  "region=europe-west1,az=d"
}
region=us-east1,az=d
{1,2,3}
{
  "region=us-east1,az=b",
  "region=us-east1,az=c",
  "region=us-east1,az=d"
}
```

## 第 7 章 部署拓扑
```
region=us-west1,az=b
{4,5,6}
{
  "region=us-west1,az=a",
  "region=us-west1,az=b",
  "region=us-west1,az=c"
}
```

现在，让我们执行一个查询来检查延迟：

```
SELECT * FROM antipattern_table;
```

```
id | city | crdb_region
---------------------------------------+---------------+---------------
3c24dee9-f006-4c2a-b556-0d7d6d0a7f6d | Portland | us-west1
8aaab90d-2ff3-471b-ac5a-39eec13b97bd | San Francisco | us-west1
9489216e-95f2-4796-98bd-ba654f85b688 | Seattle | us-west1
421f2184-99d7-4d2a-823c-e330ccbf5a87 | Madrid | europe-west1
48b68338-93b2-4761-832b-4a3a91ba77e5 | Berlin | europe-west1
```



731b6876-d745-47f3-822e-d2fe2867d639 | 巴黎 | europe-west1
a1542138-f0ff-41ba-bfa7-1da3a5226103 | 纽约 | us-east1
c66acd43-e23f-47ce-9965-6b09d6c4e502 | 芝加哥 | us-east1
c73e23a1-65c1-4063-8f93-51094e0fe475 | 波士顿 | us-east1
（共 9 行）
`时间：总计 14ms（执行 14ms / 网络 0ms）`

如果你认为 CockroachDB 从美国东西海岸和西欧收集数据只需 `14ms` 令人印象深刻，那你是对的！不幸的是，这凸显了此数据库配置中的一个重要疏忽。单独运行 `demo` 命令并不会模拟区域间延迟。我们需要添加 `--global` 标志来启用此功能。

这次，我们将按如下方式运行 `demo` 命令：
```
$ cockroach demo --global --nodes 9 --no-example-database
```

### 第 7 章 部署拓扑

如果你这次针对数据库运行相同的 SQL 命令，一旦数据复制到配置的区域中，你会看到不同的结果：
```
SELECT * FROM antipattern_table;

id | city | crdb_region

*---------------------------------------+---------------+---------------*

8c9600f7-81b2-40df-8d3c-9e00404bce7f | 巴黎 | europe-west1
8d21288b-12f7-438d-8ef1-d51e7d3335be | 马德里 | europe-west1
de45d851-f25b-4c4b-90aa-4193c469c82c | 柏林 | europe-west1
0c04c9dd-4752-4489-a71f-3f13edae8923 | 波士顿 | us-east1
87f4043b-d611-4129-a50e-dc941788110f | 芝加哥 | us-east1
d6ed2ab1-93d4-4306-8ef7-ec2bc00027f8 | 纽约 | us-east1
948d90e2-e382-4e82-92fc-a57679ebf577 | 西雅图 | us-west1
bc21e9e4-5929-47fa-a2e9-b7ff7cc29bec | 旧金山 | us-west1
c41f5a0c-f854-4815-ac3d-62eb3efe4147 | 波特兰 | us-west1

（共 9 行）
`时间：总计 929ms（执行 928ms / 网络 0ms）`
```

由于缓存，后续查询会执行得更快，但我们集群拓扑的设计目前并非最优。有多种方式可以改进：

- `视图（Views）` – 如果你的部分用户需要访问实时全局数据，可以考虑将某些地理位置敏感的数据置于视图之后，并让区域性用户使用特定区域的数据库用户连接到其本地集群。该视图将仅暴露最接近用户的数据，这意味着不会进行全局分布式查询。我们在第 5 章的“限制表数据”小节中正是这样做的。
- `独立集群（Separate clusters）` – 如果你需要符合规定的、全球范围的覆盖，但不一定是全局数据（例如，一个全球可查询的数据库），你可以创建仅跨越彼此接近区域（例如，`us-west1`、`us-central1` 和 `us-east1`）的独立集群。这样，你将实现数据隐私合规性和区域韧性。如果你需要访问所有全局数据，但不一定需要实时访问，你可以配置变更数据捕获（CDC）来异步更新集中存储中的数据。正如我们在第 5 章的“监视数据库变更”小节中所做的那样。

### 小结

你需要仔细设计你的生产集群以发挥其最佳性能。以下要点将帮助你掌握基础：

- `读取延迟（Read latencies）` – 在大多数拓扑中，读取延迟表现良好。在设计集群时，请考虑将数据固定到特定区域的影响，除非你正在使用视图之类的东西将数据锁定在这些区域中。对地理分区数据库中所有数据的查询将导致跨区域读取并增加延迟。
- `写入延迟（Write latencies）` – 不同拓扑之间的写入延迟差异比读取延迟更大。使用跨区域复制时，需要考虑延迟影响。如果你需要区域韧性，请考虑复制到一组本地区域（例如，将 `europe-west1` 数据复制到 `europe-north1` 和 `europe-central1`，而不是 `australia-southeast2` 和 `us-west1`）。
- `测试（Testing）` – 测试集群及任何应用的拓扑模式的最佳方法是启动一个真实集群并监控其性能。如果无法实现，你可以使用 `--demo` 命令或通过 Docker 和 HAProxy 容器来模拟全局分布式集群。

## 第 8 章 测试

在本章中，我们将探讨一些测试数据库的技术，无论它是运行在你的本地机器、CI/CD 流水线中、准备投入生产还是已经在生产环境中。

在应用程序测试中，很容易忽视数据库。开发阶段容易被忽视的测试领域有很多。我将与你分享一个真实案例，以强调在涉及数据库时制定测试策略的重要性。

在开发我的第一个数据库支持的应用程序时，我创建了一个包含我认为我的应用程序需要支持的所有行组合的数据集。每当我了解到一个新需求，我都会更新数据集，一切似乎都按预期工作。直到我部署到生产环境并看到生产级别的数据量时，情况才发生变化。我的应用程序变得迟钝，并且在部署后的几天里越来越慢。

在这个例子中，我忽略了在我的数据库中测试一件重要的事情：索引。

通过我的小（但看似全面的）数据集，我产生了一种虚假的信心，认为我的应用程序和数据库已经为生产环境做好了准备。当它面对生产级别的数据量时，缺失的索引露出了狰狞的面目，我的应用程序运行速度稳步下降。

在你的应用程序和数据库接近用户之前，有许多事情你需要让它们准备好。为了确保全面的测试覆盖，你需要考虑三种高层次的数据库测试类型：

- `结构测试（Structural testing）` – 专注于构成数据库的所有元素，包括其模式、表、列和视图。
- `功能测试（Functional testing）` – 专注于数据库的功能。在此测试阶段，你会提出诸如“针对其表的 CRUD 操作如何影响数据的最终状态？”这样的问题。
- `非功能测试（Nonfunctional testing）` – 专注于其他所有方面（性能、安全性等）。

在开始测试之前，让我们创建一个基本（且非常刻意的）数据库，并一起完成一些测试场景来验证其正确性。首先，我们将创建一个本地集群，以大致镜像一个三节点的基本生产拓扑集群：
```
$ mkdir certs
$ mkdir keys
$ cockroach cert create-ca \
--certs-dir=certs \
--ca-key=keys/ca.key

$ cockroach cert create-node \
--certs-dir=certs \
--ca-key=keys/ca.key \
localhost

$ cockroach start \
--certs-dir=certs \
--store=node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--locality=region=us-east1,zone=a \
--join=localhost:26257,localhost:26258,localhost:26259

$ cockroach start \
--certs-dir=certs \
--store=node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--locality=region=us-east1,zone=b \
--join=localhost:26257,localhost:26258,localhost:26259

$ cockroach start \
--certs-dir=certs \
--store=node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--locality=region=us-east1,zone=c \
--join=localhost:26257,localhost:26258,localhost:26259

$ cockroach init \
--certs-dir=certs \
--host=localhost:26257

$ cockroach cert create-client \
root \
--certs-dir=certs \
--ca-key=keys/ca.key
```

### 结构测试



