# 物化视图日志管理

## 检测 MV 日志是否被清理

检测 MV 日志未被清理的一种方法是定期检查 MV 日志表的行数。以下查询使用 SQL 生成一个用于检查当前连接用户所拥有的 MV 日志表行数的脚本：

```sql
SQL> set head off pages 0 lines 132 trimspool on
SQL> spo mvcount_dyn.sql
SQL> select 'select count(*) || ' || "'||: '" || table_name || "'" || ' from ' || table_name || ';'
from user_tables
where table_name like 'MLOG%';
SQL> spo off;
```

此脚本生成一个名为`mvcount_dyn.sql`的文件，其中包含从`MLOG$`表中选择行数的 SQL 语句。检查行数时，你必须对应用程序有一定了解，并清楚正常的行数应该是多少。以下是前面脚本生成的一些示例代码：

```sql
SQL> select count(*) || ': MLOG$_SALES' from MLOG$_SALES;
SQL> select count(*) || ': MLOG$_REGION' from MLOG$_REGION;
```

## 移动 MV 日志

你可能需要移动 MV 日志，因为初始创建脚本未指定正确的表空间。一种常见情况是未指定表空间，MV 日志默认被放置在诸如`USERS`之类的表空间中。你可以通过以下查询验证表空间信息：

```sql
SQL> select table_name, tablespace_name
from user_tables
where table_name like 'MLOG%';
```

如果任何 MV 日志表需要重新定位，请使用`ALTER MATERIALIZED VIEW LOG ON <table_name> MOVE`语句。请注意，你需要指定在其上创建 MV 的主表名称（而不是底层的`MLOG$`表）：

```sql
SQL> alter materialized view log on sales move tablespace users;
```

同时请记住，移动表时，所有关联的索引都会变得不可用（因为表中每条记录的`ROWID`刚刚发生了变化）。你可以按如下方式检查索引状态：

```sql
SQL> select a.table_name, a.index_name, a.status
from user_indexes a
,user_mview_logs b
where a.table_name = b.log_table;
```

任何不可用的索引都必须重建。以下是重建索引的示例：

```sql
SQL> alter index mlog$_sales_idx2 rebuild;
```

## 删除 MV 日志

你可能出于以下几个原因想要删除 MV 日志：

*   你最初创建了一个 MV 日志，但需求发生了变化，你不再需要它。
*   MV 日志变得很大并导致性能问题，你想通过删除它来重置大小。

在删除 MV 日志之前，你可以通过以下查询验证所有者、主表和 MV 日志表：

```sql
SQL> select
log_owner
,master     -- master table
,log_table
from user_mview_logs;
```

使用`DROP MATERIALIZED VIEW LOG ON`语句来删除 MV 日志。你不需要知道 MV 日志的名称，但需要知道在其上创建日志的主表名称。此示例删除`SALES`表上的 MV 日志：

```sql
SQL> drop materialized view log on sales;
```

如果成功，你将看到以下消息：

```sql
Materialized view log dropped.
```

如果你有权限，并且不是创建 MV 日志的表的所有者，可以在删除 MV 日志时指定模式名称：

```sql
SQL> drop materialized view log on .;
```

如果你想清理环境并删除与某个用户关联的所有 MV 日志，可以使用 SQL 生成相应的 SQL 来实现。以下脚本创建用于删除当前连接用户拥有的所有 MV 日志所需的 SQL：

```sql
SQL> set lines 132 pages 0 head off trimspool on
SQL> spo drop_dyn.sql
SQL> select 'drop materialized view log on ' || master || ';'
from user_mview_logs;
SQL> spo off;
```

前面的 SQL*Plus 代码创建了一个名为`drop_dyn.sql`的脚本，其中包含可用于删除用户所有 MV 日志的 SQL 语句。

## 刷新 MV

通常，你会定期刷新 MV。你可以手动刷新 MV 或自动化此任务。以下部分涵盖了这些相关主题：

*   从 SQL*Plus 手动刷新 MV
*   使用 shell 脚本和调度工具自动化刷新
*   使用内置的 Oracle 作业调度器自动化刷新

> **注意：** 如果你需要将一组 MV 作为一个集合进行刷新，请参阅本章后面的“管理组中的 MV”部分。

### 从 SQL*Plus 手动刷新 MV

你需要定期刷新 MV，以使其与基础表同步。为此，可以使用 SQL*Plus 调用`DBMS_MVIEW`包的`REFRESH`过程。该过程接受两个参数：MV 名称和刷新方法。此示例使用`EXEC[UTE]`语句调用该过程。正在刷新的 MV 是`SALES_MV`，刷新方法是`F`（快速）：

```sql
SQL> exec dbms_mview.refresh('SALES_MV','F');
```

你也可以从 SQL*Plus 手动运行刷新，使用一个 PL/SQL 匿名块。此示例执行快速刷新：

```sql
SQL> begin
dbms_mview.refresh('SALES_MV','F');
end;
/
```

此外，你可以使用问号（`?`）来调用强制刷新方法。这指示 Oracle 在可能的情况下执行快速刷新。如果无法进行快速刷新，则 Oracle 将执行完全刷新：

```sql
SQL> exec dbms_mview.refresh('SALES_MV','?');
```

你也可以使用`C`（完全）来明确执行完全刷新方法：

```sql
SQL> exec dbms_mview.refresh('SALES_MV','C');
```

## MV 与结果缓存

Oracle 有一个结果缓存功能，它将查询的结果存储在内存中，并使该结果集可用于后续发出的任何相同查询。如果发出了后续相同的查询，并且自原始查询发出以来基础表数据未发生变化，Oracle 会将结果提供给后续查询。对于数据相对静态且发出许多相同查询的数据库，使用结果缓存可以显著提高性能。

MV 与结果缓存相比如何？回想一下，MV 将查询的结果存储在表中，并使该结果可用于报告应用程序。这两个功能听起来相似，但在几个重要方面有所不同：

1.  结果缓存将结果存储在内存中。MV 将结果存储在表中。
2.  表中的基础数据发生变化时，结果缓存需要立即刷新。MV 在提交时或在定期间隔（例如每天）刷新。因此，如果未刷新，MV 可能拥有过时的数据。

如果你有在相对静态数据上运行的长时间查询，结果缓存可以显著提高性能。MV 更适合复制数据和存储仅在定期基础上（如每天、每周或每月）需要新结果的复杂查询的结果。

## 创建带刷新间隔的 MV

最初创建 MV 时，你可以选择指定`START WITH`和`NEXT`子句，指示 Oracle 设置一个内部数据库作业（通过`DBMS_SCHEDULER`包）以定期启动 MV 的刷新。`START WITH`参数指定你希望 MV 首次刷新发生的日期。`NEXT`参数指定一个日期表达式，Oracle 用它来计算刷新之间的间隔。例如，此 MV 在未来的 1 分钟（`sysdate+1/1440`）首次刷新，随后每天刷新（`sysdate+1`）：

```sql
SQL> create materialized view sales_mv
refresh
with primary key
fast on demand
start with sysdate+1/1440
next sysdate+1
as
select sales_id, sales_amt, sales_dtt
from sales;
```

你可以通过查询`USER_JOBS`来查看已调度作业的详细信息：

```sql
SQL> select job, schema_user
,to_char(last_date,'dd-mon-yyyy hh24:mi:ss') last_date
,to_char(next_date,'dd-mon-yyyy hh24:mi:ss') next_date
,interval, broken
from user_jobs;
```

以下是一些示例输出：


