# 第六章：查询性能指标

自计划创建以来使用的 `Total number of writes`。

`Query_hash`
一个可用于识别具有相似逻辑的查询的二进制哈希值。

`Query_plan_hash`
一个可用于识别具有相似逻辑的执行计划的二进制哈希值。

表 6-5 仅是一个示例。有关完整详细信息，请参阅联机丛书。

[www.it-ebooks.info](http://www.it-ebooks.info/)

要筛选从 `sys.dm_exec_query_stats` 返回的信息，你需要将其与其他动态管理函数（如 `sys.dm_exec_sql_text` 或 `sys.dm_query_plan`）联接。`sys.dm_exec_sql_text` 显示与计划关联的查询文本，而 `sys.dm_query_plan` 包含查询的执行计划。一旦与这些其他的 `DMO` 联接，你就可以按你想要查看的数据库或存储过程进行筛选。这些其他的 `DMO` 将在本书的其他章节中详细讨论。在本书的其余部分，我将展示结合使用 `sys.dm_exec_query_stats` 和其他函数的示例。请记住，这些查询依赖于缓存。当某个执行计划从缓存中老化退出时，该信息将会丢失。

## 本章小结

在本章中，你了解到可以使用 `Extended Events` 来识别 SQL 工作负载中对系统资源造成高压的查询。收集会话数据可以且应该使用系统存储过程实现自动化。若要即时访问正在运行的查询的统计信息，请使用 `DMV` `sys.dm_exec_query_stats`。

现在你已经掌握了收集针对系统运行的查询指标的机制，在下一章中，你将探索如何在查询运行时收集其信息，这样你就不必每次运行查询时都求助于这些度量工具。

[www.it-ebooks.info](http://www.it-ebooks.info/)

