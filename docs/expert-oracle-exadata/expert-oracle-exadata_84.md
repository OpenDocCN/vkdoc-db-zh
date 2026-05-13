# 执行统计信息分析：Flash Cache 优化效果

使用 `session snapper` 或 `mystats.sql` 深入分析执行统计信息，可以看到许多读取请求得到了优化。以下输出取自 `mystats`；与本次讨论无关的统计信息已被移除：

```
------------------------------------------------------------------------------------------
2. Statistics Report
------------------------------------------------------------------------------------------
Type    Statistic Name                                                        Value
------  ----------------------------------------------------------------  ----------------
STAT    cell IO uncompressed bytes                                      13,653,352,448
STAT    cell blocks helped by minscan optimization                          1,666,679
STAT    cell flash cache read hits                                           11,576
STAT    cell physical IO bytes eligible for predicate offload           13,653,336,064
STAT    cell physical IO interconnect bytes returned by smart scan          269,067,552
STAT    cell scans                                                                 1
STAT    physical read IO requests                                            13,048
STAT    physical read bytes                                          13,653,336,064
STAT    physical read requests optimized                                     11,576
STAT    physical read total IO requests                                      13,048
STAT    physical read total bytes                                    13,653,336,064
STAT    physical read total bytes optimized                          12,111,839,232
STAT    physical read total multi block requests                             13,037
STAT    physical reads                                                       1,666,667
STAT    physical reads direct                                                1,666,667
------------------------------------------------------------------------------------------
3. About
------------------------------------------------------------------------------------------
- MyStats v2.01 by Adrian Billington (`http://www.oracle-developer.net`)
- Based on the SNAP_MY_STATS utility by Jonathan Lewis
```

从所有这些数据中可以看出，在大约 13GB 的读取量（使用 13,048 个 I/O 请求）中，很大一部分是由 Flash Cache 满足的，具体为 11,576 个。你可以通过比较`physical read total bytes optimized`和`physical read total bytes`来理解这一点。此外，一旦数据进入 Flash Cache，传统的单块和多块读取也会受益于该段位于更快存储上这一事实，且无需额外代价。

另一方面，如果表无法从 ESFC（Exadata 闪存缓存）中受益，例如`T1_NOCOMPRESS_NOESFC`，情况则有所不同：

```sql
SQL> select count(*) from T1_NOCOMPRESS_NOESFC;

  COUNT(*)
----------
  10000000

Elapsed: 00:00:11.27
```

重复执行该语句不会产生效果：由于此操作被管理性地禁止，因此在 Flash Cache 中不会发生任何缓存。执行计划与首次展示的完全相同：

```
PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------------------
SQL_ID  9gdnw7yk14mpw, child number 0
-------------------------------------
select count(*) from T1_NOCOMPRESS_NOESFC
Plan hash value: 4286875364
-------------------------------------------------------------------------------------------
| Id  | Operation                  | Name                 | Rows  | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT           |                      |       |   452K(100)|          |
|   1 |  SORT AGGREGATE            |                      |     1 |            |          |
|   2 |   TABLE ACCESS STORAGE FULL| T1_NOCOMPRESS_NOESFC |    10M|   452K  (1)| 00:00:18 |
-------------------------------------------------------------------------------------------
14 rows selected.
```

有趣的是，优化器假设扫描的耗时相同，即 18 秒。执行时间的差异可以在执行统计信息中找到：

```
------------------------------------------------------------------------------------------
2. Statistics Report
------------------------------------------------------------------------------------------
Type    Statistic Name                                                        Value
------  ----------------------------------------------------------------  ----------------
STAT    cell IO uncompressed bytes                                    13,653,336,064
STAT    cell physical IO bytes eligible for predicate offload         13,653,336,064
STAT    cell physical IO interconnect bytes returned by smart scan          269,067,184
STAT    cell scans                                                                 1
STAT    physical read IO requests                                            13,046
STAT    physical read bytes                                          13,653,336,064
STAT    physical read total IO requests                                      13,046
STAT    physical read total bytes                                    13,653,336,064
STAT    physical read total multi block requests                             13,037
STAT    physical reads                                                       1,666,667
STAT    physical reads direct                                                1,666,667
------------------------------------------------------------------------------------------
3. About
------------------------------------------------------------------------------------------
- MyStats v2.01 by Adrian Billington (`http://www.oracle-developer.net`)
- Based on the SNAP_MY_STATS utility by Jonathan Lewis
```

Flash Cache 的效果体现在`V$SQL`中与 I/O 相关的统计信息中，你可以查询`physical_read_requests`和`optimized_phy_read_requests`。输出经过重新排列以便阅读：

```sql
SQL> @fsx4.sql

Enter value for sql_text: %esfc_example%
Enter value for sql_id:

SQL_ID        CHILD OFFLOAD IO_SAVED_% AVG_ETIME SQL_TEXT
------------- ------ ------- ---------- ---------- ---------------------------------------
...
aqmusjaqj5yy6      0 Yes          98.03       9.23 select /* esfc_example */ count(*) from ...
g85ux15kbh9hr      0 Yes          98.03       1.23 select /* esfc_example */ count(*) from ...

...

SQL_ID        PHYSICAL_READ_REQUESTS OPTIMIZED_PHY_READ_REQUESTS
------------- ---------------------- ---------------------------
aqmusjaqj5yy6                  13046                           0
g85ux15kbh9hr                  13046                       11576

2 rows selected.
```

值得庆幸的是，随着 Exadata 11.2.3.3 的引入，许多关于将段固定到 Flash Cache 的复杂性问题已得到解决。你甚至无需花费太多时间去思考它。在之前展示的对`user_tables`的查询输出中，你会注意到`cell_flash_cache`的属性是`NONE`和`DEFAULT`，但没有一个被设置为`KEEP`。仅为了 Smart Scan 的数据透明缓存这一点，就值得升级到 11.2.3.3。



#### 压缩

Exadata 的混合列压缩（HCC）在减少 Oracle 数据库内存储数据大小的能力上迈出了一大步。使用 HCC 可实现的压缩比率，将传统的信息生命周期管理观念彻底颠覆。HCC 使得考虑使用压缩而非分层存储或归档与清除策略变得切实可行。由于可以为表的不同分区定义不同的压缩方法，因此将分区与压缩相结合，可以为“归档”数据提供比实际将其从数据库中清除更加强健的解决方案。

然而，您应该记住，HCC 不适用于正在进行 DML 操作的数据。更好的方法是进行数据分区，以便 HCC 可以应用于不再更改的数据。这引出了我们的下一个主题——分区。

#### 分区

分区一直是，并且现在仍然是数据仓库系统的一个非常关键的组件。Exadata 提供的优化并不能消除对精心设计的分区策略的需求。当然，从管理角度来看，基于日期的策略非常有用。能够对较旧的数据使用更激进的压缩通常是一个好方法。但分区消除仍然是您会希望使用的一项技术。而且，当然，存储索引可以与分区良好配合，在额外的列上提供与分区消除类似的行为。

您应该记住，分区的大小会影响 Oracle 决定是否使用智能扫描。对分区对象执行串行扫描时，是否进行直接路径读取的决定是基于单个段（表、分区、子分区）的大小，而不是对象的总体大小。这可能导致对某些分区的扫描被下推执行，而对其他分区的扫描则没有。这对于已被压缩的、较冷的分区尤为重要。考虑一个创建时包含许多随机日期、并按月间隔按日期进行范围分区的表：

```sql
SQL> select segment_name, partition_name, bytes/power(1024,2) m
  2  from user_segments where segment_name= 'SMARTSCANHCC';

SEGMENT_NAME                   PARTITION_NAME                             M
------------------------------ ---------------------------------- ----------
SMARTSCANHCC                   SYS_P9676                                  944
SMARTSCANHCC                   SYS_P9677                                  968
SMARTSCANHCC                   SYS_P9678                                  384

3 rows selected.
```

```sql
SQL> select partition_name, high_value from user_tab_partitions
  2  where table_name = 'SMARTSCANHCC';

PARTITION_NAME                 HIGH_VALUE
------------------------------ --------------------------------------------------
P_START                        TO_DATE(' 1995-01-01 00:00:00', 'SYYYY-MM-DD HH24:
SYS_P9676                      TO_DATE(' 2014-05-01 00:00:00', 'SYYYY-MM-DD HH24:
SYS_P9677                      TO_DATE(' 2014-09-01 00:00:00', 'SYYYY-MM-DD HH24:
SYS_P9678                      TO_DATE(' 2014-10-01 00:00:00', 'SYYYY-MM-DD HH24:
```

```sql
SQL> select partition_name, last_analyzed, num_rows from user_tab_partitions
  2  where table_name = 'SMARTSCANHCC';

PARTITION_NAME                 LAST_ANALYZED          NUM_ROWS
------------------------------ ------------------- ----------
P_START                        2015-03-15:16:31:52          0
SYS_P9676                      2015-03-15:16:31:55     837426
SYS_P9677                      2015-03-15:16:31:58     863109
SYS_P9678                      2015-03-15:16:31:59     335539

4 rows selected.
```

该表是按间隔分区的，分区 `P_START` 是空的，并且——得益于延迟段创建——甚至尚未被创建。分区的大小使得智能扫描得以启用。SQL Monitor 报告已为适应页面进行了精简，仅显示相关信息：

```text
SQL Monitoring Report
SQL Text
------------------------------
select /*+ monitor gather_plan_statistics */ count(*) from smartscanhcc partition (SYS_P9676)

Global Information
------------------------------
Status              :  DONE (ALL ROWS)
Instance ID         :  1
Session             :  MARTIN (591:27854)
SQL ID              :  76yr0u2rhkqq8
SQL Execution ID    :  16777217
Execution Started   :  03/15/2015 16:32:52
First Refresh Time  :  03/15/2015 16:32:52
Last Refresh Time   :  03/15/2015 16:32:53
Duration            :  1s
Module/Action       :  SQL*Plus/-
Service             :  SYS$USERS
Program             :  sqlplus@enkdb03.enkitec.com (TNS V1-V3)
Fetch Calls         :  1

Global Stats
========================================================================================
| Elapsed |   Cpu   |    IO    | Application | Fetch | Buffer | Read | Read  |  Cell   |
```




```
| 耗时(秒) | 耗时(秒) | 等待(秒) | 等待(秒)   | 调用次数 | 获取量 | 请求量 | 字节数 | 下载卸载 |
|========================================================================================|
|    0.33 |    0.13 |     0.21 |        0.00 |     1 |   120K |  940 | 935MB |  97.64% |
|========================================================================================|
```

现在假设该分区经历了维护并进行了 HCC 压缩：

```
SQL> alter table smartscanhcc modify partition SYS_P9676 column store compress for query high;
表已变更。
SQL> alter table smartscanhcc move partition SYS_P9676;
表已变更。
SQL> select segment_name, partition_name, bytes/power(1024,2) m
2  from user_segments where segment_name= 'SMARTSCANHCC';
SEGMENT_NAME                   分区名                            M
------------------------------ ------------------------------ ----------
SMARTSCANHCC                   SYS_P9677                             968
SMARTSCANHCC                   SYS_P9678                             348
SMARTSCANHCC                   SYS_P9676                              16
已选择 3 行。
```

正如预期，这个分区的压缩后大小比之前更小了。如果用户现在对该段执行查询，很可能不会使用智能扫描。实际上，这可以得到证实，例如通过使用 SQL Monitor 报告（或者，如果您没有使用许可，可以查询 `V$SQL`）：

```
SQL 监控报告
SQL 文本
------------------------------
select /*+ monitor gather_plan_statistics */ count(*) from smartscanhcc partition (SYS_P9676)
全局信息
------------------------------
状态              :  已完成 (所有行)
实例 ID         :  1
会话             :  MARTIN (1043:51611)
SQL ID              :  76yr0u2rhkqq8
SQL 执行 ID    :  16777216
开始执行时间   :  06/09/2015 11:02:38
首次刷新时间  :  06/09/2015 11:02:38
最后刷新时间   :  06/09/2015 11:02:38
持续时间            :  .123527 秒
模块/操作       :  SQL*Plus/-
服务             :  SYS$USERS
程序             :  sqlplus@enkdb03.enkitec.com (TNS V1-V3)
提取调用次数         :  1
全局统计
======================================================================================
| 已用时间 |   CPU   |    IO    | 集群  |  其他   | 提取 | 缓冲区 | 读取次数 | 读取量  |
| (秒) | (秒) | 等待(秒) | 等待(秒) | 等待(秒) | 调用次数 |  获取量  |  | 字节数 |
======================================================================================
|    0.12 |    0.02 |     0.09 |     0.00 |     0.01 |     1 |   1244 |   11 |  10MB |
======================================================================================
```

缺失的关于单元下载卸载效率的列，是传统读取和缺乏智能扫描的一个指标。在某些情况下，当分区很小时，缺少智能扫描可能无关紧要。在上面的例子中，数据集从 944MB 减少到了 16MB。如果您看一下已用时间，这些数据可以被非常快速地读取。

### 混合工作负载

存在第三种类型的系统，它是前两种的结合。事实上，可以说前两种（OLTP 和数据仓库）的纯粹形式在现实世界中很少见。有许多系统并不完全符合前面描述的两个主要类别。实际上，大多数系统都表现出两者的特征。例如，考虑这样一个情况：一个“OLTP”系统在白天执行简短、独立的小型事务，而在晚上进行大量报表操作。或者从数据仓库的角度看，您运行大量报表，但有一个计划的 ELT（提取、加载、转换）过程，该过程大量使用合并子句，这当然需要进行查找。

将长时间运行、对吞吐量敏感的查询与快速、对延迟敏感的语句相结合，肯定会带来一些必须处理的额外问题。这类系统的主要问题之一是如何处理索引。


