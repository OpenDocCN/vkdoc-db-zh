# 第 26 章 ■ 计划缓存

`sys.dm_exec_query_stats`、`sys.dm_exec_procedure_stats` 和 `sys.dm_exec_trigger_stats` 视图提供了已缓存计划的查询、存储过程和触发器的聚合性能统计信息。只要计划保留在缓存中，它们就会为每个对象的每个缓存计划返回一行。这些视图在性能故障排除中非常有用。我们将在[第 28 章](http://dx.doi.org/10.1007/978-1-4842-1964-5_28)深入讨论它们的用法。

`Sys.dm_exec_query_stats` 在 SQL Server 2005 及更高版本中支持。`Sys.dm_exec_procedure_stats` 和 `Sys.dm_exec_trigger_stats` 是在 SQL Server 2008 中引入的。

**注意** 你可以在 [`technet.microsoft.com/en-us/library/ms188068.aspx`](http://technet.microsoft.com/en-us/library/ms188068.aspx) 找到更多关于执行相关 DMO 的信息。

#### 总结

查询优化是一个昂贵的过程，会在繁忙的系统上增加 CPU 负载。SQL Server 通过在称为计划缓存的特殊内存区域中缓存计划来减少这种负载。它包括 T-SQL 对象的计划，例如存储过程、触发器和用户定义函数；即席查询和批处理；以及其他一些与计划相关的实体。

SQL Server 仅当查询/批处理文本完全匹配时，才会重用即席查询和批处理的计划。此外，不同的 SET 选项和/或对未限定对象的引用可能会阻止计划重用。

缓存即席查询的计划会显著增加计划缓存的内存使用量。如果你使用的是 SQL Server 2008 及更高版本，建议启用服务器端的 `Optimize for ad-hoc workloads` 配置设置。

SQL Server `sniffs parameters` 并生成和缓存对于编译时参数值最优的计划。在数据分布不均匀的情况下，当缓存的计划对于通常提交的参数值不是最优时，这可能会导致性能问题。你可以通过语句级重新编译、`OPTIMIZE FOR` 查询提示或在 SQL Server 2016 中使用 Query Store 来解决此类问题。

你可以直接在查询中指定提示。或者，你可以使用计划指南，它允许你在不更改查询文本的情况下应用提示或强制特定的执行计划。

缓存的计划应该对每个可能的参数组合都有效。这可能导致



