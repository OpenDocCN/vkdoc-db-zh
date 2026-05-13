# 第 19 章 SQL 监控与调优

### 表 19-3 最常见的访问路径

| 访问路径 | 描述 |
|----------|------|
| 全表扫描 | 读取表中的所有行，当需要读取表的大部分行时，这种方式效率较高。当只需要读取表的一小部分数据时，可以创建索引来避免全表扫描。 |
| ROWID 扫描 | 通过`ROWID`定位一行。`ROWID`包含该行在数据文件和块中的物理位置，当检索的行数相对较少时效率高，当读取表的大部分数据时效率较低。 |
| 索引扫描 | 索引包含键值和一个`ROWID`。如果查询只需要索引键值中包含的信息，`Oracle`直接返回这些值；否则，它会使用`ROWID`从表行中检索额外的数据。 |

`连接方法`是优化器用于从多个表中检索数据的技术。连接语句源于在`WHERE`子句中连接的表。表 19-4 概述了各种连接方法。优化器必须为每个连接语句计算以下内容：

- 返回连接中每个表的行所使用的访问路径
- 连接方法
- 当连接涉及两个以上表时的连接顺序

### 表 19-4 连接方法

| 连接方法 | 描述 |
|----------|------|
| 嵌套循环 | 当连接少量数据子集，并且对连接中第二个表（内表）有唯一索引或高度选择性的索引访问时非常有用。 |
| 哈希连接 | 对于连接大型数据集效率高。在内存中基于较小表或结果集的连接键构建一个哈希表。扫描较大的表，并使用哈希表来检索行。 |
```

# 第 19 章 SQL 监控与调优

## 排序合并连接
将来自两个独立表或结果集的行进行连接。当行已排序时，排序合并连接效率较高。哈希连接通常比排序合并连接更高效。

## 笛卡尔连接
在没有连接条件时使用。

## 外连接
返回其中一个表中不满足连接条件的行。

执行计划（有时也称为解释计划）是 DBA 和开发人员用于改进 SQL 查询性能的便捷工具。如果你刚开始使用执行计划，请遵循以下启发式方法入门：

-   确定查询需要运行的快慢程度。
-   确保已生成准确的统计信息（详情参见配方 19-13）。
-   确定 `WHERE` 子句中使用的列是否存在索引（查看表上创建的索引方法参见配方 10-5）。
-   关注执行计划中代价最高的步骤，并确定对操作所涉及的列创建索引是否可能降低代价（创建索引的详情参见配方 18-7）。
-   查询大表时，确定并行提示（hint）是否可能有所帮助。
-   确定查询（或其部分）是否可以重写以获得更好的性能。

执行计划的结果有时可能反直觉，因为优化器会判断使用索引代价过高。或者优化器可能判定全表扫描比使用索引更高效。

在开发查询时，有时分阶段编写并分析每个阶段后的执行计划是有用的。例如，你可能有一个连接五个表并在 `WHERE` 子句中有多个条件的查询。不要等到开发完整个查询才生成解释计划。

相反，在迭代地向 `WHERE` 子句添加表和条件时生成执行计划。

如果你在开启统计信息的情况下生成解释计划（通过 `SET AUTOTRACE ON STATISTICS`），你还会看到类似这样的输出：

```
Statistics
----------------------------------------------------------
	276  recursive calls
	  0  db block gets
	 51  consistent gets
	  7  physical reads
	  0  redo size
	583  bytes sent via SQL*Net to client
	524  bytes received via SQL*Net from client
	  2  SQL*Net roundtrips to/from client
	  4  sorts (memory)
	  0  sorts (disk)
	  3  rows processed
```

这些统计信息在表 19-5 中描述。一般来说，查询调优的目标之一是尝试减少查询所需的 `一致性读取` 次数。

### 表 19-5. 执行计划统计信息

| 统计项 | 描述 |
| :--- | :--- |
| `recursive calls` | Oracle 执行的初步工作，例如检查语法、语义、权限、查找数据字典信息等。 |
| `db block gets` | Oracle 需要数据块当前版本的次数。 |
| `consistent gets` | 读取数据块一致性版本的次数。一致性版本是 SQL 语句启动时的数据块状态。逻辑读取等于 `db block gets` 加上 `consistent gets`。 |
| `physical reads` | 因内存中未找到数据块而从磁盘读取的块数。 |
| `redo size` | Oracle 为 SQL 操作写入重做流的信息量。 |
| `bytes sent via SQL*Net to client` | 发送到客户端工具的字节数。 |
| `bytes received via SQL*Net from client` | 从客户端工具接收的字节数。 |
| `SQL*Net roundtrips to/from client` | 向客户端发送和从客户端接收数据的次数。 |
| `sorts (memory)` | 在内存中执行的排序操作次数。 |
| `sorts (disk)` | 在磁盘上执行的排序操作次数。 |
| `rows processed` | 查询返回的总行数。 |

##### 19-11. 获取 SQL 调优建议

### 问题
你希望获得关于如何改进问题查询性能的建议。

### 解决方案
使用 `SQL 调优顾问` 来显示关于如何提高查询性能的建议。


#### 第 19 章：SQL 监控与调优

**注意** 截至本书撰写时，SQL 调优顾问属于 Oracle 数据库调优包，需要额外付费许可。

请按以下步骤从调优顾问获取调优建议：

1.  以特权用户连接，并向将执行调优分析的模式授予以下权限：

   ```sql
   GRANT ADMINISTER SQL TUNING SET TO &&tune_user;
   GRANT ADVISOR TO &&tune_user;
   GRANT CREATE ANY SQL PROFILE TO &&tune_user;
   GRANT ALTER ANY SQL PROFILE TO &&tune_user;
   GRANT DROP ANY SQL PROFILE TO &&tune_user;
   ```

   如果您使用的是 Oracle Database 11g，前面的 `GRANT ... ANY SQL PROFILE` 语句可以替换为：

   ```sql
   GRANT ADMINISTER SQL MANAGEMENT OBJECT TO &&tune_user;
   ```

2.  使用 `DBMS_SQLTUNE` 包创建调优任务。修改以下代码以匹配您要尝试调优的查询以及您的用户名：

   ```sql
   DECLARE
     tune_task_name VARCHAR2(30);
     tune_sql CLOB;
   BEGIN
     tune_sql := 'select a.emp_name, b.dept_name from emp a, dept b';
     tune_task_name := DBMS_SQLTUNE.CREATE_TUNING_TASK(
       sql_text => tune_sql,
       user_name => 'STAR_APR',
       scope => 'COMPREHENSIVE',
       time_limit => 1800,
       task_name => 'tune1',
       description => 'Basic tuning example'
     );
   END;
   /
   ```

3.  通过查询 `USER_ADVISOR_LOG` 视图来验证您的调优任务是否存在：

   ```sql
   SQL> SELECT task_name FROM user_advisor_log WHERE task_name LIKE 'tune1';
   ```

4.  运行调优任务：

   ```sql
   SQL> EXEC DBMS_SQLTUNE.EXECUTE_TUNING_TASK(task_name=>'tune1');
   ```

5.  显示 SQL 调优顾问报告。运行以下 SQL 语句显示输出：

   ```sql
   SET LONG 10000
   SET LONGCHUNKSIZE 10000
   SET LINESIZE 132
   SET PAGESIZE 200
   --
   SELECT DBMS_SQLTUNE.REPORT_TUNING_TASK('tune1') FROM dual;
   ```

   以下是输出的片段：

   3- 重构 SQL 发现（参见解释计划部分中的计划 1）

   在执行计划的行 ID 1 处发现了一个昂贵的笛卡尔积操作。

   **建议**

   -   考虑从此语句中移除未关联的表或视图，或者添加一个引用它的连接条件。

   **原理**

   应尽可能避免笛卡尔积，因为它是一种昂贵的操作，并且可能产生大量数据。

   从前面的输出可以看出，SQL 调优顾问报告正确地建议重写查询，以避免使用笛卡尔连接。

**工作原理**

SQL 调优顾问有助于自动完成调优性能不佳的查询的任务。该工具相当易于使用，其调优输出报告提供了关于如何调优查询的详细建议。

如果您需要删除调优任务，可以使用 `DBMS_SQLTUNE` 包，如下所示：

```sql
SQL> EXEC DBMS_SQLTUNE.DROP_TUNING_TASK(task_name=>'tune1');
```

**注意** SQL 调优顾问在 Oracle Enterprise Manager 中也可用。如果您有 Enterprise Manager 的访问权限，可能会发现使用图形界面更容易。

##### 19-12. 对查询强制使用自己的执行计划

### 问题

您怀疑优化器没有使用可用的索引。您希望强制优化器使用该索引，并查看查询的成本是否降低。

### 解决方案

提示（hint）的语法是将注释 `/*+ <hint> */` 紧跟在 `SELECT` 关键字之后。以下代码使用 `INDEX` 提示来指示优化器在 `EMP` 表上使用 `EMP_IDX1` 索引：

```sql
SELECT /*+ INDEX (emp emp_idx1) */
  ename
FROM emp
WHERE ename='Chaya';
```

如果使用表别名，请确保在提示中引用的是别名，而不是表名。使用别名时，如果在提示中使用表名，该提示将被忽略。以下示例在全表扫描提示中引用了表别名：

```sql
SELECT /*+ FULL(a) */ ename
FROM emp a
WHERE ename='Heera';
```


# SQL 提示

## 重要注意事项

如果你拼错了提示（hint）的名称，或者在表别名或索引名称中打错了字，Oracle 将忽略该提示。你可以通过设置 `AUTOTRACE ON` 并查看执行计划来验证提示是否被使用。

请记住，在某些情况下 Oracle 会忽略提示。Oracle 可能判定提示指令在给定场景下不可行，因此拒绝你的建议。

## 工作原理

SQL 提示允许你在执行计划中覆盖优化器的选择。有时，因为你知道一些优化器不了解的环境信息（就像相机上的运动或风景设置），查询性能问题可以得到改善。

例如，假设你有一个数据仓库数据库使用系统范围的 `ALL_ROWS` 设置，但你也想针对该数据库运行一个 OLTP 报表查询，该查询使用 `FIRST_ROWS` 设置会更高效。在这种情况下，你可以提示优化器使用 `FIRST_ROWS` 而不是 `ALL_ROWS`。

## 使用建议

我们建议你谨慎使用提示。在使用提示之前，首先尝试找出优化器为何没有选择高效的执行计划。研究是否可以重写查询，或者添加索引或生成新的统计信息是否有帮助。依赖提示来获得良好的性能既耗时，也可能在升级到较新版本的数据库时导致问题。

## 提示分类

在最新版本的 Oracle 中，有超过 200 种不同的提示。描述每一个提示超出了本教程的范围。**表 19-6** 描述了提示的主要类别。

**提示** 从 Oracle Database 11*g* 开始，你可以查询 `V$SQL_HINT` 来获取有关提示的信息。

*表 19-6. SQL 提示分类*

| 提示类别 | 描述 | 提示示例 |
| :--- | :--- | :--- |
| **优化器目标** | 优化吞吐量或响应速度。 | `ALL_ROWS`, `FIRST_ROWS` |
| **访问路径** | 检索数据的路径。 | `INDEX`, `NO_INDEX`, `FULL` |
| **查询转换** | 以不同方式执行查询。 | `STAR_TRANSFORMATION`, `MERGE`, `FACT`, `REWRITE` |
| **连接顺序** | 更改要连接的表的顺序。 | `ORDERED`, `LEADING` |
| **并行操作** | 并行执行。 | `PARALLEL`, `PARALLEL_INDEX`, `NOPARALLEL` |
| **连接操作** | 如何连接表。 | `USE_NL`, `USE_MERGE`, `USE_HASH` |
| **杂项** | 不属于特定类别但有时有用的提示。 | `APPEND`, `CACHE`, `DRIVING_SITE` |

##### 19-13. 查看优化器统计信息

## 问题

你遇到了性能问题。你想确定被查询的表是否存在统计信息。

## 解决方案

检查统计信息是否准确的一个快速简便的方法是比较 `USER_TABLES` 中的 `NUM_ROWS` 列与表的实际行数。以下脚本将提示你输入表名：
```sql
select a.num_rows/b.actual_rows
from user_tables a
,(select count(*) actual_rows from &&table_name) b
where a.table_name=upper('&&table_name');
```

下一个脚本使用 SQL 为某个用户的所有表生成前面的 SQL。该脚本使用 SQL*Plus 格式化命令来关闭标题和分页：
```sql
set head off pages 0 lines 132 trimspool on
spo show_stale.sql
select
'select ' || '''' || table_name || ': ' || '''' || '||' || chr(10) ||
' round(decode(b.actual_rows,0,0,a.num_rows/b.actual_rows),2) ' || chr(10) ||
'from user_tables a ' ||
',(select count(*) actual_rows from ' || table_name || ') b' || chr(10) ||
'where a.table_name=(' || '''' || table_name || '''' || ');'
from user_tables;
spo off;
```

此脚本会生成一个名为 `show_stale.sql` 的文件，你可以执行该文件以显示表统计信息的陈旧程度。

## 工作原理

解决方案部分中的脚本使用了收集统计信息时填充的 `NUM_ROWS` 列。如果 `NUM_ROWS` 与实际行数的百分比差异超过 10%，则统计信息可能已经陈旧。


# SQL 监控与调优：统计信息

## 查看统计信息生成时间

判断统计信息是否为近期生成，有几种其他技巧。例如，你可以检查 `USER_TABLES` 和 `USER_INDEXES` 视图的 `LAST_ANALYZED` 列，以确定你的表和索引最后一次被分析的时间。以下查询显示了当前模式下各表最后一次被分析的时间：

```sql
select table_name, last_analyzed, monitoring
from user_tables
order by last_analyzed;
```

以下查询则报告了当前模式下各索引最后一次被分析的时间：

```sql
select index_name, last_analyzed
from user_indexes
order by last_analyzed;
```

孤立地查看表和索引最后一次生成统计信息的时间可能有些误导。`LAST_ANALYZED` 列确实能告诉你统计信息是否曾经生成过，但也就仅此而已。仅凭 `LAST_ANALYZED` 时间不足以确定统计信息是否需要重新生成。

要获取关于表统计信息的更详细信息，请使用 `USER_TAB_STATISTICS` 视图。此视图包含为表生成的众多优化器统计信息。例如，该视图有一个 `STALE_STATS` 列，用于指示 Oracle 是否认为统计信息已过时。

以下查询显示了诸如 `SAMPLE_SIZE` 和 `STALE_STATS` 等统计信息：

```sql
select table_name, partition_name, last_analyzed, num_rows, sample_size, stale_stats
from user_tab_statistics
order by last_analyzed;
```

与表统计信息类似，你也可以通过查询 `USER_IND_STATISTICS` 视图来查看索引统计信息：

```sql
select index_name, partition_name, last_analyzed, num_rows, sample_size, stale_stats
from user_ind_statistics
order by last_analyzed;
```

如果你想确定自上次生成统计信息以来，发生了多少 `INSERT`、`UPDATE`、`DELETE` 和 `TRUNCATE` 活动，请查询 `USER_TAB_MODIFICATIONS` 视图。例如，你可能会遇到这样一种情况：一个表在插入大量新记录之前被截断。你可以运行如下查询来验证此类活动是否自上次生成统计信息以来发生过：

```sql
select table_name, partition_name, inserts, updates, deletes, truncated, timestamp
from user_tab_modifications
order by table_name;
```

如果你检测到表中的数据发生了显著变化，你应该考虑手动生成统计信息（详情请参阅配方 19-14）。

> **注意**：`USER_TAB_MODIFICATIONS` 视图并非实时更新。Oracle 会定期将内存中的结果刷新到此表。要强制 Oracle 将这些统计信息从内存中刷新，请运行 `DBMS_STATS.FLUSH_DATABASE_MONITORING_INFO` 过程。

## 生成统计信息

**问题**
你刚刚向一个表中加载了大量新数据。现在遇到了性能问题，你想为刚刚加载的表生成统计信息。

**解决方案**
使用 `DBMS_STATS` 包来生成新的统计信息。在下面这段代码中，为 `STAR1` 模式生成了统计信息。

```sql
SQL> exec dbms_stats.gather_schema_stats(ownname => 'STAR1', -
estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE, -
degree => DBMS_STATS.AUTO_DEGREE, -
cascade => true);
```

`ESTIMATE_PERCENT` 参数设置为 `AUTO_SAMPLE_SIZE`。这指示 Oracle 确定最佳的样本大小。

`DEGREE` 参数设置为 `AUTO_DEGREE`。这允许 Oracle 根据硬件和初始化参数选择最佳的并行度。

`CASCADE` 值为 `TRUE` 指示 Oracle 同时收集索引统计信息（针对任何被分析的表）。

> **注意**：你也可以使用 `ANALYZE` 语句配合 `COMPUTE` 或 `ESTIMATE` 子句来生成统计信息。
> 然而，Oracle 强烈建议**不要**使用 `ANALYZE` 语句来生成对象统计信息。相反，你应该使用 `DBMS_STATS` 包。

**工作原理**



# 第 20 章 数据库故障排除

数据库故障排除是一个泛称，适用于广泛的主题范围，例如解决存储问题、调查性能迟缓或监控数据库对象。SQL 通常是你用来初步了解数据库中可能出现问题的第一个工具。因此，你必须熟悉用于提取和分析数据库信息的 SQL 技术。

本章中的方法主要面向数据库管理员（DBA）。然而，如果你是一名开发人员，其中许多主题也会间接影响你与数据库高效协作的能力。了解本章所述问题的解决方案将使你对 Oracle 架构有更深入的理解，并为你提供信息，以便与那位脾气不好、守着自己一亩三分地的老旧数据库管理员一起排查应用程序问题。

当然，DBA 需要成为识别其数据库内问题领域的专家。对于任何连接性、性能或可用性问题，DBA 都是团队中寻求帮助的关键成员。一名优秀的 DBA 必须精通 SQL，尤其是在故障排除方面。

要涵盖所有可用于排查数据库问题的 SQL 语句类型是不可能的。相反，我们提供一些处理常见数据库痛点的 SQL 见解。我们将首先展示如何识别潜在问题，然后探讨如何处理存储问题和数据库审计。

调用 `DBMS_STATS` 有多种方法。例如，如果你只想为一个表生成统计信息，请使用 `GATHER_TABLE_STATS` 过程。此示例为 `D_DOMAINS` 表收集统计信息：

```sql
SQL> exec dbms_stats.gather_table_stats(ownname=>'STAR2',tabname=>'D_DOMAINS',-
> cascade=>true, estimate_percent=>20, degree=4);
```

从 Oracle Database 10g 开始，有一个默认的 `DBMS_SCHEDULER` 作业每天运行一次，自动分析你的数据库对象。要查看此作业上次运行的时间，请执行以下查询：

```sql
select
owner
,job_name
,last_start_date
from dba_scheduler_jobs
where job_name IN
('GATHER_STATS_JOB', -- 10g
'BSLN_MAINTAIN_STATS_JOB' -- 11g
);
```

对于许多数据库，这种自动统计信息收集作业是足够的。然而，在 SQL 查询所基于的表中，如果有很大比例的数据被修改后，你可能会遇到性能不佳的情况。在表数据发生显著变化的情况下，你可能需要如本方法解决方案部分所示，手动生成统计信息。

当你运行一个查询时，如果某个表没有生成统计信息，Oracle 默认会动态尝试估算优化器统计信息。要使动态采样生效，必须设置 `OPTIMIZER_DYNAMIC_SAMPLING` 初始化参数。在 Oracle Database 10g 及更高版本中，此参数的默认值为 2。如果你启用了 `AUTOTRACE`，Oracle 会如下标示 SQL 语句使用了动态采样：

注意
- 此语句使用了动态采样

##### 20-1. 确定数据库问题的原因

### 问题

用户报告称，使用数据库的应用程序行为异常。你的经理要求你检查数据库是否出了什么问题。

### 解决方案

在诊断数据库问题时，你必须有一个策略来确定从何处查找问题。例如，如果用户报告了特定的数据库错误，或者警报日志中有详细的消息，那么显然应该从调查这些报告的问题开始。

如果没有明确的起点，你可以运行自动工作量仓库（AWR）报告（需要许可）或 Statspack 报告；有关如何运行这些报告的更多详细信息，请参见第 19 章。如果你没有这些工具可用，可以直接查询数据字典以确定数据库是否承受压力以及问题的根本原因是什么。

下一个查询使用 `V$SYSSTAT` 视图来确定数据库处理用户请求所花费的时间和 CPU 时间，从而推断出数据库等待时间（通过减法）。这与可用时间有关，可用时间是挂钟时间乘以 CPU 数量。此查询的目的是确定你的数据库是否花费了过多时间等待资源（这是系统过载的迹象）：

```sql
select
round(100*CPU_sec/available_time,2) "ORACLE CPU TIME AS % AVAIL."
,round(100*(DB_sec - CPU_sec)/available_time,2) "NON-IDLE WAITS AS % AVAIL."
,case
sign(available_time - DB_sec)
when 1 then round(100*(available_time - DB_sec) / available_time, 2)
else 0
end "ORACLE IDLE AS % AVAIL."
from
(
select
(sysdate - i.startup_time) * 86400 * c.cpus available_time
,t.DB_sec
,t.CPU_sec
from v$instance i
,(select value cpus
from v$parameter
where name = 'cpu_count') c
,(select
sum(case name
when 'DB time' then round(value/100)
else 0
end) DB_sec
,sum(case name
when 'DB time' then 0
else round(value/100)
end) CPU_sec
from v$sysstat
where name in ('DB time', 'CPU used by this session')) t
where i.instance_number = userenv('INSTANCE')
);
```

以下是一些输出：

```
ORACLE CPU TIME AS % AVAIL. NON-IDLE WAITS AS % AVAIL. ORACLE IDLE AS % AVAIL.
--------------------------- -------------------------- -----------------------
31.97                       129.59                     0
```

如非空闲等待所示，该数据库显然处于高负载状态。既然我们知道了存在与资源等待相关的问题，下一个问题就是具体确定等待发生在哪里。以下查询详细说明了 Oracle 的等待时间分布。该查询使用了一些标准的 SQL*Plus 格式化命令，以使输出更易读：

```sql
col wait_class format A15
col event format A35 trunc
col "CLASS AS % OF WHOLE" format 990.00 HEAD "CLASS AS|% OF WHOLE"
col "EVENT AS % OF CLASS" like "CLASS AS % OF WHOLE" HEAD "EVENT AS|% OF CLASS"
col "EVENT AS % OF WHOLE" like "CLASS AS % OF WHOLE" HEAD "EVENT AS|% OF WHOLE"
set pagesize 30
break on wait_class on "CLASS AS % OF WHOLE"
--

select
wait_class,
round(100 * time_class / total_waits, 2) "CLASS AS % OF WHOLE", event,
round(100 * time_waited / time_class, 2) "EVENT AS % OF CLASS", round(100 * time_waited / total_waits, 2) "EVENT AS % OF WHOLE"
from
(select
wait_class
,event
,time_waited
,sum(time_waited) over (partition by wait_class) time_class
,rank() over (partition by wait_class order by time_waited desc) rank_within_class
,sum(time_waited) over () total_waits
from v$system_event
where wait_class <> 'Idle')
where rank_within_class <= 3
order by time_class desc, rank_within_class;
```

以下是输出片段：

```
                   CLASS AS     EVENT AS     EVENT AS
WAIT_CLASS         % OF WHOLE   % OF CLASS   % OF WHOLE
------------ --------------- ------------ ------------
System I/O           15.50        60.14         9.32
                                log file parallel write
                                db file parallel write    18.59         2.88
                                control file parallel write 17.80      2.76

Commit               14.34       100.00        14.34
                                log file sync

User I/O             12.40        59.47         7.37
                                db file scattered read
                                db file sequential read   26.11         3.24
                                direct path read           5.99         0.74
```

从报告的 System I/O 部分来看，`log file parallel write` 事件可能表明在线重做日志文件中存在争用。同样，从 User I/O 部分来看，`db file scattered read` 等待事件可能表明 SQL 查询正在执行过多的全表扫描。如果你认为问题是 SQL 性能问题，请参见第 19 章了解如何隔离性能不佳的 SQL。



# 第 20 章 数据库故障排除

## 常见等待事件

表 20-1 列出了一些影响性能的常见等待事件。这绝非详尽无遗的列表，它只是众多等待事件类型中的一部分，包含描述和可能的应对措施。目前存在超过 900 种不同类型的等待事件（具体数量取决于您使用的 Oracle 版本）。有关所有等待事件的完整列表和描述，请参阅 Oracle 的《数据库参考指南》，网址为`otn.oracle.com`。

[www.it-ebooks.info](http://www.it-ebooks.info/)

表 20-1. 常见数据库等待事件
| 等待事件 | 描述 | 进一步检查 |
| :--- | :--- | :--- |
| `db file scattered read` | 为多个块执行 I/O 的实际时间（通常由全表扫描引起）。 | I/O 过多，SQL 执行不佳；参见第 19 章进行 SQL 调优。 |
| `db file sequential read` | 为单个块执行 I/O 的实际时间（通常与索引扫描相关）。 | I/O 过多，SQL 执行不佳；参见第 19 章进行 SQL 调优。 |
| `free buffer waits` | 在检查缓存寻找空闲缓冲区后未找到空闲缓冲区，或等待脏缓冲区被写入磁盘时的等待时间。 | 表示缓存中空闲缓冲区不足；检查缓冲区缓存统计信息，看缓存是否过小或过大。 |
| `buffer busy waits` | 等待缓冲区变为可用的等待时间。 | 检查`V$SESSION`以确定块争用的类型。 |
| `db file parallel write` | 数据库写入器等待 I/O 完成的时间。 | 磁盘 I/O 争用。 |
| `log file parallel write` | 将重做信息从日志缓冲区写入磁盘的 I/O 完成时间。 | 减少在线重做日志的 I/O 争用；将日志移到专用磁盘或更快的磁盘。 |
| `log file sync` | 用户提交时刷新重做信息的时间。 | 减少在线重做日志的 I/O 争用；将日志移到专用磁盘或更快的磁盘。 |
| `log file switch (archiving needed)` | 因为日志写入器将切换到未归档的重做日志而等待日志切换。 | 归档目标可能空间不足或归档无法足够快地读取重做日志。 |
| `log file switch (checkpoint incomplete)` | 因为日志的检查点未完成而等待日志切换。 | 可能重做日志过少或过小（参见配方 20-3）。 |
| `control file parallel write` | 完成写入所有控制文件的时间。 | 控制文件中的 I/O 争用。 |
| `SQL*Net more data to client` | 服务器向客户端发送数据以完成传输的时间。 | 可能存在网络或中间层瓶颈。 |
| `SQL*Net more data from client` | 服务器等待客户端发送更多数据。 | 可能存在网络或中间层瓶颈。 |
| `SQL*Net message to/from client` | 用户客户端和服务器之间的通信时间。 | 可能存在网络或中间层瓶颈。 |
| `enq: TX - row lock contention` | 内部事务锁。 | 潜在的锁问题，查询`V$LOCK`。 |
| `enq: TM - contention` | 内部 DML 锁。 | 潜在的锁问题，查询`V$LOCK`。 |

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 工作原理

有时开发人员和数据库管理员（DBA）采用老式的“战舰”方法进行数据库故障排除。他们以一种碰运气的方式，随机地向数据库发送可能有助于揭示问题的查询。然而，这是一种低效的排错方式。相反，我们建议您采用更有针对性的故障排除方法。首先运行一个工具（如 AWR）或本配方解决方案部分提供的查询，以指向真正的问题所在。

解决方案部分的查询使用了 Oracle 的动态性能视图`V$SYSSTAT`和`V$SYSTEM_EVENT`来帮助定位数据库问题的可能来源。表 20-2 列出了常用于排除数据库问题的视图描述。

我们意识到，从`V$SYSSTAT`和`V$SYSTEM_EVENT`等视图中选择数据有其固有的局限性，因为这些视图报告的是自实例启动以来收集的平均值。如果可能，查询 AWR 或 Statspack 历史视图会更好。然而，尽管存在这些已知的局限性，解决方案部分的查询仍将为您提供一个开始数据库故障排除的起点。关键是，您应该从一套能够引导您努力方向的工具开始分析。

表 20-2. 常用的 Oracle 动态性能视图
| 视图 | 描述 |
| :--- | :--- |
| `V$SYSSTAT` | 系统级别的累积工作负载统计信息。 |
| `V$SESSTAT` | 会话级别的累积工作负载统计信息。 |
| `V$SEGSTAT` | 段级别的累积工作负载统计信息。 |
| `V$EVENT_NAME` | 所有等待事件及其关联的类。 |
| `V$SYSTEM_EVENT` | 所有会话因某个事件而等待的总次数。 |
| `V$SESSION_EVENT` | 单个会话因某个事件而等待的总次数。 |
| `V$SESSION_WAIT` | 当前正在等待或最近完成等待的会话的事件。 |

##### 20-2. 显示打开的游标

### 问题

一个访问数据库的应用程序抛出以下错误：`ORA-01000: maximum open cursors exceeded`

您希望查询数据字典以确定每个会话打开的游标数量。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 解决方案

以下查询将显示每个会话当前打开的游标：
```sql
select
    a.value
    ,c.username
    ,c.machine
    ,c.sid
    ,c.serial#
from v$sesstat a
    ,v$statname b
    ,v$session c
where a.statistic# = b.statistic#
and c.sid = a.sid
and b.name = 'opened cursors current'
and a.value != 0
and c.username IS NOT NULL
order by 1,2;
```

我们建议您查询`V$SESSION`而不是`V$OPEN_CURSOR`来确定打开的游标数量。`V$SESSION`提供了当前打开游标数量的更准确数字。

### 工作原理

`OPEN_CURSORS`初始化参数决定了一个会话可以打开的游标数量上限。这个设置是针对每个会话的。默认值`50`对于任何应用程序来说通常都太低了。通常将`OPEN_CURSORS`设置为`1000`这样的值。

但是，如果单个会话有超过`1000`个打开的游标，那么代码中很可能存在未关闭游标的情况。当达到此限制时，应该有人检查应用程序代码，以确定是否有游标未被关闭。

如果您工作的环境有数千个到数据库的连接，您可能只想查看消耗游标最多的前几个会话。以下查询使用了一个内联视图和伪列`ROWNUM`来显示前二十个值：
```sql
select * from (
    select
        a.value
        ,c.username
        ,c.machine
        ,c.sid
        ,c.serial#
    from v$sesstat a
        ,v$statname b
        ,v$session c
    where a.statistic# = b.statistic#
    and c.sid = a.sid
    and b.name = 'opened cursors current'
    and a.value != 0
    and c.username IS NOT NULL
    order by 1 desc,2
)
where rownum < 21;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 20-3. 判断在线重做日志大小是否合适

### 问题

您注意到存在异常多的`log file switch (checkpoint incomplete)`等待事件（描述参见表 20-1）。您希望验证生产数据库中重做日志切换的频率。

### 解决方案

`V$LOG_HISTORY`包含有关在线重做日志的历史信息。执行此查询以查看每小时的日志切换次数：
```sql
select count(*)
    ,to_char(first_time,'YYYY:MM:DD:HH24')
from v$log_history
group by to_char(first_time,'YYYY:MM:DD:HH24')
order by 2;
```

以下是输出的一个片段：
```
COUNT(*) TO_CHAR(FIRST
---------- -------------
        15 2009:07:20:00
        31 2009:07:20:01
        47 2009:07:20:02
        14 2009:07:20:03
         3 2009:07:20:04
         1 2009:07:20:05
```

从前面的输出中，您可以看到在 7 月 20 日，大约从午夜到凌晨 3:00，产生了大量的重做数据。这可能是由于夜间批处理作业，或者是由于位于世界另一端的用户在运行数据库应用程序（您的夜晚是他们的白天）所致。

### 工作原理



# `V$LOG_HISTORY` 视图的数据源自控制文件。每当发生日志切换时，此视图中就会记录一条详细记录，包含诸如切换时间和系统变更号（`SCN`）等信息。

一条通用的经验法则是，你应该将在线重做日志文件的大小设置为大约每小时切换两到三次。你不希望它们切换得过于频繁，因为日志切换会产生开销。Oracle 在日志切换过程中会启动一个检查点。在检查点期间，数据库写入器后台进程会将所有已修改的（脏）块写入磁盘，这是资源密集型操作。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 20 章 ■ 数据库故障排除

另一方面，你也不希望在线重做日志文件从不切换，因为当前在线重做日志中可能包含你在恢复事件中需要用到的事务。如果你遭遇灾难，导致当前在线重做日志发生介质故障，你可能会丢失那些尚未归档的事务。

**提示** 使用 `ARCHIVE_LAG_TARGET` 初始化参数来设置日志切换之间的最长时间（以秒为单位）。该参数的典型设置是 1800 秒（30 分钟）。值为 0（默认值）则禁用此功能。此参数常用于 Oracle Data Guard 环境，以在指定时间过后强制进行日志切换。

你还可以查询 `V$INSTANCE_RECOVERY` 视图中的 `OPTIMAL_LOGFILE_SIZE` 列，以确定你的在线重做日志文件大小是否设置正确：

```sql
select optimal_logfile_size from v$instance_recovery;
```

以下是一些示例输出：

```
OPTIMAL_LOGFILE_SIZE
--------------------
                 512
```

此查询报告被认为是最优的重做日志文件大小（以兆字节为单位），其基于 `FAST_START_MTTR_TARGET` 初始化参数设置。Oracle 建议你将所有在线重做日志配置为至少达到 `OPTIMAL_LOGFILE_SIZE` 的值。然而，在确定在线重做日志大小时，你必须考虑你的环境信息（例如切换频率）。

你可以通过执行以下查询来查看当前在线重做日志文件的大小：

```sql
select a.group#,
       a.member,
       b.status,
       b.bytes/1024/1024 meg_bytes
from v$logfile a,
     v$log b
where a.group# = b.group#
order by a.group#;
```

以下是一些示例输出：

```
GROUP# MEMBER                          STATUS              MEG_BYTES
--------- ------------------------------ -------------------- ----------
1 /oralogs01/DWREP/redo01a.log       INACTIVE                   256
1 /oralogs02/DWREP/redo01b.log       INACTIVE                   256
2 /oralogs01/DWREP/redo02a.log       CURRENT                    256
2 /oralogs02/DWREP/redo02b.log       CURRENT                    256
3 /oralogs01/DWREP/redo03a.log       INACTIVE                   256
3 /oralogs02/DWREP/redo03b.log       INACTIVE                   256
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 20 章 ■ 数据库故障排除

##### 20-4. 确定撤销表空间大小是否合适

**问题**

你有一个运行时间很长的 SQL 语句，正在抛出 `ORA-01555` “快照过旧” 错误。你想要确定你的撤销表空间大小设置是否正确。

**解决方案**

运行以下查询，以识别撤销表空间在过去一天内发生的潜在问题：

```sql
select
  to_char(begin_time,'MM-DD-YYYY HH24:MI') begin_time,
  ssolderrcnt ORA_01555_cnt,
  nospaceerrcnt no_space_cnt,
  txncount max_num_txns,
  maxquerylen max_query_len,
  expiredblks blck_in_expired
from v$undostat
where begin_time > sysdate - 1
order by begin_time;
```

以下是一些示例输出。部分输出已省略以适应页面：

```
BEGIN_TIME         ORA_01555_CNT  NO_SPACE_CNT MAX_NUM_TXNS MAX_QUERY_LEN
------------------ -------------  ------------ ------------ -------------
07-20-2009 18:10              0              0          249             0
07-20-2009 18:20              0              0          290             0
07-20-2009 18:30              0              0          244             0
07-20-2009 18:40              0              0          179             0
```

`ORA_01555_CNT` 列指示你的数据库遇到 `ORA-01555` “快照过旧” 错误的次数。如果此列报告非零值，你需要执行以下一项或多项操作：

-   确保代码中没有在游标循环内包含 `COMMIT` 语句。
-   优化抛出错误的 SQL 语句，使其运行得更快。
-   确保你拥有良好的统计信息（以便你的 SQL 高效运行）。
-   增加 `UNDO_RETENTION` 初始化参数的值。

`NO_SPACE_CNT` 列显示了在撤销表空间中请求空间但未找到可用空间的次数。如果 `NO_SPACE_CNT` 报告非零值，你可能需要为你的撤销表空间添加更多空间。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 20 章 ■ 数据库故障排除

**工作原理**

`V$UNDOSTAT` 视图中最多存储四天的信息。统计信息每十分钟收集一次，表中最多有 576 行。如果你在最近四天内停止并启动过数据库，此视图将仅包含自上次启动数据库以来的信息。

另一种获取撤销表空间大小调整建议的方法是使用 Oracle Undo Advisor，你可以通过 `SELECT` 语句查询 PL/SQL `DBMS_UNDO_ADV` 包来调用它。以下查询显示了在撤销保留设置为 900 秒的情况下，当前的撤销大小和建议大小：

```sql
select
  sum(bytes)/1024/1024 cur_mb_size,
  dbms_undo_adv.required_undo_size(900) req_mb_size
from dba_data_files
where tablespace_name LIKE 'UNDO%';
```

以下是一些示例输出：

```
CUR_MB_SIZE REQ_MB_SIZE
----------- -----------
      36864       20897
```

输出显示，撤销表空间当前分配了 36.8 GB。在之前的查询中，我们使用 900 秒作为在撤销表空间中保留信息的时间。Oracle Undo Advisor 估计，为了保留撤销信息 900 秒，撤销表空间应为 20.8 GB。在此示例中，撤销表空间的大小是足够的。如果大小不足，你将需要向现有数据文件添加空间，或者向撤销表空间添加一个数据文件。

下面是一个稍微复杂一点的使用 Oracle Undo Advisor 来查找撤销表空间所需大小的示例。此示例使用 PL/SQL 来显示有关潜在问题的信息和修复问题的建议。

```sql
SET SERVEROUT ON SIZE 1000000
DECLARE
  pro VARCHAR2(200);
  rec VARCHAR2(200);
  rtn VARCHAR2(200);
  ret NUMBER;
  utb NUMBER;
  retval NUMBER;
BEGIN
  DBMS_OUTPUT.PUT_LINE(DBMS_UNDO_ADV.UNDO_ADVISOR(1));
  DBMS_OUTPUT.PUT_LINE('Required Undo Size (megabytes): ' ||
    DBMS_UNDO_ADV.REQUIRED_UNDO_SIZE (900));
  retval := DBMS_UNDO_ADV.UNDO_HEALTH(pro, rec, rtn, ret, utb);
  DBMS_OUTPUT.PUT_LINE('Problem: ' || pro);
  DBMS_OUTPUT.PUT_LINE('Advice: ' || rec);
  DBMS_OUTPUT.PUT_LINE('Rational: ' || rtn);
  DBMS_OUTPUT.PUT_LINE('Retention: ' || TO_CHAR(ret));
  DBMS_OUTPUT.PUT_LINE('UTBSize: ' || TO_CHAR(utb));
END;
/
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 20 章 ■ 数据库故障排除

如果没有发现问题，则保留大小将返回 0。以下是一些示例输出：

```
Finding 1: 撤销表空间正常。

Required Undo Size (megabytes): 20897

Problem: 未发现问题

Advice:
Rational:
Retention: 0
UTBSize: 0
```

##### 20-5. 确定临时表空间大小是否正确

**问题**

你尝试创建一个索引，Oracle 抛出此错误：

```
ORA-01652: 无法通过 128 (在表空间 TEMP 中) 扩展 temp 段
```

你想要确定你的临时表空间大小是否设置正确。

**解决方案**

如果你使用的是 Oracle Database 11g 或更高版本，请运行以下查询以显示临时表空间中已分配和空闲的空间：

```sql
select
  tablespace_name,
  tablespace_size/1024/1024 mb_size,
  allocated_space/1024/1024 mb_alloc,
  free_space/1024/1024 mb_free
from dba_temp_free_space;
```

以下是一些示例输出：

```
TABLESPACE_NAME                MB_SIZE    MB_ALLOC     MB_FREE
--------------- ---------- ---------- ----------
TEMP                              200        200        170
```



如果 `FREE_SPACE` (`MB_FREE`) 值下降到接近零，说明你的数据库中有某些 SQL 操作正在消耗大部分可用空间。`FREE_SPACE` (`MB_FREE`) 列显示的是总可用空闲空间，包括当前已分配且可重复使用的空间。

## 数据库故障排除

如果你使用的是 Oracle Database 10g 数据库，可以运行此查询来查看临时表空间中已使用的空间：

```sql
select tablespace_name, sum(bytes_used)/1024/1024 mb_used
from v$temp_extent_pool
group by tablespace_name;
```

以下是一些示例输出：

```
TABLESPACE_NAME   MB_USED
--------------- ----------
TEMP                      120
```

如果已使用量接近当前分配的总量，你可能需要为临时表空间数据文件分配更多空间。运行以下查询以查看临时数据文件名称和分配大小：

```sql
select name, bytes/1024/1024 mb_alloc from v$tempfile;
```

以下是一些典型输出：

```
NAME                          MB_ALLOC
------------------------------ ----------
/ora02/DWREP/temp01.dbf            12000
/ora03/DWREP/temp03.dbf            10240
/ora01/DWREP/temp02.dbf             2048
```

### 工作原理

当进程已消耗完可用内存并需要更多空间时，临时表空间会被用作磁盘上的排序区域。需要排序区域的操作包括：

- 索引创建
- SQL 排序操作
- 临时表和临时索引
- 临时 LOB
- 临时 B 树

没有确切的公式来确定你的临时表空间大小是否合适。这取决于查询的数量和类型、索引构建操作、并行操作以及你的内存排序空间（程序全局区）的大小。你必须在数据库有负载时，使用解决方案部分中的某个查询来监控你的临时表空间，以确定其使用模式。

如果 Oracle 抛出 `ORA-01652` 错误，这表明你的临时表空间太小。然而，如果因为一次性事件（如大型索引构建）导致空间耗尽，Oracle 也可能抛出此错误。你必须判断是一次性的索引构建还是消耗大量临时表空间排序空间的查询，值得为此增加空间。

如果你确定需要增加空间，可以调整现有数据文件的大小，也可以添加一个新的数据文件。要调整临时表空间数据文件的大小，请使用 `ALTER DATABASE TEMPFILE...RESIZE` 语句。以下语句将一个临时数据文件的大小调整为 12 GB：

```sql
alter database tempfile '/ora03/DWREP/temp03.dbf' resize 12g;
```

你可以按如下方式向临时表空间添加一个数据文件：

```sql
alter tablespace temp add tempfile '/ora04/DWREP/temp04.dbf' size 2g;
```

## 显示表空间使用率

### 问题

你遇到了 `ORA-01653` “无法扩展表” 错误。你想查明每个表空间还剩多少空闲空间。

### 解决方案

运行以下脚本以查看数据库中的空闲字节数和已分配字节数：

```sql
select
    a.tablespace_name
    ,(f.bytes/a.bytes)*100 pct_free
    ,f.bytes/1024/1024 mb_free
    ,a.bytes/1024/1024 mb_allocated
from
    (select nvl(sum(bytes),0) bytes, x.tablespace_name
    from dba_free_space y, dba_tablespaces x
    where x.tablespace_name = y.tablespace_name(+)
    and x.contents != 'TEMPORARY' and x.status != 'READ ONLY'
    and x.tablespace_name not like 'UNDO%'
    group by x.tablespace_name) f,
    (select sum(bytes) bytes, tablespace_name
    from dba_data_files
    group by tablespace_name) a
where a.tablespace_name = f.tablespace_name
order by 1;
```

以下是一些示例输出：

```
TABLESPACE_NAME     PCT_FREE    MB_FREE MB_ALLOCATED
--------------- ------------- ---------- ------------
DOWN_TBSP_9        79.1992188       8110        10240
DTS_STAGE_DATA           84.3        843         1000
DTS_STAGE_INDEX          96.7        967         1000
INST_TBSP_1              96.8        484          500
```

### 工作原理



# 监控表空间的剩余空间

监控表空间的剩余空间是数据库管理员的一项基本活动。您可以使用表空间数据文件的 AUTOEXTEND 功能，该功能允许数据文件根据需要增长空间。一些 DBA 不愿意使用 AUTOEXTEND 功能，因为担心某个包含大量事务的失控进程会导致数据文件填满现有的磁盘空间。如果您没有为数据文件启用 AUTOEXTEND，则需要使用类似本解决方案中提供的脚本来监控表空间。

有时在对数据库执行物理文件维护时，列出与数据库关联的所有文件很有用。在这些情况下，您需要一个能显示数据文件、控制文件和在线重做日志文件的脚本。这在生产支持场景中尤其重要，例如当某个表空间即将写满，而您希望向与该表空间关联的众多数据文件之一添加空间时。

下一个脚本允许您快速识别即将写满的表空间以及哪些数据文件是扩展空间的候选对象。这里我们使用了许多 SQL*Plus 格式化命令以使输出易于阅读：

```sql
REM *****************************************************
REM * File : freesp.sql
REM *****************************************************

SET PAGESIZE 100 LINES 132 ECHO OFF VERIFY OFF FEEDB OFF SPACE 1 TRIMSP ON

COMPUTE SUM OF a_byt t_byt f_byt ON REPORT

BREAK ON REPORT ON tablespace_name ON pf

COL tablespace_name FOR A17 TRU HEAD 'Tablespace|Name'
COL file_name FOR A40 TRU HEAD 'Filename'
COL a_byt FOR 9,990.999 HEAD 'Allocated|GB'
COL t_byt FOR 9,990.999 HEAD 'Current|Used GB'
COL f_byt FOR 9,990.999 HEAD 'Current|Free GB'
COL pct_free FOR 990.0 HEAD 'File %|Free'
COL pf FOR 990.0 HEAD 'Tbsp %|Free'
COL seq NOPRINT

DEFINE b_div=1073741824
--

COL db_name NEW_VALUE h_db_name NOPRINT
COL db_date NEW_VALUE h_db_date NOPRINT

SELECT name db_name, TO_CHAR(sysdate,'YYYY_MM_DD') db_date FROM v$database;
--

SPO &&h_db_name._&&h_db_date..lis

PROMPT Database: &&h_db_name, Date: &&h_db_date
--

SELECT 1 seq, b.tablespace_name, nvl(x.fs,0)/y.ap*100 pf, b.file_name file_name, b.bytes/&&b_div a_byt, NVL((b.bytes-SUM(f.bytes))/&&b_div,b.bytes/&&b_div) t_byt, NVL(SUM(f.bytes)/&&b_div,0) f_byt, NVL(SUM(f.bytes)/b.bytes*100,0) pct_free FROM dba_free_space f, dba_data_files b
,(SELECT y.tablespace_name, SUM(y.bytes) fs
  FROM dba_free_space y GROUP BY y.tablespace_name) x
,(SELECT x.tablespace_name, SUM(x.bytes) ap
  FROM dba_data_files x GROUP BY x.tablespace_name) y
WHERE f.file_id(+) = b.file_id
AND x.tablespace_name(+) = y.tablespace_name
and y.tablespace_name = b.tablespace_name
AND f.tablespace_name(+) = b.tablespace_name
GROUP BY b.tablespace_name, nvl(x.fs,0)/y.ap*100, b.file_name, b.bytes
UNION
SELECT 2 seq, tablespace_name, j.bf/k.bb*100 pf, b.name file_name, b.bytes/&&b_div a_byt, a.bytes_used/&&b_div t_byt, a.bytes_free/&&b_div f_byt, a.bytes_free/b.bytes*100 pct_free
FROM v$temp_space_header a, v$tempfile b
,(SELECT SUM(bytes_free) bf FROM v$temp_space_header) j
,(SELECT SUM(bytes) bb FROM v$tempfile) k
WHERE a.file_id = b.file#
ORDER BY 1,2,4,3;
--

COLUMN name FORMAT A60 HEAD 'Control Files'

SELECT name FROM v$controlfile;
--

COL member FORMAT A50 HEAD 'Redo log files'
COL status FORMAT A9 HEAD 'Status'
COL archived FORMAT A10 HEAD 'Archived'
--

SELECT
a.group#, a.member, b.status,
b.archived, SUM(b.bytes)/1024/1024 mb
FROM v$logfile a, v$log b
WHERE a.group# = b.group#
GROUP BY a.group#, a.member, b.status, b.archived
ORDER BY 1, 2;
--

SPO OFF;
```

该脚本将输出假脱机到一个文件中，文件名嵌入了数据库名称和日期。

例如，如果您在 2009 年 7 月 22 日为名为 `DW11` 的数据库运行该脚本，输出文件将被命名为 `DW11_2009_07_22.lis`。该脚本还使用了一个 `B_DIV` 参数，将所有字节列除以值 `1073741824`，这使得输出以千兆字节为单位显示。



##### 20-7. 显示对象大小

## 问题

你希望主动运行一个报告，显示数据库中哪些表和索引占用了最多的空间。

## 解决方案

以下是一个典型的脚本，用于显示数据库中占用空间最大的对象（表和索引）：

```sql
select * from
(
  select
    segment_name
    ,partition_name
    ,segment_type
    ,owner
    ,bytes/1024/1024 mb
  from dba_segments
  order by bytes desc)
where rownum <=20;
```

以下是一些示例输出：

```
SEGMENT_NAME        PARTITION_NAME  SEGMENT_TYPE       OWNER           MB
------------------- --------------- ------------------ --------------- ----------
_SYSSMU1$           TYPE2           UNDO               SYS             18481.4375
CIA_DOWNLOADS                       TABLE              CIA             7868
REG_QUEUE_REP                        TABLE              REP_MV          6874
F_DOWNLOADS        DOWN_P_7        TABLE PARTITION     STAR2           3180
F_DOWNLOADS_IDX1                    INDEX              CIA_STAR        2927
```

## 工作原理

有时在排查磁盘空间问题时，查看哪些对象（表和索引）消耗了最多的存储空间是很有用的。解决方案部分中的脚本使用一个内联视图，先选择数据库中的所有段，并按 `BYTES` 列降序排序。外部查询然后将输出限制为数据库中占用空间最大的前二十个对象。

如果你有兴趣查看哪些对象在哪些数据文件中占用空间，你需要查询 `DBA_DATA_FILES` 视图。下面列出了一个脚本，它会提示你输入所有者，然后显示表以及相关数据文件中消耗的空间：

```sql
select
  a.table_name
  ,b.tablespace_name
  ,b.partition_name
  ,c.file_name
  ,sum(b.bytes)/1024/1024 mb
from dba_tables a
  ,dba_extents b
  ,dba_data_files c
where a.owner = upper('&owner')
  and a.owner = b.owner
  and a.table_name = b.segment_name
  and b.file_id = c.file_id
group by
  a.table_name
  ,b.tablespace_name
  ,b.partition_name
  ,c.file_name
order by a.table_name, b.tablespace_name;
```

##### 20-8. 监控索引使用情况

## 问题

你希望移除数据库中未被使用的任何索引。你需要一个关于索引是否正在被使用的简单“是”或“否”的答案。

## 解决方案

使用 `ALTER INDEX...MONITORING USAGE` 语句来启用基本的索引监控。以下示例在名为 `F_DOWN_DOM_FK9` 的索引上启用索引监控：

```sql
alter index F_DOWN_DOM_FK9 monitoring usage;
```

从此时起，第一次访问该索引时，Oracle 会记录这一点，并可以通过 `V$OBJECT_USAGE` 视图查看。要报告哪些被监控的索引曾被使用过，运行此查询：

```sql
select * from v$object_usage;
```

很可能你不会只监控一个索引。相反，你可能希望监控某个用户的所有索引。在这种情况下，使用 SQL 生成 SQL 来创建一个你可以运行的脚本，以打开对所有索引的监控。下面就是一个这样的脚本：

```sql
select 'alter index ' || index_name || ' monitoring usage;'
from user_indexes;
```

## 工作原理

`V$OBJECT_USAGE` 视图只显示当前连接用户的信息。如果你检查 `DBA_VIEWS` 的 `TEXT` 列，会注意到以下一行：

`where io.owner# = userenv('SCHEMAID')`

如果你以 DBA 账户登录，并想查看所有已启用监控的索引的状态（无论用户是谁），请执行此查询：

```sql
select io.name, t.name,
       decode(bitand(i.flags, 65536), 0, 'NO', 'YES'),
       decode(bitand(ou.flags, 1), 0, 'NO', 'YES'),
       ou.start_monitoring,
       ou.end_monitoring
from sys.obj$ io, sys.obj$ t, sys.ind$ i, sys.object_usage ou
where i.obj# = ou.obj#
  and io.obj# = ou.obj#
  and t.obj# = i.bo#;
```

##### 20-9. 审计对象使用情况

## 问题

你想知道数据库中哪些表实际正在被使用。

## 解决方案

使用数据库审计来确定对象是否正被 `SELECT`、`INSERT`、`UPDATE` 和 `DELETE` 语句访问。按照以下步骤启用和使用数据库审计：

1.



## 启用审计

通过将 `AUDIT_TRAIL` 初始化参数设置为有效值（例如 `DB`）来在数据库上启用审计。如果您使用的是 SPFILE，则使用 `ALTER SYSTEM` 语句设置 `AUDIT_TRAIL` 参数：
```sql
SQL> alter system set audit_trail=db scope=spfile;
```
如果您使用的是 `init.ora` 文件，请用文本编辑器打开它并将其值设置为 `DB`（有关 `AUDIT_TRAIL` 参数有效值的描述，请参见表 20-3）：
```
AUDIT_TRAIL=DB
```

**表 20-3. `AUDIT_TRAIL` 的有效设置**

| 设置 | 含义 |
| --- | --- |
| `DB` | 启用审计并将 `SYS.AUD$` 表设置为审计存储库。 |
| `DB, EXTENDED` | 启用审计并将 `SYS.AUD$` 表设置为审计存储库，同时包含 `SQLTEXT` 和 `SQLBIND` 列。 |
| `OS` | 启用审计并指定操作系统文件将存储审计信息。 |
| `XML` | 启用审计并将审计记录以 XML 格式写入操作系统文件。 |
| `XML, EXTENDED` | 启用审计并将审计记录以 XML 格式写入操作系统文件，包括 `SqlText` 和 `SqlBind` 的值。 |
| `NONE` | 禁用数据库审计。 |

2.  接下来，停止并启动您的数据库。
```sql
SQL> shutdown immediate;

SQL> startup;

SQL> show parameter audit_trail;
```
```
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
audit_trail                          string      DB
```
3.  现在，您可以按对象、权限或语句启用审计。例如，以下语句对 `INV_MGMT` 用户拥有的 `EMP` 表上的所有 DML 访问启用审计：
```sql
SQL> audit select, insert, update, delete on inv_mgmt.emp;
```
4.  从此时起，对 `INV_MGMT.EMP` 表的任何 DML 访问都将被记录在 `SYS.AUD$` 表中。您可以使用类似以下的查询来报告对表的 DML 访问：
```sql
select
  username
  ,obj_name
  ,timestamp
  ,substr(ses_actions,4,1) del
  ,substr(ses_actions,7,1) ins
  ,substr(ses_actions,10,1) sel
  ,substr(ses_actions,11,1) upd
from dba_audit_object;
```
要关闭对某个对象的审计，请使用 `NOAUDIT` 语句：
```sql
SQL> noaudit select, insert, update, delete on inv_mgmt.inv;
```
您应定期清理 `AUD$` 表，以免其在 `SYSTEM` 表空间中占用过多空间。如果您需要保存 `AUD$` 数据，可以先导出该表，然后使用 `TRUNCATE` 命令删除记录。

> **提示：** 如果您想将 `AUD$` 表移动到非 `SYSTEM` 表空间，请参阅 My Oracle Support (MetaLink) 笔记 72460.1 以获取说明。

## 工作原理

Oracle 审计是一项强大的功能，允许您确定针对表正在执行何种类型的 SQL 活动。例如，您可能希望确定哪些表正在被使用或未被使用。如果一个表未被使用，您可能希望重命名然后删除该表。审计允许您捕获用于访问表的 SQL 语句类型。

在解决方案的第 4 步中，我们使用 `SUBSTR` 函数来引用 `DBA_AUDIT_OBJECT` 视图的 `SES_ACTIONS` 列。该列包含一个 16 个字符的字符串，其中每个字符表示某个特定操作已发生。这 16 个字符按以下顺序代表以下操作：`ALTER`、`AUDIT`、`COMMENT`、`DELETE`、`GRANT`、`INDEX`、`INSERT`、`LOCK`、`RENAME`、`SELECT`、`UPDATE`、`REFERENCES` 和 `EXECUTE`。

位置 14、15 和 16 由 Oracle 保留以供将来使用。字符 `S` 表示成功，`F` 表示失败，`B` 表示成功和失败。

一旦您确定了未使用的表，您可以简单地重命名这些表，看看这是否会破坏应用程序或是否有任何用户投诉。如果没有投诉，那么一段时间后您可以考虑删除这些表。在删除任何您以后可能需要恢复的表之前，请确保使用 `RMAN` 和 Data Pump 对数据库进行了良好的备份。



##### 20-10. 细粒度审计

## 问题

你拥有一些关于员工薪水的敏感数据。你希望审计对表中特定列的任何类型访问。你需要比标准审计更细粒度的审计。

## 解决方案

使用 Oracle 细粒度审计（FGA）来监控任何用户从特定列的选择。按照以下步骤实现 FGA：

**注意** 细粒度审计功能需要 Oracle 企业版。

1.  使用 `DBMS_FGA` 包创建一个策略。此示例为 `INV` 方案中的 `EMP` 表创建一个策略，指定任何针对 `SALARY` 列的 `INSERT`、`UPDATE`、`DELETE` 或 `SELECT` 语句都将被记录到审计跟踪中：

```
begin
    dbms_fga.add_policy (
        object_schema   => 'INV',
        object_name     => 'EMP',
        audit_column    => 'SALARY',
        policy_name     => 'S1_AUDIT',
        statement_types => 'INSERT, UPDATE, DELETE, SELECT',
        audit_trail     => DBMS_FGA.DB_EXTENDED
    );
end;
/
```

2.  通过查询 `DBA_AUDIT_POLICIES` 视图来验证策略是否存在：

```
select object_schema
      ,object_name
      ,policy_name
      ,sel, ins, upd, del, policy_column
from dba_audit_policies;
```

此示例的输出如下：

```
OBJECT_SCHEMA OBJECT_NAME POLICY_NAME SEL INS UPD DEL POLICY_COL
-------------- -------------- -------------- ----- ----- ----- ----- ----------
INV            EMP           S1_AUDIT       YES  YES  YES  YES  SALARY
```

3.  要查看 FGA 审计跟踪中记录的 SQL 语句，请从 `DBA_FGA_AUDIT_TRAIL` 视图中选择：

```
select
    db_user
   ,to_char(timestamp,'dd-mon-yy hh24:mi:ss') ts
   ,sql_text
from dba_fga_audit_trail
order by timestamp;
```

**注意** `DBA_FGA_AUDIT_TRAIL` 视图基于 `FGA_LOG$` 表。

以下是一些示例输出：

```
DB_USER    TS                   SQL_TEXT
---------- -------------------- --------------------------
INV        21-jul-09 21:47:07   select * from emp
SYSTEM     21-jul-09 21:58:36   select salary from inv.emp
```

如果需要禁用策略，请使用 `DISABLE_POLICY` 过程：

```
SQL> exec dbms_fga.disable_policy('INV','EMP','S1_AUDIT');
```

要删除策略，请使用 `DROP_POLICY` 过程：

```
SQL> exec dbms_fga.drop_policy('INV','EMP','S1_AUDIT');
```

作为 `SYS` 方案，你可以按如下方式清除细粒度审计审计表中的记录：

```
truncate table fga_log$;
```

## 工作原理

细粒度审计允许你在比简单的 `INSERT`、`UPDATE`、`DELETE` 和 `SELECT` 操作更细的粒度级别上审计 SQL。FGA 审计允许你审计在列级别发生的 SQL 活动。FGA 审计还允许你对操作执行布尔检查，例如“如果选定的值在某个范围内，则审计该活动”。

你通过使用 FGA 策略来管理细粒度审计。`DBMS_FGA` 包允许你添加、禁用、启用和删除 FGA 策略。你需要 `DBMS_FGA` 包上的执行权限来管理审计策略。

**提示** 有关细粒度审计的更多详细信息，请参阅 Oracle 的《安全指南》，可从 otn.oracle.com 网站获取。

## 提示

如果你只需要知道是否正在对表进行插入、更新或删除，可以使用 `USER_TAB_MODIFICATIONS` 视图来报告此类活动。此视图包含诸如 `INSERTS`、`UPDATES`、`DELETES` 和 `TRUNCATED` 等列，这些列将提供有关如何修改表中数据的信息。

****索引**

■**符号和数字**

在多个级别聚合数据，30-32

*（星号）占位符，5

按时间段，159-160

%（百分号）字符，在字符串中搜索，

在其它查询中使用结果，32-33

字符串，169

聚合，对移动窗口执行

_（下划线）字符，在字符串中搜索



# A

`windows`，49—51

`strings`，169

`aliases`

`|| (竖线) operator`，97

`hints and`，491

`11g Release 2 (Oracle)`，275

`for objects, creating`，443—445

`ALL_DB_LINKS view`，453

`ALL operator`，122—124

`ALL static views`，230

`ALL_SYNONYMS view`，250

`ABS function`，126

`ALL_TAB_COLS system view`，130

`accessing`

`ALL_TAB_COLUMNS system view`，129—130

`LOBs with HTTP`，391—396

`ALL_TABLES view`，236

`values from subsequent or preceding`

`ALL_USERS view`，234

`rows`，42—45

`ALL_VIEWS view`，252

`alphabetical sorting`，18—20

`access path, and interpreting execution`

`ALTER DATABASE DATAFILE command`，

`plans`，485

`RESIZE option`，412—413

`access to tables, determining`，235—237

`ALTER DATABASE TEMPFILE ... RESIZE`

`Active Session History (ASH), using`，466—467

`statement`，509

`ADDM (Automatic Database Diagnostic`

`ALTER INDEX ... REBUILD command`，429

`Monitor), using`，466

`ALTER INDEX ... MONITORING USAGE`

`ADD PARTITION clause (ALTER TABLE`

`statement`，513

`statement)`，368

`ALTER PROFILE statement`，414

`addresses of rows`，429—430

`ALTER SESSION statement`

`aggregate data sets, ignoring groups in`，28—

`enabling case-insensitive queries with`，

30

293

`aggregate functions, building custom`，39—42

`turning tracing on and off with`，480—481

`aggregates, joining on`，70—71

`ALTER SYSTEM statement`，481

519

`www.it-ebooks.info`

## 索引

`ALTER TABLE ... RENAME statement`，430

`creating tablespaces and`，410

`ALTER TABLE MOVE statement`，429

`enabling`，413

`ALTER TABLE ... SHRINK SPACE statement`，

`overview of`，510

307

`auto-incrementing values, creating`，453—455

`ALTER TABLE statement`

`Automatic Database Diagnostic Monitor`

`ADD PARTITION clause`，368

`(ADDM), using`，466

`to disable constraints on table`，300, 304

`Automatic Workload Repository (AWR),`

`DISABLE ROW MOVEMENT clause`，366

`using`，465—466

`DROP PARTITION clause`，375—376

`Automatic Workload Repository reports,`

`ENABLE ROW MOVEMENT clause`，366

`uses of`，497

`MERGE PARTITIONS clause`，373—374

`AUTOTRACE`

`MOVE PARTITION clause`，365

`generating execution plans using`，471—

`SPLIT PARTITION clause`，372—373

474

`TRUNCATE PARTITION clause`，376



# B

## AND 逻辑运算符

`AND` 逻辑运算符，4

## ANSI SQL 99 连接语法

`ANSI SQL 99` 连接语法

## 平衡树 (B-树) 索引

`B-树` 索引，435 — 436

与 `ON` 子句，61

与 `NATURAL JOIN` 子句，62

与 `USING` 子句，60

## 十二进制计数

十二进制计数，143

## 不同进制间数字转换

不同进制间数字转换，124 — 127

## 基于查询构建 WHERE 条件

基于查询构建 `WHERE` 条件，15 — 16

## BFILE 对象

`BFILE` 对象

- 描述，383
- 使其可用于数据库，396 — 398

## ARCHIVE_LAG_TARGET 初始化参数

`ARCHIVE_LAG_TARGET` 初始化参数，504

## BINARY_DOUBLE 数据类型

`BINARY_DOUBLE` 数据类型，192 — 193

## BINARY_FLOAT 数据类型

`BINARY_FLOAT` 数据类型，192 — 193

## 在报告中包含 ASCII 风格字符画

在报告中包含 `ASCII` 风格字符画，279 — 280

## 将图像数据加载到 BLOB 列

将图像数据加载到 `BLOB` 列，387 — 389

## ORDER BY 子句的 ASC 选项

`ORDER BY` 子句的 `ASC` 选项，7

## 二进制大对象 (BLOBs)

二进制大对象 (`BLOBs`)，383

## 使用 ASH (活动会话历史)

使用 `ASH`（活动会话历史），466 — 467

## 二进制精度

二进制精度，189 — 190，193

## 在查询结果中为行分配排名值

在查询结果中为行分配排名值，45 — 47

## 二进制排序

二进制排序，294

## 二进制排序顺序

二进制排序顺序，19

## 关联权限组

关联权限组，416 — 418

## 二进制 XML 存储

二进制 `XML` 存储，338

## 星号 (*) 占位符

星号 (`*`) 占位符，5

## 绑定变量

绑定变量，268

## 审计

审计

- 在精细级别，516 — 518
- 对象使用情况，514 — 516

## AUDIT_TRAIL 设置

`AUDIT_TRAIL` 设置，514

## AUTOEXTEND 特性

`AUTOEXTEND` 特性，520

## 创建位图索引

创建位图索引，439

## 将图像数据加载到 BLOB 列

将图像数据加载到 `BLOB` 列，387 — 389

## BLOBs

`BLOBs`，383

## 识别阻塞的事务

识别阻塞的事务，213 — 214

## www.it-ebooks.info

`www.it-ebooks.info`

## 索引

## 逻辑运算符

逻辑运算符

- `AND`，4
- `NOT`，5
- `OR`，5

## 将大型文档加载到字符大对象 (CLOB) 列

将大型文档加载到字符大对象 (`CLOB`) 列，384 — 386

## 字符大对象 (CLOBs)

字符大对象 (`CLOBs`)，383

## BREAK 特性 (SQL*Plus)

`BREAK` 特性 (`SQL*Plus`)，263 — 264

## 使用字符大对象存储 XML 数据

字符大对象存储 `XML` 数据，338



-   **桶**，将结果排序至等大小的桶中，用于处理 `string`内的字符替换, 172—

`repo`索引, 271—273 174

构建聚合函数，39—42

`CHAR_STRING` 函数, 126—127

批量加载 LOB 数据, 389—391

检查对象是否为表，241

`检查完整性约束` 的创建, 450

■ **C**

用途, 451

数据清洗

`凯撒密码, 296

执行不区分大小写的查询，计算 293—294

禁用约束以处理非数字和无限数字，299—304

使用, 192—193

数据，从表中移除, 305—307

处理数值计算中的空值，确定数据是否可作为 200—201 日期加载, 291—293

总计和小计，37—38

确定数据是否可作为数值加载, 290—291

日历的历史, 143

笛卡尔积，78

展示模式之间的差异，307— 311

`CASCADE CONSTRAINTS` 子句（`DROP TABLESPACE` 语句），412

删除所有索引, 297—299

`CASCADE` 选项（`DISABLE` 子句), 301

检测重复行，287—289

基于 `CASE` 的测试, 111

移除重复行，289—290

`CASE` 表达式与 `ROW_NUMBER` 函数, 263—266

概述 ，287

禁用触发器，304—305

`CASE` 函数, 100—102

混淆值，294—297

执行不区分大小写的查询，293— 294

将大型文档加载到 `CLOB` 列中, 384—386

CLOB, 383

XML 数据的 `CLOB` 存储，338

`catalog.sql` 脚本

`COALESCE` 函数, 110—111

动态性能视图与其, 232

显示与 PL/SQL 对象或触发器关联的代码, 253—254

静态视图与其关系, 231

跟踪日期和时间数据的变化，148—149

列

*另请参见* 虚拟列

`BLOB`，将图像数据加载至其中，387—389

行中的值变更, 10—11

转换为行，95—96

XML 就地, 349—350

将行转换为列, 89—91

替换单个字符，296

将大型文档加载到 `CLOB` 中, 384—386

派生新列, 83—87 521

基于多个列进行透视，92—95

禁用约束，299—304

移除基于值的重复行

[www.it-ebooks.info](http://www.it-ebooks.info/)

■ 索引



显示, 244—246

子集, 51—55

启用, 303

从表中选择所有字段, 5—6

外键, 448—449

字符串中的单字符替换

主键, 445—447

列，执行操作, 172—174

唯一键, 445—446

值汇总, 在中, 23—25

约束验证，延迟执行, 218—225

逗号分隔值文件,

两张表的内容，同步操作, 138—

创建操作, 105—107

140

`COMMIT`语句，以及`DML`语句，

转换

209

字符串与数值数据类型之间，

比较

187—188

数据集上的假设, 277—279

时区间的日期和时间，

连接条件中的`NULL`值, 80—81

152—153

比较选项, 20—21

日期时间值为可读字符串，

复合分区, 357—359

143—145

数据拼接以提高可读性, 97—99

非分区表转换为分区

`CONCAT`函数, 97—98

表, 366—368

条件分支处理, 119—120

不同进制间的数字转换, 124—

基于存在性检查的条件插入或

127

更新, 21—22

数值数据类型间, 188—189

条件排序, 120—122

SQL 转 XML, 335—339

条件，检查数据以满足, 450—452

字符串转换为日期时间值, 145—146

使用表空间配置分区，

复制

364—365

批量将数据从一张表复制到另一张表，

`CONNECT BY`子句

10

描述, 204—205[](#index_split_003.html#p233)

从一张表复制行到另一张表, 9—10

生成固定数量的序列化

相关子查询, 118

主键 以及, 331—333

相关更新, 79

层次数据与, 314

`COUNT(DISTINCT ... )`聚合函数，

使用指定层次查询, 315—

164—165

317

`COUNT`函数, 24, 34—35

`CONNECT_BY_ISCYCLE`伪列，

`COUNT(*)`函数, 25

314, 329—330

计数

`CONNECT_BY_ISLEAF`伪列，

十二进制, 143

315, 324—328

组和集合中的成员, 33—35

`CONNECT BY LEVEL`功能, 150—152

模式, 179—181



## CONNECT_BY_ROOT 伪列

`CONNECT_BY_ROOT`伪列，327

## CREATE 命令

`CREATE DATABASE`命令，230, 404—

创建数据库之间的连接，406，452—453

验证连接信息，407—409

`CONNECT TO`子句，452

`CREATE INDEX`语句，378, 435

生成连续数字，196—197

`CREATE OR REPLACE`方法，442

`CREATE PROFILE`命令，413

`CREATE ROLE`命令，416，522

`CREATE SYNONYM`命令，443

调整数据文件大小，413

`CREATE TABLE`命令，425—426

数据类型精度，190—192

`CREATE TABLESPACE`语句，409—411

`CREATE USER`语句，419—420

`CREATE VIEW`命令，441

`CREATE VIEW`语句，85

## 约束

检查完整性约束，450—451，522

## CUBE 函数

`CUBE`函数，31—32

## CURRVAL 伪列

`CURRVAL`伪列，196—197

## 游标

游标驱动的分页，117

显示打开的游标，501—503

## 日期

确定日期数据类型是否可加载为，291—293

查找重叠的日期范围，146—148

确定一周中的哪一天，157—159

确定复活节的日期，162—164

找出两个日期之间的差异，160—162

确定本月第一天，156—157

确定本月最后一天，155—156

跟踪日期变化，148—149

### 日期时间值

将日期时间值转换为可读字符串，143—145

将字符串转换为日期时间值，145—146

确定两个日期时间值之间的 elapsed time，160—161

`TO_CHAR`函数中的`DAY`格式代码，158

## DBA/ALL/USER 视图

`DBA/ALL/USER_CONSTRAINTS`视图，245—247

在层次数据中识别循环，329—330

`DBA/ALL/USER_SOURCE`视图，253

`DBA/ALL/USER_SYNONYMS`视图，251

## 日期数据类型

确定日期数据类型是否可加载为，291—293

## 日期范围

查找重叠的日期范围，146—148

## CROSS JOIN 结构

`CROSS JOIN`结构，76—78

## CSV 文件

创建 CSV（逗号分隔值）文件，105—107

## 层次数据

在层次数据中识别循环，329—330

## 区间子句

`CREATE TABLE`语句中的`INTERVAL`子句，359—360

## 索引

■ INDEX

## 区间分区

`CREATE TABLE`语句中的`PARTITION BY RANGE`子句，354—355

## 列表分区

`CREATE TABLE`语句中的`PARTITION BY LIST`子句，356

## 哈希分区

`CREATE TABLE`语句中的`PARTITION BY HASH`子句，356—357

## 引用分区

`CREATE TABLE`语句中的`PARTITION BY REFERENCE`子句，361—362

## 系统分区

`CREATE TABLE`语句中的`PARTITION BY SYSTEM`子句，363—364

## 表空间子句

`CREATE TABLE`语句中的`TABLESPACE`子句，364

## 追踪变化

追踪日期变化，148—149

## 唯一访问者

追踪每天唯一网站访问者数量，164—165

## 时区

转换时区，152—153

## 两周差异

找出两个日期之间的差异，160—162

## www.it-ebooks.info

[www.it-ebooks.info](http://www.it-ebooks.info/)



### DBA/ALL/USER_TRIGGERS 视图
页码 304。

### ■D
#### DBA_AUDIT_OBJECT 视图
`SES_ACTIONS` 列，页码 515。

#### 数据库管理
页码 403。

#### DBA_AUDIT_POLICIES 视图
页码 517。

#### 数据库代码，显示
页码 253–254。

#### DBA_CONSTRAINTS 视图
页码 244–246。

#### 数据库链接
`DBA_DATA_FILES` 视图，页码 512。

##### 创建
页码 452。

#### DBA_DB_LINKS 视图
页码 453。

##### 公共链接，创建
页码 453。

#### DBA_DEPENDENCIES 视图
页码 248。

##### 引用
页码 453。

#### DBA_FGA_AUDIT_TRAIL 视图
页码 517。

#### 数据库参考指南 (Oracle)
页码 499。

#### DBA_HIST_SNAPSHOT 视图
页码 469。

#### 数据库资源管理器
页码 414。

#### DBA_HIST_SQLSTAT AWR 视图
页码 469。

#### 数据库资源配置文件设置
页码 414–415。

#### DBA_OBJECTS 表
页码 213–214。

#### 数据库
`DBA_ROLE_PRIVS` 视图，页码 255。

##### 创建
页码 404–406。

#### DBA 静态视图
页码 230。

##### 删除
页码 406–407。

#### DBA_SYNONYMS 视图
页码 250。

#### 数据库故障排除
页码 497。

#### DBA_SYS_PRIVS 视图
页码 257–258。

#### 数据定义语言语句
`DBA_TABLES` 视图，页码 236。
页码 425。

#### DBMS_CRYPTO 包
页码 296。

#### 数据字典，实例化
页码 405。

#### DBMS_FGA 包
页码 518。

#### 数据字典视图
页码 213, 230–232, 259。
页码 523。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### ■ 索引
#### DBMS_METADATA 包
页码 298。

#### 默认排序
页码 19。

#### DBMS_METADATA 过程
页码 224。

#### DEFERRABLE 关键字
页码 218–221。

#### DBMS_MONITOR，为会话跟踪所有 SQL 语句
页码 479。

#### DELETE 语句
##### 用于从表中删除所有行
页码 13。

#### DBMS_RANDOM 包
页码 102–105。

##### 用于从表中删除行
页码 12。

#### DBMS_RANDOM.SEED 过程
页码 104。

##### 与 TRUNCATE 语句比较
页码 307。

#### DBMS_RANDOM.VALUE 过程
页码 103–104。

##### 在表中删除 LOB
页码 398–400。

#### DENSE_RANK 函数
页码 46–47。

#### DBMS_SCHEDULER 作业
页码 495。

##### 密集排名
页码 46。

#### DBMS_SESSION 包
##### 用于启用 SQL 跟踪
页码 476。

##### 用于派生新列
页码 83–87。

##### 用于为会话跟踪所有 SQL 语句
页码 479。

#### ORDER BY 子句的 DESC 选项
页码 7。

#### 检测。 *见* 查找

#### DBMS_SPACE 包
页码 306。

#### 确定
*另见* 查找

#### DBMS_SQLTUNE 包，
##### REPORT_SQL_MONITOR 函数
页码 460–462。

##### 任何年份的复活节日期
页码 162–164。

##### 星期几
页码 157–159。

#### DBMS_STATS.FLUSH_DATABASE_MONITOR



首次日期位于 `month, 156—157`

`ORING_INFO` 过`ocedure, 494`

最后日期位`于 month, 155—156`

`DBMS_STATS` 包`package, 377, 494—495`

查询执行进度`s, 462—463`

`DBMS_SUPPORT`，用于跟踪所有 SQL

虚拟列`s, 86`

语句以供会话使用`g, 480`

网站的“X 天活跃”用户`site, 164—`

`DBMS_SYSTEM`，用于跟踪所有 SQL 语句

`165`

以供会话使用`g, 480`

`DETERMINISTIC`，用于声明函数为，

`DBMS_TRACE` 包`package, 482—483`

`186`

`DBMS_UNDO_ADV` 包 (PL/SQL)`, 506`

`D` 格式代码 (`TO_CHAR` 函`数), 158`

`DBMS_XMLGEN` 函`数), 337`

诊断数据库问`题, 497—501`

`DBMS_XMLSCHEMA` 包，

目录命名 `方法, 453`

`REGISTERSCHEMA` 函`数, 346—`

直`接读取, 226`

`348`

`DISABLE` 子句，`CASCADE` 选项`, 301`

`DBMS_XPLAN, 474—476`

`DISABLE ROW MOVEMENT` 子句 (`ALTER`

`DDL` (数据定义语言) 语句，

`TABLE 语句), 366`

`425`

禁用

约束`条件, 299—304`

`DDL_LOCK_TIMEOUT` 参数，设置，

主键约束`条件, 446`

`216`

死锁

`回收站功能, 435`

避`免, 216—218`

触发`器, 304—305`

描述`信息, 209`

表的磁盘空间使用情况，显示，`237—`

死锁解析错误 `消息, 217`

`240`

`DECIMAL` 数据类`型, 191`

显示

十进制精度`, 189—190`

约束`条件, 244—246`

`DECODE` 函`数, 101—102`

`数据库代码, 253—254`

`DEFAULT` 配置文件，

模式之间的差`异, 307—311`

`PASSWORD_VERIFY_FUNCTION`，

外键列未索引`的情况, 242—`

`422—423`

`244`

`524`

[www.it-ebooks.info](http://www.it-ebooks.info/)

■ 索引

已授予的角色，`254—255`

`DROP TABLESPACE 语句, 411—412`

表的索引`, 241—242`

`DROP TABLE 语句, 433`

对象依赖关系，`247—250`

`DROP USER 语句, 420—421`

对象级权`限, 256—257`

`DUAL` 表，`77`

对象大`小, 511—513`

重复行

打开的游`标, 501—503`

检`测, 287—289`

主键与外键

移`除, 289—290`

关系`, 246—247`

重复项，查找表中的`重复项, 35—37`

查询执行进度`, 459—462`



## 索引

## 动态生成表同义词

*   在网页上查询结果，113–117
*   在视图上，444
*   同义词元数据，250–251

## 动态性能数据字典

*   系统级特权，257–259
*   视图，231–232
*   表磁盘空间使用，237–240
*   动态性能视图，213, 501
*   表行数，240–241
*   `tablespace fullness`，509–511
*   用户信息，233–235

## ■ E

*   `view text`，251–252
*   `DISTINCT` 关键字，95, 164–165
*   `easy connect` 命名方法，452
*   DML 语句
*   日期或日期时间之间的时长
    *   描述，209
    *   确定值，160–161
    *   事务的隔离级别，226
*   11g 第 2 版 (Oracle)，275
*   `NOWAIT` 子句，214–216
*   在查询中消除 `NULL`，16–18
*   文档，将大文档加载到 `CLOB`
*   空字符串的处理，18
*   `columns`，384–386
*   `ENABLE NOVALIDATE` 子句，300
*   文档，`XML`
    *   从中提取关键 `XML` 元素，343–344
    *   生成复杂 `XML`，344–346
*   `ENABLE ROW MOVEMENT` 子句 (`ALTER TABLE` 语句)，366
*   启用约束，303
*   加密，296
*   `&` 变量后的点号，445
*   强制密码复杂度，422–423
*   双 `&` 标签技术 (SQL*Plus)，267–269
*   确保存在查找值，448–449
*   `DROP DATABASE` 命令，406–407
*   `DROP DATABASE LINK` 命令，452
*   `DROP PARTITION` 子句 (`ALTER TABLE` 语句)，375–376
*   企业管理器数据库控制 (Oracle)，222
*   `ESCAPE` 关键字，169
*   ETL (提取、转换、加载) 工具，107, 287
*   欧元数字，194–195
*   `EXAMPLE` 表空间，59
*   删除
    *   所有索引，297–299
    *   数据库链接，452
    *   数据库，406–407
    *   分区，375–376
    *   主键约束，446
    *   表，433–434
    *   表空间，411–412
    *   用户，420–421
*   `EXECUTE` 语句，`DBMS_STATS` 包，377
*   执行计划
    *   描述，483
    *   在查询上强制生成，490–492
    *   生成，525

[www.it-ebooks.info](http://www.it-ebooks.info/)

## ■ 索引



使用`AUTOTR[ACE`，471—474]

重叠日期`ranges`，146—148]

使用`DBM[S_XPLAN`，474—476]

`表中不存在的行`，73—74]

`解释`，483—488]

76]

`统计信息`，487—488]

`表中的序列间隙`，55—57]

`数据的存在性，测试`，117—118]

`细粒度审计（FGA）`，516—518]

`EXISTS 子句（SELECT 语句）`，67—68]

`FIRST 函数`，48]

`EXISTS 谓词`，117—118]

`五命令规则`，403—404]

执行计划。*参见* 执行计划

`FLASHBACK TABLE 语句`，435]

`EXPLAIN PLAN 语句`，473]

`FLASHBACK TABLE ... TO BEFORE DROP`
`语句`，434]

`显式锁定方法`，133—137]

`外部命名方法`，453]

`浮点计算`，193—194]

`提取、转换和加载（ETL）工具`，107, 287]

`在查询上强制执行计划`，490—492]

未建立索引的外键列，显示，242—244]

`EXTRACT 函数`，162, 343—344]

提取
`从 XML 文档中提取关键 XML 元素`，343—344]
`模式`，178—179]
`子字符串`，170—171]

`日期时间值`，144]

`EXTRACTVALUE 函数`，349—350]

`公式，生成数字到`，198—199]

`空闲空间，监控表空间的`，510]

`FROM_TZ 函数`，152—153]

■`F`

`FULL OUTER JOIN`，65—67]

`全表扫描`，306]

`FBIs（基于函数的索引）`
`基于函数的索引（FBIs）`
`创建`，438—439]
`描述`，81]
`创建和使用的限制`，186]
`加速字符串搜索和`，184—186]
`在其中使用函数`，186]

`FEEDBACK 参数`，298]

`函数`
`FGA（细粒度审计）`，516—518]
`[ABS`，126]
`AVG`，23]

`字段，按字段分组数据`，27—28]

`按相对排名筛选结果`，275—277]



## 索引

## A
`building aggregate, 39—42` 查找  
`built-in, 24`  
*另见* 确定

## C
`CASE, 100—102` 两个日期或日期之间的差异  
`CHAR_STRING, 126—127` parts, 160—162  
`COALESCE, 110—111` 表中的重复值和唯一值，  
`CONCAT, 97—98` 35—37  
`COUNT, 24,` 34—35 组内的第一个和最后一个值, 47—48  
`COUNT(*), 25` leap years, 154—155  
`COUNT(DISTINCT ... ), 164—165` 跨表匹配数据, 68—70  
`CUBE, 31—32` 缺失的行, 71—73

## D
`DBMS_XMLGEN, 337` 查询中的 NULLS , 16—18 526  
`[www.it-ebooks.info](http://www.it-ebooks.info/)` ■ INDEX in DBMS_XPLAN, 475 REGISTERSCHEMA  
`DECODE, 101—102` (DBMS_XMLSCHEMA 包),  
`DENSE_RANK, 46—47` 346—348  
`DETERMINISTIC`, 声明为, 186 REGR_R2, 133  
`EXTRACT, 162, 343—344` REGR_SLOPE, 131  
`EXTRACTVALUE, 349—350` REPLACE, 167, 297

## F
`FIRST, 48` ROLLUP, 31—32  
`FROM_TZ, 152—153` ROUND, 202

## G
`GROUPING, 25` ROW_NUMBER  
假设检验统计, 277 CASE 表达式与之关系, 263—266

## I
`INITCAP, 99` 隐式页面计算与之关系, 113—  
`INSTR, 167—170` 117  
`ISDATE`, 创建, 291—292 第 n 个结果，确定, 275—277  
`ISNUM`, 创建, 290—291 概述, 51—53  
`ISSCHEMAVALID, 348` RPAD, 279—280

## L
`LAG`, 42—45, 55—56 SCN_TO_TIMESTAMP, 148—149  
`LAST, 48` STATS_T_TEST_INDEPU, 279  
`LAST_DAY, 155—156` 字符串连接, 97—99  
`LEAD, 42—45, 55—56` SUBSTR, 126, 170—171  
`LPAD, 317` SYS_CONNECT_BY_PATH, 315, 322—324

## M
`MAX, 24, 271` SYS_CONTEXT, 234—236  
`MIN, 24, 271` SYSDATE, 45  
`MOD, 155` SYS_XMLGEN, 335—338  
在 SELECT 语句中嵌套, 85 TO_BINARY_DOUBLE, 189  
`NTILE, 272` TO_CHAR

## N
`NVL, 80, 109—111, 200`


## 索引

## 字符限制于
`145`

## NVL2
`99, 110—111`

## 将十进制数转换为十六进制数
`124`

## OBF
`296`

## RANDOM
`102—105`

## DAY 和 D 格式代码
`158`

## RANK
`45—47`` `

## 格式化日期时间值
`144`

## REBASE_NUMBER
- `创建`: `124—126`` `
- `GROUP BY 子句及`: `159—160`
- `在 SELECT 语句中引用`: `85`
- 概述: `38`

## REGEXP_COUNT
`180—181`` `

## TO_DATE
`145—146`` `

## REGEXP_INSTR
- 描述: `167`
- `及`: `173`
- 验证字符串中的数字: `194—196`` `
- 搜索模式: `174—178`

## TO_NUMBER
- 在字符串和数值数据类型之间转换: `187—188`` `
- 字符串内的字符替换: `183—184`

## TRANSLATE
`167, 172``, 295``—296`

## 子字符串，搜索
`170`

## TRUNC
`157`

## REGEXP_REPLACE
`167, 182—184`` `

## UNOBF
`296`

## REGEXP_SUBSTR
- 描述: `167`
- 提取模式: `178—179`` `
- 提取子字符串: `171`

## UPDATEXML
`349—350`` `

## WIDTH_BUCKET
`272—275`

## XMLATTRIBUTES
`345—346`

## XMLELEMENT
`345—346`

## XMLROOT
`345—346`

## XMLTABLE
`341—343`

## 分组
- `模糊读取`: `227`
- 计算成员数量: `33—35`
- `查找组内的首个和末个值`: `47—48`` `
- 在聚合数据集中忽略分组: `28—30`` `
- 汇总数据: `26—27`

## G

### 吉尼斯啤酒厂
`278`

### 无间隔时间序列，由带间隔的数据生成
`150—152`

### 生成
- `复杂 XML 文档`: `344—346`` `
- 连续数字: `196—197`` `
- 固定数量的序列主键: `330—333`
- 自动生成数字列表: `204—205`` `

## H

### 执行计划
- 使用 `AUTOTRACE`: `471—474`` `
- 使用 `DBMS_XPLAN`: `474—476`

### 基于哈希的全局索引
`380`

### 哈希，按哈希分区
`356—357`

### HAVING 子句
`28—29, 33, 36`` `

### HAVING 子句 (SELECT 语句)
`288`

### 层级数据
- 由带间隔的数据生成无间隔时间序列: `150—152`
- 生成固定数量的序列主键: `330—333`

### GV$ 视图
`231—232`` `

[www.it-ebooks.info](http://www.it-ebooks.info/)

■ 索引


# 从层级结构生成路径名

## 数字到公式或模式

*   198—226

## 表格

*   321—323，349—352，24
*   199
*   识别其中的循环，329—330

## 从层级表生成路径名

*   在层级表中识别叶子数据，321—324
*   表，324—328，352—356，28

## 随机数据

*   102—105

## 概述

*   313—315

## 统计数据

*   494—496

## 在层级内对节点排序

## 分区统计

*   377
*   318—321

## 测试数据

*   76—78

## 从上至下遍历

*   315—318

# 全局索引

*   创建，380—381

# 表的高水位标记

*   306

# 全局临时表

*   428

# 提示

# 已授予的角色

*   显示，254—255

## 类别

*   491—492

# 粒度级别

*   在此级别进行审核，516—518

## 语法

*   490

# 图形化工具

## 使用

*   491

## 用于数据库管理

*   403

# 直方图

## 用于查看数据库的元数据

*   包含在报告中，279—280
*   229—230

## 用于报告

*   创建，273—275

# 图形化网页报告

*   生成

# HTTP

*   通过其访问大型对象，391—396

## 直接从数据库

*   280—285

# XML 数据的混合存储

*   338

# 图表

*   包含在报告中，279—280

# 假设

*   在数据集上比较，277—

# `GROUP BY` 子句

*   26—27, 159—160
*   279

# 分组结果

*   返回**其中的**详细列，269—271

# 分组

## ■ **I**

*   按多个字段分组，27—28
*   按时间段分组，159—160

# `IDENTIFIED BY` 子句

*   452

# `GROUPING` 函数

*   25

# 识别

*   528

# [www.it-ebooks.info](http://www.it-ebooks.info/)

## ■ 索引

*   阻塞事务，213—214
*   基于存在性插入条件索引
*   层级数据中的循环，329—330
*   21—22

# `INSERT INTO ... SELECT` 方法

*   10, 139

# 层级表中的叶子数据

*   324—328

# `INSERT` 语句

*   使用其进行资源密集型查询
*   用于向表中添加行，8—9

## 操作系统

*   469—471

## `SELECT` 选项

*   9—10

# 资源密集型 SQL 语句

*   安装 `UTLDTREE` 脚本，250
*   附带性能报告，465—469

# 实例化数据字典

*   405

## 使用 `V$SQLSTATS` 视图

*   463—465

# `INSTR` 函数

*   167—170

# IEEE 标准与浮点

*   计算，193—194

# 内部视图定义

*   252



# ■ J

## 连接

描述连接两个或多个表中对应行的方法。有关详细信息，请参阅第 60-62 页（索引分割 001.html#p88）。

### 使连接在两个方向上可选

可以使连接在两个方向上可选。详细信息请参阅第 65-67 页（索引分割 001.html#p93）。

### 连接条件中的操作和比较 NULLS

在连接条件中操作和比较 `NULLS`。详细信息请参阅第 80-81 页（索引分割 001.html#p108, 001.html#p109）。

### 编写可选连接

编写可选连接。详细信息请参阅第 64-65 页（索引分割 001.html#p92, 001.html#p93）。

## 索引

### 添加索引以加速查询

添加索引以加速查询。详细信息请参阅第 485 页（索引分割 006.html#p513）。

### 创建

创建索引。详细信息请参阅第 435-436 页（索引分割 005.html#p463, 005.html#p464）。

### 描述

索引的描述。详细信息请参阅第 436 页（索引分割 005.html#p464）。

### 在聚合上创建位图索引

在聚合上创建位图索引。详细信息请参阅第 70-71 页（索引分割 001.html#p98, 001.html#p99）。

### 删除所有索引

删除所有索引。详细信息请参阅第 297-299 页（索引分割 004.html#p325, 004.html#p327）。

### 函数索引

函数索引。详细信息请参阅第 81, 438-439 页（索引分割 005.html#p466）。

### 本地索引

本地索引。详细信息请参阅第 377-380 页（索引分割 005.html#p405, 005.html#p408）。

### 与分区方案一起创建

与分区方案一起创建索引。详细信息请参阅第 380-381 页（索引分割 005.html#p408, 005.html#p409）。

### 重建索引

重建索引。详细信息请参阅第 429, 436 页（索引分割 005.html#p457）。

### 索引和表

索引和表。详细信息请参阅第 437 页（索引分割 005.html#p465）。

### 监控使用情况

监控索引的使用情况。详细信息请参阅第 513-514 页（索引分割 007.html#p541, 007.html#p542）。

### 为表显示索引

为表显示索引。详细信息请参阅第 241-242 页（索引分割 003.html#p269, 003.html#p270）。

### 索引组织表

创建索引组织表。详细信息请参阅第 440 页（索引分割 005.html#p468）。

# ■ K

## 无限数字

执行无限数字的计算。详细信息请参阅第 192-193 页（索引分割 002.html#p220, 002.html#p221）。

`INF` 代表无限。详细信息请参阅第 192-194 页（索引分割 002.html#p220）。

## 关键字

*另请参阅* 布尔运算符

### ALL

`ALL` 关键字。详细信息请参阅第 122-124 页（索引分割 001.html#p150, 001.html#p152）。

### ANY

`ANY` 关键字。详细信息请参阅第 94, 122-124 页（索引分割 001.html#p150, 001.html#p152）。

### DEFERRABLE

`DEFERRABLE` 关键字。详细信息请参阅第 218-221 页（索引分割 003.html#p246, 003.html#p249）。

### DISTINCT

`DISTINCT` 关键字。详细信息请参阅第 95 页（索引分割 001.html#p123），第 164-165 页（索引分割 002.html#p192）。

### ESCAPE

`ESCAPE` 关键字。详细信息请参阅第 169 页（索引分割 002.html#p197）。

### INNER JOIN

`INNER JOIN` 关键字。详细信息请参阅第 62 页（索引分割 001.html#p90）。

### INTERSECT

`INTERSECT` 运算符。详细信息请参阅第 68-70 页（索引分割 001.html#p96, 001.html#p98）。

### LIKE

`LIKE` 关键字。详细信息请参阅第 127 页（索引分割 002.html#p155）。

### MINUS

`MINUS` 运算符。详细信息请参阅第 71-73 页（索引分割 001.html#p99, 001.html#p101）。

## 其他条目

### INTERVAL 子句（CREATE TABLE 语句）

`INTERVAL` 子句（`CREATE TABLE` 语句）。详细信息请参阅第 359-360 页（索引分割 005.html#p387），第 387-389 页（索引分割 005.html#p415）。

间隔分区。详细信息请参阅第 359-360 页（索引分割 005.html#p387）。

### imp 工具

`imp` 工具，在加载数据前禁用所有外键。详细信息请参阅第 302 页（索引分割 004.html#p330）。

### ISDATE 函数

创建 `ISDATE` 函数。详细信息请参阅第 291-292 页（索引分割 004.html#p319, 004.html#p320）。

### ISNUM 函数

创建 `ISNUM` 函数。详细信息请参阅第 290-291 页（索引分割 004.html#p318, 004.html#p319）。

### IN 子句（SELECT 语句）

`IN` 子句（`SELECT` 语句）。详细信息请参阅第 67-68 页（索引分割 001.html#p95, 001.html#p96）。

### 事务的隔离级别

管理事务的隔离级别。详细信息请参阅第 226-228 页（索引分割 003.html#p254）。

### INCLUDING 子句

`INCLUDING` 子句。详细信息请参阅第 440 页（索引分割 005.html#p468）。

### INCREMENT BY 选项（序列）

`INCREMENT BY` 选项（序列）。详细信息请参阅第 199 页（索引分割 002.html#p227）。

### ISSCHEMAVALID 函数

`ISSCHEMAVALID` 函数。详细信息请参阅第 348 页（索引分割 004.html#p376）。

### 解释执行计划

解释执行计划。详细信息请参阅第 483, 488 页（索引分割 006.html#p511, 006.html#p516）。

忽略聚合数据集中的分组。详细信息请参阅第 28 页（索引分割 000.html#p56）。

连接方法，以及解释执行计划。详细信息请参阅第 486 页（索引分割 006.html#p514）。

### 图像数据，加载到 BLOB 列

将图像数据加载到 `BLOB` 列中。详细信息请参阅第 387-389 页（索引分割 005.html#p415）。

### INITCAP 函数

`INITCAP` 函数。详细信息请参阅第 99 页（索引分割 001.html#p127）。

### 初始化文件

初始化文件的位置。详细信息请参阅第 405 页（索引分割 005.html#p433）。

### INITIALLY DEFERRED 约束

`INITIALLY DEFERRED` 约束。详细信息请参阅第 220 页（索引分割 003.html#p248）。

### INITIALLY IMMEDIATE 约束

`INITIALLY IMMEDIATE` 约束。详细信息请参阅第 220 页（索引分割 003.html#p248）。

### 内联视图特性

内联视图特性。详细信息请参阅第 14 页（索引分割 000.html#p42）。

### 内联视图

使用查询作为内联视图。详细信息请参阅第 32 页（索引分割 000.html#p60）。

### LOCAL 子句（CREATE INDEX 语句）

`LOCAL` 子句（`CREATE INDEX` 语句）。详细信息请参阅第 378 页（索引分割 005.html#p406）。



PIVOT, 89—96

本地索引

`PIVOT XML`, 94](#index_split_001.html#p122)

创建, 377—](#index_split_005.html#p405)380

`PRIOR`, 315—3](#index_split_004.html#p343)17

与全局索引比较, 381](#index_split_005.html#p409)

`SIBLINGS`, 318—](#index_split_004.html#p346)319

本地命名方法, 452](#index_split_006.html#p480)

`SOME`, 122—](#index_split_001.html#p150)124

锁定待更新的行, 133—](#index_split_002.html#p161)137

`UNION`, 35, 62—](#index_split_000.html#p63)64

逻辑结构，之间的关系

`UNION ALL`, 34—](#index_split_000.html#p62)35, 74—76

物理结构与, 238](#index_split_003.html#p266)

`UNPIVOT`, 95—97](#index_split_001.html#p123)

查找值，确保存在, 448—](#index_split_006.html#p476)449

Knuth, Donald, 163](#index_split_002.html#p191)

`LPAD function`, 317](#index_split_004.html#p345)

**L**

**M**

`LAG function`, 42—](#index_split_000.html#p70)45, 55—56

大对象。`参见` `LOBs`

维护操作, `UPDATE`

`LAST_ANALYZED column`, 493](#index_split_006.html#p521)

`INDEXES clause` 用于, 381](#index_split_005.html#p409)

`LAST_DAY function`, 155—](#index_split_002.html#p183)156

在连接条件中处理 `NULLs`, 80—](#index_split_001.html#p108)81

`LAST function`, 48](#index_split_000.html#p76)

`LEAD function`, 42—](#index_split_000.html#p70)45, 55—56

匹配数据，在表间查找, 68—70](#index_split_001.html#p96)

叶数据，在层次表中识别,

`MAX function`, 24, 271](#index_split_003.html#p299)324—328

`MAXVALUE option` (序列), 199](#index_split_002.html#p227)

闰年，检测, 154—155](#index_split_002.html#p182)

成员，在组和集中计数, 33—](#index_split_000.html#p61)35

`LEFT OUTER JOIN`, 64—](#index_split_001.html#p92)65

`LEVEL pseudo-column`, 314—317](#index_split_004.html#p342)

`MERGE PARTITIONS clause` (`ALTER TABLE statement`), 373—374](#index_split_005.html#p401)

级别，在多个级别聚合数据, 30—](#index_split_000.html#p58)32

`LIKE operator`, 127, 168—](#index_split_002.html#p155)169

`MERGE statement`, 21](#index_split_000.html#p49)

限制每个会话的资源, 413—](#index_split_005.html#p441)416

合并分区, 373—](#index_split_005.html#p401)374

线性回归分析, 130—](#index_split_002.html#p158)133

关于数据库的元数据，查看, 229—](#index_split_003.html#p257)230

语言比较逻辑, 20](#index_split_000.html#p48)

方法

语言排序, 294](#index_split_004.html#p322)

用于生成 SQL 资源使用情况跟踪

列表，按列表分区, 355—356](#index_split_004.html#p383)

文件, 478](#index_split_006.html#p506)

数字列表，自动生成, 204—205](#index_split_003.html#p232)

统计的，支持线性回归

分析, 132](#index_split_002.html#p160)

加载

`MIN function`, 24, 271](#index_split_003.html#p299)

批量加载 `LOBs`, 389—](#index_split_005.html#p417)391

`MINUS operator`, 71—](#index_split_001.html#p99)73, 308

图像数据到 `BLOB columns`, 387—](#index_split_005.html#p415)389

缺失的行，查找, 71—](#index_split_001.html#p99)73

大型文档到 `CLOB columns`,

`MOD function`, 155](#index_split_002.html#p183)



384—386

`修改密码`，421—422

`大型对象 (LOBs)`

`取模运算`，155

`通过 HTTP 访问`，391—396

`监控`

`批量加载`，389—391

`索引使用`，513—514

`表中数据的删除或更新`，398—400

`性能`，457

530

[www.it-ebooks.info](http://www.it-ebooks.info/)

■ 索引

`实时 SQL 执行统计`，457—459

`规范化数据库`，数据存储于，83

`非数字 (NAN) 值`，192—194

`用于空闲空间的表空间`，510

`NOT 布尔运算符`，5

`月份`

`NOT EXISTS 谓词`，117—118

月份中的第一天，确定，156—157

`NOT NULL 约束`，451

月份中的最后一天，确定，155—156

`NOT 前缀`，169

`MOVE PARTITION 子句 (ALTER TABLE 语句)`，365

`NOVALIDATE 子句`，303

`NOWAIT 子句 (DML 语句)`，214—216

`移动`

`第 N 个结果，确定`，275—277

`数据从一个源到另一个`，287

`NTILE 函数`，272

`表`，429

`空值/NULL 值`

`自动更新的行`，365—366

`转换为真实值`，109—111

`多个字段`，更新，11—12

`查找与消除`，16—18

`多版本控制`，226

`在连接条件中`，操作与比较，80—81

`在数值计算中`，200—201

■ `N`

`排序`，112—113

`NULLS FIRST ORDER BY 选项`，112

`命名空间`

`NULLS LAST ORDER BY 选项`，112

用于数据库相关的 XML，348

`NUMBER 数据类型`，190—191

`对象与`，432

`数字`

`命名对象`，与命名空间，432

`连续的`，生成，196—197

`XML 数据的原生存储`，339—341

`不同进制间的转换`，124—127

`NATURAL JOIN 子句`，与 ANSI SQL 99，127

`连接语法`，62

`列表`，自动生成，204—205

`负数`，处理，126

`NEXTVAL 伪列`，196—197

`负值`，处理，126

`NLS_CHARACTERS 选项`，195

`自动四舍五入`，202—204

`NLS_COMP 选项`，20—21

`存储`，189—190

`NLS_COMP 参数 (ALTER TABLE 语句)`，293—294

`在字符串中`，验证，194—196

`NUMERIC 数据类型`，191

`NLS_CURRENCY 选项`，195

`数值数据类型`


## 索引

NLS 指令， 19

在两者间转换， 188—189

`NLS_LANGUAGE` 字符串， 195—196

在字符串与两者间转换， 187—188

`NLS_SORT` 选项， 20

确定数据是否可加载为，

`NLS_SORT` 参数 (`ALTER SESSION` 语句)， 290—291, 293—294

数值等价物，将字符串转换为，

`NOAUDIT` 语句， 515, 100—102

`NOCACHE` 选项 (序列)， 199

`NVL` 函数， 80, 109—111, 200

`NOCYCLE` 选项 (序列)， 199

`NVL2` 函数， 99, 110—111

节点，在层次结构级别内排序， 318—321

`NOLOGGING` 子句， 364

## O

非数字，执行计算， 192—193

`OBF` 函数， 296

无前缀的本地索引， 379—380

混淆敏感值， 294—297, 531

[www.it-ebooks.info](http://www.it-ebooks.info/)

对象依赖关系，显示， 247—250

`ESCAPE`, 169

对象级权限，显示， 256—257

`INNER JOIN`, 62

对象管理

`INTERSECT`, 68—70

为对象创建别名， 443—445

`LIKE`, 127

`MINUS`, 71—73

创建自增值， 453—455

`PIVOT`, 89—96

`PIVOT XML`, 94

检查数据是否符合条件， 450—452

`PRIOR`, 315—317

创建数据库之间的连接， 452—453

`SIBLINGS`, 318—319

`SOME`, 122—124

确保查找值存在， 448—449

`UNION`, 35, 62—64

索引

`UNION ALL`, 34—35, 74—76

位图索引，创建， 439

`UNPIVOT`, 95—97

创建， 435—436

`OPTIMAL_LOGFILE_SIZE` 列 (`V$INSTANCE_RECOVERY` 视图)， 504

基于函数的索引，创建， 438—439

重命名对象， 430—432

临时存储数据， 427—428

优化器统计信息，查看， 492—494

表

优化行和表锁定， 214—216

创建， 425—427

可选连接

删除， 433—434

双向， 65—67

强制唯一行， 445—448

编写， 64—65

索引组织表，创建， 440

Oracle

移动， 429

```
if (condVar > someVal) {console.log("xxx")}
```


# 数据库参考指南

## 取消删除
页码 434-435。

## 11g 第 2 版
页码 275。

## 视图，创建
页码 441-442。

## 企业管理器数据库控制
用于对象关系型存储 XML 数据，页码 338。

页码 222。

## 对象
用于数据库相关 XML 的命名空间。

### 替代名称，创建
页码 443-445。

页码 348。

## BFILE
页码 383。

## SQL Developer
页码 222-224，267-268。

## 重命名
页码 430-432。

## 撤消顾问
页码 506-507。

### 大小，显示
页码 511-513。

## oradebug 实用程序，启用和禁用跟踪
### 使用情况，审计
页码 514-516。

### 与...一起使用
页码 482。

## 对象，大型
*参见* LOBs。

## ORA_ROWSCN 伪列
页码 148-149。

## ON 子句，与 ANSI SQL 99 连接语法
页码 61。

## OR 布尔运算符
页码 5。

## ON COMMIT PRESERVE ROWS
页码 427。

## OPEN_CURSORS 初始化参数
页码 501-503。

### 描述
页码 27。

### 条件排序与...
页码 120-122。

## 操作系统，用于识别资源密集型查询
页码 469-471。

### 对空值排序与...
页码 112。

### 用于对结果排序
页码 6-7。

## 运算符
*参见* 布尔运算符。

## ORDER 选项（序列）
页码 199。

## ORDER BY 子句
页码 501-503。

### 描述
页码 27。

### 条件排序与...
页码 120-122。

### 对空值排序与...
页码 112。

### 用于对结果排序
页码 6-7。

## ORDER SIBLINGS BY 子句
页码 315，318-319。

## ALL
页码 122-124。

## ORGANIZATION INDEX 子句
页码 440。

## ANY
页码 94，122-124。

## OS Watcher 工具套件
页码 471。

## DEFERRABLE
页码 218-221。

## 外连接
页码 532。

## FULL
页码 65-67。

## DISTINCT
页码 95，164-165。

## LEFT
页码 64-65。

## RIGHT
页码 65。

## OVER 子句
页码 44。

### 为分区生成统计信息
页码 377。

### 按哈希
页码 356-357。

## OVERFLOW 子句
页码 440。

### 间隔
页码 359-360。

### 按列表
页码 355-356。

### 合并分区
页码 373-374。

### 按范围
页码 354-355。

### 按引用约束
页码 360-362。

## ■ P

### 从分区中移除行
页码 376。

### 重命名分区
页码 371-372。

## PAGESIZE 参数
页码 298。

### 拆分分区
页码 372-373。

## 对查询结果进行分页
页码 113-117。

### 策略
页码 352-353。

## 分页技术
页码 117。

### 相关术语
页码 351-352。

## 重叠日期范围，查找
页码 146-148。

## www.it-ebooks.info
[`www.it-ebooks.info/`](http://www.it-ebooks.info/)



## 并行性，351

## 关于虚拟列，362—363

## 参数化报告，266—269

## 分区键，355

## 父/子关系，218

## 密码

### 截断父表，299—301

### 大小写敏感性，423

### 强制复杂性要求，422—423

### 修改密码，421—422

### 密码安全设置，415—416

### `PASSWORD_VERIFY_FUNCTION`（默认配置文件），422—423

## 从层次表生成路径名，321—324

## 模式
*另见* 正则表达式

### 计数，179—181

### 提取，178—179

### 生成数字到模式，198—199

### 搜索模式，174—178

## `PCTFREE` 子句，364

## `PCTTHRESHOLD` 子句，440

## `PCTUSED` 子句，364

## 在字符串中搜索百分号 (%) 字符，169

## 性能

### 影响性能的常见等待事件，499—501

### 监控性能，457

### 提高字符串搜索的性能，184—186

## 性能报告

### `ADDM`，466

### `ASH`，466—467

### `AWR`，465—466

### 使用性能报告识别资源密集型 SQL 语句，468—469

### 概述，465 533

### `Statspack`，467—468

## 分区

### 向分区表添加分区，368—369

### 应用程序控制的分区，363—364

### 自动移动更新的行，365—366

### 复合分区，357—359

### 使用表空间配置分区，364—365

### 创建映射到分区的索引，377—380

### 创建具有自身分区方案的索引，380—381

### 分区描述，351

### 确定是否需要分区，353—354

### 删除分区，375—376

### 与现有表交换分区，369—371

### 对现有表进行分区，366—368

## `PARTITION BY HASH` 子句（`CREATE TABLE` 语句），356—357

## `PARTITION BY LIST` 子句（`CREATE TABLE` 语句），356

## `PARTITION BY RANGE` 子句（`CREATE TABLE` 语句），354—355

## `PARTITION BY REFERENCE` 子句（`CREATE TABLE` 语句），361—362

## `PARTITION BY SYSTEM` 子句（`CREATE TABLE` 语句），363—364

## 在移动窗口上执行聚合，49—51

## 悲观锁定方法，133—137

## `PFILE` 子句（`STARTUP` 命令），405

## 幻读，227

## 物理结构及其之间的关系

## 查询结果。*参见* 结果

■ 索引

■ **Q**

■ **R**



logical structures and, 238

random data, generating, 102—105

`pipe (||)` operator, 97

`RANDOM` function, 102—105

数据透视 (pivoting)
通过范围分区 (range, partitioning by), 354—355
在多个列上 (on multiple columns), 92—95
范围分区全局索引 (range-partitioned global indexes), 380
将行转换为列 (rows into columns), 91

`RANK` function, 45—47

`PIVOT` keyword (`SELECT` statement), 89—97
readability, concatenating data for, 97—99

`PIVOT XML` clause (`SELECT` statement), 94
按 ANSI SQL 隔离级别划分的读异常 (read anomalies by ANSI SQL isolation level),
PL/SQL
227
`DBMS_UNDO_ADV` package, 506
跨事务的读一致性 (read-consistency across transactions),
error message, 418
确保 (ensuring), 225—226
`POSIX` (Portable Operating System
实时 SQL 执行统计信息 (real-time SQL execution statistics),
Interface) standard, 176
监控 (monitoring), 457—459
精度 (precision)
`REBASE_NUMBER` function, 创建 (creating), 124—126
二进制 (binary), 189—190, 193
数据类型 (data type), 190—192
重建索引 (rebuilding indexes), 429, 436
十进制 (decimal), 189—190
`RECYCLEBIN` feature, 433, 435
正数和负数 (positive and negative), 203
重做日志大小调整 (redo logs, sizing), 503—504
预测序列结束之外的数据值和趋势 (predicting data values and trends beyond series end), 130—133
在文件系统上引用二进制对象 (referencing binary objects on file system), 396—398
前缀本地索引 (prefixed local indexes), 379—380
按引用约束分区 (referential constraints, partitioning by), 360—362
主键约束 (primary key constraints), 445—447
显示主键关系 (primary key relationships, showing), 246—247
引用完整性 (RI) (referential integrity (RI)), 218
生成固定数量的顺序主键 (primary keys, generating fixed number of sequential), 330—333
`PRIOR` operator, 315—317
权限 (privileges)
与权限相关的数据字典视图 (data dictionary views related to), 259
描述 (description of), 167
显示对象级权限 (object-level, displaying), 256—257
配置文件设置 (profile settings), 414—415
显示系统级权限 (system-level, displaying), 257—259
查询执行进度 (progress in execution of queries)
确定 (determining), 462—463
显示 (displaying), 459—462
`ps` utility (Linux/Unix), 469—471
`PURGE RECYCLEBIN` statement, 433
`REGR_R2` function, 133
`REGEXP_COUNT` function, 180—181
`REGEXP_INSTR` function
字符串内的字符替换 (character substitutions within strings), and, 173
描述 (description of), 167
搜索模式 (patterns, searching for), 174—178
替换字符串中的文本 (replacing text in strings and), 183—184
搜索子字符串 (substrings, searching for), 170
`REGEXP_REPLACE` function, 167, 182—184
`REGEXP_SUBSTR` function
描述 (description of), 167
提取模式 (patterns, extracting), 178—179
提取子字符串 (substrings, extracting), 171


# 正则表达式

*另见* 模式

534

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 索引

描述，175](#index_split_002.html#p203)

与性能报告一起使用，465—469](#index_split_006.html#p493)469

示例，177

与 `V$SQLSTATS` 视图一起使用，463—465](#index_split_006.html#p491)

元字符，175—176

`RESOURCE_LIMIT` 初始化参数，

相对排名，按此过滤结果，275—277](#index_split_003.html#p303)

413—414

移除

每个会话的资源，限制，413—416](#index_split_005.html#p441)416

表中的所有行，13](#index_split_000.html#p41)

恢复已删除的表，434—435](#index_split_005.html#p462)

字符串中的字符，172

结果

表中的数据，305—307](#index_split_004.html#p333)

第 n 个，确定，275—277](#index_split_003.html#p303)

基于列子集的重复行，51—55](#index_split_000.html#p79)

分页，113—117](#index_split_001.html#p141)

表中的行，6—7](#index_split_000.html#p34)

表中的行，12—13

为报告将数据排序到等大小的桶中，

基于其他表中的数据的行，67—68](#index_split_001.html#p95)

271—273](#index_split_003.html#p299)

来自分区的行，376](#index_split_005.html#p404)

垂直堆叠, 62—64

重命名

对象，430—432

分区，371—372](#index_split_005.html#p399)

视为表的内容，14—](#index_split_000.html#p42)15

从表中检索数据，3—5

在分组结果中返回详细列，

`REPLACE` 函数，167, 29](#index_split_002.html#p195)7

269—271

替换字符串中的文本，182—](#index_split_002.html#p210)184

返回不存在的行, 87—89

报告

`RIGHT OUTER` JOIN，65](#index_split_001.html#p93)

在分组结果中返回详细列，

RI (参照完整性)，218](#index_split_003.html#p246)

返回，269—271](#index_split_003.html#p297)271

角色名称，与 `USER$` 表，255](#index_split_003.html#p283)

包含图形或直方图，279—280](#index_split_003.html#p307)280

`ROLE_ROLE_PRIVS` 视图，255](#index_split_003.html#p283)

创建直方图，273—275

授予用户的角色，显示，254—255](#index_split_003.html#p282)255

在数据集上比较假设，

`ROLE_SYS_PRIVS` 视图，258](#index_split_003.html#p286)

277—279

`ROLE_TAB_PRIVS` 视图，256](#index_split_003.html#p284)

概述，263](#index_split_003.html#p291)

`ROLLBACK`，212](#index_split_003.html#p240)

参数化，266—269](#index_split_003.html#p294)

回滚数据，与临时表重做日志，428](#index_split_005.html#p456)

相对排名，按此过滤结果，275—277](#index_split_003.html#p303)277

部分回滚事务，209—212](#index_split_003.html#p237)

将结果排序到等大小的桶中

`ROLLUP` 函数，31—32](#index_split_000.html#p59)

，271—273

`ROUND` 函数，202](#index_split_003.html#p230)

避免其中的行重复，263—266](#index_split_003.html#p291)

自动四舍五入数字, 202—204

直接从数据库生成网页报告，

显示表的行数，240—241](#index_split_003.html#p268)241

，280—285

`ROWID` 值，429—430](#index_split_005.html#p457)430

`REPORT_SQL_MONITOR` 函数

## 索引

## 行锁定
* 214—216

## `DBMS_SQLTUNE` 包
* 460—

## `ROW_NUMBER` 函数
* 462

## `CASE` 表达式
* 与，263—266

## 序列号
* 重置，455

## 隐式页面计算
* 与，113—117

## 调整大小
* 确定第 n 个结果，275—277
* 表空间，412—413
* 概述，51—53
* 临时表空间数据文件，509

## `ROWNUM` 伪列
* 113—117, 204—205

## 远程数据库
* 解析位置，452

## 资源密集型查询，识别方法
* 地址，429—430
* 为其重建索引，436
* 示例，59—60

## 资源密集型 SQL 语句，识别
* 535

## 行
* 访问后续或前序行的值，42—45
* 向表中添加行，7—9
* 在查询结果中为行分配排名值，45—47
* 将列转换为行，95—96
* 将行转换为列，89—91
* 更改行中的值，10—11
* 在表之间复制行，9—10
* 在表之间复制多行，10
* 检测重复行，287—289
* 查找缺失行，71—73
* 从两个或多个表中连接对应行，60—62
* 为更新锁定行，133—137
* 从表中删除所有行，13
* 基于其他表中的数据删除行，67—68
* 删除重复行，289—290
* 从分区中删除行，376
* 从表中删除行，12—13
* 显示行之间的差异，307—311

## XML
* 验证，346—349

## `SCN` (系统变更号)
* 148

## `SCN_TO_TIMESTAMP` 函数
* 148—149

## 数据清洗
* 执行大小写不敏感查询，293—294
* 禁用约束，299—304
* 从表中删除数据，305—307
* 确定数据是否可作为日期加载，291—293
* 确定数据是否可作为数值加载，290—291
* 显示模式之间的差异，307—311
* 删除所有索引，297—299
* 检测重复行，287—289
* 删除重复行，289—290
* 在表中强制唯一性，445—448
* 概述，287
* 基于列子集删除重复项，51—55
* 禁用触发器，304—305
* 混淆值，294—297

## 搜索
* 按模式搜索，174—178

[www.it-ebooks.info](http://www.it-ebooks.info/)


`字符串`，127—130 页

在报告中避免重复，263—266 页

安全问题

返回不存在的值，87—89 页

`ALL_USERS`视图，234 页

表之间没有共同的列，

混淆敏感值，294—297 页

查找，73—76 页

`SELECT ... FOR UPDATE`选项，134—135 页

选择

更新，自动移动，365—366 页

表中的所有列，5—6 页

基于其他表中的数据进行更新，

来自另一个查询的结果，14—15 页

78—80 页

`SELECT`选项，`INSERT`语句，9—10 页

`RPAD`函数，279—280 页

`SELECT`语句

`HAVING`子句，288 页

内联视图，14 页

■**S**

在其中嵌套函数，85 页

`IN`或`EXISTS`子句，67—68 页

`SAVEPOINT`语句，210—212 页

`PIVOT`关键字，89—97 页

可扩展性功能，351 页

`PIVOT XML`子句，94 页

选择规模，190—192 页

引用函数，85 页

模式

从表中检索数据，3—4 页

描述，234 页

`SELECT *`语句，6 页

显示信息，233—235 页

在表中查找序列间隙，55—57 页

536 页

[www.it-ebooks.info](http://www.it-ebooks.info/)

■ 索引

序列号，确定间隙，

空间管理视图，239—240 页

87—89 页

稀疏排名，46 页

序列

`SPLIT PARTITION`子句（`ALTER TABLE`语句），372—373 页

创建，454 页

用于生成奇数，198—199 页

拆分分区，372—373 页

重置编号，455 页

`SPOOL`和`SPOOL OFF`命令，107 页

`SES_ACTIONS`列

`SPOOL`命令，299 页

（`DBA_AUDIT_OBJECT`视图），515 页

SQL Developer，222—224，267—268 页

会话，跟踪所有 SQL 语句

SQL*Loader，批量加载 LOB，389—391 页

`ALTER SESSION`，480—481 页

`ALTER SYSTEM`，481 页

使用`DBMS_MONITOR`，479 页

与号和双与号标记技术，267—269 页

使用`DBMS_SESSION`，479 页

`BREAK`功能，263—264 页

使用`DBMS_SUPPORT`，480 页

`PASSWORD`命令，422 页

使用`DBMS_SYSTEM`，480 页

使用`oradebug`，482 页

SQL Tuning Advisor，使用，488—490 页

概述，476—479 页

垂直堆叠查询结果，62—64 页

`SET AUTOTRACE`语句，483 页

`STARTUP`命令，`PFILE`子句，405 页

`SET`命令，298 页



# STARTUP 和 SET 命令

## STARTUP NOMOUNT 语句
STARTUP NOMOUNT statement, 405

SET CONSTRAINTS ALL DEFERRED

## START WITH 子句
START WITH 子句
statement, 220

层次化 数据与, 314

## SET 命令
SET CONSTRAINT statement, 220
起始树遍历 与, 316—317

SET LONG command, 442

START WITH option (sequences), 199

集合，计算其中的成员 在, 33—35

语句，追踪所有会话语句
SET TRANSACTION command, 225—226
- ALTER SESSION 与, 480—481
- 显示。*参见* displaying
- ALTER SYSTEM 与, 481
- 与 DBMS_MONITOR, 479
- 与 DBMS_SESSION, 479
- 与 DBMS_SUPPORT, 480
- 与 DBMS_SYSTEM, 480
- oradebug 与, 482
- 概述 of, 476—479

# 子句与关键字

## ORDER BY 子句中的 SIBLINGS 关键字
SIBLINGS 关键字 (ORDER BY 子句), 318—319

# 操作符与函数

## SOME 操作符
SOME 操作符, 122—124

## STATS_T_TEST_INDEPU 函数
STATS_T_TEST_INDEPU function, 279

## SYS_XMLGEN 函数
SYS_XMLGEN 函数 of, 335—338

# 数据处理

## 排序
sorting
- 按字母顺序，不区分大小写 方式, 18—20
- 二进制和语言学 , 294
- 条件和按函数 , 120—122
- 层次结构级别内的节点 在, 318—321
- 在空值 上, 112—113
- 选项 for, 20—21
- 结果, 6—7

## 字符串处理
strings
- *另请参见* substrings
- 在数值数据类型之间转换 `与, 187—`188
- 将日期时间值转换为 可读格式, 143—145
- 转换为日期时间 值, 145—146
- 在字符串列中执行单字符替换 在, 172—174

# 数据存储与统计

## 统计数据
statistics
- 生成 of, 494—496
- 为分区生成 在, 377
- 使用 Statspack 识别资源密集型 SQL 语句, 467—468
- Statspack 报告，用途, 497

## 存储
storing
- 货币信息, 190—192
- 临时数据 y, 427—428
- 数字, 189—190
- 结果存入大小相等的存储桶用于报告 在, 271—273
- 空间消耗大的对象，显示前 N, 353
- 以本机形式存储 XML 在, 339—341
- XMLTYPE 数 [据, 338
- 在规范化数据库中存储数据, 83

# 数据库对象与系统信息

## 表与分区
tables
- *另请参见* partitioning
- 表锁定, 214—216

## 数据字典视图
静态数据字典视图 在, 230—231

## 系统对象
- 系统变更号 (SCN), 148
- 系统权限，显示 在, 257—259
- SYSTEM 方案，与 SYS 方案比较, 233

# 其他

## XML 处理
将 XML 分解用于关系型 用途, 341—343

[www.it-ebooks.info](http://www.it-ebooks.info/)

■ 索引


## 索引

-   概述 of, 167
-   向表中添加行 , 7—9
-   替换文本 in, 182—184
-   从一个表批量复制数据到另一个表, 10
-   搜索 in, 127—130
-   转换为数字等价形式 , 100—102
-   将行从一个表复制到另一个表 , 9—10
-   创建 ing, 425—427
-   加速字符串搜索 , 184—186
-   在大对象中删除或更新数据 in, 398—400
-   构建 `STRING_TO_LIST` 函数, building, 39—42
-   子查询
    -   确定对子查询的访问 access to, 235—237
    -   显示子查询的磁盘空间使用情况 , displaying, 237—240
    -   相关子查询, 118
    -   删除 ing, 433—434
    -   将查询用作子查询 as, 32
    -   `DUAL` 子查询, 77
    -   子选择返回意外的多个值, 122—124
    -   动态生成子查询的同义词, 444
-   替换变量 iables, 268
    -   在替换变量中强制唯一行 , 445—448
    -   在替换变量中查找重复值和唯一值, 35—37
-   `SUBSTR` 函数 ion, 126, 170—171
-   子字符串
    -   提取 ing, 170—171
    -   跨子字符串查找匹配数据 , 68—70
    -   搜索 ing for, 167—170
-   查找表之间共有的行, 73—76
-   计算小计 , calculating, 37—38
-   汇总
    -   在汇总中查找序列缺口, 55—57
    -   为不同组汇总数据, 26—27
    -   从汇总中生成路径名 ing, 321—324
    -   汇总列中的值, 23—25
-   表
    -   全局临时表 , 428
    -   抑制行中重复的前导数据, 263—266
    -   表的高水位线, 306
    -   识别表中的叶子数据 in, 324—328
    -   同步两个表的内容 , 138—140
    -   索引与表, 241—242, 437
    -   创建索引组织表, 440
    -   连接两个或多个表中的对应行 , 60—62
    -   移动 ing, 429
    -   截断父表 ing, 299—301
    -   从表中查询 , 59
    -   从表中移除所有数据 ing, 305—307
    -   从表中移除所有行 ing, 13
    -   根据其他表中的数据移除行, 67—68](#index_split_001.html#p95)
    -   从表中移除单行 ing, 12—13
-   `SYS` 模式与 `SYSTEM` 模式比较, 233
-   创建对象的同义词, 443—445
-   显示同义词元数据, 250—251
-   `SYS_CONNECT_BY_PATH` 函数, 315,322—324
-   `SYS_CONTEXT` 函数, 234, 236
-   `SYSDATE` 函数, 45
-   [www.it-ebooks.info](http://www.it-ebooks.info/)

■ 索引


*   `tnsnames.ora` 文件，第 452 页
*   重命名，第 430–431 页
*   `TO_BINARY_DOUBLE` 函数，第 189 页
*   检索数据，第 3–5 页
*   `TO_CHAR` 函数
    *   显示行数，第 240–241 页
    *   字符限制，第 145 页
    *   选择所有列，第 5–6 页
    *   将十进制转换为十六进制
    *   同步两个数字的内容，第 138–140 页
    *   数字，第 124 页
    *   恢复删除，第 434–435 页
    *   `DAY` 和 `D` 格式代码，第 158 页
    *   基于其他表中的数据更新行
    *   格式化日期时间值，第 144 页
    *   第 78–80 页
    *   与 `GROUP BY` 子句，第 159–160 页
    *   `TABLESPACE` 子句 (`CREATE TABLE` 语句)，第 364 页
    *   提高输出可读性，第 85 页
    *   用途，第 38、85 页
*   表空间
    *   `TO_DATE` 函数，第 145–146 页
    *   配置分区，第 364–365 页
*   `TO_NUMBER` 函数
    *   转换字符串和数值数据类型
    *   在字符串中验证数字，第 194–196 页
    *   示例，第 59 页
    *   计算总量，第 37–38 页
    *   调整大小，第 412–413 页
*   跟踪会话的所有 SQL 语句
    *   表、索引，第 437 页
    *   使用 `ALTER SESSION`，第 480–481 页
    *   临时表，调整大小，第 507–509 页
    *   使用 `ALTER SYSTEM`，第 481 页
    *   UNDO 表空间，调整大小，第 505–507 页
    *   使用 `DBMS_MONITOR`，第 479 页
    *   临时表空间，调整大小，第 507–509 页
    *   使用 `DBMS_SESSION`，第 479 页
    *   分区术语，第 351–352 页
    *   使用 `DBMS_SUPPORT`，第 480 页
    *   三元逻辑规则，第 200 页
    *   使用 `DBMS_SYSTEM`，第 480 页
    *   测试数据，生成，第 76–78 页
    *   使用 `oradebug`，第 482 页
*   测试
    *   概述，第 476–479 页
    *   基于 `CASE` 的测试，第 111 页
*   跟踪
    *   数据存在性，第 117–118 页
    *   日期和时间数据的更改，第 148–149 页
    *   `T-tests`，第 277–279 页
    *   唯一网站访问者的天数，第 164–165 页
*   文本
    *   与视图关联的文本，显示，第 251–252 页
*   事务
    *   开始，第 212 页
    *   在字符串中替换，第 182–184 页
    *   阻塞，识别，第 213–214 页
    *   时间概念，第 143 页
    *   约束验证，延迟，第 218–225 页
    *   按时间段分组和聚合，第 159–160 页
    *   死锁，避免，第 216–218 页



# ■ 索引

## ■**U**

撤销顾问 (Oracle), `506—507`

撤销表空间，大小调整, `505—507`

恢复已删除的表, `434, 435`

UNION ALL 运算符, `34—35, 74—76`

UNION 运算符
- 垂直堆叠查询结果, `62—64`
- 与 UNION ALL 对比, `35`

UNIQUE 关键字, `447`

UNOBF 函数, `296`

UNPIVOT 运算符, `95—97`

UPDATE INDEXES 子句
- ALTER TABLE 语句, `369, 374`
- 用于维护操作, `381`

UPDATE 语句
- 用于更改数据, `11`
- 用于更新多个字段, `11—12`

UPDATEXML 函数, `349—350`

更新
- 条件更新，基于存在性, `21—22`

隔离级别，管理, `226—228`

时间
- 概述, `209`
- 时区之间转换, `152—153`
- 部分回滚, `209—212`
- 跟踪变更, `148—149`
- 确保跨时间的一致性读, `225—`

时间序列，从数据生成无间隔序列
- 存在间隔的情况, `150—152`

行和表锁定，优化, `214—`

TIMESTAMP 数据类型, `152—153, 216`

TIMESTAMP WITH TIME ZONE 数据类型, `152—153`

TRANSLATE 函数, `167, 172, 295—296`

转换
- 另见 转换
- SQL 转 XML, `335—339`
- 字符串转为数字等价物, `100—102`

遍历层级数据，从顶至
- 底, `315—318`
- 树状结构数据

生成固定数量的序列
- 主键, `330—333`

从层级表生成路径名
- 表, `321—324`

识别其中的环, `329—330`

识别层级表中的叶数据
- 表, `324—328`

概述, `313—315`

在层级内对节点排序
- 级, `318—321`

从顶至底遍历, `315—318`

趋势，预测序列结束后的趋势, `130—133`

触发器，禁用, `304—305`

三值表达式, `18`

故障排除
- 细粒度审计, `516—518`
- 诊断数据库问题, `497—501`
- 显示打开的游标, `501—503`
- 索引使用情况监控, `513—514`

_ (下划线) 字符，在字符串中
- 搜索, `169`

tkprof 工具, `477, 539`

[www.it-ebooks.info](http://www.it-ebooks.info/)



对象大小，显示，511—513

表中的大对象，398—400

对象使用情况，审计，514—516

基于其他表中的数据的行，78—80

联机重做日志，大小，503—504

`USER_CONSTRAINTS` 视图，243

表空间使用率，显示，509—511

`USER_DB_LINKS` 视图，453

临时表空间，大小，507—509

`USER_INDEXES` 视图，242

Undo 表空间，大小，505—507

`USER_IND_STATISTICS` 视图，493

`TRUNCATE` 命令，13，306—307

用户信息，显示，233—235

`TRUNCATE PARTITION` 子句（`ALTER TABLE` 语句），376

`USER_ROLE_PRIVS` 视图，255

用户
截断父表，299—301
创建，419—420
删除，420—421

`TRUNC` 函数，用于确定月份的第一天，157

`USER_SOURCE` 视图，253

`t 检验`，277—279

`USER` 静态视图，230

调优
`SQL Tuning Advisor`，使用，488—490
技术，457

`USER$` 表，与角色名，255

`USER_TABLES` 视图，235—236

`USER_TAB_MODIFICATIONS` 视图，494，516
开启与关闭
对象上的审计，515

`USER_TAB_PRIVS_MADE` 视图，256

跟踪，480—482

`USER_TAB_PRIVS_RECD` 视图，256

双竖线运算符（`||`），97

`USER_TAB_PRIVS` 视图，256

540

[www.it-ebooks.info](http://www.it-it-ebooks.info/)

■ 索引

`USER_TAB_STATISTICS` 视图，493
动态性能，231—232

`USING` 子句，与 ANSI SQL-99 连接语法，60
概述，213
静态，230—231

`UTLDTREE` 脚本，250
动态性能，213，501
为其动态生成同义词，444

`UTL_FILE` 包，396—397

`UTL_HTTP` 包，392—394

**内部，定义，252**

与权限相关的数据字典，259

■ V

空间管理，239—240

视图文本，显示，251—252

验证
字符串中的数字，194—196
`XML` 模式，346—349

`V$INSTANCE_RECOVERY` 视图，
`OPTIMAL_LOGFILE_SIZE` 列，504

值
定义，86
确定，86

虚拟列
基于其分区，362—363
限制，87

*另见* 日期时间值；空值

从后续或前行访问，42—45


# 在查询中为行分配排名

`V$LOCKED_OBJECT` 视图, 214,

结果 s, 45—47

`V$LOCK` 视图, 213—214

`自增`，创建, 453—455

`V$LOG_HISTORY` 视图, 503

将空值转换为真实值, 109—111

`V$OBJECT_USAGE` 视图, 513

在列中，汇总, 23—25

`V$SESSION_LONG`OPS 视图, 462—463

查找组内的第一个和最后一个, 47—

`V$SESSION` 视图, 214, 50248

`V$SESSTAT` 视图, 468

混淆敏感数据, 294—297

`V$SQLAREA` 视图, 459, 464

预测序列结束后的情况, 130—133

`V$SQL_MONITOR` 视图, 457—459

在行中，修改, 10—11

`V$SQL_PLAN_MONITOR` 视图, 459—460

查找表中的唯一值, 35—37

`V$SQLSTATS` 视图, 459, 464

`VARCHAR2` 数据类型，`VARCHAR` 数据类型

`V$SQL` 视图, 459, 464

比较，426

`V$SYSSTAT` 视图, 468, 498, 501

变量

`V$SYSTEM_EVENT` 视图, 501

绑定, 268

`V$UNDOSTAT` 视图, 506

替代, 268

`V$` 视图, 231—232

验证连接信息, 407—409

`VERIFY` 参数, 298

`竖线 (||)` 运算符, 97

## ■ W

查看

关于数据库的元数据, 229—230

常见的等待事件，影响优化器统计信息, 492—494

性能, 499—501

视图

直接从数据库生成网页报告

另请参见 显示；特定视图

数据库, 280—285

`ALL_TAB_`COLS, 130

在网页上显示查询结果, 113—

`ALL_TAB_COLUMN`S, 129—130117

创建, 441—442

网站，跟踪特定天数内的唯一用户数量

数据字典

164—165541

[www.it-ebooks.info](http://www.it-ebooks.info/)

## ■ 索引

周，星期几，确定, 157—159

验证 XML 架构, 346—349

`WHERE` 条件，基于查询, 15—16

`XMLATTRIBUTES` 函数, 345—346

`WIDTH_BUCKET` 函数, 272—275

`XMLELEMENT` 函数, 345—346

窗口，在窗口上执行聚合

`XMLROOT` 函数, 345—346

移动, 49—51

`XMLTABLE` 函数, 341—343

`WITH CHECK` OPTION, 441

`XMLTYPE` 数据的存储机制，

`WITHOUT VALIDATION` 子句, 371338

编写可选连接, 64—65

`XMLTYPE` 数据转换, 339—341

`X$` 表, 231

## ■ X

## ■ Y

XML

原地修改, 349—350

年份

从 XML 文档中提取关键 XML 元素

复活节日期，确定, 162—164

343—344

闰年，检测, 154—155

生成复杂的 XML 文档，

`YEAR TO MONTH` 数据类型, 162

344—346

概述 of, 335

分解以供关系使用, 341—343

## ■ Z

以原生形式存储, 339—341

将 SQL 转换为 XML, 335—339

零长度字符串的处理, 18](#index_split_000.html#p46)

树形结构数据和 XML, 313542

[www.it-ebooks.info](http://www.it-ebooks.info/)

233 Spring Street, New York, NY 10013

优惠有效期至 4/10。

[www.it-ebooks.info](http://www.it-ebooks.info/)

# 文档大纲


# Apress - Oracle SQL 配方：问题-解决方法 (2009 年 11 月) (ATTiCA)

## 面向专业人士的书籍

## 内容概览

## 目录

## 关于作者

## 关于技术审校

## 致谢

## 前言

### 本书适合谁阅读

### 本书结构

### 工具选择

### 使用 SQL 配方的示例数据

### 排版约定

### 联系作者

## 第 1 部分：数据操作基础

### 基础知识

##### 1-1. 从表中检索数据

*   问题
*   解决方案
*   工作原理

#### 1-2. 选择表中的所有列

*   问题
*   解决方案
*   工作原理

##### 1-3. 对结果进行排序

*   问题
*   解决方案
*   工作原理

##### 1-4. 向表中添加行

*   问题
*   解决方案
*   工作原理

##### 1-5. 将行从一个表复制到另一个表

*   问题
*   解决方案
*   工作原理

##### 1-6. 批量将数据从一个表复制到另一个表

*   问题
*   解决方案
*   工作原理

#### 1-7. 更改行中的值

*   问题
*   解决方案
*   工作原理

#### 1-8. 使用一条语句更新多个字段

*   问题
*   解决方案
*   工作原理

#### 1-9. 从表中删除不需要的行

*   问题
*   解决方案
*   工作原理

#### 1-10. 从表中删除所有行

*   问题
*   解决方案
*   工作原理

#### 1-11. 从另一个查询的结果中进行选择

*   问题
*   解决方案
*   工作原理

#### 1-12. 基于查询构建 Where 条件

*   问题
*   解决方案
*   工作原理

#### 1-13. 在查询中查找并消除 NULL

*   问题
*   解决方案
*   工作原理
*   NULL 没有等价物

#### 1-14. 按人的期望进行排序

*   问题
*   解决方案
*   工作原理

##### 1-15. 启用其他排序和比较选项

*   问题
*   解决方案
*   工作原理

#### 1-16. 基于存在性进行条件插入或更新

*   问题
*   解决方案
*   工作原理

### 汇总与聚合数据

##### 2-1. 汇总列中的值

*   问题
*   解决方案
*   工作原理

##### 2-2. 为不同分组汇总数据

*   问题
*   解决方案
*   工作原理

#### 2-3. 按多个字段对数据进行分组

*   问题
*   解决方案
*   工作原理

##### 2-4. 在聚合数据集中忽略分组

*   问题
*   解决方案
*   工作原理

#### 2-5. 在多个层级上聚合数据

*   问题
*   解决方案
*   工作原理

##### 2-6. 在其他查询中使用聚合结果

*   问题
*   解决方案
*   工作原理

#### 2-7. 统计分组和集合中的成员数

*   问题
*   解决方案
*   工作原理

##### 2-8. 查找表中的重复值和唯一值

*   问题
*   解决方案
*   工作原理

##### 2-9. 计算总计和小计

*   问题
*   解决方案
*   工作原理

#### 2-10. 构建您自己的聚合函数

*   问题
*   解决方案
*   工作原理

#### 2-11. 访问后续或前序行中的值

*   问题
*   解决方案
*   工作原理

##### 2-12. 为查询结果中的行分配排名值

*   问题



# 解决方案
## 工作原理
## 2-13. 查找分组内的首值和末值
### 问题
### 解决方案
### 工作原理
##### 2-14. 在移动窗口上执行聚合
### 问题
### 解决方案
### 工作原理
## 2-15. 基于列的子集删除重复行
### 问题
### 解决方案
### 工作原理
## 2-16. 查找表中的序列间隙
### 问题
### 解决方案
### 工作原理
# 从多表查询
##### 3-1. 连接两个或多个表中的对应行
### 问题
### 解决方案
### 工作原理
##### 3-2. 垂直堆叠查询结果
### 问题
### 解决方案
### 工作原理
##### 3-3. 编写可选连接
### 问题
### 解决方案
### 工作原理
## 3-4. 使连接在两个方向上都是可选的
### 问题
### 解决方案
### 工作原理
## 3-5. 基于其他表中的数据删除行
### 问题
### 解决方案
### 工作原理
##### 3-6. 跨表查找匹配数据
### 问题
### 解决方案
### 工作原理
## 3-7. 在聚合上进行连接
### 问题
### 解决方案
### 工作原理
##### 3-8. 查找缺失行
### 问题
### 解决方案
### 工作原理
## 3-9. 查找表之间共有的行所没有的行
### 问题
### 解决方案
### 工作原理
##### 3-10. 生成测试数据
### 问题
### 解决方案
### 工作原理
##### 3-11. 基于其他表中的数据更新行
### 问题
### 解决方案
### 工作原理
## 3-12. 在连接条件中操作和比较 `NULL`
### 问题
### 解决方案
### 工作原理
# 创建和派生数据
##### 4-1. 派生新列
### 问题
### 解决方案
### 工作原理
##### 4-2. 返回不存在的行
### 问题
### 解决方案
### 工作原理
##### 4-3. 将行转换为列
### 问题
### 解决方案
### 工作原理
## 4-4. 在多列上进行透视
### 问题
### 解决方案
### 工作原理
##### 4-5. 将列转换为行
### 问题
### 解决方案
### 工作原理
## 4-6. 为提高可读性而连接数据
### 问题
### 解决方案
### 工作原理
##### 4-7. 将字符串转换为数字等价物
### 问题
### 解决方案
### 工作原理
##### 4-8. 生成随机数据
### 问题
### 解决方案
### 工作原理
##### 4-9. 创建逗号分隔值文件
### 问题
### 解决方案
### 工作原理
# 通用查询模式
## 5-1. 将空值转换为真实值
### 问题
### 解决方案
### 工作原理
##### 5-2. 对空值进行排序
### 问题
### 解决方案
### 工作原理
##### 5-3. 对查询结果进行分页
### 问题
### 解决方案
### 工作原理
##### 5-4. 测试数据的存在性
### 问题
### 解决方案
### 工作原理
## 5-5. 在单条 SQL 语句中进行条件分支
### 问题
### 解决方案
### 工作原理
##### 5-6. 条件排序与按函数排序
### 问题



# 第 2 部分：数据类型及其问题

## 处理日期时间值

##### 6-1. 将日期时间值转换为可读字符串
#### 问题
#### 解决方案
#### 工作原理

##### 6-2. 将字符串转换为日期时间值
#### 问题
#### 解决方案
#### 工作原理

##### 6-3. 检测重叠的日期范围
#### 问题
#### 解决方案
#### 工作原理

### 6-4. 自动追踪数据变更的日期和时间
#### 问题
#### 解决方案
#### 工作原理

### 6-5. 从有间隔的数据生成无缝的时间序列
#### 问题
#### 解决方案
#### 工作原理

### 6-6. 在不同时区之间转换日期和时间
#### 问题
#### 解决方案
#### 工作原理

##### 6-7. 检测闰年
#### 问题
#### 解决方案
#### 工作原理

### 6-8. 计算一个月中的最后一天
#### 问题
#### 解决方案
#### 工作原理

### 6-9. 确定一个月中的第一天或第一个工作日
#### 问题
#### 解决方案
#### 工作原理

##### 6-10. 计算星期几
#### 问题
#### 解决方案
#### 工作原理

### 6-11. 按时间段进行分组和聚合
#### 问题
#### 解决方案
#### 工作原理

### 6-12. 计算两个日期或日期部分之间的差值
#### 问题
#### 解决方案
#### 工作原理

### 6-13. 确定任意年份的复活节日期
#### 问题
#### 解决方案
#### 工作原理

##### 6-14. 计算网站的“X 日活跃用户”
#### 问题
#### 解决方案
#### 工作原理

## 字符串

### 7-1. 搜索子串
#### 问题
#### 解决方案
#### 工作原理

### 7-2. 提取子串
#### 问题
#### 解决方案
#### 工作原理

##### 7-3. 单字符字符串替换
#### 问题
#### 解决方案
#### 工作原理

##### 7-4. 搜索模式
#### 问题
#### 解决方案
#### 工作原理

##### 7-5. 提取模式
#### 问题
#### 解决方案
#### 工作原理

##### 7-6. 计数模式
#### 问题
#### 解决方案
#### 工作原理

##### 7-7. 替换字符串中的文本
#### 问题
#### 解决方案
#### 工作原理

### 7-8. 加速字符串搜索
#### 问题
#### 解决方案
#### 工作原理

## 处理数字

### 8-1. 在字符串和数字数据类型之间转换
#### 问题
#### 解决方案
#### 工作原理

### 8-2. 在数字数据类型之间转换
#### 问题
#### 解决方案



## 工作原理

### 8-3. 选择数据类型的精度和标度
#### 问题
#### 解决方案
#### 工作原理

##### 8-4. 使用非数字和无限数字正确执行计算
#### 问题
#### 解决方案
#### 工作原理

##### 8-5. 验证字符串中的数字
#### 问题
#### 解决方案
#### 工作原理

##### 8-6. 生成连续数字
#### 问题
#### 解决方案
#### 工作原理

##### 8-7. 按公式或模式生成数字
#### 问题
#### 解决方案
#### 工作原理

##### 8-8. 处理数值计算中的空值
#### 问题
#### 解决方案
#### 工作原理

### 8-9. 自动四舍五入数字
#### 问题
#### 解决方案
#### 工作原理

##### 8-10. 自动生成数字列表
#### 问题
#### 解决方案
#### 工作原理

## 第三部分：您的开发环境
### 管理事务
##### 9-1. 部分回滚事务
##### 问题
##### 解决方案
##### 工作原理

##### 9-2. 识别阻塞事务
##### 问题
##### 解决方案
##### 工作原理

#### 9-3. 优化行和表锁定
##### 问题
##### 解决方案
##### 工作原理

##### 9-4. 避免死锁场景
##### 问题
##### 解决方案
##### 工作原理

##### 9-5. 延迟约束验证
##### 问题
##### 解决方案
##### 工作原理

#### 9-6. 确保跨事务的读取一致性
##### 问题
##### 解决方案
##### 工作原理

##### 9-7. 管理事务隔离级别
##### 问题
##### 解决方案
##### 工作原理

### 数据字典
#### 图形化工具与 SQL
#### 数据字典架构
##### 静态视图
##### 动态性能视图

#### 10-1. 显示用户信息
##### 问题
##### 解决方案
##### 工作原理

#### 10-2. 确定您可以访问的表
##### 问题
##### 解决方案
##### 工作原理

#### 10-3. 显示表的磁盘空间使用情况
##### 问题
##### 解决方案
##### 工作原理

#### 10-4. 显示表行数
##### 问题
##### 解决方案
##### 工作原理

#### 10-5. 显示表的索引
##### 问题
##### 解决方案
##### 工作原理

#### 10-6. 显示未被索引的外键列
##### 问题
##### 解决方案
##### 工作原理

#### 10-7. 显示约束
##### 问题
##### 解决方案
##### 工作原理

#### 10-8. 显示主键和外键关系
##### 问题
##### 解决方案
##### 工作原理

#### 10-9. 显示对象依赖关系
##### 问题
##### 解决方案
##### 工作原理

#### 10-10. 显示同义词元数据
##### 问题
##### 解决方案
##### 工作原理

#### 10-11. 显示视图文本
##### 问题
##### 解决方案
##### 工作原理

#### 10-12. 显示数据库代码
##### 问题
##### 解决方案
##### 工作原理

#### 10-13. 显示已授予的角色
##### 问题
##### 解决方案
##### 工作原理

#### 10-14. 显示对象权限
##### 问题
##### 解决方案
##### 工作原理

#### 10-15. 显示系统权限
##### 问题
##### 解决方案
##### 工作原理



## 第四部分：专题
### 常见报表问题
#### 11-1. 避免报表中出现重复行
##### 问题
##### 解决方案
##### 工作原理
#### 11-2. 参数化 SQL 报表
##### 问题
##### 解决方案
##### 工作原理
##### 11-3. 在分组结果中返回详细列
##### 问题
##### 解决方案
##### 工作原理
##### 11-4. 将结果排序到等大小的分组中
##### 问题
##### 解决方案
##### 工作原理
#### 11-5. 创建报表直方图
##### 问题
##### 解决方案
##### 工作原理
#### 11-6. 按相对排名筛选结果
##### 问题
##### 解决方案
##### 工作原理
##### 11-7. 在数据集上比较假设
##### 问题
##### 解决方案
##### 工作原理
##### 11-8. 用文本图形化表示数据分布
##### 问题
##### 解决方案
##### 工作原理
#### 11-9. 直接从数据库生成网页报表
##### 问题
##### 解决方案
##### 工作原理
### 数据清洗
##### 12-1. 检测重复行
##### 问题
##### 解决方案
##### 工作原理
##### 12-2. 删除重复行
##### 问题
##### 解决方案
##### 工作原理
#### 12-3. 确定数据是否可以加载为数值类型
##### 问题
##### 解决方案
##### 工作原理
#### 12-4. 确定数据是否可以加载为日期类型
##### 问题
##### 解决方案
##### 工作原理
#### 12-5. 执行大小写不敏感的查询
##### 问题
##### 解决方案
##### 工作原理
##### 12-6. 混淆值
##### 问题
##### 解决方案
##### 工作原理
##### 12-7. 删除所有索引
##### 问题
##### 解决方案
##### 工作原理
##### 12-8. 禁用约束
##### 问题
##### 解决方案
##### 工作原理
##### 12-9. 禁用触发器
##### 问题
##### 解决方案
##### 工作原理
##### 12-10. 从表中删除数据
##### 问题
##### 解决方案
##### 工作原理
#### 12-11. 显示架构差异
##### 问题
##### 解决方案
##### 工作原理
### 树形结构数据
##### 13-1. 从上到下遍历层次数据
##### 问题
##### 解决方案
##### 工作原理
##### 13-2. 在层次级别内对节点排序
##### 问题
##### 解决方案
##### 13-3. 从层次表生成路径名
##### 问题
##### 解决方案
##### 工作原理
#### 13-4. 在层次表中识别叶子数据
##### 问题
##### 解决方案
##### 工作原理
##### 13-5. 检测层次数据中的循环
##### 问题
##### 解决方案
##### 工作原理
#### 13-6. 生成固定数量的序列主键
##### 问题
##### 解决方案
##### 工作原理
### 处理 XML 数据
##### 14-1. 将 SQL 转换为 XML
##### 问题
##### 解决方案
##### 工作原理
##### 14-2. 以原生形式存储 XML
##### 问题
##### 解决方案
##### 工作原理
#### 14-3. 为关系型使用解析 XML
##### 问题
##### 解决方案
##### 工作原理
##### 14-4. 从 XML 文档中提取关键 XML 元素
##### 问题
##### 解决方案
##### 工作原理
##### 14-5. 生成复杂的 XML 文档
##### 问题
##### 解决方案



# 分区

## 15-1. 确定表是否应进行分区
### 问题
### 解决方案
### 工作原理

##### 15-2. 按范围分区
### 问题
### 解决方案
### 工作原理

##### 15-3. 按列表分区
### 问题
### 解决方案
### 工作原理

##### 15-4. 按哈希分区
### 问题
### 解决方案
### 工作原理

##### 15-5. 以多种方式对表进行分区
### 问题
### 解决方案
### 工作原理

##### 15-6. 按需创建分区
### 问题
### 解决方案
### 工作原理

##### 15-7. 按引用约束分区
### 问题
### 解决方案
### 工作原理

##### 15-8. 在虚拟列上分区
### 问题
### 解决方案
### 工作原理

## 15-9. 应用程序控制分区
### 问题
### 解决方案
### 工作原理

##### 15-10. 使用表空间配置分区
### 问题
### 解决方案
### 工作原理

## 15-11. 自动移动更新后的行
### 问题
### 解决方案
### 工作原理

##### 15-12. 对现有表进行分区
### 问题
### 解决方案
### 工作原理

## 15-13. 向已分区表添加分区
### 问题
### 解决方案
### 工作原理

##### 15-14. 将分区与现有表交换
### 问题
### 解决方案
### 工作原理

##### 15-15. 重命名分区
### 问题
### 解决方案
### 工作原理

##### 15-16. 拆分分区
### 问题
### 解决方案
### 工作原理

##### 15-17. 合并分区
### 问题
### 解决方案
### 工作原理

##### 15-18. 删除分区
### 问题
### 解决方案
### 工作原理

## 15-19. 从分区中删除行
### 问题
### 解决方案
### 工作原理

##### 15-20. 为分区生成统计信息
### 问题
### 解决方案
### 工作原理

##### 15-21. 创建映射到分区的索引（本地索引）
### 问题
### 解决方案
### 工作原理

##### 15-22. 创建具有自身分区方案的索引（全局索引）
### 问题
### 解决方案
### 工作原理

# 大型对象

## 16-1. 将大文档加载到 CLOB 列中
### 问题
### 解决方案
### 工作原理

## 16-2. 将图像数据加载到 BLOB 列中
### 问题
### 解决方案
### 工作原理

##### 16-3. 使用 SQL*Loader 批量加载大对象
### 问题
### 解决方案
### 工作原理

## 16-4. 使用 HTTP 访问大型对象
### 问题
### 解决方案
### 工作原理

## 16-5. 使外部大型对象（BFILE）可供数据库使用
### 问题
### 解决方案
### 工作原理

## 16-6. 删除或更新数据库表中的大型对象
### 问题
### 解决方案
### 工作原理

# 第五部分：管理

## 数据库管理

##### 17-1. 创建数据库



# 数据库管理

## 17-1. 数据库创建
*   问题
*   解决方案
*   工作原理

## 17-2. 数据库删除
*   问题
*   解决方案
*   工作原理

##### 17-3. 验证连接信息
*   问题
*   解决方案
*   工作原理

##### 17-4. 创建表空间
*   问题
*   解决方案
*   工作原理

##### 17-5. 删除表空间
*   问题
*   解决方案
*   工作原理

##### 17-6. 调整表空间大小
*   问题
*   解决方案
*   工作原理

##### 17-7. 限制每个会话的数据库资源
*   问题
*   解决方案
*   工作原理

## 17-8. 关联权限组
*   问题
*   解决方案
*   工作原理

##### 17-9. 创建用户
*   问题
*   解决方案
*   工作原理

##### 17-10. 删除用户
*   问题
*   解决方案
*   工作原理

##### 17-11. 修改密码
*   问题
*   解决方案
*   工作原理

##### 17-12. 强制密码复杂性
*   问题
*   解决方案
*   工作原理

# 对象管理

##### 18-1. 创建表
*   问题
*   解决方案
*   工作原理

##### 18-2. 临时存储数据
*   问题
*   解决方案
*   工作原理

##### 18-3. 移动表
*   问题
*   解决方案
*   工作原理

##### 18-4. 重命名对象
*   问题
*   解决方案
*   工作原理

##### 18-5. 删除表
*   问题
*   解决方案
*   工作原理

## 18-6. 恢复删除的表
*   问题
*   解决方案
*   工作原理

##### 18-7. 创建索引
*   问题
*   解决方案
*   工作原理

##### 18-8. 创建基于函数的索引
*   问题
*   解决方案
*   工作原理

##### 18-9. 创建位图索引
*   问题
*   解决方案
*   工作原理

##### 18-10. 创建索引组织表
*   问题
*   解决方案
*   工作原理

##### 18-11. 创建视图
*   问题
*   解决方案
*   工作原理

##### 18-12. 为对象创建别名
*   问题
*   解决方案
*   工作原理

## 18-13. 强制表中行的唯一性
*   问题
*   解决方案
*   工作原理

##### 18-14. 确保查找值存在
*   问题
*   解决方案
*   工作原理

##### 18-15. 按条件检查数据
*   问题
*   解决方案
*   工作原理

##### 18-16. 创建数据库之间的连接
*   问题
*   解决方案
*   工作原理

## 18-17. 创建自动递增值
*   问题
*   解决方案
*   工作原理

# SQL 监控与调优

##### 19-1. 监控实时 SQL 执行统计信息
*   问题
*   解决方案
*   工作原理

##### 19-2. 在执行计划中显示查询进度
*   问题
*   解决方案
*   工作原理

##### 19-3. 确定剩余的 SQL 工作量
*   问题
*   解决方案
*   工作原理

##### 19-4. 识别资源密集型 SQL 语句
*   问题
*   解决方案
*   工作原理


# 19-5. 利用 Oracle 性能报告识别资源密集型 SQL

*   问题
*   解决方案

## 使用 AWR

## 使用 ADDM

## 使用 ASH

## 使用 Statspack

*   工作原理

##### 19-6. 使用操作系统识别资源密集型查询

*   问题
*   解决方案
*   工作原理

##### 19-7. 使用 AUTOTRACE 显示执行计划

*   问题
*   解决方案

## 设置 AUTOTRACE

## 生成执行计划

*   工作原理

# 19-8. 使用 DBMS\_XPLAN 生成执行计划

*   问题
*   解决方案
*   工作原理

# 19-9. 跟踪会话的所有 SQL 语句

*   问题
*   解决方案
*   工作原理

## 使用 DBMS\_SESSION

## 使用 DBMS\_MONITOR

## 使用 DBMS\_SYSTEM

## 使用 DBMS\_SUPPORT

## 更改您的会话

## 更改系统

## 使用 oradebug

##### 19-10. 解释执行计划

*   问题
*   解决方案
*   工作原理

##### 19-11. 获取 SQL 调优建议

*   问题
*   解决方案
*   工作原理

# 19-12. 强制查询使用自定义执行计划

*   问题
*   解决方案
*   工作原理

##### 19-13. 查看优化器统计信息

*   问题
*   解决方案
*   工作原理

##### 19-14. 生成统计信息

*   问题
*   解决方案
*   工作原理

# 20. 数据库故障排除

##### 20-1. 确定数据库问题的原因

*   问题
*   解决方案
*   工作原理

##### 20-2. 显示打开的游标

*   问题
*   解决方案
*   工作原理

# 20-3. 确定联机重做日志大小是否合适

*   问题
*   解决方案
*   工作原理

# 20-4. 确定 UNDO 表空间大小是否合适

*   问题
*   解决方案
*   工作原理

##### 20-5. 确定临时表空间大小是否正确

*   问题
*   解决方案
*   工作原理

##### 20-6. 显示表空间使用率

*   问题
*   解决方案
*   工作原理

##### 20-7. 显示对象大小

*   问题
*   解决方案
*   工作原理

##### 20-8. 监控索引使用情况

*   问题
*   解决方案
*   工作原理

##### 20-9. 审计对象使用情况

*   问题
*   解决方案
*   工作原理

##### 20-10. 细粒度审计

*   问题
*   解决方案
*   工作原理

## 索引