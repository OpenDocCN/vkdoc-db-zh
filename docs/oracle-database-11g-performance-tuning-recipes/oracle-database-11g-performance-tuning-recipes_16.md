# 第 7 章

`SQL_ID  by3c8848gyngu, child number 0`
`-------------------------------------`
`SELECT "A1"."REGISTRATION_ID","A1"."PRODUCT_INSTANCE_ID","A1"."SOA_ID","`
`A1"."REG_SOURCE_IP_ADDR","A1"."REGISTRATION_STATUS","A1"."CREATE_DTT","A`
`1"."DOMAIN_ID","A1"."COUNT_FLG","A2"."PRODUCT_INSTANCE_ID","A2"."SVC_TAG`
`Plan hash value: 4286489280`
`-----------------------------------------------------------------------------------------`
`| Id|Operation                   |Name          | Rows  | Bytes | Cost (%CPU)| Time     |`
`-----------------------------------------------------------------------------------------`
`|  0|SELECT STATEMENT            |              |       |       | 64977 (100)|          |`
`|  1| NESTED LOOPS               |              |       |       |            |          |`
`|  2|  NESTED LOOPS              |              |     1 |   499 | 64977   (5)| 00:13:00 |`
`|  3|   NESTED LOOPS OUTER       |              |     1 |   462 | 64975   (5)| 00:13:00 |`
`|  4|    NESTED LOOPS OUTER      |              |     1 |   454 | 64973   (5)| 00:13:00 |`
`|  5|     NESTED LOOPS           |              |     1 |   420 | 64972   (5)| 00:13:00 |`
`|  6|      NESTED LOOPS OUTER    |              |     1 |   351 | 64971   (5)| 00:13:00 |`
`|  7|       NESTED LOOPS         |              |     1 |   278 | 64969   (5)| 00:13:00 |`
`|  8|        NESTED LOOPS OUTER  |              |     1 |   188 | 64967   (5)| 00:13:00 |`
`|  9|         NESTED LOOPS       |              |     1 |   180 | 64966   (5)| 00:13:00 |`
`|*10|          TABLE ACCESS FULL |REGISTRATIONS |     1 |    77 | 64964   (5)| 00:13:00 |`

此输出将帮助你确定 SQL 语句的效率，并提供有关如何调整它的见解。有关如何手动调整查询的详细信息，请参阅第 9 章；有关自动 SQL 调优的详细信息，请参阅第 11 章。

#### 工作原理

本配方“解决方案”部分描述的过程允许你快速识别资源密集型进程，然后将 OS 进程映射到数据库进程，并随后将数据库进程映射到 SQL 语句。一旦你知道了哪个 SQL 语句正在消耗资源，就可以生成执行计划，以进一步尝试确定任何可能的低效之处。

有时，资源消耗进程可能与数据库无关。在这些情况下，你必须与你的系统管理员（SA）合作，以确定该进程是什么以及是否可以调整或终止。

此外，你可能会遇到资源密集型进程，它们特定于数据库但与 SQL 语句无关。例如，你可能有一个长时间运行的 RMAN 备份进程、Data Pump 或 PL/SQL 作业正在运行。在这些情况下，请与你的 DBA 合作，以确定这些类型的进程是否可以调整或终止。

**ORADEBUG**

如果你知道操作系统 ID，可以使用 Oracle 的 `oradebug` 实用程序来显示消耗 SQL 语句最多的语句。例如，假设你已经使用 `top` 或 `ps` 等实用程序识别出一个高 CPU 消耗的操作系统进程，并且从进程名称你确定它是一个数据库进程。现在登录到 SQL*Plus 并使用 `oradebug` 来显示与该进程相关的任何 SQL。在此示例中，OS 进程 ID 是 7853：

```sql
SQL> oradebug setospid 7853;
Oracle pid: 18, Unix process pid: 7853, image: oracle@xengdb (TNS V1-V3)
```

现在显示与此进程关联的 SQL（如果有的话）：

```sql
SQL> oradebug current_sql;
```

如果有与该进程关联的 SQL 语句，它将被显示——例如：

```
select
 a.table_name
from dba_tables a, dba_indexes b, .....
```

`oradebug` 实用程序可以通过多种方法用于帮助排查性能问题。你可以通过发出以下命令来显示与会话关联的跟踪文件的名称：

```sql
SQL> oradebug tracefile_name;
```

使用 `oradebug help` 来显示所有可用选项。


## 排查数据库故障

Oracle Database 11g 提供了诊断数据库健康状况的新方法。本章包含多个操作指南，展示了如何使用数据库内置的诊断基础架构来解决数据库性能问题。您将学习如何使用 `ADRCI`（自动诊断库命令解释器）来执行各种任务，例如检查数据库警报日志、创建用于发送给 Oracle 支持工程师的诊断包，以及对数据库运行主动健康检查。

许多常见的 Oracle 数据库性能问题发生在临时表空间出现空间问题时，或者在您使用 `create table as select` (`CTAS`) 技术创建大型索引或大型表时。撤销表空间的空间问题是许多 DBA 面临的另一个常见麻烦来源。本章提供了多个操作指南，帮助您主动监控、诊断和解决与临时表空间及撤销表空间相关的问题。当生产数据库似乎挂起时，有办法收集用于分析原因的关键诊断数据，本章将向您展示如何登录到无响应的数据库以收集诊断数据。

### 7-1. 确定最佳的撤销保留期

#### 问题

您需要确定数据库中撤销保留的最佳时长。

#### 解决方案

您可以通过指定 `UNDO_RETENTION` 参数，来设定 Oracle 在事务提交后保留撤销数据的时间长度。以下是如何通过更新 `SPFILE` 中的 `UNDO_RETENTION` 参数值，为实例将撤销保留期设置为 30 分钟。

```
SQL> alter system set undo_retention=1800 scope=both;
系统已更改。
SQL>
```

要确定 `UNDO_RETENTION` 参数的最佳值，您必须首先计算数据库实际生成的撤销量。一旦您大致了解数据库生成的撤销量，就可以为 `UNDO_RETENTION` 参数计算一个更精确的值。使用以下公式计算 `UNDO_RETENTION` 参数的值：

`UNDO_RETENTION = UNDO SIZE/(DB_BLOCK_SIZE*UNDO_BLOCK_PER_SEC)`

您可以通过执行以下查询来计算数据库中生成的实际撤销量：

```
SQL> select sum(d.bytes) "undo"
  2  from v$datafile d,
  3  v$tablespace t,
  4  dba_tablespaces s
  5  where s.contents = 'UNDO'
  6  and s.status = 'ONLINE'
  7  and t.name = s.tablespace_name
  8  and d.ts# = t.ts#;

      UNDO
----------
  104857600
SQL>
```

您可以通过以下查询计算 `UNDO_BLOCKS_PER_SEC` 的值：

```
SQL> select max(undoblks/((end_time-begin_time)*3600*24))
  2  "UNDO_BLOCK_PER_SEC"
  3  FROM v$undostat;

UNDO_BLOCK_PER_SEC
------------------
            7.625
SQL>
```

您很可能记得数据库的块大小——如果不记得，可以在 `SPFILE` 中查找，或通过命令 `show parameter db_block_size` 来查找。假设您数据库的 `db_block_size` 是 8 KB（8,192 字节）。然后，您可以使用本配方前面展示的公式来计算 `UNDO_RETENTION` 参数的最佳值——例如，得出结果（以秒为单位）：1,678.69 = 104,857,600/(7.625 * 8,192)。在这种情况下，为 `undo_retention` 参数分配 1,800 秒的值是合适的，因为它比我们计算该参数值的公式所指示的值略大一些。

## 工作原理

自动撤销管理是自 11g 版本以来的默认撤销管理模式。如果您使用数据库配置助手（`DBCA`）创建数据库，Oracle 会自动创建一个名为`UNDOTBS1`的自动扩展撤销表空间。如果您是手动创建数据库，则需要在数据库创建语句中指定撤销表空间，或者也可以在之后添加撤销表空间。如果数据库没有显式的撤销表空间，Oracle 会将撤销记录存储在`SYSTEM`表空间中。

一旦设置了`UNDO_TABLESPACE`初始化参数，Oracle 会自动为您管理撤销保留。您可以选择设置`UNDO_RETENTION`参数，以指定 Oracle 在用新的撤销数据覆盖旧的撤销数据之前保留它的时间。

“解决方案”部分中的公式展示了如何基于当前数据库活动来确定撤销保留期。请注意，我们依赖动态视图`V$UNDOSTAT`来计算撤销保留期的值。因此，您必须在数据库运行一段时间后再执行查询，以确保它有机会处理典型的工作负载。

如果您配置了`UNDO_RETENTION`参数，那么撤销表空间必须足够大，以容纳在`UNDO_RETENTION`参数指定的时间内数据库生成的撤销数据。当事务提交时，数据库可能会用新的撤销数据覆盖其撤销数据。撤销保留期是数据库尝试保留旧的撤销数据的最短时间。Oracle 会为您在`UNDO_RETENTION`参数中指定的持续时间内保留撤销数据，既用于读一致性目的，也用于支持 Oracle 闪回操作。在为`UNDO_RETENTION`参数指定的时间段保存撤销数据后，数据库会将这些撤销数据标记为*过期*，并将其占用的空间提供给新事务写入撤销数据。

默认情况下，数据库使用以下标准来确定需要保留撤销数据的时间：
*   最长查询的运行时长
*   最长事务的运行时长
*   最长的闪回操作时长

理解 Oracle 数据库如何处理撤销数据的保留有些复杂。以下是其工作原理的简要总结：
*   如果您没有使用`AUTOEXTEND`选项配置撤销表空间，数据库将简单地忽略您为`UNDO_RETENTION`参数设置的值。数据库将根据数据库工作负载和撤销表空间的大小自动调整撤销保留期。因此，如果您收到指示数据库未保留足够长时间的撤销数据的错误，请确保将撤销表空间设置为较大的值。通常，在这种情况下，撤销保留时间会显著长于数据库中最长的活动查询的运行时长。
*   如果您希望数据库尽量遵守您为`UNDO_RETENTION`参数指定的设置，请确保为撤销表空间启用`AUTOEXTEND`选项。这样，Oracle 将自动扩展撤销表空间的大小，为新事务腾出写入撤销的空间，而不是覆盖旧的撤销数据。但是，如果您收到`ORA-0155`（快照过旧）错误，例如由于 Oracle 闪回操作，这意味着数据库无法有效地动态调整撤销保留期。在这种情况下，尝试增加`UNDO_RETENTION`参数的值以匹配最长的 Oracle 闪回操作的时长。或者，您可以尝试使用更大的固定大小（不带`AUTOEXTEND`选项）的撤销表空间。

确定撤销表空间正确大小或`UNDO_RETENTION`参数正确设置的关键在于理解当前数据库工作负载的性质。为了了解工作负载特征，检查`V$UNDOSTAT`视图至关重要，因为它包含显示数据库如何利用撤销空间的统计数据，以及诸如最长查询的运行时长等信息。您可以使用这些信息来计算当前数据库处理工作负载所需的撤销空间大小。请注意，`V$UNDOSTAT`视图中的每一行都显示一个十分钟时间间隔内的撤销统计信息。该表最多包含 576 行，每行对应一个十分钟间隔。因此，您可以回顾过去最多四天的撤销使用情况。

对于您感兴趣的时间段（理想情况下，该时间段应包括最长查询正在执行的时间），以下是您应在`V$UNDOSTAT`视图中监控的关键列。您可以使用这些统计信息来调整`UNDO_TABLESPACE`和`UNDO_RETENTION`初始化参数的大小。

> `begin_time`：时间间隔的开始时间。
>
> `end_time`：时间间隔的结束时间。
>
> `undoblks`：数据库在十分钟间隔内消耗的撤销块数量；这是我们公式中用于估算撤销表空间大小所使用的数据。
>
> `txncount`：在十分钟间隔内执行的事务数量。
>
> `maxquerylen`：显示在此实例中，在十分钟间隔内执行的最长查询的长度（以秒为单位）。您可以基于`MAXQUERYLEN`列的最大值来估算`UNDO_RETENTION`参数的大小。
>
> `maxqueryid`：此间隔内运行时间最长的 SQL 语句的标识符。
>
> `nospaceerrcnt`：数据库在撤销表空间中没有足够可用空间来存储新撤销数据的次数，这是因为整个撤销表空间都被活动事务占用；当然，这意味着您需要增加撤销表空间的空间。
>
> `tuned_undoretention`：数据库在提交所属撤销的事务后，将保留撤销数据的时间（以秒为单位）。

以下基于`V$UNDOSTAT`视图的查询展示了 Oracle 如何根据当前实例工作负载中最长查询的长度（`MAXQUERYLEN`列）自动调整撤销保留（查看`TUNED_UNDORETENTION`列）。

```sql
SQL> select to_char(begin_time,'hh24:mi:ss') BEGIN_TIME,
  2  to_char(end_time,'hh24:mi:ss') END_TIME,
  3  maxquerylen,nospaceerrcnt,tuned_undoretention
  4  from v$undostat;

BEGIN_TI END_TIME MAXQUERYLEN NOSPACEERRCNT TUNED_UNDORETENTION
-------- -------- ----------- ------------- -------------------
12:25:35 12:29:30         892             0                1673
12:15:35 12:25:35         592             0                1492
12:05:35 12:15:35        1194             0                2094
11:55:35 12:05:35         592             0                1493
11:45:35 11:55:35        1195             0                2095
11:35:35 11:45:35         593             0                1494
11:25:35 11:35:35        1196             0                2097
11:15:35 11:25:35         594             0                1495
11:05:35 11:15:35        1195             0                2096
10:55:35 11:05:35         593             0                1495
10:45:35 10:55:35        1198             0                2098
…
SQL>
```

请注意，`TUNED_UNDORETENTION`列的值会根据任何间隔内的最大查询长度（`MAXQUERYLEN`）持续波动。您可以看到这两个列直接相关，Oracle 会根据给定间隔（十分钟）内的最大查询长度来提高或降低调整后的撤销保留时间。以下查询显示了每个十分钟间隔内撤销块的使用情况和事务计数。

```sql
SQL> select to_char(begin_time,'hh24:mi:ss'),to_char(end_time,'hh24:mi:ss'),
  2  maxquerylen,ssolderrcnt,nospaceerrcnt,undoblks,txncount from v$undostat
  3  order by undoblks
  4  /

TO_CHAR( TO_CHAR( MAXQUERYLEN SSOLDERRCNT NOSPACEERRCNT   UNDOBLKS   TXNCOUNT
-------- -------- ----------- ------------- ---------- ----------
17:33:51 17:36:49         550           0             0          1         18
17:23:51 17:33:51         249           0             0         33        166
17:13:51 17:23:51         856           0             0         39        520
17:03:51 17:13:51         250           0             0         63        171
16:53:51 17:03:51         850           0             0        191        702
16:43:51 16:53:51         245           0             0        429        561

6 rows selected.

SQL>
```

Oracle 通过 OEM 撤销顾问界面提供了一种简单的方法来帮助设置撤销表空间的大小以及撤销保留期。您可以指定顾问进行分析的时间段，最长可回溯一周——顾问使用`AWR`的每小时快照执行其分析。您可以指定撤销保留期以支持闪回事务查询。或者，您可以让数据库根据分析期间最长的查询来确定所需的撤销保留时间。


### 7-2. 查找消耗撤销空间最多的原因

#### 问题

通常，一两个用户会话似乎占用了大量的撤销表空间。您希望识别出是哪个用户以及哪条 SQL 语句正在消耗所有这些撤销空间。

#### 解决方案

高撤销空间使用量通常涉及一个长时间运行的查询。使用以下查询可以找出您的数据库中运行时间最长的 SQL 语句。

```sql
select s.sql_text from v$sql s, v$undostat u
where u.maxqueryid=s.sql_id;
```

您可以联接 `V$TRANSACTION` 和 `V$SESSION` 视图，以找出一个会话在当前执行的事务中使用的最多撤销量，如下所示：

```sql
select s.sid, s.username, t.used_urec, t.used_ublk
from v$session s, v$transaction t
where s.saddr = t.ses_addr
order by t.used_ublk desc;
```

您也可以执行以下查询，以找出实例中当前使用撤销最多的会话：

```sql
select s.sid, t.name, s.value
from v$sesstat s, v$statname t
where s.statistic# = t.statistic#
and t.name = 'undo change vector size'
order by s.value desc;
```

该查询的输出依赖于 `V$STATNAME` 视图中的统计信息 `undo change vector size`，以显示当前消耗最多撤销的会话的 SID。`V$TRANSACTION` 视图显示活动事务的详细信息。这是另一个联接 `V$TRANSACTION`、`V$SQL` 和 `V$SESSION` 视图的查询：

```sql
select sql.sql_text sql_text, t.USED_UREC Records, t.USED_UBLK Blocks,
(t.USED_UBLK*8192/1024) KBytes from v$transaction t,
v$session s,
v$sql sql
where t.addr = s.taddr
and s.sql_id = sql.sql_id
and s.username ='&USERNAME';
```

`USED_UREC` 列显示使用的撤销记录数，`USED_UBLK` 列显示事务消耗的撤销块数。

#### 工作原理

您可以执行“解决方案”部分中描述的查询，来识别数据库中负责最多撤销使用的会话，以及负责这些会话的用户。您可以查询 `V$UNDOSTAT` 并提供适当的 `begin_time` 和 `end_time` 值，以获取某个时间区间内运行时间最长的 SQL 语句的 SQL 标识符。`MAXQUERYID` 列捕获了该 SQL 标识符。您可以使用此 ID 来查询 `V$SQL` 视图，以找出实际的 SQL 语句。同样，`V$TRANSACTION` 和 `V$SESSION` 视图共同帮助识别消耗最多撤销空间的用户。如果过度的撤销使用影响了性能，您可能需要查看应用程序，了解为什么查询会使用如此多的撤销。

### 7-3. 解决 ORA-01555 错误

#### 问题

在关键生产批处理作业的夜间运行期间，您收到了 `ORA-01555`（快照过旧）错误。您希望消除这些错误。

#### 解决方案

虽然为 `UNDO_RETENTION` 参数设置一个较高的值可能会最大限度地减少收到“快照过旧”错误的可能性，但它并不能保证数据库不会覆盖运行中的事务可能需要的较旧的撤销数据。您可以将长时间运行的批处理作业移动到数据库中没有其他程序运行的单独时间间隔，以避免这些错误。

无论如何，虽然通过这些方法可以最小化“快照过旧”错误的发生，但如果不指定*保证撤销保留*功能，您无法完全消除此类错误。当您在数据库中配置了保证撤销保留时，就不会有任何事务因“快照过旧”错误而失败。设置保证撤销保留后，Oracle 会阻止新的 DML 语句执行。实现保证撤销功能很简单。假设您希望确保数据库至少保留一小时（3,600 秒）的撤销。首先使用下面所示的 `alter system` 命令设置撤销保留阈值，然后通过指定 `retention guarantee` 子句来更改撤销表空间，从而设置保证撤销保留。

```sql
alter system set undo_retention=3600;
```
系统已更改。
```sql
alter tablespace undotbs1 retention guarantee;
```
表空间已更改。

您可以通过执行带有 `retention noguarantee` 子句的 `alter tablespace` 命令来关闭保证撤销保留。

![images](img/square.jpg) **提示** 您可以使用本配方中所示的 `alter system` 命令来启用保证撤销保留，也可以使用 `create database` 和 `create undo tablespace` 语句来启用。

#### 工作原理

Oracle 使用存储在撤销表空间中的撤销记录来帮助回滚事务、提供读一致性，并帮助恢复数据库。此外，数据库还使用撤销记录通过 Oracle 闪回查询从过去的时间点读取数据。撤销数据是多个 Oracle 闪回功能的基础，这些功能可帮助您从逻辑错误中恢复。

##### 错误的发生

`ORA-01555` 错误（快照过旧）可能在各种情况下发生。以下是一个在导出过程中发生该错误的案例。

```
EXP-00008: ORACLE error 1555 encountered
ORA-01555: snapshot too old: rollback segment number 10 with name "_SYSSMU10$" too small
EXP-00000: Export terminated unsuccessfully
```

在执行闪回事务时，您也可能收到相同的错误：

```
ERROR at line 1:
ORA-01555: snapshot too old: rollback segment number with name "" too small
ORA-06512: at "SYS.DBMS_FLASHBACK", line 37
ORA-06512: at "SYS.DBMS_FLASHBACK", line 70
ORA-06512: at li
```

当 Oracle 覆盖了另一个事务所需的撤销数据时，就会发生“快照过旧”错误。该错误是 Oracle 读一致性机制工作方式的直接结果。当长时间运行的查询执行期间，Oracle 试图从回滚段中读取任何已更改行的“前映像”时，就可能发生此错误。例如，如果一个长时间运行的查询在凌晨 1 点开始并一直运行到早上 6 点，那么在查询执行期间，数据库有可能更改属于该查询一部分的数据。当 Oracle 试图读取在凌晨 1 点时的数据状态时，如果该数据在回滚段中已不存在，则查询可能会失败。

如果您的数据库正在经历大量更新，Oracle 可能无法获取已更改的行，因为记录在回滚段中的更改前的值可能已被覆盖。更改这些行的事务可能已经提交，并且回滚段中没有更改前行值的记录，因为数据库覆盖了相关的撤销数据。由于 Oracle 无法为当前查询返回一致的数据，因此会发出 `ORA-01555` 错误。当前运行的查询需要前映像来构建读一致的数据，但前映像不可用。

`ORA-01555` 错误可能是由以下一个或两个原因造成的：对数据库的更新过多，或者撤销表空间太小。您可以增加撤销表空间的大小，但这并不能保证错误不会再次发生。



### 范围的影响

数据库将撤销数据存储在撤销范围中，存在三种不同的撤销范围类型：

> `Active`：事务当前正在使用这些范围。
>
> `Unexpired`：这些范围包含满足由`UNDO_RETENTION`初始化参数指定的撤销保留时间所需的撤销。
>
> `Expired`：这些范围包含的撤销保留时间已超过`UNDO_RETENTION`参数指定的持续时间。

如果数据库在撤销表空间中找不到足够的过期范围，或者无法获取新的撤销范围，它将重用未过期（但永远不会是活动撤销范围）的范围，这就为`ORA-01555`，“snapshot too old”错误敞开了大门。默认情况下，如果遇到空间压力以容纳新事务的撤销，数据库实际上会缩小你指定的撤销保留期。由于未过期的撤销范围包含满足撤销保留期所需的撤销记录，覆盖这些范围实际上意味着数据库正在降低你设置的撤销保留期。启用撤销保留保证有助于确保长时间运行的查询以及 Oracle Flashback 操作的成功。撤销保留保证中的“保证”是`真实`的——Oracle 肯定会至少在你指定的时间内保留撤销，并且永远不会覆盖任何包含满足撤销保留期所需撤销的未过期撤销范围。然而，这项保证伴随着一个沉重的代价——即使这意味着 DML 事务因数据库找不到空间来记录这些事务的撤销而失败，Oracle 也会保证保留。因此，在启用保证的撤销保留功能时必须格外谨慎。

### 7-4. 监控临时表空间使用情况

#### 问题

你想要监控临时表空间的使用情况。

#### 解决方案

执行以下查询以找出临时表空间中的已使用和空闲空间。

```sql
SQL> select * from (select a.tablespace_name,
      sum(a.bytes/1024/1024) allocated_mb
      from dba_temp_files a
      where a.tablespace_name = upper('&&temp_tsname') group by a.tablespace_name) x,
      (select sum(b.bytes_used/1024/1024) used_mb,
      sum(b.bytes_free/1024/1024) free_mb
      from v$temp_space_header b
      where b.tablespace_name=upper('&&temp_tsname') group by b.tablespace_name);

Enter value for temp_tsname: TEMP
…
TABLESPACE_NAME                ALLOCATED_MB    USED_MB    FREE_MB
------------------------------ ------------ ---------- ----------
TEMP                             52.9921875 52.9921875          0
SQL>
```

显然，此示例中的临时表空间急需 DBA 的帮助。

#### 工作原理

Oracle 使用临时表空间存储排序操作的中间结果以及任何临时表、临时 LOB 和临时 B 树。你可以创建多个临时表空间，但其中只有一个可以是默认临时表空间。如果你没有显式分配临时表空间，该用户将被分配默认临时表空间。

你不会在`DBA_FREE_SPACE`视图中找到关于临时表空间的信息。如本例所示，使用`V$TEMP_SPACE_HEADER`来查找任何临时表空间中有多少空闲和已用空间。

### 7-5. 识别谁在使用临时表空间

#### 问题

你注意到临时表空间填充速度很快，你想要识别导致临时表空间高使用率的用户和 SQL 语句。

#### 解决方案

发出以下查询，以找出哪个 SQL 语句正在排序段中使用空间。

```sql
SQL> select s.sid || ',' || s.serial# sid_serial, s.username,
     o.blocks * t.block_size / 1024 / 1024 mb_used, o.tablespace,
     o.sqladdr address, h.hash_value, h.sql_text
     from v$sort_usage o, v$session s, v$sqlarea h, dba_tablespaces t
     where o.session_addr = s.saddr
     and o.sqladdr = h.address (+)
     and o.tablespace = t.tablespace_name
     order by s.sid;
```

前面的查询显示了发出 SQL 语句的会话信息，以及临时表空间的名称和该 SQL 语句在该表空间中使用了多少空间。

你可以使用以下查询，找出哪些会话正在临时表空间中使用空间。请注意，信息是汇总形式的，这意味着它不会分离一个会话运行的各种排序操作——它只是给出每个会话的临时表空间总使用量。

```sql
SQL> select s.sid || ',' || s.serial# sid_serial, s.username, s.osuser, p.spid,
     s.module,s.program,
     sum (o.blocks) * t.block_size / 1024 / 1024 mb_used, o.tablespace,
     count(*) sorts
     from v$sort_usage o, v$session s, dba_tablespaces t, v$process p
     where o.session_addr = s.saddr
     and s.paddr = p.addr
     and o.tablespace = t.tablespace_name
     group by s.sid, s.serial#, s.username, s.osuser, p.spid, s.module,
     s.program, t.block_size, o.tablespace
     order by sid_serial;
```

此查询的输出将显示每个会话在临时表空间中使用的空间，以及该会话当前正在执行的排序操作数量。



