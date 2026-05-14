# 第 7 章 ■ 查询性能分析

■ **提示** 记得在 `management Studio` 中关闭 `Query ➤ Show execution plan`，否则你将看到图形化的执行计划，而非文本形式的。

## 计划缓存

访问执行计划的最后一个位置是直接从存储它们的内存空间——计划缓存——中读取。SQL Server 提供了动态管理视图和函数来访问这些数据。

要查看缓存中的执行计划列表，请运行以下查询：

```sql
SELECT p.query_plan,
       t.text
FROM   sys.dm_exec_cached_plans r
       CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) p
       CROSS APPLY sys.dm_exec_sql_text(r.plan_handle) t;
```

该查询会返回一个 XML 执行计划链接列表。打开其中任何一个都将显示执行计划。这些执行计划是编译后的计划，但不包含任何执行指标。通过动态管理视图提供的列进行进一步操作，将允许你搜索特定的存储过程或执行计划。

虽然没有运行时数据在某种程度上有所限制，但能够访问执行计划（即使是在查询执行期间）对于从事性能调优工作的人员来说是无价的资源。如前所述，你可能无法在生产环境中执行查询，因此获取到任何计划都是有用的。

## 查询资源成本

尽管查询的执行计划提供了详细的处理策略和各个步骤的预估相对成本，但它并未提供查询在 `CPU` 使用率、磁盘读写或查询持续时间方面的实际成本。在优化查询时，你可能会添加索引来降低某个步骤的相对成本。

这可能会影响到执行计划中依赖该步骤的其他步骤，有时甚至可能修改执行计划本身。因此，如果你只看执行计划，无法确定你的查询优化是使整个查询受益，还是仅仅受益于执行计划中的那一个步骤。你可以通过不同的方式来分析查询的总体成本。

在优化查询时，你应该监控其总体成本。如前所述，你可以使用 `扩展事件` 来监控查询的 `duration`、`cpu`、`reads` 和 `writes` 信息。`扩展事件` 是一种非常高效的指标收集机制。你应该计划利用这一事实，并使用此机制来收集你的查询性能指标。只需明白，收集这些信息会产生大量数据，你必须在系统中找到地方来维护它们。

还有其他收集性能数据的方法，它们比 `扩展事件` 更即时。

## 客户端统计信息

客户端统计信息从你的机器作为服务器客户端的角度捕获执行信息。这意味着记录的任何时间都包含了数据通过网络传输所需的时间，而不仅仅是 SQL Server 机器本身所花费的时间。要使用它们，只需点击 `Query ➤ Include Client Statistics`。现在，每次运行查询时，都会收集一组有限的数据，包括执行时间、受影响的行数、与服务器的往返次数等等。此外，每次查询执行都会在 `客户端统计信息` 选项卡上单独显示，并有一列聚合了多次执行的数据，显示所收集数据的平均值。

这些统计信息还会显示某个时间或计数是否在连续运行之间发生了变化，并以箭头形式显示，如图 7-13 所示。例如，考虑以下查询：

```sql
SELECT TOP 100
       p.*
FROM   Production.Product p;
```

该查询的客户端统计信息应该看起来类似于图 7-13 所示。

`图 7-13. 客户端统计信息`

虽然捕获客户端统计信息可能是一种有用的数据收集方式，但它是一组有限的数据，并且无法显示一次执行与另一次执行有何不同。你甚至可能运行了一个完全不同的查询，其数据也会与其他查询的数据混合在一起，使得平均值失去意义。如果需要，你可以重置客户端统计信息。

选择 `Query` 菜单，然后选择 `Reset Client Statistics` 菜单项。

## 执行时间

`持续时间` 和 `CPU` 都代表了查询的时间因素。要获取有关解析、编译和执行查询所需时间（以毫秒为单位）的详细信息，请使用 `SET STATISTICS TIME`，如下所示：

```sql
SET STATISTICS TIME ON
GO
SELECT soh.AccountNumber,
       sod.LineTotal,
       sod.OrderQty,
       sod.UnitPrice,
       p.Name
FROM   Sales.SalesOrderHeader soh
       JOIN Sales.SalesOrderDetail sod
         ON soh.SalesOrderID = sod.SalesOrderID
       JOIN Production.Product p
         ON sod.ProductID = p.ProductID
WHERE  sod.LineTotal > 1000;
GO
SET STATISTICS TIME OFF
GO
```

前面的 `SELECT` 语句的 `STATISTICS TIME` 输出如下：

```text
SQL Server parse and compile time:
 CPU time = 0 ms, elapsed time = 0 ms.
(32101 row(s) affected)
SQL Server Execution Times:
 CPU time = 328 ms, elapsed time = 643 ms.
SQL Server parse and compile time:
 CPU time = 0 ms, elapsed time = 0 ms.
```

执行时间中的 `CPU time = 328 ms` 部分对应于 `Profiler` 工具和 `Server Trace` 选项提供的 `CPU` 值。同样，对应的 `Elapsed time = 643 ms` 代表了其他机制提供的 `Duration` 值。

0 毫秒的解析和编译时间表明优化器为该查询重用了现有的执行计划，因此无需再次花费时间解析和编译查询。如果查询是第一次执行，则优化器必须先解析查询语法，然后编译它以生成执行计划。这可以通过使用系统调用 `DBCC FREEPROCCACHE` 清除缓存然后重新运行查询来轻松验证。

```text
SQL Server parse and compile time:
 CPU time = 32 ms, elapsed time = 33 ms.
(32101 row(s) affected)
SQL Server Execution Times:
 CPU time = 187 ms, elapsed time = 678 ms.
SQL Server parse and compile time:
 CPU time = 0 ms, elapsed time = 0 ms.
```

这次，SQL Server 花费了 32 毫秒的 `CPU` 时间和总共 33 毫秒的时间来解析和编译查询。

`注意`
除非你准备承担重新编译系统上每个查询的不小成本，否则不应在生产系统上运行 `DBCC FREEPROCCACHE`。在某种程度上，这对你的系统来说，其成本将与重启或 `SQL Server` 实例重新启动相当。

## STATISTICS IO

正如本章前面“识别高成本查询”一节所讨论的，`Reads` 列中的读取次数通常是 `duration`、`cpu`、`reads` 和 `writes` 中最重要的成本因素。查询执行的总读取次数由查询中涉及的所有表的读取次数之和组成。

对各个表执行的读取次数可能有很大差异，这取决于从各个表请求的结果集大小以及可用的索引。

为了减少总读取次数，找出查询中访问的所有表及其相应的读取次数将非常有用。这些详细信息有助于你专注于优化读取次数较多的表的数据访问。每个表的读取次数也有助于你评估为某个表实施的优化步骤对查询中引用的其他表的影响。



