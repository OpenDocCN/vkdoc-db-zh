# 7. Azure SQL 的性能监控与调优

您现在已经了解了如何保护 Azure SQL 部署的安全。另一个确保为您的应用程序提供最佳数据库的方面是了解如何扩展、配置、监控和调优性能。如果您熟悉 SQL Server，这里有个好消息：驱动 Azure SQL 的引擎与 SQL Server 的是同一个！这意味着 Azure SQL 几乎拥有您所需的任何性能功能。这也意味着您用于 SQL Server 的许多相同任务和技能同样适用于 Azure SQL。在本章中，我们将探讨您通常用于监控和调优 SQL Server 性能的所有功能和任务，并与 Azure SQL 进行比较。您还将学习一些 Azure 特有的功能和技术，以帮助您获得最佳性能。

本章将包含一些示例，供您在阅读时尝试和使用。要尝试本章中使用的任何技术、命令或示例，您需要：

*   一个 `Azure 订阅`。
*   对 `Azure 订阅` 拥有至少 `参与者` 角色访问权限。您可以在 [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles) 阅读更多关于 Azure 内置角色的信息。
*   访问 `Azure 门户`。
*   一个 `Azure SQL 托管实例` 和/或 `Azure SQL 数据库` 的部署，如我在第 4 章中所做。我部署的 `Azure SQL 数据库` 使用了 `AdventureWorks` 示例数据库，这在部分示例中将会用到。
*   要连接到 `托管实例`，您需要一个 `Azure` 中的 `跳板机` 或虚拟机。我在本书第 4 章中展示了如何操作。一个简单的方法是创建一个新的 `Azure 虚拟机`，并将其部署到与 `托管实例` 相同的虚拟网络中（您将使用与 `托管实例` 不同的子网）。
*   要连接到 `Azure SQL 数据库`，我将使用我在第 3 章部署的名为 `bwsql2022` 的 `Azure 虚拟机`，该虚拟机在第 6 章中配置了专用终结点（只要您能连接到 `Azure SQL 数据库`，也可以使用其他方法）。
*   安装 `az` CLI（更多详情请参见 [`https://learn.microsoft.com/cli/azure/install-azure-cli`](https://learn.microsoft.com/cli/azure/install-azure-cli)）。您也可以使用 `Azure Cloud Shell` 代替，因为 `az` 已预装其中。您可以在 [`https://azure.microsoft.com/features/cloud-shell/`](https://azure.microsoft.com/features/cloud-shell/) 阅读更多关于 `Azure Cloud Shell` 的信息。
*   本章将运行一些 `T-SQL`，因此请安装一个工具，如 `SQL Server Management Studio` (`SSMS`)，下载地址为 [`https://aka.ms/ssms`](https://aka.ms/ssms)。我在第 3 章部署的 `bwsql2022` `Azure 虚拟机` 中安装了 `SSMS`。
*   本章提供了脚本文件，您可以在本书源文件的 `ch7_performance` 文件夹中找到这些脚本。本章练习中，我还将使用非常流行的工具 `ostress.exe`，它包含在 `Replay Markup Language` (`RML`) Utilities 中。您可以从 [`https://aka.ms/ostress`](https://aka.ms/ostress) 下载 `RML`。请确保将 `RML` 的安装文件夹添加到系统路径中（默认为 `C:\Program Files\Microsoft Corporation\RMLUtils`）。

> **提示**
>
> 自本书第一版以来，我制作了一些新的性能故障排除视频和示例实验室，专门针对 `Azure SQL 数据库`。您可以在 [`https://aka.ms/performanceseries`](https://aka.ms/performanceseries) 找到视频，在 [`https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting`](https://github.com/microsoft/sqlworkshops-azuresqlworkshop/tree/master/azuresqlworkshop/07-PerformanceTroubleshooting) 找到示例实验室。我认为这些资源是本章示例（包含更多性能场景）的一个很好的补充。

## 性能功能

由于驱动 `Azure SQL` 的引擎与 `SQL Server` 相同，因此几乎所有性能功能都可供您使用。话虽如此，了解一些重要领域（包括相似点和不同点）至关重要，因为这些领域会影响您为 `Azure SQL` 部署确保最大性能的能力。这些领域包括最大容量、索引、`内存中 OLTP`、分区、`SQL Server 2022` 性能增强以及新的 `Azure SQL` 智能性能功能。


### 最大容量

当你选择安装 SQL Server 的平台时，通常会`预估`所需的资源规模。在许多情况下，你会规划 CPU、内存和磁盘空间等资源所需的最大容量。你可能还需要确保在 IOPS 和延迟方面具备正确的 I/O 性能能力。

在本书第 4 章和第 5 章中，我向你展示了选择、部署和配置 Azure SQL 托管实例与 Azure SQL 数据库部署的所有选项。为确保获得最佳性能，你需要牢记 Azure SQL 的这些容量限制：

*   Azure SQL 托管实例最多可支持 128 个 vCore、约 870GB 内存，以及最高 32TB 的存储空间，具体取决于你选择的服务层级和硬件配置。
*   Azure SQL 数据库最多可支持 128 个 vCore、830GB 内存，以及最高 4TB 的数据库大小，具体取决于你选择的服务层级和硬件配置。Azure SQL 数据库的超大规模部署选项可支持最高 100TB 的数据库和无限制的事务日志空间。
*   你对部署选项（如 vCore 数量）的决策会影响其他资源容量，无论是托管实例还是数据库部署。例如，通用业务型 Azure SQL 数据库的 vCore 数量会影响最大内存、最大数据库大小、最大事务日志大小以及最大日志速率等。

让我们在此暂停，以帮助你理清思路。如何查看图表或表格来弄清所有这些选择的限制呢？

对于托管实例，请从这个显示硬件配置差异的文档页面开始：[`https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits?view=azuresql#hardware-configuration-characteristics`](https://learn.microsoft.com/azure/azure-sql/managed-instance/resource-limits?view=azuresql#hardware-configuration-characteristics)。然后继续向下滚动此页面，查看按服务层级组织的其他限制，如 vCore、内存、存储等。

Azure SQL 数据库呢？你可以查看基于 vCore 的容量和限制表格：[`https://learn.microsoft.com/azure/azure-sql/database/resource-limits-vcore-single-databases`](https://learn.microsoft.com/azure/azure-sql/database/resource-limits-vcore-single-databases)。如果你选择无服务器计算层，会发现存在一些差异。请收藏这些文档链接。我一直在使用它们。

请记住，某些限制（如内存）是由 Windows 作业对象强制执行的。我在本书第 4 章中提到过此实现。使用动态管理视图 `sys.dm_os_job_object` 可以查看部署的真实内存和其他资源限制。

提示：部署后，你可以通过 `sys.dm_user_db_resource_governance` 等动态管理视图查看你的资源限制。

如果你做出了错误的选择，需要更多容量怎么办？好消息是，你可以对 Azure SQL 托管实例和数据库进行更改以获得更多（或更少）资源，而无需进行任何数据库迁移。本章后面你将看到一个这样的示例。只需记住，托管实例的更改可能需要相当长的时间。

此外，对于 Azure SQL 数据库，你可以在通用业务型和业务关键型服务层级之间来回切换（注意，数据库播种可能需要一些时间）。如果你从通用业务型迁移到超大规模，可以再迁移回来一次。但是，如果你从超大规模开始，则无法迁移回通用业务型（并且永远无法迁移到业务关键型）。

