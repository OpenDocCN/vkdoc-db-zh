# 如何连接和使用 Linux 上的 SQL Server 2022

一旦您在 Linux 上部署了 SQL Server，您就会想要连接该实例并开始使用 T-SQL 语句。我在本章的这一部分包含了一些简要提示。如果您正在寻找 Linux 上 SQL Server 的练习，您可以轻松使用来自 *Pro SQL Server on Linux* 一书的脚本（您需要为 SQL Server 2022 调整软件包名称），网址是 `https://github.com/microsoft/bobsql/tree/master/sqllinuxbook`。

## ssh 与 rdp

对于习惯使用远程桌面（`rdp`）的 Windows 用户，Linux 有一个标准的远程 shell，称为安全外壳（`ssh`），它是一个命令行界面，用于连接到计算机或虚拟机上的 bash shell。有多种方法可以使用 `ssh` 连接到您的 Linux 计算机或虚拟机。以下是我使用 `ssh` 的一些方式：

*   `ssh` 内置于 PowerShell 中，因此您可以直接从 PowerShell 命令提示符使用它。

*   如果您在 Azure 中进行了部署，Azure Cloud Shell 附带一个 bash shell 界面，并且包含了 `ssh`。

*   我个人喜欢使用免费的 MobaXterm 工具，您可以从 `https://mobaxterm.mobatek.net` 下载。这个工具的一个优点是它有一个图形界面，可以方便地从本地计算机上传和下载文件到 Linux。

## 使用我们的工具连接

由于 Linux 上的 SQL Server 是一个兼容的 SQL Server 引擎，您可以使用通常连接到 Windows 上 SQL Server 的喜爱工具，例如 SSMS。

Azure Data Studio (ADS) 已经变得非常流行，并且与 Linux 配合良好。ADS 的一个优势是它是跨平台的，因此可以在 Windows、Linux 或 macOS 上运行。ADS 还附带了笔记本的概念，您已经在本书的一些练习中见过。

命令行工具 `sqlcmd` 与 Linux 上的 SQL Server 配合良好。该工具有一个 Linux 版本，根据您的 Linux 发行版，它作为单独的软件包需要安装。

我们还有一个跨平台的替代命令行工具叫做 `mssql-cli`，您可以在 `https://docs.microsoft.com/sql/tools/mssql-cli` 阅读更多信息。

## 使用 mssql-conf 进行配置

对于 Windows 上的 SQL Server，您可能习惯于使用 SQL Server 配置管理器工具。对于 Linux 上的 SQL Server，等效的工具叫做 `mssql-conf`。`mssql-conf` 可用于配置各种设置，例如启用 SQL Server Agent 或设置跟踪标志。您可以在 `https://docs.microsoft.com/sql/linux/sql-server-linux-configure-mssql-conf` 阅读有关 `mssql-conf` 支持的所有选项的更多信息。

**注意**

SQL Server 内部的配置仍然可以通过使用 `ALTER SERVER CONFIGURATION` 或 `sp_configure` 来支持。



## 使用 adutil 进行 Active Directory 身份验证

Linux 上的 SQL Server 支持 SQL 和 Active Directory (AD) 两种登录身份验证方式。当我们最初推出适用于 SQL Server 2017 甚至 2019 的 Linux 版 SQL Server 时，设置 AD 身份验证相当复杂。我们现在拥有一个名为 `adutil` 的工具，可以简化此过程。`adutil` 工具适用于 SQL Server 2019 和 2022，并且也支持容器。您可以在 `https://docs.microsoft.com/sql/linux/sql-server-linux-ad-auth-ad-util-introduction` 阅读更多关于如何使用 `adutil` 的信息。

## SQL Server 的 Azure 扩展

目前，SQL Server 的 Azure 扩展并未包含在 Linux 版 SQL Server 的安装过程中。但您可以使用一种已有的方法，通过 Azure Arc 启用的服务器来支持 Linux 上的 SQL Server，具体文档请参见 `https://docs.microsoft.com/sql/sql-server/azure-arc/connect`。

正如我在本书第 3 章所述，SQL Server 的 Azure 扩展是一项 Windows 服务，它是 Azure Arc 代理框架的一部分，该框架本身也作为 Windows 服务运行。在 Linux 上，这些程序以守护进程形式运行，分别称为 `himds`（Arc 代理）和 `SqlServerExtension.Service`（SQL 扩展）。

## 优化 Linux 上的 SQL Server 2022

除了您使用的典型 SQL Server 优化技术（如创建适当的索引）外，运行 Linux 版 SQL Server 时还需考虑一些优化措施：

自从我们推出适用于 Linux 的 SQL Server 2017 以来，我们已经从客户经验和基准测试中学习到如何以非常具体的方式调整 Linux 内核和 SQL Server，以实现最佳性能。`https://docs.microsoft.com/sql/linux/sql-server-linux-performance-best-practices` 提供了一份非常详细的推荐配置列表。如果您只是试用 Linux 上的 SQL Server，可能不需要考虑这些配置。然而，为了获得最佳的生产环境体验，我强烈建议您通读所有这些配置。

在 Windows 上的 SQL Server 之外，存在一些设置，例如即时文件初始化 (IFI) 和锁定内存页。这些设置不适用于 Linux 上的 SQL Server。

您可能会在 Linux 上安装 SQL Server 后立即看到这两条消息出现：

```
ForceFlush is enabled for this instance
ForceFlush feature is enabled for log durability
```

我的长期同事 Robert Dorr 花费了大量时间研究 Linux 上 SQL Server 的 I/O 一致性和性能。这项工作促成了一系列我们推荐的配置选项，您可以在 `https://docs.microsoft.com/sql/linux/sql-server-linux-performance-best-practices#linux-os-configuration` 的标题为“SQL Server 和强制单元访问 (FUA) I/O 子系统功能”部分阅读这些内容。如果您喜欢阅读内幕故事，请查看 Bob 的博客文章：`https://techcommunity.microsoft.com/t5/sql-server-blog/sql-server-on-linux-forced-unit-access-fua-internals/ba-p/3199102`。

避免 OOM killer。

听起来像恐怖电影，对吧？嗯，如果您遇到内存不足 (OOM) 并导致程序被终止的情况，那可能确实很可怕。SQL Server 在动态内存管理方面做得很好，但如果它在 Linux 内消耗过多内存，就可能会遇到此问题。这就是我们创建名为 `memorylimitmb` 的配置选项的原因。默认情况下，我们实际上限制了 SQL Server 在 Linux 上可以使用的总内存，因此这对您来说可能不是问题。但我鼓励您在 `https://docs.microsoft.com/sql/linux/sql-server-linux-performance-best-practices#sql-server-configuration` 的标题为 **使用 mssql-conf 设置内存限制** 的部分阅读更多细节，以确保您的实例保持稳定。

## Linux 上 SQL Server 2022 的 HADR

SQL Server 内置了强大的高可用性和灾难恢复 (HADR) 功能，包括故障转移群集、Always On 可用性组 (AG) 以及丰富的备份/恢复选项。

让我们看看如何将这些功能用于 Linux 上的 SQL Server。

### 故障转移群集

在 Windows 上的 SQL Server 中，SQL Server 与 Windows Server 故障转移群集 (WSFC) 集成，以支持共享存储的自动高可用性功能。由于 Linux 上没有 WSFC，您可以使用名为 `Pacemaker` 的软件包来支持 SQL Server 故障转移群集。您可以在 `https://docs.microsoft.com/sql/linux/sql-server-linux-shared-disk-cluster-configure` 阅读更多关于如何设置此功能的信息。

### Always On 可用性组 (AG)

核心 AG 软件都内置于 SQL Server 引擎中。这就是为什么对于 Linux（或 Windows）上的 SQL Server，您可以设置一个*无群集*的可用性组（如果您记得我在本书第 6 章展示了如何为包含式可用性组执行此操作）。此选项不提供自动故障转移，但提供了一个副本方案。我们也将其称为读取横向扩展可用性组，因为它是分离只读工作负载的完美解决方案。您可以在 `https://docs.microsoft.com/sql/linux/sql-server-linux-availability-group-configure-rs` 阅读更多关于设置读取横向扩展可用性组的信息。

我们还支持使用 Pacemaker 在 RHEL、Ubuntu 和 SLES 上运行的 AG 实现完全自动故障转移解决方案。您可以在 `https://docs.microsoft.com/sql/linux/sql-server-linux-availability-group-cluster-rhel` 阅读更多关于如何在 RHEL 上设置此功能的信息。您也可以使用 Azure 虚拟机设置此配置，相关信息请参阅 `https://docs.microsoft.com/azure/azure-sql/virtual-machines/linux/rhel-high-availability-stonith-tutorial`。

### HPE Serviceguard

事实证明，我们所做的一项工作是将支持 SQL Server 控制 Always On 可用性组故障转移的代码放入了一个开源项目，您可以在 `https://github.com/Microsoft/mssql-server-ha` 找到。通过这样做，我们允许其他合作伙伴将其故障转移解决方案与 SQL Server 集成。其中一种解决方案来自 HPE，名为 Serviceguard。您可以在 `https://docs.microsoft.com/sql/linux/sql-server-availability-group-ha-hpe` 阅读更多关于如何将 HPE Serviceguard 与 SQL Server 配合使用的信息。

### 备份/恢复

SQL Server 的所有备份/恢复功能在 Linux 上的 SQL Server 中均可用，包括使用 URL 语法备份到 Azure 存储，以及 SQL Server 2022 新增的 S3 对象存储备份功能。

您在本书第 6 章也了解到，支持使用 T-SQL 进行快照备份，无需编写 VDI 程序或依赖 Windows VSS。实际上，我们也称此功能为*跨平台快照备份*，因为我们希望在 Linux 上支持快照备份。



## SQL Server 2022 容器

我记得大约在 2010 年，在虚拟机上运行 `SQL Server` 变得非常流行。这种从裸机抽象出来的能力，以及能够将多台机器整合到一台主机上的优势，在许多方面引发了计算领域的革命。

近年来，在 `Linux` 系统上，容器的概念对于应用程序，特别是对于开发人员来说，变得极其流行。`SQL Server 2022` 中的 `SQL Server` 容器在技术上并没有根本性的新变化。但回顾一下为何可以使用容器、如何使用它们，以及一些它们能发挥强大作用的有趣场景，仍然非常有价值。

请通过 [`https://aka.ms/sqlcontainers`](https://aka.ms/sqlcontainers) 随时了解关于 `SQL Server` 容器的最新动态。

### 为何使用容器？

关于容器，我总是需要破除的一个迷思是它们被用来替代虚拟机。这实际上是可以实现的，但我发现它们通常是对虚拟机的补充。容器的一个强大用途是将应用程序整合到运行在虚拟机中的多个容器里。

容器实际上是一个不可变镜像的实例，该镜像包含一个或多个程序，这些程序在隔离的环境中运行，并拥有完整的私有文件系统。容器仅包含从镜像运行其内部程序所必需的文件。

例如，`SQL Server` 并不需要 `Linux` 系统上运行的每一个文件和进程。因此，现在你可以在一个虚拟机中部署多个 `SQL Server` 容器，而不是部署多个虚拟机，从而节省空间和资源使用。事实上，运行多个 `SQL` 容器正是你可以在同一台虚拟机或 `Linux` 计算机上运行多个 `SQL Server` 实例的方式，因为我们不支持命名实例。

关于容器的另一个迷思是，它们的性能不如普通进程，因为它们在线程、内存、I/O 等方面从底层的 `Linux` 操作系统中被抽象了出来。这并不正确。容器只是以隔离的方式在操作系统中运行（容器只知道其内部的进程），但它们可以直接访问操作系统资源。因此，在 `Linux` 上运行的 `SQL` 容器应该与在 `Linux` 上运行的 `SQL Server` 具有相同的性能。

注意：请确保你是在正确的环境下进行比较。`Windows` 上的 `SQL` 容器运行在由 `Windows`（或 `WSL`）托管的 `Linux` 虚拟机上。这与直接在由 `Linux` 托管的 `Linux` 虚拟机中运行容器相比，多了一层抽象。

容器由像 `Docker` 这样的容器运行时来运行。实际上，像 `Docker` 这样的容器运行时知道如何获取容器镜像，并使用原生的 `Linux` API（例如 `namespaces` 和 `cgroups`）以隔离的方式执行与容器关联的程序。容器运行时还知道如何管理容器，例如停止和启动容器。

容器还有其他优势，包括：

*   **可移植性** – 容器可以在任何支持像 `Docker` 这样的容器运行时的操作系统上运行，并且你可以确信这是同一个容器镜像。因此，`SQL Server` 容器可以在 `Windows`、`Linux` 或 `macOS` 上运行，并且你可以确信这是同一个 `SQL Server` 引擎。在 `Windows` 和 `macOS` 上运行的 `Linux` 容器通过某种虚拟化技术来运行 `Linux` 程序。这种可移植性对于开发来说意义重大。无需试图维护一个供所有开发人员共享的 `SQL Server` 开发服务器，只需为他们提供 `SQL Server` 容器，让他们在任何操作系统的自有环境中运行即可。

*   **一致性** – 假设你希望所有开发人员都使用特定的 `SQL Server 2019`（或者当我们开始发布 `CU` 版本时的 `2022`）累积更新版本，以及特定的数据库架构和脚本。容器提供了跨任何操作系统实现此功能的能力。微软为 `SQL Server` 主要版本（2017、2019 和 2022）的每个 `CU` 版本都制作了容器镜像。

*   **效率** – 在 `Windows` 上为 `SQL Server` 打累积更新补丁，需要你安装一个单独的程序，该程序会在现有的 `RTM` 版本或更新之上安装更新。`SQL Server` 在 `Linux` 上有一个升级选项，可以升级到较新的 `CU` 版本。容器则不需要任何补丁更新。事实上，你无法给容器打补丁。取而代之的是，你可以切换容器。在标题为“容器切换方法”的部分中阅读更多信息，了解其工作原理。

以下是我在 `SQLBits` 上做的一个关于 `SQL Server` 容器的演讲，以了解更多内容：[`https://sqlbits.com/Sessions/Event18/Inside_SQL_Server_Containers`](https://sqlbits.com/Sessions/Event18/Inside_SQL_Server_Containers)。

### 使用 SQL Server 2022 容器

我忘了提及 `SQL Server` 容器最强大的功能之一。`SQL Server` 容器是预安装的 `SQL Server`（并且默认是 `Developer Edition`，所以它是免费的！）。我们为 `SQL Server` 构建的容器镜像，使得当你运行它们时，`SQL Server` 会直接启动并执行。其部分奥秘在于 `SQLSERVR.EXE` 被设计为一个独立程序（实际上是一个守护程序）运行，而我们的容器镜像被设置用来配置和启动 `SQL Server` 引擎。



