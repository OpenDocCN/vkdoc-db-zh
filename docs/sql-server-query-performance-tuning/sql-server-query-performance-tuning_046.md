# SQL Server 性能分析指标

这将有助于显示哪些索引因争用或 I/O 而变慢。我将在第 20 章更详细地介绍这两方面。你可能还会发现，数据库优化顾问（在第 10 章介绍）的建议或许能帮助你针对特定查询优化特定索引。

### 数据库并发性

要分析数据库阻塞对 `SQL Server` 性能的影响，可以使用表 4-5 中所示的计数器。

**表 4-5.** 用于分析 `SQL Server` 锁定的 `性能监视器` 计数器

**对象(实例[,实例 N])**

**计数器**

`SQLServer:Latches`

`Total Latch Wait Time (ms)`

`SQLServer:Locks(_Total)`

`Lock Timeouts/sec`

`Lock Wait Time (ms)`

`Number of Deadlocks/sec`

`Total Latch Wait Time (Ms)`

`闩锁` 由 `SQL Server` 内部使用，以保护内部结构（如表行）的完整性，用户无法直接控制。此计数器监视过去一秒钟内必须等待的闩锁请求的总等待时间（以毫秒为单位）。此计数器的值过高可能表明 `SQL Server` 在等待其内部同步机制上花费了过多时间。

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 4 章 ■ `CPU` 性能分析

### 锁定超时/秒 和 锁定等待时间 (毫秒)

你应该期望 `Lock Timeouts/sec` 为 `0`，且 `Lock Wait Time (ms)` 非常低。`Lock Timeouts/sec` 出现非零值且 `Lock Wait Time (ms)` 值过高，表明数据库中发生了过度的阻塞。

在这种情况下，可以采取两种方法。

-   你可以使用 `SQL Profiler` 的数据或查询 `sys.dm_exec_query_stats` 来识别当前缓存中的高开销查询，然后相应地优化这些查询。
-   你可以使用阻塞分析来诊断过度阻塞的原因。通常，首先专注于优化高开销查询是有利的，因为这反过来会减少其他查询的阻塞。在第 20 章，你将学习如何分析和解决阻塞问题。
-   `扩展事件` 提供了一个名为 `blocked_process_report` 的阻塞事件，你可以启用它并设置阈值以捕获阻塞信息。`扩展事件` 将在第 6 章介绍，而 `blocked_process_report` 将在第 20 章讨论。

请记住，某种程度的锁是系统的必要组成部分。你需要建立一个基线，以便彻底跟踪某个给定值是否值得关注。

### 死锁数/秒

你应该期望看到此计数器的值为 `0`。如果发现非零值，则应识别受害的请求，并自动重新提交数据库请求或建议用户重新提交。更重要的是，应尝试排查和解决死锁问题。第 21 章将介绍如何操作。

### 不可重用的执行计划

由于为存储过程查询生成执行计划需要 `CPU` 周期，你可以通过重用执行计划来减轻 `CPU` 的压力。要分析正在重新编译的存储过程的数量，可以查看表 4-6 中的计数器。

**表 4-6.** 用于分析执行计划可重用性的 `性能监视器` 计数器

**对象(实例[,实例 N])**

**计数器**

`SQLServer:SQL Statistics`

`SQL Re-Compilations/sec`

存储过程的重新编译会给处理器增加开销。你希望 `SQL Re-Compilations/sec` 计数器的值尽可能接近 `0`，但你永远不会看到这种情况。如果你持续看到偏离基线度量值或剧烈波动的数值，那么你应该使用 `扩展事件` 来进一步调查正在重新编译的存储过程。一旦识别出相关的存储过程，你应尝试分析并解决重新编译的原因。在第 17 章，你将学习如何分析和解决各种重新编译的原因。

### 一般行为

`SQL Server` 提供了额外的性能计数器来跟踪 `SQL Server` 系统的一些一般方面。表 4-7 列出了一些最常用的计数器。

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 4 章 ■ `CPU` 性能分析

**表 4-7.** 用于分析传入请求量的 `性能监视器` 计数器

**对象(实例[,实例 N])**

**计数器**

`SQLServer:General Statistics`

`User Connections`

`SQLServer:SQL Statistics`

`Batch Requests/sec`

### 用户连接数

多个只读的 `SQL Server` 可以在负载均衡环境中协同工作（其中 `SQL Server` 分布在多台机器上）以支持大量数据库请求。在这种情况下，最好监控 `User Connections` 计数器，以评估用户连接在多个 `SQL Server` 实例上的分布情况。在正常的应用程序行为下，`User Connections` 的值可能分布广泛。这就是基线对于确定预期行为至关重要的地方。你很快将看到如何建立这个基线。

### 批处理请求/秒

此计数器是 `SQL Server` 负载的一个良好指标。根据系统资源利用水平和 `Batch Requests/sec`，你可以估算 `SQL Server` 在不出现资源瓶颈的情况下可能能够承受的用户数量。此计数器在不同负载周期下的值，有助于你理解其与数据库连接数的关系。这也有助于你理解 `SQL Server` 与 `Web Request/sec` 的关系，即对于使用 Microsoft Internet Information Services (`IIS`) 和 Active Server Pages (`ASP`) 的 Web 应用程序而言，就是 `Active Server Pages.Requests/sec`。所有这些分析都有助于你更好地理解和预测随着用户负载变化时的系统行为。

在正常的应用程序行为下，此计数器的值可能分布广泛。一个正常的基线对于确定预期行为至关重要。

## 总结

在本章中，你学习了如何收集关于 `CPU`、网络和 `SQL Server` 整体的指标。所有这些信息都有助于你在深入尝试调优查询之前，理解系统上发生的情况。请记住，`CPU` 会受到其他资源的影响，因为它是必须管理这些资源的东西，所以某些看起来像是 `CPU` 问题的情况，实际上用磁盘或内存问题来解释更为合适。

对于 `SQL Server` 来说，网络很少是主要瓶颈。就像系统的其他部分一样，你有许多方法可以通过 `性能监视器` 计数器来观察 `SQL Server` 内部行为。关于各种系统指标的讨论到此结束。接下来，你将学习如何将这些信息整合起来创建一个基线。

[www.it-ebooks.info](http://www.it-ebooks.info/)

