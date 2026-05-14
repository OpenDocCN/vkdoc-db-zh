# 10. 在 Azure 虚拟机上运行的 SQL Server 2022

如今部署 SQL Server 2022 时，您可以选择在笔记本电脑、"物理"服务器甚至容器中运行 SQL Server。部署和运行 SQL Server 最常见的选择是 `虚拟机`。虽然使用 Hyper-V 和 VMWare 等虚拟化技术运行 SQL Server 非常流行，但许多客户现在正转向云端以获得基础设施即服务 (IaaS) 的体验。我相信使用 `Azure 虚拟机` 会为您提供出色的体验。

在 Azure 虚拟机上运行 SQL Server 的体验在不断发展和完善。因此，尽管 Azure 虚拟机针对 SQL Server 2022 并没有新增特定功能，但我认为专门用一章来介绍 Azure 虚拟机上的 SQL Server 对本书读者来说会很有价值。

在本章中，我将展示如何在 Azure 虚拟机上部署 SQL Server，以及如何使用、管理、优化和监控它。

## 什么是 Azure 虚拟机上的 SQL Server？

您可能已有现有的 SQL Server 虚拟机 (VM) 部署，或者正在考虑新建一个。那么，在哪里托管这个 VM 呢？其中一种选择是在 Azure 中，称为 Azure 虚拟机。Azure 虚拟机被称为基础设施即服务 (IaaS)，因为 Azure 将托管基础设施作为一种服务提供给您，用于运行 Windows 或 Linux 的虚拟机。这意味着 Azure 将托管您的虚拟机，并提供所有硬件，包括计算、存储和网络。您将负责管理 `guest` 虚拟机内的所有内容，包括操作系统和 SQL Server。我们会采取一些措施来帮助您管理 `guest` 虚拟机环境，您将在本章中看到这些内容。

由于您对虚拟机内部拥有完全控制权，因此可以自由使用 Windows 或 Linux 操作系统（包括容器）。在此环境中运行的 SQL Server 是完整的"盒装"产品，包括数据库引擎和所有服务。实际上，您可以在本地环境虚拟机中运行的任何 SQL Server 功能，都可以在 Azure 虚拟机中运行。这包括引擎外部的服务，如 SQL Server 集成服务 (SSIS)、SQL Server 分析服务 (SSAS) 和 SQL Server 报告服务 (SSRS)。

## 部署规划

每当您要部署新的 SQL Server 实例时，进行适当合理的规划将节省您的时间、金钱和精力。当您在 Hyper-V 虚拟机上规划新的 SQL 部署时，您会考虑诸如 CPU 数量、CPU 类型和速度以及内存等因素。需要什么样的存储容量、性能和磁盘数量？您甚至可能在拥有数据中心的公司工作，公司要求您使用网站来请求配置虚拟机。在此类场景中，通常会选择 VM `size`，这决定了您的 SQL Server 实例可用的资源。

您还需要知道应用程序和工具将如何连接到 SQL Server。您会使用远程桌面 (`rdp`) 在 VM 内部或远程运行 SSMS 等工具吗？如何设置虚拟机的网络以连接到您的应用程序？高可用性和灾难恢复呢？这些都是任何生产 SQL Server 部署都需要考虑的问题。

对于 Azure 虚拟机，这些决策并无不同。一个很大的不同在于，现在您的 SQL Server 虚拟机托管在 Azure 中。这意味着您在可能的虚拟机资源方面的选择取决于 Microsoft 提供的选项（但您将看到选择范围非常广泛）。

让我们回顾一些您需要做出的重要决策，以获得最佳的整体体验。您将使用这些规划决策在部署期间和之后做出选择。我认为您应该在部署前考虑这些选择的一个重要原因是，您可能会因为不确定如何选择而在部署时受阻。

-   您需要一个 `Azure 订阅` 来部署虚拟机，并决定使用哪个 `资源组`。资源组是组织多个 Azure 资源的绝佳方式，但它们也可以是在 Azure 中创建自己的虚拟网络的便捷方法。在 [`https://docs.microsoft.com/azure/azure-resource-manager/management/overview#resource-groups`](https://docs.microsoft.com/azure/azure-resource-manager/management/overview#resource-groups) 阅读更多关于资源组的信息。

-   **查看 Azure 中可用的虚拟机大小**，地址为 [`https://aka.ms/azurevmsizes`](https://aka.ms/azurevmsizes)。为了帮助您缩小选择范围，请查看我们关于 SQL Server 的 Azure VM 大小建议，地址为 [`https://aka.ms/SQLIaaSSizing`](https://aka.ms/SQLIaaSSizing)。同时请放心，在许多情况下，您可以在部署后更改 VM 大小，通常只需很少的停机时间。

> **提示**
>
> 您将在本章后面看到一个问题，我选择的虚拟机大小对存储性能设置了上限，该上限低于数据、日志和 tempdb 的磁盘存储选择。这可能导致意外且难以发现的性能问题。请花时间阅读我们的文档 [`https://docs.microsoft.com/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices-checklist`](https://docs.microsoft.com/azure/azure-sql/virtual-machines/windows/performance-guidelines-best-practices-checklist)，以对齐这些选择。

-   在 Azure 中托管 SQL Server 和数据是否存在**信任问题**？请访问 [`https://aka.ms/azuretrust`](https://aka.ms/azuretrust) 获取有关 Azure 隐私、合规性、安全层和监控的更多信息。


