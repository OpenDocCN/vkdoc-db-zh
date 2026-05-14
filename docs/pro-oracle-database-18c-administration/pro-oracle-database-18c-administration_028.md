# 物化视图高级主题

## 错误处理与注意事项

```
ORA-12008: error in materialized view refresh path
ORA-01653: unable to extend table MV_MAINT.SALES_MV by 16 in tablespace...
```

基于这些原因，只有当您确信它不会影响性能或可用性时，才应使用此功能。

## 注意
您不能同时为物化视图指定 `ON COMMIT` 和 `ON DEMAND` 刷新。此外，`ON COMMIT` 与 `CREATE MATERIALIZED VIEW` 语句中的 `START WITH` 和 `NEXT` 子句不兼容。

## 创建永不刷新的物化视图

您可能永远不希望某个物化视图被刷新。例如，您可能希望为了审计目的，保证拥有某个时间点上的表快照。在创建物化视图时指定 `NEVER REFRESH` 子句即可实现：

```
SQL> create materialized view sales_mv
never refresh
as
select sales_id, sales_amt
from sales;
```

如果您尝试刷新一个设置为永不刷新的物化视图，将会收到此错误：

```
ORA-23538: cannot explicitly refresh a NEVER REFRESH materialized view
```

您可以将永不刷新的视图更改为可刷新。使用 `ALTER MATERIALIZED VIEW` 语句来完成：

```
SQL> alter materialized view sales_mv refresh on demand complete;
```

### 验证刷新模式与方法
您可以使用以下查询来验证刷新模式与方法：

```
SQL> select mview_name, refresh_mode, refresh_method from user_mviews;
```

## 为查询重写创建物化视图

查询重写允许优化器识别出可以使用物化视图来满足查询需求，而无需使用底层的主表（基础表）。如果您所处的环境中用户经常编写自己的查询，并且不了解可用的物化视图，此功能可以极大地提升性能。

### 启用查询重写的先决条件
启用查询重写有三个先决条件：

- Oracle 企业版
- 将数据库初始化参数 `QUERY_REWRITE_ENABLED` 设置为 `TRUE`
- 物化视图在创建或修改时使用了 `ENABLE QUERY REWRITE` 子句

此示例创建了一个启用查询重写的物化视图：

```
SQL> create materialized view sales_daily_mv
segment creation immediate
refresh
complete
on demand
enable query rewrite
as
select
sum(sales_amt) sales_amt
,trunc(sales_dtt) sales_dtt
from sales
group by trunc(sales_dtt);
```

### 验证查询重写
您可以通过 autotrace 工具检查查询的执行计划来验证查询重写是否生效：

```
SQL> set autotrace trace explain
```

现在，假设一个用户运行了以下查询，并不知道已经存在一个物化视图聚合了所需数据：

```
SQL> select
sum(sales_amt) sales_amt
,trunc(sales_dtt) sales_dtt
from sales
group by trunc(sales_dtt);
```

以下是 autotrace 输出的部分内容，验证了查询重写正在使用中：

```
| Id  | Operation                    | Name           | Cost (%CPU)| Time    |
|   0 | SELECT STATEMENT             |                |     3   (0)| 00:00:01 |
|   1 |  MAT_VIEW REWRITE ACCESS FULL| SALES_DAILY_MV |     3   (0)| 00:00:01 |
```

从前面的输出可以看出，尽管用户直接从 `SALES` 表中选择，但优化器判定通过访问物化视图可以更高效地满足查询结果。

### 检查物化视图的查询重写状态
您可以通过查询 `USER_MVIEWS` 中的 `REWRITE_ENABLED` 列来判断物化视图是否启用了查询重写：

```
SQL> select mview_name, rewrite_enabled, rewrite_capability
from user_mviews
where mview_name = 'SALES_DAILY_MV';
```

### 诊断问题
如果出于任何原因，某个查询没有使用查询重写功能，而您认为它应该使用，请使用 `DBMS_MVIEW` 包的 `EXPLAIN_REWRITE` 过程来诊断问题。

## 创建基于复杂查询的快速刷新物化视图

在许多情况下，当您基于连接多个表的查询创建物化视图时，它会被视为复杂物化视图，因此只能进行完全刷新。然而，在某些场景下，当物化视图查询中引用了两个连接在一起的表时，您可以创建快速刷新的物化视图。

本节介绍如何使用 `DBMS_MVIEW` 包的 `EXPLAIN_MVIEW` 过程来确定是否可以对复杂查询进行快速刷新。

### 准备基础表与物化视图日志
为了帮助您完全理解此示例，本节展示了用于创建基础表的 SQL。假设有两个基础表，定义如下：

```
SQL> create table region(
region_id number
,reg_desc varchar2(30)
,constraint region_pk primary key(region_id));
--
SQL> create table sales(
sales_id  number
,sales_amt number
,region_id number
,sales_dtt date
,constraint sales_pk primary key(sales_id)
,constraint sales_fk1 foreign key (region_id) references region(region_id));
```

此外，已为 `REGION` 和 `SALES` 创建了物化视图日志，如下所示：

```
SQL> create materialized view log on region with primary key;
SQL> create materialized view log on sales with primary key;
```

同时，为此示例，基础表中插入了以下数据：

```
SQL> insert into region values(10,'East');
SQL> insert into region values(20,'West');
SQL> insert into region values(30,'South');
SQL> insert into region values(40,'North');
--
SQL> insert into sales values(1,100,10,sysdate);
SQL> insert into sales values(2,200,20,sysdate-20);
SQL> insert into sales values(3,300,30,sysdate-30);
```

### 创建并测试物化视图
假设您想创建一个连接 `REGION` 和 `SALES` 基础表的物化视图，如下所示：

```
SQL> create materialized view sales_mv
as
select
a.sales_id
,b.reg_desc
from sales  a
,region b
where a.region_id = b.region_id;
```

接下来，让我们尝试快速刷新该物化视图：

```
SQL> exec dbms_mview.refresh('SALES_MV','F');
```

此时会抛出此错误：

```
ORA-12032: cannot use rowid column from materialized view log...
```

该错误表明物化视图存在问题，无法进行快速刷新。

### 使用 EXPLAIN_MVIEW 诊断问题
要确定此物化视图是否可以变得可快速刷新，请使用 `DBMS_MVIEW` 包的 `EXPLAIN_MVIEW` 过程的输出。此过程要求您首先创建一个 `MV_CAPABILITIES_TABLE`。Oracle 提供了一个脚本来完成此操作。请以物化视图所有者的身份运行此脚本：

```
SQL> @?/rdbms/admin/utlxmv.sql
```

创建表后，运行 `EXPLAIN_MVIEW` 过程来填充它：

```
SQL> exec dbms_mview.explain_mview(mv=>'SALES_MV',stmt_id=>'100');
```

现在，查询 `MV_CAPABILITIES_TABLE` 以查看此物化视图可能存在哪些潜在问题：

```
SQL> select capability_name, possible, msgtxt, related_text
from mv_capabilities_table
where capability_name like 'REFRESH_FAST_AFTER%'
and statement_id = '100'
order by 1;
```

接下来是输出的部分列表。`P`（代表可能）列对于每个快速刷新的可能性都包含一个 `N`（代表否）：

```
CAPABILITY_NAME            P  MSGTXT                        RELATED_TEXT
-------------------------  -  ----------------------------  ---------------
REFRESH_FAST_AFTER_INSERT  N  the SELECT list does not have  B
the rowids of all the detail tables
REFRESH_FAST_AFTER_INSERT  N  mv log must have ROWID        MV_MAINT.REGION
REFRESH_FAST_AFTER_INSERT  N  mv log must have ROWID        MV_MAINT.SALES
```

`MSGTXT` 指出了问题：物化视图日志需要基于 `ROWID`，并且表的 `ROWID` 必须出现在 `SELECT` 子句中。

### 重新创建物化视图与日志
因此，首先删除并使用 `ROWID`（而不是主键）重新创建物化视图日志：

```
SQL> drop materialized view log on region;
SQL> drop materialized view log on sales;
--
SQL> create materialized view log on region with rowid;
SQL> create materialized view log on sales with rowid;
--
SQL> drop materialized view sales_mv;
--
SQL> create materialized view sales_mv
as
select
a.rowid sales_rowid
,b.rowid region_rowid
,a.sales_id
,b.reg_desc
from sales  a
,region b
where a.region_id = b.region_id;
```

接下来，重置 `MV_CAPABILITIES_TABLE`，并通过 `EXPLAIN_MVIEW` 过程重新填充它：

```
SQL> delete from mv_capabilities_table where statement_id=100;
SQL> exec dbms_mview.explain_mview(mv=>'SALES_MV',stmt_id=>'100');
```

输出显示，现在可以快速刷新该物化视图了。



