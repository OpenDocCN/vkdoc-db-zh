# 指示此节点的机架和数据中心
dc=DC1
rack=RAC1
```

当你配置 `GossipingPropertyFileSnitch` 时，如果存在 `cassandra-rackdc.properties` 文件，它总是会加载它。

以下三种探嗅器将查找 `cassandra-rackdc.properties` 文件来确定集群中节点所属的数据中心和机架：

*   `GossipingPropertyFileSnitch`
*   `Ec2Snitch`
*   `Ec2MultiRegionSnitch`

## 总结

理解 Cassandra 如何存储数据以及如何执行读写操作对于调整数据库性能至关重要。

修复 Cassandra 节点以纠正数据不一致是一项基本任务，掌握 Cassandra 提供的各种手动修复类型是很有好处的。

理解如何配置 `cassandra-topology.properties` 和 `cassandra-rackdc.properties` 文件有助于配置数据中心和集群。


# 6. Cassandra 查询语言简介

本章对 Cassandra 查询语言（CQL）进行了快速总结。

理解如何执行各种类型的 DML（数据操作语言）操作至关重要，例如选择、插入和更新数据，您将学习到 Cassandra DML 操作的复杂之处。理解 DDL 操作（如创建和删除对象）也很重要，本章将解释如何执行各种类型的 DDL 操作。

本章将展示如何创建键空间和表等结构。您将学习如何创建主键，并研究二级索引在 Cassandra 中的作用。物化视图是二级索引的更佳替代方案，您将学习如何使用它们。

我回顾了各种 CQL 数据类型，包括高级数据类型，如集合、类型和用户定义类型（UDT）。

本章还解释了用户定义函数（UDF）和用户定义聚合（UDA）。

### 使用键空间

Cassandra 使用键空间（keyspace）这种逻辑实体来分组表。键空间有点类似于关系数据库系统中使用的命名数据库。然而，键空间的真正目的是充当命名空间，用于指定 Cassandra 如何复制数据。这意味着，如果您的数据集在复制要求上有所不同，您可以使用不同的键空间来存储数据。

键空间是一个逻辑结构，Cassandra 不仅在其中存储表数据，还存储您为应用程序创建的所有其他实体，例如物化视图、函数、聚合和 UDT。

注意
您可以在每个键空间的基础上控制 Cassandra 的数据复制。

在以下部分中，我将展示如何管理键空间。

### 管理键空间

Cassandra 将其数据存储在表中，并将表分组到键空间中。键空间是一个逻辑容器，您可以为其定义适用于该键空间中所有表的选项。

注意
Cassandra 建议您为每个应用程序使用单个键空间。

当您引用一个表时，必须通过提供该表所在的键空间来完全限定它。如果表属于当前键空间，则无需提供键空间名称。您很快会学到，可以通过执行 `USE` 语句来指定当前键空间。

### 创建键空间

在创建键空间之前，您必须登录 `cqlsh`。登录后，运行 `describe cluster` 命令以查看集群的详细信息。

```
$ cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.10 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> describe cluster;
Cluster: Test Cluster
Partitioner: Murmur3Partitioner
cqlsh>
```

您使用 `CREATE KEYSPACE` SQL 语句创建键空间。该语句的语法如下：

```
CREATE  KEYSPACE [IF NOT EXISTS] keyspace_name
WITH REPLICATION = {replication_map}
[AND DURABLE_WRITES =  true|false] ;
```

创建键空间时可以指定两个选项：第一个是 `replication`，第二个是 `durable_writes`。这两个选项的含义如下：

*   `replication`：这是一个强制选项，用于指定副本放置策略和副本数量。`replication_map` 属性允许您指定数据中心中数据的副本数量。`replication` 选项必须包含子选项 `'class'`，用于指定复制策略。复制策略告诉 Cassandra 应在此键空间的数据中心和机架中存储数据副本的位置。有两种可能的复制策略：
    *   `SimpleStrategy`：顾名思义，它通过为集群内所有数据中心设置相同的集群范围复制因子来使用简单的复制策略。示例如下：

```
        {'class': 'SimpleStrategy', 'replication_factor' : 3};
        ```

如果您运行的是单节点测试集群，复制因子必须为 1。`SimpleStrategy` 仅适用于开发和概念验证用途。出于生产目的，必须使用 `NetworkTopologyStrategy`。提示 切勿使用复制因子 1 来存储您无法承受丢失的数据！
    *   `NetworkTopologyStrategy`：允许您通过为每个数据中心单独设置复制因子来指定更复杂的复制策略。示例：

```
        {'class': 'NetworkTopologyStrategy', 'DC1' : 1, 'DC2' : 3}
        ```

要使用 `NetworkTopologyStrategy`，您必须在 snitch 属性文件中指定数据中心名称，或使用名为 `datacenter1` 的单一数据中心。`replication` 选项的第二部分允许您通过为 `replication_factor` 属性配置值来设置此键空间中数据的副本数量。
*   `durable_writes`：告诉 Cassandra 是否应对当前键空间中的任何更新使用提交日志。默认值为 `true`，在生产数据库中应使用此值。通过为 `durable_writes` 指定 `false` 作为值，可以绕过首先将更改写入提交日志。这将加快对此键空间中表的写入速度，但如果在数据库将内存表刷新到 SSTable 之前节点发生故障，您将面临数据丢失的风险。

警告
当您为复制配置 `SimpleStrategy` 时，请勿禁用 `durable_writes`。

以下示例展示了如何创建名为 `cycling` 的键空间：

```
cqlsh> CREATE KEYSPACE IF NOT EXISTS cycling
WITH REPLICATION = { 'class' : 'NetworkTopologyStrategy',
'datacenter1' : 3 };
cqlsh>
```

如果没有错误，您可以确定数据库已创建所需的键空间。您可以运行 `describe keyspaces` 命令来确认数据库确实创建了该键空间。

```
cqlsh> describe keyspaces;
cycling  system_schema  system_auth  system  system_distributed  system_traces
cqlsh>
```

如您所见，`CREATE KEYSPACE` 语句中的 `IF NOT EXISTS` 部分是可选的，有助于避免在键空间 `cycling` 已存在时发生错误。

注意
键空间是定义复制的容器。

您可以在数据库显示的键空间列表中看到新的键空间 `cycling`。

接下来，您可以执行 `describe cycling` 命令来查看此键空间的详细信息。

```
cqlsh> describe cycling;
CREATE KEYSPACE cycling WITH replication = {'class': 'NetworkTopologyStrategy', 'datacenter1': '3'}  AND durable_writes = true;
cqlsh>
```

Cassandra 将所有 `nodetool` 和 `cqlsh` 命令的历史记录存储在 `$CASSANDRA_USER/.cassandra` 目录中的单独文件中，用户是启动集群的用户。

```
$ ls -altr
drwx------ 6 root root 4096 May 26 07:25 ..
-rw-r--r-- 1 root root  127 Jun 23 10:13 nodetool.history
-rw------- 1 root root 1308 Jun 24 07:24 cqlsh_history
drwxr-xr-x 2 root 4096 Jun 24 07:24 .
$
```

### 在具有多个数据中心的集群中创建键空间

您可以创建一个键空间，为每个数据中心设置不同的复制因子，如下例所示：

```
cqlsh> CREATE KEYSPACE "Cycling"
WITH REPLICATION = {
'class' : 'NetworkTopologyStrategy',
'datacenter1'  : 3 ,
'datacenter2 ',  2 ,
'datacenter3' :, 1 ,
};
```

注意
表、函数和 UDT 等对象绑定到特定的键空间。要使用这些对象，您必须在每次访问对象时指定键空间名称，或者将该键空间设为您的当前键空间。

#### 修改键空间

有时您可能希望修改键空间，这意味着您想更改创建键空间时可以设置的两个属性之一。这两个属性（我在“创建键空间”部分已描述）是 `replication` 和 `durable_writes`。

#### 修改复制因子

如果您想更改键空间的复制因子，可以通过执行 `ALTER KEYSPACE` 命令来实现，其语法如下：

```
ALTER KEYSPACE "KeySpace Name"
WITH replication = {'class': 'Strategy name', 'replication_factor' : 'No.Of replicas'};
```

**注意**

您在创建键空间时或在之后通过修改键空间来设置复制策略。

下面是一个示例，展示了如何通过修改名为 `cycling` 的键空间，将复制因子从 1 更改为 3：

```
cqlsh> ALTER KEYSPACE cycling
WITH replication = {'class':'SimpleStrategy',
'replication_factor': 3};
```

**注意**

您不能更改键空间的名称。

如果您配置了 `NetworkTopologyStrategy`，则需要做更多工作。由于复制因子的更改会影响键空间将要复制到或不再复制到的所有节点，您还必须让受影响的节点为复制因子的更改做好准备。以下是这种情况下的操作步骤：

1.  更改所有数据中心的复制级别。

    ```
    cqlsh> ALTER KEYSPACE cycling WITH REPLICATION =
    {'class' : 'NetworkTopologyStrategy', 'dc1' : 3, 'dc2' :2};
    ```

2.  如下运行 `nodetool repair` 命令，以对键空间执行完全修复：

    ```
    $ nodetool repair -full
    ```

在一个节点上完成修复后，开始下一个节点的修复。

### 需要运行 nodetool repair 命令的原因

当您提高复制因子时，更改不会自动在所有节点上生效。提高集群的复制因子后，您必须在该集群的所有节点上运行 `nodetool repair` 命令。同样，当您提高数据中心的复制因子时，也必须在该数据中心的所有节点上运行 `nodetool repair` 命令。

运行 `nodetool repair` 命令会告知 Cassandra 创建所需的额外副本，以满足您配置的复制因子。

**提示**

集群的写入吞吐量与复制因子成反比。

如果您降低了复制因子，之后必须运行 `nodetool cleanup` 命令，无论更改是在数据中心内还是在集群中。这将使 Cassandra 能够释放不再需要的副本所占用的空间。

### 防止键空间向某些数据中心发送副本

有时您可能希望阻止键空间仅在某些数据中心写入副本，或者允许键空间仅写入一个特定的数据中心。通过将复制因子设置为零 (`0`)，您可以阻止键空间向特定数据中心发送副本。示例如下：

```
cqlsh> ALTER KEYSPACE cycling
WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'dc1' :3,,
'dc2': 0, 'dc3': 3 };
```

此命令排除了数据中心 `DC2`（通过将其复制因子设置为 `0`）。

#### 修改持久写入属性

`ALTER KEYSPACE` 命令还允许您更改 `durable_writes` 属性。以下示例展示了如何更改键空间的 `durable_writes` 属性：

```
cqlsh> ALTER KEYSPACE cycling
WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'datacenter1'
: 3}
AND DURABLE_WRITES = true;
```

您可以通过运行以下命令来确认所做的更改已生效。

```
cqlsh> SELECT * FROM system.schema_keyspaces;
keyspace_name | durable_writes | strategy_class | strategy_options
----------------+----------------+-------------------------------------------
cycling | True | org.apache.cassandra.locator.NetworkTopologyStrategy | {"datacenter1":"3"}
mykeyspace | True | org.apache.cassandra.locator.SimpleStrategy | "replication_factor":"4"}
...
```

### 修复键空间

当您添加一个数据中心时，请务必运行以下 `nodetool repair` 命令来对键空间执行完全修复：

```
$ nodetool repair –full cycling;
```

#### 指定要使用的键空间

通常，您在某个键空间中工作，但希望切换到不同的键空间，并将该新键空间设为当前工作键空间。您必须使用关键字 `USE`（还能是什么？）来使一个键空间成为当前的默认键空间。

在以下示例中，您希望名为 `myKeyspace` 的键空间成为默认键空间：

```
cqlsh> USE myKeySpace;
cqlsh:myKeySpace>
```

### 使用键空间限定符

当您的代码处理多个键空间时，反复发出 `USE <KEYSPACE>` 命令会变得很麻烦。在这种情况下，您可以简单地指定键空间限定符，而不必执行 `USE <KEYSPACE>` 命令。Cassandra 允许您在执行以下语句时指定键空间限定符：

*   `ALTER TABLE`
*   `CREATE TABLE`
*   `DELETE`
*   `INSERT`
*   `SELECT`
*   `TRUNCATE`
*   `UPDATE`

要指定键空间中的表，请先指定键空间的名称，后跟一个句点，然后是表名。以下示例展示了如何将数据插入名为 `race_winners` 的表中，该表位于 `cycling` 键空间中。

```
cqlsh> INSERT INTO cycling.race_winners (race_name, race_position, cyclist_name) VALUES (...);
```

### 移除键空间

移除键空间很简单。执行 `DROP KEYSPACE` 语句来移除键空间，如下所示：

```
cqlsh> DROP KEYSPACE cycling;
cqlsh>
```

如您所见，当您发出 `DROP KEYSPACE` 语句时，Cassandra 不会给出任何反馈；它确实删除了该键空间，您可以通过运行 `DESCRIBE KEYSPACES` 命令来验证该键空间已不存在。

```
cqlsh> DESCRIBE KEYSPACES;
system_traces  system_schema  system_auth  system  system_distributed
cqlsh>
```

当您移除一个键空间时，数据库会删除其所有组成实体，如表、聚合、类型和 UDF。如果您已在 `cassandra.yaml` 文件中设置了 `auto_snapshot` 属性，Cassandra 可以自动备份该键空间。

默认情况下，`auto_snapshot` 属性值为 `true`，这意味着当您删除一个键空间（或表）时，数据库会自动为该键空间（或表）创建一个快照。您也可以选择在删除键空间之前手动备份它，如第 9 章所述。

**注意**

DataStax 强烈建议您将 `auto_snapshot` 属性的默认值 `true` 保持不变。

### 系统键空间

Cassandra 使用一组系统键空间来存储有关集群配置以及存储在该键空间中的对象的详细信息。

`DESCRIBE KEYSPACES` 命令显示集群中的所有系统键空间。

```
cqlsh> DESCRIBE KEYSPACES;
system_traces system_schema  system_auth system  system_distributed
cqlsh>
```

**提示**

默认情况下，`system-auth` 键空间的复制因子设置为 `1`。DataStax 建议您将 `system-auth` 键空间的复制因子设置为每个数据中心的节点数。不过，您也可以将此键空间的复制因子设置为 `5` 或 `7`，以提供充足的冗余度。

以下是各种系统键空间中存储数据的简要描述：

*   `system`：此键空间包含一些表，这些表包含有关物化视图、提示、索引、对等节点、分区信息以及驱动程序使用的预处理语句的信息。
*   `system_schema`：此键空间包含有关表列和索引、UDF、触发器、用户定义类型和物化视图的信息。
*   `system_distributed`：此键空间只有一个名为 `repair_history` 的表，用于存储有关键空间修复活动的信息。

### 从系统表获取集群拓扑信息

你可以通过查询系统表 `PEERS` 来获取集群拓扑信息，例如对等节点的 IP 地址、数据中心和机架的名称以及令牌值，如下所示：

```
cqlsh> SELECT * FROM system.peers;
peer            | data_center | host_id                    | preferred_ip | rack  | release_version | rpc_address     | schema_version                       | tokens
192.168.159.129 |         dc1 | 632fe5cd-26fa-4b8b-9842-182be2f954e5 |         null | rack1 |            3.10 | 192.168.159.129 | 86afa796-d883
'-1470738432828591905', '-1533929660503050360', '-1588268933422992241', '-1790127126337716384', '-1795704774997895089', '-1933674708843479324', '-2001904010895819169', '-2137322678046758761', '-2315125955255423142', '-2349800311170714405', '-239001164349178402', '-2439785688421178687', '-
...
```

### 获取有关函数、聚合和用户类型的信息

在本章后续部分，你将学习 Cassandra 的用户定义实体，如 UDF、聚合和类型。一旦创建了用户定义的实体，就可以使用系统表来检索有关这些实体的信息。

要检索有关 UDF 的信息，请查询系统表 `SCHEMA_FUNCTIONS`。

```
cqlsh> SELECT * FROM system.schema_functions;
```

你可以通过查询 `SCHEMA_AGGREGATES` 表来获取用户定义聚合的详细信息。

```
cqlsh> SELECT * FROM system.schema_aggregates;
```

最后，你可以查询 `SCHEMA_USERTYPES` 表以查看数据库中所有用户定义的类型。

```
cqlsh> SELECT * FROM system.schema_usertypes;
```

现在你已经了解了如何使用键空间，是时候开始将它们用于其设计目的了：通过创建一个表，作为表和其他实体的逻辑存储库。

### 创建表

Cassandra CQL 表存储行并具有名称。创建表时，你需要定义行的列、一个用于标识每行的强制性主键，以及任何你选择的附加选项。

你必须执行 `CREATE TABLE` 语句来创建一个新表。`CREATE TABLE` 语句的核心语法如下：

```
create_table_statement ::=  CREATE TABLE [ IF NOT EXISTS ] table_name
'('
column_definition
( ',' column_definition )*
[ ',' PRIMARY KEY '(' primary_key ')' ]
')' [ WITH table_options ]
```

以下是一个典型的建表语句：

```
cqlsh> use cycling;
cqlsh:cycling> CREATE TABLE loads (
...     machine inet,
...     cpu int,
...     mtime timeuuid,
...     load float,
...     PRIMARY KEY ((machine, cpu), mtime)
...     ) WITH CLUSTERING ORDER BY (mtime DESC);
cqlsh:cycling>
```

在创建 `loads` 表之前，你运行了 `use cycling` 命令，以便数据库知道在这个键空间中创建新表。或者，你本可以指定 `CREATE TABLE cycling.loads`。这样也能让数据库知道你想将新表放置在哪个键空间中。

在建表语句中，以下是你需要重点关注的主要内容：
* 列定义
* 主键和分区键
* 聚集列
* 表选项（紧凑存储和聚集顺序）

在接下来的章节中，我将简要回顾构成建表语句的关键实体。

> **注意**
>
> 主键由两部分组成：第一列或列组是强制性的分区键，后跟一个或多个聚集列。

#### Column_definition

`column_definition` 子句由列名及其类型以及两个修饰符组成。
* **Static**：将列声明为静态列。静态列对于共享相同分区键的所有行具有相同的值（稍后解释）。当然，只有非主键列才能是静态的。
* **Primary key**：主键唯一标识一行，所有表都必须定义主键。我将在下一节解释主键和分区键。

#### 主键、分区键和聚集列

Cassandra 的主键概念与规范化关系数据库的主键概念有很大不同。表的主键指定了存储在该表中数据的位置和顺序。

定义主键时，它可以有两部分：一个分区键和一个聚集键。至少，它必须有一个分区键。这两部分含义如下：
* Cassandra 使用主键定义的第一部分，即分区键，来将表中的数据分布到集群的各个节点上。分区键决定哪个节点将存储表中的特定行。复合分区键可以拆分数据，以便将相关数据存储在不同的分区上。
* 数据库使用键定义的第二部分，称为聚集键或聚集列（或多个列），来对分区内数据进行排序。

在定义主键的分区键和聚集键时，可以指定多个列。生成的键是一个复合主键。这种设计背后的思想是轻松地将表数据分布到集群的各个节点上。它还允许更高的性能并促进故障转移。

> **提示**
>
> 一旦创建了表，就不能更改其主键。你必须创建一个新表并将数据插入到新表中。

选择正确的主键、分区键和聚集列是 Cassandra 数据建模的一个关键方面，选择会显著影响查询性能。

表必须有主键。你可以用表中的一个或多个列来指定主键，列的顺序很重要。Cassandra 主键的独特之处在于它比关系数据库中的典型主键包含更多元素。

如前所述，主键由两部分组成：分区键和聚集列。你将在接下来的章节中了解更多关于主键这两个组成部分的信息。

> **注意**
>
> 分区键将行分组到同一个副本集中。聚集列决定了行在副本中的存储方式。

### 分区键

分区键是主键中第一个也是必需的组件。它可以包含一个或多个列。下面是一个例子：

```
CREATE TABLE t (k text PRIMARY KEY)
```

这里，列 `k` 是分区键，没有聚集列。

表分区是一组分区键值相同的行。分区键可以有一个或多个列，如果是多列键，则一个分区内所有行的分区键列的值必须完全相同。

下面是一个说明分区键概念的例子：

```
CREATE TABLE t (
a int,
b int,
c int,
d int,
PRIMARY KEY ((a, b), c, d)
);
```

假设你对这个表执行以下简单查询：

```
SELECT * FROM t;
a | b | c | d
---+---+---+---
0 | 0 | 0 | 0    // 第 1 行
0 | 0 | 1 | 1    // 第 2 行
0 | 1 | 2 | 2    // 第 3 行
0 | 1 | 3 | 3    // 第 4 行
1 | 1 | 4 | 4    // 第 5 行
```

在这种情况下，请注意以下几点：
* 第 1 行和第 2 行共享同一个分区。
* 第 3 行和第 4 行共享同一个分区。
* 第 5 行位于不同的分区。

> **注意**
>
> 当没有聚集列时，主键与分区键相同。

以下是分区的关键属性：
* Cassandra 保证来自一个分区的所有行都存储在同一组副本节点中。因此，你应该仔细选择分区键，以便将经常一起查询的行放在同一个分区中。这意味着数据库在多个节点上搜索数据的工作量更少。

> **提示**
>
> 如果分区键包含太多数据，可能会导致创建“热点”。
>
* Cassandra 在单个分区内以原子性和隔离性执行所有更新操作。


# 使用简单主键创建表

当你使用简单主键定义表时，你需要指定单个列名作为分区键。因此，主键和分区键是同一个。如果你有一个包含大量值的列，由于 Cassandra 会将分区分布到多个节点上，因此插入和查询数据会变得很容易。

如果你的应用需要一个具有单一标识符的简单查找表，那么简单主键就是合适的选择。在下面的例子中，你指定 `id` 作为主键，这样通过提供自行车手的 ID 号就能轻松获取他们的姓名：

```
cqlsh> USE cycling;
cqlsh> CREATE TABLE cyclist_name ( id UUID PRIMARY KEY, lastname text, firstname text );
```

`id` 列采用 `UUID` 格式。`UUID`（通用唯一标识符）是一个 128 位的值，其位模式符合各种类型。在 `CQL` 中，`uuid` 类型是基于随机数的 4 型 `UUID`。一个 `UUID` 通常由以连字符分隔的十六进制数字序列表示。

将 `uuid` 类型用作代理键很常见，可以单独使用，也可以与其他值结合使用。你可以生成一个 4 型 `UUID` 值，并在 `INSERT` 或 `UPDATE` 语句中使用该值，如下所示：

```
cqlsh:cycling> insert into cyclist_name (id, firstname, lastname)
... values
... (uuid(), 'sam', 'alapati');
cqlsh:cycling> select * from cyclist_name;
id                                   | firstname | lastname
--------------------------------------+-----------+----------
3c4acfde-dc65-4c9f-b2ec-9047d677641a |       sam |  alapati
(1 rows)
cqlsh:cycling>
```

提示

生成 `UUID` 的另一种方法是使用内置的 `CQL` 函数 `NOW()`。例如，你可以按以下方式从当前时间生成 `UUID`：

```
cql> INSERT INTO "users" ("username, id, 'address" VALUES ('alapati', NOW(), '12345, Main St, Anytown');
```

使用 `NOW()` 函数可以为插入到 `USERS` 表中的每一行生成一个唯一的 `UUID`，而无需显式指定 `UUID` 常量。

你也可以在表定义的末尾指定主键，如下所示：

```
cqlsh> USE cycling;
cqlsh> CREATE TABLE cyclist_name ( id UUID, lastname text, firstname text, PRIMARY KEY (id) );
```

这里展示的两种情况，你都先指定了键空间 (`USE cycling`) 来设置当前键空间。你也可以使用以下表示法来标识键空间，而无需先执行 `USE` 语句：

```
cqlsh> CREATE TABLE cycling.cyclist_name ( id UUID, lastname text, firstname text, PRIMARY KEY (id) );
```

## 定义复合分区键

Cassandra 通过分区键将完整的数据行存储在节点上。你可以使用复合分区键将数据分布到多个分区（多个节点）上。如果一列的数据量很大，那么数据量太大，无法存储在单个分区中。

你可以为分区键指定多个列，以便将数据分段到多个存储桶中。复合分区键由两个或更多列组成。你不是将所有数据分组到单个分区中，而是将数据分组到一组更小的分区中。

注意

搜索特定行时，你需要指定整个主键；而从同一分区中选择多行时，只需指定分区键。

当你遇到由于对一个分区进行大量写入而导致向节点写入数据速度变慢时，复合分区键很有帮助。例如，如果你的应用正在写入大量数据，你可以使用四列分区键按年/月/日/小时将数据分解成块。

如果你没有使用二级索引，则在查询表时必须提供复合键的所有列。

以下示例展示了如何将表中的两列指定为主键中的复合分区键：

```
cqlsh> CREATE TABLE cycling.rank_by_year_and_name (
race_year int,
race_name text,
cyclist_name text,
rank int,
PRIMARY KEY ((race_year, race_name), rank)
);
```

这里的主键包含一个分区键和一个聚簇列。分区键是复合的，因为你对分区键的两列使用了双括号 `(race_year, race_name)`。主键还有一个聚簇列 `rank`。

你必须指定分区键的所有列才能从此表中检索数据，因为你没有在表上创建二级索引。要获取完成不同比赛的自行车手的排名，你必须同时提供年份和比赛名称的值，如下所示：

```
cqlsh:cycling> SELECT * FROM cycling.rank_by_year_and_name WHERE race_year=2015 AND race_name='Tour of Japan - Stage 4 - Minami > Shinshu';
race_year | race_name                                 |rank|    cyclist_name
-----------+------------------------------------------+----+----------------
2015 | Tour of Japan - Stage 4 - Minami > Shinshu | 1  | Benjamin PRADES
2015 | Tour of Japan - Stage 4 - Minami > Shinshu | 2  |    Adam PHELAN
2015 | Tour of Japan - Stage 4 - Minami > Shinshu | 3  |   Thomas LEBAS
(3 rows)
cqlsh:cycling>
```

如果你只指定了部分分区键，如下例所示，`Cassandra` 会报错：

```
cqlsh:cycling> SELECT * FROM cycling.rank_by_year_and_name WHERE race_year=2016;
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
```

该错误是因为你只使用了部分分区键 (`race_year`)，而省略了分区键中的第二列 `race_name` 列。

你可以按照 `Cassandra` 的要求，在仅使用部分分区键的查询中添加 `ALLOW FILTERING` 子句来绕过此错误。

```
cqlsh:cycling> SELECT * FROM cycling.rank_by_year_and_name WHERE race_year=2015 ALLOW FILTERING;
race_year | race_name                                  | rank | cyclist_name
-----------+-------------------------------------------+---+------------
2015 |   Giro d'Italia - Stage 11 - Forli > Imola | 1 | Ilnur ZAKARIN
2015 |   Giro d'Italia - Stage 11 - Forli > Imola | 2 | Carlos BETANCUR
...
(5 rows)
```

在查询中添加 `ALLOW FILTERING` 子句可能会带来性能损失。影响取决于一个分区中有多少行，以及这些行是否分布在多个 `SSTable` 文件中。但是，`Cassandra` 保证查询将被限制在单个节点上。


## 复合键与聚类列

在创建主键时，聚类列是可选的，并跟在表的必需分区键之后。聚类列的顺序定义了表中分区的聚类顺序。对于每个分区，Cassandra 会根据你指定的聚类顺序对行进行物理排序。

聚类是 Cassandra 根据你定义的聚类列对每个分区内的数据进行排序的方式。默认情况下，Cassandra 按升序对列进行排序。

同时包含分区键和聚类列的主键称为复合主键。分区键可以是简单的或复合的。

下面是一个示例，展示了聚类列如何为分区定义聚类顺序：

```sql
CREATE TABLE t (
a int,
b int,
c int,
PRIMARY KEY (a, b)
);
```

插入一些行并运行以下 `SELECT` 语句：

```sql
SELECT * FROM t;
a | b | c
---+---+---
0 | 0 | 4     // row 1
0 | 1 | 9     // row 2
0 | 2 | 2     // row 3
0 | 3 | 3     // row 4
```

在这种情况下，主键定义指定了`(a, b)`，意味着列`a`是分区键，列`b`是聚类列。此外，你可以看到 Cassandra 内部将属于同一分区（列`a` =0）的行按照聚类列`b`的值顺序存储。

以这种方式聚类列意味着查询一个分区中的一系列行可以非常快速地返回。例如像 `SELECT * FROM t WHERE a=0 AND b>1 AND b<=3` 这样的查询。

**注意**

当你指定一个复合主键时，Cassandra 根据其分区键将整行存储在某个节点上，并使用聚类列对数据进行排序。

如果你的应用程序从一个大分区中检索数据，它通常需要读取整个分区才能获取少量数据。如果你通过指定聚类列对分区键的数据进行排序，数据库可以更高效地检索数据，因为它不必读取整个分区。

通过指定聚类列来分组数据类似于在关系数据库中连接表的方式，只是更好。

在下表中，列`category`充当分区键，列`points`充当聚类列。当你查询此表时，对于每个类别，数据库按降序对`points`列进行排序。

```sql
CREATE TABLE cycling.cyclist_category (
category text, points int, id UUID, lastname text,
PRIMARY KEY (category, points))
WITH CLUSTERING
ORDER BY (points DESC);
```

## 静态列

在创建表时，你可以将非聚类列声明为静态。静态列仅在特定分区内是静态的。

以下是展示如何声明静态列的示例：

```sql
CREATE TABLE t (
k text,
s text STATIC,
i int,
PRIMARY KEY (k, i)
);
INSERT INTO t (k, s, i) VALUES ('k', 'I''m shared', 0);
INSERT INTO t (k, s, i) VALUES ('k', 'I''m still shared', 1);
SELECT * FROM t;
Output is:
k |                  s | i
---+--------------------+---
k  | "I'm still shared" | 0
k  | "I'm still shared" | 1
```

关于静态列需要记住的事项：

*   你不能将作为主键的列声明为静态。
*   没有任何聚类列的表不能有静态列。

## 表选项

你可以使用`WITH`关键字设置两个关键的表选项：`COMPACT STORAGE`和`CLUSTERING ORDER`。

### 紧凑表

你可以使用`COMPACT STORAGE`选项定义紧凑表。此选项仅用于向后兼容，以处理 CQL 版本 3 之前创建的表定义。对于新表，你不得指定此选项，因此我不详细讨论此选项。

### 聚类顺序

`CLUSTERING ORDER`选项允许你将聚类顺序更改为使用列的逆自然顺序。默认情况下，排序意味着`ASC`（升序）。你可以指定`ASC`（默认顺序）或`DESC`（降序）顺序。

### 其他选项

除了我描述的这两个选项外，表还支持其他选项，例如允许你编写注释、指定读修复选项以及等待垃圾回收墓碑（删除标记）的时间。以下是表选项的简要描述：

*   `comment`：关于表的注释。
*   `read_repair_chance`：查询超过一致性级别所需节点的概率，用于执行读修复。默认值为 0.1。
*   `dclocal_read_repair_chance`：查询同一数据中心内超过一致性级别所需节点的概率，用于执行读修复。默认值为 0。
*   `gc_grace_seconds`：垃圾回收墓碑前的等待时间。默认值为 864000 秒。
*   `bloom_filter_fp_chance`：SSTable 布隆过滤器假阳性的目标概率。
*   `default_expiration_time`：表的生存时间（TTL），以秒为单位。

还有三个非常重要的、影响性能的表选项，与压缩、压缩和缓存相关。我将在第 11 章详细解释这三个选项，该章节讨论 Cassandra 性能调优。如果你愿意，可以提前阅读。

## 使用计数器列跟踪值

你可以使用一种特殊的列，称为计数器，来跟踪一个递增的值。例如，你可能希望创建一个计数器列来跟踪加入游戏的在线游戏玩家数量，或者页面浏览量或日志消息的数量。

你必须为计数器列指定`counter`数据类型。但是，你不能在任何表中将列指定为计数器列；你必须创建一个专门的表来保存计数器列。专门的表必须只包含一个主键（可以是复合的）和计数器列。你不能将计数器列用作主键；除主键列外的所有列都必须是计数器列。

**注意**

你不能索引计数器列。你也不能删除它。

你不能使用`INSERT`语句将数据加载到计数器列中；相反，你必须使用`UPDATE`语句来插入和修改计数器列的值。

以下示例展示了如何创建计数器列：

```sql
CREATE TABLE popular_count (
id UUID PRIMARY KEY,
popularity counter
);
```

## 修改表

你可以通过执行`ALTER TABLE`语句来修改表结构或其选项。`ALTER TABLE`语句允许你执行以下操作。

*   你可以更改列的类型，如下所示：

    ```sql
    ALTER TABLE customers ALTER lastKnownAddress TYPE UUID;
    ```

*   你可以向表中添加新列。

    ```sql
    ALTER TABLE customers ADD address varchar;
    ```

*   你还可以通过添加`WITH`指令来更改一些表选项。

    ```sql
    ALTER TABLE customers
    WITH comment = "complete customer records table"
    AND read_repair_chance = 0.4;
    ```

**注意**

一旦创建表，你就不能更改`COMPACT STORAGE`和`CLUSTERING ORDER`选项。

你只能按照允许的方式转换某些 CQL 数据类型。有关 CQL 类型兼容性，请参阅 CQL 文档。例如，你可以将现有的 ASCII 或 text 类型仅转换为 varchar 类型。

### 查看表的配置选项

如前所述，您在创建表时可以指定许多选项。您也可以通过修改表来调整若干选项。如何知道表在任意时刻具有哪些选项？运行 `describe table` 命令以查看表的所有属性。

在以下示例中，您创建表 `cyclist_name`，然后运行命令 `describe cyclist_name` 来查看表的属性：

```cqlsh> use cycling;
cqlsh:cycling> CREATE TABLE cyclist_name ( id UUID, lastname text, firstname text, PRIMARY KEY (id) );
cqlsh:cycling> describe cyclist_name;
CREATE TABLE cycling.cyclist_name (
id uuid PRIMARY KEY,
firstname text,
lastname text
) WITH bloom_filter_fp_chance = 0.01
AND caching = {'keys': 'ALL', 'rows_per_partition': 'NONE'}
AND comment = ''
AND compaction = {'class': 'org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy', 'max_threshold': '32', 'min_threshold': '4'}
AND compression = {'chunk_length_in_kb': '64', 'class': 'org.apache.cassandra.io.compress.LZ4Compressor'}
AND crc_check_chance = 1.0
AND dclocal_read_repair_chance = 0.1
AND default_time_to_live = 0
AND gc_grace_seconds = 864000
AND max_index_interval = 2048
AND memtable_flush_period_in_ms = 0
AND min_index_interval = 128
AND read_repair_chance = 0.0
AND speculative_retry = '99PERCENTILE';
cqlsh:cycling>
```

在这个表的众多属性中，您只设置了少数几个。然而，列表很长，因为对于所有您未指定的表选项，数据集使用了默认值。

### 删除和截断表

与任何关系型数据库一样，您可以删除和截断 Cassandra 表。

#### 删除表

您可以使用 `DROP TABLE` 语句来删除表。

```cqlsh> DROP TABLE cycling.cyclist_name;
cqlsh> DROP TABLE cycling.cyclist_name IF EXISTS;
```

在删除表之前，必须首先删除任何基于该表创建的物化视图。

#### 截断表

使用 `TRUNCATE` 语句截断表会保留表结构，但会删除表中的所有数据，包括任何基于该表的物化视图中的数据。

```cqlsh> TRUNCATE TABLE customers;
cqlsh> TRUNCATE customers;
```

这两种命令的工作方式相同。关键字 `TABLE` 是可选的，因为表是您可以截断的唯一对象。

在截断表之前，您需要执行以下操作：

1.  使用 `CONSISTENCY` 命令将一致性级别设置为 `ALL`。

    ```cqlsh> CONSISTENCY ALL;
    Consistency level set to ALL.
    cqlsh>
    ```

2.  运行 `nodetool status` 命令以确保所有节点均已启动并可用。

    ```cqlsh> nodetool status
    Datacenter: datacenter1
    =======================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --  Address    Load       Tokens    Owns (effective)  Host ID          Rack
    UN  127.0.0.1  159.91 KiB  256      100.0%           c51011d6-06da-47b8-bd55-70ac2e51716d  rack1
    $
    ```

### 从表中删除行和列

您可以使用 `DELETE` 语句从表中删除行。

```cqlsh:mydb> DELETE FROM cyclists where pk = 'salapati';
```

您可以通过以下方式发出 `DELETE` 语句来从表中删除行：

```cqlsh:mydb> DELETE session-token FROM users where pk = 'salapati';
```

此语句将仅从 `pk` 值为 `'salapati'` 的一行中删除 `session_token` 列。`WHERE` 子句指定了要从表中删除的行。

#### 删除多行

您可以通过指定关键字 `IN` 并在括号中提供逗号分隔的值列表来删除多行。

```cqlsh> DELETE FROM cycling.cyclist_name WHERE firstname IN ('Sam', 'James');
```

如果您希望从表中删除整列，可以通过省略 `WHERE` 子句来实现。

```cqlsh:mydb> DELETE session_token FROM users;
```

#### 从集合（Set）、列表（List）或映射（Map）中删除

要从作为单个列存储的映射中删除元素，您需要指定列名以及方括号中的元素键。

```cqlsh> DELETE sponsorship ['sponser_name'] FROM cycling.races WHERE race_name = 'Criterium du Dauphine';
```

要从列表中删除数据，请指定列名以及列表索引位置。

```cqlsh> DELETE categories[3] FROM cycling.cycling.history WHERE lastname ='TIRALONGO';
```

要从集合中删除所有元素，只需指定列名。

```cqlsh> DELETE sponsorship FROM cycling.races WHERE race_name = 'Criterium De Dauphine';
```

#### 使用时间戳（TIMESTAMP）删除旧数据

您可以使用时间戳来指定要删除的列。

```cqlsh> DELETE firstname, lastname
FROM cycling.cyclist_name
USING TIMESTAMP 1318452291034
WHERE lastname = 'ALAPATI';
```



## 从数据库中移除已删除的数据

Cassandra 不会在你执行 `DELETE` 语句后立即从表中移除行和列。相反，它会在删除操作之后的第一次合并过程中完全移除这些值。这样做是为了提高性能。

### Cassandra 如何使用墓碑标记来标记已删除的数据

Cassandra 会用墓碑标记来标记你删除的数据，并在一段宽限期后将其完全移除。Cassandra 将所有删除操作视为插入或更新插入（根据情况可能是更新或插入）。

当你删除数据时，Cassandra 会向作为 `DELETE` 命令一部分的分区添加一个删除标记。这个删除标记被称为墓碑标记，数据库会将这些标记写入一个或多个节点上的 SSTable。每个墓碑标记都有一个过期期限，之后数据库会在其常规的 SSTable 合并过程中删除该墓碑标记。

### 指定生存时间值

或者，你可以为记录（行或列）或表指定一个 `TTL` 值，以设置列中数据的可选过期期限（`counter` 类型数据除外）。这样做将使 Cassandra 在经过一定时间后删除列或表中的数据。在生存时间周期结束时，数据库会用墓碑标记标记该记录，并在合并过程中按照前面说明的方式处理它。

通过设置 `TTL` 来使数据过期并非没有代价。它会带来额外的开销，即在内存和磁盘上额外使用 8 个字节来记录 `TTL` 和过期时间。

**注意**

指定 `TTL` 是一种隐式删除数据的方式。你可以通过在表的列上设置 `TTL` 或为表指定 `default_time_to_live` 属性来实现。

你可以使用 `INSERT` 或 `UPDATE` 语句指定过期时间 (`TTL`)。以下示例展示了如何通过向 `INSERT` 命令添加 `USING TTL` 选项，指定 `users` 表中的 `password` 列在 1 天（86,400 秒）后过期：

```cql
cqlsh:mydb> INSERT INTO users
(user_name, password)
VALUES ('cbrown', 'ch@ngem4a') USING ttl 86400;
```

此后，你可以通过执行以下 `UPDATE` 语句将该期限增加到 2 天：

```cql
cqlsh:mydb> UPDATE users USING TTL 172800 SET password = 'ch@ngem4a'
WHERE user_name = 'cbrown';
```

你可以使用 `default_time_to_live` 属性为整张表设置默认 `TTL`，这是在创建表时可以指定的 CQL 表属性之一。此属性的默认值为 0，表示写入表中的数据永远不会过期。该值以秒为单位。一旦设置，Cassandra 会将 `TTL` 值应用于表中的每一列。当表的 `TTL` 超过时，Cassandra 将为整张表添加墓碑标记。

**注意**

通过将 `default_time_to_live` 表属性设置为零，你实际上移除了在该表中指定的任何列 `TTL`。

你可以在使用 `CREATE TABLE` 语句创建表时，或稍后使用 `ALTER TABLE` 语句修改表时，为表应用默认 `TTL`。

一旦列自创建以来经过的秒数超过了你配置的 `TTL` 值，Cassandra 就会认为数据已过期，尽管它仍会在查询结果中报告该数据。Cassandra 会在下一次读取时用墓碑标记（一种删除标记）标记过期数据，但会将数据最多保留 `gc_grace_seconds` 指定的时间。

一旦 `gc_grace_seconds` 属性指定的时间过去，Cassandra 会在其常规合并过程中自动移除那些已被标记为墓碑的数据。

## 垃圾回收如何工作

你在创建表时设置 `gc_grace_seconds` 属性。该属性指定了 Cassandra 用墓碑标记数据后，经过多少秒数据才有资格被垃圾回收。

`gc_grace_seconds` 属性的默认值很长（864,000 秒，即 10 天），因此在创建表时你可以保留此属性不变。默认值如此之长的原因是为了让 Cassandra 有充足的时间在永久删除数据之前，最大限度地保证数据的一致性。

虽然 `gc_grace_seconds` 属性的默认值是 10 天，但你可以为那些用户不会显式删除数据的表减少此值，例如仅包含已配置 `TTL` 的数据的表，或你已配置了 `default_time_to_live` 属性的表。

**提示**

你可以为每张表配置不同的墓碑标记宽限期。

在将 `gc_grace_seconds` 属性值从其默认设置 10 天降低之前，请记住以下几点：

*   Cassandra 不会重放比 `gc_grace_seconds` 设置更早的提示（这些提示是重放错过的写入操作的指令）。这意味着减少 `gc_grace_seconds` 属性值可能导致最近恢复的节点丢失一些写入操作。
*   Cassandra 还会执行批处理操作，这涉及按顺序重放数据库中的更改。Cassandra 会等待，直到批处理更改创建后 `gc_grace_seconds` 时间过去，才会重放该批处理更改。如果你减少了 `gc_grace_seconds` 属性的值，你将面临批处理写入可能恢复一些已删除数据的风险。



## 僵尸记录与节点修复

当你删除数据时，处理删除请求的节点会为该记录创建一个墓碑（tombstone），并将此墓碑传递给其他存储了该被删除记录副本的节点。如果其中一个副本节点恰好处于宕机状态，它将无法接收到墓碑，并继续存储数据被删除前的版本。

如果数据库在宕机节点恢复之前，已经删除了带有墓碑标记的记录，那么数据库会将该节点上（已删除的）记录视为新数据。随后，数据库会将此“新”数据发送给其他节点。这种在删除后重新出现在表中的已删除行，就被称为**僵尸**。

僵尸数据出现的原因是，一个节点在长时间宕机后，在数据库能够运行修复之前重新上线。如果该节点在重新上线前没有被修复，数据库会看到未被标记墓碑的数据，并将其作为新数据复制到其他节点。这就是为什么在恢复节点（使用 `nodetool repair` 命令）并允许其加入集群之前，你必须运行修复的原因。

注意

数据库不会在其宽限期内重放（插入或更新）对已标记墓碑记录的变更。

表属性 `gc_grace_seconds` 为墓碑设置了宽限期。这个宽限期有助于防止僵尸记录的出现，它让恢复中的节点有时间以正常方式处理墓碑。当你恢复宕机节点时，数据库会通过其**提示移交**（hinted handoff）功能，重放该节点在宕机期间错过的所有变更（插入和更新）。在记录的墓碑宽限期结束之前，数据库不会重放对该记录的删除变更。如果节点在宽限期结束时未能成功恢复上线，它将错过这些删除操作。

一旦墓碑的宽限期结束，数据库会在其正常的压缩（compaction）操作期间删除该墓碑。

你可以通过以下方式防止僵尸记录的出现：

*   节点一旦恢复，立即在该节点上运行 `nodetool repair` 命令。
*   每隔 `gc_grace_seconds` 时间，对每张表运行一次 `nodetool repair` 命令。

注意

`DROP KEYSPACE` 或 `DROP TABLE` 命令会立即删除数据。

请注意以下几点：

*   如果你选择为表中的所有记录指定生存时间（TTL），它们会自然过期，你不需要为该表运行 `nodetool repair` 命令。
*   你可以通过启动压缩过程（例如使用 `SizeTieredCompactionStrategy` 进行数据压缩）来立即删除所有墓碑。
*   当你在表级别设置了 `default_time_to_live` 属性时，一旦记录超过该表的 TTL，Cassandra 会直接移除该记录，而不会创建墓碑或等待压缩过程运行。

### Cassandra 数据库中的索引

表必须有一个主索引，你已经学习了如何创建和使用主索引。

在创建表时，二级索引不是必需的。二级索引使你能够对原本无法查询的列进行查询。

二级索引有助于在非主键列上过滤表数据。Cassandra 不允许你运行直接匹配非主键列的查询，因为这可能无法从表中检索出连续的数据块。

由于非主键不参与数据排序，Cassandra 将表的数据分散存储在位于不同节点上的多个分区中。这意味着，如果你查询某个非主键列的特定值，最终可能需要读取表的所有分区；因此，Cassandra 不允许此类访问。

你可以在某列上构建二级索引，但在使用此类索引时存在一些限制。如果你的查询除了包含二级索引列外，还指定了一个分区键，那么查询是可行的，因为数据库可以通过读取单个节点分区的数据来满足此查询。但是，如果你在查询中没有将二级索引限制在特定的分区键内，那么查询将需要从所有节点读取数据，而数据库将不允许此操作。当你尝试此查询时，会看到以下错误：

```
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
```

要使此类查询生效，你必须在查询中添加 `ALLOW FILTERING` 选项。

### 何时使用索引

当表中许多行都包含被索引的值时，使用索引是合适的。如果表中某列的唯一值过多，你会因此产生更多的查询和索引维护开销。

假设你有一个名为 `races` 的表，其中有数亿条关于多年来参加各种自行车比赛的自行车手的条目。你的目标是根据排名查找自行车手。由于许多自行车手在同一年参赛，他们的排名在比赛年份列上会共享相同的值。在这种情况下，`races` 表中的 `race_year` 列非常适合作为该表的索引。

注意

SSTable 附加二级索引（SASI）是二级索引的一种新实现，提供了性能改进。然而，截至本书撰写时（2017 年 10 月），SASI 索引仍处于实验阶段。

### 何时不应使用二级索引

索引并非总是福音。如果你不加区别地为所有表创建索引，它很容易变成诅咒。建议不要在以下情况下创建索引：

*   高基数和低基数的列
*   经常更新或删除的列
*   包含计数器列的表
*   在大型分区中搜索行时

在接下来的章节中，我将详细阐述在此处列出的情况下不建议使用索引的原因。

#### 高基数和低基数列

高基数列和非常低基数的列都不适合作为索引候选。高基数意味着列中有许多不同的值，而低基数意味着不同的值很少。

为具有大量不同值的列创建和使用索引是低效的。你最好将表本身维护为一种索引形式，而不是创建内置的 Cassandra 索引。

如果你的表很大但查询很少，你可以选择在高基数列上创建索引。只是不要在使用频繁的大表上这样做。

至于非常低基数的列，考虑布尔值列这种极端情况。在这种情况下，索引的每个值在索引中只对应一行，并且这些行非常大，因为每个被索引列只有两个可能的值。

#### 经常更新或删除的列

如前所述，Cassandra 使用墓碑来标记删除的数据。Cassandra 会在索引中存储墓碑，直到达到 10,000 个单元格的限制。一旦超过墓碑限制，任何涉及被索引列的查询都将失败。

#### 使用索引在大型分区中搜索行

如果你在大型表中查询被索引的列，通常需要从多个分区收集响应。随着你向集群中添加更多节点，查询响应会变得越来越慢。

避免此处描述情况的方法是，在查询被索引列时缩小搜索范围。

除了这里描述的所有潜在问题之外，由于数据库在集群的每个节点上都存储了索引表，如果查询访问多个节点，查询最终可能成为一个性能问题。

# Cassandra 二级索引与物化视图

### 创建二级索引

您可以通过执行 `CREATE INDEX` 语句在表上创建二级索引。假设您有如下表格，其主键由复合分区键 `(race_year, race_name)` 和一个聚类列 (`rank`) 组成：

```sql
cqlsh> CREATE TABLE cycling.rank_by_year_and_name (
    race_year int,
    race_name text,
    cyclist_name text,
    rank int,
    PRIMARY KEY ((race_year, race_name), rank)
);
```

您无法在 `race_year` 列上查询此表，因为该列仅是复合分区键的一部分。您可以在 `race_year` 列上创建一个二级索引，从而能够基于此列进行查询。创建索引的方法如下：

```sql
cqlsh> CREATE INDEX ON cycling.rank_by_year_and_name (race_year);
```

现在，您可以利用新建的二级索引发出以下查询：

```sql
cqlsh> SELECT * FROM cycling.rank_by_year_and_name WHERE race_year=2015;
```

在 `CREATE INDEX` 语句中，您没有为新索引指定名称。为索引命名是可选的；如果您不指定，数据库将为索引分配一个系统生成的名称。您也可以像下面这样指定自己的索引名称：

```sql
cqlsh> CREATE INDEX my_index1 ON cycling.rank_by_year_and_name (race_year);
```

您可以为表的任意列（包括聚类列）创建二级索引。您可以在一个表上创建多个二级索引。

您可以在已包含数据的表上创建索引，也可以在新建的空表上创建索引。

### 删除索引

与主索引不同，您可以删除二级索引。发出 `DROP INDEX` 语句以删除索引：

```sql
cqlsh> DROP INDEX myIndex1;
```

此语句会删除一个现有的二级索引。如果您希望避免因索引不存在而报错，可以使用以下语句代替：

```sql
cqlsh> DROP INDEX myIndex1 IF EXISTS;
```

### Cassandra 中的物化视图

如前所述，二级索引在处理高基数数据时表现不佳，因为对这些索引的查询会使数据库访问集群中的所有节点。物化视图是处理高基数数据的合适替代方案。

物化视图是从现有表使用新的主键构建而成的表。您可以在主键列中包含值为 null 的行。

**提示**

当您将物化视图用于查询低基数数据时，会阻碍性能。然而，请谨慎行事，因为物化视图仍需进一步完善才能完全发挥效用。在处理大量更新时，物化视图（MVs）导致集群停机的情况并非闻所未闻。

到 Cassandra 3.11 或 4.x 版本发布时，物化视图的可用性应该会大大提升。

### 创建物化视图

以下示例展示了如何以常规表为基础或源表来创建物化视图：

```sql
cqlsh> CREATE MATERIALIZED VIEW cycling.cyclist_by_age
AS SELECT age, name, country
FROM cycling.cyclist
WHERE age IS NOT NULL AND cid IS NOT NULL
PRIMARY KEY (age, cid);
```

在这个例子中：

*   `AS SELECT` 子句指定了您想从基表或目标表复制到新物化视图的列。这里是 `age`、`name` 和 `country`。
*   `FROM` 子句指向原始表或源表，物化视图的数据来源于此。
*   `WHERE` 子句确保主键列不为 null，因为这是创建物化视图的必要条件。
*   `PRIMARY KEY` 子句指定了主键列，本例中是 `age` 和 `cid`。`age` 列恰好也是源表的主键列，您必须将源表中所有此类主键列也包含在物化视图的主键中。

### 删除物化视图

您可以通过发出 `DROP MATERIALIZED VIEW` 命令来删除物化视图：

```sql
cqlsh> DROP MATERIALIZED VIEW cycling.cyclist_by_age;
```

您也可以通过截断源表来删除一个或多个物化视图，这将移除基于该表的所有物化视图。当您删除一个表时，必须首先删除所有以该表为源表的物化视图。

### 使用物化视图进行反规范化

在物化视图的示例中，您按 `age` 对物化视图进行了分区，因此您可以基于骑行者的年龄对其运行查询。物化视图通过创建多个数据相同但根据不同的主键以不同方式组织的表，来帮助对数据进行反规范化。

您可以在同一个表上创建多个物化视图。因此，您可以创建具有不同主键（如 `birthday` 或 `country`）的视图。这有助于您通过这些主键来组织数据，并让您能够按 `birthday` 和 `country` 查询这些物化视图。

### 更新物化视图

当您更新源表时，数据库会自动更新基于该表创建的任何物化视图，从源表删除数据时也是如此。然而，更新是异步进行的，因此物化视图会在父表更新之后延迟一段时间才得到更新。

Cassandra 需要执行额外的读取操作才能更新物化视图。此读取涉及所有副本的数据一致性检查，这意味着对物化视图的写入比普通表的写入要慢。

删除操作同样存在问题。由于数据库可能不会在物化视图中连续存储来自父表的相同行，因此从源表删除行通常可能需要数据库在相应的物化视图中创建多个墓碑。

### 使用 SELECT 语句查询数据

数据操作语言（DML）是使您能够查询、插入、更新和删除数据的 CQL 语句集合。查询数据是您在 Cassandra 数据库中最常执行的 DML 操作。因此，透彻理解如何在 CQL 中使用 `SELECT` 语句是一个好主意。

`SELECT` 语句返回与请求匹配的结果。每一行都包含作为 `SELECT` 语句一部分的选择项的值。

以下是 CQL 中 `SELECT` 语句的基本语法：

```sql
SELECT selectors | DISTINCT partition
FROM [keyspace_name.] table_name
[WHERE partition_conditions [ AND solr_query = 'search_expression' [LIMIT n] |
[solr_query = 'search_expression'  [LIMIT n]]
[AND clustering_conditions
[AND regular_column_conditions]]]
[GROUP BY column_name]
[ORDER BY PK_column_name ASC|DESC]
[LIMIT N | PER PARTITION LIMIT N]
[ALLOW FILTERING]
```

`SELECT` 语句包含以下关键子句：

*   选择子句
*   WHERE 子句
*   GROUP BY 子句
*   排序子句

以下各节描述了 `SELECT` 语句的关键子句。

### 选择子句

`选择子句` 决定了查询返回哪些列，以及数据库在返回结果之前必须对结果集应用的任何转换。`SELECT` 语句必须使用通配符选择器 (`*`) 或一个或多个选择器来定义列。

选择器可以是：

*   列名
*   一个术语
*   函数调用
*   对 `COUNT` 函数的特殊调用 `COUNT(*)`

以下是一些典型的 `SELECT` 语句：

```sql
cql> SELECT COUNT(*) FROM users;
cql> SELECT name, occupation FROM users;
cql> SELECT name, occupation FROM users WHERE userid IN (1111,1112,111130);
cql> SELECT JSON name, occupation FROM users WHERE userid = 999;
```

您可以通过在 `SELECT` 语句中指定 `AS` 子句来为顶级选择器起别名，如下所示：

```sql
cql> SELECT name AS customer_name FROM customers;
```

输出将是：

```
customer_name
------------
Alapati
...
```


### WHERE 子句

对于来自关系型数据库背景的我们来说，CQL `SELECT` 语句中的 `WHERE` 子句有一些令人意外之处。与其他数据库类似，`WHERE` 子句用于指定数据库必须查询的行。但是，适用以下规则：

*   `WHERE` 子句只能由属于 `PRIMARY` 键一部分的列上的关系（条件）组成，或者你必须在这些列上定义一个 `SECONDARY INDEX`（二级索引）。
*   对于任何分区键，对聚簇列的关系限制仅为那些寻求连续行集的关系。

假设你创建了以下表，然后向其中插入了一些数据：

```sql
cqlsh> create table users ( id UUID PRIMARY KEY, lastname text, firstname text );
cqlsh> insert into users (id, firstname, lastname)
... values
...  (uuid(), 'sam', 'alapati');
```

以下查询肯定可以工作，因为你指定了主键 `id` 列：

```sql
cqlsh> select * from users where id=5361a682-9aea-46df-91b0-40572cfa9c97;
id                                   | firstname | lastname
--------------------------------------+-----------+----------
5361a682-9aea-46df-91b0-40572cfa9c97 |       sam |  alapati
(1 rows)
cqlsh>
```

然而，以下查询将无法工作：

```sql
cqlsh > select * from users where lastname="alapati";
InvalidRequest: Error from server: code=2200 [Invalid query] message="Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING"
```

这个查询不起作用的原因在于，`lastname` 列上没有二级索引，并且该列的选择条件无法选出一个连续的行集。如果你希望按该列进行查询，你需要在 `lastname` 列上创建一个二级索引。

### 创建二级索引

你可以通过在 `lastname` 列上创建一个二级索引来使这个查询生效。二级索引是表中除主键包含的键之外的任何列上的索引。

一旦你为 `lastname` 列创建了二级索引，之前的查询就能正常工作：

```sql
cqlsh:cycling> create index on users(lastname);
cqlsh:cycling> select * from users where lastname="alapati";
id                                   | firstname | lastname
--------------------------------------+-----------+----------
5361a682-9aea-46df-91b0-40572cfa9c97 |       sam |  alapati
(1 rows)
Cqlsh:cycling>
```

除了像这里展示的在简单列上创建二级索引外，你还可以在基于集合的列（如基于 `map`、`set` 和 `list` 的集合）上创建索引。

### 二级索引的缺点

二级索引由于几个原因可以说是喜忧参半。例如，所有节点都必须维护基于该节点上分区数据的二级索引的本地副本。由于对二级索引的查询通常需要遍历多个节点，执行查询的开销并不小。

二级索引不适用于基数过高或过低的列。基数是指一个列中不同值的数量。索引也不适合处理你频繁修改或删除的数据，原因在于数据库必须处理更新和删除产生的所有墓碑（tombstones），然后压缩（compaction）过程才能处理它们。

理想情况下，你应该创建多个表来满足你想对同一块数据执行的不同查询。与关系型数据库中二级索引是查询的基本组成部分不同，Cassandra 的二级索引充其量是一种备用策略，你最好创建物化视图（materialized views），它遵循了反规范化（denormalization）的推荐模式。我将在“使用物化视图”一节讨论物化视图。

### SASI：二级索引的新实现

Cassandra 3.4 引入了一种更新的二级索引实现，称为 SSTable 附加二级索引（SASI）。与传统二级索引存储在“隐藏”表中不同，数据库将 SSI 索引作为 SSTable 文件的一部分存储。

你可以使用 `CREATE CUSTOM INDEX` 命令创建 SASI 索引：

```sql
cqlsh> CREATE CUSTOM INDEX on users (lastname)
USING 'org.apache.cassandra.index.sasi.SASIIndex';
cqlsh>
```

你可以同时使用传统索引和 SASI 索引。较新的二级索引允许你执行不等式搜索，并使用 `LIKE` 关键字执行文本搜索，这两者都是传统索引无法做到的。

你可以通过使用元组表示法将 `CLUSTERING COLUMNS`（聚簇列）在关系条件中组合在一起，我将在本章后面进行解释。

### 编写条件语句

一个查询可以扫描表的分区以检索一段数据。为了让这类查询生效，你必须顺序存储这段数据，以便使用聚簇列来定义所需的数据段。

让我们用一个简单的例子来展示如何编写条件语句。表 `race_times` 显示了自行车手参加各种比赛的比赛时间：

```sql
cqlsh> CREATE TABLE cycling.race_times (race_name text, cyclist_name text,
race_time text,
PRIMARY KEY (race_name, race_time);
```

这里 `race_name` 是主键，`race_time` 是聚簇列。你现在可以指定一个条件运算符来查找一段数据。你可以使用 `race_time` 列来实现，如下所示：

```sql
cqlsh> SELECT * FROM cycling.race_times
WHERE RACE_NAME = '17th Santos Tour Down Under'
AND race_time >='19:15:19"'
AND race_time <= '19:15:39');
```

## 如何使用 GROUP BY 子句对查询结果进行分组

指定 `GROUP BY` 子句可以将所有共享某一列（或一组列）相同值的行收集到单个行中。

当你指定一个聚合函数（如 `AVG`）以及一个 `GROUP BY` 子句时，Cassandra 会为每个组生成一个单独的值。否则，指定聚合函数将为所有行生成一个单一的值。

以下是一个展示如何指定 `GROUP BY` 子句的示例：

```sql
cqlsh> SELECT weatherstation_id, date, MAX(temperature)
FROM temperature_by_day
GROUP BY weatherstation_id, date;
```

在这个例子中，表的分区键包含两列 `weatherstation_id` 和 `date`，聚簇键是 `event_time`。`GROUP BY` 使用了分区键 (`weatherstation_id`, `date`)。

如果我愿意，我也可以同时使用分区键和聚簇键进行 `GROUP BY`，如下所示：

```sql
cqlsh> SELECT weatherstation_id, date, MAX(temperature)
FROM temperature_by_day
GROUP BY weatherstation_id, date, event_time;
```

## 使用 ORDER BY 子句对查询结果进行排序

你可以微调 Cassandra 显示 `SELECT` 语句结果的顺序。指定 `ORDER BY` 子句来对查询结果进行排序。你只能选择表定义中带有聚簇顺序（clustering order）的列。

你必须在 `WHERE` 子句中定义分区键，并通过指定 `ORDER BY` 子句来定义用于对结果排序的聚簇列。

以下是一个展示如何使用 `ORDER BY` 子句的例子。

首先，创建以下表：

```sql
cqlsh>  CREATE TABLE cycling.cyclist_cat_pts
(CATEGORY text, points int, id UUID, lastname text,
PRIMARY KEY (category, points) );
```

现在运行带有 `ORDER BY` 子句的 `SELECT` 语句。

```sql
cqlsh> SELECT * FROM cycling.cyclist_cat_pts WHERE category = 'GC'
ORDER BY points;
```

在 `cqlsh` 中使用 `ORDER BY` 子句需要你用 `PAGING OFF` 命令关闭分页。

```sql
cqlsh> PAGING OFF
```



## 限制查询结果

您可以通过在 `SELECT` 语句中指定 `LIMIT` `N` 选项来限制其输出的行数，如下所示：

```
cqlsh> SELECT * FROM cycling.cyclist_name LIMIT 3;
```

您还可以指定一个 `PER PARTITION LIMIT` `N` 选项，以限制特定分区返回的行数。首先，创建一个表，该表会将数据排序到多个分区中，并将数据插入到该表中。在此示例中，您使用 `race_year` 和 `race_name` 列作为复合分区键，`rank` 作为聚簇列：

```
cqlsh> CREATE TABLE cycling.rank_by_year_and_name (
    race_year int,
    race_name text,
    cyclist_name text,
    rank int,
    PRIMARY KEY ((race_year, race_name), rank)
);
```

有了这个表，您就可以执行以下 `SELECT` 语句。`PER PARTITION LIMIT N` 子句为分区键 `race_year` 和 `race_name` 的每种组合检索前五场比赛。

```
cqlsh> SELECT * FROM cycling.rank_by_year_and_name PER PARTITION LIMIT 5;
```

## 过滤查询结果

Cassandra 不允许需要过滤的查询。它只允许不需要过滤的查询，在这种情况下，它会返回结果集中读取的所有记录。这样做的原因是使查询具有可预测的性能。查询的执行时间将始终与查询返回的数据量成正比。

假设您有以下表：

```
CREATE TABLE emails (
    emailId int,
    time int,
    from text,
    content text,
    PRIMARY KEY(emailId, time)
);
```

您可以通过发出以下查询来检索表中的所有数据：

```
SELECT * from emails;
```

但是，如果您发出以下查询，Cassandra 会报错：

```
SELECT * FROM emails WHERE time = 1418306451235;
Bad Request: Cannot execute this query as it might involve data filtering and thus may have unpredictable performance. If you want to execute this query despite the performance unpredictability, use ALLOW FILTERING.
```

错误的原因是，当 Cassandra 无法保证即使只查询少量值也不会扫描大量数据时，它就不会运行查询。Cassandra 是在为您着想，试图节省您的计算资源。

要执行此查询，数据库需要从表 `emails` 中检索所有行，然后过滤掉那些 `time` 列不具有请求值的行。但是，如果您知道表中超过 90% 的行都具有 `time` 列的请求值，那么您的查询将是高效的，并且您必须在查询中指定 `ALLOW FILTERING` 子句。

如果表包含一百万行，而只有五行包含 `time` 列的请求值，您的查询将浪费资源，因为 Cassandra 将读取所有一百万行来检索五行。在这种情况下，您应该考虑添加一个二级索引。因此，您可以在 `from` 列上添加一个索引并运行以下查询：

```
SELECT * FROM emails WHERE from = 'Sam Alapati';
```

Cassandra 将检索 Sam Alapati 发送的所有电子邮件，并且不会要求您使用 `ALLOW FILTERING` 子句，因为它使用 `from` 列上的二级索引来查找匹配的行，而无需过滤结果。

您比 Cassandra 更了解您的数据。您可以通过指定 `ALLOW FILTERING` 子句来覆盖默认行为，从而使相同的查询成功运行。

## 使用内置函数聚合结果

Cassandra 3.1 提供了内置的标准聚合函数，例如 `min`、`max`、`avg`、`sum` 和 `count`，用于聚合结果。我提供两个聚合函数的示例：`sum` 和 `count`。

在表 `cycling.cyclist_points` 中，主键是 (`id`, `race_points`)。您可以借助 `sum` 函数找到特定自行车手的比赛积分总和。

```
cqlsh> SELECT sum(race_points)
FROM cycling.cyclist_points
WHERE id=e3b19ec4-774a-4d1c-9e5a-decec1e30aac;
```

在表 `cycling.country_flag` 中，主键是 (`country`, `cyclist_name`)。您可以使用 `count` 函数来查找来自比利时的自行车手数量。

```
cqlsh> SELECT count(cyclist_name) FROM cycling.country_flag WHERE country='Belgium';
```

您可以在查询中应用任何自定义的用户定义函数（UDF），如下所示，其中 `fLog` 是一个用于从此表检索数据的 UDF：

```
cqlsh> SELECT id, lastname, fLog(race_points) FROM cycling.cyclist_points;
```

## 将查询结果格式化为 JSON

您可以以 JSON 格式检索表的数据。要以 JSON 格式获取查询的整个结果，只需在 `SELECT` 语句后插入关键字 `json`。

```
cqlsh> SELECT json name, checkin_id, timestamp from checkin;
```

如果您希望仅以 JSON 格式检索某些列，可以通过将列名包装在 `toJson()` 中来实现。

```
cqlsh> SELECT name, checkin_id, toJson(timestamp) from checkin;
```

## 从集合列中选择数据

查询集合列的方式与查询任何其他列相同。以下示例显示如何为特定自行车手 ID 检索数据。这里，列 `teams` 是一个集合：

```
cqlsh> SELECT lastname, teams
FROM cycling.cyclist_career_teams
WHERE id = 5b6962dd-3f90-4c93-8f61-eabfa4a803e2;
```

当您查询带有集合的表时，Cassandra 将返回完整的集合。结果将根据元素类型排序。例如，文本元素将按字母顺序排列。如果键是整数类型，则顺序将基于键值。如果您需要数据库按数据插入的顺序返回结果，请使用列表而不是集合。

## 使用 IN 关键字进行 CQL 行的多键获取

您可以指定 `IN` 关键字来定义要一起检索的一组聚簇列，从而获取多行。

与 `ORDER BY` 子句一样，当您指定 `IN` 关键字时，必须使用 `PAGING OFF` 命令在 cqlsh 中关闭分页。以下是一些展示如何指定 `IN` 关键字的查询示例。

以下查询根据聚簇列 `category_id` 检索并对结果进行排序：

```
cqlsh> SELECT * FROM cycling.cyclist_cat_pts
WHERE category_id IN ('Time-trial', 'Sprint')
ORDER BY id DESC;
```

以下示例显示了如何指定多个聚簇列。这里 `race_id` 列是分区键，`race_start_date` 和 `race_end_date` 列是聚簇列：

```
cqlsh> SELECT * FROM cycling.calendar WHERE race_id = 101
AND (race_start_date, race_end_date)
IN (('2015-05-09', '2015-05-31'), ('2015-05-06', '2015-05-31'));
```



## 使用 `INSERT` 语句插入数据

你执行 `INSERT` 语句来向表中的一行写入一个或多个列。你必须至少指定已定义主键的列，否则你将无法标识该行。

以下是 `INSERT` 语句的通用语法：
```
INSERT INTO [keyspace_name.] table_name (column_list)
VALUES (column_values)
[IF NOT EXISTS]
[USING TTL seconds | TIMESTAMP epoch_in_microseconds]
```

以下是一个示例，展示如何借助 `VALUES` 子句执行插入操作：
```
cqlsh> INSERT INTO cycling.cyclist_name (id, lastname, firstname)
VALUES (5b6962dd-3f90-4c93-8f61-eabfa4a803e2, 'VOS','Marianne');
```

当你使用 JSON 语法时，指定 `VALUES` 子句是可选的，但请确保你在表名后指定了 `JSON` 关键字。
```
cqlsh> INSERT INTO cycling.cyclist_category JSON '{
"category" : "GC",
"points" : 780,
"id" : "829aa84a-4bba-411f-a4fb-38167a987cda",
"lastname" : "SUTHERLAND" }';
```

提示：如果一行不存在，Cassandra 会创建该行；如果该行已存在，它会更新该行。你实际上无法分辨是创建还是更新这两种操作中的哪一个发生了。你可以指定 `IF NOT EXISTS` 条件，仅当数据不存在时才插入，但这会带来明显的性能损失，因此不应常规使用。

`INSERT` 语句中的 `UPDATE_PARAMETER` 子句也出现在 `UPDATE` 和 `DELETE` 语句中，因此值得深入了解这个参数。`UPDATE_PARAMETER` 子句支持以下参数：
*   `TIMESTAMP`: 为操作设置时间戳。默认情况下，协调器使用语句执行开始时的当前时间作为时间戳。
*   `TTL`: 此参数为你插入的值指定生存时间（以秒为单位）。这是一个可选参数，默认情况下值不过期。将 `TTL` 设置为值 0 与未指定 TTL 相同。

## 使用 `UPDATE` 语句修改数据

Cassandra 的 `UPDATE` 功能实际上是一种 `UPSERT`。当你更新一行时，如果该行之前不存在，Cassandra 会创建它，否则更新它。因此，`UPDATE` 语句最终会写入一行或多行的一个或多个列值。

如果你插入一行，其主键与当前行相同，数据库会替换该行。如果你更新一行且主键不存在，Cassandra 会创建该行。因为 Cassandra 采用追加模型，插入和更新操作的工作方式相同；这两种操作之间没有本质区别。

你可以在除计数器列之外的所有列上指定 TTL 秒或 Timestamp 微秒作为选项。

你可以更新多行中的一个列，如下例所示：
```
cqlsh> UPDATE users
SET state = 'TX'
WHERE user_id IN (12345, 23456, 34567);
```

你可以一次更新一个或多个列。以下命令展示了如何在同一行中更新多个列：
```
cqlsh> UPDATE users
SET name = 'Sam Alapati',
email='samalapati@gmail.com'
WHERE user_id = 23456;
```

你可以更新集合 set、映射 map 或列表 list 中的数据。

你也可以执行条件更新，如下所示：
```
cqlsh> UPDATE users SET id = 12345
WHERE lastname = 'ALAPATI' and firstname = 'Sam'
IF age =50;
```

条件更新是轻量级事务，会带来性能损失，因此你必须谨慎使用它们。

变更数据捕获 (CDC) 日志记录：Cassandra 提供变更数据捕获 (CDC) 日志记录来跟踪已更改的数据。你按表配置 CDC 日志记录。当你决定使用 CDC 日志记录时，还必须指定 CDC 日志可以使用的磁盘空间量的限制。当数据库将内存表数据刷新到磁盘时，数据库会将包含与你已启用 CDC 日志记录的表相关的数据的提交日志段移动到 CDC 目录。你可以在创建表时配置 CDC 日志记录，或者修改现有表以添加相关的表属性。

`cassandra.yaml` 文件中的 `cdc_raw_directory` 属性使你能够指定数据库存储 CDC 日志的目录。此目录的默认位置如下：
*   软件包安装：`/var/lib/Cassandra/cdc_raw`
*   Tarball 安装：`install_location/data/cdc_raw`

## 处理高级数据类型

你可以创建几种高级数据类型，例如：
*   集合 (Collections)：使你能够将数据分组并存储在单个列中。
*   元组 (Tuples)：使你能够将多个值一起存储在单个列中。
*   用户定义类型 (UDTs)：使你能够将多个数据字段附加到单个列。

注意：UDT 仅适用于单个键空间。要在不同的键空间中使用 UDT，你必须重新创建该 UDT。

我将在以下部分解释这三种高级数据类型。

### 集合

集合数据类型允许你将数据分组存储在单个列中。例如，如果一个用户有多个地址或电子邮件地址，你可以将所有地址一起存储在一个集合列中。

请记住，集合仅适用于值数量有限的数据。也就是说，集合中每个项目的最大大小是有限的。当存储无限增长的数据时（例如数据库每秒捕获和存储的事件），集合不是合适的方式。

CQL 允许你创建三种集合类型：
*   Set：一组具有唯一值的元素。数据库不以有序方式存储值，但 cqlsh 会以排序方式返回元素，例如对于文本值按字母顺序。
*   List：列表对多个值进行分组，但这些值不必唯一。此外，列表以特定顺序存储元素，你可以使用索引值来插入和查询列表的值。
*   Map：映射借助键值对建立两个项目之间的关系。每个键对应一个值，且不能有重复键。

提示：Cassandra 会读取整个集合，从而对性能产生不利影响。请保持集合的大小远小于每种集合类型的最大限制。

你通过在尖括号中指定集合类型（set、list 或 map），后跟诸如 `int` 或 `text` 之类的类型来声明集合列。这里有几个例子：
```
list
list
```

你可以在这三种类型的集合列上创建索引。

在以下部分中，我将展示如何创建这三种集合类型。

#### 创建 Set 类型

当你在列中有数据与另一列中的数据存在多对一关系时，可以使用 set 类型。set 类型在这种情况下很有帮助，因为你在 Cassandra 中不进行表连接。

例如，在以下示例中，由 `id` 列表示的单个骑手可能在不同时期是多个车队的成员：
```
cqlsh> CREATE TABLE cycling.cyclist_career_teams
( id UUID PRIMARY KEY, lastname text, teams set<text>);
```

你可以通过以下方式查询作为 set 一部分的车队：
```
cqlsh:cycling> SELECT lastname,teams
...         FROM cycling.cyclist_career_teams;
lastname        | teams
-----------------+------------------------------------------------------------------------------------------------------
ARMITSTEAD |                    {'AA Drink - Leontien.nl', 'Boels-Dolmans Cycling Team', 'Team Garmin - Cervelo'}
VOS | {'Nederland bloeit', 'Rabobank Women Team', 'Rabobank-Liv Giant', 'Rabobank-Liv Woman Cycling Team'}
BRAND |   {'AA Drink - Leontien.nl', 'Leontien.nl', 'Rabobank-Liv Giant', 'Rabobank-Liv Woman Cycling Team'}
VAN DER BREGGEN |                 {'Rabobank-Liv Woman Cycling Team', 'Sengers Ladies Cycling Team', 'Team Flexpoint'}
(4 rows)
cqlsh:cycling>
```

集合类型中单个项目的最大大小为 65,535 字节。

注意：你可以通过指定单独的 TTL 属性来使集合的每个元素过期。


# Cassandra 集合与用户定义函数

### 集合

### 创建列表类型

当处理与另一列存在多对多关系的列时，请使用 `list` 数据类型。在以下示例中，`events` 列是一个列表，用于存储每个月内的所有赛事：

```sql
cqlsh> CREATE TABLE cycling.upcoming_calendar ( year int,
month int, events list,
PRIMARY KEY ( year, month ));
```

您可以查询该表以获取某年某月的赛事列表。

```sql
cqlsh:cycling> SELECT * FROM cycling.upcoming_calendar WHERE year=2015 AND month=06;
year | month | events
------+-------+---------------------------------------------
2015 |     6 | ['Criterium du Dauphine', 'Tour de Suisse']
(1 rows)
cqlsh:cycling>
```

`list` 集合类型中单个项目的最大大小为 2GB。

### 创建映射类型

以下示例展示了如何创建一个包含 `map` 类型的表。`teams` 列具有 `map` 类型，每个团队显示骑手在该年份所属的团队年份和名称。

```sql
cqlsh> CREATE TABLE cycling.cyclist_teams ( id UUID PRIMARY KEY, lastname text,
firstname text, teams map );
```

您可以这样查询该表：

```sql
cqlsh> SELECT lastname, firstname, teams
FROM cycling.cyclist_teams;
```

此查询显示与特定骑手相关的所有团队及其所属年份。

### 在集合中指定冻结值

您可以将类型标记为 `frozen`，以将多个组件序列化为单个值。您无法更新冻结类型中的单个字段（Cassandra 将这些值视为一个 `blob`）。

示例如下：

```sql
cqlsh> CREATE TABLE mykeyspace.users (
id uuid PRIMARY KEY,
name frozen ,
direct_reports set>,    // 一个集合 set
addresses map>     // 一个集合 map
score set>>              // 一个包含嵌套冻结集合的集合
);
```

### 元组

您可以使用 `tuple` 数据类型将多个值一起存储在一列中。以下示例展示了如何为其中一列创建包含 `tuple` 数据类型的表：

```sql
cqlsh> CREATE TABLE cycling.nation_rank (nation text PRIMARY KEY, info
tuple);
```

以下查询从 `nation_rank` 表中检索数据：

```sql
cqlsh> SELECT * FROM cycling.nation_rank;
```

### 用户定义类型

通过定义您自己的数据类型（称为用户定义类型），您可以将多个命名和类型化的数据字段附加到同一列。您可以指定任何有效的数据类型，包括集合甚至其他 UDT，作为 UDT 的字段。

以下示例展示了如何创建一个名为 `cyclist.fullname` 的简单 UDT：

```sql
cqlsh> CREATE TYPE cycling.fullname (firstname text, lastname text);
```

然后，您可以创建一个使用新类型 `cyclist.fullname` 的表：

```sql
cqlsh> CREATE TABLE cycling.race_winners
(race_name text, race_position int, cyclist_name FROZEN,
PRIMARY KEY (race_name, race_position));
```

向表中插入数据后，您可以运行以下查询：

```sql
cqlsh:cycling> SELECT * FROM cycling.race_winners
WHERE race_name = 'National Championships South Africa WJ-ITT (CN)';
race_name                                       | race_position | cyclist_name
-------------------------------------------------+---------------+--------------------------------------------------
National Championships South Africa WJ-ITT (CN) |             1 |      {firstname: 'Frances', lastname: 'DU TOUT'}
National Championships South Africa WJ-ITT (CN) |             2 |       {firstname: 'Lynette', lastname: 'BENSON'}
...
(5 rows)
cqlsh:cycling>
```

请注意查询如何利用您定义的 `full_name` 类型来显示骑手的 `firstname` 和 `lastname` 列的值。

### 用户定义函数与用户定义聚合

当处理包含数百个节点的大型数据集和集群时，在客户端应用聚合与将大量计算推送到服务器端之间存在显著差异。通过让服务器处理计算，您将节省网络带宽，简化客户端代码，并获得额外好处。

在以下部分中，我将解释用户定义函数（UDF）和用户定义聚合（UDA）的要点。

### 用户定义函数

关于 UDF，您首先需要知道的是它的作用域并非像 Oracle 或 MySQL 等数据库中的函数和存储过程那样是数据库级别的。用户定义函数的作用域仅限于键空间。

### 关于 UDF 的注意事项

以下是您需要了解的关于 UDF 的关键点。

*   您可以使用 Java、JavaScript、Groovy 和 Scala 等语言来编写 UDF。
*   您可以通过两种方式处理 `null` 输入：
    *   您可以指定 `CALLED ON NULL INPUT` 子句，这意味着 Cassandra 将始终调用 UDF。
    *   如果您指定 `RETURNS NULL ON NULL INPUT` 子句，那么当任何参数为 `null` 时，Cassandra 会跳过 UDF 的执行，并返回一个 `null`。
*   与输入参数一样，您必须为返回类型指定有效类型，例如原始类型、集合、元组或 CDT。

### UDF 的语法

创建 UDF 的语法如下：

```sql
CREATE [OR REPLACE] FUNCTION [IF NOT EXISTS]
[keyspace.]functionName (param1 type1, param2 type2, ...)
CALLED ON NULL INPUT | RETURNS NULL ON NULL INPUT
RETURN returnType
LANGUAGE language
AS '
// 您的源代码
';
```

`OR REPLACE` 子句和 `IF NOT EXISTS` 子句是互斥的。

### UDF 示例

以下是一个展示如何创建和使用 UDF 的示例。该 UDF 名为 `maxof (currentvalue int, testvalue int)`。此函数允许您获取传递给它的两个整数的最大值。

默认情况下，UDF 是禁用的。因此，在尝试以下操作之前，您必须首先在 `cassandra.yaml` 文件中设置 `enable_user_defined_functions: true` 以启用它：

```sql
cqlsh> CREATE FUNCTION maxof(currentvalue int, testvalue int)
RETURNS NULL ON NULL INPUT
RETURNS int
LANGUAGE java
AS 'return Math.max(currentvalue,testvalue);';
```

然后，您可以创建一个测试表并向其中插入一些数据，之后可以使用新的 UDF `maxOf` 执行查询，以获取两个值 `val1` 和 `val2` 之间的最大值。一些输入值被特意指定为 `null`。示例如下：

```sql
cqlsh> SELECT id,val1,val2,maxOf(val1,val2) FROM test WHERE id IN(1,2,3);
id | val1 | val2 | udf.maxof(val1, val2)
-----+------+------+
1  |  100 |  200 |                   200
2  |  100 | null |                  null
3  | null |  200 |                  null
```

由于 UDF 指定了 `RETURNS NULL ON NULL INPUT` 子句，因此当输入参数为 `null` 时，Cassandra 会返回一个 `null`。

### 用户定义聚合函数

Cassandra 允许您创建用户定义聚合函数，您可以在查询结果中将它们应用于数据。

您首先创建聚合函数，您的查询可以仅包含聚合函数本身，而不包含任何列。

### 内置函数和聚合

Cassandra 提供了一些方便的内置函数和聚合供您使用。列表如下：

*   `COUNT`：`COUNT` 函数获取行数的计数。
*   `SUM`：`SUM` 函数使您能够对特定列的所有值求和。
*   `MIN`/`MAX`：`MIN`/`MAX` 函数帮助您计算列的最小值/最大值。
*   `AVG`：`AVG` 函数计算给定列中所有值的平均值。

## 总结

理解 Cassandra 中索引的正确用法是复杂的，因为它与您在传统关系数据库中使用索引的方式非常不同。

Cassandra 集合，如 `set`、`map`、`list` 和 UDF 非常有用，了解它们的工作原理是很有益的。


# 7. 在 Docker、Apache Spark 和 Cassandra 集群管理器上运行 Cassandra

第 2 章和第 3 章展示了如何在多个节点上创建一个通用的 Cassandra 集群。现在你已经对 Cassandra 的架构以及如何配置集群有了相当的了解，是时候学习如何在各种环境中编排 Cassandra 集群了。

容器技术如今正风靡一时。本章首先向你展示如何使用 Docker 容器创建一个 Cassandra 集群。我会快速解释什么是 Docker 容器以及如何使用它们。然后，我将向你展示如何在 Docker 上安装和配置 Cassandra。

在讨论完 Docker 上的 Cassandra 之后，我将展示如何使用 Docker-Compose 和 BDD（行为驱动开发）创建一个 Cassandra 数据库。

Cassandra 集群管理器（CCM），顾名思义，并非 Cassandra 内部用来管理集群的工具。相反，它是一个便捷的工具，用于在单台服务器上设置多节点 Cassandra 集群。虽然 CCM 是用于开发和测试的，但了解如何使用此工具设置集群是很有益的。

最后，本章以对名为 SMACK（Apache Spark、Apache Mesos、Akka、Cassandra 和 Apache Kafka）的新兴技术栈的简要介绍作为结束，这是一套日益流行用于创建分布式企业应用程序的开源工具集。

### Cassandra 与 Docker

在本节中，我将展示如何在 Docker 容器上运行 Cassandra 集群。容器是一个轻量级的、独立的、可执行的软件包，包含了运行软件所必需的一切，例如代码、运行时、系统工具、系统库和配置文件。

### Docker：快速入门

Docker 是一个应用程序，它使你能够轻松地在容器中运行应用。容器类似于虚拟机（VM），但便携性更好。与在虚拟机管理程序中运行的 VM 不同，它直接在主机操作系统上运行。

### 安装 Docker

在本节中，我将展示如何在 Ubuntu 16.04 服务器上安装 Docker。

1.  Docker 是 Ubuntu 16.04 安装的一部分。然而，由于这可能不是最新版本，请从源下载，如下所示：

    ```
    $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    OK
    $
    ```

2.  通过以下方式将 Docker 仓库添加到 APT 源中：

    ```
    $ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    $
    ```

3.  用你刚刚添加的新 Docker 软件包更新软件包数据库。

    ```
    $ sudo apt-get update
    ```

4.  执行以下操作，确保你安装的是来自 Docker 仓库的 Docker CE（社区版），而不是默认的 Ubuntu 仓库：

    ```
    $ apt-cache policy docker-ce
    docker-ce:
    Installed: (none)
    Candidate: 17.06.0~ce-0~ubuntu
    Version table:
    17.06.0~ce-0~ubuntu 500
    500 https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
    ...
    $
    ```

    输出验证了可用的 Docker 安装文件是用于 Ubuntu-Xenial (16.04)的，所以你可以继续进行。你的`docker-ce`版本号可能与我的不同。
5.  此时，你已准备好安装 Docker。

    ```
    $ sudo apt-get install -y docker-ce
    Reading package lists... Done
    Building dependency tree
    Reading state information... Done
    ...
    Processing triggers for ureadahead (0.100.0-19) ...
    $
    ```

### 管理 Docker

接下来，你可以通过`systemctl status docker`命令检查 Docker 是否正在运行。

```
$ sudo systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: e
   Active: active (running) since Thu 2017-06-29 10:50:24 EDT; 5s ago
     Docs: https://docs.docker.com
 Main PID: 6598 (dockerd)
   CGroup: /system.slice/docker.service
           ├─6598 /usr/bin/dockerd -H fd://
           └─6603 docker-containerd -l unix:///var/run/docker/libcontainerd/dock
...
$
```

输出表明 Docker 服务正在运行。

### 使用 Docker 命令行实用程序

你使用`docker`命令行实用程序来管理 Docker 实例。只需在命令行输入`docker`即可查看所有可用选项，数量相当多。

例如，你可以通过以下方式查看关于 Docker 的系统级信息：

```
$ sudo docker info
Containers: 6
Running: 0
Paused: 0
Stopped: 6
...
Operating System: Ubuntu 16.04.2 LTS
OSType: linux
Architecture: x86_64
CPUs: 1
Total Memory: 7.796GiB
Name: ubuntu1
...
$
```

### 理解 Docker 镜像

所有 Docker 容器都从一个 Docker 镜像运行。默认情况下，Docker 从 Docker Hub 拉取其镜像，这是一个由 Docker 公司维护的公共 Docker 注册表。其他组织，包括 Linux 发行版，也在 Docker Hub 上托管他们的镜像。

你可以运行以下命令来检查你的 Docker 安装是否能够访问并从 Docker Hub 下载镜像：

```
$ sudo docker run hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.
$
```

你可以使用`docker search`命令在 Docker Hub 上搜索镜像。如果你想查找所有可用的 Ubuntu 镜像，请输入：

```
$ sudo docker search ubuntu
```

然后，你可以通过`docker pull`命令指定镜像名称来下载你想要的镜像。

```
$ sudo docker pull ubuntu
```

你可以通过以下方式查看已下载到你服务器的所有镜像：

```
$ sudo docker images
REPOSITORY        TAG           IMAGE ID          CREATED         SIZE
hello-world       latest        1815c82652c0       2 weeks ago    1.84kB
ubuntu            latest        d355ed3537e9      12 days ago     119MB
...
$
```

最后，你可以使用`docker run`命令，用你已下载的镜像运行一个容器。

```
$ sudo docker run -it ubuntu
samalapati@bd78ccf18941:/#
```

这将启动一个运行 Ubuntu 操作系统的容器。`i`选项代表交互式，`t`选项提供进入容器的 Shell 访问权限。请注意，命令提示符发生了变化，因为你现在位于新的 Ubuntu 容器内部。容器 ID 是`bd78ccf18941`。

现在你已经掌握了 Docker 的基础知识，是时候学习如何在 Docker 上运行 Cassandra 容器了。



### 在 Docker 上运行 Cassandra 集群

让我们在 Docker 上创建一个三节点的 Cassandra 集群。首先，让 Docker 从 Docker Hub 拉取 Cassandra 3.04 镜像来启动第一个 Cassandra 实例。

```bash
$ sudo docker run --name cassandra1 -m 4g -d cassandra:3.0.4
34842881b8d4b42a9cc6627405f6dd63a21cace23276091e869e8b708fb6ce4b
$
```

在此命令中，
*   `--name` 选项使您能够指定此实例的名称。
*   `-m` 选项让您为此实例指定内存。
*   `cassandra:3.0.4` 子句告诉 Docker 应下载哪个版本的 Cassandra 镜像。

接下来，运行 `docker ps` 命令以检查 Cassandra 容器实例 `cassandra1` 是否正在运行。

```bash
$ sudo docker ps
CONTAINER ID       IMAGE          COMMAND                 CREATED             STATUS             PORTS                                  NAMES
34842881b8d4        cassandra:3.0.4     "/docker-entrypoin..."   8 seconds ago       Up 7 seconds        7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   cassandra1
#
```

您运行熟悉的 `nodetool status` 命令来检查 Cassandra 节点 `cassandra1` 的状态。

```bash
$ nodetool status
nodetool: command not found
samalapati@ubuntu1:/tmp# sudo docker exec -i -t cassandra1 sh -c 'nodetool status'
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID      Rack
UN  172.17.0.2  102.52 KB  256     100.0%     f217c613-3eb9-445a-884a-183e8d177927  rack1
root@ubuntu1:/tmp#
```

现在第一个 Cassandra 节点已经运行，是时候通过创建第二个节点来组成集群了，使用第一个节点作为种子节点。您创建的第二个节点及任何其他节点将与这个节点进行 gossip 通信并注册自己。为此，您需要第一个 Cassandra 节点 `cassandra1` 的 IP 地址。您可以通过以下方式获取：

```bash
$ sudo docker inspect -f'{{.NetworkSettings.IPAddress}}' cassandra1
172.17.0.3
$
```

该命令显示第一个节点的 IP 地址是 `172.17.0.3`。现在您已经掌握了创建第二个 Cassandra 节点所需的一切，您将其命名为 `cassandra2`。

```bash
