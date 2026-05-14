# 第六章 性能特性

另一个值得考虑使用的选项是 `查询优化器热修复`。这等同于使用跟踪标志 4199 来启用查询优化器热修复，我在 `第五章` 的“跟踪标志”一节中讨论过这一点。不同之处在于，您可以仅为在设置了此选项为开启的数据库中运行的查询启用优化器热修复。

可以使用 `ALTER DATABASE SCOPED CONFIGURATION` 设置的完整选项集可在我们的文档中找到：[`docs.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql`](https://docs.microsoft.com/sql/t-sql/statements/alter-database-scoped-configuration-transact-sql)。

##### Linux 内核配置

我已经向您展示了 SQL Server 实例和数据库配置选项。这些选项适用于 Windows 和 Linux 上的 SQL Server。对于 Linux 内核，是否有任何我建议您为 SQL Server 使用的特殊调优？答案是肯定的，但它并不像您想象的那么复杂。

SQL Server 是一个会最大化利用 CPU、内存和 I/O 资源的应用程序。因此，在我们构建 SQL Server 并与客户在社区技术预览版 (CTP) 构建期间合作时，我们发现有几个 Linux 内核选项是有帮助的。这些选项的完整列表包含在我们文档的此部分中：《SQL Server 2017 on Linux 的性能最佳实践和配置指南》，[`docs.microsoft.com/sql/linux/sql-server-linux-performance-best-practices`](https://docs.microsoft.com/sql/linux/sql-server-linux-performance-best-practices)。

您可以查看这些配置选项并根据您的工作负载决定进行更改。我们发现，在大多数应用程序工作负载下，这些选项有助于 SQL Server 在 Linux 上发挥最佳性能。

在查阅这些选项时，请参考您的 Linux 发行版的文档以及进行更改的正确方法。在某些 Linux 发行版上，这些选项可能已经设置为推荐值。例如，我发现，在 Red Hat Enterprise Linux 上，默认通过 `tuned-adm` 功能启用了 `throughput-performance` 配置文件，其中包含我们推荐的许多设置（注意：在您的 RHEL 系统上运行 `man tuned-adm` 以了解有关 RHEL 配置文件的更多信息）。

关于 Linux 和机器配置的另外几个要点：

• 密切关注与能源和功耗相关的 BIOS 设置。为获得最佳性能，请确保您的 BIOS 设置使用最大可能的功耗。

• 对于在虚拟机中运行的 SQL Server on Linux，请咨询您的虚拟机提供商，了解虚拟 CPU、NUMA 和其他与机器相关设置的正确配置。SQL Server on Linux 唯一不能使用的设置是任何支持 `动态内存` 的虚拟机配置。

关于操作系统配置的最后一点说明：在 Windows 上，有两个影响 SQL Server 性能的配置设置：`内存中锁定页`（您可以阅读




更多详情，请参阅[启用内存锁定页选项](https://docs.microsoft.com/sql/database-engine/configure-windows/enable-the-lock-pages-in-memory-option-windows)和**即时文件初始化**（你可以在[数据库即时文件初始化](https://docs.microsoft.com/sql/relational-databases/databases/database-instant-file-initialization)中阅读更多内容）。对于运行于 Linux 上的 SQL Server，这两种选项都不是必需的。Linux 中不存在 Windows 上那样的**内存锁定页**概念（即，Linux 中没有等效的 `AWE API`）。使用 `memorylimitmb` 选项可以避免 `sqlservr` 进程出现内存问题和分页。

此外，SQL Server 在创建文件时使用 Linux API 调用，其即时文件初始化行为是默认行为。注意：与在 Windows 上一样，SQL Server 仍然需要将事务日志文件清零，以便在恢复期间正确识别事务日志的结尾。更多信息，请参见 Anthony Nocentino 的博客文章 [Linux 上 SQL Server 的即时文件初始化](http://www.centinosystems.com/blog/sql/instant-file-initialization-in-sql-server-on-linux)。

## 调优以取得成功

你已经见识了 SQL Server 内置的性能能力。我已向你展示了一些重要的 SQL Server、数据库和 Linux 配置选项。掌握了这些知识后，我相信每个使用 SQL Server 的人都应该了解另外几个重要的主题，以调优你的 SQL Server 数据库、对象和应用程序。这包括数据库和事务日志文件的物理放置、创建正确的索引、保持统计信息准确且最新，以及开发者为确保其应用程序充分发挥 SQL Server 潜力而应掌握的技巧。

