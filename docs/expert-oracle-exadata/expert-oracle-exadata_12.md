# 创建 HCC 压缩表时会发生什么？

HCC 的跟踪和检测代码内嵌在 `ADVCMP` 组件中。通过使用 Oracle 的新型统一跟踪设施（`UTS`）进行跟踪，你实际上可以看到在压缩表时发生了什么。为 HCC 启用 `UTS` 跟踪的语法记录在 `ORADEBUG` 中。调用 `ORADEBUG` 允许你查看可以被跟踪的内容：

```
SQL> oradebug doc component ADVCMP

Components in library ADVCMP:
--------------------------------------------------------------------------
ADVCMP_MAIN           Archive Compression (kdz)
ADVCMP_COMP           Archive Compression: Compression (kdzc, kdzh, kdza)
ADVCMP_DECOMP         Archive Compression: Decompression (kdzd, kdzs)
ADVCMP_DECOMP_HPK     Archive Compression: HPK (kdzk)
ADVCMP_DECOMP_PCODE   Archive Compression: Pcode (kdp)
```

尽管该组件在 11g Release 2 中已有文档记录，但本章的示例来自 12.1.0.2。有趣的是，不带参数的 `ORADEBUG doc component` 命令会向你显示代码位置！从前面的代码可以推断，Oracle 中的 `KDZ*` 例程似乎与 `HCC` 相关。这在查看跟踪文件时会有所帮助。

考虑以下启用压缩跟踪的语句（警告：这可能生成数 GB 的跟踪数据——切勿在生产环境中运行，仅在专用实验室环境中运行。否则，你可能会填满 `/u01` 并导致严重问题）：

```
SQL> alter session set events 'trace[ADVCMP_MAIN.*] disk=high';
```

如果你对 `UTS` 语法感到好奇，只需运行 `oradebug doc event`，正如合著者 Tanel Poder 在他的博客中描述的那样。启用跟踪后，你就可以开始压缩。使用 `alter table ... move` 或 `create table ... column store compress for ...` 语法来开始操作和跟踪。

```
alter session set tracefile_identifier = 't2ql';
alter session set events 'trace[ADVCMP_MAIN.*] disk=high';

create table t2_ql
column store compress for query low
as select * from t2;

alter session set events 'trace[ADVCMP_MAIN.*] off';
```

> **注意**
>
> 要注意，该跟踪信息非常非常详细，很容易生成数 GB 的跟踪数据，填满数据库软件的挂载点，从而导致所有数据库停止运行！再次强调，切勿在专用实验室环境之外运行此类跟踪。

在 Oracle 实际开始压缩之前，它会分析传入的数据以找出压缩列的最佳方式。跟踪将输出类似这样的行：

```
kdzcinit(): ctx: 0x7f9f4b7a1868  actx: (nil)  zca: (nil)  ulevel: 1  ncols: 8 totalcols: 8
kdzainit(): ctx: 7f9f4b7a2d48 ulevel 1 amt 1048576 row 4096 min 5
kdzalcd(): objn: 61487 ulevel: 1
kdzalcd(): topalgo: -1 err: 100
kdza_init_eq(): objn: 61487  ulevel: 1  enqueue state:0
kdzhDecideAlignment(): pnum: 0 min_target_size: 32000 max_target_size: 32000
alignment_target_size: 128000 ksepec: 0 postallocmode: 0 hcc_flags: 0
kdzh_datasize(): freesz: 0 blkdtsz: 8168 flag: 1 initrans: 3 dbidl: 8050 dbhsz: 22
dbhszz: 14 drhsz: 9 maxmult: 140737069841520
kdzh_datasize(): pnum: 0 ds: 8016 bs: 8192 ov: 20 alloc_num: 7 min_targetsz:
32000 max_targetsz: 32000 maxunitsz: 40000 delvec_size: 7954
kdzhbirc(): pnum: 0 buffer 1 rows soff: 0
...
```

看起来对 `KDZA*` 的调用为对象 `61487`（表 `T2_QL`）初始化了数据分析器。跟踪中的 `ULEVEL` 可能与压缩算法 `1`（`Query Low`，你将在块转储中看到）有关。与 `kdzh_datasize()` 相关的输出看起来与 `CU` 头和压缩信息有关。接下来的几行涉及到填充一个缓冲区，需要获取关于待压缩数据的概念。一旦第一个缓冲区被填满，分析器就会创建一个新的 `CU`。对于更高的压缩级别，Oracle 会尝试在压缩前对数据进行预排序。这在所有情况下可能都没有意义——如果分析器检测到这种情况，它将跳过该列的排序。如果列置换能带来整体压缩效益，Oracle 也可以执行列置换。

此操作的结果呈现在分析器上下文中：

```
Compression Analyzer Context Dump Begin
---------------------------------------
ctx: 0x7f9f4b7a2d48  objn: 61487
Number of columns: 8
ulevel: 1
ilevel: 4645
Top algorithm: 0
Sort column: None
Total Output Size: 109435
Total Input Size: 396668
Grouping: Column-major, columns separate

Column Permutation Information
------------------------------
Columns not permuted

Total Number of rows/values  : 4096

Column Information
------------------
Col     Algo    InBytes/Row      OutBytes/Row    Ratio   Type    Name     Type Name
---     ----    -----------      ------------    -----   ----    ----     ---------
0       1025            4.0              3.01      1.3   2       NULL     NUMBER
1       257            11.0              4.11      2.7   1       NULL     CHAR
2       1025            4.0              3.01      1.3   2       NULL     NUMBER
3       1025            3.0              0.05     60.5   2       NULL     NUMBER
4       1025            5.0              4.08      1.2   2       NULL     NUMBER
5       1025            4.9              4.01      1.2   2       NULL     NUMBER
6       257             4.0              3.07      1.3   2       NULL     NUMBER
7       257            61.0              5.41     11.3   1       NULL     CHAR
Total                                 96.8         26.75       3.6

Column Metrics Information
--------------------------
Col     Unique  Repeat   AvgRun  DUnique DRepeat AvgDRun
---     ------  ------   ------  ------- ------- -------
0         4096       0      1.0        0       0     0.0
1         4096       0      1.0        0       0     0.0
2         4096       0      1.0        0       0     0.0
3           47      46     95.3        0       0     0.0
4         4082      14      1.0        0       0     0.0
5         4061      35      1.0        0       0     0.0
6         3646     408      1.0        0       0     0.0
7         3841     241      1.0        0       0     0.0
Compression Analyzer Context Dump End
-------------------------------------
```

分析的结果随后被存储在数据字典中以供重用。

> **注意**
>
> 当分析已经执行过时（例如，在向 `HCC` 压缩段插入数据时），则不需要此步骤。

不幸的是，一旦分析器信息被存储后，似乎没有简单的方法来提取它。在写入 `CU` 之前，你可以在 `kdzhailseb()` 中看到很多关于它的有趣信息，然后才会为表转储 `CU`。跟踪文件中引用的函数在 `ORADEBUG` 短堆栈中也很容易看到。跟踪也有助于确认 Oracle 尝试创建的多种 `CU` 大小，正如在 `kdzhDecideAlignment()` 中发现的那样。表 3-2 列出了针对所有四种压缩算法使用不同建表语句的结果。

**表 3-2.** Oracle 12.1.0.2 的目标 `CU` 大小及其对齐目标

| 压缩类型 | `min_target_size` | `max_target_size` |
| --- | --- | --- |
| `Query Low` | 32000 | 32000 |
| `Query High` | 32000 | 64768 |
| `Archive Low` | 32000 | 261376 |
| `Archive High` | 261376 | 261376 |

请记住，`Query High` 和 `Archive Low` 使用的是相同的算法（在撰写本文时是 `GZIP`）。我们做出的另一个观察是，实际的 `CU` 大小可能会变化，但 `Archive High` 除外，它的大小似乎是固定的。


## HCC 性能

有三个与表压缩相关的性能领域需要考虑。第一个是加载性能，它解决加载数据需要多长时间的问题。由于压缩总是在计算节点上、在直接路径操作期间发生，因此衡量压缩对加载的影响至关重要。第二个关注领域是查询性能，即解压缩和其他副作用对查询压缩数据的影响。第三个关注领域是 DML 性能，即压缩算法对其他 DML 活动（如更新和删除）的影响。

#### 加载性能

正如您可能预料的那样，加载时间往往随着应用的压缩量而增加。正如俗话所说，“天下没有免费的午餐。” 压缩在计算上是昂贵的——这一点毋庸置疑。压缩算法越激进，您将使用的 CPU 周期就越多。Oracle 当前实现的算法范围从 LZO 到 GZIP 和 BZIP2，其中 LZO 产生最低的压缩比但压缩时间最短。BZIP2 可能为您提供最佳压缩，但代价是巨大的 CPU 使用率。有一种观点认为，使用 `ARCHIVE HIGH` 压缩的数据在重复查询之前最好先解压缩到 `ARCHIVE LOW`。压缩比在很大程度上取决于数据——一系列重复十亿次的字符“c”可以用很少的数据表示，而已经压缩的数据（如 JPEG 图像）则无法进一步压缩。

在介绍之后，是时候看一个例子了。表 T3 中的数据是相当随机的。为了增加数据量，它已被复制多次（`insert into t3 select * from t3`，总大小为 11.523 GB）。然后，该表已使用 BASIC、OLTP 和所有 HCC 压缩算法进行压缩。结果报告在表 3-3 中。

表 3-3.

压缩对示例表的影响。加载是串行执行的

| 表名 | 压缩方式 | 压缩比 | 加载时间 | 加载时间比 |
| --- | --- | --- | --- | --- |
| T3 | 无 | 1.0 | 00:01:16.91 | 1.0 |
| T3_BASIC | BASIC | 1.1 | 00:03:02.98 | 2.4 |
| T3_OLTP | OLTP | 1.0 | 00:03:07.22 | 2.4 |
| T3_QL | Query Low | 6.7 | 00:01:56.98 | 1.5 |
| T3_QH | Query High | 15.2 | 00:04:23.41 | 3.4 |
| T3_AL | Archive Low | 15.5 | 00:04:57.51 | 3.9 |
| T3_AH | Archive High | 20.6 | 00:17:07.17 | 13.4 |

如您所见，对于这个特定数据集，Query High 和 Archive Low 产生几乎相同的压缩比。加载数据大约多花了 30 秒。启用压缩加载数据肯定会带来影响——创建表的时间大约是没有压缩时的 2.5 到 4 倍。有趣的是，BASIC 和 OLTP 压缩在使用的存储空间上几乎没有任何区别。并非每个数据集都同样可压缩。这里的时间是从 X2 系统测得的；当前的 Exadata 系统拥有更快、更高效的 CPU。

一旦启用 HCC，压缩比会增加几个数量级。加载时间随着存储节省而增加，但 Archive High 除外，我们稍后会回到这一点。然而，在 Exadata 中，串行加载数据是相当慢的，通过并行化加载操作可以显著提高性能。

注意

您可以阅读更多关于并行操作的内容，请参见第 6 章。

需要注意的是，尽管 Exadata 是一个非常强大的平台，但您不应用并行查询和并行 DML 使系统过载！另请注意，您不必插入到 HCC 压缩段中。根据您的策略，您可以在后续阶段引入压缩。

#### 查询性能

加载时间只是第一个重要的性能指标。数据加载后，它也需要在合理的时间内可访问！大多数系统加载数据一次，读取数据的次数要多得多。当涉及到压缩时，查询性能好坏参半。根据查询的类型，压缩可以加速查询，也可以减慢查询速度。解压缩会带来额外的 CPU 使用开销，特别是如果必须在计算节点上执行。使用更多 CPU 来解压缩数据的补偿在于磁盘访问。压缩数据意味着需要从磁盘物理读取的块更少。如果查询有利于 HCC 压缩行的列式格式，则可能获得额外的收益。这些加起来通常超过解压缩的额外成本。

HCC 压缩数据主要有两种访问模式：智能扫描（Smart Scan）或传统的块 I/O。这些的重要性在于 HCC 数据的解压缩方式。对于非智能扫描，解压缩将不得不在计算节点上执行。在此背景下一个有趣的问题是，通过其 `ROWID` 检索一行需要多长时间。逻辑上，CU 越大，解压缩所需的时间越长。您可能在本章前面的部分问过自己，Query High 和 Archive Low 之间的区别是什么，除了 CU 大小之外。关键在于，Archive High 提供稍好的压缩和卸载的全表扫描。使用 Archive Low 进行的 `ROWID` 访问（通常不是智能扫描）由于较大的 CU 大小而本质上稍差。与许多架构决策一样，这再次归结为“了解您的数据”。

另一个重要方面与生命周期管理相关。想象这样一种情况：分区表在其原生形式下符合智能扫描的条件。报告和任何依赖于卸载扫描的数据访问都将以预期的速度执行。在分区生命周期内引入压缩完全有可能将其大小减小到该段不再符合智能扫描条件的程度。这可能会对性能产生影响，但不一定如此。我们已经看到许多案例，其中存储节省加上列式格式远远超过了缺失的智能扫描。

回到另一组已进行压缩的演示表，我们想演示不同类型的 I/O 及其对查询性能的影响。本节中使用的演示表包含 1.28 亿行，并具有以下大小：

```sql
SQL> select owner,segment_name,segment_type,bytes/power(1024,2) m, blocks
2   from dba_segments
3   where segment_name like 'T3%' and owner = 'MARTIN'
4   order by m;

OWNER                SEGMENT_NAME                   SEGMENT_TYPE                M      BLOCKS
-------------------- ------------------------------ ------------------ ---------- ----------
MARTIN               T3_AH                          TABLE                    2240      286720
MARTIN               T3_QH                          TABLE                    3136      401408
MARTIN               T3_AL                          TABLE                    3136      401408
MARTIN               T3_QL                          TABLE                    6912      884736
MARTIN               T3_BASIC                       TABLE                   71488     9150464
MARTIN               T3_OLTP                        TABLE                   80064    10248192
MARTIN               T3                             TABLE                80100.75    10252896

8 rows selected.
```

第一步，使用对 `DBMS_STATS` 的调用计算所有表的统计信息，如下所示：

```sql
SQL> exec dbms_stats.gather_table_stats(ownname => user, tabname => 'T3', -
2   method_opt=>'for all columns size 1', degree=>4)
```

每个表所用的时间列在表 3-4 中。

表 3-4.

### DML 性能


