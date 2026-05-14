# 第 1 章 为什么要在 Linux 上运行 SQL Server？

## 我们是如何构建它的

赫尔辛基项目，即 SQL Server on Linux，是我在微软 25 年职业生涯中遇到的最令人惊叹的软件成就之一。在本节中，我将向您展示我们在 Linux 上构建 SQL Server 并能够将其以高质量和高性能推向市场的惊人背景和历程。我将讨论一个重要的软件组件，它使 SQL Server 能够运行在 Linux 上，称为 SQL 平台抽象层（`SQLPAL`），它基于一个名为 `Drawbridge` 的微软研究院项目。此外，我还将探讨进程架构以及各个组件如何交互以使 SQL Server 在 Linux 上运行。

当时，增长最快的客户选择的操作系统变成了 Linux。我们拥有托管“混合”环境（Linux 和 Windows Server）的客户，他们询问我们是否会考虑推出 Linux 版本的 SQL Server。这并不是说他们放弃了 Windows Server，而是他们希望在公司内标准化使用 SQL Server，但需要同时支持 Linux 和 Windows Server 的选项。

最终，Linux 合作伙伴也开始询问我们是否会考虑移植到 Linux。像 Red Hat 和 SUSE 这样的公司在企业业务中看到了巨大的增长，并认为提供数据平台的选择将有助于客户采用。

所有这些因素在 2014 年末和 2015 年初萦绕在我们 SQL Server 工程领导层——Shawn Bice、Rohan Kumar 和 Lindsey Allen 的脑海中。他们在说服微软执行领导层允许我们为 Linux 构建 SQL Server 方面发挥了关键作用。他们还重新聘用了 Slava Oks 来负责构建，并让 Tobias Ternstrom 转而领导该项目的项目管理。他们将组建一个出色的团队来启动名为 `Helsinki` 的项目。

至此，我们已具备了动力、批准和资源来推进项目。现在的问题是，我们如何构建它？以及我们如何快速将其推向市场？

##### Drawbridge

2011 年 3 月，微软研究院的一个团队发表了一篇题为 *自上而下重新思考库操作系统* [`www.microsoft.com/en-us/research/wp-content/uploads/2016/02/asplos2011-drawbridge.pdf`](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/asplos2011-drawbridge.pdf) 的论文。该论文基于一个名为 `Drawbridge` 的项目原型和一个称为 `库操作系统` 的概念。回顾 2011 年，虚拟化是一个非常热门的话题，并且已经变得非常流行。虚拟机是执行整合项目和将应用程序与底层硬件抽象化的常用机制。它们提供隔离性、兼容性，并使您无需依赖特定的主机计算机。因此，它们至今仍然流行，并在 Azure 虚拟机和 Amazon EC2 等公共云环境中使用。唯一的问题是虚拟机资源“开销大”。也就是说，您需要整个操作系统在客户机中运行以支持您的应用程序，即使您并不需要客户机操作系统附带的所有服务。

`Drawbridge` 团队寻求一种创建更轻量级解决方案的方法，同时保留虚拟化的优势。此外，他们通过研究发现，Windows 应用程序所需的许多服务和应用程序编程接口（API）调用实际上并不需要在 Windows 内核中运行。相反，可以在用户模式下运行驱动许多 Windows API 的代码，从而减少对内核的上下文切换。减少上下文切换可以提高性能，并使应用程序和计算机资源的使用更加高效。

该项目的结果是一个概念，称为在 `库操作系统` 上运行的 `微进程`，有效地创建了一个平台抽象层（`PAL`）。图 1-1 展示了最终的成果


