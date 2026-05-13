# Oracle Exadata 统计信息

以下是关于存储单元（Cell）的统计信息列表。

## 智能 I/O 与性能相关统计

这些统计信息主要属于 `SQLT` 和 `CACHE` 类别，记录了智能扫描、I/O 卸载及闪存缓存相关的活动。

*   `cell num smart IO sessions in rdbms block IO due to no cell mem` (SQLT) - 因无单元内存导致的 RDBMS 块 I/O 智能 I/O 会话数
*   `cell num smart IO sessions in rdbms block IO due to open fail` (SQLT) - 因打开失败导致的 RDBMS 块 I/O 智能 I/O 会话数
*   `cell num smart IO sessions in rdbms block IO due to user` (SQLT) - 因用户原因导致的 RDBMS 块 I/O 智能 I/O 会话数
*   `cell num smart IO sessions using passthru mode due to cellsrv` (SQLT) - 因 cell 服务导致的使用直通模式的智能 I/O 会话数
*   `cell num smart IO sessions using passthru mode due to timezone` (SQLT) - 因时区导致的使用直通模式的智能 I/O 会话数
*   `cell num smart IO sessions using passthru mode due to user` (SQLT) - 因用户原因导致的使用直通模式的智能 I/O 会话数
*   `cell num smart file creation sessions using rdbms block IO mode` (SQLT) - 使用 RDBMS 块 I/O 模式的智能文件创建会话数
*   `cell num smartio automem buffer allocation attempts` (SQLT) - 智能 I/O 自动内存缓冲区分配尝试次数
*   `cell num smartio automem buffer allocation failures` (SQLT) - 智能 I/O 自动内存缓冲区分配失败次数
*   `cell num smartio permanent cell failures` (SQLT) - 智能 I/O 永久性单元故障数
*   `cell num smartio transient cell failures` (SQLT) - 智能 I/O 临时性单元故障数
*   `cell overwrites in flash cache` (CACHE) - 闪存缓存中的单元覆盖写次数
*   `cell partial writes in flash cache` (CACHE) - 闪存缓存中的单元部分写次数
*   `cell physical IO bytes eligible for predicate offload` (SQLT) - 适合进行谓词卸载的物理 I/O 字节数
*   `cell physical IO bytes saved by columnar cache` (CACHE) - 列式缓存节省的物理 I/O 字节数
*   `cell physical IO bytes saved by storage index` (CACHE) - 存储索引节省的物理 I/O 字节数
*   `cell physical IO bytes saved during optimized RMAN file restore` (SQLT) - 优化的 RMAN 文件恢复过程中节省的物理 I/O 字节数
*   `cell physical IO bytes saved during optimized file creation` (SQLT) - 优化的文件创建过程中节省的物理 I/O 字节数
*   `cell physical IO bytes sent directly to DB node to balance CPU` (SQLT) - 为平衡 CPU 直接发送到数据库节点的物理 I/O 字节数
*   `cell physical IO interconnect bytes` (SQLT) - 物理 I/O 互连字节数
*   `cell physical IO interconnect bytes returned by smart scan` (SQLT) - 智能扫描返回的物理 I/O 互连字节数
*   `cell physical write IO bytes eligible for offload` (USER) - 适合进行卸载的物理写 I/O 字节数
*   `cell physical write IO host network bytes written during offloa` (USER) - 卸载期间写入的主机网络物理写 I/O 字节数
*   `cell physical write bytes saved by smart file initialization` (CACHE) - 智能文件初始化节省的物理写字节数
*   `cell scans` (SQLT) - 单元扫描次数
*   `cell simulated physical IO bytes eligible for predicate offload` (SQLT DEBUG) - 适合进行谓词卸载的模拟物理 I/O 字节数
*   `cell simulated physical IO bytes returned by predicate offload` (SQLT DEBUG) - 谓词卸载返回的模拟物理 I/O 字节数
*   `cell smart IO session cache hard misses` (SQLT) - 智能 I/O 会话缓存硬未命中次数
*   `cell smart IO session cache hits` (SQLT) - 智能 I/O 会话缓存命中次数
*   `cell smart IO session cache hwm` (SQLT) - 智能 I/O 会话缓存高水位线
*   `cell smart IO session cache lookups` (SQLT) - 智能 I/O 会话缓存查找次数
*   `cell smart IO session cache soft misses` (SQLT) - 智能 I/O 会话缓存软未命中次数
*   `cell statistics spare1` (SQLT) - 单元统计信息备用 1
*   `cell statistics spare2` (SQLT) - 单元统计信息备用 2
*   `cell statistics spare3` (SQLT) - 单元统计信息备用 3
*   `cell statistics spare4` (SQLT) - 单元统计信息备用 4
*   `cell statistics spare5` (SQLT) - 单元统计信息备用 5
*   `cell statistics spare6` (SQLT) - 单元统计信息备用 6
*   `cell transactions found in commit cache` (SQLT) - 在提交缓存中找到的事务数
*   `cell writes to flash cache` (CACHE) - 写入闪存缓存的次数

## 单元处理与行统计

这些统计信息记录了存储单元对行数据的处理情况，主要属于 `SQLT` 类别。

*   `chained rows processed by cell` (SQLT) - 单元处理的链式行数
*   `chained rows rejected by cell` (SQLT) - 单元拒绝的链式行数
*   `chained rows skipped by cell` (SQLT) - 单元跳过的链式行数
*   `error count cleared by cell` (SQLT) - 单元清除的错误计数
*   `sage send block by cell` (SQLT) - 单元发送的消息块数

```
73 rows selected.
```

## 版本差异与 EHCC 统计

如果你拥有本书的第一版，你会注意到在 12.1.0.2 版本中，与单元相关的计数器比第一版编写时的标准生产版本 11.2.0.2 要多得多。使用类似于上面所示的查询，可以重点关注与 HCC（混合列压缩）功能相关的统计信息，该功能在 第 3 章 中有介绍：

```sql
NAME                                                 CLASS_NAME
-------------------------------------------------- ---------------
EHCC Analyze CUs Decompressed                       DEBUG
EHCC Analyzer Calls                                 DEBUG
EHCC Archive CUs Compressed                         DEBUG
EHCC Archive CUs Decompressed                       DEBUG
EHCC Attempted Block Compressions                   DEBUG
EHCC Block Compressions                             DEBUG
EHCC CU Row Pieces Compressed                       DEBUG
EHCC CUs Compressed                                 DEBUG
EHCC CUs Decompressed                               DEBUG
EHCC CUs all rows pass minmax                       DEBUG
EHCC CUs no rows pass minmax                        DEBUG
EHCC CUs some rows pass minmax                      DEBUG
EHCC Check CUs Decompressed                         DEBUG
EHCC Columns Decompressed                           DEBUG
EHCC Compressed Length Compressed                   DEBUG
EHCC Compressed Length Decompressed                 DEBUG
EHCC Conventional DMLs                              DEBUG
EHCC DML CUs Decompressed                           DEBUG
EHCC Decompressed Length Compressed                 DEBUG
EHCC Decompressed Length Decompressed               DEBUG
EHCC Dump CUs Decompressed                          DEBUG
EHCC Normal Scan CUs Decompressed                   DEBUG
EHCC Pieces Buffered for Decompression              DEBUG
EHCC Preds all rows pass minmax                     DEBUG
EHCC Preds no rows pass minmax                      DEBUG
EHCC Preds some rows pass minmax                    DEBUG
EHCC Query High CUs Compressed                      DEBUG
EHCC Query High CUs Decompressed                    DEBUG
EHCC Query Low CUs Compressed                       DEBUG
EHCC Query Low CUs Decompressed                     DEBUG
EHCC Rowid CUs Decompressed                         DEBUG
EHCC Rows Compressed                                DEBUG
EHCC Rows Not Compressed                            DEBUG
EHCC Total Columns for Decompression                DEBUG
EHCC Total Pieces for Decompression                 DEBUG
EHCC Total Rows for Decompression                   DEBUG
EHCC Turbo Scan CUs Decompressed                    DEBUG
EHCC Used on Pillar Tablespace                      DEBUG
EHCC Used on ZFS Tablespace                         DEBUG
```

```
39 rows selected.
```

## 统计信息说明

所有以 `cell%` 开头的统计信息，顾名思义，都与存储单元相关。其中一些统计信息由单元自身测量和维护，然后在通过 `iDB` 协议进行任何交互时发送回数据库会话。另一些则维护在 `Oracle` 数据库内核的 `Exadata` 特定部分中。名称中带有“`XT`”的统计信息与另一个产品相关，不在本章讨论范围内。

每个数据库会话在从其对应的单元会话收到回复时，也会接收到单元统计信息，然后使用这些信息更新相关的数据库 `V$-视图`。这就是 `Oracle` 数据库层能够洞察单元这个“黑匣子”内部情况的方式，例如实际完成的 I/O 操作次数和单元闪存缓存的命中次数。

请注意，有一些 `chained rows [...] cell statistics`，它们显然使用了不同的命名约定，将“`cell`”放在了统计信息名称的末尾。



以 EHCC 开头的统计数据与混合列压缩相关。在对 HCC 段执行智能扫描时，你总会看到这些值在增加。在智能扫描过程中，当工作线程快速处理磁盘上的压缩数据时，存储服务器上的 `cell CU%` 计数器会递增。每当智能扫描发现与查询谓词匹配的数据时，存储服务器会解压 CU 中的列（注意：不是整个压缩单元！）并将其传递给 RDBMS 层。正是在 RDBMS 层处理过程中，即使数据已经被解压，EHCC 计数器也会增加。遗憾的是，本章无法逐一讨论每个 `EHCC%` 计数器，但在这里对它们进行一个快速的分类是合适的。

### 选定子集的性能计数器参考

本节解释前面输出中列出的一些更重要、更有趣的统计信息。由于本书第一版中本章内容已经太多，我们必须谨慎选择，以减少所描述计数器的数量，为值得涵盖的新特性留出空间。我们希望所做的选择是合理的。

在许多性能故障排除场景中，你可能无需钻研得如此深入。等待接口和实时 SQL 监控功能（你将在第 12 章中读到）应该能提供足够的信息。然而，理解幕后发生的事情以及这些统计信息为何会增加，将使你更深入地了解 Exadata 内部机制，并使你能够更有效地排查异常的性能问题。值得注意的统计信息将按字母顺序介绍。

#### cell CUs sent compressed

这里要介绍的第一个统计计数器在存储服务器上递增。如果你看到这个数字上升，说明你正在目睹存储服务器上的内存压力。这不是最严重的内存压力表现形式（见下一节），但由于内存短缺，存储服务器在扫描 HCC 压缩数据时通常执行的一些工作无法在存储服务器上完成。过滤操作仍在进行，谓词仍在被评估。只是列信息未在存储服务器上解压。这里，第 3 章中阐述的规则再次产生巨大影响：使用 HCC 时，你应该只引用你打算使用的列，而不是无处不在的 `select * from table...` 语句。发送到 RDBMS 层的未解压列越多，你的会话需要执行的工作就越多。

#### cell CUs sent head piece

如果存储服务器面临更大的内存压力且内存分配失败，存储服务器可能无法解压任何内容，整个 CU 必须被发送到 RDBMS 层进行处理。这可能是压缩数据智能扫描期间最糟糕的情况。

如果你想查看存储服务器上的内存统计信息，可以使用 `cellsrvstat` 工具（本章稍后讨论）来检查名为 `mem` 的统计信息组。本章末尾会解释 `cellsrvstat`；可以使用 `cellsrvstat -stat_group=mem` 命令查询与内存相关的统计信息。

#### cell CUs sent uncompressed

在经历了前面那些意味着问题的计数器之后，这个计数器是你希望在智能扫描期间看到的。这等同于 `ORA-00000: normal, successful completion.`。如果你看到这个计数器增加，说明 CU 已在存储服务器上处理并解压。同样，你只会在存储服务器解压 CU 的智能扫描期间看到这个统计计数器增加。请记住，CU 跨越多个 Oracle 数据块。在检查相关统计信息时，你可能需要先检查 CU 的大小。

#### cell blocks helped by commit cache

在存储服务器中进行智能扫描时，正常的数据一致性规则仍然需要应用，有时需要借助 Undo 数据。需要记住的一个重要概念是，智能扫描产生的是**一致读**。因此，一致读保证也必须对智能扫描有效。这里只有一个困难：智能扫描完全在存储服务器中工作，它无法访问数据库实例缓冲区高速缓存中的任何 Undo 数据。当需要撤消对数据块的更改并使其回退到可以安全读取的 SCN 时（你稍后会看到），可能就需要使用 Undo。

请记住，根据设计，存储服务器在智能扫描期间不与其他存储服务器通信。单个存储服务器无法从跨其他存储服务器条带化存放的 Undo 段中读取 Undo 数据。当需要时，一致读缓冲区克隆和回滚必须在数据库层内部完成。每当智能扫描碰到一行数据，其锁字节仍然被设置（由于稍后解释的原因，该行/块未被清理），它就必须针对该特定块切换到块 I/O 模式，并将整个数据块发送回数据库层，以便借助那里可用的 Undo 数据进行常规的一致读处理——。锁字节是在块中的行级别设置的，如下列块转储摘录所示：

```
tab 0, row 0, @0x1b75
tl: 1035 fb: --H-FL-- lb: 0x2  cc: 6
col  0: [ 2]  c1 0b
col  1: [999]
31 20 20 20 20...
```

这就是 Oracle 实现行级锁的方式。锁字节 (`lb`) 指向块的感兴趣事务列表中的一个条目。在此示例中，它是第二个条目：

```
Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.01d2ab27
0x02   0x000a.001.0000920f  0x00009f97.26c9.45  --U-    6  fsc 0x0000.01d2b741
0x03   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
```

无需过多细节，第二个 ITL 条目显示了一个提交（Flag = U），由`Xid`列指示的事务总共影响了六行，等于块中的总行数（块转储中 `nrows=6`，此处未显示）。它还指向撤消记录地址（Uba），这是从该块中撤消该更改所必需的，但存储服务器无法访问它。

请注意，当一个块被正确清理（换句话说，锁字节被清除）且其块头中的清理 SCN 早于查询开始时间 SCN（快照 SCN）时，存储服务器就知道不需要对该块进行回滚。块转储中清理 SCN 名为`csc`，表示最后一次对该块进行适当清理的时间：

```
Block header dump:  0x017c21c3
Object id on Block? Y
seg/obj: 0x12075  csc: 0x00.1d2ab27  itc: 3  flg: E  typ: 1 - DATA
brn: 0  bdba: 0x17c21c0 ver: 0x01 opc: 0
inc: 0  exflg: 0
```

如果对该块的最新更改发生在查询开始之前，那么存储服务器中的块映像是有效的，足以满足给定 SCN 的查询。存储服务器如何知道在数据库层执行的查询的起始 SCN？这是执行计划中存储感知行源的任务，它们在为自己设置智能扫描会话时，通过 iDB 将 SCN 传达给存储服务器。

现在，当块中的某些行确实设置了锁字节，或者块头中的清理 SCN 恰好高于查询的快照 SCN 时，存储服务器无法自行确定块/数据版本的有效性，需要将该块发送回数据库层进行常规的、非智能扫描的处理。如果需要为许多锁定的行和未清理的块进行此类检查，将会大大减慢智能扫描处理速度。然而，有一个优化措施有助于减少存储服务器必须回退到数据库层块 I/O 处理的次数。



每当 `Smart Scan`（智能扫描）在段扫描期间找到被锁定的行时，它都会检查是哪个事务锁定了该行。这可以通过读取由锁字节指向的当前数据块头中该事务的 `ITL` 条目轻松实现。请注意，位图索引段块和 `HCC` 压缩块不会为块中的每一行都设置一个锁字节，但原理是相同的：Oracle 能够从当前数据块本身找出被锁定行的事务 ID。

如果锁定的事务尚未提交，`Smart Scan` 就会为该块回退到块 `I/O` 模式，数据库层将不得不通过正常的一致性读取缓冲区克隆/回滚机制进行处理，对此没有变通方法。正如你在后续示例中所见，在首次对包含未提交事务的表执行查询时，回退到块模式可能会产生显著的性能影响。后续查询将不再需要通过物理 `I/O` 从磁盘读取块，而是可以依赖缓冲区缓存中已有的块。尽管如此，一致性读取处理仍然需要将块回滚。

如果事务已经提交，但在某些块中未清理锁字节（这通常发生在大更新之后，称为延迟块清除），`Smart Scan` 则不必回退到块 `I/O` 和数据库内的一致性读取机制。它知道这行实际上已不再被锁定，因为锁定事务已经提交，即使锁字节仍然存在。

## 提交缓存

单元如何知道给定事务是否已在 `RDBMS` 层提交？这是通过在所谓的提交缓存中缓存最近提交的事务数量来实现的。如果没有提交缓存，当 `Smart Scan` 扫描大量锁字节仍被设置的行时，性能可能会受到影响。你肯定不希望 `Smart Scan` 每次遇到另一个锁定行时，都将块发送到数据库层进行一致性读取处理。提交缓存可能只是一个内存中的哈希表，按事务 ID 组织，它跟踪哪些事务已提交，哪些未提交。当单元 `Smart Scan` 遇到锁定行时，它将提取事务 ID（来自数据块的 `ITL` 部分），并检查提交缓存中是否有该事务的信息。此检查将使统计信息 `cell commit cache queries`（单元提交缓存查询）增加 1。如果提交缓存中没有该事务，那么 `Smart Scan` 就不走运了，必须回退到相对较慢的单块 `I/O` 处理来执行一致性读取处理。另一方面，缓存命中则会使统计信息 `cell blocks helped by commit cache`（提交缓存帮助的单元块）增加 1。

## 测试演示

为了展示提交缓存对查询的影响，我们进行了一次小测试。为确保测试结果可复现，我们不得不稍微“强制”一下——这种情况是夸大的，你在实际生产环境中不太可能看到类似效果。（谁会不带 `where` 子句更新一张非常大的表呢？）首先，创建了一张足够大的表，其大小保证了查询时会触发 `Smart Scan`。接着，在会话 1 中执行了一次更新，修改了该表的所有块以确保延迟块清除，然后执行命令将缓冲区缓存刷新到磁盘。被强制写入磁盘的块并未完全“清理”，原因将在接下来的几段中解释。在此只需知道这些块是“脏”的，需要处理。此外，修改块的活跃事务尚未提交。

如果此时（首次）查询该表，查询耗时将会相当长。在示例中，大约花了 25 秒：
```sql
SQL> select count(*) from t;

COUNT(*)
----------
1000000

Elapsed: 00:00:25.54
```
原因很快被确定为单块读取。该表在创建时，存储子句被设置为明确不在智能闪存缓存中缓存块，以确保读取时间一致。在当前生产系统中，闪存缓存很可能满足单块读取。以下是该查询的一些重要统计信息：
```
Statistic Name                                                         Value
----------------------------------------------------------------  ----------------
CPU used by this session                                                      22
active txn count during cleanout                                          166,672
cell blocks helped by minscn optimization                                      4
cell blocks processed by cache layer                                      167,961
cell blocks processed by data layer                                            4
cell blocks processed by txn layer                                             4
cell commit cache queries                                               167,957
cell physical IO bytes eligible for predicate offload                  1,365,409,792
cell physical IO interconnect bytes                                     1,528,596,408
cell physical IO interconnect bytes returned by smart scan              1,365,854,136
cell scans                                                                     1
cleanouts and rollbacks - consistent read gets                            166,672
consistent gets                                                          2,044,459
consistent gets direct                                                     166,676
data blocks consistent reads - undo records applied                      1,710,729
physical read total IO requests                                            22,459
physical read total multi block requests                                    1,838
```
这些数据表明涉及大量的一致性读取处理。多块读取的次数也相当低。在 22,459 次 `I/O` 请求中，只有 1,838 次是多块读取——其余的则是单块 `I/O`。你还可以看到，缓存层为准备 `Smart Scan` 处理打开了 167,961 个块，但在除了四个情况外的所有情况下都不得不放弃了处理。（这四个块被 `minscn` 优化“帮助”了，你可以在下一节中了解更多相关信息。）还需注意，`Smart Scan` 返回的字节数几乎与有资格进行谓词下推的字节数相同。换句话说，尽管使用了 `Smart Scan` 访问数据，但在 `I/O` 方面完全没有节省（统计信息 “cell scans” 为 1）。

后续执行同一语句时，将不再需要执行单块 `I/O` 从磁盘读取“脏”块，而是可以利用缓冲区缓存中已有的块。第二次查询该表的执行时间降至 2.96 秒。仍然需要相同的一致性读取处理，并且仍然只有四个块通过 `Smart Scan` 处理。然而，由于块已被读入缓冲区缓存，至少可以跳过物理 `I/O`。记录的 “`CPU used for this session`” 从 499 降至 277，并且只能看到很少的单块 `I/O` 请求。块 “`cleanouts and rollbacks - consistent read gets`” 的数量没有显著变化，这在意料之中，因为事务尚未提交。

当会话 1 中的用户提交后，针对 Exadata 上该表的查询情况有所改善。非 Exadata 平台在此处不会看到好处。随后首次执行该表的查询表现出以下特征：
```sql
SQL> select count(*) from t;

COUNT(*)
----------
1000000

Elapsed: 00:00:01.52
```
最相关的统计信息如下所示：
```
Statistic Name                                                         Value
----------------------------------------------------------------  ----------------
```



`----------------------------------------------------------------  ----------------`

```
CPU used by this session                                                 22
cell blocks helped by commit cache                                  166,672
cell blocks helped by minscn optimization                                4
cell blocks processed by cache layer                                166,676
cell blocks processed by data layer                                 166,676
cell blocks processed by txn layer                                  166,676
cell commit cache queries                                           166,672
cell physical IO bytes eligible for predicate offload          1,365,409,792
cell physical IO interconnect bytes                                26,907,936
cell physical IO interconnect bytes returned by smart scan       26,907,936
cell scans                                                                1
cell transactions found in commit cache                             166,672
consistent gets                                                      167,058
consistent gets direct                                              166,676
physical read total IO requests                                       1,308
physical read total multi block requests                              1,308
```

### 统计信息解读

与该查询前两次的执行情况相比，您可以看到所有数据块都触发了智能扫描——这要归功于提交缓存（`166,672`）以及受益于 `minscn` 优化的四个数据块。CPU 使用率从之前的 `499` 下降到 `22`，且所有一致性读取均处于直接模式。所有 I/O 请求都是多块读取，并且使用智能扫描带来了显著的节省：在大约 `1.3` GB 符合谓词下推条件的数据中，只有 `26.9` MB 被返回到 RDBMS 层。

总而言之，当您的会话执行智能扫描时，**cell blocks helped by commit cache** 统计值的增加表明，存储单元（cell）因检查锁定行是否仍在提交缓存中保持锁定状态而产生了一些开销。但是，与此同时，您不会看到性能下降。如果没有此优化，整个智能扫描将会变慢，因为它必须与数据库层进行交互。您还会看到数据库层执行更多的逻辑 I/O（请参阅以 consistent gets from cache 开头的统计信息），而不是仅为读取区段位置和高水位标记下的数据块数量而在每个区段上仅执行一次逻辑 I/O（通过读取区段头）。

### 关于延迟块清理（Block Cleanout）的说明

实际上，在数据库层以及非 Exadata 数据库中也存在类似的优化。Oracle 可以将已提交的事务信息缓存在数据库会话的私有内存中，这样在遇到许多锁定行时，就不必对回滚段头执行缓冲区获取。每当会话将已提交事务的状态缓存在内存中时，它会增加其 `V$SESSTAT` 数组中 **Commit SCN cached** 统计值。每当它从该缓存中进行查找时，它会增加 **Cached Commit SCN referenced** 统计值。

如果您不熟悉 Oracle 中的延迟块清理机制，您可能想知道为什么 Oracle 数据块在事务已经提交后，其行的锁字节仍然被置位。这就是 Oracle 与大多数其他主流商业 RDBMS 产品的不同之处。Oracle 不必将包含未提交行的数据块保留在内存中；数据库写入器（`DBWR`）可以自由地将它们写入磁盘，从而释放内存用于其他数据块。现在，如果在提交事务时，仅仅为了清理锁字节而将所有事务从磁盘读回来，效率会非常低。如果有许多这样的数据块，您的提交时间将增加到无法接受的水平。相反，Oracle 仅仅在其回滚段头槽中将事务标记为完成。任何未来读取该数据块的会话只需检查该回滚段头槽中的事务是否仍然活动。如果您稍后通过块 I/O 将数据块读入数据库层，读取会话将清理该数据块（清除由已提交事务修改的行的锁字节），这样在未来的读取中就不需要进一步检查事务状态。这就是为什么在某些查询中您可以看到生成了重做日志（redo）。

然而，存储单元不执行块清理——单元自身不修改数据块，因为数据库块的修改需要写入重做操作。但是，单元如何写入由数据库层管理并在多个单元之间条带化的重做日志文件呢？在非 Exadata 平台上，块清理和直接路径读取也有一个有趣的副作用，因为直接路径读取不利用缓冲区缓存。

请注意，对于只修改少量数据块且大部分被修改的数据块仍在缓冲区缓存中的小型事务，Oracle 可以在提交时立即执行块清理。此外，刚才讨论的问题不适用于使用直接路径加载插入加载表的数据库（通常是数据仓库）（并且索引分区在表分区加载后构建），因为在直接路径加载的情况下，新格式化的表块中的表行不会被锁定。如果索引是在数据加载之后创建的，那么叶块中的索引条目也是如此。

#### cell blocks helped by minscn optimization

Exadata 单元服务器还有一项旨在进一步提高一致性读取效率的优化。它被称为最小活动 SCN 优化（Minimum Active SCN optimization），它会跟踪数据库中任何仍处于活动（未提交）事务的最低 `SCN`。这使得 Oracle 可以轻松地将锁定事务的 `ITL` 条目中的 `SCN` 与数据库中最“老”的活动事务的最低 `SCN` 进行比较。

当 Oracle 数据库在启动智能扫描会话时能够将此 `MinSCN` 信息发送给单元，那么只要传递给单元的已知最小活动 `SCN` 高于块中事务 `ITL` 条目的 `SCN`，单元就可以避免与数据库层交换数据。每当智能扫描处理一个数据块并在 `ITL` 槽中发现一个带有活动事务的锁定行时，它就可以断定该事务一定已经提交。多亏了数据库会话传递的 `MinSCN`，**cell blocks helped by minscn optimization** 统计值就会增加（每个数据块增加一次）。

如果没有此优化，Oracle 将不得不检查提交缓存（在 **cell blocks helped by commit cache** 统计信息章节中描述）。如果它在提交缓存中找不到关于此事务的信息，它将与数据库层交互以查明锁定事务是否已经提交。此优化是 `RAC` 感知的；实际上，最小 `SCN` 被称为全局最小 `SCN`（Global Minimum `SCN`），每个实例中的 `MMON` 进程将跟踪 `MinSCN` 并在每个节点的 `SGA` 的内存结构中保持其同步。您可以从 `x$ktumascn` 固定表中查询当前已知的全局最小活动 `SCN`，如下所示（以 `SYS` 身份执行）：

```
SQL> COL min_act_scn FOR 99999999999999999
SQL>
SQL> SELECT min_act_scn FROM x$ktumascn;

       MIN_ACT_SCN
------------------
     9920890881859
```

这个 **cell blocks helped by minscn optimization** 统计值也是您不必担心的，但在对高级智能扫描问题甚至错误进行故障排除时，它可能会派上用场，特别是当智能扫描似乎因为必须回退到块 I/O 并与数据库进行过多交互而中断时。


## 单元格缓存层处理

由……层处理的单元格块统计数据是衡量单元格中卸载过程深度的良好指标。Exadata 存储服务器的主要意义和优势在于，部分 Oracle 内核代码已被移植到在存储单元中运行的`cellsrv`可执行文件中。换言之，处理能力和智能性被带到了存储层。这使得 Oracle 数据库层能够将数据扫描、过滤和投影工作卸载到单元格中。为此，单元格必须能够像数据库一样读取和理解 Oracle 数据块和行内容。**由缓存层处理的单元格块**这一统计指标，表明单元格处理（打开、读取并用于智能扫描）了多少数据块，而不仅仅是将读取的块传递给数据库层。

当单元格以块 I/O 模式仅将块传回数据库时，此统计信息不会更新。但当单元格自身使用这些块进行智能扫描时，为实现一致读而打开块时首先执行的操作之一就是检查块缓存层头。这是为了确保是正确的块、未损坏且有效且一致。这些测试由缓存层函数（`KCB`，即内核缓存缓冲区管理）完成，并作为**由缓存层处理的单元格块**报告回数据库。

在数据库层，对于常规块 I/O，对应的统计信息是`consistent gets from cache`和`consistent gets from cache (fastpath)`，具体取决于用于一致缓冲区获取的缓冲区固定代码路径。注意，`cellsrv`仅执行一致模式缓冲区获取（`CR`读取），不执行当前模式块获取。因此，你在统计信息中看到的所有当前模式获取都是在数据库层完成的，并被报告为`db block gets from cache`或`db block gets from cache (fastpath)`。该统计信息是衡量`cellsrv`为你的会话执行了多少逻辑读的有用且简单的指标。

请注意，在`SQL`计划执行期间看到一些数据库层 I/O 处理是完全正常的，因为该计划可能正在访问多个表（并将它们连接）。因此，在大型事实表和九个维度表之间进行十表连接时，你很可能会看到所有维度都使用常规的、缓存的块 I/O 进行扫描（如果存在索引则使用索引），而只有大型事实表的访问路径才会利用智能扫描。

## 单元格数据层处理

前一个统计信息统计的是缓存层（`KCB`）执行的所有块获取，而此统计信息类似，但统计的是在单元格中由数据层处理的块。此统计信息专门适用于读取表块或物化视图块（其物理上与表块类似）。信息是通过一个名为`KDS`（内核数据扫描）的数据层模块收集的，该模块可以从表块中提取行和列，并将其传递给各种求值函数进行过滤和谓词检查。与数据库层一样，单元格中的数据层能够读取块并提取相关部分。它还负责将操作结果写入`Exadata`特定格式，以便发送到数据库层，使用列简单投影和过滤技术进行处理。

如果单元格智能扫描可以在无需回退到数据库块 I/O 模式的情况下完成所有处理，那么对于大多数智能扫描处理，**由数据层处理**的统计信息加上**由索引层处理**的统计信息应等于**由缓存层处理**的值。这意味着每个实际打开的块都通过了缓存和事务层检查，并被传递到数据层或索引层进行行和列提取。如果**由数据层处理**加上**由索引层处理**的统计信息之和小于**由缓存层处理**的统计信息值，则意味着其余的块未被单元格完全处理，必须送回数据库进行常规块 I/O 处理。

## 单元格索引层处理

此统计信息类似于前面的**由数据层处理的单元格块**，但在智能扫描`B*Tree`或位图索引段块时递增。从索引块中提取行的代码路径与从表中提取行所执行的代码路径不同。**由索引层处理的单元格块**统计了智能扫描处理了多少索引段块。

## 单元格事务层处理

此统计信息显示在存储单元的事务层中处理了多少块。以下是存储单元中智能扫描期间一致读操作序列的简化说明：

缓存层（`KCB`）打开数据块并检查其头部、最后修改`SCN`和清理状态。如果单元格中的块在运行当前智能扫描的查询的快照`SCN`之后未被修改，则该块可以传递给事务层进行处理。但是，如果磁盘（单元格）上的块映像在查询的快照`SCN`之后已被修改，缓存层已经知道该块必须回滚以实现一致读。在这种情况下，该块根本不会传递到单元格事务层，而是单元格回退到块 I/O，并将该块传递给数据库层进行正常的一致读处理。如果块由缓存层传递给事务层（`KTR`），则事务层可以使用提交缓存和`MinActiveSCN`优化来避免执行一致读处理，如果遇到已提交事务的锁定行和未清理块，则可以减少与数据库层的通信量。当不需要在数据库层执行一致读处理时，一致读将由存储单元内的数据层或索引层代码执行。然而，如果一致读无法在单元格内完成，则必须将整个数据块传输回数据库层，并在那里执行一致读。

此解释的要点是，如果智能扫描以最优方式工作，则它们不必中断工作并在智能扫描处理期间与数据库层交换数据。理想情况下，所有扫描工作都在存储单元中完成，一旦有足够的行准备返回，它们就以批处理方式发送到数据库。如果是这种情况，那么**由数据层处理（或由索引层处理）的单元格块**统计信息将等于**由缓存层处理（以及由事务层处理）的单元格块**，这表明所有块都可以在单元格内完全处理并从中提取行，而无需回退到数据库层的块 I/O 和一致读。

请记住，与存储单元中一致读相关的所有复杂性仅在进行智能扫描时才重要。在执行常规块 I/O 时，单元格只是将读取的块直接传回数据库层，一致读逻辑照常在数据库层执行。除非你发现智能扫描等待事件中穿插着`cell single block physical reads`，并且占用了查询响应时间的很大一部分，否则你真的不必担心这些指标。

## 单元格提交缓存查询

这是单元格智能扫描从单元格提交缓存哈希表中查找事务状态的次数。通常，在智能扫描扫描的每个块中发现的每个未提交事务执行一次提交缓存查找——前提是`MinActiveSCN`优化尚未启动并消除检查单个事务状态的需要。这与之前讨论的**由提交缓存帮助的单元格块**统计信息密切相关。


#### 单元闪存缓存读取命中率

此统计数据显示有多少 I/O 请求是由单元闪存缓存满足的，从而无需执行磁盘读取。重点在于“磁盘读取”，而不仅仅是物理读取。从 PCIe 闪存卡读取同样需要物理读取（导致闪存卡 I/O 的系统调用），就像 Linux 中任何对块设备的读取一样。当你看到这个数字时，意味着所需的块不在数据库层的缓冲区缓存中（或者访问路径选择了直接路径读取），但幸运的是，I/O 请求所需的全部或部分块位于单元闪存缓存中。（官方术语是 Exadata 智能闪存缓存。）

请注意，这个数字显示的是 I/O 请求的数量，而不是从单元闪存缓存读取的块数。请记住，单元闪存缓存既可用于常规块读取，也可用于单元智能扫描。为了获得最佳性能，尤其是在 Exadata 上运行 OLTP 数据库时，你应该尝试让大多数单块读取由数据库缓冲区缓存满足，如果不行，则从单元闪存缓存获取。你可以在第 5 章中阅读更多关于 Exadata 闪存缓存的信息。Oracle 决定从单元版本 11.2.3.3.x 及更高版本开始，默认情况下单元将同时从闪存缓存和磁盘执行智能扫描，如果可能，无需任何配置更改。

#### 单元索引扫描

每次在 B\* 树或位图索引段上启动智能扫描时，此统计信息就会递增。请注意，为了在索引段上使用智能扫描，执行计划行源操作符必须是索引快速全扫描，并且采用直接路径读取。此统计信息在智能扫描会话开始时更新。因此，如果你监控一个已执行长时间运行查询一段时间的会话的值，你可能看不到此统计信息为你的会话递增。

当仅对非分区索引段运行串行会话智能扫描时，此统计信息将递增 1。然而，当对分区索引段运行智能扫描时，`cell index scans` 统计信息将为每个使用智能扫描的分区递增。是否尝试智能扫描的决定是在运行时针对分区或子分区段的每个段单独评估的。决定是为每个段（表、索引或分区）在运行时做出的。由于智能扫描需要对 PGA 进行直接路径读取，而直接路径读取的决定又基于扫描段的大小和其他因素，因此同一表的不同分区访问可能使用不同的扫描方法。你可能会发现多分区表或索引中的某些分区没有使用智能扫描/直接路径读取进行扫描，因为 Oracle 决定由于其尺寸较小而使用缓冲读取。在这种情况下，`cell index scans` 统计信息不会递增那么多，并且在 ASH 或 SQL 监控报告中的表/索引扫描行源路径上，你会看到 `cell multiblock physical read` 等待事件出现。

#### 单元 I/O 未压缩字节数

此统计数据显示在单元中扫描的数据的未压缩大小，在扫描 HCC 压缩数据时非常有用。通过一个例子最容易理解此统计信息。考虑以下表：

```sql
SQL> select segment_name, segment_type, bytes/power(1024,2) m, s.blocks,
  2  t.compression, t.compress_for
  3  from user_segments s, user_tables t
  4  where s.segment_name = t.table_name
  5  and segment_name = 'BIGTAB_QH';

SEGMENT_NAME                   SEGMENT_TYPE                M     BLOCKS COMPRESS COMPRESS_FOR
------------------------------ ------------------ ---------- ---------- -------- ------------
BIGTAB_QH                      TABLE                10209.75    1306848 ENABLED  QUERY HIGH
```

如你所见，一个名为 `BIGTAB_QH` 的表使用 Query High 算法进行了 HCC 压缩。它占用大约 10GB 的磁盘空间（压缩后）。作为参考，未压缩的表大小约为 20G。针对此表进行智能扫描会揭示以下与本讨论相关的统计计数器。这些统计信息是使用 Snapper 在五秒间隔内捕获的。简化为所需的最少信息：

```sql
sid username     statistic                                                                  delta
--- ---------- ------------------------------------------------------------------ ---------------
297 MARTIN      physical read total IO requests                                                9.68k
297 MARTIN      physical read total multi block requests                                       9.68k
297 MARTIN      physical read requests optimized                                               9.47k
297 MARTIN      physical read total bytes optimized                                             9.9G
297 MARTIN      physical read total bytes                                                      10.13G
297 MARTIN      physical reads                                                                 1.24M
297 MARTIN      physical reads direct                                                          1.24M
297 MARTIN      physical read IO requests                                                      9.68k
297 MARTIN      physical read bytes                                                           10.13G
297 MARTIN      cell scans                                                                        1
297 MARTIN      cell blocks processed by cache layer                                          1.24M
297 MARTIN      cell blocks processed by txn layer                                            1.24M
297 MARTIN      cell blocks processed by data layer                                           1.24M
297 MARTIN      cell blocks helped by minscn optimization                                     1.24M
297 MARTIN      cell CUs sent uncompressed                                                    4.46k
297 MARTIN      cell CUs processed for uncompressed                                           4.46k
297 MARTIN      cell IO uncompressed bytes                                                    26.57G
```

因此，如果你扫描一个 10GB 的压缩段，`physical read total bytes` 统计信息会增加大约 10GB，但 `cell I/O uncompressed bytes` 会增加 26.57 GB，这反映了扫描数据的总未压缩大小。此统计信息仅在执行智能扫描压缩卸载时递增，当你直接通过块 I/O 将压缩块读取到数据库层时不会递增。有趣的是，在智能扫描未压缩段时，此统计信息也会被填充。


#### 单元格快速响应会话次数

此统计信息显示 Oracle 启动了智能扫描代码但未立即建立智能扫描会话的次数。相反，它选择先执行一些块 I/O 操作，希望找到足够的行来满足数据库会话。此优化用于 `FIRST ROWS` 执行计划选项，无论是使用 `FIRST_ROWS_n` 提示（或等效的 `init.ora` 参数）还是 `WHERE rownum < X` 条件（这也可能在执行计划中启用第一行选项）。其理念是，如果只抓取几行，Oracle 希望避免设置单元格智能扫描会话（涉及所有单元格，这要归功于 ASM 条带化）的开销，但它会先执行一些常规的块 I/O 操作。以下是一个使用 `ROWNUM` 谓词进行 first-rows 优化的示例：

```
select * from t3 where owner like 'S%' and rownum <= 10

Plan hash value: 3128673074

----------------------------------------------------------------------------
| Id  | Operation                           | Name | E-Rows | Cost (%CPU)|
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |      |        |     4 (100)|
|*  1 |  COUNT STOPKEY                      |      |        |            |
|*  2 |   TABLE ACCESS STORAGE FULL FIRST ROWS | T3   |     11 |     4   (0)|
----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter(ROWNUM<=10)
   2 - storage("OWNER" LIKE 'S%')
       filter("OWNER" LIKE 'S%')
```

如果您在此查询运行的同时运行 Snapper，您很可能会看到 `cell num fast response sessions` 增加，因为 Oracle 试图避免智能扫描会话的建立：

```
NAME                                                              VALUE
---------------------------------------------------------------- ----------
cell num fast response sessions                                       1
cell num fast response sessions continuing to smart scan               0
```

单元格快速响应功能由 `_kcfis_fast_response_enabled` 参数控制，并且默认启用。

#### 继续执行智能扫描的快速响应会话次数

此统计信息显示单元格智能扫描快速响应会话启动的次数，但 Oracle 因为在前几个 I/O 操作中未找到足够匹配的行，不得不切换到真正的智能扫描会话。下一个示例基于前一个示例，但在查询中添加了额外的谓词：

```
select * from t3 where owner like 'S%' and object_name LIKE '%non-existent%'
and rownum <= 10

Plan hash value: 3128673074

----------------------------------------------------------------------------
| Id  | Operation                            | Name | E-Rows | Cost (%CPU)|
----------------------------------------------------------------------------
|   0 | SELECT STATEMENT                     |      |        |     9 (100)|
|*  1 |  COUNT STOPKEY                       |      |        |            |
|*  2 |   TABLE ACCESS STORAGE FULL FIRST ROWS| T3   |     10 |     9   (0)|
----------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   1 - filter(ROWNUM<=10)
   2 - storage(("OBJECT_NAME" LIKE '%non-existent%' AND "OWNER" LIKE
               'S%' AND "OBJECT_NAME" IS NOT NULL))
       filter(("OBJECT_NAME" LIKE '%non-existent%' AND "OWNER" LIKE
               'S%' AND "OBJECT_NAME" IS NOT NULL))
```

使用 Snapper 观察统计信息显示，继续执行智能扫描的快速响应会话数量增加了：

```
NAME                                                              VALUE
---------------------------------------------------------------- ----------
cell num fast response sessions                                       1
cell num fast response sessions continuing to smart scan               1
```

#### 因故使用直通模式的智能 IO 会话次数

有三个相关的统计计数器，其中 `reason` 可以是 `user`、`cellsrv` 或 `timezone to`，用以指示 Oracle 数据库发起智能扫描但随后执行失败的次数。在 11.2.0.4 和 12.1.0.2 中，您还可以获得通过直通模式发送的实际数据量。它记录在 `cell num bytes in passthru during predicate offload` 中。在这种情况下，`cellsrv` 没有启动智能扫描，而是完全回退到块 I/O 模式。读取的块只是被传递到数据库，而不是在单元格内处理。这意味着，虽然您仍然看到 `cell smart scan` 等待事件和 `cell physical IO interconnect bytes returned by smart scan` 在增加（这表明智能扫描正在发生），但并未利用智能扫描的全部功能，因为单元格只是读取数据块并将它们返回到数据库层和会话的 PGA 中。换句话说，在直通模式下，单元格不会打开数据块并仅提取所需列的匹配行，而是返回段的所有物理块，保持原样。请注意，存储索引可以在直通模式下用于消除 I/O，但请记住这些索引必须首先通过常规的智能扫描填充。如果一开始就没有智能扫描，这将是困难的。如果正在扫描的段缓存在闪存缓存中，您将看到它被使用。

除非您遇到单元格内存不足等问题，否则您不应该在最新的数据库和 Exadata 单元格版本上看到任何直通智能扫描发生。您可以通过在会话中将 `_kcfis_cell_passthru_enabled` 设置为 `TRUE` 并运行智能扫描，在测试环境中测试直通模式会发生什么。您仍然会看到智能扫描的 `cell smart scan` 等待事件，但它们更慢，因为它们正在将所有块返回给数据库进行处理。我们唯一一次系统地看到这个问题是在 12c RDBMS 发布并认证在 Exadata 上时，但单元格上的软件版本不支持任何卸载。

我们也曾因为时区升级失败并卡住，而看到 `cell num smart IO sessions using passthru mode due to timezone once`。如果 `cellsrv` 几乎耗尽内存，您将看到原因标为 `cellsrv` 的计数器在增加。

**注意**

`Cell num smart IO sessions using passthrough mode due to reason` 很难检测。大多数性能工具会显示 `cell smart table scan` 事件，而其他通常用于判断智能扫描是否发生的统计信息，在智能扫描正常工作时也会增加。RDBMS 12c 的实时 SQL 监视器（在第 12 章中介绍）现在在“other data”列中显示有关直通模式的信息。


## 闪存缓存中的单元覆盖

这个特定的会话统计信息是在 Oracle `11.2.0.4` 中引入的。它在 Oracle `12.1.0.2` 中也可见，但在 `12.1.0.1` 中不可见。在 `11.2.3.2.x` 中引入`回写闪存缓存 (WBFC)`之前，无需担心对闪存缓存的写入。`单元智能闪存缓存`主要用于加速 OLTP 类工作负载的读取，并且从单元版本 `11.2.3.3.3.x` 开始，它也被系统地用于`智能扫描`。如果你想衡量闪存缓存对工作负载的益处，可以检查前面描述的`cell flash cache read hits`值。或者，你也可以考虑`physical read requests optimized`以及`physical read total bytes optimized`，但这两个统计信息也会包含来自存储索引的信息。

写入是不同的。在引入`WBFC`后相当长一段时间内，没有可用的统计信息来衡量对闪存缓存的写入。这种情况在 `11.2.0.4` 中发生了变化，并引入了一些新的统计信息，例如：

* `Cell overwrites in Flash Cache`
* `Cell partial writes in Flash Cache`
* `Physical writes optimized`

考虑一个所有单元都启用了`WBFC`的 `12.1.0.2` 系统，你可以看到 Oracle 后台进程负责了大量这类写入。如果你想捕获用户会话写入`WBFC`的情况，需要在它们断开连接之前进行。以下是我们 `12.1.1.1.1` 单元/`12.1.0.2` RDBMS 系统上的一个示例：

```sql
select se.sid, sn.name, s.value, se.program
from v$sesstat s natural join v$statname sn
left join v$session se on (s.sid = se.sid)
where sn.name in (
  'physical write requests optimized',
  'cell writes to flash cache',
  'cell overwrites in flash cache')
and s.value <> 0
order by s.sid,name;
```

```
SID NAME                                     VALUE PROGRAM
---------- ----------------------------------- ---------- -----------------------------------
1 cell overwrites in flash cache            6258 oracle@enkdb03.enkitec.com (DBW1)
1 cell writes to flash cache               10848 oracle@enkdb03.enkitec.com (DBW1)
1 physical write requests optimized         5405 oracle@enkdb03.enkitec.com (DBW1)
66 cell overwrites in flash cache            9894 oracle@enkdb03.enkitec.com (DBW2)
66 cell writes to flash cache               15218 oracle@enkdb03.enkitec.com (DBW2)
66 physical write requests optimized         7593 oracle@enkdb03.enkitec.com (DBW2)
132 cell overwrites in flash cache              94 oracle@enkdb03.enkitec.com (LGWR)
132 cell writes to flash cache                 218 oracle@enkdb03.enkitec.com (LGWR)
132 physical write requests optimized           38 oracle@enkdb03.enkitec.com (LGWR)
197 cell overwrites in flash cache           62991 oracle@enkdb03.enkitec.com (CKPT)
197 cell writes to flash cache               62991 oracle@enkdb03.enkitec.com (CKPT)
197 physical write requests optimized        20997 oracle@enkdb03.enkitec.com (CKPT)
262 cell writes to flash cache                2300 oracle@enkdb03.enkitec.com (LG00)
262 physical write requests optimized           76 oracle@enkdb03.enkitec.com (LG00)
392 cell writes to flash cache                   3 oracle@enkdb03.enkitec.com (LG01)
392 physical write requests optimized            1 oracle@enkdb03.enkitec.com (LG01)
782 cell overwrites in flash cache            3510 oracle@enkdb03.enkitec.com (MMON)
782 cell writes to flash cache                3552 oracle@enkdb03.enkitec.com (MMON)
782 physical write requests optimized         1184 oracle@enkdb03.enkitec.com (MMON)
977 cell overwrites in flash cache              63 oracle@enkdb03.enkitec.com (LMON)
977 cell writes to flash cache                  63 oracle@enkdb03.enkitec.com (LMON)
977 physical write requests optimized           21 oracle@enkdb03.enkitec.com (LMON)
1496 cell overwrites in flash cache           11502 oracle@enkdb03.enkitec.com (DBW0)
1496 cell writes to flash cache               17822 oracle@enkdb03.enkitec.com (DBW0)
1496 physical write requests optimized         8888 oracle@enkdb03.enkitec.com (DBW0)
1498 cell overwrites in flash cache              33 oracle@enkdb03.enkitec.com (ARC0)
1498 cell writes to flash cache                  33 oracle@enkdb03.enkitec.com (ARC0)
1498 physical write requests optimized           11 oracle@enkdb03.enkitec.com (ARC0)
28 rows selected.
```

请注意，对闪存缓存的写入量大约是 RDBMS 报告的写入量的两倍；这是由 `ASM` 镜像引起的。此系统上的 `ASM` 磁盘组是以普通冗余创建的。



## 单元物理 I/O 字节数（适用于谓词下压）

此性能计数器是理解**智能扫描**最重要的统计数据之一。当你对一个段进行智能扫描时，此统计数据会显示：如果返回该段中的每一个比特，智能扫描将需要处理多少字节。本质上，此统计数据覆盖了从段起始位置一直到其高水位标记的所有字节（因为扫描会贯穿整个段）。需要注意的是，这是理论上需要扫描的最大字节数，但它没有考虑可能让智能扫描跳过磁盘数据的存储索引。

即使存储索引让你避免了扫描一个 10GB 段的 80%，将实际 I/O 量减少到只有 2GB，此统计数据仍然会显示被扫描段的总大小，而不考虑任何优化。实践经验表明，这种情况经常发生。你需要密切关注 `cell physical IO bytes saved by storage index` 统计数据，如下所示：

```
sid username     statistic                                                         delta
790 MARTIN       cell physical IO interconnect bytes                              3.19M
790 MARTIN       cell physical IO bytes eligible for predicate offload           21.85G
790 MARTIN       cell physical IO bytes saved by storage index                    2.1M
790 MARTIN       cell physical IO interconnect bytes returned by smart scan       3.19M
790 MARTIN       cell num smartio automem buffer allocation attempts                1
790 MARTIN       cell scans                                                          1
790 MARTIN       cell blocks processed by cache layer                             2.67M
790 MARTIN       cell blocks processed by txn layer                               2.67M
790 MARTIN       cell blocks processed by data layer                              2.67M
790 MARTIN       cell blocks helped by minscn optimization                        2.67M
790 MARTIN       cell IO uncompressed bytes                                     21.84G
790 MARTIN       cell flash cache read hits                                     18.95k
```

请不要担心这里显示的其他统计数据——它们也是本章的一部分。请注意，`cell physical IO bytes eligible for predicate offload` 仅仅是根据数据文件中数据块的物理大小进行计数，而不是任何解压缩、过滤或投影后的“最终”数据大小。

如果你会话的 `V$SESSTAT`（或查看整个实例时的 Statspack/AWR 数据）中此数字没有增加，那么这是智能扫描未被使用的另一个指标。智能扫描会话所扫描过的任何块范围（甚至因为存储索引优化而跳过的范围）都应该会使此统计数据增加。另一个值得了解的事实是，当智能扫描回退到直通（全块传输）模式（前文已描述）时，`cell physical IO bytes eligible for predicate offload` 统计数据无论如何都会增加，尽管在直通模式下，单元中没有进行谓词下压和智能扫描过滤。

## 单元物理 I/O 字节数（由存储索引节省）

这是另一个重要的统计数据，它显示了智能扫描会话能够跳过不从磁盘读取的数据块的数量，这得益于 `cellsrv` 中的内存存储索引。此统计数据 `cell physical IO bytes saved by storage index` 与 `cell physical IO bytes eligible for predicate offload` 密切相关。如果两者的比率接近 1，那就清楚地表明智能扫描极大地受益于存储索引，并因此避免了大量的 I/O。请记住第四章的内容：存储索引不是持久化结构：它们会随时间演变，并且不能保证总是可用的。

另外请注意，此统计数据是累积的。如果你想调查由于存储索引而可以跳过多少字节，你需要在 SQL 语句执行前获取该统计数据的当前值，并在执行完成后立即再次获取，以计算两者之间的差值。该统计数据也会被汇总到 `physical read requests optimized` 和 `physical read total bytes optimized` 中。

## 单元物理 I/O 字节数（直接发送到数据库节点以平衡 CPU）

如果出现此统计数据显示——例如在 Snapper 运行期间——这表明存储服务器存在问题。在某些条件下，例如当单元 CPU 负载严重，而 RDBMS 层有备用 CPU 周期时，后者可以负责为其解压缩 CU（列单元）。RDBMS 层和存储层之间存在一定数量的通信，包括交换与 CPU 相关的信息。如果一个单元 CPU 受限，它可能会将未压缩的列或整个 CU 发送回来。

看到此统计数据显示计数器增加，表明系统存在问题，你应该调查为什么单元的 CPU 负载如此之高。使用 `dcli` 是调查 CPU 负载的一个良好起点。如果问题仅限于某个单元，值得连接到它并进行额外的故障排除。请注意，这纯粹是 CPU 问题，不一定与磁盘/内存相关。有针对这些方面的不同统计数据。

## 单元物理 I/O 互联字节数

这是一个简单但基础的统计数据，它显示了在存储单元和你的数据库会话之间传输的任何数据的字节数。这包括所有数据——数据库发送和接收的——智能扫描结果集、从单元读取的全块、临时 I/O 读写、日志写入、任何补充的 iDB 流量等等。因此，此统计数据显示所有流量（以字节为单位），无论其方向、内容或性质如何。

在测量写入 I/O 指标时，看到 `cell physical I/O interconnect bytes` 统计数据比 `physical write total bytes` 统计数据高两到三倍是完全正常的。这是因为后者是在 Oracle 数据库级别测量的，而 `cell physical I/O interconnect bytes` 是在单元级别、完成 `ASM` 镜像后测量的。例如，如果 `LGWR` 向一个具有高冗余度（三重镜像）的 `ASM` 磁盘组写入 1MB，那么总共 3MB 的数据将通过互联网络发送。

## 单元物理 I/O 互联字节数（由智能扫描返回）

此重要统计数据显示了智能扫描返回给数据库层的数据字节数。为了使智能扫描效率最高，实际返回的字节数应远少于扫描（即从磁盘读取）的字节数。这正是 Exadata 智能扫描特性的主要目的——单元可能每秒读取千兆字节的数据，但由于它们通过谓词下压进行早期过滤，它们可能只将一小部分行发送回数据库层。此外，由于投影下压，智能扫描只返回请求的列，而不是整行数据。当然，如果应用程序使用 `SELECT *` 来获取表的所有列，投影下压就没有帮助，但使用谓词下压的早期过滤仍然非常有用。

此统计数据是 `cell physical I/O interconnect bytes` 统计数据的一个子集，但它只计算由智能扫描会话返回的字节数，不包括其他流量。例如，如果在磁盘上排序，你可能会看到 `cell physical I/O interconnect bytes` 报告的值大于 `cell physical I/O interconnect bytes returned by smart scan`。



#### 单元格扫描

此统计指标在性质上与 `单元格索引扫描` 类似，但 `单元格扫描` 显示了在表和物化视图片段（包括其分区）上执行的智能扫描次数。在串行执行情况下，此统计指标在每次片段扫描开始时递增一次。当扫描分区表时（其中每个分区都是一个独立的片段），该统计指标将为每个分区递增一次。在并行扫描情况下，`单元格扫描` 统计指标会递增更多，因为并行从进程会对查询协调器分配给它们的块范围（`PX 颗粒`）执行其扫描。因此，对每个块范围的扫描都被报告为一次独立的 `单元格扫描`。`表扫描（ROWID 范围）` 统计指标的出现表明正在发生对块范围的 `PX` 扫描。

#### 单元格智能 IO 会话缓存命中

此统计指标显示数据库会话成功重用单元格中先前初始化的智能扫描会话的次数。当单个执行计划扫描多个片段（如分区表）或在单次执行过程中重新访问同一片段时，此统计指标会出现。

#### 单元格智能 IO 会话缓存查找

此统计指标在数据库会话尝试重用单元格中先前初始化的智能扫描会话时递增。如果 `单元格智能 IO 会话缓存命中` 统计指标也同时递增，则表示查找成功并且可以重用先前的会话上下文。智能 I/O 会话缓存仅在打开游标的执行（及后续获取）期间有效。一旦执行完成，即使是同一游标的后续执行，也必须建立新的智能扫描会话，并将新的一致性读取快照 `SCN` 也传送给单元格。

#### 在提交缓存中找到的单元格事务

此统计指标与 Oracle 必须保证的一致性读取（`CR`）机制相关，即使在 Exadata 上也是如此。它显示了智能扫描会话检查 `单元格提交缓存` 以决定是否需要 `CR 回滚`，并在 `单元格提交缓存` 中找到事务状态信息的次数。这避免了往返数据库层以使用那里可用的 undo 数据检查该事务状态的开销。您可以在 `提交缓存辅助的单元格块` 统计指标部分阅读更多关于 Exadata 智能扫描如何处理一致性读取的信息。

#### 单元格处理的链式行

在解释这个特定统计指标的含义之前，让我们先了解什么是链式行以及智能扫描如何处理链式行。在一些特殊情况下，行会从一个块移动到另一个块，或“跨越”多个块。Oracle 必须管理那些太大而无法放入一个块的行。它还必须处理增长的行，例如，更新一个最初只包含几个字符的 `varchar2` 列，将其更新为几百个字符。

行链接意味着一行被分布或分散存储在多个块中。从技术上讲，一行被分成多个行片段。在许多情况下，头部片段和行的其余部分位于同一块中，这对 Exadata 处理是有利的。行链接最常发生在大型行上，这是处理行中大量数据的代价。对于一个链式行，头部片段可能在块 x 中，而行的其余部分在块 y 和 z 中。行的每个片段都有一个指向下一个片段的指针，称为 `NRID`（下一个 `ROWID`）。这是非常合乎逻辑的：如果你想存储 100KB 的行，那么你根本无法将它们塞入一个 8K 的块中，行链接是不可避免的。您可以在块转储中看到这一点。该表是使用以下语句创建的：

```sql
CREATE TABLE chaines2
(
id, a, b, c, d, e
) as
WITH v1 as (
SELECT rownum n FROM dual CONNECT BY level <= 10000
)
SELECT  rownum id,
rpad('a',1980,'*'),
rpad('b',1980,'*'),
rpad('c',1980,'*'),
rpad('d',1980,'*'),
rpad('e',1980,'*')
FROM v1,
v1
WHERE rownum <= 100000;
```

检查这些行，您会看到它们分布在多个块中：

```
Start dump data blocks tsn: 5 file#:5 minblk 2132987 maxblk 2132987
[...]
block_row_dump:
tab 0, row 0, @0x22
tl: 7225 fb: --H-F--N lb: 0x0  cc: 5
nrid:  0x01608bfc.0
col  0: [ 2]  c1 02
col  1: [1980]
61 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a
[...]
col  2: [1980]
62 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a
[...]
col  3: [1980]
63 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a
[...]
col  4: [1261]
64 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a 2a
end_of_block_dump
```

并非所有列都适合放入这一行，只有列 d 的大约三分之二被包含在内。该行必须在另一个块中继续，因为列 d 的剩余部分以及列 e 需要被表示。这体现在行头信息中，更具体地说是在标志位中：`--H-F--N`。它翻译为：（`H`）头部片段、（`F`）第一个片段和（`N`）后续片段，解读为：该行的头部片段和第一个片段位于此块中，但该行的另一部分位于另一个块中。Oracle 如何找到下一个块？信息被编码在 `NRID` 中，即块地址。此示例中的 `NRID` 是 `0x01608bfc.0`。十六进制数的第一部分是块地址；最后部分（`.0`）是该块中的第 n 行。`DBMS_UTILITY` 提供了一组函数，允许我们将 `NRID` 解码为文件和块地址：

```sql
SQL> select
 2    dbms_utility.data_block_address_file(to_number('01608bfc','xxxxxxxxxxxxx')) fno,
 3    dbms_utility.data_block_address_block(to_number('01608bfc','xxxxxxxxxxxx')) blockno
 3  from dual;

FNO    BLOCKNO
---------- ----------
5    2132988
```

如果您也转储该块，您将看到该行在继续：

```
Start dump data blocks tsn: 5 file#:5 minblk 2132988 maxblk 2132988
[...]
block_row_dump:
tab 0, row 0, @0x14ec
tl: 2708 fb: -----LP- lb: 0x0  cc: 2
col  0: [719]
[...]
col  1: [1980]
[...]
tab 0, row 1, @0x18
tl: 4527 fb: --H-F--N lb: 0x0  cc: 4
nrid:  0x01608bfd.0
col  0: [ 2]  c1 03
```


