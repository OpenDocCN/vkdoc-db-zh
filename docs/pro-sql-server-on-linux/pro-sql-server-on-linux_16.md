# 第 1 章：为什么 SQL Server 可以在 Linux 上运行？

对于 Windows，其可执行文件格式被称为可移植执行体（PE）格式，而对于 Linux，则被称为可执行与可链接格式（ELF）格式。任何为 Windows 编译和链接的程序都使用 PE 格式（例如 `SQLSERVR.EXE`），并且只能在 Windows 上运行；同样，Linux 上的 ELF 二进制文件也是如此。因此，我们的宿主扩展代码是 `SQLSERVR` Linux 进程的一部分，被编译为 ELF 二进制文件。它包含特殊逻辑，知道如何直接加载 `SQLPAL.DLL`（因为它是 PE 格式的二进制文件），并为它提供必要的链接以使用宿主扩展中的 ABI 接口（换句话说，宿主扩展指示 `SQLPAL` 如何“动态地与之对话”）。

宿主扩展还知道如何将磁盘上特殊打包文件中的 `SQLSERVR.EXE` 和 LibOS 二进制文件（我们将在第 2 章详细讨论这些）映射到进程地址空间。然后，`SQLPAL.DLL` 知道如何加载 `SQLSERVR.EXE`，并让正常的 Windows 进程和 DLL 加载过程从此处开始。这里有一个关键点：所有这些代码都是为英特尔兼容处理器编译的，因此，用 Robert Dorr 的话来说，“它本质上都是汇编代码。” 这也是我们的 `SQLSERVR.EXE` 代码无需更改就能在 Linux 上运行的原因之一。

所以，总结一下，`SQLSERVR.EXE` 及其依赖的 DLL 被加载和运行的方式与在 Windows 上完全相同。LibOS 组件也是如此。`SQLPAL.DLL` 是为 Windows 编译的，但其内部建有逻辑来实现 Windows 内核服务，或者在必要时通过调用其他函数（使用 ABI 接口）来实现某些内核服务。但 `SQLPAL.DLL` 并不知道这些接口是在为 Linux 编译的代码中实现的。宿主扩展处理了所有这些事情。

架构图中还有两个我尚未谈到的组件：

*   注意到 `SQLAGENT` 也列在此图中。这是因为在 Linux 上安装 SQL Server 时，你会将 `SQLAGENT.EXE` 与 `SQLSERVR.EXE` 一起加载到 `SQLSERVR` Linux 进程中。我知道这听起来很奇怪，但它工作得很好。`SQLPAL` 实现了进程隔离，这些进程并没有意识到它们处于同一个 Linux 进程中。
*   注意右下角有一个名为 `Parent Watchdog`（父看门狗）的进程。你将在本书后面听到更多相关内容，但实际上，这是一个名为 `SQLSERVR`、原生为 Linux 编译的进程，是 SQL Server 启动时最先启动的程序。然后我们使用 Linux 的 `fork()` API 调用来创建另一个 `SQLSERVR` 进程，该进程就是如图中所示的 SQL Server 引擎及其所有组件。它通过信号监控子 `SQLSERVR` 进程以执行转储，并在必要时使用 `systemd` 服务进行重启，从而提供了一个便捷的目的。你可以阅读这篇博文了解更多关于其工作原理的内容：[`blogs.msdn.microsoft.com/bobsql/2018/07/18/sql-server-on-linux-why-do-i-have-two-sql-server-processes/`](https://blogs.msdn.microsoft.com/bobsql/2018/07/18/sql-server-on-linux-why-do-i-have-two-sql-server-processes/)。随着我在第 2 章讲述探索 SQL Server on Linux 安装了什么，这种关系将变得更加明显。

我意识到这看起来非常复杂，事实也确实如此，但它同时也简单而优雅。正是这种架构，让我们能够快速且高质量、高性能地将 SQL Server on Linux 推向市场。

我在本章中包含这些细节，并不是因为你必须了解它们才能使用 SQL Server on Linux，而是为了澄清其工作原理上的任何困惑，并为该设计和架构提供可信度。

而最重要的一点是。正如你将在下一节学到的，其核心...



SQL Server 数据库引擎与我们产品历史上在数千台客户服务器上运行的，是经过验证、可扩展的相同 SQL Server 引擎。

## Windows 上的 SQL Server 与 Linux 上的是否相同？

我记得曾就此话题与 Slava Oks 有过一次对话，他对我说：“Bob，查询处理器依然是同一个查询处理器。” 这句话解释了为何我们能够在 Linux 上的 SQL Server 实现与 Windows 上可比的性能。它也解释了为何数据库可以跨平台还原，以及为何如果应用程序原本是为在 Windows Server 上运行针对 SQL Server 而构建的，它们几乎无需更改即可连接到 Linux 上的 SQL Server。

## Linux 上 SQL Server 的功能

SQL Server 2017 拥有许多专注于性能、安全性和高可用性的功能和特性。以下是 Linux 上 SQL Server 可用的部分功能和特性：

- 核心的 `SQLOS` 系统，用于调度、内存管理及资源治理与管理，提供内置的可扩展性，并支持重要的服务器架构，如 `NUMA`。
- 用于缓冲区管理、查询处理、查询执行、存储引擎和访问方法的核心引擎组件。
- 核心管理操作，如 `BACKUP/RESTORE`、索引管理和 `DBCC` 命令。
- 我们著名的 `T-SQL` 语言，除了该版本中不支持的任何特性或功能外，其余均可原样工作。
- 内存中工作负载特性，如 `列存储索引` 和 `内存中 OLTP`。
- 新的数据库智能特性，如自适应查询处理 (`AQP`) 和 `自动调整`（你将在第 4 章听到更多关于这些特性的内容），这些特性基于 `查询存储` 的遥测数据。
- `Always On 可用性组`（在本书后续内容中，我将用“可用性组”或 `AG` 来指代此功能）得到完全功能支持。正如你将在第 8 章看到的，可用性组通过一种称为 `Pacemaker` 的集群技术支持主要的故障转移功能（少数例外）。此外，SQL Server 2017 中的一个新特性提供了对 `无集群` 可用性组的支持，无需集群软件。
- Linux 上的 `Always On 故障转移群集实例` 使用 `Pacemaker` 得到支持。
- 安全特性，如 `始终加密`、`动态数据屏蔽`、`行级安全性`、`审核` 和 `透明数据加密 (TDE)`。
- SQL Server 和 `Active Directory` 身份验证用于登录。
- 加密连接使用 `传输层安全性 (TLS)` 得到支持。
- 丰富的编程特性，如 `SQLCLR`（仅限 SAFE 程序集）、`JSON` T-SQL 功能和 `图数据库`。
- `SQL Server 代理` 调度服务支持 `T-SQL` 命令子系统。
- `SQL Server 集成服务 (SSIS)` 支持基本的提取、转换和加载 (`ETL`) 操作。
- 工具可直接用于连接 Linux 上的 SQL Server，包括 `SQL Server Management Studio`（运行于 Windows）、`SQL Server Data Tools`（运行于 Windows）以及我们在 `Visual Studio Code` 中的 `mssql` 扩展（跨平台工具）。
- 我们在 Linux 上支持原生命令行工具，包括 `sqlcmd` 和 `bcp`。
- 我们构建了新的开源、跨平台工具，可在 Windows、Linux 或 MacOS 上运行，并连接到 Linux 或 Windows 上的 SQL Server：`SQL Server Operations Studio` 和 `mssql-cli`。
- SQL Server 诊断功能，如 `扩展事件`、`动态管理视图`、`目录视图` 和 `查询计划` 诊断能力。

我可能遗漏了你最喜欢的功能，但这份列表使得 Linux 上的 SQL Server 成为一个非常吸引人的方案。有关 Linux 上 SQL Server 功能的完整列表，请查阅我们的文档：[`docs.microsoft.com/sql/linux/sql-server-linux-editions-and-components-2017`](https://docs.microsoft.com/sql/linux/sql-server-linux-editions-and-components-2017)。



对于阅读本书但不熟悉 SQL Server 版本的读者来说，了解某些功能仅在特定版本中可用这一点非常重要。要获取特定版本可用功能的完整列表，请参阅此文档页面：[`docs.microsoft.com/sql/linux/sql-server-linux-editions-and-components-2017?view=sql-server-linux-2017#includessnoversionincludesssnoversion-mdmd-editions`](https://docs.microsoft.com/sql/linux/sql-server-linux-editions-and-components-2017?view=sql-server-linux-2017#includessnoversionincludesssnoversion-mdmd-editions)。

Linux 上的 SQL Server 2017 提供以下版本：

*   **企业版**：这是功能最全面的版本。正如其名，它设计用于企业级数据库应用程序。从许可角度来看，企业版有两种变体：企业版和企业核心版。企业核心版具有全部功能，而企业版在使用特定数量的计算核心方面有一些限制。企业版仅适用于与 Microsoft 有合同协议的特定客户。
*   **标准版**：此版本旨在为面向较小部门或中等规模工作负载的应用程序提供 SQL Server 的基本功能。我们从 SQL Server 2016 SP1 开始做出的一个重大改变是，向标准版开放了一些之前仅在企业版中可用的功能。此举的理由是确保开发人员可以构建应用程序，而无需过多担心他们的应用程序针对的是哪个 SQL Server 版本。这些功能在标准版上有大小限制，但现在都可以使用了。你可以在此博客文章中阅读有关此更改的更多信息：[`blogs.msdn.microsoft.com/sqlreleaseservices/sql-server-2016-service-pack-1-sp1-released/`](https://blogs.msdn.microsoft.com/sqlreleaseservices/sql-server-2016-service-pack-1-sp1-released/)。
*   **开发版**：这是一个免费版本，包含企业版的所有功能。但是，此版本的许可限制其不能用于生产环境。你可以使用此版本来构建和测试你的应用程序。
*   **Web 版**：此版本类似于标准版，但限制更低，并且专门针对 Web 托管商定价。
*   **速成版**：这是最基本的版本，但免费且可用于生产环境。不过它的限制很大，不应被用于任何类型的应用程序扩展。但如果你作为一名开发人员刚开始接触 SQL Server，SQL Server 速成版可能会很有用。它有一个通往标准版和企业版的简易升级路径。对于 Linux，SQL Server 速成版可以作为一个非常实用的、仅用于配置的副本服务器。这将在第 8 章中更详细地讨论。

我还应该提到，SQL Server 提供了一个评估版。它包含企业版的所有功能，但未获得生产使用的许可，并且具有基于时间的过期许可。但这是在企业服务器上测试 SQL Server 功能的绝佳方式。在第 2 章中，我将描述如何选择你想要使用的 SQL Server 版本。

## 哪些功能不可用

拥有如此强大的功能阵容，也意味着 SQL Server 中有一些领域是不可用的



2017 年在 Linux 上发布（自普遍可用版本起）。SQL Server 产品包含的某些功能存在依赖项或需要外部程序，这些程序在 Linux 上的 SQL Server 中运行并非如此直接。

`注意` 我们正在积极研究许多此类功能，以便在未来版本或 SQL Server on Linux 的更新中包含。

