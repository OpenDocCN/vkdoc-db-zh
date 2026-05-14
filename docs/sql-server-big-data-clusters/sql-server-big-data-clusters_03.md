# Spark

随着能够在`Linux`上运行`SQL Server`的能力出现，将`SQL Server`与各种开源及基于`Linux`的产品和技术集成开启了诸多新的可能性。其中最令人兴奋的组合之一，是在`SQL Server 大数据集群`中实现了对`Apache Spark`的集成。

`SQL Server`是一个关系型数据库平台。虽然像`PolyBase`这样的技术能够将非关系型数据（或来自另一个关系型平台如`Oracle`或`Teradata`的关系型数据）读入`SQL Server`所需的关系型格式中，但本质上，`SQL Server`过去并不擅长处理非结构化或非关系型数据。在这方面，`Spark`是一个改变游戏规则的存在。

在产品中集成`Spark`意味着，您现在可以根据自己的偏好，轻松地在`SQL Server 大数据集群`内部使用`Spark`或`SQL Server`来处理和分析海量的、各种类型的数据。这种处理大量数据的能力提供了最大的灵活性，并使数据集的并行和分布式处理成为现实。

## MapReduce

`Apache Spark`于 2009 年在加州大学伯克利分校创建，主要是为了解决一种名为`MapReduce`技术的局限性。`MapReduce`编程模型由`Google`开发，是用于为互联网上所有网页建立索引的基础技术（您可能更熟悉其`Hadoop MapReduce`形式）。简而言之，`MapReduce`是一个框架，用于在大量被称为节点的计算机上并行处理海量数据集。这种跨多个节点的并行处理至关重要，因为数据集的规模已经增长到无法再由单台机器高效处理的地步。通过将工作（和数据）分布到多台机器上，可以实现并行性，从而更快地处理这些大数据集。

在`MapReduce`框架上运行一个查询通常需要经历四个执行步骤：

1.  `输入分片`

    `MapReduce`任务的输入被分割成逻辑上的数据分布，这些数据存储在文件块中。`MapReduce`任务会计算哪些记录适合放入一个逻辑块或“分片”，并决定处理该任务所需的映射器数量。

2.  `映射`

    在映射阶段，我们的查询在每个“分片”上分别执行，并为该特定分片生成针对该特定查询的输出。输出始终是由映射过程返回的某种形式的键/值对。

3.  `混洗`

    `混洗`过程，简单来说，就是对映射过程返回的数据进行排序和合并的过程。

4.  `归约`

    最后一步，`归约`，汇总混洗过程返回的结果，并返回一个单一的输出。

解释`MapReduce`过程的最佳方式是查看一个可视化示例。图 2-4 展示了一个计算数据集中单词出现次数的`MapReduce`任务示例。为了简单且便于可视化展示，我们使用一个简短的句子作为数据集："`SQL Server` is no longer just `SQL` but it is much more."

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig4_HTML.jpg](img/480532_2_En_2_Fig4_HTML.jpg)

图 2-4
MapReduce 示例任务

在该示例中，包含我们任务输入的数据集（句子“`SQL Server` is no longer just `SQL` but is also much more”）被分割成三个不同的`分片`。这些`分片`在`映射`阶段被处理，得出每个分片的单词计数。结果被发送到`混洗`步骤，该步骤将所有结果按顺序排列。最后，`归约`步骤计算每个单词的总出现次数，并将其作为最终输出返回。

从图 2-4 中的（简单）示例可以看出，`MapReduce`在跨并行环境分发和协调数据处理方面非常高效。然而，`MapReduce`框架也有一些缺点，最显著的是编写需要多次遍历数据（例如，机器学习算法）的大型程序很困难。对于数据集的每一次遍历，都必须编写一个独立的`MapReduce`任务，每个任务都需要从头开始重新加载它所需的数据。由于这一点，以及`MapReduce`访问数据的方式，在`MapReduce`框架内处理数据可能相当缓慢。

## Spark：改进 MapReduce

创建`Spark`是为了解决这些问题，并使大数据分析更加灵活和高性能。它通过实现内存技术来实现这一点，该技术允许在处理步骤之间共享数据，并允许进行临时数据处理，而不必编写复杂的`MapReduce`任务来处理数据。此外，`Spark`支持广泛的库，这些库可以增强或扩展`Spark`的功能，例如处理流数据、执行机器学习任务，甚至通过`SQL`语言查询数据。

`Spark`在外观和行为上与`MapReduce`框架非常相似，因为`Spark`也是一个协调和管理处理数据的任务的协调器。与`MapReduce`一样，`Spark`使用工作节点来执行实际的数据处理。这些工作节点通过所谓的`Spark`应用程序（定义为`驱动器进程`）来获知要做什么。`驱动器进程`本质上是`Spark`应用程序的核心，它跟踪`Spark`应用程序的状态，响应输入或输出，并调度工作并将其分配给工作节点。`驱动器进程`的一个优点是，它可以通过语言 API（如`Python`或`R`）从不同的编程语言“驱动”。`Spark`负责将各种语言中的命令转换为在工作节点上处理的`Spark`代码。

图 2-5 展示了`逻辑上的``Spark`架构概览。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig5_HTML.jpg](img/480532_2_En_2_Fig5_HTML.jpg)

图 2-5
Spark 逻辑架构

我们特意在提及`Spark`架构时使用了“`逻辑上`”这个词是有原因的。尽管图 2-5 暗示`工作节点`是集群中独立的机器，但实际上在`Spark`中，您可以在一台机器上运行任意多的`工作节点`。事实上，出于测试和开发任务的考虑，`驱动器进程`和`工作节点`都可以在单台机器上以`本地模式`运行。

图 2-5 还展示了`Spark`应用程序如何在集群上协调工作。您作为用户编写的代码由`驱动器进程`翻译成您的`工作节点`能够理解的语言；它将工作分发给各个`工作节点`，由它们处理数据。在图中，我们特别突出了`工作节点`内部的`缓存`。`缓存`是`Spark`能够如此高效地执行数据处理的原因之一，因为它可以将中间处理结果存储在节点的内存中，而不是像`Hadoop MapReduce`那样存储在磁盘上。

## SQL Server 大数据集群中的 Spark

在`SQL Server 大数据集群`内部，`Spark`被包含在一个独立的容器中，该容器与一个`SQL Server on Linux`容器共享一个 Pod。

我们尚未触及的一个方面是，`SQL Server`外部的非关系型数据在`大数据集群`内部是如何存储的。如果您熟悉基于`Spark`或`Hadoop`的大数据基础架构，下一节内容应该不会让您感到意外。



### HDFS

HDFS，即 Hadoop 分布式文件系统，是 Spark 架构中存储数据的主要方法。HDFS 在如何存储和处理文件系统上的数据方面具有许多优势，例如容错能力以及将数据分布在组成 HDFS 集群的多个节点上。

HDFS 的工作方式是将数据拆分成独立的块（称为数据块），并在数据存储到文件系统时，将这些块分布到组成 HDFS 环境的各个节点上。这些数据块（默认大小为 64 MB）随后会在多个节点上进行复制，以实现容错。如果一个节点发生故障，数据块的副本在其他节点上仍然可用，这意味着文件系统可以轻松地从单个节点的数据丢失中恢复。

图 2-6 展示了 HDFS 架构的简化概览。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig6_HTML.jpg](img/480532_2_En_2_Fig6_HTML.jpg)

图 2-6

HDFS 架构

在许多方面，HDFS 映射了 Hadoop 的架构，从这个意义上说，也映射了我们在上一节中展示的 Spark 架构。由于文件系统中存储的数据具有分布式特性，数据分布在也负责处理该数据的 Spark 架构节点上是可能的，实际上也是预期的。这种数据分布带来了巨大的性能优势；由于数据驻留在负责处理该数据的同一节点上，因此无需在存储架构中移动数据。再加上在 Spark 工作节点内进行数据缓存的额外好处，数据确实可以非常高效地处理。

需要指出的一点是，与 Hadoop 不同，Spark 并不一定局限于驻留在 HDFS 中的数据。Spark 可以通过 API 和原生支持访问存储在各种来源的数据。例子包括各种云存储平台，如 Azure Blob Storage，或关系型数据源，如 `SQL Server`。

### 将物理基础设施部分整合起来

现在是时候看看全局了。图 2-7 完整地展示了前几节讨论的技术如何在 `SQL Server Big Data Clusters` 内协同工作。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig7_HTML.jpg](img/480532_2_En_2_Fig7_HTML.jpg)

图 2-7

包含 Spark、HDFS 和 Kubernetes 的 `SQL Server Big Data Cluster` 架构

从图 2-7 可以看出，`SQL Server Big Data Clusters` 在 Kubernetes 部署的容器内整合了多种不同的角色。将 `SQL Server` 和 Spark 与 HDFS 数据节点一起放在同一个容器中，允许这两种产品以性能优化的方式访问存储在 HDFS 中的数据。在 `SQL Server` 中，这种数据访问将通过 `PolyBase` 发生，而 Spark 则可以原生查询驻留在 HDFS 中的数据。

图 2-7 中的架构还为我们提供了处理和查询数据的两条不同路径。我们可以选择使用可用的 `SQL Server` 实例将数据以关系格式存储，或者我们可以使用 HDFS 文件系统并在其中存储（非关系）数据。当数据存储在 HDFS 中时，我们可以以任何我们喜欢的方式访问和处理该数据。如果你的用户更熟悉编写 `T-SQL` 查询来检索数据，你可以使用 `PolyBase` 将 HDFS 存储的数据通过外部表引入 `SQL Server`。另一方面，如果用户更喜欢使用 Spark，他们可以编写直接从 HDFS 访问数据的 Spark 应用程序。然后，如果需要，用户可以调用 Spark API 来将存储在 `SQL Server` 中的关系数据与存储在 HDFS 中的非关系数据结合起来。

## 逻辑大数据集群架构

如本章引言所述，大数据集群可以分为四个逻辑区域。将这些区域视为各种基础设施和管理部分的集合，它们在集群内执行特定功能。每个区域又包含一个或多个它所执行的角色。例如，在数据池区域中，有两个角色：存储池和 SQL 数据池。

图 2-8 展示了四个逻辑区域以及作为每个区域一部分的各种角色的概览。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig8_HTML.jpg](img/480532_2_En_2_Fig8_HTML.jpg)

图 2-8

大数据集群架构

你可以立即推断出这四个逻辑区域：控制区域（内部命名为控制平面）以及计算、数据和应用程序区域。在接下来的章节中，我们将分别深入探讨这些逻辑区域中的每一个，描述其功能以及其中执行的角色。在我们开始仔细研究控制平面之前，你可能已经注意到图 2-8 中还显示了一个额外的角色：`SQL Server Master Instance`。

`SQL Server master instance` 是 Kubernetes 节点内的一个 `SQL Server on Linux` 部署。`SQL Server master instance` 就像通往你的大数据集群的入口点，并通过 `Azure Data Studio` (`ADS`)（见图 2-9）或其他工具（如 `SQL Server Management Studio`）提供外部连接端点。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig9_HTML.jpg](img/480532_2_En_2_Fig9_HTML.jpg)

图 2-9

通过 `Azure Data Studio` 连接到 `SQL Server master instance`

在许多方面，`SQL Server master instance` 的行为就像一个普通的 `SQL Server` 实例。你可以使用 `Azure Data Studio` 访问它并浏览该实例，查询存储在其中的系统数据库和用户数据库。与传统的 `SQL Server` 实例相比，一个重大变化是 `SQL Server master instance` 将把你的查询分发到计算池中的所有 `SQL Server` 节点，并通过 `PolyBase` 访问存储在数据平面内 HDFS 上的数据。

默认情况下，`SQL Server master instance` 还启用了 `Machine Learning Services`。这使你可以直接从查询中运行使用 `R`、`Python` 或 `Java` 的数据库内分析。使用 `SQL Server Big Data Cluster` 提供的数据虚拟化选项，`Machine Learning Services` 也可以访问存储在 HDFS 文件系统内的非关系数据。这意味着你的数据分析师或科学家可以选择使用 Spark 或 `SQL Server Machine Learning Services` 来分析或操作化存储在大数据集群中的数据。我们将在第 7 章中更详细地探讨这些选项。



### 控制平面

图 2-10 所示的控制平面是您进入大数据集群管理环境的入口。它提供了如 Grafana 等各种管理和日志工具，是您执行所有大数据集群管理的集中地。此外，大数据集群内部的安全性也是通过控制平面来管理和控制的。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig10_HTML.jpg](img/480532_2_En_2_Fig10_HTML.jpg)

图 2-10 大数据集群控制平面

关于大数据集群的管理，我们将在第 3 章讨论可用于管理大数据集群的各种管理工具。

除了提供一个可执行所有大数据集群管理任务的集中位置外，控制平面在协调向底层计算和数据区域分发任务方面也扮演着非常重要的角色。访问控制平面是通过控制器端点进行的。

控制器端点用于大数据集群的部署和配置管理。该端点通过 REST API 进行访问，大数据集群内部的一些服务以及我们用于部署和配置大数据集群的命令行工具都会访问这些 API。

在下一章中，您将非常熟悉控制器端点，我们将使用 `azdata` 来部署和配置一个大数据集群。

### 计算区域

计算区域（见图 2-11）由一个或多个计算池组成。一个计算池是一组 Kubernetes Pod 的集合，这些 Pod 包含运行在 Linux 上的 SQL Server。使用计算池，您可以通过 PolyBase 以分布式方式访问各种数据源。例如，计算池可以访问大数据集群本身 HDFS 内部存储的数据，或者通过任何 PolyBase 连接器（如 Oracle 或 MongoDB）访问数据。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig11_HTML.jpg](img/480532_2_En_2_Fig11_HTML.jpg)

图 2-11 大数据集群计算区域

拥有计算池的主要优势在于，它提供了将查询分布式或横向扩展到每个计算池内部多个节点的选项，从而提升了 PolyBase 查询的性能。

默认情况下，在计算逻辑区域内您可以访问一个计算池。但是，在某些情况下，例如当您希望将资源专用于访问特定数据源时，您可以添加多个计算池。计算池内部每个 Kubernetes Pod 的所有管理和配置都由 SQL Server 主实例处理。

### 数据区域

数据区域（图 2-12）用于在您的大数据集群中持久化和缓存数据，它分为两种不同的角色：存储池和 SQL 数据池，它们在大数据集群中具有不同的功能。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig12_HTML.jpg](img/480532_2_En_2_Fig12_HTML.jpg)

图 2-12 数据平面架构

#### 存储池

存储池由结合了 Spark、运行在 Linux 上的 SQL Server 和 HDFS 数据节点的 Kubernetes Pod 组成。图 2-13 展示了 Pod 的内容。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig13_HTML.jpg](img/480532_2_En_2_Fig13_HTML.jpg)

图 2-13 存储池内部的存储节点

HDFS 数据节点被组合成一个存在于您大数据集群内部的单一 HDFS 集群。存储池的主要功能是提供一个 HDFS 存储集群，用于存储通过例如 Spark 导入的数据。通过创建一个 HDFS 集群，您基本上可以在大数据集群内部访问一个数据湖，用于存储各种非关系型数据，如 Parquet 或 CSV 文件。

HDFS 集群会自动安排数据持久化，因为您导入存储池的数据会自动分散到存储池内部的所有存储节点上。这种跨节点的数据分布也使您能够快速分析大量数据，因为分析的负载分布在多个节点上。这种架构的一个优势是，您可以使用数据节点上的本地存储，也可以向节点添加自己的持久存储子系统。

与计算池类似，存储节点中存在的 SQL Server 实例是通过 SQL Server 主实例访问的。因为存储节点结合了 SQL Server 和 Spark，所以驻留在存储节点上或由存储节点管理的所有数据也可以直接通过 Spark 访问。这意味着您不必使用 PolyBase 来访问 HDFS 环境内部的数据。这为数据分析或数据工程提供了更大的灵活性。

#### SQL 数据池

数据平面的另一个区域是 SQL 数据池。这组 Pod 与存储池的不同之处在于，它没有将 Spark 或 HDFS 一起组合到节点中。相反，SQL 数据池由一个或多个运行在 Linux 上的 SQL Server 实例组成。这些实例被称为分片，您可以在图 2-14 中看到它们的示意图。

![../images/480532_2_En_2_Chapter/480532_2_En_2_Fig14_HTML.jpg](img/480532_2_En_2_Fig14_HTML.jpg)

图 2-14 SQL 数据池内部对外部数据源的横向扩展和缓存

SQL 数据池的主要角色是优化使用 PolyBase 对外部源的访问。SQL 数据池随后可以对外部源的数据进行分区和缓存，并最终提供针对外部数据源的并行和分布式查询。为了提供这种并行和分布式功能，SQL 数据池内部的数据集被划分为跨 SQL 数据池内部节点的分片。

## 总结

在本章中，我们从物理和逻辑两个角度探讨了 SQL Server 大数据集群的架构。在物理架构中，我们重点关注了构成大数据集群的技术，如容器、SQL-on-Linux 和 Spark。在逻辑架构中，我们讨论了大数据集群内部执行特定角色或任务的不同逻辑区域。

对于大数据集群中使用的每种技术，我们都简要介绍了其起源以及该技术在大数据集群中扮演的角色。由于 SQL Server 大数据集群中使用了广泛的技术和解决方案，我们尽可能全面地描述了各种技术；然而，我们也意识到，我们无法像希望的那样详细描述每一种技术。例如，仅 Spark 一项，就已经出版发行了数十本描述其工作原理以及如何利用该技术的书籍。在 SQL-on-Linux、HDFS 和 Kubernetes 领域，情况也并无太大不同。因此，最好将本章视为对这些技术或解决方案的简单而简要的介绍，足以让您开始理解和使用 SQL Server 大数据集群。



## 3. 大数据集群的部署

现在是时候安装属于您自己的 `SQL Server 2019 大数据集群` 了！我们将详细处理三种不同的场景，并且会为每个场景使用一台全新的机器：

*   在 `Windows` 上独立安装 `PolyBase`
*   在 `Linux` 上使用 `kubeadm` 部署大数据集群
*   使用 `Azure Kubernetes Service (AKS)` 部署大数据集群

从同一台机器运行所有选项是完全可以的。但考虑到您可能不会用到全部方案，我们认为每次都从头开始会更合理。

我们将介绍使用 `Microsoft Windows` 操作系统的安装过程。本指南的目标是让您的 `大数据集群` 尽快启动并运行，因此我们会配置一些可能并非最佳实践的选项（比如将所有服务账户和目录都保留为默认设置）。请根据您的需求随意修改这些设置。

如果您选择 `AKS` 安装，则需要一个有效的 `Azure` 订阅。如果您还没有 `Azure` 订阅，可以免费创建一个，其中包含可免费使用的额度。

## 一个小助手：`Chocolatey`

在开始之前，我们想请您关注 `Chocolatey`（或称 `choco`）。如果您没听说过它，`choco` 是一个适用于 `Windows` 的免费软件包管理器，它允许我们通过在 `PowerShell` 或 `命令提示符` 中执行一行命令来安装许多先决条件。您可以在 [`http://chocolatey.org`](http://chocolatey.org) 找到更多信息（参见图 3-1），您甚至可以在那里创建账户并提供自己的软件包。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig1_HTML.jpg](img/480532_2_En_3_Fig1_HTML.jpg)
图 3-1
`Chocolatey` 主页

不过，从一个简单用户的角度来看，无需创建账户或下载任何安装程序。

要在您的系统上启用 `choco`，请以管理员模式打开一个 `PowerShell` 窗口，并运行清单 3-1 中所示的脚本。

```
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```
清单 3-1
在 `PowerShell` 中安装 `Chocolatey` 的脚本

相应的命令执行完成后，`choco` 即安装完毕并可以使用了。

## 安装本地 `PolyBase` 实例

如果您只对 `SQL Server 2019 大数据集群` 的数据虚拟化功能感兴趣，那么安装过程实际上比完整环境要简单和轻量得多。实现数据虚拟化功能的 `PolyBase` 特性，可以在任何平台的常规 `SQL Server 2019` 安装过程中安装。

如果您想通过 `PolyBase` 使用 `Teradata`，则需要安装 `C++ Redistributable 2012` 才能与我们的 `SQL Server` 实例进行实际通信。在任何情况下，安装 `SQL Server Management Studio (SSMS)` 都可能会有所帮助，并且安装好它以便重现我们在本书中展示的示例是件好事。

让我们通过 `Chocolatey` 安装前面提到的软件包。只需运行清单 3-2 中的三个命令，`choco` 就会处理其余的事情。

```
choco install sql-server-management-studio -y
choco install vcredist2012 -y
```
清单 3-2
安装 `PolyBase` 先决条件的脚本

安装好先决条件后，我们就可以开始实际的 `SQL Server` 安装了。导航到 [`www.microsoft.com/en-us/evalcenter/evaluate-sql-server-2019`](http://www.microsoft.com/en-us/evalcenter/evaluate-sql-server-2019) 并按照说明进行下载。

运行下载的文件，如图 3-2 所示。

选择“下载媒体”作为安装类型。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig2_HTML.jpg](img/480532_2_En_3_Fig2_HTML.jpg)
图 3-2
`SQL Server 2019` 安装程序 - 安装类型选择

然后确认语言和目录，如图 3-3 所示。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig3_HTML.jpg](img/480532_2_En_3_Fig3_HTML.jpg)
图 3-3
`SQL Server 2019` 安装程序 - 下载媒体对话框

下载完成并成功后，您将看到图 3-4 中的消息。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig4_HTML.jpg](img/480532_2_En_3_Fig4_HTML.jpg)
图 3-4
`SQL Server 2019` 安装程序 - 下载媒体成功

现在导航到您保存下载文件的文件夹。挂载映像，如图 3-5 所示。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig5_HTML.jpg](img/480532_2_En_3_Fig5_HTML.jpg)
图 3-5
`SQL Server 2019` 安装程序 - 挂载 ISO

安装可以无人值守运行，但对于第一次安装，可能更有意义的是探索您的选项。运行 `setup.exe`，转到“安装”选项卡，然后选择“新建 `SQL Server` 独立安装”，如图 3-6 所示。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig6_HTML.jpg](img/480532_2_En_3_Fig6_HTML.jpg)
图 3-6
`SQL Server 2019` 安装程序 - 主屏幕

选择评估版，如图 3-7 所示，确认许可条款，并在后续屏幕上勾选“检查更新”复选框。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig7_HTML.jpg](img/480532_2_En_3_Fig7_HTML.jpg)
图 3-7
`SQL Server 2019` 安装程序 - 版本选择

安装程序规则会识别在运行安装程序时可能出现的潜在问题。如图 3-8 所示的失败和警告必须在安装完成前得到纠正。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig8_HTML.jpg](img/480532_2_En_3_Fig8_HTML.jpg)
图 3-8
`SQL Server 2019` 安装程序 - 安装规则

从图 3-9 所示的功能选择对话框中，勾选“`PolyBase 查询对外部数据`”。同时勾选其子节点“用于 HDFS 数据源的 Java 连接器”。


![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig9_HTML.jpg](img/480532_2_En_3_Fig9_HTML.jpg)

图 3-9

`SQL Server 2019 安装程序` – 功能选择

使用“实例配置”为 `SQL Server` 实例指定名称和实例 ID。如图 3-10 所示，实例 ID 成为安装路径的一部分。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig10_HTML.jpg](img/480532_2_En_3_Fig10_HTML.jpg)

图 3-10

`SQL Server 2019 安装程序` – 实例配置

在图 3-11 所示的对话框中，选择配置一个启用 `PolyBase` 的独立实例。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig11_HTML.jpg](img/480532_2_En_3_Fig11_HTML.jpg)

图 3-11

`SQL Server 2019 安装程序` – `PolyBase` 配置

如图 3-12 所示，`PolyBase` HDFS 连接器需要 Java；系统将提示您要么随 `SQL Server` 安装 `Open JRE`，要么提供您计算机上现有 Java 安装的位置（如果有的话）。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig12_HTML.jpg](img/480532_2_En_3_Fig12_HTML.jpg)

图 3-12

`SQL Server 2019 安装程序` – Java 安装位置

然后确认如图 3-13 所示的默认账户。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig13_HTML.jpg](img/480532_2_En_3_Fig13_HTML.jpg)

图 3-13

`SQL Server 2019 安装程序` – 服务器配置

坚持使用 Windows 身份验证，并如图 3-14 所示添加您当前的用户。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig14_HTML.jpg](img/480532_2_En_3_Fig14_HTML.jpg)

图 3-14

`SQL Server 2019 安装程序` – 数据库引擎配置

单击图 3-15 所示摘要页面上的 `安装`，并等待安装程序完成。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig15_HTML.jpg](img/480532_2_En_3_Fig15_HTML.jpg)

图 3-15

`SQL Server 2019 安装程序` – 概述

安装成功完成后，将显示如图 3-16 所示的状态摘要，然后您可以关闭安装程序。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig16_HTML.jpg](img/480532_2_En_3_Fig16_HTML.jpg)

图 3-16

`SQL Server 2019 安装程序` – 完成

使用 `SQL Server Management Studio (SSMS)` 或任何其他 `SQL Server` 客户端工具（如 `Azure Data Studio`）连接到实例，打开一个新查询，并运行清单 3-3 中所示的脚本。

```
exec sp_configure @configname = 'polybase enabled', @configvalue = 1;
RECONFIGURE
```

清单 3-3
通过 `T-SQL` 启用 `PolyBase`

输出应为

```
Configuration option 'polybase enabled' changed from 0 to 1. Run the RECONFIGURE statement to install.
```

如图 3-17 所示，单击 `对象资源管理器` 菜单中的 `重新启动`，以重新启动 `SQL Server` 实例。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig17_HTML.jpg](img/480532_2_En_3_Fig17_HTML.jpg)

图 3-17

重新启动 `SQL Server` 实例

您已大功告成——现在您可以访问启用 `PolyBase` 的 `SQL Server 2019` 安装了。

## 使用 `Azure Data Studio` 处理大数据群集

作为微软 `SQL 客户端工具` 战略的一部分，处理大数据群集所需的大部分任务通过 `Azure Data Studio (ADS)` 而非 `SQL Server Management Studio` 或其他工具完成，这可能并不令您感到惊讶。对于那些不熟悉此工具的读者，我们将从简要介绍开始，包括如何获取此工具。

### 什么是 `Azure Data Studio`？

`Azure Data Studio` 是一个跨平台（Windows、MacOS 和 Linux）、可扩展且可定制的工具，可用于经典的 `T-SQL` 查询和命令，以及诸如笔记本等多种新功能。后者可以通过扩展来启用，这些扩展通常通过 `VSIX` 文件安装，您在处理 `Visual Studio` 或 `Visual Studio Code`^(²) 的其他扩展时可能对此很熟悉。

它最初于 2017 年以 `SQL Operations Studio` 的名称公开，但在 2018 年正式发布前进行了品牌重塑。虽然产品名称有点误导性，但它不仅适用于云（Azure）服务，也适用于本地解决方案和需求。

例如，它没有为 `SQL Server Agent` 提供开箱即用的界面，但作为交换提供了内置的图表功能，这表明它与其说是替代品，不如说是对 `SQL Server Management Studio (SSMS)` 的补充工具。`SSMS` 针对的是管理和运维群体（数据库管理员），而 `Azure Data Studio` 更适合各种数据专业人员，包括数据科学家。

### 获取并安装 `Azure Data Studio`

您可以直接从微软获取免费的 `Azure Data Studio` 副本： [`https://docs.microsoft.com/en-us/sql/azure-data-studio/download`](https://docs.microsoft.com/en-us/sql/azure-data-studio/download)。为您选择的平台下载安装程序并运行。

或者，只需运行此 `Chocolatey` 命令（清单 3-4），它将为您安装最新版本。

```
choco install azure-data-studio -y
```

清单 3-4
通过 `choco` 安装 `ADS`

## 安装“真正的”大数据群集

如果您想充分利用完整的大数据群集功能集，则需要一个包含所有不同角色和池的完整安装。

### 在 Linux 上使用 kubeadm

一个非常简单的大数据集群部署方法是在一个全新安装的 Ubuntu 16.04 或 18.04 虚拟机或物理机上使用 kubeadm。

微软提供了一个能完成所有工作的脚本，因此除了 Linux 安装本身，您几乎不需要做太多事情，这可能是让您第一个大数据集群启动并运行的最简单方法。

首先，运行列表 3-5 中的命令，确保您的 Linux 机器是最新的。

```
sudo apt update&&apt upgrade -y
sudo systemctl reboot
列表 3-5
为 Ubuntu 打补丁
```

然后，下载脚本，使其可执行，并以 root 权限运行它，如列表 3-6 所示。

```
curl --output setup-bdc.sh https://raw.githubusercontent.com/microsoft/sql-server-samples/master/samples/features/sql-big-data-cluster/deployment/kubeadm/ubuntu-single-node-vm/setup-bdc.sh
chmod +x setup-bdc.sh
sudo ./setup-bdc.sh
列表 3-6
下载并执行部署脚本
```

如图 3-18 所示，脚本会要求您输入密码，然后自动开始准备步骤。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig18_HTML.jpg](img/480532_2_En_3_Fig18_HTML.jpg)
图 3-18
在 Linux 上使用 kubeadm 进行部署

在预取镜像、配置 Kubernetes 以及所有其他必需步骤之后，大数据集群的部署将开始，如图 3-19 所示。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig19_HTML.jpg](img/480532_2_En_3_Fig19_HTML.jpg)
图 3-19
在 Linux 上使用 kubeadm 进行部署

一旦整个脚本执行完毕，您就完成了！如图 3-20 所示，脚本还将提供在部署过程中创建的所有端点。

![../images/480532_2_En_3_Chapter/480532_2_En_3_Fig20_HTML.jpg](img/480532_2_En_3_Fig20_HTML.jpg)
图 3-20
在 Linux 上使用 kubeadm 成功部署

您的集群现已完全部署并准备就绪！

### Azure Kubernetes Service (AKS)

另一个直接部署集群的方法是使用 Azure Kubernetes Service，其中 Kubernetes 集群在 Microsoft Azure 云中设置并提供。虽然部署是从任何机器（您的本地 PC 或 VM）启动和控制的，但集群本身将在 Azure 中运行，因此这意味着部署需要一个 Azure 帐户，并且会在您的 Azure 订阅上产生费用。

您可以通过 Azure Data Studio 中的向导或通过一个名为 `azdata` 的工具（在 Linux 上部署前一个集群的脚本也调用了它）手动部署。两种方法都需要先安装一些先决条件。一个完整的安装实际上需要几个工具和助手：

*   Python
*   Curl 和 SQL Server 命令行工具，这样我们才能与集群通信并向其上传数据。
*   Kubernetes CLI
*   `azdata`，这将创建、维护和删除大数据集群。
*   Notepad++ 和 7Zip，这些不是实际要求，但如果您想调试安装，可能会得到一个包含可能很大的文本文件的 tar.gz 文件。Windows 默认不处理这些。

列表 3-7 中的脚本会将这些安装到您的本地机器（或您运行脚本的任何机器）上，因为部署是从这里控制和触发的。我们将通过 Chocolatey 安装这些先决条件。

```
choco install python3 -y
choco install sqlserver-cmdlineutils -y
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
python -m pip install --upgrade pip
python -m pip install requests
python -m pip install requests --upgrade
choco install curl -y
choco install kubernetes-cli -y
choco install notepadplusplus -y
choco install 7zip -y
choco install visualcpp-build-tools -y
pip3 install kubernetes
pip3 install -r https://aka.ms/azdata
列表 3-7
大数据集群先决条件的安装脚本
```

虽然相关供应商显然为大多数工具提供了可视化/手动安装程序，但脚本化方法使整个体验变得容易得多。

此外，由于我们想使用脚本部署到 Azure，我们需要列表 3-8 中所示的 `azure-cli` 包，以便能够连接到我们的 Azure 订阅。

```
choco install azure-cli -y
列表 3-8
安装 azure-cli
```

虽然您技术上可以在 Azure 门户中或通过手动 PowerShell 脚本准备所有内容（我们需要一个资源组、Kubernetes 集群等），但有一个更简单的方法：从 [`https://github.com/Microsoft/sql-server-samples/tree/master/samples/features/sql-big-data-cluster/deployment/aks`](https://github.com/Microsoft/sql-server-samples/tree/master/samples/features/sql-big-data-cluster/deployment/aks) 获取 Python 脚本，它会自动为您处理整个过程和设置。

将脚本下载到您的桌面或其他合适的位置，并打开命令提示符。导航到保存脚本的文件夹。您也可以如列表 3-9 所示使用命令提示符下载。

```
curl --output deploy-sql-big-data-aks.py https://raw.githubusercontent.com/microsoft/sql-server-samples/master/samples/features/sql-big-data-cluster/deployment/aks/deploy-sql-big-data-aks.py
列表 3-9
下载部署脚本
```

当然，您也可以根据需要修改和审查脚本，例如，将一些参数（如 VM 大小）设为静态而非变量，或更改某些值的默认值。

首先，我们需要登录 Azure，这将使用列表 3-10 中所示的命令完成。

```
az login
列表 3-10
从命令提示符触发登录到 azure
```

