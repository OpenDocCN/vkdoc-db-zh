# 第 7 章 ■ 分析查询性能

关于扩展事件（Extended Events）数据的一个小提示：如果数据要收集到文件中，那么之后你需要将数据加载到表中，或者直接对其进行查询。你可以通过使用这个系统函数查询扩展事件文件来直接读取：

```sql
SELECT *
FROM sys.fn_xe_file_target_read_file('C:\Sessions\QueryPerformanceMetrics*.xel',
NULL, NULL, NULL);
```

该查询将每个事件作为单行返回。关于事件的数据存储在 XML 列 `event_data` 中。

你需要使用 `XQuery` 来直接读取数据，但一旦读取，你就可以对捕获的数据进行搜索、排序和聚合。在下一节中，我将带你逐步完成这个机制的完整示例。

### 识别高成本查询

SQL Server 的目标是在最短时间内将结果集返回给用户。为此，SQL Server 内置了一个基于成本的优化器，称为**查询优化器**，它会生成一个成本效益高的策略，称为**查询执行计划**。查询优化器会权衡许多因素，包括（但不限于）执行查询所需的 CPU、内存和磁盘 I/O 的使用情况，这些数据来源于各种渠道，例如索引维护的或动态生成的统计数据、数据约束，以及对查询运行系统的一些了解，如 CPU 数量和内存量。根据所有这些信息，优化器创建一个成本效益高的执行计划。

在会话返回的数据中，`cpu_time` 和 `logical_reads` 或 `physical_reads` 字段也显示了查询在哪些方面成本较高。`cpu_time` 字段表示执行查询所使用的 CPU 时间。两个读取字段表示查询操作的页面（大小为 8KB）数量，从而表明查询引起的内存或 I/O 压力。它也表明了磁盘压力，因为对于操作查询，内存页面在首次访问数据时需要被填充，在内存瓶颈时需要换出到磁盘。查询的 `logical_reads` 数量越高，可能对磁盘造成的压力就越大。过多的逻辑页面也会增加 CPU 在管理这些页面上的负载。这不是一个自动的关联。你不能总是指望读取次数最多的查询就是性能最差的。但这是一个通用指标，也是一个良好的起点。尽管最小化 I/O 次数并不是成本效益高计划的必然要求，但你通常会发现成本最低的计划通常具有最少的 I/O，因为 I/O 操作是昂贵的。

导致大量逻辑读取的查询通常会对相应的大量数据集获取锁。即使读取（相对于写入）也可能需要根据隔离级别对所有数据获取共享锁。

这些查询会阻塞所有其他出于修改目的（而非读取）而请求此数据（或部分数据）的查询。由于这些查询本身成本高昂且执行时间长，它们会长时间阻塞其他查询。被阻塞的查询接着又会阻塞进一步的查询，在数据库中引入一连串的阻塞。（第 13 章将介绍锁模式。）

因此，识别出高成本查询并首先优化它们是有意义的，从而做到以下几点：

- 提升高成本查询本身的性能
- 降低对系统资源的整体压力
- 减少数据库阻塞

高成本查询可分为以下两种类型：

- **单次执行**：查询的单次执行成本很高。
- **多次执行**：查询本身可能成本不高，但重复执行该查询会对系统资源造成压力。

你可以使用不同的方法来识别这两种类型的高成本查询，如下节所述。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 单次执行的高成本查询

你可以通过分析会话输出文件或查询 `sys.dm_exec_query_stats` 来识别高成本查询。在这个例子中，我们将从识别执行大量逻辑读取的查询开始，因此你应该按 `logical_reads` 数据列对会话输出进行排序。你可以更改排序方式，按持续时间或 CPU 排序，甚至以有趣的方式组合它们。你可以通过以下步骤访问会话信息：

1.  捕获一个包含典型工作负载的会话。
2.  将会话输出保存到文件。
3.  使用“文件”->“打开”打开文件，并选择一个 `.xel` 文件以使用数据浏览器窗口。在那里对信息进行排序。
4.  或者，你可以查询跟踪文件进行分析，按 `logical_reads` 字段排序。

```sql
WITH xEvents
AS (SELECT object_name AS xEventName,
           CAST (event_data AS XML) AS xEventData
    FROM sys.fn_xe_file_target_read_file('C:\Sessions\QueryPerformanceMetrics*.xel', NULL, NULL, NULL)
)
SELECT xEventName,
       xEventData.value('(/event/data[@name=''duration'']/value)[1]', 'bigint') Duration,
       xEventData.value('(/event/data[@name=''physical_reads'']/value)[1]', 'bigint') PhysicalReads,
       xEventData.value('(/event/data[@name=''logical_reads'']/value)[1]', 'bigint') LogicalReads,
       xEventData.value('(/event/data[@name=''cpu_time'']/value)[1]', 'bigint') CpuTime,
       CASE xEventName
            WHEN 'sql_batch_completed'
            THEN xEventData.value('(/event/data[@name=''batch_text'']/value)[1]', 'varchar(max)')
            WHEN 'rpc_completed'
            THEN xEventData.value('(/event/data[@name=''statement'']/value)[1]', 'varchar(max)')
       END AS SQLText,
       xEventData.value('(/event/data[@name=''query_hash'']/value)[1]', 'binary(8)') QueryHash
INTO Session_Table
FROM xEvents;

SELECT st.xEventName,
       st.Duration,
       st.PhysicalReads,
       st.LogicalReads,
       st.CpuTime,
       st.SQLText,
       st.QueryHash
FROM Session_Table AS st
ORDER BY st.LogicalReads DESC;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

让我们稍微分解一下这个查询。首先，我创建了一个名为 `xEvents` 的公共表表达式（CTE）。我这样做只是因为它使代码更容易阅读。它并没有从根本上改变任何行为。当我需要从文件读取并转换数据类型时，我更喜欢这样做。这样我后续语句中的 XML 查询就更有意义了。请注意，我在读取文件时使用了通配符 `QueryPerformanceMetrics*.xel`。这使我能够读取扩展事件会话创建的所有滚动文件（更多细节见第 6 章）。

根据收集的数据量和文件的大小，直接对从扩展事件收集的文件运行查询可能会异常缓慢。在这种情况下，请使用相同的基本函数 `sys.fn_xe_file_target_read_file` 将数据加载到表中，而不是直接查询。完成后，你可以对表应用索引以加快查询速度。我使用前面的脚本将数据放入一个表中，然后查询该表以获取输出。这对于测试来说效果很好，但对于更永久的解决方案，你会希望有一个专门的数据库来存储此类数据，表结构要适当，而不是像我这里使用 `INTO` 这样的快捷方式。

在某些情况下，你可能从系统监视器输出中识别出 CPU 存在巨大压力。对 CPU 的压力可能是由于大量 CPU 密集型操作造成的，例如存储过程重新编译、聚合函数、数据排序、哈希连接等。在这种情况下，你应该按 `cpu_time` 字段对会话输出进行排序，以识别那些占用大量处理器周期的查询。

### 多次执行的高成本查询



正如我之前提到的，有时单个查询本身可能成本不高，但多次执行同一查询的累积效应可能会给系统资源带来压力。在这种情况下，按 `logical_reads` 字段排序无法帮助你识别这类成本较高的查询。你真正需要知道的是该查询多次执行所产生的总读取次数、总 CPU 时间或累计持续时间。

*   查询会话输出，并按你感兴趣的某些值进行分组。
*   访问 `sys.dm_exec_query_stats` DMO（动态管理对象），从生产服务器检索信息。这假设你处理的是近期发生或不依赖于已知历史记录的问题，因为此数据仅来自当前存储在过程缓存中的内容。

但如果你需要准确的历史数据视图，可以查阅通过扩展事件收集的指标。一旦会话数据被导入到数据库表中，就可以执行 `SELECT` 语句来查找同一查询多次执行的总读取次数，如下所示：

```sql
SELECT COUNT(*) AS TotalExecutions,
st.xEventName,
st.SQLText,
SUM(st.Duration) AS DurationTotal,
SUM(st.CpuTime) AS CpuTotal,
SUM(st.LogicalReads) AS LogicalReadTotal,
SUM(st.PhysicalReads) AS PhysicalReadTotal
FROM Session_Table AS st
GROUP BY st.xEventName, st.SQLText
ORDER BY LogicalReadTotal DESC;
```

前面脚本中的 `TotalExecutions` 列表示查询执行的次数。

`LogicalReadTotal` 列表示该查询多次执行所执行的逻辑读取总数。

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 7 章 ■ 查询性能分析

通过此方法识别出的成本较高的查询，比通过会话识别出的单次执行成本较高的查询，更能准确反映负载情况。例如，一个需要 50 次读取的查询可能被执行了 1,000 次。单次查询本身可能被认为足够“廉价”，但该查询执行的总读取次数结果为 50,000 (`=50 * 1,000`)，这就不能被认为是廉价的了。优化此查询，即使每次执行减少 10 次读取，也能使总读取次数减少 10,000 (`=10 * 1,000`)，这可能比优化一个单次执行需要 5,000 次读取的查询更有益。

这种方法的问题在于，大多数查询的 `WHERE` 子句中会有一组不同的条件，或者过程调用会传入不同的参数值。这使得仅按带参数的查询或过程进行简单分组变得不可能。你可以采用多种方法来处理这个问题。既然你已经使用了扩展事件，实际上可以利用它来为你服务。例如，`rpc_completed` 事件会捕获过程名作为一个字段。你可以直接按该字段分组。对于批处理，你可以添加 `query_hash` 字段，然后按其分组。另一种方法是清理数据，移除参数值，正如微软开发者网络（MSDN）上所述：[`bit.ly/1e1I38f`](http://bit.ly/1e1I38f)。虽然最初是为 SQL Server 2005 编写的，但其概念在 SQL Server 2014 中同样适用。

要从 `sys.dm_exec_query_stats` 视图中获取相同的信息，只需针对该 DMV 进行查询即可。

```sql
SELECT s.totalexecutioncount,
t.text,
s.TotalExecutionCount,
s.TotalElapsedTime,
s.TotalLogicalReads,
s.TotalPhysicalReads
FROM (SELECT deqs.plan_handle,
SUM(deqs.execution_count) AS TotalExecutionCount,
SUM(deqs.total_elapsed_time) AS TotalElapsedTime,
SUM(deqs.total_logical_reads) AS TotalLogicalReads,
SUM(deqs.total_physical_reads) AS TotalPhysicalReads
FROM sys.dm_exec_query_stats AS deqs
GROUP BY deqs.plan_handle
) AS s
CROSS APPLY sys.dm_exec_sql_text(s.plan_handle) AS t
ORDER BY s.TotalLogicalReads DESC ;
```

另一种利用执行 DMOs 中可用数据的方法是使用 `query_hash` 和 `query_plan_hash` 作为聚合机制。尽管一个给定的存储过程或参数化查询可能传入不同的值，但它们的 `query_hash` 和 `query_plan_hash` 在大多数情况下是相同的。

这意味着你可以根据这些哈希值进行聚合，以识别那些原本无法看到的通用计划或通用查询模式。以下只是对前一个查询的轻微修改：

```sql
SELECT s.TotalExecutionCount,
t.text,
s.TotalExecutionCount,
s.TotalElapsedTime,
s.TotalLogicalReads,
s.TotalPhysicalReads
FROM (SELECT deqs.query_plan_hash,
SUM(deqs.execution_count) AS TotalExecutionCount,
SUM(deqs.total_elapsed_time) AS TotalElapsedTime,
SUM(deqs.total_logical_reads) AS TotalLogicalReads,
SUM(deqs.total_physical_reads) AS TotalPhysicalReads
FROM sys.dm_exec_query_stats AS deqs
GROUP BY deqs.query_plan_hash
) AS s
CROSS APPLY (SELECT plan_handle
FROM sys.dm_exec_query_stats AS deqs
WHERE s.query_plan_hash = deqs.query_plan_hash
) AS p
CROSS APPLY sys.dm_exec_sql_text(p.plan_handle) AS t
ORDER BY TotalLogicalReads DESC;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 7 章 ■ 查询性能分析

这种方法比收集会话数据所需的所有工作要容易得多，以至于让人疑惑为什么还要使用扩展事件。主要原因是精确性。`sys.dm_exec_query_stats` 视图是针对某个特定计划在内存中存在期间的运行时聚合。而扩展事件会话则是在你运行它的任意时间范围内的历史跟踪。你甚至可以将会话结果添加到数据库中。拥有这样一个数据列表，你可以更精确地生成关于事件的总计，而不是仅仅依赖于某个特定时间点。但要理解，大量的性能问题故障排查都聚焦于服务器近期发生的情况，由于 `sys.dm_exec_query_stats` 基于缓存，该 DMV 通常代表了系统近期的状况，因此 `sys.dm_exec_query_stats` 极其重要。但是，如果你处理的是“现在到底是什么运行得慢”这种更具战术性的情况，你会使用 `sys.dm_exec_requests`。

### 识别运行缓慢的查询

因为用户的体验深受其请求响应时间的影响，所以你应该定期监控传入 SQL 查询的执行时间，找出运行缓慢查询的响应时间，创建一个查询性能基线。如果运行缓慢查询的响应时间（或持续时间）变得无法接受，那么你就应该分析性能下降的原因。不过，并非每个性能低下的查询都是由资源问题引起的。

其他问题，如阻塞，也可能导致查询性能下降。阻塞将在第 12 章中详细讨论。

要识别运行缓慢的查询，只需更改针对会话数据的查询，改变排序依据，如下所示：

```sql
WITH xEvents AS
(SELECT object_name AS xEventName,
CAST (event_data AS xml) AS xEventData
FROM sys.fn_xe_file_target_read_file
('C:\Sessions\QueryPerformanceMetrics*.xel', NULL, NULL, NULL)
)
SELECT
xEventName,
xEventData.value('(/event/data[@name=''duration'']/value)[1]','bigint') Duration,
xEventData.value('(/event/data[@name=''physical_reads'']/value)[1]','bigint') PhysicalReads,
xEventData.value('(/event/data[@name=''logical_reads'']/value)[1]','bigint') LogicalReads,
xEventData.value('(/event/data[@name=''cpu_time'']/value)[1]','bigint') CpuTime,
xEventData.value('(/event/data[@name=''batch_text'']/value)[1]','varchar(max)') BatchText,
xEventData.value('(/event/data[@name=''statement'']/value)[1]','varchar(max)') StatementText,
xEventData.value('(/event/data[@name=''query_plan_hash'']/value)[1]','binary(8)') QueryPlanHash
FROM xEvents
ORDER BY Duration DESC;
```



对于运行缓慢的系统，你应在优化过程前后记录慢速查询的耗时。应用优化技术后，你需要评估其对系统的整体影响。有可能你的优化步骤会对其他查询产生负面影响，导致它们变得更慢。

`[www.it-ebooks.info](http://www.it-ebooks.info/)`

## 第 7 章 ■ 分析查询性能

## 执行计划

一旦识别出代价高昂的查询，你需要找出其代价高昂的`原因`。你可以从`扩展事件`或`sys.dm_exec_procedure_stats`中识别出代价高昂的过程，在`Management Studio`中重新运行它，并查看`查询优化器`所使用的`执行计划`。`执行计划`显示了`查询优化器`用来执行查询的处理策略（包括多个中间步骤）。

为了创建`执行计划`，`查询优化器`会评估各种索引和连接策略的排列组合。

由于存在大量潜在计划的可能性，这个优化过程可能需要很长时间才能生成最具成本效益的`执行计划`。为了防止对`执行计划`进行过度优化，优化过程被分为多个阶段。每个阶段是一套转换规则，用于评估各种索引和连接策略的排列组合，最终目标是找到一个足够好的计划，而非一个完美的计划。

正是这种“足够好”与“完美”之间的差异，可能导致因`执行计划`优化不足而性能低下。`查询优化器`在直接采用当前成本最低的计划之前，只会尝试有限次数的优化。

完成一个阶段后，`查询优化器`会检查所得计划的估计成本。如果`查询优化器`确定该计划足够“便宜”，它就会使用该计划，而不再进行后续的优化阶段。然而，如果计划不够“便宜”，优化器将进入下一个优化阶段。我将在第 9 章更深入地探讨`执行计划`的生成。

SQL Server 以多种形式和两种不同类型显示查询`执行计划`。在 SQL Server 2012 中，最常用的形式是`图形化执行计划`和`XML 执行计划`。实际上，`图形化执行计划`只是为屏幕显示而解析的`XML 执行计划`。两种`执行计划`类型分别是`估计执行计划`和`实际执行计划`。`估计`计划代表来自`查询优化器`的结果，而`实际`计划是同一计划外加一些运行时指标。`估计执行计划`的优点在于它不需要执行查询。这些类型生成的计划可能不同，但仅当执行期间发生语句级重编译时才会如此。大多数情况下，两种类型的计划是相同的。主要区别在于`实际执行计划`中包含了一些`估计执行计划`中没有的执行统计信息。

`图形化执行计划`使用图标来表示查询的处理策略。要获取`图形化估计执行计划`，请选择“查询”->“显示估计的执行计划”。`XML 执行计划`包含与`图形化计划`相同的数据，但以更便于编程访问的格式呈现。此外，借助 SQL Server 的 XQuery 功能，可以像查询表一样查询`XML 执行计划`。`XML 执行计划`通过 `SET SHOWPLAN_XML` 语句生成`估计计划`，通过 `SET STATISTICS XML` 语句生成`实际执行计划`。你也可以右键单击`图形化执行计划`并选择“显示执行计划 XML”。你还可以使用 DMO `sys.dm_exec_query_plan` 直接从计划缓存中提取计划。缓存中存储的计划没有运行时信息，因此从技术上讲它们是`估计计划`。

■ `注意` 你应该确保数据库设置为 `兼容级别 120`，以便它能准确反映对 SQL Server 2014 的更新。


