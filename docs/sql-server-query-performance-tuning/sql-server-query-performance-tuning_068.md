# 第 7 章 ■ 分析查询性能

在一个简单的查询中，你可以通过仔细查看查询语句来确定访问了哪些独立的表。随着查询变得越来越复杂，这变得越来越困难。对于存储过程、数据库视图或函数，识别优化器实际访问的所有表变得更加困难。无论查询复杂度如何，你都可以使用 `STATISTICS IO` 来获取此信息。

要在 Management Studio 中开启 `STATISTICS IO`，请导航至“查询” ➤ “查询选项” ➤ “高级” ➤ “设置 STATISTICS IO”。你也可以通过以下程序化的方式获取此信息：

```sql
SET STATISTICS IO ON;

GO

SELECT soh.AccountNumber,
sod.LineTotal,
sod.OrderQty,
sod.UnitPrice,
p.Name
FROM Sales.SalesOrderHeader soh
JOIN Sales.SalesOrderDetail sod
ON soh.SalesOrderID = sod.SalesOrderID
JOIN Production.Product p
ON sod.ProductID = p.ProductID
WHERE sod.SalesOrderID = 71856;

GO

SET STATISTICS IO OFF;

GO
```

如果你运行此查询并查看其执行计划，它由三个聚集索引查找和两个循环联接组成。如果移除 WHERE 子句并再次运行查询，你会得到一系列扫描和一些哈希联接。这是个有趣的事实——但你不知道它如何影响查询的 I/O 使用！你可以像前面那样使用 `SET STATISTICS IO` 来比较优化器使用的两种处理策略之间的查询开销（以逻辑读取计）。

当查询使用哈希联接时，你会得到以下 `STATISTICS IO` 输出：

```
(121317 行受影响)
表 'Workfile'。扫描计数 0，逻辑读取 0...
表 'Worktable'。扫描计数 0，逻辑读取 0...
表 'SalesOrderDetail'。扫描计数 1，逻辑读取 1246...
表 'SalesOrderHeader'。扫描计数 1，逻辑读取 689...
表 'Product'。扫描计数 1，逻辑读取 6...
(1 行受影响)
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

现在，当你重新加入 WHERE 子句以适当地过滤数据时，得到的 `STATISTICS IO` 输出如下：

```
(2 行受影响)
表 'Product'。扫描计数 0，逻辑读取 4...
表 'SalesOrderDetail'。扫描计数 1，逻辑读取 3...
表 'SalesOrderHeader'。扫描计数 0，逻辑读取 3...
(1 行受影响)
```

由于索引查找和循环联接，`SalesOrderDetail` 表的逻辑读取数从 1,246 降至 3。它也没有显著影响 `Product` 表的数据检索开销。

在解释 `STATISTICS IO` 的输出时，你主要参考逻辑读取的数量。当数据未在内存中找到时，物理读取和预读的次数将不为零，但一旦数据被加载到内存中，物理读取和预读往往会变为零。

了解查询使用的所有表及其对应的读取次数还有另一个优势。在表架构（包括索引）或数据未改变的情况下，重新执行相同的查询时，持续时间和 CPU 值可能会波动很大，因为运行在 SQL Server 机器上的基本服务和后台应用程序会影响被观察查询的处理时间。但是，不要忘记逻辑读取并不总是最准确的衡量标准。持续时间和 CPU 绝对有用，并且是任何查询调优的重要部分。

在优化步骤中，你需要一个不波动的开销数值作为参考。对于具有固定表架构和数据的查询，其读取（或逻辑读取）在多次执行之间不会变化。例如，如果你执行前面的 `SELECT` 语句十次，你可能会得到十个不同的持续时间和 CPU 数值，但每次的读取数将保持不变。因此，在优化过程中，你可以参考单个表的读取次数，以确保你确实降低了该表的数据访问开销。只是永远不要假设它是你唯一的衡量标准，甚至不是主要的衡量标准。它只是一个恒定的衡量标准，因此很有用。

尽管逻辑读取的数量也可以从扩展事件中获得，但使用 `STATISTICS IO` 时你还能获得另一个好处。当结合查询使用不同的 `SET` 语句（前面提到过）时，Profiler 或 Server Trace 选项显示的查询逻辑读取数会增加。但 `STATISTICS IO` 显示的逻辑读取数不包括使用 `SET` 语句时访问的额外页。因此，`STATISTICS IO` 提供了逻辑读取数量的一致数值。

## 总结

在本章中，你了解到可以使用扩展事件来识别 SQL 工作负荷中给系统资源造成高压力的查询。收集会话数据可以并且应该使用系统存储过程实现自动化。要即时访问有关正在运行查询的统计信息，请使用 DMV `sys.dm_exec_query_stats`。

你可以使用 Management Studio 进一步分析这些查询，以找出查询处理策略中开销较大的步骤。为了获得更好的性能，在分析查询时考虑执行计划中使用的索引和联接机制非常重要。`SET STATISTICS IO` 提供的单个表的数据检索（或读取）次数有助于你集中关注读取次数最多的表的数据访问机制。你还应该关注最昂贵查询的 CPU 开销和总时间。

一旦你识别出一个昂贵的查询并完成初步分析，下一步应该是针对性能优化该查询。由于索引是最常用的性能调优技术之一，在下一章中，我将深入讨论 SQL Server 中可用的各种索引机制。

[www.it-ebooks.info](http://www.it-ebooks.info/)

