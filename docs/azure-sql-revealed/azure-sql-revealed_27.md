# 8. Azure SQL 的可用性

至此，你已经经历了从部署、配置、安全到扩展、监控和性能调优的旅程。Azure SQL *核心* 的最后一个环节是**可用性**。多年来，我与 SQL Server 客户交流时，几乎没人不需要其数据库具备高可用性。我也几乎从未遇到过不关心灾难恢复能力的客户。因此，本章实际上讲述的是 Azure SQL 的**高可用性和灾难恢复 (HADR)**。我想告诉你的是，在我过去一年评估和使用 Azure SQL 之后，我认为其内置的 HADR 功能是该服务的一大亮点。事实上，我相信在你学完本章后，会确信仅凭其 HADR 功能，就值得将 Azure SQL 作为你的部署目标。

在本章中，我们将大部分时间深入探讨 HADR 功能的细节，包括备份与还原、内置可用性、通过 Azure 扩展 HADR，以及数据库可用性和一致性。最后，我将介绍如何监控你的部署的 HADR。

本章包含供你边阅读边尝试和使用的示例。要尝试本章中使用的任何技术、命令或示例，你需要具备：

*   一个 Azure 订阅。
*   至少对 Azure 订阅拥有“参与者”角色访问权限。你可以在 [`https://learn.microsoft.com/azure/role-based-access-control/built-in-roles`](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles) 阅读更多关于 Azure 内置角色的信息。
*   访问 Azure 门户。
*   我将使用本书前面部署的几个 Azure SQL 数据库和一个托管实例。我还部署了一个新的 Azure SQL 商业关键服务层数据库，其中一个示例使用了 AdventureWorksLT 示例。
*   要连接到托管实例，你需要一个 Azure 中的*跳板*或虚拟机进行连接。我在本书的第 4 章中展示了如何操作。一个简单的方法是创建一个新的 Azure 虚拟机，并将其部署到与托管实例相同的虚拟网络（你需要使用与托管实例不同的子网）。
*   要连接到 Azure SQL 数据库，我将使用我在第 3 章中部署的 Azure VM，名为 `bwsql2022`，并在第 6 章中为其配置了专用终结点（你也可以使用其他方法，只要能连接到 Azure SQL 数据库即可）。
*   安装 `az` CLI（详情请参见 [`https://learn.microsoft.com/cli/azure/install-azure-cli`](https://learn.microsoft.com/cli/azure/install-azure-cli)）。你也可以使用 Azure Cloud Shell，因为 `az` 已经预装。你可以在 [`https://azure.microsoft.com/features/cloud-shell/`](https://azure.microsoft.com/features/cloud-shell/) 阅读更多关于 Azure Cloud Shell 的信息。
*   安装 Azure PowerShell。使用以下文档了解如何为你的客户端安装 Azure PowerShell：[`https://learn.microsoft.com/powershell/azure/install-azure-powershell`](https://learn.microsoft.com/powershell/azure/install-azure-powershell)。我在我的 Azure VM 和任何可能运行本章示例的客户端上都安装了 Azure PowerShell。
*   本章将运行一些 T-SQL，因此请安装一个工具，如 SQL Server Management Studio (SSMS)，下载地址：[`https://aka.ms/ssms`](https://aka.ms/ssms)。我在 `bwsql2022` Azure 虚拟机中安装了 SSMS。
*   对于本章，我提供了一些脚本文件供部分示例使用。你可以在本书源文件的 `ch8_availability` 文件夹中找到这些脚本。我还将在本章练习中使用非常流行的工具 `ostress.exe`，它包含在 RML Utilities 中。你可以从 [`https://aka.ms/ostress.exe`](https://aka.ms/ostress.exe) 下载 RML。确保将 RML 安装的文件夹添加到你的系统路径中（默认路径为 `C:\Program Files\Microsoft Corporation\RMLUtils`）。
*   本章我还使用了 Azure SQL Database 中的数据库监视器和 Microsoft Copilot 技能，但这对你来说是可选的。

## HADR 功能

在我们通过示例深入探讨每个主题之前，我想先和你回顾一下 Azure SQL 附带的令人惊叹的 HADR 功能。

### 自动备份和时间点还原

Azure SQL 就是 SQL Server，因此完整的 `BACKUP` 和 `RESTORE` 功能是可用的。然而，PaaS 的承诺是提供**托管功能**。因此，Azure SQL 为托管实例和数据库提供了自动化备份系统，以满足你的恢复点目标 (RPO) 和历史数据需求。事实上，对于 Azure SQL 数据库，你完全无需接触 `BACKUP` T-SQL 语句。托管实例允许执行 `COPY_ONLY` 备份到 Azure 存储。

Azure SQL 的所有备份都存储在与数据库和日志文件分离的存储上，并带有自动的地域冗余镜像。Azure SQL 还提供长期备份保留选项。

Azure SQL 将使用完整备份、差异备份和日志备份来支持完整的时间点还原界面。此外，你还拥有以下还原能力：

*   还原已删除的数据库。
*   托持实例支持从 Azure Blob 存储执行 `RESTORE` T-SQL 语句，该存储可以来自本地 SQL Server 备份或托管实例的 `COPY_ONLY` 备份。
*   请记住，正如本书前面提到的，版本化的托管实例允许你将备份还原到 SQL Server 2022。



