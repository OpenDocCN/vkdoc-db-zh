# 索引

任何与 SQL Server 打交道的人都知道，如果没有合适的索引，很难获得所需的查询性能。

你在 SQL Server 中可以使用的每种索引选项，在 Azure SQL 中都可用，包括聚集索引、非聚集索引、联机索引和可恢复索引。你可以在 [`https://learn.microsoft.com/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described`](https://learn.microsoft.com/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described) 阅读索引入门，在 [`https://learn.microsoft.com/sql/relational-databases/indexes/perform-index-operations-online`](https://learn.microsoft.com/sql/relational-databases/indexes/perform-index-operations-online) 了解联机索引的详细信息，在 [`https://learn.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations?view=sql-server-ver16#resumable-index-considerations`](https://learn.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations?view=sql-server-ver16#resumable-index-considerations) 了解可恢复联机索引的详细信息。可恢复联机索引非常引人注目，因为你可以暂停和恢复大型索引构建，甚至可以在失败点继续索引构建或重建。

`列存储索引`简直令人惊叹。我不断看到客户未能充分利用这一功能。对于合适的工作负载，列存储索引可以将读取查询性能提升 100 倍。列存储索引在你选择的 Azure SQL 任何部署选项中都受支持。关于列存储的一个误解是它是一种内存技术。事实是，当列存储索引加载到内存中并使用压缩时，其性能最佳，这样更多数据可以容纳在你的内存限制内。然而，列存储索引并不需要全部加载到内存中。要开始了解列存储索引，请参阅文档：[`https://learn.microsoft.com/sql/relational-databases/indexes/columnstore-indexes-overview`](https://learn.microsoft.com/sql/relational-databases/indexes/columnstore-indexes-overview)。

### 内存中 OLTP

在 SQL Server 2014 中（在 SQL Server 2016 中得到了极大增强），我们引入了一项用于高速事务的革命性功能，称为内存中 OLTP（代号 Hekaton）。如果你选择业务关键型服务层级，内存中 OLTP 可用于 Azure SQL 托管实例和数据库。

内存优化表是使用内存中 OLTP 的机制。内存优化表是真正的内存驻留表，因为它们必须完全加载到内存中。可用于存储内存优化表的内存是业务关键型服务层级内存限制的一个子集。部署的 vCore 数量决定了可用于内存优化表的内存百分比。超大规模支持一部分内存中 OLTP 对象，包括内存优化表类型、表变量和本机编译模块，但不支持内存优化表。

注意：内存优化表需要一个内存优化文件组。Azure SQL 会为任何数据库创建此文件组，即使它不是业务关键型（BC）服务层级。这样，如果你迁移到 BC，文件组已经为内存优化表设置好了。

刚接触内存中 OLTP？请从我们的文档开始：[`https://learn.microsoft.com/sql/relational-databases/in-memory-oltp/overview-and-usage-scenarios`](https://learn.microsoft.com/sql/relational-databases/in-memory-oltp/overview-and-usage-scenarios)。

## 分区

分区常与 SQL Server 配合用于包含大量行的表，通过按表中的列细分数据来提升性能。关于分区和 Azure SQL，请考虑以下几点：

*   Azure SQL Database 和托管实例均支持分区。
*   对于 Azure SQL 托管实例，您只能将文件组与分区结合使用。（请记住，Azure SQL Database 仅具有主分区和文件组，而托管实例支持用户定义的文件组）。

需要分区入门知识吗？请从此文档页面开始：[`https://learn.microsoft.com/sql/relational-databases/partitions/partitioned-tables-and-indexes`](https://learn.microsoft.com/sql/relational-databases/partitions/partitioned-tables-and-indexes)。

注意
作为开发者，您可能想了解 Azure SQL Database 中一些与 SQL 分区无关的有趣 `分区化` 技术。更多信息请阅读：[`https://learn.microsoft.com/azure/architecture/best-practices/data-partitioning-strategies#partitioning-azure-sql-database`](https://learn.microsoft.com/azure/architecture/best-practices/data-partitioning-strategies#partitioning-azure-sql-database)。

## SQL Server 2022 增强功能

SQL Server 2022 包含多项与性能相关的增强功能，这些功能在 Azure SQL 托管实例和 Azure SQL Database 中同样可用：

*   `智能查询处理` (IQP) 改进

    IQP ([`https://aka.ms/iqp`](https://aka.ms/iqp)) 是一系列内置于查询处理器中的功能，旨在让您的 T-SQL 查询运行更快或在无需更改代码的情况下保持性能。

    Azure SQL 托管实例和数据库包含来自 SQL Server 2017 和 2019 的所有 IQP 功能。此外，SQL Server 2022 的 IQP 功能也已在 Azure SQL 中可用。更多详情请参阅本章最后一节“智能性能”。

*   `查询存储提示`

    计划指南一直是强制或塑造独立于应用程序所发送查询的查询计划的有效方法。但此功能存在限制，且使用起来可能较为复杂。

    查询存储提示允许您向应用程序发出的任何传入查询“添加”查询提示，以塑造或更改查询计划。这些提示持久存储在查询存储中，使您能够基于一系列提示强制执行某个查询计划。一些新的 IQP 功能在后台使用了查询存储提示。了解更多：[`https://aka.ms/querystorehints`](https://aka.ms/querystorehints)。

注意
重要的是要知道，一些“隐藏瑰宝”功能，如循环扫描和缓冲池加速，都在后台用于 Azure SQL 的所有版本。更多信息请访问：[`https://learn.microsoft.com/sql/relational-databases/reading-page`](https://learn.microsoft.com/sql/relational-databases/reading-page)。

### 智能性能

在过去几个 SQL Server 版本中，我们一直致力于提供内置功能，以增强性能而无需您更改应用程序。我们的目标是利用数据和自动化做出明智决策，使您的查询运行更快。我们将此称为 `智能性能`。这些功能存在于 Azure SQL 中，但我们在云端走得更远。我们利用云的力量提供更多功能。您可以在本章最后一节（IQP 是其中的一部分）了解更多关于 Azure SQL 智能性能的细节。

## 性能配置与维护

在本书第 5 章中，我介绍了配置 Azure SQL 托管实例和数据库的许多选项。有一些可能影响性能的配置选项值得深入探讨。这包括 `tempdb` 数据库、数据库选项配置、文件和文件组、`最大并行度` 以及 `资源调控器`。此外，值得比较一下与 SQL Server 相比，为 Azure SQL 数据库维护索引和统计信息所需执行的各项任务。

### Tempdb

`tempdb` 数据库是应用程序使用的重要共享资源。确保 `tempdb` 配置正确会影响您提供一致性能的能力。Azure SQL 中 `tempdb` 的使用方式与 SQL Server 相同，但您配置 `tempdb` 的能力有所不同，包括文件放置位置、文件数量和大小、`tempdb` 大小以及 `tempdb` 配置选项。

在 Azure SQL Database 中，`tempdb` 文件始终自动存储在本地 SSD 驱动器上，因此 I/O 性能不应成为问题。

SQL Server 专业人员通常使用多个数据库文件来为 `tempdb` 表分区分配。对于 Azure SQL Database，文件数量随 vCore 数量扩展（例如，2 vCore = 4 个文件），最多为 16 个。文件数量无法通过针对 `tempdb` 的 T-SQL 进行配置，而需通过 `更改部署选项` 来设置。`tempdb` 数据库的最大大小也随 vCore 数量扩展。

我在第 5 章中提到，在 Azure SQL 托管实例中，您可以对 `tempdb` 进行一些配置选项，包括扩展文件数量、文件的逻辑名称以及自动增长增量大小值。

`tempdb` 数据库选项 `MIXED_PAGE_ALLOCATION` 设置为 `OFF`，`AUTOGROW_ALL_FILES` 设置为 `ON`。这无法配置，但与 SQL Server 一样，它们是推荐的默认设置。

### 数据库配置

正如我在第 5 章所述，通过 `ALTER DATABASE` 和 `ALTER DATABASE SCOPED CONFIGURATION`，Azure SQL 几乎提供了与 SQL Server 相同的所有数据库配置选项。请查阅文档：[`https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-transact-sql) 和 [`https://learn.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql`](https://learn.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql)。您将在本章后面看到 `ALTER DATABASE` 中针对 Azure SQL 的新选项。

对于性能而言，有一个无法更改的数据库选项是数据库的 `恢复模式`。默认为完整恢复，且无法修改。我经常看到社区比较 SQL Server 和 Azure SQL 的 OLTP 性能，但没有使用完整恢复，这可能会对批量插入等场景的性能产生影响。这确保您的数据库能够满足 Azure 服务级别协议 (SLA)。因此，不支持批量操作的最小日志记录。`tempdb` 支持批量操作的最小日志记录。

### 文件与文件组

SQL Server 专业人员通常使用文件和文件组，通过物理文件放置来提升 I/O 性能。Azure SQL 不允许用户将文件放置在特定的磁盘系统上。然而，Azure SQL 在 I/O 速率、IOPS 和延迟方面有性能保障，因此将用户与物理文件放置相隔离也可能是一种优势。

Azure SQL 数据库通常只有一个数据库文件（Hyperscale 可能有多个），其大小通过 Azure 界面进行配置。没有创建额外文件的功能，但同样地，考虑到 IOPS 和 I/O 延迟的保障，你无需为此担心。

**注意**
> Hyperscale 拥有独特的架构，初始部署时可能会根据你的 vCore 选择创建一个或多个文件。例如，对于一个 8 vCore 的部署，我见过 Hyperscale 创建了总计 40GB 的多个文件。此实现方式可能会改变，你不应依赖于此。Hyperscale 只是简单地根据需要创建文件和分配大小以满足你的要求。

Azure SQL 托管实例支持添加数据库文件和配置大小，但不支持文件的物理放置。托管实例的文件数量和文件大小可以用来提升 I/O 性能。然而，正如我在本书第 4 章中提到的，下一代通用服务层使用了不同的 I/O 架构，让你无需添加文件或更改大小即可控制 I/O 性能。此外，为了管理性目的（例如与分区结合使用，或使用像 `DBCC CHECKFILEGROUP` 这样的命令），Azure SQL 托管实例支持用户定义的文件组。

### 最大并行度

最大并行度（`MAXDOP`）会影响单个查询的性能，其在 Azure SQL 引擎中的工作方式与 SQL Server 相同。在 Azure SQL 中配置 `MAXDOP` 对于提供一致的性能可能非常重要。你可以像在 SQL Server 中一样，使用以下技术在 Azure SQL 中配置 `MAXDOP`：

*   支持在 Azure SQL 中使用 `ALTER DATABASE SCOPED CONFIGURATION` 来配置 `MAXDOP`。
*   支持在托管实例中使用 `sp_configure` 配置 “max degree of parallelism”。
*   完全支持 `MAXDOP` 查询提示，包括查询存储提示。
*   支持在托管实例中使用资源调控器配置 `MAXDOP`。

阅读更多关于 `MAXDOP` 的信息，请访问 [`https://learn.microsoft.com/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option`](https://learn.microsoft.com/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option)。

你将在本章名为“智能性能”的部分了解到更多关于智能查询处理（IQP）的一个名为 `DOP feedback` 的功能，它可以动态地辅助 SQL 引擎调整查询的 `DOP`。

### 资源调控器

资源调控器是 SQL Server 中的一项功能，可通过 I/O、CPU 和内存来控制工作负载的资源使用。虽然资源调控器在 Azure SQL 数据库中是在后台使用的，但对于 Azure SQL 托管实例，它支持用户自定义的工作负载组和池。如果你希望在 Azure SQL 托管实例中使用资源调控器，请查阅我们的文档：[`https://learn.microsoft.com/sql/relational-databases/resource-governor/resource-governor`](https://learn.microsoft.com/sql/relational-databases/resource-governor/resource-governor)。

### 维护索引

不幸的是，SQL 的索引不会自我维护，它们偶尔确实需要维护。公平地说，索引维护（具体指重建或重组）没有一个单一的答案。我见过很多客户在非必要时过于频繁地执行重建或重组。同样，也有很多时候这些操作可以提升性能。你可以考虑查看我们关于索引碎片的文档，作为索引维护有意义的一个原因：[`https://learn.microsoft.com/sql/relational-databases/indexes/reorganize-and-rebuild-indexes`](https://learn.microsoft.com/sql/relational-databases/indexes/reorganize-and-rebuild-indexes)。

**注意**
> 我并没有说出全部真相。对于 Azure SQL，这里有一个解决方案可以帮助决定是创建还是删除索引。但我不会提前透露太多。这个故事的结局在本章末尾。

SQL Server 的索引偶尔需要重组，有时需要重建。Azure SQL 支持 SQL Server 中用于重组和重建索引的所有选项，包括联机和可恢复索引操作。

联机和可恢复索引操作对于维护最大的应用程序可用性极为重要。阅读有关这些功能的全部信息，请访问 [`https://learn.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations`](https://learn.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations)。

### 维护统计信息

正确的统计信息是查询性能的生命线。SQL Server 提供了基于数据库修改自动保持统计信息最新的选项，Azure SQL 支持所有这些选项。我们的文档对统计信息如何用于查询性能有非常详细的解释：[`https://learn.microsoft.com/sql/relational-databases/statistics/statistics`](https://learn.microsoft.com/sql/relational-databases/statistics/statistics)。

自动统计信息更新的一个有趣方面是我们专门为 Azure SQL 引入的一个数据库范围配置，旨在帮助提高应用程序可用性。你可以从我的同事 Dimitri Furman 的博客文章中阅读关于此功能的详细信息：[`https://techcommunity.microsoft.com/t5/azure-sql-database/improving-concurrency-of-asynchronous-statistics-update/ba-p/1441687`](https://techcommunity.microsoft.com/t5/azure-sql-database/improving-concurrency-of-asynchronous-statistics-update/ba-p/1441687)。

## 监控与故障排除性能

如果你想确保 SQL 应用程序获得最佳性能，你需要学习如何监控和故障排除性能场景。Azure SQL 随附了 SQL Server 的性能工具和功能来协助你完成此任务。这包括来自 Azure 生态系统的工具以及驱动 Azure SQL 的 SQL Server 引擎内置的功能。

在本章的这一部分，你将不仅学习到监控功能，还将学习如何将它们应用于 Azure SQL 的性能场景，包括示例。

### 监控工具与功能

Azure 提供了与 SQL 集成但位于 Azure 生态系统之外的工具，称为 Azure Monitor。

你习惯于使用动态管理视图（`DMV`）和扩展事件吗？Azure SQL 拥有你需要的功能。你需要调试查询计划吗？Azure SQL 拥有 SQL Server 的所有功能，包括轻量级查询分析和 `showplan` 详细信息。

查询存储已成为性能调优的基石，它在 Azure SQL 中默认是开启的。Azure 门户包含可视化效果，例如查询性能见解，无需像 SSMS 这样的工具即可查看查询存储数据。

此外，自从本书第一版以来，出现了两个突破性的新工具，可以显著节省你的时间和精力：Database Watcher 和 Azure SQL Database 中的 Microsoft Copilot 技能。

所有这些汇聚成一套强大的工具和功能，帮助你监控和故障排除 Azure SQL 的性能。


