# 第 27 章 ■ 扩展事件

```sql
EventInfo([Event], [Event Time], [CPU Time], [Duration], [Logical Reads], [Physical Reads]
,[Writes], [Rows], [Statement], [PlanHandle], File_Name, File_Offset)
as
(
select
Data.value('/event[1]/@name','sysname') as [Event]
,Data.value('/event[1]/@timestamp','datetime') as [Event Time]
,Data.value('((/event[1]/data[@name="cpu_time"]/value/text())[1])','bigint')
as [CPU Time]
,Data.value('((/event[1]/data[@name="duration"]/value/text())[1])','bigint')
as [Duration]
,Data.value('((/event[1]/data[@name="logical_reads"]/value/text())[1])'
,'int') as [Logical Reads]
,Data.value('((/event[1]/data[@name="physical_reads"]/value/text())[1])'
,'int') as [Physical Reads]
,Data.value('((/event[1]/data[@name="writes"]/value/text())[1])','int') as [Writes]
,Data.value('((/event[1]/data[@name="row_count"]/value/text())[1])','int') as [Rows]
,Data.value('((/event[1]/data[@name="statement"]/value/text())[1])','nvarchar(max)')
as [Statement]
,Data.value('xs:hexBinary(((/event[1]/action[@name="plan_handle"]/value/text())[1]))'
,'varbinary(64)') as [PlanHandle]
,File_Name, File_Offset
from TargetData
)
select
ei.[Event], ei.[Event Time]
,ei.[CPU Time] / 1000 as [CPU Time (ms)]
,ei.[Duration] / 1000 as [Duration (ms)]
,ei.[Logical Reads], ei.[Physical Reads], ei.[Writes], ei.[Rows], ei.[Statement]
,ei.[PlanHandle], ei.File_Name, ei.File_Offset, qp.Query_Plan
from EventInfo ei outer apply sys.dm_exec_query_plan(ei.PlanHandle) qp
```

后续步骤取决于你的目标。在某些情况下，当你分析原始事件数据时，就能看到明显的优化目标。在其他情况下，你将需要执行额外的分析，并查看执行频率，基于 `query_hash` 或 `query_plan_hash` 动作数据来聚合数据。

你也可以考虑创建一个按计划运行的进程，提取新收集的数据并将其持久化存储在表中。如果查询计划仍在计划缓存中，这种方法会增加捕获它们的机会。在此类实现中，你可以使用 `ring_buffer` 作为目标，而非 `event_file`。

## 监控页面拆分事件

扩展事件可以帮助你解决那些用其他方法难以甚至无法排查的问题。其中一个例子就是监控 `页面拆分` 事件，它允许你识别遭受页面拆分和碎片化的索引。

捕获实际的页面拆分是一个棘手的过程。尽管 SQL Server 2012 暴露了 `page_split` 事件，但它不区分在不断增长的索引中分配新页面时发生的页面拆分和 `常规` 页面拆分。幸运的是，你可以改用 `transaction_log` 事件的 `LOP_DELETE_SPLIT` 操作。此操作标记在拆分事件发生时原始页面上行的删除。

清单 27-26 展示了创建扩展事件会话的代码，该会话在其中一个数据库中捕获页面拆分信息。此会话使用 `histogram` 目标，按索引统计事件。

***清单 27-26.*** 捕获页面拆分事件

```sql
create event session PageSplits_Tracking
on server
add event sqlserver.transaction_log
(
where operation = 11 -- lop_delete_split
and database_id = 17
)
add target package0.histogram
(
set
filtering_event_name = 'sqlserver.transaction_log',
source_type = 0, -- event column
source = 'alloc_unit_id'
)
```

清单 27-27 中的代码展示了如何从目标中提取数据。

***清单 27-27.*** 分析页面拆分信息

```sql
;with Data(alloc_unit_id, splits)
as
(
select c.n.value('(value)[1]', 'bigint') as alloc_unit_id, c.n.value('(@count)[1]'
,'bigint') as splits
from
(
select convert(xml,target_data) target_data
from sys.dm_xe_sessions s with (nolock) join sys.dm_xe_session_targets t on
s.address = t.event_session_address
where s.name = 'PageSplits_Tracking' and t.target_name = 'histogram'
) as d cross apply
target_data.nodes('HistogramTarget/Slot') as c(n)
)
select
```


## 第 27 章 ■ 扩展事件

```sql
s.name + '.' + o.name as [表], i.index_id, i.name as [索引]
,d.Splits, i.fill_factor as [填充因子]
from
Data d join sys.allocation_units au with (nolock) on
d.alloc_unit_id = au.allocation_unit_id
join sys.partitions p with (nolock) on
au.container_id = p.partition_id
join sys.indexes i with (nolock) on
p.object_id = i.object_id and p.index_id = i.index_id
join sys.objects o with (nolock) on
i.object_id = o.object_id
join sys.schemas s on
o.schema_id = s.schema_id
```

你也可以在分析不同值如何实时影响页面拆分时，于索引 `FILLFACTOR` 调优期间使用此技术。

## Azure SQL 数据库中的扩展事件

Microsoft Azure SQL 数据库 v12 也支持扩展事件。尽管公开的事件列表相对较小，但它支持在排查常见性能问题时很有帮助的事件，例如检测低效查询、阻塞问题、`tempdb` 溢出、过量内存授予等。

在撰写本书时，SQL Azure 支持三种事件目标：例如 `ring_buffer`、`event_counter` 和 `event_file`。你可以通过使用本章的查询来查询目录视图，从而分析支持的扩展事件对象列表。

你可以在 SQL 数据库中创建扩展事件会话并查询目标，就像在常规 SQL Server 中一样。不过，在语法上存在一些细微差异。首先，你必须使用 `CREATE EVENT SESSION ON DATABASE` 子句，而不是 `ON SERVER` 子句。SQL 数据库将事件的作用域限定在数据库级别，而非服务器级别。数据库管理视图的命名约定也不同。你需要在名称中添加 `_database`；例如，使用 `sys.dm_xe_database_sessions` 视图而非 `sys.dm_xe_sessions` 视图。

**注意** 你可以在 [`azure.microsoft.com/en-us/documentation/articles/sql-database-xevent-db-diff-from-svr/`](https://azure.microsoft.com/en-us/documentation/articles/sql-database-xevent-db-diff-from-svr/) 阅读关于 Azure SQL 数据库中扩展事件支持的文章。

#### 总结

扩展事件是一种轻量级且高度可扩展的监视和调试基础设施，将在未来版本的 SQL Server 中取代 SQL 跟踪。它解决了 SQL 跟踪在可用性方面的限制，并且通过仅收集所需信息以及在事件执行的早期阶段进行谓词分析，从而降低了对 SQL Server 的开销。

SQL Server 在每个新版本中都会公开新的扩展事件。从 SQL Server 2012 开始，所有 SQL 跟踪事件都有对应的扩展事件。此外，新的 SQL Server 功能不提供任何 SQL 跟踪支持，而是依赖于扩展事件。

扩展事件以 XML 格式提供数据。每种事件类型都有其自己的架构，其中包括该事件类型的特定数据列。你可以使用一组全局可用的可用操作向事件数据添加额外信息，并且可以对事件数据应用谓词，过滤掉不需要的事件。

事件数据可以存储在多个内存中和磁盘上的目标中，这使你可以收集原始事件数据或执行一些分析和聚合，例如计数和分组事件，或跟踪不匹配的事件对。

`system_health` 事件会话提供有关 SQL Server 一般组件健康状况、资源使用情况和高严重性错误的信息。此会话在每个 SQL Server 实例上默认创建并运行。收集的事件之一是 `xml_deadlock_report`，它允许你获取最近死锁的死锁图，而无需设置 SQL 跟踪事件或 `T1222` 跟踪标志。

扩展事件是一项很棒的技术，它允许你使用其他方法无法解决的复杂场景进行故障排除。尽管学习曲线陡峭，但学习和使用扩展事件是非常有益的。


