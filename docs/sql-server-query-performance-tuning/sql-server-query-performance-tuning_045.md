# 网络段

## 与基线对比

### 每秒总字节数

你可以使用 `Bytes Total/Sec` 计数器来判断网络接口卡（`NIC`）或网络适配器的性能。`Bytes Total/sec` 计数器应报告高值，表明大量传输成功。将此值与 `Network Interface\Current Bandwidth` 性能计数器报告的值进行比较，后者反映了每个适配器的带宽。

为流量高峰预留空间，通常平均使用率不应超过容量的 50%。如果这个数字接近连接容量，并且处理器和内存使用率适中，那么连接本身可能就是问题所在。

### 网络利用率百分比

`% Net Utilization` 计数器表示网络段上正在使用的网络带宽百分比。此计数器的阈值取决于网络类型。例如，对于以太网，当 SQL Server 位于共享网络集线器上时，推荐的阈值是 30%。对于位于专用全双工网络上的 SQL Server，即使网络使用率接近 100% 是可接受的，但将网络利用率保持在可接受阈值以下以负载高峰预留空间是有利的。

> **注意**：你必须安装 `Network Monitor Driver` 才能使用 `network segment` 对象计数器收集性能数据。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 4 章 ■ CPU 性能分析

在 Windows Server 2012 R2 中，你可以从网络适配器的局域网连接属性中安装 `Network Monitor Driver`。`Network Monitor Driver` 位于网络适配器的网络组件协议列表中。

你也可以查看 `sys.dm_os_wait_stats` 中与网络相关的等待统计信息。但是，一个经常出现的等待是 `ASYNC_NETWORK_IO`。虽然这可能表示网络相关的等待，但更常见的情况是反映了因编程代码效率低下、未能有效使用结果集而导致的等待。

## 网络瓶颈解决方案

一些常见的网络瓶颈解决方案如下：

-   优化应用程序工作负载
-   添加网络适配器
-   调节和避免中断

让我们更详细地考虑这些解决方案。

### 优化应用程序工作负载

要优化数据库应用程序和数据库服务器之间的网络流量，请在应用程序中进行以下设计更改：

-   与其发送长的 SQL 字符串，不如为 SQL 查询创建存储过程。然后，你只需要通过网络发送存储过程的名称及其参数。
-   将多个数据库请求分组到一个存储过程中。这样，对于存储过程中实现的一组 SQL 查询，只需要一个跨网络的数据库请求。
-   请求小的数据集。不要请求应用程序逻辑中未使用的表列。
-   将数据密集型的业务逻辑移动到数据库中，作为存储过程或数据库触发器，以减少网络往返次数。
-   如果数据不经常更改，尝试将信息缓存在应用程序中，而不是频繁调用数据库获取与上次调用完全相同的信息。
-   最小化网络调用，例如返回未使用的多个结果集。一个常见问题是由 SQL Server 返回的结果集包含每个语句的行数导致的。你可以在查询开头使用 `SET NOCOUNT ON` 来禁用此功能。

## SQL Server 整体性能

要分析 SQL Server 实例的整体性能，除了检查硬件资源利用率外，还应该检查 SQL Server 本身的一些常规方面。你可以使用表 4-3 中介绍的性能计数器。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 4 章 ■ CPU 性能分析

**表 4-3. 用于分析通用 SQL 压力的性能监视器计数器**

| 对象(实例[,实例 N]) | 计数器 |
| :--- | :--- |
| SQLServer:Access Methods | `FreeSpace Scans/sec` |
| | `Full Scans/sec` |
| | `Table Lock Escalations/sec` |
| | `Worktables Created/sec` |
| SQLServer:Latches | `Total Latch Wait Time (ms)` |
| SQLServer:Locks(_Total) | `Lock Timeouts/sec` |
| | `Lock Wait Time (ms)` |
| | `Number of Deadlocks/sec` |
| SQLServer:SQL Statistics | `Batch Requests/sec` |
| | `SQL Re-Compilations/sec` |
| SQLServer:General Statistics | `Processes Blocked` |
| | `User Connections` |
| | `Temp Tables Creation Rate` |
| | `Temp Tables for Destruction` |

让我们将这些分解到不同的关注领域，以便在更有用的上下文中展示这些计数器。

### 缺失索引

要分析缺失索引导致表扫描或检索大数据集的可能性，你可以使用表 4-4 中的计数器。

**表 4-4. 用于分析过度数据扫描的性能监视器计数器**

| 对象(实例[,实例 N]) | 计数器 |
| :--- | :--- |
| SQLServer:Access Methods | `Full Scans/sec` |

### 每秒完全扫描次数

此计数器监控对基表或索引执行的无限制完全扫描的次数。扫描不一定是坏事。但它们确实代表了更广泛的数据访问，因此很可能表明存在问题。导致 `Full Scans/sec` 值过高的几个主要原因如下：

-   缺失索引
-   请求的行数过多
-   谓词选择性不够
-   T-SQL 不当
-   数据分布或数量不支持查找

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 4 章 ■ CPU 性能分析

为了进一步调查产生这些问题的查询，请使用扩展事件来识别这些查询（我将在下一章介绍此工具）。存在缺失索引、请求过多行或 T-SQL 编写不佳的查询将产生大量逻辑读（由扫描整个表或整个索引引起）以及增加的 CPU 时间。

请注意，可能会对存储过程中使用的临时表执行完全扫描，因为大多数情况下你不会在临时表上建立索引（或者你不需要索引）。尽管如此，将此计数器添加到基线有助于识别临时表使用可能增加的情况，如果使用不当，可能会对性能不利。

### 动态管理对象

检查缺失索引的另一种方法是查询动态管理视图 `sys.dm_db_missing_index_details`。

此管理视图返回的信息可以基于正在针对数据库运行的查询的执行计划建议索引候选。视图 `sys.dm_db_missing_index_details` 是统称为 *缺失索引功能* 的一系列 DMV 的一部分。这些 DMV 基于存储在缓存中的执行计划生成的数据。你可以直接查询此视图以收集数据，从而决定是否要基于视图中提供的信息创建索引。缺失索引也会显示在给定查询的 XML 执行计划中，但我将在下一章更详细地介绍这一点。虽然这些视图对于建议可能的索引很有用，但由于它们无法链接到特定查询，因此不清楚这些索引中哪一个最有用。你最好使用我在下一章展示的技术将缺失索引与特定查询关联起来。对于所有缺失索引建议，在系统上实施任何建议之前，你必须先进行测试。

与缺失索引相反的问题是从不使用的索引。DMV `sys.dm_db_index_usage_stats` 显示了哪些索引已被使用，至少是在 SQL Server 实例上次重启以来。不幸的是，此 DMV 中的计数器有多种方式会被重置或移除，因此你不能完全依赖它来获得 100% 准确的索引使用视图。你也可以使用较低级别的 DMV `sys.dm_db_index_operational_stats` 查看正在使用的索引。



