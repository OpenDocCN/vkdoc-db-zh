# 第 28 章 ■ 系统故障排除

## 高 CPU 负载

查询涉及大量连接和非可搜索参数（SARGable）谓词，和/或连接条件及 `WHERE` 子句中存在函数。SQL Server 在基数估计期间必须应用启发式方法，这可能导致针对实际数据产生不正确的结果。

您应该监控系统中内存授予的情况。即使是少量的 `RESOURCE_SEMAPHORE` 等待也可能表明严重的性能问题。这可能是内存压力以及查询极低效和优化不良的迹象。

您可以通过查看 `SQL Server:Memory Manager` 对象中的 `memory grants pending` 性能计数器来确认问题。此计数器显示等待内存授予的查询数量。理想情况下，计数器值应始终为零。

`sys.dm_exec_query_resource_semaphores` 视图显示了两个 Resource Semaphore 队列的统计信息，包括已授予和可用的工作空间内存、等待队列中的查询数量以及其他几个参数。您还可以查看 `sys.dm_exec_query_memory_grants` 视图，它提供有关内存授予请求（包括挂起和未完成）的信息。清单 28-6 说明了如何获取相关信息，以及查询文本和执行计划。

***清单 28-6.*** 从 `sys.dm_exec_query_memory_grants` 视图获取查询信息

```sql
select
mg.session_id, t.text as [SQL], qp.query_plan as [Plan], mg.is_small, mg.dop
,mg.query_cost, mg.request_time, mg.required_memory_kb, mg.requested_memory_kb
,mg.wait_time_ms, mg.grant_time, mg.granted_memory_kb, mg.used_memory_kb
,mg.max_used_memory_kb
from
sys.dm_exec_query_memory_grants mg with (nolock)
cross apply sys.dm_exec_sql_text(mg.sql_handle) t
cross apply sys.dm_exec_query_plan(mg.plan_handle) as qp
```

SQL Server 2012 SP3、SQL Server 2014 SP2 和 SQL Server 2016 有多项增强功能，可简化内存授予的故障排查。`sys.dm_exec_query_stats` 视图在输出列中提供了内存授予相关的统计信息。还有一个 `query_memory_grant_usage` 扩展事件，可用于实时跟踪内存分配。最后，SQL Server 2016 中的 Query Store 会收集内存授予指标以及其他参数。

如果您运行的是旧版本的 SQL Server，可以从缓存的执行计划中获取有关内存授予的信息。与其他指标一样，那里的内存授予信息缺乏实际的执行统计信息，并且它还显示了串行执行计划所需的内存授予请求信息，不涉及并行开销。

清单 28-7 显示了如何使用 `sys.dm_exec_cached_plans` 视图从缓存的执行计划中获取内存授予信息。或者，您也可以使用 `sys.dm_exec_query_stats` 视图和 `sys.dm_exec_query_plan` 函数获取类似信息。

***清单 28-7.*** 从缓存计划获取内存授予信息

```sql
;with xmlnamespaces(default 'http://schemas.microsoft.com/sqlserver/2004/07/showplan')
,Statements(PlanHandle, ObjType, UseCount, StmtSimple)
as
(
select cp.plan_handle, cp.objtype, cp.usecounts, nodes.stmt.query('.')
from sys.dm_exec_cached_plans cp with (nolock)
cross apply sys.dm_exec_query_plan(cp.plan_handle) qp
cross apply qp.query_plan.nodes('//StmtSimple') nodes(stmt)
)
select top 50
s.PlanHandle, s.ObjType, s.UseCount
,p.qp.value('@CachedPlanSize','int') as CachedPlanSize
,mg.mg.value('@SerialRequiredMemory','int') as [SerialRequiredMemory KB]
,mg.mg.value('@SerialDesiredMemory','int') as [SerialDesiredMemory KB]
from Statements s
cross apply s.StmtSimple.nodes('.//QueryPlan') p(qp)
cross apply p.qp.nodes('.//MemoryGrantInfo') mg(mg)
order by
mg.mg.value('@SerialRequiredMemory','int') desc
```

您可以通过使用 `MAX_GRANT_PERCENT` 查询提示（在 SQL Server 2012 SP3、SQL Server 2014 SP2 和 SQL Server 2016 中支持）或通过限制 Resource Governor 工作负载组中的 `REQUEST_MAX_MEMORY_GRANT_PERCENT` 设置来限制内存授予的最大大小。然而，最好的方法是通过简化和优化查询，以从执行计划中移除内存密集型操作符，如哈希、排序，有时还包括并行。通常可以通过索引调优和查询重构来实现。

`CXMEMTHREAD` 是另一个您可能在系统中遇到的与内存相关的等待类型。当多个线程尝试同时从未分配的内存堆中分配内存时，就会发生这些等待。您经常可以在具有大量即席查询的系统中观察到这些等待的百分比很高，因为 SQL Server 不断分配和释放计划缓存内存。如果计划缓存内存分配是根本原因，启用 `Optimize for Ad-hoc Workloads` 配置设置可以帮助解决此问题。

SQL Server 有三类内存对象。其中一些是在服务器范围内全局创建的。其他则按 NUMA 节点或按 CPU 进行分区。在 SQL Server 2016 之前的版本中，您可以使用启动跟踪标志 `T8048` 将每个 NUMA 节点的分区切换为每个 CPU 的分区，这有助于减少 `CXMEMTHREAD` 等待，但代价是额外的内存使用。另一方面，SQL Server 2016 在检测到争用时，会自动将这种分区提升到每个 NUMA 级别，然后是每个 CPU 级别，因此不需要 `T8048`。

■ **注意** 您可以阅读有关[非统一内存访问 (NUMA) 架构](http://technet.microsoft.com/en-us/library/ms178144.aspx)的更多信息。

清单 28-8 向您展示了如何分析内存对象的内存分配。如果您发现主要的内存消费者是按 NUMA 节点分区的，并且在系统中可以看到很大比例的 `CXMEMTHREAD` 等待，您可以考虑应用 `T8048` 跟踪标志。这在每个 NUMA 节点有超过八个 CPU 的服务器场景中尤其重要，因为旧版本的 SQL Server 存在每个 NUMA 节点内存对象可伸缩性的已知问题。正如我已经提到的，此跟踪标志在 SQL Server 2016 中不需要。

***清单 28-8.*** 分析内存对象分区和内存使用情况

```sql
select type, pages_in_bytes
,case
when (creation_options & 0x20 = 0x20)
then 'Global PMO. Cannot be partitioned by CPU/NUMA Node. T8048 not applicable.'
when (creation_options & 0x40 = 0x40)
then 'Partitioned by CPU. T8048 not applicable.'
when (creation_options & 0x80 = 0x80)
then 'Partitioned by Node. Use T8048 to further partition by CPU.'
else 'Unknown'
end as [Partitioning Type]
from sys.dm_os_memory_objects
order by pages_in_bytes desc
```

■ **注意** 您可以阅读 Microsoft CSS 团队发布的一篇文章，该文章解释了[如何调试 `CXMEMTHREAD` 等待类型](http://blogs.msdn.com/b/psssql/archive/2012/12/20/how-it-works-cmemthread-and-debugging-them.aspx)。

## 高 CPU 负载

尽管听起来很奇怪，但服务器上的低 CPU 负载并不一定是好兆头。它表明服务器未被充分利用。尽管未充分利用为系统留下了增长空间，但它增加了 IT 基础设施和运营成本；需要托管和维护更多的服务器。显然，高 CPU 负载也不好。持续对 SQL Server 施加 CPU 压力会使系统响应迟缓和缓慢。

有几个指标可以帮助您检测服务器是否在 CPU 压力下工作。这些包括高百分比的 `SOS_SCHEDULER_YIELD` 等待，当工作线程等待时发生...


运行就绪状态。您可以分析`% processor time`（处理器时间百分比）和`processor queue length`（处理器队列长度）性能计数器，并比较`sys.dm_os_wait_stats`视图中的信号等待时间和资源等待时间，如清单 28-9 所示。

信号等待表示等待 CPU 的时间，而资源等待表示等待资源（例如磁盘页面）的时间。尽管微软建议信号等待类型不应超过 25%，但我认为在繁忙系统中，15%到 20%是更好的目标值。

***清单 28-9.*** 比较信号等待与资源等待

```
select
    sum(signal_wait_time_ms) as [Signal Wait Time (ms)]
    ,convert(decimal(7,4), 100.0 * sum(signal_wait_time_ms) / sum (wait_time_ms)) as [% Signal waits]
    ,sum(wait_time_ms - signal_wait_time_ms) as [Resource Wait Time (ms)]
    ,convert(decimal(7,4), 100.0 * sum(wait_time_ms - signal_wait_time_ms) / sum (wait_time_ms)) as [% Resource waits]
from
    sys.dm_os_wait_stats with (nolock)
```

许多因素可能导致系统中的 CPU 负载，而糟糕的 T-SQL 代码是首要原因。命令式处理、游标、XQuery、多语句用户定义函数和复杂计算尤其消耗 CPU 资源。

