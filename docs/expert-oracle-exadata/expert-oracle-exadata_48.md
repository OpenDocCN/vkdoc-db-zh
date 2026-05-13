# 系统 I/O 类中的 Exadata 等待事件

分配给系统 I/O 类的 Exadata 等待事件重要性较低，通常不会显示出是主要的时间消耗者。备份事件是最值得关注的，因为它们记录了在 Exadata 平台上已优化代码段的耗时。其他事件则仅是维护性事件。非备份事件列于表 10-1 中，而备份事件将在后续章节中详述。

**表 10-1. 其他系统 I/O 类事件**

| 事件 | 描述 |
| --- | --- |
| `cell manager closing cell` | 这是一个与关机相关的事件。此事件的单元哈希号包含在 `v$session_wait` 视图的 `P1` 列中。`P2` 和 `P3` 列未使用。 |
| `cell manager discovering disks` | 这是一个与启动相关的事件。此事件的单元哈希号包含在 `v$session_wait` 视图的 `P1` 列中。`P2` 和 `P3` 列未使用。 |
| `cell manager opening cell` | 这是一个与启动相关的事件。此事件的单元哈希号包含在 `v$session_wait` 视图的 `P1` 列中。`P2` 和 `P3` 列未使用。 |

## 单元智能增量备份

此事件用于衡量在进行增量 1 级备份时等待 RMAN 的时间。Exadata 通过将大部分处理卸载到存储层来优化增量备份。新增此等待事件是为了计入等待优化的增量备份处理（已卸载到存储单元）所花费的时间。

未经优化的增量备份处理非常消耗资源。备份进程需要捕获自上次增量备份以来数据文件的每一次更改。执行此任务需要大量 I/O，因为必须从头到尾扫描数据文件。为了减少进行 1 级备份所需的时间，Oracle 在 Exadata 之前很久就引入了块更改跟踪。块更改跟踪使用一个在生成重做时更新的小型二进制文件来实现。块更改跟踪文件的引入使增量备份速度提升了数倍。

### 事件含义

有趣的是，增量 0 级备份不会导致此等待事件被写入跟踪文件，尽管 RMAN 命令中包含了“增量”一词。这是因为 0 级备份——尽管名称如此——根本不会执行增量备份。它会生成一个完整备份，并将其标记为未来增量备份的基线。以下是一个 10046 跟踪文件的摘录，该文件生成于创建增量 1 级备份的进程，时间为 11.2.0.4 版本且启用了块更改跟踪：

```
*** 2014-07-06 15:23:11.171
*** SESSION ID:(200.471) 2014-07-06 15:23:11.171
*** CLIENT ID:() 2014-07-06 15:23:11.171
*** SERVICE NAME:(SYS$USERS) 2014-07-06 15:23:11.171
*** MODULE NAME:(backup incr datafile) 2014-07-06 15:23:11.171
*** ACTION NAME:(0000048 STARTED16) 2014-07-06 15:23:11.171
WAIT #0: nam=’enq: TC - contention’ ela= 30442809 name|mode=1413677062 checkpoint ID=65586 0=0
  obj#=-1 tim=1404678191171373
WAIT #0: nam=’enq: CF - contention’ ela= 223 name|mode=1128660997 0=0 operation=0 obj#=-1
  tim=1404678191172055
...
WAIT #0: nam=’change tracking file synchronous read’ ela= 523 block#=9344 blocks=64 p3=0
  obj#=-1 tim=1404678192231084
WAIT #0: nam=’change tracking file synchronous read’ ela= 523 block#=9472 blocks=64 p3=0
  obj#=-1 tim=1404678192231665
WAIT #0: nam=’change tracking file synchronous read’ ela= 501 block#=9920 blocks=64 p3=0
  obj#=-1 tim=1404678192232261
WAIT #0: nam=’ cell smart incremental backup ’ ela= 150 cellhash#=379339958 p2=0 p3=0
  obj#=-1 tim=1404678192234400
WAIT #0: nam=’ cell smart incremental backup ’ ela= 170 cellhash#=2133459483 p2=0 p3=0
  obj#=-1 tim=1404678192235378
WAIT #0: nam=’ cell smart incremental backup ’ ela= 160 cellhash#=3176594409 p2=0 p3=0
  obj#=-1 tim=1404678192237663
WAIT #0: nam=’KSV master wait’ ela= 70 p1=0 p2=0 p3=0 obj#=-1 tim=1404678192243295
WAIT #0: nam=’KSV master wait’ ela= 1438 p1=0 p2=0 p3=0 obj#=-1 tim=1404678192244762
WAIT #0: nam=’ASM file metadata operation’ ela= 57 msgop=33 locn=0 p3=0
  obj#=-1 tim=1404678192244786
WAIT #0: nam=’KSV master wait’ ela= 63 p1=0 p2=0 p3=0 obj#=-1 tim=1404678192251632
WAIT #0: nam=’KSV master wait’ ela= 1340 p1=0 p2=0 p3=0 obj#=-1 tim=1404678192253010
WAIT #0: nam=’ASM file metadata operation’ ela= 62 msgop=33 locn=0 p3=0
  obj#=-1 tim=1404678192253032
WAIT #0: nam=’ cell smart incremental backup ’ ela= 198 cellhash#=379339958 p2=0 p3=0
  obj#=-1 tim=1404678192255205
WAIT #0: nam=’ cell smart incremental backup ’ ela= 214 cellhash#=2133459483 p2=0 p3=0
  obj#=-1 tim=1404678192255495
WAIT #0: nam=’ cell smart incremental backup ’ ela= 212 cellhash#=3176594409 p2=0 p3=0
  obj#=-1 tim=1404678192255802
WAIT #0: nam=’ cell smart incremental backup ’ ela= 10 cellhash#=379339958 p2=0 p3=0
  obj#=-1 tim=1404678192256723
WAIT #0: nam=’ cell smart incremental backup ’ ela= 100 cellhash#=2133459483 p2=0 p3=0
  obj#=-1 tim=1404678192256846
```

您可以在 `change tracking file synchronous read` 中看到块更改跟踪文件的效果。实际的备份工作——扫描自上次备份以来的数据文件更改——是在单元上执行的，这在 `cell smart incremental backup` 事件中可见。您可以在 `v$backup_datafile` 视图中评估卸载到单元的有效性。使用 `blocks_skipped_in_cell` 列并将其与 `blocks_read` 进行关联。

### 参数

此事件使用的唯一参数是 `P1`，它显示哪个单元负责生成此事件：

*   `P1` - 单元哈希号
*   `P2` - 未使用
*   `P3` - 未使用
*   `obj#` - 未使用

**注意**
`obj#` 字段是许多等待事件的一部分，即使是一些与特定对象无关的事件。请注意，在某些情况下，该值可能由某个事件设置，然后在等待结束时未被适当清除，导致无意义的值保留在下一个等待事件中。在前面的示例中，`obj#` 已被清除（设置为值 -1）。

## 单元从备份智能恢复

此事件用于衡量在进行恢复操作时等待 RMAN 的时间。Exadata 通过将处理卸载到存储单元来优化 RMAN 恢复。



### 事件含义

此事件实际上记录的是恢复操作期间与文件初始化相关的时间。以下是从一个恢复进行中时采集的 `10046` 跟踪文件中截取的片段：

```sql
WAIT #0: nam='KSV master wait' ela= 68 p1=0 p2=0 p3=0 obj#=-1 tim=1404740066131406
WAIT #0: nam='RMAN backup & recovery I/O' ela= 3320 count=1 intr=256 timeout=4294967295
  obj#=-1 tim=1404740066135786
WAIT #0: nam='RMAN backup & recovery I/O' ela= 3160 count=1 intr=256 timeout=4294967295
  obj#=-1 tim=1404740066139862
WAIT #0: nam='RMAN backup & recovery I/O' ela= 3201 count=1 intr=256 timeout=4294967295
  obj#=-1 tim=1404740066143790
WAIT #0: nam='RMAN backup & recovery I/O' ela= 2 count=1 intr=256 timeout=2147483647
  obj#=-1 tim=1404740066143822
WAIT #0: nam='cell smart restore from backup' ela= 853 cellhash#=2133459483 p2=0 p3=0
  obj#=-1 tim=1404740066146969
WAIT #0: nam='cell smart restore from backup' ela= 137 cellhash#=379339958 p2=0 p3=0
  obj#=-1 tim=1404740066148003
WAIT #0: nam='cell smart restore from backup' ela= 167 cellhash#=3176594409 p2=0 p3=0
  obj#=-1 tim=1404740066148948
WAIT #0: nam='cell smart restore from backup' ela= 269 cellhash#=2133459483 p2=0 p3=0
  obj#=-1 tim=1404740066149335
WAIT #0: nam='cell smart restore from backup' ela= 261 cellhash#=379339958 p2=0 p3=0
  obj#=-1 tim=1404740066149767
WAIT #0: nam='cell smart restore from backup' ela= 375 cellhash#=3176594409 p2=0 p3=0
  obj#=-1 tim=1404740066150294
WAIT #0: nam='cell smart restore from backup' ela= 124 cellhash#=3176594409 p2=0 p3=0
  obj#=-1 tim=1404740066152236
```

### 参数

此事件唯一使用的参数是 `P1`，它显示了哪个存储 cell 是产生此事件的责任方。
- `P1` - Cell 哈希值
- `P2` - 未使用
- `P3` - 未使用

## Exadata 中的“其他”与“空闲”类等待事件

这些是相对次要的事件，主要在存储 cell 启动、关闭以及发生故障时出现。在正常运行的系统上，您可能看不到它们。但有一个例外，即 `cell smart flash unkeep` 事件。表 10-2 列出了“其他”类中与“cell”相关的等待事件及其参数。`cell smart flash unkeep` 事件将在另一节单独介绍。

**表 10-2：其他与空闲类杂项事件**

| 事件 | 描述 |
| --- | --- |
| `cell manager cancel work request` | 此事件信息量不大，因为 `v$session_wait` 视图中其三个参数 (`P1`, `P2`, `P3`) 均未使用。 |
| `cell worker online completion` | 这似乎是一个启动事件。此事件在 `v$session_wait` 视图中的 `P1` 列包含了 cell 哈希值。`P2` 和 `P3` 列未使用。 |
| `cell worker retry` | 此事件在 `v$session_wait` 视图中的 `P1` 列包含了 cell 哈希值。`P2` 和 `P3` 列未使用。 |
| `cell worker idle` | 在此空闲事件中，`v$session_wait` 视图中的 `P1`、`P2` 和 `P3` 列均未使用。 |

### cell smart flash unkeep

此事件记录了当 Oracle 必须将数据块从 Exadata 智能闪存缓存中刷新出来时所花费的等待时间。当一个表（其存储子句指定它应固定在 Exadata 智能闪存缓存中）被截断 (`TRUNCATE`) 或删除 (`DROP`) 时，就可能发生这种情况。这是一个重要的区别。从 Exadata `11.2.3.3.0` 版本开始，cell 软件会自动缓存来自单块 I/O 以及智能扫描 (`Smart Scans`) 的数据。这使得下一次扫描可以同时从磁盘和闪存读取，从而显著提升性能。在 `11.2.3.3.0` 之前，需要手动将表和分区固定 (`PIN`) 在闪存缓存中。

通过一个例子来理解会更容易。考虑以下用例：数据库 `DB12C` 中的表 `T1` 的数据对象 ID 为 `79208`。在该表创建并对其进行了一些智能扫描形式的轻量级查询活动后，您可以观察到类似以下模式的输出：

```bash
[oracle@enkdb03 ~]$ dcli -g cell_group -l cellmonitor \
> "cellcli -e list flashcachecontent where objectNumber = 79208 detail" | grep cache
enkcel04: cachedKeepSize:        0
enkcel04: cachedSize:            4374528
enkcel04: cachedWriteSize:       4358144
enkcel05: cachedKeepSize:        0
enkcel05: cachedSize:            5521408
enkcel05: cachedWriteSize:       5521408
enkcel06: cachedKeepSize:        0
enkcel06: cachedSize:            5177344
enkcel06: cachedWriteSize:       5177344
```

如您所见，得益于智能扫描，缓存中已经填充了数据——尽管在 `CELL_FLASH_KEEP` 子句中并未指定 `KEEP` 关键字。同样由于此原因，您也可以看到 `KEEP` 大小为 0。



### 事件含义

在当前阶段，截断或删除表对智能闪存缓存没有即时影响。那么，你何时会看到这个事件呢？

回到`CELL_FLASH_CACHE`子句：一旦存储子句被更改为`KEEP`，单元格将开始缓存表中的数据，填充`cachedKeepSize`的值。在上一个例子中，需要再执行几次查询，直到 keep 缓存被填充：

```
[oracle@enkdb03 ~]$ dcli -g cell_group -l cellmonitor \
> "cellcli -e list flashcachecontent where objectNumber = 79208 detail" | grep cache
enkcel04: cachedKeepSize:        1239744512
enkcel04: cachedSize:            1243955200
enkcel04: cachedWriteSize:       5365760
enkcel05: cachedKeepSize:        1406525440
enkcel05: cachedSize:            1410203648
enkcel05: cachedWriteSize:       5980160
enkcel06: cachedKeepSize:        1355415552
enkcel06: cachedSize:            1358880768
enkcel06: cachedWriteSize:       6381568
```

如果你现在截断表，它将显示`cell smart flash unkeep`事件。以下是一个针对`truncate`语句的 tkprof 示例：

```
truncate table t1

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          6          0           0
Execute      1      0.24       0.57         12        480       5667           0
Fetch        0      0.00       0.00          0          0          0           0
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        2      0.24       0.58         12        486       5667           0

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 198

Elapsed times include waiting on following events:
  Event waited on                             Times   Max. Wait  Total Waited
  ----------------------------------------   Waited  ----------  ------------
  enq: IV -  contention                          10        0.00          0.00
  row cache lock                                  4        0.00          0.00
  Disk file operations I/O                        1        0.00          0.00
  enq: RO - fast object reuse                     6        0.00          0.00
  reliable message                                2        0.00          0.00
  log file sync                                   2        0.00          0.00
  cell smart flash unkeep                        18        0.01          0.01
  cell single block physical read                12        0.00          0.00
  local write wait                                3        0.00          0.00
  gc current grant 2-way                         16        0.00          0.00
  DFS lock handle                              1452        0.00          0.30
  gc current grant busy                           7        0.00          0.00
  SQL*Net message to client                       1        0.00          0.00
  SQL*Net message from client                     1        4.00          4.00
********************************************************************************
```

当检查原始跟踪文件时，你会注意到`cell smart flash unkeep`事件之前有少数几个`enq: RO – fast object reuse`事件，这些事件用于标记在删除或截断后清理缓冲区缓存所花费的时间。`cell smart flash unkeep`基本上是该事件的扩展，用于同时清理存储服务器上的 Exadata 智能闪存缓存。

### 参数

此事件使用的唯一参数是`P1`，它显示哪个单元格负责生成该事件：

*   `P1` - 单元格哈希值
*   `P2` - 未使用
*   `P3` - 未使用

## 非 Exadata 特定事件

除了新的单元格事件外，还有一些非 Exadata 特定的等待事件你也应该了解。这些事件可能在你管理其他平台上的 Oracle 时已经熟悉。它们恰好在 Exadata 环境中也很重要，因此，当你开始管理 Exadata 时，你现有的知识和技能可以派上用场并让你处于有利地位。

### direct path read

Oracle 使用`Direct path reads`将数据直接读入 PGA 内存（而不是缓冲区缓存）。它们是 Exadata 卸载的一个组成部分，因为只有使用`direct path read`机制时，SQL 处理才能被卸载到存储单元。当查询被卸载时，`direct path read`等待事件实际上被`cell smart table scan`和`cell smart index scan`等待事件所取代。然而，`direct path read`机制仍然被这些新等待事件所涵盖的代码使用。也就是说，执行计划必须包含并行（表）扫描，或者 Oracle 必须决定使用串行`direct path read`机制，如第 2 章中所述。

### 事件含义

此事件记录 Oracle 等待直接路径读取完成所花费的时间。你应该知道`direct path read`等待事件可能非常具有误导性。与智能扫描事件类似，记录的事件数量以及与之相关的时间看起来可能不准确。这是由于直接路径读取是以异步和重叠的方式完成的。本质上，由于 Oracle 发起 I/O 请求的异步/重叠性质，此事件不是 I/O 延迟等待，而经典的同步 I/O 事件则是。还值得一提的是，从 11gR2 开始，Oracle 包含一项增强功能，导致串行直接路径读取比之前的版本更频繁地发生。参见 MOS Note 793845.1，其中简要提到了这一变化。许多 DBA 一定想知道为什么在数据库从 10.2 迁移到 11.2 和 12c 之后，I/O 配置文件发生了变化。

尽管不像在其他平台上那么相关，但`direct path read`等待事件仍然会在 Exadata 平台上因各种操作而出现，但通常不会用于全表扫描，除非表（或分区）相对较小。你可能看到`direct path read`的另一种情况是，当对相关段的扫描不可卸载时，例如对于索引组织表。一个 10046 跟踪和直接路径读取的例子在 cell smart table scan 部分的描述中展示。

### 参数

此事件的参数向你准确显示了扫描的是哪个段（obj）以及在此事件期间扫描了哪些块：

*   `P1` - 文件号
*   `P2` - 第一个数据库块地址
*   `P3` - 块计数
*   `obj#` - 被扫描表的对象号

如`cell smart table scan`部分所述，参数包含有关正在访问的文件和对象的更具体信息。文件中的偏移量也在`P2`参数中提供，`P3`参数中提供了连续读取的块数。

### Enq: KO—fast object checkpoint

`enq:KO`事件有一个奇怪的名字。不要被它吓退。该事件本质上是一个`object checkpoint`事件。`V$LOCK_TYPE`视图将 KO 锁描述如下：

```
SQL> select type, name, description from v$lock_type
  2  where type = 'KO';

TYPE  NAME                       DESCRIPTION
----- ------------------------------ ---------------------------------------------
KO    Multiple Object Checkpoint     Coordinates checkpointing of multiple objects
```


