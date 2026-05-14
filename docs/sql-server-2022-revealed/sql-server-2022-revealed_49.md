# 9. SQL Server 2022 在 Linux、容器和 Kubernetes 上的应用

2016 年，我们宣布将在 Linux 操作系统上发布 SQL Server，此举震惊了数据库行业（您可以在 [`https://blogs.microsoft.com/blog/2016/03/07/announcing-sql-server-on-linux`](https://blogs.microsoft.com/blog/2016/03/07/announcing-sql-server-on-linux) 阅读 Scott Guthrie 的原始博文）。这一公告促成了 SQL Server 2017 在 Linux 上的发布。我们使用名为平台抽象层 (`PAL`) 的软件，以一种创新方法构建了 Linux 版 SQL Server。这使我们能够快速将 SQL Server 引入 Linux 市场，而无需重写整个引擎。`PAL` 还提供了兼容性，使得核心引擎代码保持不变，允许您跨操作系统备份和还原数据库。这意味着查询处理器和列存储索引等核心引擎功能在 Windows 和 Linux 版 SQL Server 上完全相同。在 SQL Server 2019 中，我们完善了 SQL Server 2017 中不存在的大多数功能，包括复制、`CDC`、`DTC` 事务、机器学习和数据虚拟化。

我们确保使用 `PAL` 的方法能使 SQL Server 的性能与 Windows 版 SQL Server 一样快甚至更快（您可以在我们工程团队的博文 [`https://cloudblogs.microsoft.com/sqlserver/2016/12/16/sql-server-on-linux-how-introduction`](https://cloudblogs.microsoft.com/sqlserver/2016/12/16/sql-server-on-linux-how-introduction) 中阅读原始的 `PAL` 方法）。合作伙伴提交的 `TPC-H` 结果证明了这一点（您可以在 [`www.tpc.org/tpch/results/tpch_price_perf_results5.asp?resulttype=noncluster&version=3`](http://www.tpc.org/tpch/results/tpch_price_perf_results5.asp?resulttype=noncluster&version=3) 查看示例）。

通过支持 Linux 版 SQL Server，我们开启了容器和 Kubernetes 支持的新可能性。这使我们能够触及新的客户和应用程序，这在以前是不可能的。

SQL Server 2022 继续保持对 Linux、容器和 Kubernetes 的支持。本章并非深入探讨 Linux 版 SQL Server 的工作原理。实际上，我写过一整本关于该主题的书，名为 `Pro SQL Server on Linux`。在本章中，我将介绍 SQL Server 2022 的差异，同时也与您回顾一些关于如何使用和优化 Linux、容器及 Kubernetes 上 SQL Server 的基础知识。

## SQL Server 2022 在 Linux 上

SQL Server 2022 在 Linux 上与 SQL Server 2019 非常相似，只有一些细微差别。这包括我们支持的 Linux 发行版版本、增强的 `DTC` 事务支持以及一些我们不支持的功能。有关 SQL Server Linux 的所有最新信息，请持续关注 [`https://aka.ms/sqllinux`](https://aka.ms/sqllinux)。

### SQL Server 2022 的新特性

SQL Server 2022 *正式*支持以下 Linux 发行版：

*   Red Hat Enterprise Linux 8.0–8.5 Server
*   Ubuntu 20.04 LTS
*   SUSE Linux Enterprise Server (`SLES`) 15

我经常被问及对其他 Linux 发行版（如 `CentOS`）的支持。SQL Server on Linux 可以在其他 Linux 发行版上正常运行，但我们只官方支持 `RHEL`、`Ubuntu` 和 `SLES`。这些是我们测试过的发行版，并且我们与这些公司有协议来提供官方生产支持。

注意：我们不支持在 Windows Subsystem for Linux (`WSL`) 上运行 Linux 版 SQL Server。但是，我们支持在包括 `WSL` 集成的 Windows 上使用 Docker 运行 SQL Server 容器。

#### 仅限 Linux 的新功能

SQL Server 2022 for Linux 的一个新功能是管理 `DTC` 事务。我们在 SQL Server 2019 中为 Linux 添加了 `DTC` 支持。然而，对于监控和管理事务的支持有限，尤其是在出现问题时。在 SQL Server 2022 中，我们能够在 `PAL` 内添加 `WMI` 支持以实现这一新功能。您可以在 [`https://docs.microsoft.com/sql/linux/sql-server-linux-configure-msdtc`](https://docs.microsoft.com/sql/linux/sql-server-linux-configure-msdtc) 阅读更多信息。

此外，正如我在本书第 6 章所述，我们现在支持 `T-SQL` 快照备份，因此您可以在 Linux 上执行快照备份，而无需编写 `VDI` 程序或依赖 Windows `VSS` 等程序。

SQL Server 2022 的一些新增强功能我们暂不支持：

*   安装过程中的 Azure SQL Server 扩展（您可以使用其他方法安装该扩展）
*   分布式可用性组的多 `TCP` 连接
*   `TLS 1.3`
*   Intel `QAT` 备份压缩
*   Microsoft Purview 策略

注意：正如您在本书第 3 章所学，`Synapse Link` 需要自承载集成运行时 (`SHIR`)。`SHIR` 仅在 Windows `OS` 上工作，但您可以指示它连接到 Linux 上的 SQL Server 实例。

这是撰写本书时的列表。到我们发布 SQL Server 2022 时，我们可能已经添加了对这些功能的支持。请持续关注所有最新细节：[`https://docs.microsoft.com/sql/linux/sql-server-linux-editions-and-components-2022#unsupported-features-and-services`](https://docs.microsoft.com/sql/linux/sql-server-linux-editions-and-components-2022#unsupported-features-and-services)。

在早期介绍 SQL Server 2022 时，我收到一个问题，大意是：“鲍勃，看起来你们最近在 Linux 上的投入不多啊。”我的回答是：我们有在投入。我们可能没有投资于特定于 Linux 的新功能，但请看看引擎中所有丰富的功能，它们在 Windows 和 Linux 上都能工作，包括云连接功能、内置查询智能、SQL Server 的分类账、`tempdb`、包含的 `AGs`、基于 `REST` 的 `Polybase` 以及新的 `T-SQL` 增强功能。这就是我们为 Linux 版 SQL Server 构建的兼容性故事的力量。



