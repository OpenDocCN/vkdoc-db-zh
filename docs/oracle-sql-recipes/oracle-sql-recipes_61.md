# 执行计划

## 19-1. 实时监控 SQL 执行统计信息

### 问题

你希望实时识别消耗资源最多的 SQL 语句。

### 解决方案

如果你的数据库是 Oracle Database 11g，你可以使用以下查询，从`V$SQL_MONITOR`中进行选择，以监控 SQL 查询的近实时资源消耗情况：

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 19 章 ■ SQL 监控与调优

```sql
select * from (
select
    a.sid session_id
    ,a.sql_id
    ,a.status
    ,a.cpu_time/1000000 cpu_sec
    ,a.buffer_gets
    ,a.disk_reads
    ,b.sql_text sql_text
from v$sql_monitor a
    ,v$sql b
where a.sql_id = b.sql_id
order by a.cpu_time desc)
where rownum <=20;
```

该查询的输出结果不容易在一页内完整显示。以下是输出的一个子集：

```
SESSION_ID SQL_ID        STATUS    CPU_SEC BUFFER_GETS DISK_READS SQL_TEXT
---------- ------------- --------- -------- ----------- ---------- ------------
       194 4k8runghhh31d DONE      7        8.66        0          select count(*)
       191 9tjq6dfrwgzfr EXECUTING 45.57    1944        7          select output f
```

在该查询中，使用了一个内联视图首先检索所有记录，并按`CPU_TIME`降序排列。然后，外部查询使用`ROWNUM`伪列将结果集限制为前二十行。你可以修改前面的查询，按你选择的统计信息排序。也可以修改它，只显示当前正在执行的查询。例如，下面的 SQL 语句按磁盘读取次数排序监控正在执行的查询：

```sql
select * from (
select
    a.sid session_id
    ,a.sql_id
    ,a.status
    ,a.cpu_time/1000000 cpu_sec
    ,a.buffer_gets
    ,a.disk_reads
    ,substr(b.sql_text,1,15) sql_text
from v$sql_monitor a
    ,v$sql b
where a.sql_id = b.sql_id
    and a.status='EXECUTING'
order by a.disk_reads desc)
where rownum <=20;
```

`注意` 如果你使用的是 Oracle Database 10g 或更早版本，请参阅配方 19-4 以显示消耗资源的查询。

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 19 章 ■ SQL 监控与调优

### 工作原理

`V$SQL_MONITOR`中的统计信息每秒更新一次，因此你可以查看资源消耗的实时变化情况。默认情况下，如果 SQL 语句并行运行或消耗超过 5 秒的 CPU 或 I/O 时间，就会收集这些统计信息。

`V$SQL_MONITOR`视图包含`V$SQL`、`V$SQLAREA`和`V$SQLSTATS`视图中统计信息的一个子集。`V$SQL_MONITOR`视图显示每次执行资源密集型 SQL 语句的实时统计信息，而`V$SQL`、`V$SQLAREA`和`V$SQLSTATS`则包含 SQL 语句多次执行的累计统计信息集。

一旦 SQL 语句执行结束，其运行时统计信息不会立即从`V$SQL_MONITOR`中清除。根据数据库的活动情况，这些统计信息可能会保留一段时间。如果你的数据库非常活跃，统计信息可能在查询完成后不久就被清除。

`提示` 你可以通过组合以下列来唯一标识`V$SQL_MONITOR`中 SQL 语句的一次执行：`SQL_ID`、`SQL_EXEC_START`、`SQL_EXEC_ID`。

## 19-2. 显示查询在执行计划中的进度

### 问题

你想确切查看 Oracle 在 SQL 执行计划中的哪个步骤花费了时间。你使用的是 Oracle Database 11g。

### 解决方案

如果你使用的是 Oracle Database 11g，你可以在 SQL 查询运行时，观察其执行计划的进度。`V$SQL_PLAN_MONITOR`视图包含 SQL 语句执行计划中每个步骤的一行记录。

此查询将在查询运行时显示执行计划中每个步骤使用的行数和内存量。你需要使用一些 SQL*Plus 格式化命令来使输出可读：

```sql
COL SID FORMAT 99999
COL status FORMAT A15
COL start_time FORMAT A12
COL plan_line_id FORMAT 99999 HEAD "Plan ID"
COL plan_options FORMAT A16
COL mem_bytes FORMAT 99999999
COL temp_bytes FORMAT 99999999
SET LINESIZE 132 PAGESI 100 TRIMSP ON
BREAK ON sid on status on start_time NODUP SKIP 1
--
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 19 章 ■ SQL 监控与调优

```sql
select
```


