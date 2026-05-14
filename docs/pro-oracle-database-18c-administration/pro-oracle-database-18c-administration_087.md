# Oracle SQL 监控与性能诊断

## V$SQL_MONITOR 视图

`V$SQL_MONITOR` 中的统计数据每秒更新一次，因此您可以查看资源消耗的变化情况。默认情况下，如果 SQL 语句并行运行或消耗超过 5 秒的 CPU 或 I/O 时间，就会收集这些统计数据。

`V$SQL_MONITOR` 视图包含 `V$SQL`、`V$SQLAREA` 和 `V$SQLSTATS` 视图中包含的统计数据的一个子集。`V$SQL_MONITOR` 视图显示每个资源密集型 SQL 语句执行的实时统计数据，而 `V$SQL`、`V$SQLAREA` 和 `V$SQLSTATS` 则包含多次执行 SQL 语句所产生的累积统计数据集。

SQL 语句执行结束后，运行时统计数据不会立即从 `V$SQL_MONITOR` 中刷新。根据数据库中的活动情况，统计数据可能会保留一段时间。然而，如果您的数据库非常活跃，统计数据可能会在查询完成后不久被刷新。

**提示**
您可以通过以下列的组合在 `V$SQL_MONITOR` 中唯一标识 SQL 语句的执行：`SQL_ID`、`SQL_EXEC_START`、`SQL_EXEC_ID`。

## 识别高资源消耗 SQL

您也可以查询 `V$SQLSTATS` 等视图来确定哪些 SQL 语句消耗了过量的资源。例如，使用以下查询根据 CPU 时间识别消耗资源最多的十个查询：

```sql
SQL> select * from(
select s.sid, s.username, s.sql_id
,sa.elapsed_time/1000000, sa.cpu_time/1000000
,sa.buffer_gets, sa.sql_text
from v$sqlarea sa
,v$session s
where s.sql_hash_value = sa.hash_value
and   s.sql_address    = sa.address
and   s.username is not null
order by sa.cpu_time desc)
where rownum <= 10;
```

在上面的查询中，使用了一个内联视图首先检索所有记录，并按 `CPU_TIME` 降序对输出进行排序。然后外部查询使用 `ROWNUM` 伪列将结果集限制为前 10 行。该查询可以很容易地修改为按 `CPU_TIME` 以外的列排序。例如，如果您想按 `BUFFER_GETS` 报告资源使用情况，只需在 `ORDER BY` 子句中用 `BUFFER_GETS` 替换 `CPU_TIME` 即可。`CPU_TIME` 列以微秒计算；为了将其转换为秒，它被除以 1,000,000。

**提示**
请记住，`V$SQLAREA` 包含的是给定会话持续期间的累积统计数据。因此，如果一个会话多次运行相同的查询，该连接的统计数据将是该查询所有运行的总和。相比之下，`V$SQL_MONITOR` 显示的是给定 SQL 语句当前运行累积的统计数据。因此，每次查询运行时，`V$SQL_MONITOR` 中都会报告该查询的新统计数据。

## Oracle 诊断工具

Oracle 提供了几种用于诊断数据库性能问题的实用程序：

*   自动工作量存储库（AWR）
*   自动数据库诊断监视器（ADDM）
*   活动会话历史（ASH）
*   Statspack

AWR、ADDM 和 ASH 是多年前在 Oracle Database 10g 中引入的。这些工具提供了高级报告功能，使您可以排查和解决性能问题。这些新实用程序需要从 Oracle 获得额外许可。较旧的 Statspack 实用程序是免费的，不需要许可。

所有这些工具都严重依赖底层的 `V$` 动态性能视图。Oracle 维护着大量的这些视图，用于跟踪和累积数据库性能指标。例如，如果您运行以下查询，您会注意到在 Oracle Database 18c 中，大约有 760 个 `V$` 视图：

```sql
SQL> select count(*) from v$fixed_table where name like 'V$%';
COUNT(*)
```

`V$FIXED_TABLE` 提供有关动态性能视图的信息，包括 RAC 环境的底层 X$ 表和 GV$ 视图。

Oracle 性能实用程序依赖于从这些内部性能视图收集的定期快照。关于性能统计，最有用的两个视图是 `V$SYSSTAT` 和 `V$SESSTAT` 视图。`V$SYSSTAT` 视图提供超过 800 种类型的数据库统计信息。这个 `V$SYSSTAT` 视图包含整个数据库的信息，而 `V$SESSTAT` 视图包含关于各个会话的统计信息。`V$SYSSTAT` 和 `V$SESSTAT` 视图中的一些值代表资源的当前使用情况。这些值是：

*   当前打开的游标数
*   当前登录数
*   当前会话游标缓存数
*   已分配的工作区内存

其余的值是累积的。`V$SYSSTAT` 中的值是从实例启动开始对整个数据库累积的。`V$SESSTAT` 中的值是从会话开始起对每个会话累积的。一些更重要的与性能相关的累积值如下：

*   使用的 CPU
*   一致读
*   物理读
*   物理写

对于累积统计数据，衡量周期性使用的方法是在起始点记录一个统计值，然后在稍后的时间点再次记录该值并捕获差值。这就是 Oracle 性能实用程序（如 AWR 和 Statspack）所采用的方法。Oracle 会定期获取动态等待接口视图的快照并将其存储在存储库中。

以下部分详细介绍了如何通过 SQL 命令行访问 AWR、ADDM、ASH 和 Statspack。

**提示**
您可以从 Enterprise Manager 访问 AWR、ADDM 和 ASH。如果您有权访问 Enterprise Manager，您会发现该界面相当直观且视觉上很有帮助。

### 使用 AWR

AWR 报告适用于查看整个系统的性能并识别消耗资源最多的 SQL 查询。运行以下脚本以生成 AWR 报告：

```sql
SQL> @?/rdbms/admin/awrrpt
```

从 AWR 输出中，您可以通过检查报告的“SQL Ordered by Elapsed Time”或“SQL Ordered by CPU Time”部分来识别消耗资源最多的语句。以下是一些示例输出：

```sql
SQL ordered by CPU Time                  DB/Inst: O18C/o18c  Snaps: 1668-1669
-> Resources reported for PL/SQL code includes the resources used by all SQL
statements called by the code.
-> %Total - CPU Time      as a percentage of Total DB CPU
-> %CPU   - CPU Time      as a percentage of Elapsed Time
-> %IO    - User I/O Time as a percentage of Elapsed Time
-> Captured SQL account for   3.0E+03% of Total CPU Time (s):              0
-> Captured PL/SQL account for  550.9% of Total CPU Time (s):              0
CPU                   CPU per           Elapsed
Time (s)  Executions    Exec (s) %Total   Time (s)   %CPU   %IO         SQL Id
---------- ------------ ---------- ------ ---------- ------ ----- --------------
3.2           1        3.24  930.5       10.5   30.9  74.9  93jktd5vtxb98
```

Oracle 将自动每小时对数据库进行一次快照，并填充存储统计数据的底层 AWR 表。默认情况下，统计数据会保留 7 天。

您也可以通过运行 `awrsqrpt.sql` 报告为特定 SQL 语句生成 AWR 报告。当您运行以下脚本时，系统将提示您输入感兴趣的查询的 `SQL_ID`：

```sql
SQL> @?/rdbms/admin/awrsqrpt.sql
```

### 使用 ADDM

ADDM 报告提供了有关哪些 SQL 语句是调优候选者的有用信息。使用以下 SQL 脚本生成 ADDM 报告：

```sql
SQL> @?/rdbms/admin/addmrpt
```

查找报告中标识为“SQL Statements Consuming Significant Database Time”的部分。以下是一些示例输出：

```
FINDING 2: 29% impact (65043 seconds)

SQL statements consuming significant database time were found.
RECOMMENDATION 1: SQL Tuning, 6.7% benefit (14843 seconds)
ACTION: Investigate the SQL statement with SQL_ID "46cc3t7ym5sx0" for
```



ADDM 报告通过分析 AWR 表中的数据，识别潜在瓶颈和高资源消耗的 SQL 查询。

## 使用 ASH

ASH 报告让您能够聚焦于近期执行过、可能仅短暂存在的 SQL 语句。运行以下脚本生成 ASH 报告：

```sql
SQL> @?/rdbms/admin/ashrpt
```

在输出结果中查找标记为"Top SQL"的部分。以下是一段示例输出：

```
Top SQL with Top Events           DB/Inst: O18C/o18c  (Jan 30 14:49 to 15:14)
Sampled #
SQL ID             Planhash        of Executions     % Activity
----------------------- -------------------- -------------------- --------------
Event                          % Event Top Row Source                    % RwSrc
------------------------------ ------- --------------------------------- -------
dx5auh1xb98k5           1677482778                    1          46.57
CPU + Wait for CPU               46.57 HASH JOIN                           23.53
```

以上输出表明该查询正在等待 CPU 资源。在这种情况下，问题可能是另一个查询占用了 CPU 资源。
ASH 报告何时比 AWR 或 ADDM 报告更有用？AWR 和 ADDM 输出展示的是按总数据库时间排序的高消耗 SQL。如果 SQL 性能问题是短暂且稍纵即逝的，它可能不会出现在 AWR 和 ADDM 报告中。在这些情况下，ASH 报告更为有用。

## 使用 Statspack

如果您没有使用 AWR、ASH 和 ADDM 报告的许可，免费的 Statspack 工具可以帮助您识别性能不佳的 SQL 语句。以`SYS`身份运行以下脚本安装 Statspack：

```sql
SQL> @?/rdbms/admin/spcreate.sql
```

此脚本会创建拥有 Statspack 存储库的`PERFSTAT`用户。创建完成后，以`PERFSTAT`用户身份连接，并运行此脚本来启用 Statspack 统计信息的自动收集：

```sql
SQL> @?/rdbms/admin/spauto.sql
```

收集了一些快照后，您可以以`PERFSTAT`用户身份运行以下脚本创建 Statspack 报告：

```sql
SQL> @?/rdbms/admin/spreport.sql
```

报告创建后，查找标记为"SQL Ordered by CPU"的部分。以下是一段示例输出：

```
SQL ordered by CPU  DB/Inst: O18C/o18c  Snaps: 30-31
-> Total DB CPU (s):           1,432
-> Captured SQL accounts for  100.5% of Total DB CPU
-> SQL reported below exceeded  1.0% of Total DB CPU
CPU                  CPU per          Elapsd                        Old
Time (s)   Executions  Exec (s)  %Total   Time (s)    Buffer Gets  Hash Value
---------- ------------ ---------- ------ ---------- --------------- ----------
1430.41            1   1430.41    99.9    1432.49            482   690392559
Module: SQL*Plus
select a.table_name from my_tables
```

> **提示：** Statspack 文档请参见 `ORACLE_HOME/rdbms/admin/spdoc.txt` 文件。

## 检测与解决锁问题

有时，开发人员或应用用户会报告一个通常只需几秒运行的进程，现在需要几分钟，并且似乎没有任何进展。在这些情况下，问题通常是以下之一：

*   空间相关问题（例如，归档重做日志目标已满，导致所有事务挂起）。
*   一个进程锁定了表中的一行，但既未提交也未回滚，从而阻止其他会话修改同一行。

在此场景中，首先检查 `alert.log` 看看最近是否有明显的问题发生（例如表空间无法再分配区间）。如果 `alert.log` 文件中没有明显问题，则运行 SQL 查询以查找锁问题。此处列出的查询是第 3 章介绍的锁检测脚本的更复杂版本。该查询显示诸如锁定会话的 SQL 语句和等待会话的 SQL 语句等信息：

```sql
SQL> set lines 80
SQL> col blkg_user form a10
SQL> col blkg_machine form a10
SQL> col blkg_sid form 99999999
SQL> col wait_user form a10
SQL> col wait_machine form a10
SQL> col wait_sid form 9999999
SQL> col obj_own form a10
SQL> col obj_name form a10
SQL> col blkg_sql form a50
SQL> col wait_sql form a50
--
SQL> select
s1.username    blkg_user, s1.machine     blkg_machine
,s1.sid         blkg_sid, s1.serial#     blkg_serialnum
,s1.process     blkg_OS_PID
,substr(b1.sql_text,1,50) blkg_sql
,chr(10)
,s2.username    wait_user, s2.machine     wait_machine
,s2.sid         wait_sid, s2.serial#     wait_serialnum
,s2.process     wait_OS_PID
,substr(w1.sql_text,1,50) wait_sql
,lo.object_id   blkd_obj_id
,do.owner       obj_own, do.object_name obj_name
from v$lock          l1
,v$session       s1
,v$lock          l2
,v$session       s2
,v$locked_object lo
,v$sqlarea       b1
,v$sqlarea       w1
,dba_objects     do
where s1.sid = l1.sid
and s2.sid = l2.sid
and l1.id1 = l2.id1
and s1.sid = lo.session_id
and lo.object_id = do.object_id
and l1.block = 1
and s1.prev_sql_addr = b1.address
and s2.sql_address = w1.address
and l2.request > 0;
```

此查询的输出不太适合放在一页上。运行此查询时，您需要对其进行格式化以便在屏幕上显示。以下是一段示例输出，表明`SALES`表被锁定，并且另一个进程正在等待锁被释放：

```
BLKG_USER  BLKG_MACHI  BLKG_SID BLKG_SERIALNUM BLKG_OS_PID
---------- ---------- --------- -------------- ------------------------
BLKG_SQL                                           C WAIT_USER  WAIT_MACHI
-------------------------------------------------- - ---------- ----------
WAIT_SID WAIT_SERIALNUM WAIT_OS_PID
-------- -------------- ------------------------
WAIT_SQL                                           BLKD_OBJ_ID OBJ_OWN
-------------------------------------------------- ----------- ----------
OBJ_NAME

MV_MAINT   speed2            32            487 26216
update sales set sales_amt=100 where sales_id=1      MV_MAINT   speed2
116            319 25851
```

这种情况在应用程序未在代码中适当地显式发出`COMMIT`或`ROLLBACK`时很常见。这会导致锁留在行上，并在锁被释放前阻止事务继续进行。在此场景中，您可以尝试找出阻塞事务的用户，并查看该用户是否需要点击一个类似“提交您的更改”的按钮。如果无法做到这一点，您可以手动终止其中一个会话。请注意，终止会话可能会产生不可预见的影响（例如回滚用户认为已提交的数据）。

如果您决定终止某个用户会话，需要确定要终止的会话的`SID`和序列号。确定后，使用`ALTER SYSTEM KILL SESSION`语句终止会话：

```sql
SQL> alter system kill session '32,487';
```

再次提醒，终止会话时要谨慎。确保您了解终止会话并因此回滚该会话中当前所有活动事务的影响。

另一种终止会话的方法是使用操作系统命令，例如`KILL`。在之前的输出中，您可以从`BLKG_OS_PID`和`WAIT_OS_PID`列中确定操作系统进程 ID。在操作系统层面终止进程之前，请确保该进程并非关键进程。对于此示例，要终止阻塞的操作系统进程，首先检查阻塞进程的 PID：

```bash
$ ps -ef | grep 26216
```

以下是一段示例输出：

```
oracle   26222 26216  0 16:49 ?        00:00:00 oracleo12c
```

然后，使用`KILL`命令，如下所示：

```bash
$ kill -9  26216
```

`KILL`命令将会立即终止进程。与该进程关联的所有未提交事务将由 Oracle 进程监视器回滚。

## 解决开放游标问题



## OPEN_CURSORS 参数

`OPEN_CURSORS` 初始化参数决定了一个会话可以同时打开的游标数量上限。此设置是按会话生效的。默认值 50 对于任何应用程序来说通常都太低了。当应用程序超过允许的打开游标数时，会抛出以下错误：

```
ORA-01000: maximum open cursors exceeded
```

通常，在以下情况下会遇到上述错误：

*   `OPEN_CURSORS` 初始化参数设置得过低
*   开发人员编写的代码未能正确关闭游标

要调查此问题，首先需要确定该参数的当前设置：

```
SQL> show parameter open_cursors;
```

如果该值小于 300，请考虑将其设置得更高一些。对于繁忙的 OLTP 系统，通常将此值设置为 1,000。您可以在数据库打开状态下动态修改该值，如下所示：

```
SQL> alter system set open_cursors=1000;
```

如果您使用的是 `spfile`，请考虑同时在内存和 `spfile` 中进行更改：

```
SQL> alter system set open_cursors=1000 scope=both;
```

在将 `OPEN_CURSORS` 设置为更高的值后，如果应用程序仍然超过最大值，那么您的代码中很可能存在未能正确关闭游标的问题。

### 调查游标使用情况

如果您工作的环境有数千个到数据库的连接，您可能只想查看消耗游标最多的前几个会话。以下查询使用一个内联视图和伪列 `ROWNUM` 来显示前 20 个值：

```
SQL> select * from (
select a.value, c.username, c.machine, c.sid, c.serial#
from v$sesstat  a
,v$statname b
,v$session  c
where a.statistic# = b.statistic#
and   c.sid        = a.sid
and   b.name       = 'opened cursors current'
and   a.value     != 0
and   c.username IS NOT NULL
order by 1 desc,2)
where rownum < 21;
```

如果单个会话的打开游标数超过 1,000，那么很可能是代码编写方式导致游标没有被关闭。当达到限制时，应该有人检查应用程序代码，以确定是否有游标未被关闭。

**提示**
建议您查询 `V$SESSION` 而不是 `V$OPEN_CURSOR` 来确定打开的游标数量。`V$SESSION` 提供的当前打开游标计数更为准确。

## 疑难解答撤销表空间问题

撤销表空间的问题通常属于以下性质：

*   `ORA-01555: snapshot too old`
*   `ORA-30036: unable to extend segment by ... in undo tablespace 'UNDOTBS1'`

上述错误可能由多种不同问题引起，例如撤销表空间大小设置不正确，或者 SQL 或 PL/SQL 代码编写不当。快照过旧也可能在导出操作期间由于更新非常大的表而出现，并且通常在撤销保留时间或大小设置不当时遇到。对于导出操作，在较安静的时间重新运行是一个简单的解决方法。

### 确定撤销表空间大小是否正确

假设您有一个运行时间很长的 SQL 语句，它正在抛出 `ORA-01555: snapshot too old` 错误，并且您想确定增加撤销表空间的空间是否有助于缓解该问题。运行以下查询以识别撤销表空间的潜在问题。该查询检查过去一天内发生的问题：

```
SQL> select to_char(begin_time,'MM-DD-YYYY HH24:MI') begin_time
,ssolderrcnt    ORA_01555_cnt, nospaceerrcnt  no_space_cnt
,txncount       max_num_txns, maxquerylen    max_query_len
,expiredblks    blck_in_expired
from v$undostat
where begin_time > sysdate - 1
order by begin_time;
```

以下是一些示例输出。为适应页面，输出的一部分已被省略：

```
BEGIN_TIME       ORA_01555_CNT NO_SPACE_CNT MAX_NUM_TXNS BLCK_IN_EXPIRED
---------------- ------------- ------------ ------------ ---------------
01-31-2013 08:21             0            0           51               0
01-31-2013 08:31             0            0            0               0
01-31-2013 12:11             0            0          629             256
```

`ORA_01555_CNT` 列指示您的数据库遇到 `ORA-01555: snapshot too old` 错误的次数。如果此列报告非零值，您需要执行以下一项或多项任务：

*   确保代码中不包含游标循环内的 `COMMIT` 语句。
*   调优抛出错误的 SQL 语句，使其运行得更快。
*   确保您有良好的统计信息（以便您的 SQL 能高效运行）。
*   增加 `UNDO_RETENTION` 初始化参数值。

`NO_SPACE_CNT` 列显示在撤销表空间中请求空间的次数。在此示例中，没有此类请求。然而，如果 `NO_SPACE_CNT` 报告非零值，则您可能需要向撤销表空间添加更多空间。

`V$UNDOSTAT` 视图中最多存储 4 天的信息。统计信息每 10 分钟收集一次，表中最多有 576 行。如果您在过去 4 天内停止并重新启动了数据库，则此视图仅包含您上次启动数据库以来的信息。

### 使用 Oracle 撤销顾问

获取撤销表空间大小设置建议的另一种方法是使用 Oracle 撤销顾问，您可以通过从 `SELECT` 语句中调用 PL/SQL `DBMS_UNDO_ADV` 包来使用它。以下查询显示当前撤销大小以及撤销保留设置为 900 秒时的建议大小：

```
SQL> select
sum(bytes)/1024/1024                  cur_mb_size
,dbms_undo_adv.required_undo_size(900) req_mb_size
from dba_data_files
where tablespace_name =
(select
value
from v$parameter
where name = 'undo tablespace');
```

以下是一些示例输出：

```
CUR_MB_SIZE REQ_MB_SIZE
----------- -----------
36864       20897
```

输出显示撤销表空间当前分配了 36.8GB。在前面的查询中，您使用了 900 秒作为在撤销表空间中保留信息的时间。为了保留撤销信息 900 秒，Oracle 撤销顾问估计撤销表空间应为 20.8GB。在此示例中，撤销表空间大小是足够的。如果大小不足，您将必须向现有数据文件添加空间，或者向撤销表空间添加一个数据文件。

以下是一个稍复杂的示例，使用 Oracle 撤销顾问查找撤销表空间所需大小。此示例使用 PL/SQL 显示有关潜在问题以及修复建议的信息：

```
SQL> SET SERVEROUT ON SIZE 1000000
SQL> DECLARE
pro    VARCHAR2(200);
rec    VARCHAR2(200);
rtn    VARCHAR2(200);
ret    NUMBER;
utb    NUMBER;
retval NUMBER;
BEGIN
DBMS_OUTPUT.PUT_LINE(DBMS_UNDO_ADV.UNDO_ADVISOR(1));
DBMS_OUTPUT.PUT_LINE('Required Undo Size (megabytes): ' || DBMS_UNDO_ADV.REQUIRED_UNDO_SIZE (900));
retval := DBMS_UNDO_ADV.UNDO_HEALTH(pro, rec, rtn, ret, utb);
DBMS_OUTPUT.PUT_LINE('Problem:   ' || pro);
DBMS_OUTPUT.PUT_LINE('Advice:    ' || rec);
DBMS_OUTPUT.PUT_LINE('Rational:  ' || rtn);
DBMS_OUTPUT.PUT_LINE('Retention: ' || TO_CHAR(ret));
DBMS_OUTPUT.PUT_LINE('UTBSize:   ' || TO_CHAR(utb));
END;
/
```

如果没有发现问题，则保留大小将返回 0。以下是一些示例输出：

```
Finding 1:The undo tablespace is OK.
Required Undo Size (megabytes): 64
Problem:   No problem found
Advice:
Rational:
Retention: 0
UTBSize:   0
```



