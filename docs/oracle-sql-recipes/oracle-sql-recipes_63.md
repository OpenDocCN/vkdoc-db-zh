# 第 19 章 SQL 监控与调优

##### 19-4. 识别资源密集型 SQL 语句

### 问题
您希望识别哪些 SQL 语句消耗了最多的资源。

### 解决方案
使用以下查询，基于 CPU 时间识别出最消耗资源的前十个查询：

```sql
select * from(
    select
        sql_text
        ,buffer_gets
        ,disk_reads
        ,sorts
        ,cpu_time/1000000 cpu_sec
        ,executions
        ,rows_processed
    from v$sqlstats
    order by cpu_time DESC)
where rownum < 11;
```

这里使用了一个内联视图，先检索所有记录，并按 `CPU_TIME` 降序对输出进行排序。然后外层查询使用 `ROWNUM` 伪列将结果集限制为前 10 行。

这个查询可以轻松修改为按 `CPU_TIME` 以外的列排序。例如，如果您想按 `BUFFER_GETS` 报告资源使用情况，只需将 `ORDER BY` 子句改为使用 `BUFFER_GETS` 而不是 `CPU_TIME`。`CPU_TIME` 列以微秒计算；要将其转换为秒，需除以 1000000。

### 工作原理
`V$SQLSTATS` 视图显示最近执行过的 SQL 语句的性能统计信息。您也可以使用 `V$SQL` 和 `V$SQLAREA` 来报告 SQL 资源使用情况。`V$SQLSTATS` 速度更快，并且信息保留时间更长，但它只包含了 `V$SQL` 和 `V$SQLAREA` 中列的子集。因此，在某些情况下您可能需要查询 `V$SQL` 或 `V$SQLAREA`。例如，如果您想显示首次解析查询的用户等信息，可以使用 `V$SQLAREA` 的 `PARSING_USER_ID` 列：

```sql
select * from(
    select
        b.sql_text
        ,a.username
        ,b.buffer_gets
        ,b.disk_reads
        ,b.sorts
        ,b.cpu_time/1000000 cpu_sec
    from v$sqlarea b
        ,dba_users a
    where b.parsing_user_id = a.user_id
    order by b.cpu_time DESC)
where rownum < 11;
```

##### 19-5. 使用 Oracle 性能报告识别资源密集型 SQL

### 问题
您希望使用 Oracle 的性能调优报告之一来识别资源密集型查询。

### 解决方案
如果您注意到数据库性能在某一天变慢或几个小时内运行迟缓，您应在发现问题的时段运行以下报告之一：

*   自动工作量存储库（`AWR`）
*   自动数据库诊断监视器（`ADDM`）
*   活动会话历史（`ASH`）
*   Statspack

接下来的几个小节将分别描述这些报告。

**注意** 截至本书撰写时，您需要从 Oracle 公司购买额外许可证才能使用 `AWR`、`ADDM` 和 `ASH` 工具。如果您没有许可证，可以使用免费的 Statspack 工具。

### 使用 AWR
要监控长时间运行的操作，您可以查询 `V$SESSION_LONGOPS` 视图。以下示例展示了针对该视图的查询，并使用了 `SQL*Plus` 格式化命令以生成可读的报告。

```sql
COL opname FORMAT A16 HEAD "Operation|Type"
COL sql_text FORMAT A33 HEAD "SQL|Text" TRUNC
COL start_time FORMAT A15 HEAD "Start|Time"
COL how_long FORMAT 99,990 HEAD "Time|Run"
COL secs_left FORMAT 99,990 HEAD "Appr.|Secs Left"
COL sofar FORMAT 9,999,990 HEAD "Work|Done"
COL totalwork FORMAT 9,999,990 HEAD "Total|Work"
COL percent FORMAT 999.90 HEAD "%|Done"

select
    a.username
    ,a.opname
    ,b.sql_text
    ,to_char(a.start_time,'DD-MON-YY HH24:MI') start_time
    ,a.elapsed_seconds how_long
    ,a.time_remaining secs_left
    ,a.sofar
    ,a.totalwork
    ,round(a.sofar/a.totalwork*100,2) percent
from v$session_longops a
    ,v$sql b
where a.sql_address = b.address
and a.sql_hash_value = b.hash_value
and a.sofar <> a.totalwork
and a.totalwork != 0;
```

#### 工作原理
`V$SESSION_LONGOPS` 视图显示运行时间超过六秒的各种数据库操作的状态。要使用 `V$SESSION_LONGOPS`，您必须使用基于成本的优化器，并满足以下条件：

*   `TIMED_STATISTICS` 初始化参数设置为 `TRUE`
*   为查询中的对象生成了统计信息

`V$SESSION_LONGOPS` 视图能让您大致了解当前正在运行的查询还需要多长时间完成。根据我们的经验，结果并非 100%准确，因此请在其报告的时间基础上预留一些弹性时间。


