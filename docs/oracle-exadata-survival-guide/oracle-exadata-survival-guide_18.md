# 第 8 章 性能测量

![image](img/frontdot.jpg)

在架构方面，Exadata 是一次巨大的飞跃。与传统的商用硬件配置不同，Exadata 提供了一套软硬件匹配的解决方案。然而，它运行的是与您可以在任何非 Exadata 平台上安装的相同的 Oracle `11.2.0.3`。因此，相同的基本性能规则仍然适用；不同之处在于增加了智能扫描、智能闪存缓存和功能下推等功能。由于本书的目的是让您熟悉 Exadata 并具备相关的专业知识，我们将重点介绍 Exadata 特有的性能指标。此外，我们还将探讨与 Exadata 相关的性能主题及其内部机制。



Exadata 提供了大量可供使用的性能指标。这是个好消息。不幸的是，这也意味着要面对海量的数据。在追求性能的道路上，拥有大量的指标可供检查是好事；但缺点是你可能在浩瀚的数据中迷失方向，忘记了自己的初衷。知道要查找什么、如何判断情况良好，更重要的是，如何判断情况糟糕，可能很困难。性能监控的关键不在于查看每一个可用的指标，而在于明智地选择指标。你为何选择特定指标以及你应该看到什么值，这些远比指标本身重要。

最终用户并不关心 Exadata 每秒能提供多少次 I/O；她衡量性能的标准是响应时间。基本上，事情进展得越快，最终用户就越满意。处理性能问题应该是一个监控和优化响应时间的过程。这正是 Oracle 等待接口成为非常有用的工具的地方。这些等待事件在第 7 章中有讨论。Exadata 还提供了其他指标，以提供额外的信息，帮助你进一步提升性能。诸如智能扫描节省的字节数、通过存储索引避免的 I/O，以及智能闪存缓存统计信息等数据，都是提升性能的关键。本章将讨论此类指标、如何获取数据以及这些数字的含义。我们还将讨论如何使用这些信息来监控和排查 Exadata 性能问题。

## 三思而后行

在深入探讨 Exadata 性能指标之前，我们必须回顾一下 Exadata 系统的一些关键部分。无论查询或语句是否使用智能扫描，数据库服务器都会要求存储单元实际检索所需的数据。当使用智能扫描时，存储单元还会通过列投影和谓词过滤协助处理数据。这些过程涉及单元读取数据或索引块、过滤所需的行并仅提取感兴趣的列。存储索引也通过允许存储单元绕过不包含感兴趣列值的 1MB 数据单元来为此过程做出贡献。从数据库服务器的角度来看，存储单元是分发所需数据的“黑盒子”。这些“黑盒子”也会向数据库服务器传回记录存储单元执行了多少工作的指标。存储单元本身提供了丰富的性能指标，其中许多不会传回数据库服务器。不过，你会发现从存储单元获得的信息足以在数据库服务器级别排查许多性能问题。

智能扫描是 Exadata 的命脉；它们是实现超越普通商用硬件配置性能水平的关键。理解智能扫描的性能指标非常重要，基于此，我们将再次讨论它们的一些方面。

## 智能扫描再探

在第 2 章中，我们详细讨论了智能扫描和下推；本章不再重复那么详细的内容。我们将讨论你应该了解的重要方面，以便正确分析提供的指标，从而对性能问题做出合理的诊断。

智能扫描需要直接路径读取和全表扫描、索引快速全扫描或索引全扫描。排查与智能扫描相关的性能问题的第一步是查看是否使用了符合条件的扫描。执行计划会给你这个信息，因此它是合乎逻辑的起点。回到之前的一个例子，我们可以在以下计划中看到符合条件的步骤：

```sql
SQL> select *
  2  from emp
  3  where empid = 7934;

EMPID EMPNAME                                      DEPTNO
---------- ---------------------------------------- ----------
      7934 Smorthorper7934                                  15

Elapsed: 00:00:00.21

Execution Plan

Plan hash value: 3956160932

| Id  | Operation                 | Name | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT          |      |     1 |    28 |  6361   (1)| 00:00:01 |
|*  1 |  TABLE ACCESS STORAGE FULL| EMP  |     1 |    28 |  6361   (1)| 00:00:01 |

Predicate Information (identified by operation id):

1 - storage("EMPID"=7934)
       filter("EMPID"=7934)

Statistics

1  recursive calls
    1  db block gets
40185  consistent gets
22594  physical reads
  168  redo size
  680  bytes sent via SQL*Net to client
  524  bytes received via SQL*Net from client
    2  SQL*Net roundtrips to/from client
    0  sorts (memory)
    0  sorts (disk)
    1  rows processed

SQL>
```

正如我们在第 2 章中所说，仅仅因为计划步骤中有`STORAGE`，并不能保证执行了智能扫描。你必须深入查看，确认是否使用了直接路径读取。在判断是否使用了智能扫描之前，还适用其他标准，例如计划输出中存在列投影数据。谓词过滤也可以在智能扫描中进行。存储索引也可以提供好处，允许 Oracle 绕过不包含感兴趣列值的 1MB 数据段。请注意，在提供的计划中，全扫描步骤包含关键字`STORAGE`，并且谓词信息包含一个`storage()`条目。再次强调，这些并不能保证执行了智能扫描，只表明该语句*符合*智能扫描执行的条件。然而，查询计划中缺少这些项，确实表明不可能发生智能扫描。你可能还会看到一些计划，其中`STORAGE`关键字存在但缺少`storage()`谓词信息，这表明没有执行谓词下推。在这种情况下，你可能会发现查询仍然从列投影中受益。

即使执行计划中存在所有上述标准，仍然有可能不执行智能扫描。为什么？记住，智能扫描需要直接路径读取；如果没有执行直接路径读取，那么就不会执行智能扫描。任何时候使用并行执行时都会发生直接路径读取，但也可以串行发生，特别是当`_serial_direct_read`参数设置为`TRUE`时。在调查智能扫描性能时，必须检查执行计划以及任何表明可能使用了直接路径读取的指标。此类指标在第 2 章中讨论过，相关的等待事件在第 6 章中讨论。这两个信息源都将用于排查智能扫描性能问题。

## 性能计数器与指标

`V$SQL`和`GV$SQL`视图提供了两个我们用于证明智能扫描执行的指标：`io_cell_offload_eligible_bytes`和`io_cell_offload_returned_bytes`。它们用于计算因执行智能扫描而未读取的数据百分比。我们在第 2 章中提供了这个脚本，但再次查看并理解它报告的内容是有益的：

```sql
SQL> select sql_id,
  2  io_cell_offload_eligible_bytes qualifying,
  3  io_cell_offload_eligible_bytes - io_cell_offload_returned_bytes actual,
  4  round(((io_cell_offload_eligible_bytes - io_cell_offload_returned_bytes)/io_cell_offload_eligible_bytes)*100, 2) io_saved_pct,
  5  sql_text
  6  from v$sql
  7  where io_cell_offload_returned_bytes> 0
  8  and instr(sql_text, 'emp') > 0
  9  and parsing_schema_name = 'BING';

SQL_ID        QUALIFYING     ACTUAL IO_SAVED_PCT SQL_TEXT
------------- ---------- ---------- ------------ -------------------------------------
gfjb8dpxvpuv6  185081856   42510928        22.97 select * from emp where empid = 7934

SQL>
```



作为回顾，`io_cell_offload_eligible_bytes`列报告了查询执行期间可以卸载到存储单元的数据字节数。`io_cell_offload_returned_bytes`列报告了通过常规 I/O 路径返回的字节数。这两个值之间的差值反映了查询执行期间实际卸载的字节数。利用这两个指标可以证明智能扫描的执行情况。

另外两个列也能提供关于智能扫描和存储索引使用情况的信息：`cell physical IO bytes saved by storage index`和`cell physical IO interconnect bytes returned by smart scan`。正如第 3 章所述，`cell physical IO bytes saved by storage index`报告的内容如其名称所示，即由于智能扫描期间存储索引允许 Oracle 绕过某些 1MB 块，从而未被读取的总字节数。以下是第 3 章中的一个示例，供复习参考：

```sql
SQL> select *
  2  from v$mystat
  3  where statistic# = (select statistic# from v$statname where name = 'cell physical IO bytes saved by storage index');

SID STATISTIC#      VALUE
---------- ---------- ----------
      1107        247 1201201152

SQL>
```

该值在会话级别通过 `V$MYSTAT` 报告。需要记住的是，此统计信息在会话持续期间是累积的。在查询执行前后查询 `V$MYSTAT` 并计算差值，即可显示该特定查询所节省的字节数。

`cell physical IO interconnect bytes returned by smart scan`报告通过智能扫描操作返回的实际字节数。`V$SQLSTATS` 会报告此值，以及另外两个值 `physical_read_bytes` 和 `physical_write_bytes`。这些值可用于评估智能扫描的“效率”（为方便表述而称）。此主题在第 10 章中有更深入的介绍，但需要注意，当查询使用不符合智能扫描执行条件的排序或索引访问路径时，这种“效率”可能超过 100%。

了解语句等待的内容也能提供关于智能扫描执行的信息。直接路径读通常表示智能扫描活动，但并非总是如此。LOB、索引组织表、非全扫描或快速全扫描的索引扫描，以及通过 `rowid` 取表数据，也可能使用直接路径读，但不符合智能扫描的条件。因此，即使在执行计划中已注意到存储访问并记录了直接路径读等待，Oracle 在处理查询或语句时也可能并未使用智能扫描。

表明未使用智能扫描的 Exadata 等待事件是 `cell multiblock physical read` 和 `cell single-block physical read`。`cell multiblock physical read` 等待不仅适用于表，也适用于块大小大于数据库块大小的 LOB 以及安全文件读。当块大小小于或等于数据库块大小时，将使用 `cell single block physical read`。如第 7 章所述，`cell single block physical read` 可应用于索引访问路径，但由于此类等待在索引全扫描和索引快速全扫描期间不会发生，因此可以判断未执行智能扫描。

用户关心时间，特别是响应时间，因此性能调整的第一步应该是审视 Exadata 等待接口。观察查询和语句的时间花费在哪里，可以帮助定位性能不佳的领域。了解时间消耗在何处，为您解决此类性能问题提供了初步方向，并可能证明数据库并非故障所在。

## 动态计数器

`V$SYSSTAT` 和 `V$SESSTAT` 视图提供了丰富的动态性能计数器，可帮助您诊断性能问题。根据会话花费时间的位置，这些视图可以帮助定位可能需要关注的领域。例如，如果 Oracle 使用了大量 CPU 资源，一个可能的排查途径是查看给定会话的逻辑读计数如何增加。`V$SYSSTAT` 提供了以下与读相关的计数器列表：

```sql
SQL> select name, value
  2  from v$sysstat
  3  where name like '%read%'
  4  /
```


## 数据库会话与系统统计信息分析

| 名称                                           | 值           |
| ---------------------------------------------- | ------------ |
| session logical reads                          | 82777757613  |
| session logical reads in local numa group      | 0            |
| session logical reads in remote numa group     | 0            |
| physical read total IO requests                | 21591992     |
| physical read total multi block requests       | 12063669     |
| physical read requests optimized               | 3355758      |
| physical read total bytes optimized            | 41508200448  |
| physical read total bytes                      | 11935356259328 |
| logical read bytes from cache                  | 669321109651456 |
| physical reads                                 | 1456537556   |
| physical reads cache                           | 43060982     |
| physical read flash cache hits                 | 0            |
| physical reads direct                          | 1413476574   |
| physical read IO requests                      | 21382598     |
| physical read bytes                            | 11931955658752 |
| recovery blocks read                           | 0            |
| recovery blocks read for lost write detection  | 0            |
| physical reads direct temporary tablespace     | 26335818     |
| DBWR thread checkpoint buffers written         | 11428910     |
| recovery array reads                           | 0            |
| recovery array read time                       | 0            |
| physical reads cache prefetch                  | 36901401     |
| physical reads prefetch warmup                 | 233120       |
| physical reads retry corrupt                   | 0            |
| physical reads direct (lob)                    | 14262        |
| cold recycle reads                             | 0            |
| physical reads for flashback new               | 0            |
| flashback cache read optimizations for block new | 0          |
| flashback direct read optimizations for block new | 0         |
| redo blocks read for recovery                  | 0            |
| redo k-bytes read for recovery                 | 0            |
| redo k-bytes read for terminal recovery        | 0            |
| redo KB read                                   | 0            |
| redo KB read (memory)                          | 0            |
| redo KB read for transport                     | 0            |
| redo KB read (memory) for transport            | 0            |
| gc read wait time                              | 373          |
| gc read waits                                  | 5791         |
| gc read wait failures                          | 0            |
| gc read wait timeouts                          | 180          |
| gc reader bypass grants                        | 0            |
| Number of read IOs issued                      | 5714868      |
| read-only violation count                      | 0            |
| Batched IO vector read count                   | 1455         |
| transaction tables consistent reads - undo records applied | 262 |
| transaction tables consistent read rollbacks   | 15           |
| data blocks consistent reads - undo records applied | 16126989 |
| no work - consistent read gets                 | 77857741416  |
| cleanouts only - consistent read gets          | 19160349     |
| rollbacks only - consistent read gets          | 187199       |
| cleanouts and rollbacks - consistent read gets | 1563685      |
| table scans (direct read)                      | 441631       |
| lob reads                                      | 9246167      |
| index fast full scans (direct read)            | 558680       |
| securefile direct read bytes                   | 0            |
| securefile direct read ops                     | 0            |
| securefile inode read time                     | 0            |
| cell flash cache read hits                     | 3355758      |

已选择 58 行。

### 使用 `V$SYSSTAT` 与 `V$SESSTAT`

`V$SYSSTAT` 提供了系统范围的统计信息视图，因此使用此视图来精确定位特定会话的问题将很困难。`V$SESSTAT` 提供了会话级别的统计信息，当某个特定会话成为问题时，它将更有用。查看我们一个数据库中 SID 为 1141 的非零统计信息，得到以下输出：

```sql
SQL> select sn.name, ss.value
  2  from v$sesstat ss join v$statname sn on sn.statistic# = ss.statistic#
  3  and ss.sid in (select sid from v$session where status ='ACTIVE')
  4  and ss.value > 0
  5  and ss.sid=1141
  6  order by 1, 2
  7  /

NAME                                           VALUE
----------------------------------------- ---------------
cell flash cache read hits                      246424
cell physical IO interconnect bytes        4037787648
cluster wait time                                    2
enqueue releases                                246444
enqueue requests                                246444
ges messages sent                                  758
in call idle wait time                       13559183
logons cumulative                                   1
logons current                                       1
messages received                                    4
messages sent                                       34
non-idle wait count                            2902944
non-idle wait time                               57613
physical read requests optimized                246424
physical read total IO requests                 246441
physical read total bytes                   4037689344
physical read total bytes optimized         4037410816
physical write total IO requests                     2
physical write total bytes                       32768
session pga memory                              7841624
session pga memory max                          7841624
session uga memory                               180736
session uga memory max                           180736
user calls                                           2

24 rows selected.

SQL>
```

给定会话有更多可用的统计信息；该查询过滤掉了值为 0 的项。

### 何时以及如何使用它们

既然你知道在哪里可以找到有用的性能指标和等待统计信息，下一步就是知道如何以及何时使用它们来诊断性能问题。尽管有大量可用的统计信息和指标可用于深入挖掘性能问题，但性能评估和调优的主要关注点和驱动力仍然是时间。用户通过任务从开始到完成所需的时间来衡量性能，世界上所有的 I/O 统计信息对大多数用户来说意义不大。性能之旅的第一站是查看相关时间段的 AWR 报告。AWR 提供了查找问题和开始诊断过程所需的信息。

## AWR 报告分析与性能调优

AWR 报告基于对性能指标和等待事件计时的定期快照生成。这种方式能够基于快照之间记录的变化来生成报告。利用 AWR 报告提供的信息，可以找出运行时间长的 SQL 语句及其关联的 `SQL_id`，从而生成并分析执行计划。此外，可以使用 `V$SESSTAT`、`V$SYSTEM_EVENT` 和 `V$SYSTAT` 视图深入分析等待事件和指标，以期发现性能下降的根本原因。查看“`SQL Ordered by Elapsed Time`”报告有助于发现问题语句和查询。这不仅仅是耗时的问题，还涉及到耗时和执行次数的关系，因为较长的总耗时如果伴随着高执行次数，会导致每次执行的平均时间很短。例如，如果总耗时为 7734.27 秒，执行次数为 1281，那么每次执行的平均耗时约为 6 秒，这在大多数标准下是合理的。另一方面，如果总耗时为 3329.17 秒，执行次数仅为 3，那么每次执行的平均耗时约为 1110 秒。如此量级的单次执行耗时值得更仔细的检查。这正是 `V$SESSTAT`、`V$SYSTEM_EVENT` 和 `V$SYSTAT` 视图能够提供更详细信息的地方。

另一个很好的性能信息来源是 10046 跟踪；可以是其原始形式，也可以是由 `tkprof` 工具格式化的报告。我们通常更喜欢 `tkprof` 格式的输出，因为它提供了执行计划以及相关的等待事件和时间。

![image](img/sq.jpg) **注意** 仅当 10046 跟踪事件设置在级别 8 或更高时，`tkprof` 报告中才会提供等待信息。较低的级别不会捕获等待统计信息。

查看提供了等待信息的 `tkprof` 格式化跟踪文件的一部分，可以获得以下数据：

```
********************************************************************************

SQL ID: 23bfbq45y94fk Plan Hash: 2983102491

DELETE FROM APP_RULE_TMP_STAGE
WHERE
 TMP_RULE_NAME = :B1

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute     22      9.38     125.83      25833       4732     493424      346425
Fetch        0      0.00       0.00          0          0          0           0
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total       23      9.38     125.83      25833       4732     493424      346425

Misses in library cache during parse: 1
Misses in library cache during execute: 4
Optimizer mode: ALL_ROWS
Parsing user id: 113     (recursive depth: 1)
Number of plan statistics captured: 1

Rows (1st) Rows (avg) Rows (max)  Row Source Operation
---------- ---------- ----------  ---------------------------------------------------
         0          0          0  DELETE  APP_RULE_TMP_STAGE (cr=420 pr=3692 pw=0 time=26339915 us)
     44721      44721      44721   INDEX RANGE SCAN APP_RULE_TMP_STG_IDX2 (cr=377 pr=353 pw=0 time=1603541 us cost=36 size=7291405 card=3583)(object id 196496)

Elapsed times include waiting on following events:
  Event waited on                             Times   Max. Wait  Total Waited
  ----------------------------------------   Waited  ----------  ------------
  library cache lock                              1        0.00          0.00
  row cache lock                                 18        0.00          0.00
  library cache pin                               1        0.00          0.00
  Disk file operations I/O                        6        0.00          0.00
  cell single block physical read             25783        1.30        114.29
  gc cr grant 2-way                            1587        0.00          0.18
  gc current grant 2-way                      14205        0.00          1.53
  gc current grant congested                     69        0.00          0.00
  gc cr grant congested                          14        0.00          0.00
  cell list of blocks physical read              20        0.31          0.48
  gc current multi block request                 12        0.00          0.00
  log buffer space                                2        0.83          0.86
********************************************************************************
```

在这个语句中，值得关注的等待是 `cell single block physical read`，鉴于其总耗时（以秒为单位报告）。执行计划给了我们一个线索，这可能是一个需要与 `INDEX RANGE SCAN` 步骤一起考虑的等待。请记住，使用 `INDEX RANGE SCAN` 会使该语句不符合 Smart Scan 执行的条件。另请注意，这里没有使用并行执行。考虑到该语句的总耗时，它本应符合自动并行度（Auto Degree of Parallelism）的条件。使用并行执行很可能通过将 I/O 分布到多个并行查询从属进程上来改善该语句的性能，从而减少总的等待时间。

### 补充说明

我们已经在第 2 章中介绍了与 Smart Scan 相关的查询计划。回顾那部分内容总是有帮助的。

有三个计划步骤可以指示可能的 Smart Scan 活动：

*   `TABLE ACCESS STORAGE FULL`
*   `INDEX STORAGE FULL SCAN`
*   `INDEX STORAGE FAST FULL SCAN`

在本章前面，我们提到过，计划步骤中的 `STORAGE` 一词仅表示可能发生 Smart Scan 执行，因为没有直接路径读取（direct-path reads），就不会触发 Smart Scan。然而，执行计划可以为性能调优提供一个良好的起点。执行计划步骤中缺少 `STORAGE` 则表明 Smart Scan *没有*发生。如果你检查某个给定语句的计划并在某个表或索引访问步骤中找到了 `STORAGE` 关键字，你可以使用 `V$SQL` 和 `GV$SQL` 中的数据来验证 Smart Scan 的执行情况。如第 2 章所述，这些视图中至少有两列提供了关于 Smart Scan 对查询或语句有多大益处的信息。我们介绍过的两列是 `io_cell_offload_eligible_bytes` 和 `io_cell_offload_returned_bytes`。我们还提供了一个可以计算实现的 I/O 节省百分比的查询，这在本章前面的“性能计数器和指标”部分也再次说明过。

### 性能计数器参考

Exadata 更新了许多计数器，但这里不会讨论完整的列表。我们选择描述那些我们认为在性能和故障排除方面最“有趣”的计数器。很可能你不需要深入钻研这些来解决大多数性能问题，因为下一节讨论的等待接口和 SQL 监控通常已经是足够的诊断工具。了解这些计数器是好的，这就是我们提供它们的原因。我们将按名称列出它们，描述它们的目的及其背后的机制，以便让你深入了解 Exadata 是如何工作的。

`cell blocks helped by commit cache`


#### 一致读机制概述

提交缓存是 Oracle 多年来使用的一致读机制的一部分，区别在于其实现于存储单元层面而非数据库服务器层面。回顾一下，在非 Exadata 硬件的标准 Oracle 数据库中，一致读涉及根据查询开始时间，利用可用的 Undo 数据，基于锁字节是否被设置来重构该时间点的行数据。在一致读处理期间可能出现两种条件。

#### 提交 SCN 与快照 SCN 的比较

第一种条件是提交 SCN 低于快照 SCN（在查询开始时建立）。此时，Oracle 可以判定块中的数据无需重构，并可继续处理其他块。

第二种条件是提交 SCN 大于快照 SCN。此时，Oracle 知道需要回滚以返回一致数据，并且在非 Exadata 系统上，它通过可用的 Undo 记录来完成此操作。更复杂的是，提交并不总是清理其触及的每个块；会设置一个阈值以限制提交后立即发生的块清理活动。（此限制为块缓冲区缓存的 10%；超过此限制的任何块必须等待下一个触及它们的事务来进行清理。）

#### 延迟块清理问题

这导致数据块处于必须确定事务状态的状态。通常在非 Exadata 系统上，数据库层执行此处理。想象一下，需要将这些块从存储单元发送回数据库层，以通过 Undo 记录执行标准的一致读处理。对于少数块，这在 Exadata 上可能可行，但存在需要以这种方式处理的块数量极大的可能性。性能会很慢，从而剥夺了 Exadata 的一个关键特性。由于 Undo 数据对存储单元不可用（存储单元无法访问数据库缓冲区缓存），且任何存储单元不与其他存储单元通信，因此必须采用另一种机制。

#### 提交缓存优化

Exadata 采用一种优化，以最小化数据库层处理一致读请求的需要。它被称为提交缓存。提交缓存跟踪哪些事务 ID 已提交，哪些未提交。通过从数据块中的感兴趣事务列表（ITL）中提取事务 ID，存储单元可以访问此提交缓存并查看引用的事务是否已提交。如果该特定事务 ID 的信息在提交缓存中不可用，存储单元会向数据库层请求其状态。收到后，它会将其添加到提交缓存中，因此其余存储单元无需再次发出请求。

每次存储单元在提交缓存中找到其所需的事务信息时，`cell blocks helped by commit cache` 计数器会增加 1。在事务活动频繁期间监控此计数器，当等待接口和 SQL 监视器未提供足够数据来诊断性能问题时，会非常有帮助。在 Smart Scan 期间看到此计数器增加，表明单元正在执行延迟块清理，这是因为插入、更新或删除的数据量大于清理阈值所要求的。提交缓存正是为此目的而设计，并阻止存储单元持续与数据库层通信。它还显著减少了 Smart Scan 期间数据库层的逻辑 I/O。

#### 最小活动 SCN 优化

**受 minscn 优化帮助的存储单元块**

另一个 Exadata 特有的一致读优化是最小活动 SCN 优化，它跟踪仍处于活动状态的事务的最低 SCN。这通过允许 Exadata 将来自 ITL 的事务 SCN 与数据库中最旧仍活动事务的最低 SCN 进行比较，来提高一致读性能。数据库层在 Smart Scan 操作开始时将此信息发送到单元。当提交 SCN 低于传递给单元的最小活动 SCN 时，可以避免数据库层与单元之间不必要的通信。在块 ITL 中找到的任何低于最小活动 SCN 的事务 SCN，存储单元都已知其已提交。每个从此机制受益的块都会递增 `cell blocks helped by minscn optimization` 计数器。在性能方面，此优化也减少了对前面讨论的提交缓存的检查。因为这是 Exadata，很可能运行在其上的数据库是 RAC 数据库，所以最小活动 SCN 是 RAC 感知的。实际上称为全局最小 SCN，每个实例中的 MMON 进程跟踪此 SCN 并更新每个节点的 SGA。`X$TUMASCN` 表包含 RAC 集群的当前全局最小 SCN，如下所示：

```
SQL> select min_act_scn
  2  from x$ktumascn;

MIN_ACT_SCN
-----------
    12345678
SQL>
```

`cell blocks helped by minscn optimization` 计数器不是您会经常使用的（如果有的话）。但是，当 Smart Scan 因频繁回退到块 I/O 以从数据库层获取事务信息而中断时，这是一个很好的起点。

#### 存储单元提交缓存查询计数器

**存储单元提交缓存查询**

每次 Smart Scan 查询单元提交缓存以获取事务状态时，此计数器会递增。对于在块中找到的每个未提交事务，当 MinActiveSCN 优化不适用于其事务 SCN（意味着事务 SCN 大于或等于最小活动 SCN）时，会执行此操作。此计数器与 `cell blocks helped by minscn` 计数器密切相关。

#### 存储单元缓存层处理计数器

**由缓存层处理的存储单元块**

这是 Exadata 收集的四个“层”统计信息之一，报告所列处理层的活动。此特定统计信息报告为 Smart Scan 操作由存储单元实际处理的块数。当单元将块传回数据库服务器（块 I/O 模式）时，此统计信息不会递增。当单元在 Smart Scan 期间实际处理块时，它会递增。当单元为一致读打开块时，会检查块缓存头，以确保正在读取正确的块，并且该块有效且未损坏。缓存层进程（内核缓存缓冲区管理，或 KCB）执行这些功能并将 `cell blocks processed by cache layer` 计数报告回数据库。

当数据库层处理常规块 I/O 时，可能会更新两个统计信息之一：`consistent gets from cache` 或 `consistent gets from cache (fastpath)`。存储单元通过 CELLSRV 仅执行一致读取，而非当前模式读取。记录在 `db block gets from cache` 或 `db block gets from cache (fastpath)` 中的所有当前模式读取均来自数据库层。`cell blocks processed by cache layer` 计数器中记录的任何计数都提供了有关存储单元执行了多少逻辑读取的数据，无论是从 `V$SYSSTAT` 的系统范围数据，还是通过 `V$SESSTAT` 的会话级别数据。

在查询执行计划中同时发现数据库层和单元层处理并不罕见。较小的表不太可能符合 Smart Scan 的条件，因此将由数据库服务器处理。使用小型查找表和大型处理表的多表连接可能会产生此类计划。

#### 存储单元数据层处理

**由数据层处理的存储单元块**



## 数据库性能统计指标详解

### 概述
上一个计数器记录了单元格缓存读取活动，而本统计信息则记录从表或物化视图中读取的物理块。数据层模块的内核数据扫描（KDS）从数据块中提取行和列，传递给谓词过滤和列投影处理。当存储单元格无需数据库块 I/O 即可完成所有必要工作时，此计数器会递增。

### 统计指标关联分析
当会话级的本统计信息计数与会话级的 `cell blocks processed by index layer` 计数相加时，可以确定存储单元格是否完成了所有一致性读取。如果每个处理块都在存储层读取，则这些计数器的总和将等于会话级的 `cell blocks processed by cache layer` 计数器计数。如果总和较小，则表示数据库层执行了常规块 I/O 处理。其差值应为数据库服务器处理的块数。

#### cell blocks processed by index layer
存储单元格可以以类似方式处理索引块与表和物化视图块。此统计信息记录通过智能扫描处理 B*树和位图索引段的索引块数量。

#### cell blocks processed by txn layer
由于此计数器记录事务层处理的块数，了解该层的内部操作将很有帮助。回顾智能扫描处理一致性读取的基本说明，我们来看一下在此过程中会发生什么。

缓存层通过 KCB 打开块，然后检查头部、最后修改的 SCN 和清理状态。如果最后修改的 SCN 不大于快照 SCN（因为自查询开始后未被修改），事务层将获取此块。如果最后修改的 SCN 大于快照 SCN，则表明块自查询开始后已被修改，需要回滚到快照 SCN。这会导致块被传递回数据库层进行块 I/O 处理。

如果块通过第一次测试，它将被传递到事务层，由内核事务读一致性进程（KTR）处理，该进程使用最小活动 SCN 和提交缓存，减少与数据库层的通信量。然后事务层执行数据块处理以提取请求的信息，与数据层和索引层协同工作，在存储单元格级别执行一致性读取。

#### cell flash cache read hits
此计数器在第 4 章中已讨论过，这里仅简要提及。这是另一个累积指标，存在于会话级和实例级。监控此指标的最简单方法是使用我们在第 4 章中提供的脚本。我们也在此提供，因为监控智能闪存缓存是性能故障排除的关键部分。

```sql
SQL> select statistic#, value
  2  from v$mystat
  3  where statistic# in (select statistic# from v$statname where name = 'cell flash cache read hits');

STATISTIC#      VALUE
---------- ----------
       605          1

SQL>
SQL> select count(*)
  2  from emp;

COUNT(*)

SQL>
SQL> column   val new_value endval
SQL>
SQL> select statistic#, value val
  2  from v$mystat
  3  where statistic# in (select statistic# from v$statname where name = 'cell flash cache read hits');

STATISTIC#        VAL
---------- ----------
       605        857

SQL>
SQL>
SQL> select &endval - &beginval flash_hits
  2  from dual;

FLASH_HITS

SQL>
```

此计数器还提供物理 I/O 请求的数量，因为访问智能闪存缓存是物理 I/O 而非逻辑 I/O。

#### cell index scans
每次在 B*树或位图索引段上启动智能扫描时，此计数器都会递增。由于这需要执行计划中的索引快速全扫描步骤，并且还必须使用直接路径读取，因此必须同时满足这两个条件才能使此计数器递增。

智能扫描是段级操作，因此在非分区索引上串行运行的查询将对每个访问的索引段递增一次。对于并行访问的分区索引段，每个并行查询从进程访问每个分区段时，此计数器可能递增一次。由于分区段的大小可能不同，对于给定的分区索引，可以通过不同方法访问这些段。较大的段可能通过直接路径读取访问，因此此计数器将递增。较小的分区可能不会触发直接路径读取，在这种情况下，不会执行智能扫描，计数器值也不会增加。

#### cell IO uncompressed bytes
如果不使用压缩，您将不会看到此统计信息发生变化，因为它仅在智能扫描压缩卸载发生时递增。此统计信息报告压缩数据的实际未压缩大小。例如，如果有 10MB 的压缩数据，解压缩后为 20MB，则在该数据卸载时，`cell IO uncompressed bytes` 计数器将增加 20MB。`physical read total bytes` 计数器将增加 10MB，因为这是段大小。看到 `cell IO uncompressed bytes` 计数器比 `physical read total bytes counter` 增长更快不是问题，因为如前所述，`physical read total bytes` 计数器根据读取的段大小递增，而 `cell IO uncompressed bytes` 计数器报告处理的未压缩总字节数。

#### cell num fast response sessions
Oracle 可以选择延迟完整智能扫描执行，首先执行少量块 I/O 操作，尝试满足查询。`FIRST_ROWS`、`FIRST_ROWS_n` 和 `where rownum` 查询会触发此行为，当 Oracle 决定不立即执行智能扫描时，此计数器会递增。为了提高查询效率和速度，Oracle 可以避免设置智能扫描，转而执行少量块 I/O 操作来满足查询。此计数器报告自实例启动以来 Oracle 执行此操作的次数。此计数器递增并不意味着未执行完整的智能扫描，只是 Oracle 尝试尽可能少地工作来返回所需结果。

#### cell num fast response sessions continuing to smart scan
在 Oracle 决定延迟运行智能扫描以减少可能的工作量后，可能认为必须执行智能扫描才能向调用会话返回正确结果。当做出此切换时，此计数器会递增。由于这发生在 Oracle 最初决定不运行完整智能扫描之后，因此适用相同的触发条件——正在运行 `FIRST_ROWS`、`FIRST_ROWS_n` 和 `where rownum` 查询。此外，当此统计信息递增时，您可能会注意到一些 `db file sequential read` 等待。尽管大多数此类读取报告为 `cell single block physical read` 等待事件，但较旧命名的事件在 Exadata 上仍可能递增。

#### cell num smart IO sessions using passthru mode due to _______


