# 第六章

## 性能功能

若要提供一个具备行业竞争力、能处理全球最大企业工作负载的数据平台，数据库引擎必须快速。该引擎必须天生快速，且必须提供配置、调优等功能，使用户和开发者能够最大限度地利用计算资源。它还必须提供相应功能，让用户只需极少甚至无需修改应用程序，就能获得超出预期的性能加速。最终，一个世界级的数据平台应内置能适应并自动调整常见查询性能问题的能力。

运行在 `Linux` 上的 `SQL Server` 拥有这一切乃至更多。若您已阅读本书前面所有章节，我已展示了部署的基础知识，以及如何创建数据库和应用程序。我还展示了用于最大化 `SQL Server` 投资回报的大量工具和功能。在本章中，我将描述 `SQL Server` 数据平台的主要性能功能。本章的目标是让任何阅读者都能理解如何使用运行在 `Linux` 上的 `SQL Server`，为数据库应用程序实现最佳性能。

我将首先描述 `SQL Server` 数据库引擎的内置功能，这些是实现最高性能所需的基础。然后，我将讨论实现最佳性能的各种配置选择，包括 `SQL Server` 实例、数据库以及 `Linux` 操作系统的配置。本章将提供一节关于性能调优的内容，以便您能确保最高效、最充分地利用 `SQL Server` 资源。接着，我将讨论 `SQL Server` 中允许应用程序超越传统数据库应用功能实现性能加速的特性。最后，我将以讨论 `SQL Server` 中提供自动化以适应和自动调整查询性能的新特性来结束本章。

在 2017 年的 Microsoft Ignite 大会上，我进行了一场关于 `SQL Server` 性能功能的演讲，题为“体验 Microsoft SQL Server 2017：速度与激情”。我建议您观看此演讲作为本章的补充。您可以在以下网址找到该视频：[`channel9.msdn.com/Events/Ignite/Microsoft-Ignite-Orlando-2017/BRK3109`](https://channel9.msdn.com/Events/Ignite/Microsoft-Ignite-Orlando-2017/BRK3109)。

© Bob Ward 2018

B. Ward, *Pro SQL Server on Linux*, [`doi.org/10.1007/978-1-4842-4128-8_6`](https://doi.org/10.1007/978-1-4842-4128-8_6)

## 第六章 性能功能

本章的示例将使用完整的 `WideWorldImporters` 数据库。因此，在继续之前，您需要将 `WideWorldImporters` 备份文件复制到您的 `Linux` 服务器上。您将在本章中还原它，然后将其用于各种示例。要进行复制，如果您的 `Linux` 服务器连接到互联网，请使用以下命令：

```
wget https://github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Full.bak
```

我提供了以下脚本来帮助您在 `Linux` 服务器上还原此示例：

`cpwwi.sh`：将 `WideWorldImporters-Full.bak` 文件复制到 `/var/opt/mssql`

`restorewwi_linux.sql`：还原 `WideWorldImporters` 数据库

`restorewwi.sh`：使用 `sqlcmd` 执行 `restorewwi_linux.sql` 脚本

请记住，可通过点击本书在 Apress.com 目录页面上的“下载源代码”按钮获取这些脚本。该页面的网址是 [www.apress.com/us/book/9781484241271](http://www.apress.com/us/book/9781484241271)。

### 注意

在本章中，我经常提到此数据库中的某些对象，因此它



查看完整的 T-SQL 脚本有助于了解数据库的构成。我在示例中提供了一个名为`wwi.sql`的脚本，其中包含了数据库定义和所有对象。

#### 内置性能

我们构建的 SQL Server 数据库引擎快速且可扩展，足以满足地球上最庞大的数据库工作负载需求。我撰写本章的这一部分，并非要求读者执行任何操作，而是为了让读者了解使用 SQL Server *可能*实现什么。Linux 上的 SQL Server 数据库引擎旨在理解如何动态扩展并最大化利用计算机、虚拟机或容器的 CPU、I/O 和内存资源。读者阅读本书可能是为了了解 SQL Server，并判断它是否是一个值得信赖、可运行业务的数据平台，或者是否可以作为新应用程序的数据源。本节旨在提供必要的信息来回答这些问题。

要了解 SQL Server 的潜力，只需查看我们的 TPC 基准性能即可。目前，Linux 上的 SQL Server 在衡量分析查询性能的 1TB TPC-H 基准测试中拥有两个最佳性能结果。（您可以阅读更多关于 TPC 基准测试的信息：[`www.tpc.org`](http://www.tpc.org)，并且可以在这里查看我提到的特定 TPC-H 基准测试结果：[`www.tpc.org/3331`](http://www.tpc.org/3331) 和 [`www.tpc.org/3327`](http://www.tpc.org/3327)。）

Linux 上的 SQL Server 2017 建立在我们工程团队在 SQL Server 2016 中完成的出色工作之上。作为本章这一部分的补充，我鼓励您阅读名为“SQL Server 2016 性能飞跃”的博客系列：[`blogs.msdn.microsoft.com/bobsql/tag/it-just-runs-faster`](https://blogs.msdn.microsoft.com/bobsql/tag/it-just-runs-faster)，以及我在 2016 年 Microsoft Ignite 大会上关于同一主题的演讲：[`channel9.msdn.com/Events/Ignite/2016/BRK3043-TS`](https://channel9.msdn.com/Events/Ignite/2016/BRK3043-TS)。

##### SQL Server 内置可扩展性

我在第 1 章（作为 SQL Server 架构的一部分）描述了 SQLOS 组件。`SQLOS`旨在为 SQL Server 引擎提供调度和内存服务。所有 SQL Server 引擎组件都使用`SQLOS`来创建工作线程池并执行任务。SQL Server 引擎组件必须使用`SQLOS`服务，通过非抢占式系统进行调度。使用这种调度系统可以最小化内核上下文切换，从而最大化 CPU 资源使用率和效率。

为了提供这些服务，`SQLOS`根据检测到的逻辑 CPU 数量创建一个*schedulers*（调度器）列表，并从一个总的线程池中为每个调度器分配一组工作线程。当创建新任务来执行一个工作单元（例如，登录或查询）时，它们会被分配到特定调度器上的工作线程运行。此外，调度系统设计用于通过*nodes*（节点）来检测和利用 NUMA 计算机架构。节点和 CPU 为`SQLOS`和 SQL Server 提供了天然的扩展单元。`SQLOS`允许工作线程在 NUMA 节点内的任何 CPU 上运行，但避免跨越节点边界以减少对外部内存的访问。识别 NUMA 节点和 CPU 使得 SQL Server 能够将工作线程分散在各个 CPU 上以提供最大的可扩展性。此外，SQL Server 可以按节点和/或 CPU 分区内部数据结构和列表，以确保代码在高并发用户工作负载下不会遇到瓶颈。

`SQLOS`通过以下这些 DMV（动态管理视图）公开调度器、节点和 CPU 信息：
- `dm_os_schedulers`：此 DMV 列出了 SQL Server 用于工作线程调度的调度器，包括统计信息……

（此处原文结束）



### 第 6 章：性能能力

## 概述

本节将讨论 SQL Server 中的各种性能能力，重点关注 DMV（动态管理视图）和系统特性。

## 调度器与任务

你可以使用 DMV 来查看有多少工作线程正在运行，或者有多少任务正在等待从池中获取工作线程。图 6-1 展示了在我的一台配置有四个逻辑 CPU 的虚拟机上运行的 Linux 版 SQL Server 中，此 DMV 的输出示例：（注：你可以使用示例脚本 `dm_os_schedulers.sql` 来查看你的 SQL Server 的此输出）。

**图 6-1.** 在运行于 Linux 上、拥有四个 CPU 的虚拟机中的 `dm_os_schedulers`

四个状态为 `VISIBLE ONLINE` 的调度器是用于运行 SQL Server 任务和工作线程的*正常*调度器。`HIDDEN` 调度器则用于安排一些后台任务和其他任务（如备份）的工作。你可以在 [`docs.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-os-schedulers-transact-sql`](https://docs.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-os-schedulers-transact-sql) 找到此 DMV 的完整文档。

## SQLOS DMV

### `dm_os_nodes`

此 DMV 列出了运行 SQL Server 的服务器上检测到的所有 NUMA 节点。即使是不具备 NUMA 节点的服务器也会有一个 `node_id=0`。`Node_id=64` 是为专用管理员连接（DAC）保留的一个特殊节点。我将在本书后面的章节中详细讨论 DAC。

SQLOS 通过这些 DMV 暴露任务、工作线程和线程：

### `dm_os_workers`

此 DMV 列出了所有正在执行或能够为 SQL Server 执行任务的工作线程。SQL Server 的工作线程是在调度器分组中的池中创建的。SQL Server 将从池中分配工作线程来执行到达的任务（而不是为每个请求专门分配一个线程）。默认情况下，`max worker threads = 0` 的配置（这意味着*动态*）。值为 0 表示 SQL Server 根据 [`docs.microsoft.com/sql/database-engine/configure-windows/configure-the-max-worker-threads-server-configuration-option`](https://docs.microsoft.com/sql/database-engine/configure-windows/configure-the-max-worker-threads-server-configuration-option) 中记录的公式创建一个最大工作线程池。

`sys.dm_os_sys_info` DMV 列出了当前计算出的 `max worker threads` 值。对于大多数场景，你应该能够将 `max worker threads` 配置值保留为 0，让 SQL Server 计算该值。然而，在某些情况下，你可能希望使用系统存储过程 `sp_configure` 覆盖默认值，将 `max worker threads` 更改为固定值。我将在本章后面讨论这个选择。

### `dm_os_tasks`

SQL Server 中的任何*工作单元*都是一个任务。任何登录、查询或后台任务都是一个任务。可以将任务视为 SQL Server 中用于在 SQL Server 引擎中执行代码的函数。任务由工作线程运行。在某些情况下，任务可能需要等待工作线程。在这些情况下，该任务将在此 DMV 中列出，并且在 `dm_os_waiting_tasks` 中该任务的 `wait_type` 将是 `THREADPOOL`。大量任务等待 `THREADPOOL` 可能表明 SQL Server 存在问题。

我多年前在这篇博客文章中记录了一些关于 `THREADPOOL` 等待的信息：[`blogs.msdn.microsoft.com/psssql/2009/11/24/doctor-this-sql-server-appears-to-be-sick/`](https://blogs.msdn.microsoft.com/psssql/2009/11/24/doctor-this-sql-server-appears-to-be-sick/)

我的朋友 Paul Randal 在他的博客文章 [`www.sqlskills.com/help/waits/threadpool/`](https://www.sqlskills.com/help/waits/threadpool/) 中对如何处理 `THREADPOOL` 等待有另一段精彩的描述。当一个任务绑定到工作线程时，它就变成了一个*请求*，并将出现在 `sys.dm_exec_requests` 中。

## 自动软 NUMA

由于现代 NUMA 和 CPU 架构可以支持每个 CPU 超过八个物理核心，我们在 SQL Server 2016 中引入了一个名为**自动软 NUMA** 的特性，这样 SQL Server 可以在代码中对节点和 CPU 进行逻辑分区，以提供更好的可扩展性。

为了理解 SQL Server 如何检测 CPU 和 NUMA 节点（以及使用自动软 NUMA），让我们看一个例子。我在微软的实验室中有一台机器，拥有四个 NUMA 节点，每个节点一个 CPU 插槽，每个插槽 24 个核心（未启用超线程，因此总共 96 个 CPU）。

当我在该计算机上安装 SQL Server 时，`ERRORLOG` 文件通过以下条目检测 CPU 和核心，如图 6-2 所示。

**图 6-2.** SQL Server `ERRORLOG` 中检测到可用 CPU 的条目

如果 SQL Server 检测到每个 CPU 的物理核心数超过八个，则将启用自动软 NUMA。然后 SQL Server 将尝试对 NUMA 节点（在 SQL Server 内部是逻辑的）进行分区，以尽可能接近每个节点八个 CPU。图 6-3 显示了自动软 NUMA 如何在此计算机上跨四个 NUMA 节点在 SQL Server 内部逻辑地分区了这 96 个 CPU。

**图 6-3.** NUMA 和 CPU 映射的 `ERRORLOG` 条目

自动软 NUMA 默认启用，但你可以通过 `ALTER SERVER CONFIGURATION` 禁用它（也许你怀疑它导致了性能问题）。你可以在我们的文档 [`docs.microsoft.com/sql/t-sql/statements/alter-server-configuration-transact-sql`](https://docs.microsoft.com/sql/t-sql/statements/alter-server-configuration-transact-sql) 中阅读更多关于自动软 NUMA 的信息。

要了解如何读取 `ERRORLOG` 中关于“CPU 掩码”的映射，以查看哪些 CPU 映射到特定的 NUMA 节点，请参阅这篇我详细讨论自动软 NUMA 以及如何解读 CPU 映射的博客文章：[`blogs.msdn.microsoft.com/bobsql/2016/11/29/how-it-works-it-just-runs-faster-auto-soft-numa`](https://blogs.msdn.microsoft.com/bobsql/2016/11/29/how-it-works-it-just-runs-faster-auto-soft-numa)。

## 线程亲和性

由于 SQLOS 控制调度器和线程执行，因此可以将 SQL Server 工作线程分配给特定的 NUMA 节点和/或 CPU。此过程称为*亲和性*。我将在本书后面的章节中进一步讨论亲和性。

## 动态内存与缓存管理

SQL Server 旨在适应并最大化计算资源。内存可能是数据库性能最重要的资源之一。SQL Server 为数据库页（缓冲区）以及缓存的 T-SQL 查询和计划等重要资源提供了一个强大的内存管理系统。SQL Server 还管理其他内部需求的内存，并通过内存书记员系统进行跟踪。（我在第 5 章提到的一个动态管理视图是 `dm_os_memory_clerks`）。

SQL Server 内存管理的另一个惊人方面是*动态内存*。SQL Server 具有内置功能，可以根据内存需求增长或缩减其内存占用。你会发现，在安装 SQL Server 后而不创建任何



