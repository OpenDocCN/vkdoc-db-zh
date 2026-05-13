# EHCC 相关计数器

正如本章引言所述，由于 HCC 相关的计数器数量过于庞大，本节无法涵盖每一个计数器。因此，我们将 HCC 处理过程与其他内容结合起来进行阐述。总而言之，Exadata 中的 HCC 处理存在两种场景。第一种是**智能扫描（Smart Scan）**，此时存储单元负责解压缩 CUs（压缩单元），并将仅与查询相关的信息传递给 RDBMS 层。在这种场景下，你会发现名为 `cell CU%` 的计数器递增。这些计数器是在存储单元级别递增的。此外，你还会注意到名为 `EHCC%` 的计数器也会递增，正如后文所示，这可能导致**重复计数**。这些 `EHCC%` 计数器与 RDBMS 层上的 HCC 处理相关。

如果你发现 `EHCC%` 计数器递增而 `cell CU%` 没有递增，那么你的查询并未被**下推（offloaded）**。换句话说，执行过程中没有涉及智能扫描。因此，HCC 处理需要完全在 RDBMS 层上进行。Oracle 对 HCC 数据的压缩和解压缩过程进行了完善的检测。例如，考虑创建一个 HCC 压缩表。登录到一个新会话后，`EHCC%` 计数器已被重置。

```sql
SQL> create table t1_qh column store compress for query high as select * from t1;
```

一次快速查询揭示了结果：

```sql
SQL> !cat hcc_stats.sql

select name, value value_bytes, round(value/power(1024,2),2)  value_mb
from v$statname natural join v$mystat
where (name like ’%EHCC%’ or name like ’cell CU%’)
and value <> 0;

SQL> @hcc_stats

NAME                                                                  VALUE_BYTES        VALUE_MB
---------------------------------------------------------------- --------------- ---------------
EHCC CUs Compressed                                                           2148               0
EHCC Query High CUs Compressed                                               2148               0
EHCC Compressed Length Compressed                                        67439392           64.32
EHCC Decompressed Length Compressed                                   10709897349        10213.75
EHCC Rows Compressed                                                     10349046            9.87
EHCC CU Row Pieces Compressed                                               9899             .01
EHCC Analyzer Calls                                                             1               0

7 rows selected.
```

翻译成中文，这意味着压缩了 2148 个 CUs，全部使用了 Query High 算法。数据可以从大约 10,214 MB 压缩至约 64 MB，这是一个可观的节省。统计数据显示，超过一百万行数据被压缩成了 9899 个 CU 行片段。压缩分析器——在第 3 章有详细解释——被调用了一次，因为这里只有一个非分区表需要压缩。

查询数据展现出的场景与刚才演示的表创建不同。以下是一个未被下推的段扫描示例（`cell CU%` 计数器已在前文讨论，此处不再显示）：

```sql
SQL> select /* gather_plan_statistics test002 */ count(id),id
  2   from t1_qh group by id having count(id) > 100000;

no rows selected

SQL> @hcc_stats

NAME                                                                  VALUE_BYTES   VALUE_MB
---------------------------------------------------------------- ----------- ----------
EHCC CUs Decompressed                                                      4129          0
EHCC Query High CUs Decompressed                                           4129          0
EHCC Compressed Length Decompressed                                   130329591     124.29
EHCC Decompressed Length Decompressed                                2.0697E+10    19738.6
EHCC Columns Decompressed                                                  4129          0
EHCC Total Columns for Decompression                                      24774        .02
EHCC Total Rows for Decompression                                        20000000      19.07
EHCC Pieces Buffered for Decompression                                     4161          0
EHCC Total Pieces for Decompression                                       19829        .02
EHCC Turbo Scan CUs Decompressed                                           4129          0

10 rows selected.
```

在这里，你可以看到解压缩了 4129 个 CUs，全部使用了 Turbo Scan 解压缩方法。共有 19,829 个片段需要解压缩，产生了 20,000,000 行，这正是整个表的行数。实际解压的列数比所有 CUs 中的总列数要少：4129 列 vs. 27,774 列，这很好地展示了得益于**面向列**的方法，能够减少所需完成的工作量。通过比较 `EHCC Decompressed Length Decompressed` 和 `EHCC Compressed Length Decompressed`，你可以推断出解压缩的有效性。

关于 EHCC 统计信息有一点需要注意：在存储单元层和 RDBMS 层存在一些**重复计数**。请看下面的例子。它使用了 Adrian Billington 的“mystats”脚本来捕获 SQL 语句执行期间会话计数器的变化。该工具可以从 `oracle-developer.net` 下载，我建议你了解一下。以下仅摘取了输出的相关信息：

```sql
SQL> @mystats start
SQL> select /* test006 */ count(id),id
  2  from BIGTAB_QH group by id having count(id) > 100000;
SQL> @mystats stop t=1

STAT    cell CUs processed for uncompressed                                104,289
STAT    cell CUs sent uncompressed                                         104,289
STAT    cell IO uncompressed bytes                                  71,828,529,728
STAT    cell blocks helped by minscn optimization                          404,724
STAT    cell blocks processed by cache layer                                404,724
STAT    cell blocks processed by data layer                                 404,724
STAT    cell blocks processed by txn layer                                  404,724
STAT    cell flash cache read hits                                           2,991
STAT    cell scans                                                               1
STAT    EHCC CUs Decompressed                                               208,578
STAT    EHCC Columns Decompressed                                           208,578
STAT    EHCC Compressed Length Decompressed                          4,744,301,064
STAT    EHCC Decompressed Length Decompressed                     143,641,134,208
STAT    EHCC Pieces Buffered for Decompression                              208,872
STAT    EHCC Query High CUs Decompressed                                    104,289
STAT    EHCC Total Columns for Decompression                              1,668,624
STAT    EHCC Total Pieces for Decompression                                  542,756
STAT    EHCC Total Rows for Decompression                                512,000,000
STAT    EHCC Turbo Scan CUs Decompressed                                    104,289

------------------------------------------------------------------------------------------
3. About
------------------------------------------------------------------------------------------
- MyStats v2.01 by Adrian Billington (http://www.oracle-developer.net)
- Based on the SNAP_MY_STATS utility by Jonathan Lewis
```


在上述输出中，您看到的是一个双重计数的例子。该单元报告在此智能扫描（Smart Scan）过程中处理了 104,289 个 CU（单元发送的 CU 未经压缩）。当数据到达关系数据库管理系统（RDBMS）层时，它被计入 `Turbo Scan CUs decompressed` 计数器中，这与单元报告的数量相符。尽管数据在到达 RDBMS 层时已经解压缩，但它似乎很可能再次遍历相同的代码路径，导致某些统计信息被再次递增。这在 `EHCC CUs Decompressed`、`EHCC Columns Decompressed` 以及 `EHCC Pieces Buffered for Decompression` 中也清晰可见。

另一个双重计数的例子体现在 `cell IO uncompressed bytes` 和 `EHCC Decompressed Length Decompressed` 中，后者的值是前者的两倍。

#### 优化的物理读取请求 (`physical read requests optimized`)

此统计信息显示避免了多少次磁盘 I/O 请求，这些请求通过从闪存缓存（Flash Cache）而非磁盘读取数据，和/或得益于存储索引（storage index）消除 I/O 而被避免。此统计信息也会传播到 `V$SQL/V$SQLSTATS` 和 `V$SEGMENT_STATISTICS` 视图中。

#### 优化的物理读取总字节数 (`physical read total bytes optimized`)

此统计信息显示避免了多少字节的物理磁盘驱动器 I/O，这些 I/O 通过从闪存缓存读取和/或得益于存储索引消除 I/O 而被避免。当您同时看到 `cell physical I/O bytes saved by storage index` 统计信息也等量增加时，这意味着部分 I/O 完全得益于存储索引而得以避免。如果存储索引节省的字节数小于优化的总字节数，则其余的 I/O 是通过从闪存缓存读取而优化的，而非老旧的机械硬盘。在这种情况下，您还可以期望看到 `Flash Cache read hits`。

#### 表获取续行 (`table fetch continued row`)

此统计信息并非 Exadata 特有，但在排查使用智能扫描时数据库意外执行的单块读取问题时，它很有相关性。此统计信息计算当 Oracle 在 `buffer cache` 中找不到链接行的下一个行片时，不得不使用常规单块读取来获取它的次数。有关更深入的信息，请参阅本章前文关于链接行的描述。

#### 表扫描（直接路径读取） (`table scans (direct read)`)

此统计信息并非 Exadata 特有；在任何使用直接路径读取对表段执行全表扫描的 Oracle 数据库中都能看到它。在串行执行期间，此统计信息在表或段扫描开始时递增。然而，在并行执行中，每当一个从进程开始扫描分配给它的一个新的 ROWID 范围时，它就会递增一次。直接路径读取是智能扫描发生的先决条件。当您没有看到预期的智能扫描时，一个快速的故障排查选项是使用此统计信息检查是否发生了直接路径读取。另一个快速提示：当使用 Snapper 对正在执行且未扫描多个分区的查询进行故障排查时，您可能看不到此统计信息的条目。这并不意味着没有发生直接路径读取——可能是您在计数器已经递增之后才开始排查该会话。

#### 表扫描（长表） (`table scans (long tables)`)

这是一个与前一个类似的统计信息，但它显示被扫描的表是否被认为是大表。实际上，Oracle 对每个段都是单独考虑的，因此表的某些分区可能被认为是小的，有些被认为是大的。被 Oracle 认为是小的段会递增 `table scans (short tables)` 计数器。如果在全扫描期间总是读取到高水位标记（high water mark）的段大于 `buffer cache` 的 10%，则该表被认为是大表，即使是串行全段扫描也会考虑使用直接路径读取。请注意，此决策逻辑还考虑了其他因素，这些因素在第 2 章中有更详细的解释。缓冲缓存百分之十的规则实际上源自 `_small_table_threshold` 参数。该参数默认为缓冲缓存大小（以块计）的百分之二，但在 Oracle 11.2 的早期版本中，Oracle 使用 `5 × _small_table_threshold` 作为其直接路径扫描决策阈值（取决于缓冲缓存中的块数和一些其他因素）。在当前版本（包括 11.2.0.3）中，即使表的大小只是略大于 `_small_table_threshold`，它也有资格进行直接路径读取/智能扫描。同样，该逻辑在第 2 章中有所阐述。

同样值得注意的是，对段头的单块 I/O 不再决定段的大小。Oracle 11.2.0.2 及更高版本使用关于表的字典信息。



## 理解 SQL 语句性能

本节重点介绍 SQL 语句的性能指标，并帮助理解语句的时间消耗分布及其瓶颈所在。我们将回顾第 10 章中涵盖的指标，重点关注它们的使用方法和时机。各种 SQL 性能监控工具的主体内容将在下一章介绍。

### 用于 Exadata SQL 统计的监控视图

针对单个 SQL 语句的大多数 Exadata 特定性能统计信息，主要可通过以下视图进行监控：
*   `V$SQL` 和 `V$SQLAREA`
*   `V$SQLSTATS` 和 `V$SQLSTATS_PLAN_HASH`
*   `V$SQL_MONITOR` 和 `V$SQL_PLAN_MONITOR`
*   `V$ACTIVE_SESSION_HISTORY` 以及持久化到 AWR 存储库的 `DBA_HIST_ACTIVE_SESS_HISTORY`

访问与 AWR 相关的视图需要您获得相应的许可。请注意，您在 `V$SQL%` 视图中看到的所有 Exadata 特定指标，实际上与您从 `V$SESSTAT` 视图中看到的指标相同。它们源于相同的数据源，只是累积方式不同。`V$SESSTAT` 为会话累积统计信息，而不论哪个 SQL 语句或命令使其递增；而带有 `V$SQL` 前缀的视图则聚合不同 SQL 语句的统计信息，而不论执行它们的会话。因此，可以看到某些 Exadata 指标按 SQL 语句进行了聚合。

### 示例：V$SQL 视图列

以下是 `V$SQL%` 视图列的示例。（`V$SQL` 视图中还有更多列；这里展示的是在智能扫描上下文中重要的那些。）

```
SQL> desc v$sql

Name                                      Null?    Type
----------------------------------------- -------- ----------------------------
SQL_TEXT                                           VARCHAR2(1000)
SQL_FULLTEXT                                       CLOB
SQL_ID                                             VARCHAR2(13)
SHARABLE_MEM                                       NUMBER
PERSISTENT_MEM                                     NUMBER
RUNTIME_MEM                                        NUMBER
[...]
IO_CELL_OFFLOAD_ELIGIBLE_BYTES                     NUMBER
IO_INTERCONNECT_BYTES                              NUMBER
PHYSICAL_READ_REQUESTS                             NUMBER
PHYSICAL_READ_BYTES                                NUMBER
PHYSICAL_WRITE_REQUESTS                            NUMBER
PHYSICAL_WRITE_BYTES                               NUMBER
OPTIMIZED_PHY_READ_REQUESTS                        NUMBER
LOCKED_TOTAL                                       NUMBER
PINNED_TOTAL                                       NUMBER
IO_CELL_UNCOMPRESSED_BYTES                         NUMBER
IO_CELL_OFFLOAD_RETURNED_BYTES                     NUMBER
CON_ID                                             NUMBER
IS_REOPTIMIZABLE                                   VARCHAR2(1)
IS_RESOLVED_ADAPTIVE_PLAN                          VARCHAR2(1)
IM_SCANS                                           NUMBER
IM_SCAN_BYTES_UNCOMPRESSED                         NUMBER
IM_SCAN_BYTES_INMEMORY                             NUMBER
```

以粗体显示的列是针对 Exadata 处理的特定列，但并非 Exadata 独有。如果您在非 Exadata 环境中描述 `V$SQL`，您将获得完全相同的列。

### 关键列及其含义

表 11-1 列出了最值得关注的列，为了可读性，明确排除了部分物理读列。

**表 11-1. V$SQL 列及其含义**

| 列名 | 指标含义 |
| --- | --- |
| `IO_CELL_OFFLOAD_ELIGIBLE_BYTES` | 有多少字节的段读取操作被卸载到存储节点。存储节点要么读取了这些数据，要么在存储索引帮助下跳过了块范围。该指标对应于 `V$SESSTAT` 中的 cell physical IO bytes eligible for predicate offload 统计信息。 |
| `IO_INTERCONNECT_BYTES` | 在数据库节点和存储节点之间发送的总通信量字节数（读和写）。 |
| `OPTIMIZED_PHY_READ_REQUESTS` | 因存储索引而完全避免，或针对存储节点闪存缓存卡完成的磁盘 I/O 请求数量。 |
| `IO_CELL_UNCOMPRESSED_BYTES` | 存储节点在智能扫描期间扫描过的未压缩数据的大小。请注意，存储节点无需实际解压所有数据即可知道未压缩长度。HCC 压缩单元头部在其内部同时存储了压缩和未压缩的 CU 长度信息。此指标对于估算 HCC 压缩带来的 I/O 减少很有用。请注意，此指标仅适用于 HCC 段。对于常规的块级压缩，此指标仅显示数据的压缩后大小。 |
| `IO_CELL_OFFLOAD_RETURNED_BYTES` | 此指标显示作为卸载智能扫描访问路径的结果返回了多少数据。当与 `IO_CELL_OFFLOAD_ELIGIBLE_BYTES` 比较（用于衡量存储节点和数据库之间的 I/O 减少）或与 `IO_CELL_UNCOMPRESSED_BYTES` 比较（用于衡量因卸载和压缩带来的总 I/O 减少）时，这是衡量智能扫描卸载效率的主要指标。 |

### 示例查询分析

以下是一个查询示例的输出，针对一个 EHCC 压缩表，其中表扫描被卸载到存储节点。`V$SQL` 表的输出经过了转置和简化，只显示相关详细信息，以提高可读性：

```
SQL_TEXT                      : select /* hccquery001 */ ...
SQL_ID                        : 5131dsd26qfc5
DISK_READS                    : 1304924
BUFFER_GETS                   : 1304934
IO_CELL_OFFLOAD_ELIGIBLE_BYTES: 10689937408
IO_INTERCONNECT_BYTES         : 84656896
PHYSICAL_READ_REQUESTS        : 10258
PHYSICAL_READ_BYTES           : 10689937408
PHYSICAL_WRITE_REQUESTS       : 0
PHYSICAL_WRITE_BYTES          : 0
OPTIMIZED_PHY_READ_REQUESTS   : 10110
LOCKED_TOTAL                  : 1
PINNED_TOTAL                  : 2
IO_CELL_UNCOMPRESSED_BYTES    : 27254591356
IO_CELL_OFFLOAD_RETURNED_BYTES: 84656896
-----------------
PL/SQL procedure successfully completed.
```

在此案例中，`IO_CELL_OFFLOAD_RETURNED_BYTES` 远小于 `IO_CELL_OFFLOAD_ELIGIBLE_BYTES`；因此，智能扫描无疑有助于减少存储节点和数据库之间的数据流量。后者是衡量数据库中执行了多少智能扫描的良好指标：

```
SQL> !cat sscan.sql
WITH offloaded_yes_no AS
(SELECT inst_id,
CASE
WHEN (IO_CELL_OFFLOAD_ELIGIBLE_BYTES > 0)
THEN 'YES'
ELSE 'NO'
END sscan
FROM gv$sql
)
SELECT COUNT(sscan),
sscan as smart_scan,
inst_id
FROM offloaded_yes_no
GROUP BY sscan,
inst_id;
```

此外，回到前面的例子，`IO_CELL_UNCOMPRESSED_BYTES` 显著大于 `PHYSICAL_READ_BYTES`，这表明 HCC 帮助减少了存储节点必须从磁盘读取的字节数，这要归功于压缩。请注意，`IO_INTERCONNECT_BYTES` 并不比 `IO_CELL_OFFLOAD_RETURNED_BYTES` 大很多，这表明对于此 SQL，几乎所有通信量都是由智能扫描返回的数据引起的。没有由于其他原因（例如非最优排序导致的临时表空间读/写，或由链接行或数据库内一致性读处理引起的哈希连接或其他工作区操作或数据库块 I/O）产生的额外通信量。



智能扫描（Smart Scanning）能加快从数据段中检索数据的速度，但它并不能神奇地加速连接、排序和聚合操作。这些操作发生在数据从数据段被检索出来之后。一个值得注意的例外是布隆过滤器（Bloom filter）下推到单元格（cells），这使得单元格能够利用在哈希连接中基于驱动行源数据构建的哈希位图来过滤来自探测表的数据。消费者可能会拖慢生产者的速度，但这是所有存储系统的普遍规律。

虽然此示例使用了 `V$SQL` 视图（它显示 SQL 子游标级别的统计信息），但你也可以使用 `V$SQL_PLAN_MONITOR`（其中的 `PLAN_LINE_ID`、`PLAN_OPERATION` 等列）来测量每个执行计划行的这些指标。这很有用，因为单个执行计划通常会访问并连接多个表，而不同的表可能从智能扫描下推（Smart Scan offloading）中受益的程度不同。第 12 章 介绍了更多使用这些数据的脚本和工具。

## 查询 cellsrv 内部处理统计信息

在 Exadata 软件的早期版本中，深入了解 Exadata 的内部处理过程并非总是易事。在大多数情况下，性能分析师必须连接到单元格本身，然后将事件转储到跟踪文件中才能获取这些指标。一个更简单的方法是访问 `V$CELL` 系列视图。本节将解释其中一些视图，以及为什么查询它们可以为你提供关于单元格服务器上各种处理步骤的有趣见解——而无需退出 SQL*Plus！`V$CELL` 系列包括这些 Oracle 提供了 API 的视图。如果你查询 `V$FIXED_VIEW_DEFINITION` 中以 `GV%CELL%` 开头的视图，你会发现 `V$CELL` 视图是基于名为 `X$KCFIS%`（内核缓存文件智能存储，Kernel Cache File Intelligent Storage）的 `X$` 表。并非所有这些 `X$` 表都有对应的“官方” `V$` 视图。

本节将详细阐述 `V$CELL%` 视图，以及本书尚未介绍的另一个单元格上的工具：`cellsrvstat`。

### V$CELL 系列视图

`V$CELL%` 视图的数量随着每个版本稳步增加，12c 也不例外。Oracle 12.1.0.2 列出了以下这些：

```
SQL> select table_name from dict where regexp_like(table_name, 'DBA.*(ASM|CELL)|^V\$CELL');

TABLE_NAME
--------------------------------------------------------------------------------
DBA_HIST_ASM_BAD_DISK
DBA_HIST_ASM_DISKGROUP
DBA_HIST_ASM_DISKGROUP_STAT
DBA_HIST_CELL_CONFIG
DBA_HIST_CELL_CONFIG_DETAIL
DBA_HIST_CELL_DB
DBA_HIST_CELL_DISKTYPE
DBA_HIST_CELL_DISK_NAME
DBA_HIST_CELL_DISK_SUMMARY
DBA_HIST_CELL_GLOBAL
DBA_HIST_CELL_GLOBAL_SUMMARY
DBA_HIST_CELL_IOREASON
DBA_HIST_CELL_IOREASON_NAME
DBA_HIST_CELL_METRIC_DESC
DBA_HIST_CELL_NAME
DBA_HIST_CELL_OPEN_ALERTS
V$CELL
V$CELL_CONFIG
V$CELL_CONFIG_INFO
V$CELL_DB
V$CELL_DB_HISTORY
V$CELL_DISK
V$CELL_DISK_HISTORY
V$CELL_GLOBAL
V$CELL_GLOBAL_HISTORY
V$CELL_IOREASON
V$CELL_IOREASON_NAME
V$CELL_METRIC_DESC
V$CELL_OFL_THREAD_HISTORY
V$CELL_OPEN_ALERTS
V$CELL_REQUEST_TOTALS
V$CELL_STATE
V$CELL_THREAD_HISTORY

33 rows selected.
```

相比于 11.2.0.4 版本，数量多了不少：

```
SQL> select table_name from dict where regexp_like(table_name, 'DBA.*CELL|^V\$CELL');

TABLE_NAME
------------------------------
V$CELL
V$CELL_CONFIG
V$CELL_OFL_THREAD_HISTORY
V$CELL_REQUEST_TOTALS
V$CELL_STATE
V$CELL_THREAD_HISTORY

6 rows selected.
```

由于本章已经非常长，因此精心挑选只包含了其中最重要的视图。这些视图的 AWR 版本被省略了，因为它们本质上允许在 `SYSAUX` 表空间上对信息进行长期归档。

#### V$CELL

首先想到的视图是 `V$CELL`。在 Oracle 12.1.0.2 中，该视图的定义如下所示：

```
SQL> desc v$cell

Name                                     Null?    Type
---------------------------------------- -------- ----------------------------
CELL_PATH                                         VARCHAR2(400)
CELL_HASHVAL                                      NUMBER
CON_ID                                            NUMBER
CELL_TYPE                                         VARCHAR2(400)
```

`CON_ID` 和 `CELL_TYPE` 是 12c 中新增的；Oracle 11.2 仅显示单元格路径（一个 IP 地址）和单元格哈希值。你可能会使用此视图将单元格哈希值映射到单元格，如在 第 10 章 讨论的与智能扫描相关的等待事件中所发现的那样。

#### V$CELL_OFL_THREAD_HISTORY

这个有趣的视图记录了 `cellsrv` 线程在过去十分钟内的活动历史，从概念上讲，类似于存储单元格的 ASH。此视图与 `V$CELL_THREAD_HISTORY` 类似，但有额外的列，允许对信息进行类似 ASH 的版本控制。以下是 12.1.0.2 的视图定义：

```
SQL> desc V$CELL_OFL_THREAD_HISTORY

Name                                     Null?    Type
---------------------------------------- -------- ----------------------------
CELL_NAME                                         VARCHAR2(1024)
GROUP_NAME                                        VARCHAR2(1024)
PROCESS_ID                                        NUMBER
SNAPSHOT_ID                                       NUMBER
SNAPSHOT_TIME                                     DATE
THREAD_ID                                         NUMBER
JOB_TYPE                                          VARCHAR2(32)
WAIT_STATE                                        VARCHAR2(32)
WAIT_OBJECT_NAME                                  VARCHAR2(32)
SQL_ID                                            VARCHAR2(13)
DATABASE_ID                                       NUMBER
INSTANCE_ID                                       NUMBER
SESSION_ID                                        NUMBER
SESSION_SERIAL_NUM                                NUMBER
CON_ID                                            NUMBER
```

如你所见，该视图显示了单元格进程，这些进程在内部映射到 `cellsrv` 线程。对于每个线程，你可以看到作业类型，例如在智能扫描期间的 `PredicateOflFilter`，以及一个状态。更有趣的是，你可以看到导致负载的 SQL ID、数据库 ID、实例号以及会话 SID 和序列号。但要小心——如果信息不可用，`SQL_ID` 由 13 个空格组成，而不是 null 或空字符串：

```
SQL> select count(''''||sql_id||''''),''''||sql_id||''''
  2   from v$cell_ofl_thread_history
  3  group by ''''||sql_id||'''';

COUNT(''''||SQL_ID||'''') ''''||SQL_ID||'''
------------------------- ---------------
                       2 'f254uv2p53y7j'
                  126363 '              '

2 rows selected.
```

换句话说，你可以看到在给定时间点是谁导致了多少个工作线程处于忙碌状态。下面是一个如何从系统中获取当前信息的示例：

```
SELECT CELL_NAME,
       GROUP_NAME,
       SNAPSHOT_ID,
       JOB_TYPE,
       WAIT_STATE,
       WAIT_OBJECT_NAME,
       SQL_ID
FROM
       (SELECT CELL_NAME,
               GROUP_NAME,
               SNAPSHOT_ID,
               JOB_TYPE,
               WAIT_STATE,
               WAIT_OBJECT_NAME,
               SQL_ID,
               DATABASE_ID,
               INSTANCE_ID,
               SESSION_ID,
               SESSION_SERIAL_NUM,
               CON_ID,
               MAX (SNAPSHOT_ID) over (partition BY cell_name) max_snap
          FROM V$CELL_OFL_THREAD_HISTORY
       )
WHERE snapshot_id = max_snap;
```

正如视图名称所示，历史信息是可用的，而不仅仅是当前状态的快照。如果你将会话信息和 SQL ID 与 ASH 监控信息关联起来，你应该能够非常准确地描绘出在给定时间点单元格的负载情况。



#### V$CELL_STATE

此视图提供了关于单元的大量信息。与`V$CELL_CONFIG`类似，它依赖于一个描述 XML 字段的列。在本例中，该列是`STATISTICS_TYPE`。12.1.0.2 版本中该视图的定义如下：

```
SQL> desc v$cell_state
Name                                              Null?    Type
------------------------------------------------- -------- ----------------------------
CELL_NAME                                                  VARCHAR2(1024)
STATISTICS_TYPE                                            VARCHAR2(15)
OBJECT_NAME                                                VARCHAR2(1024)
STATISTICS_VALUE                                           CLOB
CON_ID                                                     NUMBER
```

您可以查询的不同统计信息取决于 RDBMS/单元的版本。对于 12.1.0.2 和 Exadata 版本 12.1.2.1.0，您可以查看以下指标：

*   `IOREASON`：将单元的 I/O 分解为各种可想象的类别
*   `RCVPORT`：包含接收到的网络流量详情
*   `FLASHLOG`：关于`FLASHLOG`功能使用的非常详细的信息
*   `SENDPORT`：包含发送的网络流量详情
*   `PREDIO`：包含关于 Exadata 如何处理智能扫描的信息
*   `NPHYSDISKS`：列出每个单元的物理磁盘数量
*   `CELL`：与`cellsrvstat`输出类似的信息
*   `THREAD`：关于`cellsrv`工作线程的线程相关信息
*   `PHASESTAT`：关于智能扫描各个阶段的信息
*   `CAPABILITY`：单元软件功能
*   `LOCK`：分解单元中按对象类型统计的互斥锁等待
*   `OFLGROUP`：卸载服务器统计信息

I/O 原因如此有趣，以至于 Oracle 在 12c 中决定为它们单独提供一个名为`V$CELL_IOREASON`的视图。使用此视图的技巧在于，您必须根据统计类型再次解析输出。它有助于为给定类型仅选择`STATISTICS_VALUE`，并制定如何解析 XML 数据的策略。以下是一个示例，用于列出给定单元在 12.1.0.2 上的所有`IOREASONS`：

```
SELECT x.cell_name, x.statistics_type, x.object_name, stats.*
FROM   V$CELL_STATE x,
       XMLTABLE ('/ioreasongroup_stats'
                 PASSING xmltype(x.STATISTICS_VALUE)
                 COLUMNS ioreasons XMLTYPE PATH '*'
                ) xt,
       xmltable ('/stat'
                 passing xt.ioreasons
                 columns name path './@name',
                         value path '/stat'
                ) stats
where x.statistics_type = 'IOREASON'
  and stats.name <> 'reason'
  and x.cell_name = '192.168.12.8'
/
```

输出（经过缩减）如下：

```
CELL_NAME            STATISTICS_TYPE OBJECT_NAME          NAME            VALUE
-------------------- --------------- -------------------- --------------- ---------------------
192.168.12.8         IOREASON        UNKNOWN              reads           369113
192.168.12.8         IOREASON        UNKNOWN              writes          170592
192.168.12.8         IOREASON        RedoLog Write        reads           0
192.168.12.8         IOREASON        RedoLog Write        writes          51493
192.168.12.8         IOREASON        RedoLog Read         reads           1152
192.168.12.8         IOREASON        RedoLog Read         writes          0
192.168.12.8         IOREASON        ArchLog Read         reads           0
192.168.12.8         IOREASON        ArchLog Read         writes          0
192.168.12.8         IOREASON        MediaRecovery Write  reads           0
192.168.12.8         IOREASON        MediaRecovery Write  writes          0
```

`CELL`统计信息提供了最全面的输出；它产生的信息原本只能从系统状态转储中获得。其中一些信息实际上过于复杂，无法完全解析，此时转储到 XML 文件会有所帮助。只需将`STATISTICS_VALUE`转换为`XMLType`，然后从动态性能视图中选择，将输出粘贴到文本文件中，并用您喜欢的浏览器打开即可。`PREDIO`和`CELL`统计信息就是真正需要这种处理方式的很好例子。



