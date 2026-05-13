# 15. 监控索引

本书已讨论了许多主题，例如索引是什么、它们的作用、构建索引的模式，以及确定 SQL Server 数据库应如何建立索引的诸多其他方面。所有这些信息对于索引数据库的最后一步——分析数据库以确定哪些索引是必需的——都是必要的。本书最后三章将整合实施索引方法所需的信息。

首先，本章将讨论一种可用于监控索引的通用实践。它将包括可采取的步骤，用以观察索引的行为并理解它们如何影响环境。此方法可应用于单个数据库、一个服务器或整个 SQL Server 环境。无论数据库支持何种类型的运营或业务，都可以使用类似的监控流程。

监控索引背后的主要目标是创建收集、存储和报告索引相关信息的能力。这些信息将来自多种来源。监控的来源应该很熟悉，因为它们常用于与索引类似的任务，例如性能调优。对于某些来源，信息将随时间推移而收集，以提供总体趋势的概念。对于其他来源，在特定时间点的快照就足够了。重要的是要随时间收集一些信息，以提供一个基准来比较性能。这将有助于理解索引使用情况和有效性何时发生了变化。

监控索引需要从多个来源收集信息。本章将讨论的来源如下：

*   性能计数器
*   动态管理对象
*   事件追踪

针对这些来源中的每一个，后续章节将描述需要收集的内容，并就如何收集这些信息提供指导。到本章结束时，将形成一个框架，能够提供启动分析阶段所需的信息。

> 注意
> 
> 本章的所有监控信息将收集在一个名为 `IndexingMethod` 的数据库中。脚本可以在该数据库中运行，也可以在另一个性能监控数据库中运行。

### 性能计数器

监控索引的第一个信息来源是 SQL Server 性能计数器。性能计数器是 Microsoft 提供的度量标准，用于衡量服务器上应用程序和硬件中事件的发生速率或资源的状态。对于某些性能计数器，存在通用准则可用于指示何时可能存在索引问题。对于其他计数器，其速率或级别的变化可能表明需要调整服务器上的索引。

使用性能计数器的主要挑战在于，它们代表的是服务器级别或 SQL Server 实例级别的计数器状态。它们并不指示可能发生索引问题的数据库或表级别。不过，在考虑到可用于监控索引和识别潜在索引需求的其他工具时，这种详细程度是可接受且有用的。在此级别收集计数器信息的一个优势是，我们被迫考虑全局以及所有索引对性能的影响。在孤立的情况下，一个表上几个性能不佳的索引可能是可以接受的。然而，当与其他索引不佳的表结合时，总体性能可能会达到一个临界点，此时索引性能问题需要得到解决。借助性能计数器提供的服务器级别统计信息，我们将能够识别何时达到了这个临界点。

SQL Server 和 Windows Server 有许多可用的性能计数器。但从索引的角度来看，许多性能计数器可以忽略。最有用的性能计数器是那些映射到与索引操作或访问方式相关的操作，例如转送的记录和索引搜索。有关对索引最有用的性能计数器的定义，请参见表 15-1。收集每个计数器的原因及其如何影响索引决策将在下一章讨论。

表 15-1
索引相关性能计数器

| 选项名称 | 描述 |
| --- | --- |
| `Access Methods\Forwarded Records/sec` | 每秒通过转送记录指针获取的记录数 |
| `Access Methods\FreeSpace Page Fetches/sec` | 每秒在已分配给对象的页面中获取页面以插入或修改记录的次数 |
| `Access Methods\FreeSpace Scans/sec` | 每秒启动的扫描次数，用于在已分配给对象的页面中搜索空闲空间以插入或修改记录 |
| `Access Methods\Full Scans/sec` | 每秒无限制的完全扫描次数。可以是基表扫描或完整索引扫描 |
| `Access Methods\Index Searches/sec` | 每秒索引搜索次数。这些搜索用于启动范围扫描和单索引记录获取，以及重新定位索引 |
| `Access Methods\Page compression attempts/sec` | 每秒使用 PAGE 压缩尝试压缩页面的次数；包括失败的页面压缩 |
| `Access Methods\Pages compressed/sec` | 每秒使用 PAGE 压缩成功压缩的页面数 |
| `Access Methods\Page Splits/sec` | 每秒发生的页拆分次数，由索引页面溢出导致 |
| `Buffer Manager\Page Lookups/sec` | 在缓冲池中查找页面的请求数 |
| `Locks(*)\Lock Wait Time (ms)` | 上一秒中锁的总等待时间（毫秒） |
| `Locks(*)\Lock Waits/sec` | 每秒导致调用方需要等待的锁请求数 |
| `Locks(*)\Number of Deadlocks/sec` | 每秒导致死锁的锁请求数 |
| `SQL Statistics\Batch Requests/sec` | 每秒接收到的 Transact SQL 命令批次数 |



收集性能计数器的方法有多种。本章的监控将使用动态管理视图 `sys.dm_os_performance_counters`。该视图会为实例的所有 SQL Server 计数器返回一行。返回的值是计数器的原始值。根据计数器类型，该值可以是时间点的状态值，也可以是持续累积的聚合值。

为了开始收集用于监控的性能计数器信息，需要创建一个表来存储这些信息。清单 15-1 中的表定义满足此需求。在收集性能计数器时，将使用一个表来存储计数器名称及其值，并为每一行添加日期戳以标识信息的收集时间。

```sql
USE IndexingMethod;
GO
CREATE TABLE dbo.IndexingCounters
(
counter_id INT IDENTITY(1, 1),
create_date DATETIME2(0),
server_name VARCHAR(128) NOT NULL,
object_name VARCHAR(128) NOT NULL,
counter_name VARCHAR(128) NOT NULL,
instance_name VARCHAR(128) NULL,
Calculated_Counter_value FLOAT NULL,
CONSTRAINT PK_IndexingCounters
PRIMARY KEY CLUSTERED (counter_id)
);
GO
CREATE NONCLUSTERED INDEX IX_IndexingCounters_CounterName
ON dbo.IndexingCounters (counter_name)
INCLUDE (create_date, server_name, object_name, Calculated_Counter_value);
```

清单 15-1
性能计数器快照表

为了收集索引监控信息，将从 `sys.dm_os_performance_counters` 收集信息，并用于计算该动态管理视图中的适当值。这些值与从其他工具（如 `性能监视器`）查看性能计数器信息时可用的值相同。需要几个步骤来填充 `dbo.IndexingCounters`。该动态管理视图包含原始计数器值。为了正确计算这些值，需要对动态管理视图中的值进行多次快照，每次快照间隔数秒，然后计算值之间的差异。在清单 15-2 中，计数器值在 10 秒后计算。时间一到，计数器值即被计算并插入到 `dbo.IndexingCounters` 表中。此脚本应被计划并频繁执行。理想情况下，应每 1-5 分钟收集一次此信息，不过频率可以根据特定环境中的索引活动级别进行调整。您可以自定义此过程的时机以满足特定数据库的索引需求。

注意

性能计数器信息可以更频繁地收集。例如，`性能监视器` 默认为每 15 秒收集一次。对于索引监控而言，无需如此高的频率。

```sql
USE IndexingMethod;
GO
DROP TABLE IF EXISTS #Counters;
SELECT pc.object_name,
pc.counter_name
INTO #Counters
FROM sys.dm_os_performance_counters pc
WHERE pc.cntr_type IN ( 272696576, 1073874176 )
AND (
pc.object_name LIKE '%:Access Methods%'
AND (
pc.counter_name LIKE 'Forwarded Records/sec%'
OR pc.counter_name LIKE 'FreeSpace Scans/sec%'
OR pc.counter_name LIKE 'FreeSpace Page Fetches/sec%'
OR pc.counter_name LIKE 'Full Scans/sec%'
OR pc.counter_name LIKE 'Index Searches/sec%'
OR pc.counter_name LIKE 'Page Splits/sec%'
OR pc.counter_name LIKE 'Page compression attempts/sec%'
OR pc.counter_name LIKE 'Pages compressed/sec%'
)
)
OR (
pc.object_name LIKE '%:Buffer Manager%'
AND (
pc.counter_name LIKE 'Page life expectancy%'
OR pc.counter_name LIKE 'Page lookups/sec%'
)
)
OR (
pc.object_name LIKE '%:Locks%'
AND (
pc.counter_name LIKE 'Lock Wait Time (ms)%'
OR pc.counter_name LIKE 'Lock Waits/sec%'
OR pc.counter_name LIKE 'Number of Deadlocks/sec%'
)
)
OR (
pc.object_name LIKE '%:SQL Statistics%'
AND pc.counter_name LIKE 'Batch Requests/sec%'
)
GROUP BY pc.object_name,
pc.counter_name;
DROP TABLE IF EXISTS #Baseline;
SELECT GETDATE() AS sample_time,
pc1.object_name,
pc1.counter_name,
pc1.instance_name,
pc1.cntr_value,
pc1.cntr_type,
x.cntr_value AS base_cntr_value
INTO #Baseline
FROM sys.dm_os_performance_counters pc1
INNER JOIN #Counters c ON c.object_name = pc1.object_name
AND c.counter_name = pc1.counter_name
OUTER APPLY (
SELECT cntr_value
FROM sys.dm_os_performance_counters pc2
WHERE pc2.cntr_type           = 1073939712
AND UPPER(pc1.counter_name) = UPPER(pc2.counter_name)
AND pc1.object_name         = pc2.object_name
AND pc1.instance_name       = pc2.instance_name
) x;
WAITFOR DELAY '00:00:15';
INSERT INTO dbo.IndexingCounters (
create_date,
server_name,
object_name,
counter_name,
instance_name,
Calculated_Counter_value
)
SELECT GETDATE(),
LEFT(pc1.object_name, CHARINDEX(':', pc1.object_name) - 1),
SUBSTRING(pc1.object_name, 1 + CHARINDEX(':', pc1.object_name), LEN(pc1.object_name)),
pc1.counter_name,
pc1.instance_name,
CASE
WHEN pc1.cntr_type = 65792 THEN pc1.cntr_value
WHEN pc1.cntr_type = 272696576 THEN
COALESCE((1. * pc1.cntr_value - x.cntr_value) / NULLIF(DATEDIFF(s, sample_time, GETDATE()), 0), 0)
WHEN pc1.cntr_type = 537003264 THEN COALESCE((1. * pc1.cntr_value) / NULLIF(base.cntr_value, 0), 0)
WHEN pc1.cntr_type = 1073874176 THEN
COALESCE(
(1. * pc1.cntr_value - x.cntr_value) / NULLIF(base.cntr_value - x.base_cntr_value, 0)
/ NULLIF(DATEDIFF(s, sample_time, GETDATE()), 0),
0
) END AS real_cntr_value
FROM sys.dm_os_performance_counters pc1
INNER JOIN #Counters c ON c.object_name = pc1.object_name
AND c.counter_name = pc1.counter_name
OUTER APPLY (
SELECT cntr_value,
base_cntr_value,
sample_time
FROM #Baseline b
WHERE b.object_name   = pc1.object_name
AND b.counter_name  = pc1.counter_name
AND b.instance_name = pc1.instance_name
) x
OUTER APPLY (
SELECT cntr_value
FROM sys.dm_os_performance_counters pc2
WHERE pc2.cntr_type           = 1073939712
AND UPPER(pc1.counter_name) = UPPER(pc2.counter_name)
AND pc1.object_name         = pc2.object_name
AND pc1.instance_name       = pc2.instance_name
) base;
```

清单 15-2
性能计数器快照脚本

第一次为索引收集性能计数器时，无法将计数器值与 SQL Server 数据库实例的其他合理值进行比较。但随着时间的推移，可以保留先前的性能计数器样本以进行比较。作为监控的一部分，重要的是识别出性能计数器值代表您环境典型活动的时期。利用这些数据，可以识别异常情况或使用非典型、需要关注的时期。



### 性能计数器与动态管理对象监控

### 建立性能基线表

要存储这些值，请将它们插入到一个类似于清单 15-3 的表中。此表包含开始日期和结束日期，以指示基线所代表的范围。此外，还有最小值、最大值、平均值和标准差列，用于存储从收集的计数器中获取的值。最小值和最大值将有助于理解性能计数器的变化情况。平均值提供了计数器值在“良好”状态时的概貌。标准差使我们能够了解计数器值的变异性。数值越低，表示计数器值越频繁地聚集在平均值附近。数值越高，则表示计数器值变化更频繁，并且通常更接近最小值和最大值。

```sql
USE IndexingMethod;
GO
CREATE TABLE dbo.IndexingCountersBaseline
(
    counter_baseline_id INT IDENTITY(1, 1),
    start_date DATETIME2(0),
    end_date DATETIME2(0),
    server_name VARCHAR(128) NOT NULL,
    object_name VARCHAR(128) NOT NULL,
    counter_name VARCHAR(128) NOT NULL,
    instance_name VARCHAR(128) NULL,
    minimum_counter_value FLOAT NULL,
    maximum_counter_value FLOAT NULL,
    average_counter_value FLOAT NULL,
    standard_deviation_counter_value FLOAT NULL,
    CONSTRAINT PK_IndexingCountersBaseline
        PRIMARY KEY CLUSTERED (counter_baseline_id)
);
GO
```
**清单 15-3 性能计数器基线表**

### 填充基线数据

将值填充到 `dbo.IndexingCountersBaseline` 表中的过程有两个步骤。首先，需要从代表典型一周的性能计数器中收集样本。如果没有典型的一周，则选择本周并为其收集样本。收集到典型一周的数据后，下一步是将信息聚合到基线表中。这实质上是总结 `dbo.IndexingCounters` 表中特定日期范围内的信息。在清单 15-4 中，数据范围是从 2019 年 8 月 1 日到 8 月 15 日。接下来是验证基线。仅仅因为过去一周的平均值显示每秒转发记录数为 100，并不意味着该值代表了一个良好的基线。应使用服务器和数据库的经验来影响基线中的值。如果最近的趋势低于或高于正常水平，请根据需要调整基线。

```sql
USE IndexingMethod;
GO
DECLARE @StartDate DATETIME = '20190911',
        @EndDate       DATETIME = '20190918';
INSERT INTO dbo.IndexingCountersBaseline (
    start_date,
    end_date,
    server_name,
    object_name,
    counter_name,
    instance_name,
    minimum_counter_value,
    maximum_counter_value,
    average_counter_value,
    standard_deviation_counter_value
)
SELECT MIN(create_date),
       MAX(create_date),
       server_name,
       object_name,
       counter_name,
       instance_name,
       MIN(Calculated_Counter_value),
       MAX(Calculated_Counter_value),
       AVG(Calculated_Counter_value),
       STDEV(Calculated_Counter_value)
FROM dbo.IndexingCounters
WHERE create_date BETWEEN @StartDate AND @EndDate
GROUP BY server_name,
         object_name,
         counter_name,
         instance_name;
```
**清单 15-4 填充计数器基线表**

### 其他收集与查看工具

还有其他方法可以收集和查看 SQL Server 实例的性能计数器。Windows 应用程序性能监视器可用于实时查看性能计数器，也可用于将性能计数器记录到二进制或文本文件中。命令行实用程序 `Logman` 可用于与性能监视器交互，以创建数据收集器并根据需要启动和停止它们。此外，`PowerShell` 也可以协助收集性能计数器。

这些替代方案都是收集数据库和索引性能计数器的有效选择。关键在于，如果我们想要监控索引，就必须收集必要的信息，以便了解潜在的索引问题何时可能出现。选择一个最顺手的工具，并立即开始收集这些计数器。

## 动态管理对象

监控索引的一些最佳性能信息包含在动态管理对象（DMO）中。DMO 包含关于索引的逻辑和物理使用情况以及整体物理结构的信息。对于监控而言，有四个 DMO 可提供有关索引使用情况的信息：`sys.dm_db_index_usage_stats`、`sys.dm_db_index_operational_stats`、`sys.dm_db_index_physical_stats` 和 `sys.dm_os_wait_stats`。本节将介绍一个使用这些 DMO 监控索引的过程。

接下来的前三节将讨论 `sys.dm_db_index_*` 系列 DMO。第 5 章定义并演示了这些 DMO 的内容。关于这些 DMO 需要记住的一点是，它们可能会由于服务器上的各种操作（例如重启服务或重新创建索引）而被刷新。第四个 DMO `sys.dm_os_wait_stats` 与索引监控相关，并提供可在索引分析期间提供帮助的信息。

> **警告**
>
> 索引 DMO 没有行级别的信息来精确指示何时重置了为索引收集的信息。因此，可能存在报告的统计信息略高于或低于实际情况的情况。虽然这应该不会对分析结果产生重大影响，但需要牢记这一点。



### 索引使用统计信息

动态管理对象 `sys.dm_db_index_usage_stats` 提供了关于索引如何被使用以及索引最后被使用的时间的信息。当我们想要跟踪索引是否正在被使用以及针对它们执行了哪些操作时，这会非常有用。

对这个 DMO 的监控过程（与其他 DMO 类似）包括以下步骤：

1.  创建一个表来保存快照信息。
2.  将 DMO 的当前状态插入快照表。
3.  将最新的快照与前一个快照进行比较，并将行之间的差异量插入输出的历史表中。

要构建这个过程，我们首先需要创建快照表和历史表。这些表的结构将与 DMO 的架构相同，并包含 DMO 的所有列以及一个 `create_date` 列。为了与源 DMO 保持一致，表中的列将匹配 DMO 的架构。

```sql
USE IndexingMethod;
GO
CREATE TABLE dbo.index_usage_stats_snapshot
(
snapshot_id INT IDENTITY(1, 1),
create_date DATETIME2(0),
database_id SMALLINT NOT NULL,
object_id INT NOT NULL,
index_id INT NOT NULL,
user_seeks BIGINT NOT NULL,
user_scans BIGINT NOT NULL,
user_lookups BIGINT NOT NULL,
user_updates BIGINT NOT NULL,
last_user_seek DATETIME,
last_user_scan DATETIME,
last_user_lookup DATETIME,
last_user_update DATETIME,
system_seeks BIGINT NOT NULL,
system_scans BIGINT NOT NULL,
system_lookups BIGINT NOT NULL,
system_updates BIGINT NOT NULL,
last_system_seek DATETIME,
last_system_scan DATETIME,
last_system_lookup DATETIME,
last_system_update DATETIME,
CONSTRAINT PK_IndexUsageStatsSnapshot
PRIMARY KEY CLUSTERED (snapshot_id),
CONSTRAINT UQ_IndexUsageStatsSnapshot
UNIQUE (create_date, database_id, object_id, index_id)
);
CREATE TABLE dbo.index_usage_stats_history
(
history_id INT IDENTITY(1, 1),
create_date DATETIME2(0),
database_id SMALLINT NOT NULL,
object_id INT NOT NULL,
index_id INT NOT NULL,
user_seeks BIGINT NOT NULL,
user_scans BIGINT NOT NULL,
user_lookups BIGINT NOT NULL,
user_updates BIGINT NOT NULL,
last_user_seek DATETIME,
last_user_scan DATETIME,
last_user_lookup DATETIME,
last_user_update DATETIME,
system_seeks BIGINT NOT NULL,
system_scans BIGINT NOT NULL,
system_lookups BIGINT NOT NULL,
system_updates BIGINT NOT NULL,
last_system_seek DATETIME,
last_system_scan DATETIME,
last_system_lookup DATETIME,
last_system_update DATETIME,
CONSTRAINT PK_IndexUsageStatsHistory
PRIMARY KEY CLUSTERED (history_id),
CONSTRAINT UQ_IndexUsageStatsHistory
UNIQUE (create_date, database_id, object_id, index_id)
);
```
清单 15-5 索引使用统计信息快照表统计信息

捕获索引使用历史统计信息的下一个组件是收集 `sys.dm_db_index_usage_stats` 中的当前值。与性能监视器脚本类似，收集查询需要安排为大约每 4 小时运行一次。数据库环境的活动量以及索引被修改的频率应有助于确定捕获信息的频率。务必在任何索引碎片整理过程之前安排一次快照，以捕获在索引重建时可能丢失的信息。

```sql
USE IndexingMethod;
GO
INSERT INTO dbo.index_usage_stats_snapshot
SELECT GETDATE(),
database_id,
object_id,
index_id,
user_seeks,
user_scans,
user_lookups,
user_updates,
last_user_seek,
last_user_scan,
last_user_lookup,
last_user_update,
system_seeks,
system_scans,
system_lookups,
system_updates,
last_system_seek,
last_system_scan,
last_system_lookup,
last_system_update
FROM sys.dm_db_index_usage_stats;
```
清单 15-6 索引使用统计信息快照填充

填充索引使用统计信息的快照后，需要将最新快照和前一个快照之间的差异量插入到 `index_usage_stats_history` 表中。由于 `sys.dm_db_index_usage_stats` 的行中没有指示符来标识索引统计信息何时被重置，因此识别两个索引条目之间是否存在差异的过程是：如果索引上的任何统计信息返回负值，则删除该行。生成的查询实现了此逻辑，并同时删除了没有任何新活动发生的行。

```sql
USE IndexingMethod;
GO
WITH IndexUsageCTE
AS (SELECT DENSE_RANK() OVER (ORDER BY create_date DESC) AS HistoryID,
create_date,
database_id,
object_id,
index_id,
user_seeks,
user_scans,
user_lookups,
user_updates,
last_user_seek,
last_user_scan,
last_user_lookup,
last_user_update,
system_seeks,
system_scans,
system_lookups,
system_updates,
last_system_seek,
last_system_scan,
last_system_lookup,
last_system_update
FROM dbo.index_usage_stats_snapshot)
INSERT INTO dbo.index_usage_stats_history
SELECT i1.create_date,
i1.database_id,
i1.object_id,
i1.index_id,
i1.user_seeks - COALESCE(i2.user_seeks, 0),
i1.user_scans - COALESCE(i2.user_scans, 0),
i1.user_lookups - COALESCE(i2.user_lookups, 0),
i1.user_updates - COALESCE(i2.user_updates, 0),
i1.last_user_seek,
i1.last_user_scan,
i1.last_user_lookup,
i1.last_user_update,
i1.system_seeks - COALESCE(i2.system_seeks, 0),
i1.system_scans - COALESCE(i2.system_scans, 0),
i1.system_lookups - COALESCE(i2.system_lookups, 0),
i1.system_updates - COALESCE(i2.system_updates, 0),
i1.last_system_seek,
i1.last_system_scan,
i1.last_system_lookup,
i1.last_system_update
FROM IndexUsageCTE i1
LEFT OUTER JOIN IndexUsageCTE i2 ON i1.database_id = i2.database_id
AND i1.object_id   = i2.object_id
AND i1.index_id    = i2.index_id
AND i2.HistoryID   = 2
--验证没有行小于 0
AND NOT (
i1.system_seeks - COALESCE(i2.system_seeks, 0)  < 0
OR i1.system_scans - COALESCE(i2.system_scans, 0)     < 0
OR i1.system_lookups - COALESCE(i2.system_lookups, 0) < 0
OR i1.system_updates - COALESCE(i2.system_updates, 0) < 0
OR i1.user_seeks - COALESCE(i2.user_seeks, 0)         < 0
OR i1.user_scans - COALESCE(i2.user_scans, 0)         < 0
OR i1.user_lookups - COALESCE(i2.user_lookups, 0)     < 0
OR i1.user_updates - COALESCE(i2.user_updates, 0)     < 0
);
GO
```
清单 15-7 索引使用统计信息快照填充

### 索引操作统计信息

动态管理对象 `sys.dm_db_index_operational_stats` 提供了在查询计划执行期间发生在索引上的物理操作的信息。这些信息可用于跟踪索引被使用时发生的物理计划操作及其操作速率。该 DMO 监控的其他内容之一是压缩的成功率。

监控此 DMO 的过程涉及几个简单的步骤。首先，将创建表来存储 DMO 输出的快照和历史信息。然后，定期将 DMO 输出的快照插入快照表。检索快照后，当前快照与前一个快照之间的差异量将被插入历史表。

该过程使用一个与 `sys.dm_db_index_operational_stats` 架构几乎相同的快照表和历史表。架构的主要差异是添加了一个 `create_date` 列，用于标识快照发生的时间。以下代码提供了快照表和历史表所需的架构。

**注意**

列 `version_generated_inrow`、`version_generated_offrow`、`ghost_version_inrow`、`ghost_version_offrow`、`insert_over_ghost_version_inrow` 和 `insert_over_ghost_version_offrow` 是 SQL Server 2019 中新增的。如果在之前版本的 SQL Server 中使用此代码，应删除这些列。



```
USE IndexingMethod;
GO
CREATE TABLE dbo.index_operational_stats_snapshot
(
    snapshot_id INT IDENTITY(1, 1),
    create_date DATETIME2(0),
    database_id SMALLINT NOT NULL,
    object_id INT NOT NULL,
    index_id INT NOT NULL,
    partition_number INT NOT NULL,
    hobt_id BIGINT NOT NULL,
    leaf_insert_count BIGINT NOT NULL,
    leaf_delete_count BIGINT NOT NULL,
    leaf_update_count BIGINT NOT NULL,
    leaf_ghost_count BIGINT NOT NULL,
    nonleaf_insert_count BIGINT NOT NULL,
    nonleaf_delete_count BIGINT NOT NULL,
    nonleaf_update_count BIGINT NOT NULL,
    leaf_allocation_count BIGINT NOT NULL,
    nonleaf_allocation_count BIGINT NOT NULL,
    leaf_page_merge_count BIGINT NOT NULL,
    nonleaf_page_merge_count BIGINT NOT NULL,
    range_scan_count BIGINT NOT NULL,
    singleton_lookup_count BIGINT NOT NULL,
    forwarded_fetch_count BIGINT NOT NULL,
    lob_fetch_in_pages BIGINT NOT NULL,
    lob_fetch_in_bytes BIGINT NOT NULL,
    lob_orphan_create_count BIGINT NOT NULL,
    lob_orphan_insert_count BIGINT NOT NULL,
    row_overflow_fetch_in_pages BIGINT NOT NULL,
    row_overflow_fetch_in_bytes BIGINT NOT NULL,
    column_value_push_off_row_count BIGINT NOT NULL,
    column_value_pull_in_row_count BIGINT NOT NULL,
    row_lock_count BIGINT NOT NULL,
    row_lock_wait_count BIGINT NOT NULL,
    row_lock_wait_in_ms BIGINT NOT NULL,
    page_lock_count BIGINT NOT NULL,
    page_lock_wait_count BIGINT NOT NULL,
    page_lock_wait_in_ms BIGINT NOT NULL,
    index_lock_promotion_attempt_count BIGINT NOT NULL,
    index_lock_promotion_count BIGINT NOT NULL,
    page_latch_wait_count BIGINT NOT NULL,
    page_latch_wait_in_ms BIGINT NOT NULL,
    page_io_latch_wait_count BIGINT NOT NULL,
    page_io_latch_wait_in_ms BIGINT NOT NULL,
    tree_page_latch_wait_count BIGINT NOT NULL,
    tree_page_latch_wait_in_ms BIGINT NOT NULL,
    tree_page_io_latch_wait_count BIGINT NOT NULL,
    tree_page_io_latch_wait_in_ms BIGINT NOT NULL,
    page_compression_attempt_count BIGINT NOT NULL,
    page_compression_success_count BIGINT NOT NULL,
    version_generated_inrow BIGINT NOT NULL,
    version_generated_offrow BIGINT NOT NULL,
    ghost_version_inrow BIGINT NOT NULL,
    ghost_version_offrow BIGINT NOT NULL,
    insert_over_ghost_version_inrow BIGINT NOT NULL,
    insert_over_ghost_version_offrow BIGINT NOT NULL,
    CONSTRAINT PK_IndexOperationalStatsSnapshot
        PRIMARY KEY CLUSTERED (snapshot_id),
    CONSTRAINT UQ_IndexOperationalStatsSnapshot
        UNIQUE (create_date, database_id, object_id, index_id, partition_number)
);

CREATE TABLE dbo.index_operational_stats_history
(
    history_id INT IDENTITY(1, 1),
    create_date DATETIME2(0),
    database_id SMALLINT NOT NULL,
    object_id INT NOT NULL,
    index_id INT NOT NULL,
    partition_number INT NOT NULL,
    hobt_id BIGINT NOT NULL,
    leaf_insert_count BIGINT NOT NULL,
    leaf_delete_count BIGINT NOT NULL,
    leaf_update_count BIGINT NOT NULL,
    leaf_ghost_count BIGINT NOT NULL,
    nonleaf_insert_count BIGINT NOT NULL,
    nonleaf_delete_count BIGINT NOT NULL,
    nonleaf_update_count BIGINT NOT NULL,
    leaf_allocation_count BIGINT NOT NULL,
    nonleaf_allocation_count BIGINT NOT NULL,
    leaf_page_merge_count BIGINT NOT NULL,
    nonleaf_page_merge_count BIGINT NOT NULL,
    range_scan_count BIGINT NOT NULL,
    singleton_lookup_count BIGINT NOT NULL,
    forwarded_fetch_count BIGINT NOT NULL,
    lob_fetch_in_pages BIGINT NOT NULL,
    lob_fetch_in_bytes BIGINT NOT NULL,
    lob_orphan_create_count BIGINT NOT NULL,
    lob_orphan_insert_count BIGINT NOT NULL,
    row_overflow_fetch_in_pages BIGINT NOT NULL,
    row_overflow_fetch_in_bytes BIGINT NOT NULL,
    column_value_push_off_row_count BIGINT NOT NULL,
    column_value_pull_in_row_count BIGINT NOT NULL,
    row_lock_count BIGINT NOT NULL,
    row_lock_wait_count BIGINT NOT NULL,
    row_lock_wait_in_ms BIGINT NOT NULL,
    page_lock_count BIGINT NOT NULL,
    page_lock_wait_count BIGINT NOT NULL,
    page_lock_wait_in_ms BIGINT NOT NULL,
    index_lock_promotion_attempt_count BIGINT NOT NULL,
    index_lock_promotion_count BIGINT NOT NULL,
    page_latch_wait_count BIGINT NOT NULL,
    page_latch_wait_in_ms BIGINT NOT NULL,
    page_io_latch_wait_count BIGINT NOT NULL,
    page_io_latch_wait_in_ms BIGINT NOT NULL,
    tree_page_latch_wait_count BIGINT NOT NULL,
    tree_page_latch_wait_in_ms BIGINT NOT NULL,
    tree_page_io_latch_wait_count BIGINT NOT NULL,
    tree_page_io_latch_wait_in_ms BIGINT NOT NULL,
    page_compression_attempt_count BIGINT NOT NULL,
    page_compression_success_count BIGINT NOT NULL,
    version_generated_inrow BIGINT NOT NULL,
    version_generated_offrow BIGINT NOT NULL,
    ghost_version_inrow BIGINT NOT NULL,
    ghost_version_offrow BIGINT NOT NULL,
    insert_over_ghost_version_inrow BIGINT NOT NULL,
    insert_over_ghost_version_offrow BIGINT NOT NULL,
    CONSTRAINT PK_IndexOperationalStatsHistory
        PRIMARY KEY CLUSTERED (history_id),
    CONSTRAINT UQ_IndexOperationalStatsHistory
        UNIQUE (create_date, database_id, object_id, index_id, partition_number)
);
```
清单 15-8 索引操作统计信息快照表

表创建完成后，下一步是捕获 `sys.dm_db_index_operational_stats` 中信息的当前快照。可以使用清单 15-9 中的脚本来填充这些信息。由于**索引方法**旨在捕获服务器上所有数据库的索引信息，`sys.dm_db_index_operational_stats` 的参数值被设置为 `NULL`。这将返回服务器上所有数据库中所有表的所有索引的所有分区的结果。如果需要限制数据库、表或索引的列表，则可以添加筛选器以减少要收集的索引指标数量。与索引使用情况统计信息类似，此信息应大约每 4 小时捕获一次，其中一次计划捕获点应在服务器上的索引维护操作之前。

```
USE IndexingMethod;
GO
TRUNCATE TABLE dbo.index_operational_stats_snapshot

INSERT INTO dbo.index_operational_stats_snapshot
SELECT GETDATE(),
       database_id,
       object_id,
       index_id,
       partition_number,
       hobt_id,
       leaf_insert_count,
       leaf_delete_count,
       leaf_update_count,
       leaf_ghost_count,
       nonleaf_insert_count,
       nonleaf_delete_count,
       nonleaf_update_count,
       leaf_allocation_count,
       nonleaf_allocation_count,
       leaf_page_merge_count,
       nonleaf_page_merge_count,
       range_scan_count,
       singleton_lookup_count,
       forwarded_fetch_count,
       lob_fetch_in_pages,
       lob_fetch_in_bytes,
       lob_orphan_create_count,
       lob_orphan_insert_count,
       row_overflow_fetch_in_pages,
       row_overflow_fetch_in_bytes,
       column_value_push_off_row_count,
       column_value_pull_in_row_count,
       row_lock_count,
       row_lock_wait_count,
       row_lock_wait_in_ms,
       page_lock_count,
       page_lock_wait_count,
       page_lock_wait_in_ms,
       index_lock_promotion_attempt_count,
       index_lock_promotion_count,
       page_latch_wait_count,
       page_latch_wait_in_ms,
       page_io_latch_wait_count,
       page_io_latch_wait_in_ms,
       tree_page_latch_wait_count,
       tree_page_latch_wait_in_ms,
       tree_page_io_latch_wait_count,
       tree_page_io_latch_wait_in_ms,
       page_compression_attempt_count,
       page_compression_success_count,
       version_generated_inrow,
       version_generated_offrow,
       ghost_version_inrow,
       ghost_version_offrow,
       insert_over_ghost_version_inrow,
       insert_over_ghost_version_offrow
FROM sys.dm_db_index_operational_stats(NULL, NULL, NULL, NULL)
WHERE database_id > 4;
```
清单 15-9 填充索引操作统计信息快照

填充快照表之后，下一步是填充历史表。与之前一样，历史表的目的是存储两个快照之间统计信息**差值**。这些差值提供了发生了哪些操作的信息，并且还有助于将这些操作限定在特定时间范围内，以便在需要时，可以将更多重点放在核心时段或非核心时段的这些操作上。识别统计信息何时被重置的业务规则与索引使用情况统计信息类似：如果索引上的任何统计信息返回负值，则将忽略前一个快照中的那一行。此外，任何返回全零值的行也不会被包含在内。清单 15-10 展示了用于生成历史差值的代码。



#### 清单 15-10: 索引操作统计快照填充

```sql
USE IndexingMethod;
GO
WITH IndexOperationalCTE
AS (SELECT DENSE_RANK() OVER (ORDER BY create_date DESC) AS HistoryID,
create_date,
database_id,
object_id,
index_id,
partition_number,
hobt_id,
leaf_insert_count,
leaf_delete_count,
leaf_update_count,
leaf_ghost_count,
nonleaf_insert_count,
nonleaf_delete_count,
nonleaf_update_count,
leaf_allocation_count,
nonleaf_allocation_count,
leaf_page_merge_count,
nonleaf_page_merge_count,
range_scan_count,
singleton_lookup_count,
forwarded_fetch_count,
lob_fetch_in_pages,
lob_fetch_in_bytes,
lob_orphan_create_count,
lob_orphan_insert_count,
row_overflow_fetch_in_pages,
row_overflow_fetch_in_bytes,
column_value_push_off_row_count,
column_value_pull_in_row_count,
row_lock_count,
row_lock_wait_count,
row_lock_wait_in_ms,
page_lock_count,
page_lock_wait_count,
page_lock_wait_in_ms,
index_lock_promotion_attempt_count,
index_lock_promotion_count,
page_latch_wait_count,
page_latch_wait_in_ms,
page_io_latch_wait_count,
page_io_latch_wait_in_ms,
tree_page_latch_wait_count,
tree_page_latch_wait_in_ms,
tree_page_io_latch_wait_count,
tree_page_io_latch_wait_in_ms,
page_compression_attempt_count,
page_compression_success_count,
version_generated_inrow,
version_generated_offrow,
ghost_version_inrow,
ghost_version_offrow,
insert_over_ghost_version_inrow,
insert_over_ghost_version_offrow
FROM dbo.index_operational_stats_snapshot)
INSERT INTO dbo.index_operational_stats_history
SELECT i1.create_date,
i1.database_id,
i1.object_id,
i1.index_id,
i1.partition_number,
i1.hobt_id,
i1.leaf_insert_count - COALESCE(i2.leaf_insert_count, 0),
i1.leaf_delete_count - COALESCE(i2.leaf_delete_count, 0),
i1.leaf_update_count - COALESCE(i2.leaf_update_count, 0),
i1.leaf_ghost_count - COALESCE(i2.leaf_ghost_count, 0),
i1.nonleaf_insert_count - COALESCE(i2.nonleaf_insert_count, 0),
i1.nonleaf_delete_count - COALESCE(i2.nonleaf_delete_count, 0),
i1.nonleaf_update_count - COALESCE(i2.nonleaf_update_count, 0),
i1.leaf_allocation_count - COALESCE(i2.leaf_allocation_count, 0),
i1.nonleaf_allocation_count - COALESCE(i2.nonleaf_allocation_count, 0),
i1.leaf_page_merge_count - COALESCE(i2.leaf_page_merge_count, 0),
i1.nonleaf_page_merge_count - COALESCE(i2.nonleaf_page_merge_count, 0),
i1.range_scan_count - COALESCE(i2.range_scan_count, 0),
i1.singleton_lookup_count - COALESCE(i2.singleton_lookup_count, 0),
i1.forwarded_fetch_count - COALESCE(i2.forwarded_fetch_count, 0),
i1.lob_fetch_in_pages - COALESCE(i2.lob_fetch_in_pages, 0),
i1.lob_fetch_in_bytes - COALESCE(i2.lob_fetch_in_bytes, 0),
i1.lob_orphan_create_count - COALESCE(i2.lob_orphan_create_count, 0),
i1.lob_orphan_insert_count - COALESCE(i2.lob_orphan_insert_count, 0),
i1.row_overflow_fetch_in_pages - COALESCE(i2.row_overflow_fetch_in_pages, 0),
i1.row_overflow_fetch_in_bytes - COALESCE(i2.row_overflow_fetch_in_bytes, 0),
i1.column_value_push_off_row_count - COALESCE(i2.column_value_push_off_row_count, 0),
i1.column_value_pull_in_row_count - COALESCE(i2.column_value_pull_in_row_count, 0),
i1.row_lock_count - COALESCE(i2.row_lock_count, 0),
i1.row_lock_wait_count - COALESCE(i2.row_lock_wait_count, 0),
i1.row_lock_wait_in_ms - COALESCE(i2.row_lock_wait_in_ms, 0),
i1.page_lock_count - COALESCE(i2.page_lock_count, 0),
i1.page_lock_wait_count - COALESCE(i2.page_lock_wait_count, 0),
i1.page_lock_wait_in_ms - COALESCE(i2.page_lock_wait_in_ms, 0),
i1.index_lock_promotion_attempt_count - COALESCE(i2.index_lock_promotion_attempt_count, 0),
i1.index_lock_promotion_count - COALESCE(i2.index_lock_promotion_count, 0),
i1.page_latch_wait_count - COALESCE(i2.page_latch_wait_count, 0),
i1.page_latch_wait_in_ms - COALESCE(i2.page_latch_wait_in_ms, 0),
i1.page_io_latch_wait_count - COALESCE(i2.page_io_latch_wait_count, 0),
i1.page_io_latch_wait_in_ms - COALESCE(i2.page_io_latch_wait_in_ms, 0),
i1.tree_page_latch_wait_count - COALESCE(i2.tree_page_latch_wait_count, 0),
i1.tree_page_latch_wait_in_ms - COALESCE(i2.tree_page_latch_wait_in_ms, 0),
i1.tree_page_io_latch_wait_count - COALESCE(i2.tree_page_io_latch_wait_count, 0),
i1.tree_page_io_latch_wait_in_ms - COALESCE(i2.tree_page_io_latch_wait_in_ms, 0),
i1.page_compression_attempt_count - COALESCE(i2.page_compression_attempt_count, 0),
i1.page_compression_success_count - COALESCE(i2.page_compression_success_count, 0),
i1.version_generated_inrow - COALESCE(i2.version_generated_inrow, 0),
i1.version_generated_offrow - COALESCE(i2.version_generated_offrow, 0),
i1.ghost_version_inrow - COALESCE(i2.ghost_version_inrow, 0),
i1.ghost_version_offrow - COALESCE(i2.ghost_version_offrow, 0),
i1.insert_over_ghost_version_inrow - COALESCE(i2.insert_over_ghost_version_inrow, 0),
i1.insert_over_ghost_version_offrow - COALESCE(i2.insert_over_ghost_version_offrow, 0)
FROM IndexOperationalCTE i1
LEFT OUTER JOIN IndexOperationalCTE i2 ON i1.database_id = i2.database_id
AND i1.object_id = i2.object_id
AND i1.index_id = i2.index_id
AND i1.partition_number = i2.partition_number
AND i2.HistoryID = 2
--验证没有行数小于 0
AND NOT (i1.leaf_insert_count - COALESCE(i2.leaf_insert_count, 0)  0
OR i1.leaf_delete_count - COALESCE(i2.leaf_delete_count, 0) > 0
OR i1.leaf_update_count - COALESCE(i2.leaf_update_count, 0) > 0
OR i1.leaf_ghost_count - COALESCE(i2.leaf_ghost_count, 0) > 0
OR i1.nonleaf_insert_count - COALESCE(i2.nonleaf_insert_count, 0) > 0
OR i1.nonleaf_delete_count - COALESCE(i2.nonleaf_delete_count, 0) > 0
OR i1.nonleaf_update_count - COALESCE(i2.nonleaf_update_count, 0) > 0
OR i1.leaf_allocation_count - COALESCE(i2.leaf_allocation_count, 0) > 0
OR i1.nonleaf_allocation_count - COALESCE(i2.nonleaf_allocation_count, 0) > 0
OR i1.leaf_page_merge_count - COALESCE(i2.leaf_page_merge_count, 0) > 0
OR i1.nonleaf_page_merge_count - COALESCE(i2.nonleaf_page_merge_count, 0) > 0
OR i1.range_scan_count - COALESCE(i2.range_scan_count, 0) > 0
OR i1.singleton_lookup_count - COALESCE(i2.singleton_lookup_count, 0) > 0
OR i1.forwarded_fetch_count - COALESCE(i2.forwarded_fetch_count, 0) > 0
OR i1.lob_fetch_in_pages - COALESCE(i2.lob_fetch_in_pages, 0) > 0
OR i1.lob_fetch_in_bytes - COALESCE(i2.lob_fetch_in_bytes, 0) > 0
OR i1.lob_orphan_create_count - COALESCE(i2.lob_orphan_create_count, 0) > 0
OR i1.lob_orphan_insert_count - COALESCE(i2.lob_orphan_insert_count, 0) > 0
OR i1.row_overflow_fetch_in_pages - COALESCE(i2.row_overflow_fetch_in_pages, 0) > 0
OR i1.row_overflow_fetch_in_bytes - COALESCE(i2.row_overflow_fetch_in_bytes, 0) > 0
OR i1.column_value_push_off_row_count - COALESCE(i2.column_value_push_off_row_count, 0) > 0
OR i1.column_value_pull_in_row_count - COALESCE(i2.column_value_pull_in_row_count, 0) > 0
OR i1.row_lock_count - COALESCE(i2.row_lock_count, 0) > 0
OR i1.row_lock_wait_count - COALESCE(i2.row_lock_wait_count, 0) > 0
OR i1.row_lock_wait_in_ms - COALESCE(i2.row_lock_wait_in_ms, 0) > 0
OR i1.page_lock_count - COALESCE(i2.page_lock_count, 0) > 0
OR i1.page_lock_wait_count - COALESCE(i2.page_lock_wait_count, 0) > 0
OR i1.page_lock_wait_in_ms - COALESCE(i2.page_lock_wait_in_ms, 0) > 0
OR i1.index_lock_promotion_attempt_count - COALESCE(i2.index_lock_promotion_attempt_count, 0) > 0
OR i1.index_lock_promotion_count - COALESCE(i2.index_lock_promotion_count, 0) > 0
OR i1.page_latch_wait_count - COALESCE(i2.page_latch_wait_count, 0) > 0
OR i1.page_latch_wait_in_ms - COALESCE(i2.page_latch_wait_in_ms, 0) > 0
OR i1.page_io_latch_wait_count - COALESCE(i2.page_io_latch_wait_count, 0) > 0
OR i1.page_io_latch_wait_in_ms - COALESCE(i2.page_io_latch_wait_in_ms, 0) > 0
OR i1.tree_page_latch_wait_count - COALESCE(i2.tree_page_latch_wait_count, 0) > 0
OR i1.tree_page_latch_wait_in_ms - COALESCE(i2.tree_page_latch_wait_in_ms, 0) > 0
OR i1.tree_page_io_latch_wait_count - COALESCE(i2.tree_page_io_latch_wait_count, 0) > 0
OR i1.tree_page_io_latch_wait_in_ms - COALESCE(i2.tree_page_io_latch_wait_in_ms, 0) > 0
OR i1.page_compression_attempt_count - COALESCE(i2.page_compression_attempt_count, 0) > 0
OR i1.page_compression_success_count - COALESCE(i2.page_compression_success_count, 0) > 0
OR i1.version_generated_inrow - COALESCE(i2.version_generated_inrow, 0) > 0
OR i1.version_generated_offrow - COALESCE(i2.version_generated_offrow, 0) > 0
OR i1.ghost_version_inrow - COALESCE(i2.ghost_version_inrow, 0) > 0
OR i1.ghost_version_offrow - COALESCE(i2.ghost_version_offrow, 0) > 0
OR i1.insert_over_ghost_version_inrow - COALESCE(i2.insert_over_ghost_version_inrow, 0) > 0
OR i1.insert_over_ghost_version_offrow - COALESCE(i2.insert_over_ghost_version_offrow, 0) > 0
);
```


### 索引物理统计信息

用于监控索引的最后一个动态管理对象是 `sys.dm_db_index_physical_stats`。该动态管理对象提供数据库中索引当前物理结构的统计信息。这些信息的价值在于确定索引的碎片化程度，这一点将在第 8 章中更详细地讨论。从监控的角度来看，收集物理统计信息是为了辅助后续分析。其目标是识别可能影响索引存储效率的潜在问题，或者反之，从而影响查询性能。

对于物理统计信息动态管理对象，统计信息的收集方式与其他动态管理对象略有不同。此动态管理对象与其他对象的主要区别在于收集信息时可能对数据库造成的影响。与另外两个引用内存中表的动态管理对象不同，`index_physical_stats` 会读取索引中的页面以确定索引的实际碎片化和物理布局。有关使用 `sys.dm_db_index_physical_stats` 的影响的更多信息，请参考第 5 章。为了适应这种差异，统计信息仅存储在历史表中；不会计算检索历史记录之间的时间点差异。此外，由于动态管理对象中包含的统计信息的性质，计算差异值几乎没有价值。

开始收集索引物理统计信息所需的第一个要素是前面提到的历史表。如代码清单 15-11 所示，该表使用与动态管理对象相同的架构，并增加了 `create_date` 列。

**提示**

在生成动态管理对象所需的表架构时，使用了 SQL Server 2012 中引入的表值函数。函数 `sys.dm_exec_describe_first_result_set` 可用于标识查询的列名和数据类型。

```
USE IndexingMethod;
GO
CREATE TABLE dbo.index_physical_stats_history
(
history_id INT IDENTITY(1, 1),
create_date DATETIME2(0),
database_id SMALLINT,
object_id INT,
index_id INT,
partition_number INT,
index_type_desc NVARCHAR(60),
alloc_unit_type_desc NVARCHAR(60),
index_depth TINYINT,
index_level TINYINT,
avg_fragmentation_in_percent FLOAT,
fragment_count BIGINT,
avg_fragment_size_in_pages FLOAT,
page_count BIGINT,
avg_page_space_used_in_percent FLOAT,
record_count BIGINT,
ghost_record_count BIGINT,
version_ghost_record_count BIGINT,
min_record_size_in_bytes INT,
max_record_size_in_bytes INT,
avg_record_size_in_bytes FLOAT,
forwarded_record_count BIGINT,
compressed_page_count BIGINT,
hobt_id BIGINT NULL,
columnstore_delete_buffer_state TINYINT NULL,
columnstore_delete_buffer_state_desc NVARCHAR(60) NULL,
version_record_count BIGINT NULL,
inrow_version_record_count BIGINT NULL,
inrow_diff_version_record_count BIGINT NULL,
total_inrow_version_payload_size_in_bytes BIGINT NULL,
offrow_regular_version_record_count BIGINT NULL,
offrow_long_term_version_record_count BIGINT NULL,
CONSTRAINT PK_IndexPhysicalStatsHistory
PRIMARY KEY CLUSTERED (history_id),
CONSTRAINT UQ_IndexPhysicalStatsHistory
UNIQUE
(
create_date,
database_id,
object_id,
index_id,
partition_number,
alloc_unit_type_desc,
index_depth,
index_level
)
);
代码清单 15-11
索引物理统计信息历史表
```

`index_physical_stats` 的历史记录收集方式与前两个动态管理对象不同。由于它仅记录历史，因此无需捕获快照信息来为历史记录构建两个快照之间的差异。相反，当前统计信息会直接插入历史表，如代码清单 15-12 所示。此外，由于 `index_physical_stats` 在收集统计信息时会对索引执行物理操作，因此在生成历史信息时需要注意几点。首先，脚本将通过 `CURSOR` 驱动的循环独立于其他数据库从每个数据库收集信息。这为每个数据库的统计信息收集提供了批处理分离，并限制了动态管理对象的影响。其次，查询应在非核心时段执行。每日维护窗口的开始时间最为理想。重要的是，此信息应在碎片整理或重新索引之前收集，因为这些操作将更改动态管理对象提供的信息。通常，此信息是碎片整理过程中的一个步骤收集的，这将在第 8 章中讨论。如果是这样，则无需收集两次信息。为碎片整理收集并存储，以供以后监控索引使用。

```
USE IndexingMethod;
GO
DECLARE @DatabaseID INT;
DECLARE DatabaseList CURSOR FAST_FORWARD FOR
SELECT database_id
FROM sys.databases
WHERE state_desc = 'ONLINE'
AND database_id > 4;
OPEN DatabaseList;
FETCH NEXT FROM DatabaseList
INTO @DatabaseID;
WHILE @@FETCH_STATUS = 0
BEGIN
INSERT INTO dbo.index_physical_stats_history (
create_date,
database_id,
object_id,
index_id,
partition_number,
index_type_desc,
alloc_unit_type_desc,
index_depth,
index_level,
avg_fragmentation_in_percent,
fragment_count,
avg_fragment_size_in_pages,
page_count,
avg_page_space_used_in_percent,
record_count,
ghost_record_count,
version_ghost_record_count,
min_record_size_in_bytes,
max_record_size_in_bytes,
avg_record_size_in_bytes,
forwarded_record_count,
compressed_page_count,
hobt_id,
columnstore_delete_buffer_state,
columnstore_delete_buffer_state_desc,
version_record_count,
inrow_version_record_count,
inrow_diff_version_record_count,
total_inrow_version_payload_size_in_bytes,
offrow_regular_version_record_count,
offrow_long_term_version_record_count
)
SELECT GETDATE(),
database_id,
object_id,
index_id,
partition_number,
index_type_desc,
alloc_unit_type_desc,
index_depth,
index_level,
avg_fragmentation_in_percent,
fragment_count,
avg_fragment_size_in_pages,
page_count,
avg_page_space_used_in_percent,
record_count,
ghost_record_count,
version_ghost_record_count,
min_record_size_in_bytes,
max_record_size_in_bytes,
avg_record_size_in_bytes,
forwarded_record_count,
compressed_page_count,
hobt_id,
columnstore_delete_buffer_state,
columnstore_delete_buffer_state_desc,
version_record_count,
inrow_version_record_count,
inrow_diff_version_record_count,
total_inrow_version_payload_size_in_bytes,
offrow_regular_version_record_count,
offrow_long_term_version_record_count
FROM sys.dm_db_index_physical_stats(@DatabaseID, NULL, NULL, NULL, 'SAMPLED');
FETCH NEXT FROM DatabaseList
INTO @DatabaseID;
END;
CLOSE DatabaseList;
DEALLOCATE DatabaseList;
代码清单 15-12
填充索引物理统计信息历史表
```

## 等待统计信息

另一个提供索引相关信息的 DMO 是`sys.dm_os_wait_stats`。此 DMO 收集 SQL Server 为启动或继续执行查询或其他请求而正在等待的资源相关信息。大多数性能调优方法都包含收集和分析等待统计信息的过程。从索引角度来看，有许多等待资源可以表明 SQL Server 实例上可能存在索引问题。通过监控这些统计信息，我们可以在这些问题可能存在时得到通知。表 15-2 提供了一个最常表明可能存在索引问题的等待类型简短列表。

### 表 15-2: 与索引相关的等待统计信息

| 选项名 | 描述 |
| --- | --- |
| `CXPACKET` | 同步参与并行查询的线程。此等待类型意味着并行查询正在尝试同步并行查询中操作员之间的数据，可能表示工作负载不平衡或工作线程被前置请求阻塞。 |
| `IO_COMPLETION` | 表示等待诸如排序等各种情况的 I/O 操作（通常是同步 I/O）。此等待类型代表非数据页 I/O。 |
| `LCK_M_*` | 当任务正在等待获取索引或表上的锁时发生。 |
| `PAGEIOLATCH_*` | 当任务正在等待 I/O 请求中的缓冲区的闩锁时发生。长时间等待可能表明磁盘子系统存在问题。 |

与性能计数器类似，等待统计信息是反映 SQL Server 实例整体信息的一般健康指标。它们不直接指向资源；相反，它们收集的是 SQL Server 实例上等待特定资源时的信息。

> **注意**
> 许多第三方供应商的性能监控工具将收集等待统计信息作为其监控的一部分。如果您的环境中已经安装了工具，请检查是否可以从该工具中检索等待统计信息。

收集等待统计信息遵循使用快照表和历史表的模式。为此，数据将首先收集在快照表中，快照之间的增量存储在历史表中。如清单 15-13 所示的快照和历史表包含支持快照和历史模式所需的列。

```sql
USE IndexingMethod;
GO
CREATE TABLE dbo.wait_stats_snapshot
(
wait_stats_snapshot_id INT IDENTITY(1, 1),
create_date DATETIME2(0),
wait_type NVARCHAR(60) NOT NULL,
waiting_tasks_count BIGINT NOT NULL,
wait_time_ms BIGINT NOT NULL,
max_wait_time_ms BIGINT NOT NULL,
signal_wait_time_ms BIGINT NOT NULL,
CONSTRAINT PK_wait_stats_snapshot
PRIMARY KEY CLUSTERED (wait_stats_snapshot_id)
);
CREATE TABLE dbo.wait_stats_history
(
wait_stats_history_id INT IDENTITY(1, 1),
create_date DATETIME2(0),
wait_type NVARCHAR(60) NOT NULL,
waiting_tasks_count BIGINT NOT NULL,
wait_time_ms BIGINT NOT NULL,
max_wait_time_ms BIGINT NOT NULL,
signal_wait_time_ms BIGINT NOT NULL,
CONSTRAINT PK_wait_stats_history
PRIMARY KEY CLUSTERED (wait_stats_history_id)
);
```

**清单 15-13**
等待统计信息快照和历史表

要收集等待统计信息，需要查询`sys.dm_os_wait_stats`的输出。与本章讨论的其他 DMO 不同，在插入数据之前需要对信息进行一些汇总。在之前版本的 SQL Server 中，`wait_stats` DMO 包含等待类型`MISCELLANEOUS`的两行。为了适应这种差异，清单 15-14 中的示例脚本使用聚合来解决此问题。`wait_stats_snapshot`与其他快照之间的另一个区别是信息收集的频率。`Wait_stats`报告请求资源不可用时的信息。能够将此信息与一天中的特定时间联系起来可能至关重要。因此，`wait_stats`信息应大约每小时收集一次。在较繁忙的环境中可以更频繁地收集，以增加粒度，了解重要事件发生的时间。

```sql
USE IndexingMethod;
GO
TRUNCATE TABLE dbo.wait_stats_snapshot
INSERT INTO dbo.wait_stats_snapshot (
create_date,
wait_type,
waiting_tasks_count,
wait_time_ms,
max_wait_time_ms,
signal_wait_time_ms
)
SELECT GETDATE(),
wait_type,
waiting_tasks_count,
wait_time_ms,
max_wait_time_ms,
signal_wait_time_ms
FROM sys.dm_os_wait_stats;
```

**清单 15-14**
等待统计信息快照填充

每次收集快照时，都需要将其与上一个快照之间的增量添加到`wait_stats_history`表中。为了确定`sys.dm_os_wait_stats`中的信息何时被重置，使用了`waiting_tasks_count`列。如果该列中的值低于上一个快照，则 DMO 中的信息已被重置。清单 15-15 提供了填充历史表的代码。

虽然只有少数等待类型指向索引问题，但历史表将显示所有遇到的等待类型的结果。原因是需要将资源上的等待与发生的其他等待总数进行比较。例如，如果`CXPACKET`是服务器上相对最低的等待，那么研究查询并确定可以减少此等待类型发生的索引就没有太大价值，因为其他问题可能更显著地影响性能。

```sql
USE IndexingMethod;
GO
WITH WaitStatCTE
AS (SELECT create_date,
DENSE_RANK() OVER (ORDER BY create_date DESC) AS HistoryID,
wait_type,
waiting_tasks_count,
wait_time_ms,
max_wait_time_ms,
signal_wait_time_ms
FROM dbo.wait_stats_snapshot)
INSERT INTO dbo.wait_stats_history
SELECT w1.create_date,
w1.wait_type,
w1.waiting_tasks_count - COALESCE(w2.waiting_tasks_count, 0),
w1.wait_time_ms - COALESCE(w2.wait_time_ms, 0),
w1.max_wait_time_ms - COALESCE(w2.max_wait_time_ms, 0),
w1.signal_wait_time_ms - COALESCE(w2.signal_wait_time_ms, 0)
FROM WaitStatCTE w1
LEFT OUTER JOIN WaitStatCTE w2 ON w1.wait_type = w2.wait_type
AND w1.waiting_tasks_count >= COALESCE(w2.waiting_tasks_count, 0)
AND w2.HistoryID = 2
WHERE w1.HistoryID = 1
AND w1.waiting_tasks_count - COALESCE(w2.waiting_tasks_count, 0) > 0;
```

**清单 15-15**
等待统计信息历史填充


### 数据清理

尽管监控所需的所有信息对索引分析都是必要的，但这些信息并非需要无限期保留。若没有在合理时间后清理已收集信息的任务，监控流程就不算完整。一个普遍可接受的信息清理计划是：在 3 天后清除快照信息，在 90 天后清除历史信息。如果需要将历史数据保留更长时间，可以据此进行调整。

快照信息仅用于准备历史信息，一旦增量创建完成就不再需要。由于 SQL Agent 作业可能出错，且收集点可能与上一次相隔一天，通常 3 天的窗口期能为流程提供所需余地，并适应可能出现的任何问题。

历史表中的数据比快照信息更为关键，需要保留更长时间。这些信息为索引分析期间的活动提供数据支持。保留此信息的窗口期应与完成索引方法三次或更多次迭代通常所需的时间相匹配。这样，保留的信息可以在流程的几个周期中作为参考。

安排清理流程时，应至少每天执行一次，并在非核心处理时段进行。这将最小化每次执行时删除的信息量，并减少删除操作与服务器上其他活动可能产生的争用。如代码清单 15-16 所示的删除脚本涵盖了本节讨论的所有表。

```sql
USE IndexingMethod
GO
DECLARE @SnapshotDays INT = 3
,@HistoryDays INT = 90
DELETE FROM dbo.index_usage_stats_snapshot
WHERE create_date < DATEADD(d, -@SnapshotDays, GETDATE())
DELETE FROM dbo.index_usage_stats_history
WHERE create_date < DATEADD(d, -@HistoryDays, GETDATE())
DELETE FROM dbo.index_operational_stats_snapshot
WHERE create_date < DATEADD(d, -@SnapshotDays, GETDATE())
DELETE FROM dbo.index_operational_stats_history
WHERE create_date < DATEADD(d, -@HistoryDays, GETDATE())
DELETE FROM dbo.index_physical_stats_history
WHERE create_date < DATEADD(d, -@HistoryDays, GETDATE())
DELETE FROM dbo.wait_stats_snapshot
WHERE create_date < DATEADD(d, -@SnapshotDays, GETDATE())
DELETE FROM dbo.wait_stats_history
WHERE create_date < DATEADD(d, -@HistoryDays, GETDATE())
```

代码清单 15-16
索引监控快照与历史记录清理

### 事件追踪

可以为监控索引收集的最后一套信息是事件追踪。追踪信息收集代表生产活动的 SQL 语句，这些语句可用于索引分析，以根据生产环境中的查询活动及其中存储的数据来识别可能有用的索引。虽然迄今为止收集的统计信息提供了活动对索引的影响以及 SQL Server 实例上其他资源使用情况的信息，但事件追踪收集的是导致这些统计信息产生活动的原因。对于 SQL Server，有两种方法可用于收集事件追踪数据：

*   SQL Trace

*   扩展事件

为了完整性，两种方法都将讨论。在我看来，在 SQL Server 中应仅使用扩展事件来收集事件追踪数据。这是因为扩展事件与 SQL Server 的集成度很高，有助于防止其在监控时引起性能问题。而且它能检索的详细程度远超 SQL Trace 的能力。

#### SQL Trace

SQL Trace 以及由此衍生的 SQL Profiler 是 SQL Server 最初的追踪工具。它是 DBA 最常用的工具之一，可以轻松收集 SQL Server 中的事件。使用 SQL Trace 收集信息时需要注意几个方面。首先，SQL Trace 很可能会收集大量信息，需要为此做好准备。换句话说，服务器和数据库越活跃，追踪（`.trc`）文件就越大。过滤事件有助于确保只返回相关细节。同样，不要在已经使用率很高或专用于数据或事务日志文件的驱动器上收集追踪信息。这样做可能（并且很可能）会影响这些驱动器上的 I/O 性能。监控的最终目标是提高系统性能；需要尽量减小监控的影响。

最后，SQL Trace 和 SQL Profiler 在 SQL Server 2012 中已被弃用。这并不意味着这些工具不再起用，但它们计划在未来的 SQL Server 版本中移除。尽管 SQL Trace 已被弃用，但在某些场景下，例如为数据库引擎优化顾问构建工作负载时，它仍然是收集追踪信息最方便的工具。

注意
建议随时了解 SQL Server 中的已弃用功能。有关已弃用功能的更多信息，请参阅 SQL Docs 的 [`https://learn.microsoft.com/en-us/sql/database-engine/deprecated-database-engine-features-in-sql-server-2022`](https://learn.microsoft.com/en-us/sql/database-engine/deprecated-database-engine-features-in-sql-server-2022)。

创建 SQL Trace 会话有四个基本步骤：

1.  构建追踪会话。

2.  为会话分配事件和列。

3.  向会话添加筛选器。

4.  启动 SQL Trace 会话。

接下来的几页将介绍这些步骤，并描述在 SQL Server 2022 中创建 SQL Trace 会话所使用的组件。该脚本也适用于早期版本的 SQL Server。

要开始使用 SQL Trace 进行监控，必须首先创建一个追踪会话。会话使用 `sp_trace_create` 存储过程创建。该过程接受多个参数来配置会话如何收集信息。在代码清单 15-17 所示的示例会话中，SQL Trace 会话将创建文件，当文件大小达到 50 MB 限制时自动滚动更新。限制文件大小是为了便于文件管理。在大多数环境中，复制许多 50 MB 的文件比复制 1 GB 或更大的文件更容易。如果偏好更大的文件而非大量小文件，可以调整此大小。追踪文件创建在 `c:\temp` 目录下，文件名为 `IndexingMethod`。如果此文件夹不存在，请务必创建。注意，此名称可以更改为适合监控设置所在服务器和数据库需求的任何名称。

```sql
USE master;
GO
DECLARE @rc INT,
@TraceID INT,
--Maximum .trc file size in MB
@maxfilesize BIGINT = 50,
--File name and path, minus the extension
@FileName NVARCHAR(256) = N'c:\temp\IndexingMethod';
EXEC @rc = sp_trace_create @TraceID OUTPUT, 0, @FileName, @maxfilesize, NULL;
IF (@rc  0)
RAISERROR('Error creating trace file', 16, 1);
SELECT *
FROM sys.traces
WHERE id = @TraceID;
```

代码清单 15-17
创建 SQL Trace 会话



在创建了 SQL Trace 会话后，下一步是向会话中添加事件。有两种事件对于索引监控最有价值：`RPC:Completed` 和 `SQL:BatchCompleted`。`RPC:Completed` 在远程过程调用完成时返回结果；其最佳示例是存储过程的完成。另一个事件 `SQL:BatchCompleted` 在即席（ad hoc）和预准备（prepared）批处理完成时发生。通过这两个事件，将收集服务器上所有已完成的 SQL 语句。

要向 SQL Trace 会话添加事件，需使用存储过程 `sp_trace_setevent`。该存储过程添加事件以及从事件中请求的列到跟踪中，每次执行存储过程都会添加一对事件和列。对于两个事件，每个事件有 15 列，则需要执行该存储过程 30 次。对于示例会话（如清单 15-18 所示），每个会话收集以下列：

*   `ApplicationName`
*   `ClientProcessID`
*   `CPU`
*   `DatabaseID`
*   `DatabaseName`
*   `Duration`
*   `EndTime`
*   `HostName`
*   `LoginName`
*   `NTUserName`
*   `Reads`
*   `SPID`
*   `StartTime`
*   `TextData`
*   `Writes`

事件和列的代码可以在系统目录视图中找到。事件列在视图 `sys.trace_events` 中。可用的列列在 `sys.trace_columns` 中。列视图还包括一个指示符，用于标识是否可以筛选该列的值，这在创建 SQL Trace 会话的下一步中很有用。

```sql
USE master;
GO
DECLARE @on INT = 1,
@FileName NVARCHAR(256) = N'c:\temp\IndexingMethod',
@TraceID INT;
SET @TraceID = (
SELECT id FROM sys.traces WHERE path LIKE @FileName + '%'
);
-- RPC:Completed
EXEC sp_trace_setevent @TraceID, 10, 1, @on;
EXEC sp_trace_setevent @TraceID, 10, 10, @on;
EXEC sp_trace_setevent @TraceID, 10, 11, @on;
EXEC sp_trace_setevent @TraceID, 10, 12, @on;
EXEC sp_trace_setevent @TraceID, 10, 13, @on;
EXEC sp_trace_setevent @TraceID, 10, 14, @on;
EXEC sp_trace_setevent @TraceID, 10, 15, @on;
EXEC sp_trace_setevent @TraceID, 10, 16, @on;
EXEC sp_trace_setevent @TraceID, 10, 17, @on;
EXEC sp_trace_setevent @TraceID, 10, 18, @on;
EXEC sp_trace_setevent @TraceID, 10, 3, @on;
EXEC sp_trace_setevent @TraceID, 10, 35, @on;
EXEC sp_trace_setevent @TraceID, 10, 6, @on;
EXEC sp_trace_setevent @TraceID, 10, 8, @on;
EXEC sp_trace_setevent @TraceID, 10, 9, @on;
--SQL:BatchCompleted
EXEC sp_trace_setevent @TraceID, 12, 1, @on;
EXEC sp_trace_setevent @TraceID, 12, 10, @on;
EXEC sp_trace_setevent @TraceID, 12, 11, @on;
EXEC sp_trace_setevent @TraceID, 12, 12, @on;
EXEC sp_trace_setevent @TraceID, 12, 13, @on;
EXEC sp_trace_setevent @TraceID, 12, 14, @on;
EXEC sp_trace_setevent @TraceID, 12, 15, @on;
EXEC sp_trace_setevent @TraceID, 12, 16, @on;
EXEC sp_trace_setevent @TraceID, 12, 17, @on;
EXEC sp_trace_setevent @TraceID, 12, 18, @on;
EXEC sp_trace_setevent @TraceID, 12, 3, @on;
EXEC sp_trace_setevent @TraceID, 12, 35, @on;
EXEC sp_trace_setevent @TraceID, 12, 6, @on;
EXEC sp_trace_setevent @TraceID, 12, 8, @on;
EXEC sp_trace_setevent @TraceID, 12, 9, @on;
```
清单 15-18 向 SQL Trace 会话添加事件和列

下一步是从 SQL Trace 会话中筛选掉不需要的事件。没有必要在每次 SQL Trace 会话中，为所有数据库和所有应用程序一直收集所有语句。事实上，在清单 15-19 中，来自系统数据库（数据库 ID 小于 5）的事件将从会话中移除。用于筛选 SQL Trace 会话的存储过程是 `sp_trace_setfilter`。该存储过程接受来自 `sys.trace_columns` 的列 ID。可以对事件中未包含的列进行筛选，并且筛选器适用于所有事件。

```sql
USE master;
GO
DECLARE @intfilter INT = 5,
@FileName NVARCHAR(256) = N'c:\temp\IndexingMethod',
@TraceID INT;
SET @TraceID = (
SELECT id FROM sys.traces WHERE path LIKE @FileName + '%'
);
--Remove system databases from output
EXEC sp_trace_setfilter @TraceID, 3, 0, 4, @intfilter;
```
清单 15-19 向 SQL Trace 会话添加筛选器

设置 SQL Trace 监控的最后一步是启动跟踪。此任务使用存储过程 `sp_trace_setstatus` 完成，如清单 15-20 所示。通过此过程，可以启动、暂停和停止 SQL Trace 会话。一旦跟踪启动，它将在提供的文件位置开始创建 `.trc` 文件，SQL Trace 监控的配置即告完成。当 SQL Trace 会话的收集期结束时，将使用状态码 2（而不是 1）的此脚本来终止会话。清单 15-21 提供了此脚本。

```sql
USE master;
GO
DECLARE @FileName NVARCHAR(256) = N'c:\temp\IndexingMethod',
@TraceID INT;
SET @TraceID = (
SELECT id FROM sys.traces WHERE path LIKE @FileName + '%'
);
-- Set the trace status to start
EXEC sp_trace_setstatus @TraceID, 1;
```
清单 15-20 启动 SQL Trace 会话

> 注意
> 一些 SQL Server 专家更喜欢不使用数据库引擎优化顾问，因为可能收到无效的建议，他们更倾向于手动分析数据库并确定所需的索引。这种偏见错过了发现容易解决的问题（low-hanging fruit）或通过更改聚集索引位置来提高性能的机会。

```sql
USE master;
GO
DECLARE @FileName NVARCHAR(256) = N'c:\temp\IndexingMethod',
@TraceID INT;
SET @TraceID = (
SELECT id FROM sys.traces WHERE path LIKE @FileName + '%'
);
-- Set the trace status to stop
EXEC sp_trace_setstatus @TraceID, 0;
```
清单 15-21 停止 SQL Trace 会话

本节中的 SQL Trace 会话示例相当基础。在任何给定的生产环境中，可能需要设计更智能的进程，该进程在每个跟踪文件中收集指定时间段的信息，而不是使用文件大小来控制文件滚动速率。对 SQL Trace 收集信息以进行监控索引的这些类型的更改，应该不会影响在本章后面使用 SQL Trace 信息用于预期目的的能力。使用 SQL Trace 时，还有最后一项需要考虑。跟踪信息不需要像性能计数器和 DMO 信息那样持续收集。相反，SQL Trace 信息通常更适合收集 4-8 小时的时间段，该时间段代表数据库平台上的常规活动日。如果存在已知的性能不佳时间段，则可以专门针对该特定时间段。

使用 SQL Trace，有可能收集过多信息，这会压倒分析阶段并延迟索引建议。因此，筛选和针对关键时间和事件可以使发现有用信息的任务变得显著更容易完成。



#### 扩展事件

扩展事件是 SQL Server 中首选的跟踪工具。它包含的功能远多于 SQL Trace，但缺乏后者的直观简洁性。如果有机会在两种解决方案之间选择，使用扩展事件创建跟踪是更优的选择。创建扩展事件会话有两种方式。第一种是通过 T-SQL，本章将演示这种方法。第二种是使用 SQL Server Management Studio 中的图形用户界面，其中包含用于构建新会话的向导。创建会话的最佳实践在很大程度上与 SQL Trace 相同。例如，请务必在非数据和日志文件存储驱动器上收集会话日志。

将在扩展事件中创建的跟踪将收集与 SQL Trace 相同的一般信息。主要区别在于会话的创建方式以及某些事件和列的名称。在扩展事件中捕获的事件是 `rpc_completed` 和 `sql_batch_completed`，分别替代了 RPC:Completed 和 SQL:BatchCompleted。每个事件都捕获自己的一组列或数据元素，如表 15-3 所列。

**表 15-3 扩展事件列**

| 事件 | 列 |
| --- | --- |
| `rpc_completed` | • `connection_reset_option`• `cpu_time`• `data_stream`• `duration`• `logical_reads`• `object_name`• `output_parameters`• `physical_reads`• `result`• `row_count`• `statement`• `writes` |
| `sql_batch_completed` | • `batch_text`• `cpu_time`• `duration`• `logical_reads`• `physical_reads`• `result`• `row_count`• `writes` |

扩展事件会话中将包含一些额外的数据，这些数据作为全局字段或操作提供，可用于扩展每个事件中包含的默认信息。这些类似于上一个会话中 SQL Trace 会话包含的元素。要包含的全局字段如下：

*   `client_app_name`
*   `client_hostname`
*   `database_id`
*   `database_name`
*   `nt_username`
*   `process_id`
*   `session_id`
*   `sql_text`
*   `username`

定义好会话后，下一步是创建会话。扩展事件利用 T-SQL 数据定义语言（DDL）而不是存储过程来创建会话。清单 15-22 中的代码提供了会话的 DDL 并启动了会话。对于添加的每个事件，使用 `ADD EVENT` 语法，并使用 `ACTION` 子句来包含全局字段。为方便起见，该会话被设计为将输出存储在 SQL Server 的默认日志文件夹中，文件名为 `EventTracingforIndexTuning`。

```sql
USE master;
GO
IF EXISTS (
SELECT *
FROM sys.server_event_sessions
WHERE name = 'EventTracingforIndexTuning'
)
DROP EVENT SESSION [EventTracingforIndexTuning] ON SERVER;
CREATE EVENT SESSION [EventTracingforIndexTuning]
ON SERVER
ADD EVENT sqlserver.rpc_completed
(ACTION (
package0.process_id,
sqlserver.client_app_name,
sqlserver.client_hostname,
sqlserver.database_id,
sqlserver.database_name,
sqlserver.nt_username,
sqlserver.session_id,
sqlserver.sql_text,
sqlserver.username
)
),
ADD EVENT sqlserver.sql_batch_completed
(ACTION (
package0.process_id,
sqlserver.client_app_name,
sqlserver.client_hostname,
sqlserver.database_id,
sqlserver.database_name,
sqlserver.nt_username,
sqlserver.session_id,
sqlserver.sql_text,
sqlserver.username
)
)
ADD TARGET package0.event_file
(SET filename = N'EventTracingforIndexTuning')
WITH (
STARTUP_STATE = ON
);
GO
ALTER EVENT SESSION [EventTracingforIndexTuning] ON SERVER STATE = START;
GO
```
**清单 15-22 创建并启动扩展事件会话**

与 SQL Trace 会话类似，扩展事件会话可以启动和停止。不需要暂停它们，因为会话的元数据独立于会话是否正在运行而存在。清单 15-22 包含了启动跟踪的语法。清单 15-23 显示了停止跟踪的代码。此外，如果 SQL Server 重启，扩展事件跟踪会话将被保留，并且可以配置为自动重启，这与 SQL Trace 在重启后消失不同。

```sql
USE master;
GO
ALTER EVENT SESSION [EventTracingforIndexTuning] ON SERVER STATE = STOP;
GO
```
**清单 15-23 停止扩展事件会话**

这个扩展事件会话代表了一个相对简单的用例。扩展事件的好处在于它能够轻松捕获 SQL Server 实例的工作负载。使用来自跟踪的工作负载，可以定位、分析常见查询，并在需要时通过适当的索引更改进行优化。这允许利用这些信息改进环境中的性能，然后使用额外的跟踪来监控和测试这些改进。

### 查询存储

查询存储在 SQL Server 2016 中引入，是一个每个数据库的数据存储，包含执行计划信息和相关的执行统计信息。虽然它不一定直接提供索引调优信息，但此功能的进步允许实现自动索引功能。这基于 Azure SQL Database 中 SQL Server 现有的自动索引管理功能，详见 [`https://docs.microsoft.com/en-us/sql/relational-databases/automatic-tuning/automatic-tuning?view=sql-server-ver15#automatic-index-management`](https://docs.microsoft.com/en-us/sql/relational-databases/automatic-tuning/automatic-tuning%253Fview%253Dsql-%25C2%25A Dserver-%25C2%25A Dver15%2523automatic-index-management)。截至撰写本文时，自动索引管理在 SQL Server 2022 或更早版本中不可用。

查询存储提供了对数据库中执行的查询、其资源消耗、执行计划等的直接洞察。这提供了一个清晰的界面来理解计划缓存中的内容，无需进行复杂的系统视图查询，这在较大的 SQL Server 上可能具有挑战性。

使用查询存储的另一个好处是，在执行计划因性能改进而被其他执行计划替换的情况下，有时这可能与索引相关。这可能是由于索引的统计信息不佳，或者缺乏最佳索引以在不替换计划的情况下提供所需的性能。对于这两种情况，通过监控查询存储活动来识别这些性能问题，可以洞察环境中所需的索引。

出于监控索引的目的，这些原因提供了利用查询存储的另一个理由。要在数据库上启用查询存储，请执行清单 15-24 中的代码。请注意，从 SQL Server 2022 开始，默认启用查询存储。

虽然深入探讨查询存储本身超出了本书的范围，但在数据收集频率、存储数据量、数据丢弃率以及查询存储当前是否可写等方面具有很大的灵活性。建议进一步阅读查询存储相关资料；一个很好的起点是 Apress 出版的《Query Store for SQL Server 2019》。

```sql
USE [master]
GO
ALTER DATABASE [AdventureWorks2017] SET QUERY_STORE = ON
GO
ALTER DATABASE [AdventureWorks2017] SET QUERY_STORE (OPERATION_MODE = READ_WRITE)
GO
```
**清单 15-24 在 AdventureWorks2017 上启用查询存储**


