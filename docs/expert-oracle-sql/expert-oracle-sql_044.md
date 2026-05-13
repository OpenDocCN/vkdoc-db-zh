# 如果您无法更改或不愿更改正在分析的代码，可以通过清单 4-2 中的两条语句之一来设置`STATISTICS_LEVEL`参数。

**清单 4-2. 在会话或系统级别启用运行时数据收集**

```sql
ALTER SESSION SET STATISTICS_LEVEL=ALL;
ALTER SYSTEM  SET STATISTICS_LEVEL=ALL;
```

任何 Oracle 性能专家若未能提醒您在系统级别将`STATISTICS_LEVEL`设置为`ALL`可能带来的严重性能问题，都是失职的。主要的担忧是额外的开销可能导致应用程序性能严重下降。然而，许多专家会更进一步，建议您只应在进行短期故障排除时才在系统级别将`STATISTICS_LEVEL`设置为`ALL`，并且永久性地在系统级别将其设置为`ALL`是完全不可取的。我倒不至于如此极端。

收集运行时统计信息的主要开销源于对 Unix 操作系统例程`GETTIMEOFDAY`或其等效函数（在其他平台上）的频繁调用。尽管这个操作系统调用在某些操作系统上可能效率很低，但在其他系统上却可能非常高效。此外，在后续版本的 Oracle 数据库中，采样频率（由隐藏参数`_rowsource_statistics_sampfreq`控制）已经降低。我最近在一个负载很重的 Solaris/SPARC-64 生产系统上将`STATISTICS_LEVEL`在系统级别设置为`ALL`，发现在一个两小时的批处理运行中，其对总耗时没有可测量的影响。

即使您永久性地在系统范围内将`STATISTICS_LEVEL`设置为`ALL`，运行时引擎统计信息也不会保存在 AWR 中。为了解决这个问题，我会运行自己的计划任务来捕获这些数据。实现此功能的代码相当长，因此我未在此处包含，但它可以在下载的资料中找到。

您可能考虑在系统级别永久设置`STATISTICS_LEVEL`为`ALL`的原因，是为了在事后处理意外的生产性能问题；如果一个 SQL 语句有时运行良好，有时却不行，您可能无法随意重现该问题，而拥有来自实际事件的数据可能是无价的。

最后需要提醒您注意的是，在版本 10.1.0.2 到 11.1.0.7 之间，您可能会遇到错误 8289729，该错误会导致 SYSAUX 表空间大小急剧增加。请查阅 Oracle 支持说明 874518.1 或 Martin Widlake 的这篇博客（`http://mwidlake.wordpress.com/2011/06/02/why-is-my-sysaux-tablespace-so-big-statistics_levelall/`），但也不必太过沮丧，因为在我的案例中，这种额外的数据收集并未损害性能。

## 启用 SQL 追踪

有几种方法可以启用 SQL 追踪。最古老也是最广为人知的启用 SQL 追踪的方法是设置 10046 事件。因此您经常会听到人们将 SQL 追踪称为“10046 追踪”。然而，现今启用 SQL 追踪的首选方式是使用例程`DBMS_MONITOR.SESSION_TRACE_ENABLE`，如何调用它的示例在 PL/SQL 包和类型参考手册中提供。更多信息可以在性能调优指南（12cR1 之前）或 SQL 调优指南（12cR1 及以后版本）中找到。

除了启用从运行时引擎收集统计信息外，SQL 追踪还会生成一个文本文件，其中包含关于会话中正在发生的事情的大量附加信息。其中包括，此追踪文件可以帮助您查看：

*   运行了哪些 SQL 语句以及它们运行的频率。
*   运行了哪些递归 SQL 语句（触发器或函数中的 SQL 语句）。
*   打开游标（执行 SQL 语句）花费了多少时间，从游标中提取数据花费了多少时间，以及在所谓的空闲等待事件中花费了多少时间（这意味着会话正在数据库之外执行应用程序逻辑）。
*   关于单个等待事件的详细信息，例如单个数据库文件多块读等待所读取的确切块数。

尽管对于许多性能专家来说，SQL 追踪是主要的诊断工具，但我几乎从不使用它来调整个别的 SQL 语句！对于绝大多数 SQL 调优任务，运行时引擎统计信息已经绰绰有余，并且您很快就会看到，有一些数据字典视图可以非常方便地展示这些数据。

## 显示操作级别数据

一旦通过上述方法之一收集了运行时引擎统计信息，接下来的任务就是显示它们。让我们看看两种方法：

*   使用`DBMS_XPLAN.DISPLAY_CURSOR`
*   使用`V$SQL_PLAN_STATISTICS_ALL`

### 使用 DBMS_XPLAN.DISPLAY_CURSOR 显示运行时引擎统计信息

通过使用`DBMS_XPLAN.DISPLAY_CURSOR`的非默认参数，您可以在操作表中看到运行时引擎统计信息。清单 4-3 展示了如何在运行清单 4-1 中的语句后立即执行此操作。

**清单 4-3. 显示运行时引擎统计信息**

```sql
SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'ALLSTATS LAST'));
```

```
| Id  | Operation                       | Name           | Starts | E-Rows | A-Rows |
|   0 | SELECT STATEMENT                |                |      1 |        |      1 |
|   1 |  SORT AGGREGATE                 |                |      1 |      1 |      1 |
|*  2 |   HASH JOIN                     |                |      1 |   7956 |   1385 |
|*  3 |    TABLE ACCESS FULL            | CUSTOMERS      |      1 |     61 |     80 |
|   4 |    PARTITION RANGE ALL          |                |      1 |    918K|    918K|
|   5 |     BITMAP CONVERSION TO ROWIDS |                |     28 |    918K|    918K|
|   6 |      BITMAP INDEX FAST FULL SCAN| SALES_CUST_BIX |     28 |        |  35808 |
```

清单 4-3 经过了一些编辑，移除了操作表中的各种标题和部分列。再次说明，这只是为了让输出在页面上可读。

`DBMS_XPLAN.DISPLAY_CURSOR`有一个`FORMAT`参数，要获取上次执行该语句的所有运行时执行统计信息，方法是指定`'ALLSTATS LAST'`的值；您可以查阅文档了解更多变体。生成的输出相当有启发性：

*   首先，查看`Starts`列。这告诉您操作 0 到 4 运行了一次，但操作 5 运行了 28 次。这是因为`SALES`表中有 28 个分区，需要为每个分区运行一次`TABLE ACCESS FULL`操作。
*   `E-Rows`和`A-Rows`列分别是估计的和实际的行数，您可以看到最大的差异出现在操作 2，其估计值比实际值高出五倍多。
*   `A-Time`列中的实际耗时比`DBMS_XPLAN.DISPLAY`显示的估计值具有更高的准确性。

重要的是要意识到，所有运行时引擎统计信息反映的是该操作所有执行的累积结果。因此，操作 5 在扫描完所有 28 个分区后返回了 918K 行，0.46 秒的耗时也反映了扫描整个`SALES`表的累积时间。

**语句的所有执行 vs. 操作的所有执行**



我提到过 `ALLSTATS LAST` 格式会显示 SQL 语句**最后一次**运行的统计信息。然而，我也指出过 `A-Time` 列反映的是操作 5 所有执行的累计时间。这两种说法都是正确的，因为操作 5 的 28 次执行都包含在 SQL 语句的**最后一次**执行中。如果我指定了 `ALLSTATS ALL` 格式，并且该语句被多次运行，那么操作 5 的 `Starts` 值将是 28 的倍数。

`A-Time` 列反映了执行该操作及其所有子操作所花费的时间。例如，查看操作 2 的 `HASH` 连接及其两个子操作：

*   操作 2 的第一个子操作是操作 3 中对 `CUSTOMERS` 表的全表扫描。报告显示操作 2 耗时 0.2 秒。
*   ID 为 4 的 `PARTITION RANGE ALL` 操作是操作 2 的第二个子操作，显然耗时 0.66 秒。
*   如果我们将 0.2 和 0.66 相加，操作 2 的子操作总耗时 0.86 秒。
*   用操作 2 报告的 1.22 减去 0.86，可以看到 `HASH JOIN` 本身耗时 0.36 秒。

## 使用 `V$SQL_PLAN_STATISTICS_ALL` 显示运行时引擎统计信息

虽然我发现 `DBMS_XPLAN` 函数的输出在分析执行计划并理解其作用方面非常出色，但在进行运行时统计信息的详细分析时，我觉得 `DBMS_XPLAN` 的输出格式非常难以处理。主要问题是列数众多，即使假设你有一个能显示所有列的电子页面，眼睛也很难左右浏览。

视图 `V$SQL_PLAN_STATISTICS_ALL` 返回了 `DBMS_XPLAN.DISPLAY_CURSOR` 所显示的所有运行时引擎统计信息。当我进行任何类型的运行时引擎数据分析时，我几乎总是使用工具在网格中显示该视图的输出。即使你没有像 SQL Developer 这样的图形工具，也可以总是从该视图中以逗号分隔值（CSV）格式获取数据，然后放入电子表格中；这同样有效。

以网格形式显示统计信息之所以有用，是因为你可以按需排列列的顺序，并可以暂时隐藏当前不关注的列。还有一些额外的技巧可以让阅读这些数据更容易：

*   `V$SQL_PLAN_STATISTICS_ALL` 包含语句最后一次执行和所有执行的所有统计信息。包含最后一次执行信息的列以 `LAST_` 为前缀。因为这些信息可能就是你所需要的，所以大多数时候你可能不希望选择 `STARTS`、`OUTPUT_ROWS`、`CR_BUFFER_GETS`、`CU_BUFFER_GETS`、`DISK_READS`、`DISK_WRITES` 和 `ELAPSED_TIME` 这些列，因为它们反映了语句的所有执行情况，会引起混淆。
*   你可以使用 `DEPTH` 列对 `OPERATION` 列进行缩进。
*   你可以为可能引起混淆的列添加标签以避免混淆。
*   将 `LAST_ELAPSED_TIME` 除以 1000000 可将微秒转换为秒，这对我们大多数人来说更具可读性。

清单 4-4 展示了一个针对 `V$SQL_PLAN_STATISTICS_ALL` 的简单查询，说明了这些要点。

### 清单 4-4. 从 `V$SQL_PLAN_STATISTICS_ALL` 中选择部分列

```sql
 SELECT DEPTH
        ,LPAD (' ', DEPTH) || operation operation
        ,options
        ,object_name
        ,time "EST TIME (Secs)"
        ,last_elapsed_time / 1000000 "ACTUAL TIME (Secs)"
        ,CARDINALITY "EST ROWS"
        ,last_output_rows "Actual Rows"
    FROM v$sql_plan_statistics_all
   WHERE sql_id = '4d133k9p6xbny' AND child_number = 0
ORDER BY id;
```

这个查询只选择了几个列，但在实践中你会选择更多的列。你感兴趣的语句的 `SQL_ID` 和子编号可以使用第 1 章中描述的技术来获取。

## 使用 Snapper 显示会话级统计信息

虽然对于运行时性能分析最重要的统计信息位于 `V$SQL_PLAN_STATISTICS_ALL` 中，但默认情况下 Oracle 会持续收集大量的统计信息。出于某种难以理解的原因，Oracle 只给客户提供了一堆难以理解的视图来查看这些统计信息，我们大多数人因此灰心，不愿尝试使用它们。幸运的是，“大多数”并不等于“全部”，已经有许多人找到了让这些信息更易于理解的方法。最早的一个用于以可读方式显示会话统计信息的脚本来自 Tom Kyte，可以在 `http://asktom.oracle.com/runstats.html` 找到。在我看来，查看此类信息的最佳工具来自 Oracle 专家 Tanel Poder，他发布了一个通用脚本，可以以易于阅读的格式提取和显示众多运行时统计信息。他宝贵的 Snapper 脚本的最新版本可在 `http://blog.tanelpoder.com/` 获取，我鼓励你下载这个免费且简单易用的工具。

当会话在 CPU 上“卡住”时，Snapper 特别有用。每次我在这种情况下使用 Snapper 时，都能立即看到问题所在。清单 4-5 展示了一个实际 Snapper 运行结果的摘录，该运行针对一个持续占用 CPU 的长时间运行查询。

### 清单 4-5. snapper 突出显示的争用问题

```
TYPE, STATISTIC                                                 ,     HDELTA,

STAT, session logical reads                                     ,      1.06M,
 STAT, concurrency wait time                                     ,          1,
 STAT, consistent gets                                           ,      1.06M,
 STAT, consistent gets from cache                                ,      1.06M,
 STAT, consistent gets - examination                             ,      1.06M,
 STAT, consistent changes                                        ,      1.06M,
 STAT, free buffer requested                                     ,         17,
 STAT, hot buffers moved to head of LRU                          ,         27,
 STAT, free buffer inspected                                     ,         56,
 STAT, CR blocks created                                         ,         17,
 STAT, calls to kcmgas                                           ,         17,
 STAT, data blocks consistent reads - undo records applied       ,      1.06M,
 STAT, rollbacks only - consistent read gets                     ,         17,
 TIME, DB CPU                                                    ,      5.82s,
 TIME, sql execute elapsed time                                  ,      5.71s,
 TIME, DB time                                                   ,      5.82s,
 WAIT, latch: cache buffers chains                               ,    11.36ms,
--  End of Stats snap 1, end=2013-01-31 02:29:11, seconds=10
```

当我第一次看到清单 4-5 中的输出时，最让我震惊的是在 10 秒的快照中应用了 106 万条撤销记录！该查询与一批在查询开始后不久启动的插入会话发生冲突，查询正忙于通过撤销这些插入来制作一致读副本。我立即看出，杀死并重启该查询将使其快速完成。有人问我是否需要在重启前收集统计信息。我能够自信地表示，那没有必要。所有的猜测都被消除了！

## SQL 性能监视器




