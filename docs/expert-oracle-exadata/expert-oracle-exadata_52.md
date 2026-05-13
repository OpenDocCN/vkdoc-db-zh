# 理解 Exadata Smart Scan 指标与性能计数器

当您使用支持存储感知行源的 Oracle 执行计划时，您仍然无法完全确定是否真正尝试了 Smart Scan 并且其是否如预期那样工作。在第 10 章中介绍的等待接口是性能相关信息的一个可靠来源。在查询执行期间，可以参考相关的 V$ 视图来确定您的会话正在等待什么。考虑以下可能性：

*   **仅 CPU 使用率**：这似乎意味着使用了缓冲数据访问（而非直接路径），这可以从遍历缓冲区高速缓存且所有数据恰好已缓存时缺少 I/O 等待事件中看出。但请注意：ASH 以及从中获取性能数据的工具使用一秒的采样间隔，可能会遗漏非常短暂的物理 I/O 事件。经典的 SQL Trace 或使用 snapper 从 `v$sesstat`/`v$mystat` 捕获信息所获取的数据来源与 ASH 不同。
*   **cell multiblock physical read**：显然使用了缓冲多块读取（看起来像是全段扫描），但多块读取等待也可能针对 LOB 和 SecureFile 读取操作报告，其中对于 LOB，LOB 块大小大于块大小。否则，对于 LOB 访问将报告单块读取。在 Exadata 单元软件 12.1.1.1.1 及更高版本中，内联 LOB 可以被下载。
*   **cell single block physical read**：显然使用了缓冲单块读取。如果这是您看到的唯一 I/O 等待事件（且未与多块读取一起出现），那么您似乎根本没有使用全段扫描。有时单块读取的出现是由于执行计划中的其他操作（如某些索引范围扫描）或数据块中的行链接。

如果您在会话中看到常规的 `cell multiblock physical read` 等待事件，那么显然没有使用直接路径读取。这主要发生在串行执行的操作中，例如当您使用 `parallel_degree_policy = MANUAL` 或 `LIMITED` 时。并行执行从属进程很可能会执行直接路径读取扫描，这些扫描随后将被下载并作为 Smart Scan 执行。另一方面，当您使用新的自动并行度策略（`parallel_degree_policy = AUTO` 或 `ADAPTIVE`）并且 Oracle 决定执行内存中并行查询时，即使是并行操作，Oracle 也会使用通过缓冲区高速缓存的读取，其等待事件将显示为“缓冲”读取结果。

除了这些问题之外，还有更多原因和特殊情况会导致 Smart Scan 被静默地不使用或回退到常规块 I/O 模式——这可能会比您预期的更显著地减慢查询和工作负载。值得庆幸的是，Oracle Exadata 提供了非常完善的检测工具，通过使用性能框架，分析人员可以审查性能不佳的查询并进行相应优化。

## Exadata 动态性能计数器

虽然 Oracle 等待接口的等待事件为我们提供了关于数据库响应时间消耗在何处的关键信息，但 Exadata 动态性能计数器将我们向前推进了一步，并解释了 Oracle 内核正在执行何种操作或任务——以及执行了多少次。等待事件和性能计数器相辅相成，不应单独使用。Oracle 动态性能计数器也称为 `V$SESSTAT` 或 `V$SYSSTAT` 统计信息（或计数器），因为这些视图用于访问它们。当使用 12c 多租户数据库时，您可能也会对 `V$CON_SYSSTAT` 感兴趣。


## 如何及何时使用性能计数器

在排查性能问题时，你应始终从分析等待事件和 SQL ID 级别的活动度量开始排查过程。这些指标追踪了最终用户所关心的**时间**。如果需要更详细的细节，接着再查看性能计数器。如果标准的等待事件信息未能提供足够详情，性能计数器能为你提供关于 Oracle 会话正在做什么的、非常深入的洞察。例如，如果你的会话似乎消耗了大量 CPU，你可以查看某个会话的 `session logical reads` 或 `parse count (hard)` 计数器是否比平常增长得更多。或者，如果你在全表扫描期间看到一些意外的单块读取，你可以检查 `table fetch continued row` 或类似 `data blocks consistent reads – undo records applied` 的统计信息是否增加，这些指标表明可能存在链接/迁移的行或一致性读（CR）缓冲区克隆加上回滚开销。另一个有用的度量是 `user commits`，它能让你了解一个实例（或选定会话中）完成了多少数据库事务。因此，下次当一个会话似乎在等待 `log file sync` 等待事件时，你可以从 `V$SESSTAT` 中检查其 `user commits` 计数器值，看看该会话提交工作的频率如何。

不幸的是，Oracle 并未记录所有性能计数器。已记录的计数器可以在 Oracle 12c 参考手册的附录 E “统计信息描述”中找到。一些与单元相关的性能计数器记录在 Exadata 文档集的 Exadata 存储服务器软件用户指南中。然而，文档中的细节程度有时不足以让你完全理解某个给定的性能计数器。经验会告诉你哪些是重要的，哪些不是。

尽管在 AWR 报告中也能找到性能统计信息，但当问题正在发生时使用它们最有价值。在计算这些计数器的变化率（“delta”）时，你可以更详细地观察你的会话正在执行什么操作。

你本章前文已了解到性能计数器被分配到一个类中。但即使在同一个类内，你也可以区分出相关的计数器组。以物理 I/O 为例。有针对读和写的统计信息，不同的读取类型如下所示：

```sql
SQL> select name from v$statname where name like 'physical reads%' order by name;

NAME
----------------------------------------------------------------
physical reads
physical reads cache
physical reads cache for securefile flashback block new
physical reads cache prefetch
physical reads direct
physical reads direct (lob)
physical reads direct for securefile flashback block new
physical reads direct temporary tablespace
physical reads for flashback new
physical reads prefetch warmup
physical reads retry corrupt

11 rows selected.
```

现在，如果你要执行一条 SQL 语句，并在查询开始和结束时捕获计数器的值，你就可以计算结束值与开始值之间的差值。在下一个例子中，我们针对物理读取做了这件事。会话执行了以下语句：

```sql
SQL> select count(*) from bigtab union all select count(*) from bigtab;
```

在查询执行期间，以下与物理读取相关的统计计数器发生了变化：

```
physical read bytes                                              : 27603034112
physical read IO requests                                        : 26379
physical read requests optimized                                 : 23671
physical reads                                                   : 3369127
physical reads direct                                            : 3369127
physical read total bytes                                        : 27528650752
physical read total bytes optimized                              : 24767488000
physical read total IO requests                                  : 26310
physical read total multi block requests                         : 26286
```

这应该能让你非常清晰地了解数据库层为该特定会话和查询所报告的物理读取情况。请注意，这些并非计入的唯一物理读取——在后面的章节中，你将了解到与智能扫描处理相关的完整 I/O 事件谱系。

等待接口则会给你以下信息，取自一个经过 tkprof 处理的跟踪文件：

```
Elapsed times include waiting on following events:
Event waited on                                         Times   Max. Wait  Total Waited
-------------------------------------------------------- Waited  ----------  ------------
Disk file operations I/O                                     2        0.00          0.00
SQL*Net message to client                                    2        0.00          0.00
enq: KO - fast object checkpoint                             6        0.00          0.00
reliable message                                             2        0.00          0.00
cell smart table scan                                     2969        0.01          2.55
SQL*Net message from client                                   2        1.49          1.49
```

这两者——统计计数器和等待接口——都提供了关于会话活动的信息。然而，你应该已经清楚的是，会话统计信息能为你提供关于磁盘读取更详细的信息。其他统计信息——请记住，在 12.1.0.2 中，Oracle 为你的会话维护了 1178 个统计信息——能让你更深入地了解所执行处理的其他方面。仅仅查看等待信息，你无法确定是否访问了闪存缓存来提供相关数据。性能计数器允许你确认，在 26,379 次物理读取请求中，有 23,671 次使用了闪存缓存。在该会话的当前查询中，存储索引被特别禁用，因此并未发挥作用（尽管严格来说，由于缺少 where 子句，这并非必要）。

动态性能计数器提供了重要的线索，使你能够更好地引导你的故障排查工作。请注意，像 Statspack 和 AWR 报告这样的工具严重依赖 `V$SYSSTAT` 计数器。它们只是将这些自实例启动以来不断增长的数字的值存储在其存储库表中。因此，每当你运行 Statspack/AWR 报告时，报告的只是选定快照之间数值的差值。Statspack 和 AWR 报告的核心就是向你展示不同时间点的 `V$SYSSTAT`（及其他视图）数值之间的差值。

虽然 `V$SYSSTAT` 视图对于监控和诊断整个实例的性能（如 AWR 和 Statspack 报告所做的）是没问题的，但它的问题在于，你无法使用全系统范围的统计信息来诊断单个会话的问题。系统级统计信息将你所有（可能数千个）会话的度量数据聚合到一组计数器中。这就是为什么 Oracle 还有 `V$SESSTAT`，它为每个会话单独跟踪所有这些独立的计数器！实例中的每个会话都有自己成百上千个性能计数器，只跟踪其自身的活动。这个动态性能视图确实是一个金矿——如果实例中只有少数会话（或用户）有问题，你可以只监控他们的活动，而不会被数据库中所有其他用户的噪音所干扰。


如前所述，`V$SYSSTAT` 累积了实例范围内的性能计数器；它们从零开始，并且只在实例生命周期内增加。大多数 `V$SESSTAT` 计数器始终是递增的（累计统计），但也有一些例外，例如 `logons current` 和 `session pga/uga memory`。无论如何，在检查计数器值时，你不应仅仅查看计数器的当前值，特别是当你的会话已经连接了一段时间时。问题在于，即使你在某个长时间运行的连接池会话的 `V$SESSTAT` 中看到一个看起来很大的数字，你如何知道其中有多少是在今天、此时、当你遇到问题时递增或添加的，而不是几周前该会话登录时累积的呢？换句话说，在排查当前发生的问题时，你应该查看当前时间段内、特定问题时间区间的性能指标。排查过去的问题时，也适用类似的原则。

## Oracle Session Snapper

这就是为什么合著者 Tanel Poder 编写了一个“小巧”的辅助工具，名为 Oracle Session Snapper。该工具可以方便地显示来自 `V$SESSTAT` 及其他各种会话级性能视图的会话当前活动。关于此工具的一个重要方面是，它“只”是一个动态解析的匿名 PL/SQL 块；它不需要在数据库中进行任何安装或拥有 DDL 权限。这应该使其易于部署和使用。当前版本的 Snapper 可在线获取，地址为 `ExpertOracleExadata.com`。

以下是一个如何运行 Snapper 来测量 SID 789 活动度的示例（针对单个五秒的时间间隔）。在此示例中，脚本被重命名为 `snapper4.sql`，以便与之前的版本区分。请阅读 Snapper 的头部说明以获取详细文档。在此示例中，Snapper 被指示报告会话 789 的性能计数器的任何差异。此外，它还从 `V$SESSION` 采样等待事件相关信息，并以类似 ASH 的格式呈现。

**注意**

遗憾的是，Snapper 生成的有关会话统计的输出对于本书来说太宽了。这里使用了一个小过滤器将其修剪到可管理的尺寸。

```sql
sid username      statistic                                                                delta
--- ------------ -------------------------------------------------------------------------- ---------
789 MARTIN       向客户端发送/从客户端接收的请求数                                                  1
789 MARTIN       打开的游标累计数                                                               1
789 MARTIN       用户调用次数                                                                     2
789 MARTIN       当前固定的游标数                                                         1
789 MARTIN       会话逻辑读次数                                                      2.59M
789 MARTIN       用户 I/O 等待时间                                                          166
789 MARTIN       非空闲等待时间                                                          166
789 MARTIN       非空闲等待次数                                                       8.14k
789 MARTIN       会话 UGA 内存使用量                                                        6.23M
789 MARTIN       会话 PGA 内存使用量                                                         8.85M
789 MARTIN       锁存等待次数                                                                3
789 MARTIN       锁存请求次数                                                             2
789 MARTIN       锁存转换次数                                                          4
789 MARTIN       锁存释放次数                                                             2
789 MARTIN       全局锁存同步获取次数                                                      6
789 MARTIN       全局锁存释放次数                                                       2
789 MARTIN       物理读取总 IO 请求数                                         20.24k
789 MARTIN       物理读取总多块请求数                                20.22k
```



`789 MARTIN       优化后的物理读请求                                             18.31k`

`789 MARTIN       优化后的物理读总字节数                                          19.17G`

`789 MARTIN       物理读总字节数                                                    21.18G`

`789 MARTIN       存储节点物理 IO 互联字节数                                         416.83M`

`789 MARTIN       GES 消息发送数                                                         3`

`789 MARTIN       一致性获取                                                       2.59M`

`789 MARTIN       从缓存中获得的一致性获取                                               13`

`789 MARTIN       从缓存中获得的一致性获取（快速路径）                                    13`

`789 MARTIN       直接一致性获取                                                 2.59M`

`789 MARTIN       从缓存中逻辑读取的字节数                                        106.5k`

`789 MARTIN       物理读次数                                                        2.59M`

`789 MARTIN       直接物理读次数                                                 2.59M`

`789 MARTIN       物理读 IO 请求数                                                    20.24k`

`789 MARTIN       物理读字节数                                                  21.19G`

`789 MARTIN       调用 kcmgcs 的次数                                                          13`

`789 MARTIN       调用获取快照 SCN（kcmgss）的次数                                         1`

`789 MARTIN       文件 IO 等待时间                                                    53.16k`

`789 MARTIN       适合进行谓词下推卸载的存储节点物理 IO 字节数                21.19G`

`789 MARTIN       智能扫描返回的存储节点物理 IO 互联字节数          417.12M`

`789 MARTIN       存储节点智能 IO 自动内存缓冲区分配尝试次数                       1`

`789 MARTIN       表扫描（长表）                                                         1`

`789 MARTIN       表扫描（直接读）                                                         1`

`789 MARTIN       表扫描获取的行数                                               15.54M`

`789 MARTIN       表扫描获取的数据块数                                              2.59M`

`789 MARTIN       存储节点扫描次数                                                        1`

`789 MARTIN       缓存层处理的数据块数                                          2.59M`

`789 MARTIN       事务层处理的数据块数                                            2.59M`

`789 MARTIN       数据层处理的数据块数                                           2.59M`

`789 MARTIN       受益于最小 SCN 优化的数据块数                             2.59M`

`789 MARTIN       存储节点 IO 解压缩字节数                                           21.24G`

`789 MARTIN       会话游标缓存命中次数                                                 1`

`789 MARTIN       会话游标缓存计数                                                        1`

`789 MARTIN       工作区内存分配量                                             3.09k`

`789 MARTIN       解析次数（总计）                                                       1`

`789 MARTIN       执行次数                                                             1`

`789 MARTIN       通过 SQL*Net 发送到客户端的字节数                                          1`

`789 MARTIN       通过 SQL*Net 从客户端接收的字节数                                  298`

`789 MARTIN       闪存缓存读命中次数                                           18.41k`

模拟的 ASH 信息符合其限制范围，显示如下：

```
SYS@DBM011:1> @snapper4 ash 5 1 789
Sampling SID 789 with interval 5 seconds, taking 1 snapshots...
-- Session Snapper v4.12 BETA - by Tanel Poder ( http://blog.tanelpoder.com )
-- Enjoy the Most Advanced Oracle Troubleshooting Script on the Planet! :)
----------------------------------------------------------------------------------------------
Active% | INST | SQL_ID          | SQL_CHILD | EVENT                           | WAIT_CLASS
----------------------------------------------------------------------------------------------
    50% |    1 | cdjur50gj9h6s   | 0         | ON CPU                          | ON CPU
    28% |    1 | cdjur50gj9h6s   | 0         | cell smart table scan           | User I/O
--  End of ASH snap 1, end=2014-08-06 09:04:33, seconds=5, samples_taken=36
PL/SQL procedure successfully completed.
```

正如您从输出中看到的，在五秒间隔内捕获的语句显然执行了一次智能扫描，这在模拟的 ASH 部分中有所指示，并在上文的`cell scans`值为`1`（发生了一次段扫描）的部分得到确认。在此上下文中无需担心 ASH 问题。引用 Snapper 文档：“Snapper 中的‘ASH’功能只是对`GV$SESSION`视图进行采样，因此您**不需要**诊断包许可证即可使用 Snapper 的‘ASH’输出。”

此输出还显示会话`789`执行了`2.59M`次逻辑读。它发出了总共`20.24k`次读取 IO 请求以读取`21.18G`数据，几乎完全由多块 I/O 请求满足。在此示例中，选择了人类可读的增量值，但 Snapper 当然也可以打印精确值。


### Exadata 性能计数器的含义与解析

在介绍了它们的用途之后，现在是时候深入探讨性能计数器的含义了。无论性能工具利用这些指标绘制出的图表或图片多么精美，如果你不了解其含义，它们在故障排查中的效用将*非常有限*。在撰写本章时，我们面临一个两难境地：市面上有太多有趣的性能计数器，每一个都值得单独用一节来讲解。但是，如果我们那样做，本章将超过 100 页。为了将篇幅控制在合理范围内，本章主要涵盖 Exadata 特有的统计信息，并且只涉及最相关的那些。关于未能纳入本章的事件信息，请关注作者的博客。

以下是一个脚本，用于列出`V$STATNAME`中与存储单元相关的所有统计信息及其统计类别，该类别表明了 Oracle 内核工程师预期使用这些计数器的目的：

```sql
SQL> SELECT
2      name
3    , TRIM(
4        CASE WHEN BITAND(class,  1) =   1 THEN 'USER  ' END ||
5        CASE WHEN BITAND(class,  2) =   2 THEN 'REDO  ' END ||
6        CASE WHEN BITAND(class,  4) =   4 THEN 'ENQ   ' END ||
7        CASE WHEN BITAND(class,  8) =   8 THEN 'CACHE ' END ||
8        CASE WHEN BITAND(class, 16) =  16 THEN 'OSDEP ' END ||
9        CASE WHEN BITAND(class, 32) =  32 THEN 'PARX  ' END ||
10        CASE WHEN BITAND(class, 64) =  64 THEN 'SQLT  ' END ||
11        CASE WHEN BITAND(class,128) = 128 THEN 'DEBUG ' END
12       ) class_name
13  FROM
14      v$statname
15  WHERE
16      name LIKE '%cell%'
17  ORDER BY
18      name
19 /
```

在一个 Oracle 12.1.0.2 系统上，上述查询产生以下输出：

```
NAME                                              CLASS_NAME
------------------------------------------------- ---------------
cell CUs processed for compressed                 SQLT
cell CUs processed for uncompressed               SQLT
cell CUs sent compressed                          SQLT
cell CUs sent head piece                          SQLT
cell CUs sent uncompressed                        SQLT
cell IO uncompressed bytes                        SQLT
cell XT granule bytes requested for predicate offload DEBUG
cell XT granule predicate offload retries         DEBUG
cell XT granules requested for predicate offload  DEBUG
cell blocks helped by commit cache                SQLT
cell blocks helped by minscn optimization         SQLT
cell blocks processed by cache layer              DEBUG
cell blocks processed by data layer               DEBUG
cell blocks processed by index layer              DEBUG
cell blocks processed by txn layer                DEBUG
cell commit cache queries                         SQLT
cell flash cache read hits                        CACHE
cell index scans                                  SQLT
cell interconnect bytes returned by XT smart scan DEBUG
cell logical write IO requests                    USER
cell logical write IO requests eligible for offload USER
cell num block IOs due to a file instant restore in progress SQLT
cell num bytes in block IO during predicate offload SQLT
cell num bytes in passthru during predicate offload SQLT
cell num bytes of IO reissued due to relocation   SQLT
cell num fast response sessions                   SQLT
cell num fast response sessions continuing to smart scan SQLT
cell num smart IO sessions in rdbms block IO due to big payload SQLT
```


