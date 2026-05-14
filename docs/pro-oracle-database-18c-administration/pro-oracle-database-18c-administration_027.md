# 物化视图技术

物化视图（MV）技术在 Oracle 数据库第 7 版中引入。该功能最初被称为快照，您仍能在一些数据字典结构中看到这种命名方式。物化视图允许您在某个时间点执行 SQL 查询，并将结果集存储在一个表中（本地或远程数据库均可）。在物化视图初始填充后，您可以重新运行物化视图查询，并将新鲜结果存储到底层表中。物化视图有三种主要用途：

*   复制数据以将查询工作负载分流到单独的报告数据库
*   通过定期计算和存储复杂数据聚合的结果来提高查询性能，使用户能够查询（复杂聚合的）时间点结果
*   如果查询重写未发生，则阻止查询执行。

物化视图可以是基于表、视图和其他物化视图的查询。基础表通常被称为主表。当您创建物化视图时，Oracle 会在内部创建一个表（与物化视图同名）以及一个物化视图对象（可在 `DBA/ALL/USER_OBJECTS` 中看到）。

## 理解物化视图

注意
本章大部分示例将使用 `SALES` 表作为基础。

介绍物化视图的一个好方法是，如果没有物化视图功能，您将如何手动执行一项任务。假设您有一个存储销售数据的表：

```sql
SQL> create table sales(
sales_id  number
,sales_amt number
,region_id number
,sales_dtt date
,constraint sales_pk primary key(sales_id));
--
SQL> insert into sales values(1,101,10,sysdate-10);
SQL> insert into sales values(2,511,20,sysdate-20);
SQL> insert into sales values(3,11,30,sysdate-30);
SQL> commit;
```

并且，您有一个报告历史每日销售情况的查询：

```sql
SQL> select
sum(sales_amt) sales_amt
,sales_dtt
from sales
group by sales_dtt;
```

您从数据库性能报告中观察到，这个查询每天执行数千次，并消耗大量数据库资源。业务用户使用该报告来显示历史销售信息，因此不需要每次运行报告时都重新执行该查询。为了减少该查询消耗的资源量，您决定创建一个表并按如下方式填充：

```sql
SQL> create table sales_daily as
select
sum(sales_amt) sales_amt
,sales_dtt
from sales
group by sales_dtt;
```

表创建后，您设置一个每日进程来删除并完全刷新它：

```sql
-- 步骤 1 删除每日聚合销售数据：
SQL> delete from sales_daily;
--
-- 步骤 2 用聚合销售表的快照重新填充表：
SQL> insert into sales_daily
select
sum(sales_amt) sales_amt
,sales_dtt
from sales
group by sales_dtt;
```

您告知用户，他们可以通过从 `SALES_DAILY` 中选择（而不是运行直接从主 `SALES` 表中选择和聚合的查询）来获得亚秒级的查询结果：

```sql
SQL> select * from sales_daily;
```

上述过程大致描述了物化视图完全刷新的过程。Oracle 的物化视图技术自动化并极大地增强了这一过程。本章涵盖了实现基本和复杂物化视图功能的步骤。阅读本章并完成示例后，您应该能够在各种情况下创建物化视图以复制和聚合数据。

在深入探讨创建物化视图的细节之前，先介绍与物化视图相关的基本术语和有用的数据字典视图是很有帮助的。接下来的两个小节将简要描述各种物化视图功能以及包含物化视图元数据的众多数据字典视图。

注意
本章不涵盖多主复制和可更新物化视图等主题。有关这些主题的更多详细信息，请参阅《Oracle 高级复制指南》，该指南可从 Oracle 网站（`http://otn.oracle.com`）的技术网络区域下载。

## 物化视图术语


## 与刷新物化视图相关的术语

许多术语与刷新物化视图（MV）相关。在深入了解如何实现相关功能之前，您应该熟悉这些术语。表 **15-1** 定义了与 MV 相关的各种术语。

**表 15-1 MV 术语表**

| 术语 | 含义 |
| :--- | :--- |
| **物化视图（MV）** | 用于复制数据和提升查询性能的数据库对象。 |
| **MV SQL 语句** | 定义物化视图底层基表中存储何种数据的 SQL 查询。 |
| **MV 底层表** | 与 MV 同名、用于存储 MV SQL 查询结果的数据库表。 |
| **主体（基）表** | 在 MV SQL 语句的 `FROM` 子句中被引用的表。 |
| **完全刷新** | 删除 MV 并使用 MV SQL 语句完全重新填充的过程。 |
| **快速刷新** | 自上次刷新以来，仅将基表上发生的 DML 更改应用到 MV 的过程。 |
| **MV 日志** | 跟踪 MV 基表上 DML 更改的数据库对象。快速刷新需要 MV 日志。它可以基于主键、`ROWID` 或对象 ID。 |
| **简单 MV** | 基于可快速刷新的简单查询的 MV。 |
| **复杂 MV** | 基于无法进行快速刷新的复杂查询的 MV。 |
| **构建模式** | 指定 MV 是应立即填充还是延迟填充的模式。 |
| **刷新模式** | 指定 MV 是应按需刷新、提交时刷新还是永不刷新的模式。 |
| **刷新方法** | 指定 MV 刷新应为完全刷新还是快速刷新的选项。 |
| **查询重写** | 允许优化器选择使用 MV（而不是基表）来满足查询要求的功能（即使查询没有直接引用这些 MV）。 |
| **本地 MV** | 与基表位于同一数据库中的 MV。 |
| **远程 MV** | 位于与基表不同数据库中的 MV。 |
| **刷新组** | 在同一一致事务点进行刷新的一组 MV。 |

在阅读本章后续内容时，请随时参考 **表 15-1**。后续章节将对这些术语和概念进行解释和阐述。

## 引用有用的视图

当您处理 MV 时，有时可能很难记住在特定情况下应查询哪个数据字典视图。系统提供了多种数据字典视图。**表 15-2** 描述了与 MV 相关的数据字典视图。本章在适当的地方展示了这些视图的使用示例。这些视图对于故障排除、诊断问题以及理解您的 MV 环境非常宝贵。

**表 15-2 MV 数据字典视图定义**

| 数据字典视图 | 含义 |
| :--- | :--- |
| `DBA/ALL/USER_MVIEWS` | 关于 MV 的信息，例如所有者、基查询、上次刷新时间等。 |
| `DBA/ALL/USER_MVIEW_REFRESH_TIMES` | MV 上次刷新时间、MV 名称、主体表、主体所有者。 |
| `DBA/ALL/USER_REGISTERED_MVIEWS` | 所有已注册的 MV；有助于识别哪些 MV 正在使用哪些 MV 日志。 |
| `DBA/ALL/USER_MVIEW_LOGS` | MV 日志信息。 |
| `DBA/ALL/USER_BASE_TABLE_MVIEWS` | 具有 MV 日志的表的基表名称和上次刷新日期。 |
| `DBA/ALL/USER_MVIEW_AGGREGATES` | 在 MV 的 `SELECT` 子句中出现的聚合函数。 |
| `DBA/ALL/USER_MVIEW_ANALYSIS` | 关于 MV 的信息。Oracle 建议您使用 `DBA/ALL/USER_MVIEWS` 代替这些视图。 |
| `DBA/ALL/USER_MVIEW_COMMENTS` | 与 MV 关联的任何注释。 |
| `DBA/ALL/USER_MVIEW_DETAIL_PARTITION` | 分区和新鲜度信息。 |
| `DBA/ALL/USER_MVIEW_DETAIL_SUBPARTITION` | 子分区和新鲜度信息。 |
| `DBA/ALL/USER_MVIEW_DETAIL_RELATIONS` | MV 所依赖的本地表和 MV。 |
| `DBA/ALL/USER_MVIEW_JOINS` | MV 定义中 `WHERE` 子句里两列之间的连接。 |


## 创建基本物化视图

本节介绍如何创建物化视图（MV）。最常用的两种配置如下：

*   创建按需刷新的完全刷新物化视图
*   创建按需刷新的快速刷新物化视图

理解这些基本配置非常重要。它们为你后续使用物化视图功能的所有操作奠定基础。因此，本节从这些基本配置开始讲解。稍后，本节将涵盖更高级的配置。

### 创建完全可刷新的物化视图

本节解释如何设置一个定期进行完全刷新的物化视图，这可能是最简单的例子了。完全刷新适用于基表中的大部分行在每次刷新间隔内发生变化的物化视图。在某些情况下，由于 Oracle 施加的限制无法进行快速刷新时，也必须使用完全刷新（本节稍后会详细介绍；另请参阅本章后面的“从 SQL *Plus 手动刷新物化视图”一节）。

**注意**
要创建物化视图，你需要同时拥有 `CREATE MATERIALIZED VIEW` 系统特权和 `CREATE TABLE` 系统特权。如果创建物化视图的用户不是基表的所有者，那么要执行提交时刷新（`ON COMMIT REFRESH`），还需要对基表具有 `SELECT` 访问权限。

本节中的物化视图示例基于之前创建的 `SALES` 表。假设你想创建一个报告每日销售额的物化视图。使用 `CREATE MATERIALIZED VIEW...AS SELECT` 语句来实现。以下语句为物化视图命名，指定其属性，并定义了该物化视图所基于的 SQL 查询：

```sql
SQL> create materialized view sales_daily_mv
segment creation immediate
refresh
complete
on demand
as
select
sum(sales_amt) sales_amt
,trunc(sales_dtt) sales_dtt
from sales
group by trunc(sales_dtt);
```

`SEGMENT CREATION IMMEDIATE` 子句在 Oracle 11g Release 2 及更高版本中可用。此子句指示 Oracle 在创建物化视图时创建段并分配区。这是 Oracle 以前版本的行为。如果你不希望立即创建段，请使用 `SEGMENT CREATION DEFERRED` 子句。如果新创建的物化视图包含任何行，则无论你是否使用 `SEGMENT CREATION DEFERRED`，都会创建段并分配区。

让我们查看 `USER_MVIEWS` 数据字典视图，以验证物化视图是否按预期创建。运行此查询：

```sql
SQL> select mview_name, refresh_method, refresh_mode
,build_mode, fast_refreshable
from user_mviews
where mview_name = 'SALES_DAILY_MV';
```

以下是此物化视图的输出：

```
MVIEW_NAME      REFRESH_ REFRES BUILD_MOD FAST_REFRESHABLE
--------------- -------- ------ --------- ------------------
SALES_DAILY_MV  COMPLETE DEMAND IMMEDIATE DIRLOAD_LIMITEDDML
```

检查 `USER_OBJECTS` 和 `USER_SEGMENTS` 视图以了解创建了什么也很有用。当你查询 `USER_OBJECTS` 时，请注意已创建了一个物化视图对象和一个表对象：

```sql
SQL> select object_name, object_type
from user_objects
where object_name like 'SALES_DAILY_MV'
order by object_name;
```

对应的输出是：

```
OBJECT_NAME          OBJECT_TYPE
-------------------- -----------------------
SALES_DAILY_MV       MATERIALIZED VIEW
SALES_DAILY_MV       TABLE
```

物化视图是一个逻辑容器，它将数据存储在一个常规的数据库表中。查询 `USER_SEGMENTS` 视图显示了基表、其主键索引以及存储物化视图查询返回数据的表：

`DBA/ALL/USER_MVIEW_KEYS` | 物化视图定义的 `SELECT` 子句中的列或表达式 |
`DBA/ALL/USER_TUNE_MVIEW` | 执行 `DBMS_ADVISOR.TUNE_MVIEW` 过程的结果 |
`V$MVREFRESH` | 当前正在刷新的物化视图的信息 |
`DBA/ALL/USER_REFRESH` | 关于物化视图刷新组的详细信息 |
`DBA_RGROUP` | 关于物化视图刷新组的信息 |
`DBA_RCHILD` | 物化视图刷新组中的子对象 |



```markdown
# 创建可快速刷新的物化视图

当你创建一个可快速刷新的物化视图时，它首先使用物化视图查询的完整结果集填充物化视图表。初始结果集就位后，只需将自上次刷新以来基础表中修改的数据应用到物化视图。换句话说，自上次刷新以来对主表进行的任何更新、插入或删除都会被复制过来。当你有一段时间内对基础表的修改数量相对于表中的总行数较少时，此功能非常合适。

以下是实现快速刷新物化视图的步骤：

1.  创建基础表（如果尚未创建）。
2.  在基础表上创建物化视图日志。
3.  创建一个可快速刷新的物化视图。

此示例使用先前创建的 `SALES` 表。一个可快速刷新的物化视图需要在基础表上创建物化视图日志。当发生快速刷新时，物化视图日志必须有一种唯一的方法来识别哪些记录已被修改，因此需要刷新。你可以用两种不同的方法来做到这一点。一种方法是在创建物化视图日志时指定 `PRIMARY KEY` 子句；另一种是使用 `ROWID` 子句。如果底层基础表有主键，则使用基于主键的物化视图日志。如果底层基础表没有主键，则必须使用 `ROWID` 创建物化视图日志。在大多数情况下，你可能会为每个基础表定义一个主键。然而，现实情况是，有些系统设计不佳，或者有某些罕见原因导致表没有主键。

在此示例中，基础表上定义了主键，因此你使用 `PRIMARY KEY` 子句创建物化视图日志：

```sql
SQL> create materialized view log on sales with primary key;
```

如果基础表上没有定义主键，在尝试创建物化视图日志时会抛出此错误：

```
ORA-12014: table does not contain a primary key constraint
```

如果基础表没有主键，并且你没有添加主键的选项，则必须在创建物化视图日志时指定 `ROWID`：

```sql
SQL> create materialized view log on sales with rowid;
```

当你使用基于主键的快速刷新物化视图时，基础表的主键列必须是快速刷新物化视图 `SELECT` 语句的一部分（参见本章后面的“基于复杂查询创建快速刷新物化视图”一节）。对于此示例，物化视图中不会有聚合列。这种类型的物化视图通常用于将数据从一个环境复制到另一个环境：物化视图中不会有聚合列。这种类型的物化视图通常用于将数据从一个环境复制到另一个环境：

```sql
SQL> create materialized view sales_rep_mv
segment creation immediate
refresh
with primary key
fast
on demand
as
select
sales_id
,sales_amt
,trunc(sales_dtt) sales_dtt
from sales;
```

此时，检查与物化视图关联的对象非常有用。以下查询从 `USER_OBJECTS` 中选择：

```sql
SQL> select object_name, object_type
from user_objects
where object_name like '%SALES%'
order by object_name;
```

已创建的对象如下：

```
OBJECT_NAME          OBJECT_TYPE
-------------------- -----------------------
MLOG$_SALES          TABLE
RUPD$_SALES          TABLE
SALES                TABLE
SALES_PK             INDEX
SALES_PK1            INDEX
SALES_REP_MV         TABLE
SALES_REP_MV         MATERIALIZED VIEW
```

前面输出中的一些对象需要一些解释：

*   `MLOG$_SALES`
*   `RUPD$_SALES`
*   `SALES_PK1`
```


## 架构组件

首先，当创建一个物化视图日志时，也会创建一个对应的表，用于存储基础表中发生变化的行及其变化方式（插入、更新或删除）。物化视图日志表的名称遵循格式 `MLOG$_<base table name>`。

同样会创建一个格式为 `RUPD$_<base table name>` 的表。当您使用主键创建可快速刷新的物化视图时，Oracle 会自动创建这个 `RUPD$` 表。该表的存在是为了支持可更新物化视图功能。除非您正在处理可更新物化视图（有关可更新物化视图的更多详细信息，请参阅《Oracle Advanced Replication Guide》），否则您无需担心此表。如果您没有使用可更新物化视图功能，则可以忽略 `RUPD$` 表。

此外，Oracle 会创建一个格式为 `<base table name>_PK1` 的索引。此索引会自动为主键物化视图创建，并基于基础表的主键列。如果这是基于 `ROWID` 而不是主键，则索引名称格式为 `I_SNAP$_<table_name>` 并基于 `ROWID`。如果您没有显式地为基础表的主键索引命名，Oracle 将为物化视图表的主键索引分配一个系统生成的名称，例如 `SYS_C008780`。

## 查看数据

现在您了解了底层的架构组件，让我们看看物化视图中的数据：

```sql
SQL> select sales_amt, to_char(sales_dtt,'dd-mon-yyyy')
from sales_rep_mv
order by 2;
```

输出如下：

```
SALES_AMT TO_CHAR(SALES_DTT,'D
---------- --------------------
511 10-jan-2013
101 20-jan-2013
127 30-jan-2013
99 30-jan-2013
11 31-dec-2012
```

## 刷新物化视图

让我们向基础 `SALES` 表添加两条记录：

```sql
SQL> insert into sales values(6,99,20,sysdate-6);
SQL> insert into sales values(7,127,30,sysdate-7);
SQL> commit;
```

此时，检查 `MLOG$` 表很有指导意义。您应该看到两条记录，它们标识了 `SALES` 表中数据是如何变化的：

```sql
SQL> select count(*) from mlog$_sales;
```

有两条记录：

```
COUNT(*)
----------
2
```

接下来，让我们刷新物化视图。此物化视图可快速刷新，因此您需要调用 `DBMS_MVIEW` 包的 `REFRESH` 过程并使用 `F`（代表快速）参数：

```sql
SQL> exec dbms_mview.refresh('SALES_REP_MV','F');
```

快速检查物化视图会显示两条新记录：

```sql
SQL> select sales_amt, to_char(sales_dtt,'dd-mon-yyyy')
from sales_rep_mv
order by 2;
```

示例输出如下：

```
SALES_AMT TO_CHAR(SALES_DTT,'D
---------- --------------------
511 10-jan-2013
101 20-jan-2013
127 23-jan-2013
99 24-jan-2013
127 30-jan-2013
99 30-jan-2013
11 31-dec-2012
```

此外，`MLOG$` 的计数已降为零。物化视图刷新完成后，就不再需要这些记录了：

```sql
SQL> select count(*) from mlog$_sales;
```

输出如下：

```
COUNT(*)
----------
0
```

## 验证刷新方法

您可以通过查询 `USER_MVIEWS` 视图来验证物化视图最近一次的刷新方法：

```sql
SQL> select mview_name, last_refresh_type, last_refresh_date
from user_mviews
order by 1,3;
```

示例输出如下：

```
MVIEW_NAME                LAST_REF LAST_REFR
------------------------- -------- ---------
SALES_REP_MV              FAST     30-JAN-13
```

## 架构流程

图 15-2 展示了物化视图新的架构组件；再次强调，理解这些组件很重要，请在此处暂停几分钟。

![../images/214899_3_En_15_Chapter/214899_3_En_15_Fig2_HTML.jpg](img/214899_3_En_15_Fig2_HTML.jpg)

**图 15-2** 可快速刷新物化视图的架构组件

图中的数字描述了可快速刷新物化视图的数据流：

1.  用户创建事务。
2.  数据在基础表中提交。
3.  基础表上的内部触发器填充物化视图日志表。
4.  通过 `DBMS_MVIEW` 包发起快速刷新。
5.  自上次刷新以来产生的 DML 更改被应用到物化视图。物化视图不再需要的行从物化视图日志中删除。
6.  用户可以从物化视图查询数据，其中包含基础表数据的时间点快照。

当您很好地理解了快速刷新的架构后，学习高级物化视图概念就不会有困难。如果这是您第一次接触物化视图，重要的是要认识到物化视图的数据存储在一个常规的数据库表中。这将帮助您从架构上理解什么是可能的，什么是不可能的。在很大程度上，由于物化视图和物化视图日志是基于表的，大多数适用于常规数据库表的特性也可以应用于物化视图表和物化视图日志表。例如，以下 Oracle 特性可轻松应用于物化视图：

*   存储和表空间放置
*   索引
*   分区
*   压缩
*   加密
*   日志记录
*   并行性

下一节将展示如何创建具有各种特性的物化视图的示例。

## 超越基础

有许多物化视图特性可用。许多特性与您可以应用于任何表的属性相关，例如存储、索引、压缩和加密。其他特性与创建的物化视图类型及其刷新方式相关。接下来的几个部分将描述这些特性。

## 创建物化视图并为物化视图和索引指定表空间

每个物化视图都有一个与之关联的基础表。此外，根据物化视图的类型，可能会自动创建一个索引。在创建物化视图时，您可以为基础表和索引指定表空间和存储特性。下一个示例展示了如何为物化视图表和索引指定要使用的表空间：

```sql
SQL> create materialized view sales_mv
tablespace users
using index tablespace users
refresh with primary key
fast on demand
as
select sales_id ,sales_amt, sales_dtt
from sales;
```

## 在物化视图上创建索引

物化视图将其数据存储在常规数据库表中。因此，您可以在基础表上创建索引（就像对任何其他表一样）。通常，遵循与在常规表上创建索引相同的准则（有关创建索引的更多详细信息，请参见第 8 章）。请记住，虽然索引可以显著提高查询性能，但维护索引（针对任何插入、更新和删除操作）会带来开销。索引也会消耗磁盘空间。

下面是一个在物化视图的列上创建索引的示例。语法与在常规表上创建索引相同：

```sql
SQL> create index sales_mv_idx1 on sales_mv(sales_dtt) tablespace users;
```

您可以通过查询 `USER_INDEXES` 视图来显示为物化视图创建的索引：

```sql
SQL> select a.table_name, a.index_name
from user_indexes a
,user_mviews  b
where a.table_name = b.mview_name;
```

**注意**

如果您使用 `WITH PRIMARY KEY` 子句创建一个简单的物化视图，该视图从一个具有主键的基础表中选择，Oracle 会自动在物化视图中相应的主键列上创建一个索引。如果您使用 `WITH ROWID` 子句创建一个简单的物化视图，该视图从一个具有主键的基础表中选择，Oracle 会自动在名为 `M_ROW$$` 的隐藏列上创建一个名为 `I_SNAP$_<table_name>` 的索引。

## 分区物化视图

您可以像对数据库中的任何其他表一样对物化视图表进行分区。如果您处理大型物化视图，可能需要考虑分区以便更好地管理和维护大型表。在创建物化视图时使用 `PARTITION` 子句。此示例构建了一个按 `SALES_ID` 进行哈希分区的物化视图：

```sql
SQL> create materialized view sales_mv
partition by hash (sales_id)
partitions 4
refresh on demand complete with rowid
as
select sales_id, sales_amt, region_id, sales_dtt
from sales;
```

## 查询结果分区存储

查询的结果集存储在一个分区表中。您可以像查看数据库中任何其他分区表一样，在 `USER_TAB_PARTITIONS` 和 `USER_PART_TABLES` 中查看此表的分区详细信息。有关分区策略和维护的更多详细信息，请参见第 12 章。

## 压缩物化视图

如前所述，当您创建物化视图时，会创建一个底层表来存储数据。因为此表是一个常规数据库表，所以您可以实现诸如压缩之类的功能；例如，

```
SQL> create materialized view sales_mv
compress
as
select sales_id, sales_amt
from sales;
```

您可以使用以下查询确认压缩详细信息：

```
SQL> select table_name, compression, compress_for
from user_tables
where table_name='SALES_MV';
```

输出如下：

```
TABLE_NAME                     COMPRESS  COMPRESS_FOR
------------------------------ --------  ------------
SALES_MV                       ENABLED   BASIC
```

> **注意**
> 基本表压缩不需要 Oracle 的额外许可，而 `ROW STORE COMPRESS ADVANCED` 压缩（12c 之前；通过 `COMPRESS FOR OLTP` 启用）则需要高级压缩选项，这确实需要 Oracle 的额外许可。有关详细信息，请参阅 Oracle 数据库许可信息，可从 Oracle 网站（[`http://otn.oracle.com`](http://otn.oracle.com)）的技术网络区域获取。

## 加密物化视图列

如前所述，当您创建物化视图时，会创建一个底层表来存储数据。因为此表是一个常规数据库表，所以您可以实现诸如列加密之类的功能；例如，

```
SQL> create materialized view sales_mv
(sales_id   encrypt no salt
,sales_amt  encrypt)
as
select
sales_id
,sales_amt
from sales;
```

要使前一条语句生效，您必须为数据库创建并打开一个安全钱包。此功能需要 Oracle 的高级安全选项。由于物化视图存储在表空间中，也可以在表空间级别进行加密，而不仅仅是在列级别。

您可以通过描述物化视图来验证加密是否到位：

```
SQL> desc sales_mv
Name                        Null?    Type
----------------------------------- ----------------------------
SALES_ID                   NOT NULL NUMBER ENCRYPT
SALES_AMT                           NUMBER ENCRYPT
```

## 启用 Oracle 钱包

Oracle 钱包是 Oracle 用于实现加密的机制。钱包是一个包含加密密钥的操作系统文件。钱包通过以下步骤启用：

1.  修改 `SQLNET.ORA` 文件以包含钱包的位置：

```
    ENCRYPTION_WALLET_LOCATION=
    (SOURCE=(METHOD=FILE) (METHOD_DATA=
    (DIRECTORY=/ora01/app/oracle/product/18.1.0.1/db_1/network/admin)))
```

2.  使用 `ALTER SYSTEM` 命令创建钱包文件（`ewallet.p18`）：

```
    SQL> alter system set encryption key identified by foo;
```

3.  启用加密：

```
    SQL> alter system set encryption wallet open identified by foo;
```

有关实现加密的完整详细信息，请参阅 Oracle 高级安全管理员指南，该指南可从 Oracle 网站（[`http://otn.oracle.com`](http://otn.oracle.com)）的技术网络区域免费下载。

## 在预建表上构建物化视图

在数据仓库环境中，有时您需要创建一个表，用大量数据填充它，然后将其转换为物化视图。或者，您可能正在复制一个大表，并发现最初使用 Data Pump 通过预建表填充远程物化视图更有效。以下是在预建表上构建物化视图的步骤：

1.  创建一个表。
2.  用数据填充它。
3.  在步骤 1 中创建的表上创建物化视图。

这里有一个简单的例子来说明这个过程。首先，创建一个表：

```
SQL> create table sales_mv
(sales_id    number
,sales_amt number);
```

现在，用数据填充该表。例如，在数据仓库环境中，可以使用 Data Pump、SQL*Loader 或外部表加载表。

最后，运行 `CREATE MATERIALIZED VIEW...ON PREBUILT TABLE` 语句将表变为物化视图。物化视图名称和表名称必须相同。此外，查询中的每个列必须对应表中的一个列；例如，

```
SQL> create materialized view sales_mv
on prebuilt table
using index tablespace users
as
select sales_id, sales_amt
from sales;
```

现在，`SALES_MV` 对象是一个物化视图。如果您尝试删除 `SALES_MV` 表，将抛出以下错误，表明 `SALES_MV` 现在是一个物化视图：

```
SQL> drop table sales_mv;
ORA-12083: must use DROP MATERIALIZED VIEW to drop "MV_MAINT"."SALES_MV"
```

预建表功能在数据仓库环境中非常有用，这些环境中通常有很长一段时间基础表没有被主动更新。这使您有时间加载预建表并确保其内容与基础表完全相同。在预建表上创建物化视图后，您可以快速刷新该物化视图，使其与基础表保持同步。
如果您的基础表（在物化视图的 `SELECT` 子句中指定）正在持续更新，那么在预建表上创建物化视图可能不是一个可行的选择。这是因为无法确保预建表与基础表保持同步。

> **注意**
> 对于在预建表上创建的物化视图，如果您随后发出 `DROP MATERIALIZED VIEW` 语句，底层表不会被删除。当您需要修改基础表（例如添加列）时，这有一些有趣的影响。有关详细信息，请参阅本章后面的“修改基础表 DDL 并传播到物化视图”部分。

## 创建未填充的物化视图

创建物化视图时，您可以选择指示 Oracle 是否最初用数据填充该物化视图。例如，如果最初构建物化视图需要几个小时，您可能希望首先定义物化视图，然后将其作为单独的作业进行填充。

此示例使用 `BUILD DEFERRED` 子句指示 Oracle 不要最初用查询结果填充物化视图：

```
SQL> create materialized view sales_mv
tablespace users
build deferred
refresh complete on demand
as
select sales_id, sales_amt
from sales;
```

此时，查询物化视图将返回零行。在稍后的某个时间点，您可以启动完全刷新以用数据填充物化视图。

## 创建在提交时刷新的物化视图

当主表中的数据被修改时，您可能需要立即将它们复制到物化视图。在这种情况下，在创建物化视图时使用 `ON COMMIT` 子句。主表上必须创建物化视图日志，此技术才能工作：

```
SQL> create materialized view log on sales with primary key;
```

接下来，创建一个在提交时刷新的物化视图：

```
SQL> create materialized view sales_mv
refresh
on commit
as
select sales_id, sales_amt from sales;
```

当在主表中插入并提交数据时，任何更改也会在由物化视图查询选择的物化视图中可用。

`ON COMMIT` 可刷新物化视图有一些限制您需要注意：

*   主表和物化视图必须在同一数据库中。
*   您不能在基础表上执行分布式事务。
*   此方法不适用于包含对象类型或 Oracle 提供类型的物化视图。

同时还要考虑在两个地方同时提交数据的相关开销；这会影响高事务 OLTP 系统的性能。此外，如果在更新物化视图时出现任何问题，则基础表无法提交事务。例如，如果创建物化视图的表空间已满（并且无法再分配另一个区），当您尝试插入基础表时会看到如下错误：


