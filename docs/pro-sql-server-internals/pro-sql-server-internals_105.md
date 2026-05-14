# 第 28 章 ■ 系统故障排查

```sql
,(ps.total_logical_reads + ps.total_logical_writes) / ps.execution_count as [Avg IO]
,ps.total_logical_reads as [Total Reads], ps.last_logical_reads as [Last Reads]
,ps.total_logical_writes as [Total Writes], ps.last_logical_writes as [Last Writes]
,ps.total_worker_time as [Total Worker Time], ps.last_worker_time as [Last Worker Time]
,ps.total_elapsed_time / 1000 as [Total Elapsed Time]
,ps.last_elapsed_time / 1000 as [Last Elapsed Time]
,ps.last_execution_time as [Last Exec Time]
from
sys.dm_exec_procedure_stats ps with (nolock)
cross apply sys.dm_exec_query_plan(ps.plan_handle) qp
order by
[Avg IO] desc
```

SQL Server 2016 引入了另一个视图 `sys.dm_exec_function_stats`，它允许你跟踪标量用户定义函数的执行统计信息。它适用于 T-SQL、CLR 和 In-Memory OLTP 标量函数；但是，它不捕获表值函数的执行统计信息。

`sys.dm_exec_function_stats` 视图返回的信息与 `sys.dm_exec_procedure_stats` 返回的信息类似。实际上，只要替换列表 28-5 中的 DMV 名称，该代码同样有效。

市场上有大量工具可帮助你自动化数据收集和分析过程，包括 SQL Server 管理数据仓库。它们都能帮助你实现相同的目标，即在系统中找到优化目标。

最后，值得提及的是，数据仓库和报表系统通常遵循不同的规则。在这些系统中，扫描大量数据的 I/O 密集型查询很常见。针对此类系统的性能调优可能需要与 OLTP 环境不同的方法，并且它们通常会导致数据库架构变更，而不是索引调整。

## 并行性

并行性可能是故障排查中最令人困惑的方面之一。它通过 `CXPACKET` 等待类型显现出来，该等待类型常出现在系统顶级等待列表中。`CXPACKET` 等待类型（代表 `Class eXchange`，即类交换）在并行线程等待其他线程完成其执行时发生。

让我们考虑一个简单的例子，并假设我们有一个包含两个线程的并行计划，后面跟着 *交换/重分区流* 操作符。当一个并行线程完成其工作时，它会等待另一个线程完成。等待的线程不消耗任何 CPU 资源；它只是等待，从而产生 `CXPACKET` 等待类型。

`CXPACKET` 等待类型仅仅表明系统中存在并行性，而这通常属于“视情况而定”的范畴。当大型复杂查询利用并行性时，这是有益的，因为它可以显著减少其执行时间。然而，并行性管理和交换操作符总是伴随着开销。例如，如果一个串行计划在单个 CPU 上一秒钟完成，那么使用两个 CPU 的并行计划的执行时间总是会超过 0.5 秒。并行性管理总是需要额外的 CPU 时间。尽管并行计划的响应（总耗时）时间更短，但 CPU 时间总是会大于串行计划的情况。

当大量 OLTP 查询等待可用 CPU 来执行时，你需要避免这种开销。`SOS_SCHEDULER_YIELD` 和 `CXPACKET` 等待比例过高就是这种情况的一个迹象。

一个常见的误解认为，在 OLTP 系统中 `CXPACKET` 等待占比很高的情况下，你应该完全禁用并行性，并将服务器级别的 `MAXDOP` 设置设为 1。然而，这并不是处理并行性等待的正确方法。你需要调查 OLTP 系统中并行性的根本原因，并分析为什么 SQL Server 生成了并行执行计划。在大多数情况下，这是由于复杂和/或未优化的查询导致的。查询优化可以简化执行计划并消除并行性。


