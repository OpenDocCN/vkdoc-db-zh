# 服务层级与配置

通用型和业务关键型层级目前支持的数据库最大尺寸为 4-8TB（具体取决于部署选项和最终服务层级）。Azure SQL Database 提供的最终服务层级是 **超大规模**，它旨在利用云的弹性，让数据库几乎可以无限制地增长，从而让开发者摆脱管理海量数据库的所有负担。微软对 Azure SQL 进行了架构重构，使其能够以 `Hyperscale` 方式运行。方法是将原始的 SQL Server 引擎重构为不同、更小、独立、分布式的服务，这些服务作为一个实体协同工作，但比原始架构更具弹性和可扩展性。该服务层级适用于大多数未来可能需要高度可扩展存储（目前最高可达 100TB）的业务工作负载。`Hyperscale` 还支持为读取密集型工作负载部署多个只读副本，所有副本均提供数据服务，利用横向扩展策略来提高并发性和性能。由于文件快照存储在 Azure Blob 存储中，`Hyperscale` 支持“近乎即时”的数据库备份和数分钟内完成的恢复，因为它不再是一个受数据量大小限制的操作。正如您可能猜到的，这一特性在管理超大型数据库时至关重要。由于更高的日志吞吐量和更快的事务提交时间（无论数据量大小），`Hyperscale` 能提供比通用型层级更高的整体性能。如果您对微软如何构建此架构以及它带来的架构变革感兴趣，可以在此处查看详细信息：[`https://aka.ms/sdsth`](https://aka.ms/sdsth)。

总而言之，微软的建议是从通用型层级开始，并根据需要进行调整。利用云的弹性，您可以根据需要向上或向下扩展资源。如果性能不够理想，您可以切换到业务关键型或 `Hyperscale`。请记住，`Hyperscale` 的性能更高，而业务关键型的性能最高，成本通常也遵循相同的模式。有关各服务层级当前的详细比较，请参考：[`https://aka.ms/sdbsth`](https://aka.ms/sdbsth)。

## 硬件代次

一旦选择了服务层级，根据部署选项，vCore 模型会为 **硬件代次** 提供多种选择。硬件代次的选择允许您选择用于计算和内存的物理硬件。当今最常用的硬件代次是 `Gen4` 和 `Gen5`。在不过多涉及规格参数的情况下，`Gen4` 硬件允许每个 vCore 配备显著更多的内存，而 `Gen5` 硬件则允许您将计算资源扩展到更高的水平。需要重点注意的是，随着技术的进步，可用的硬件代次将不断变化和发展。例如，`Gen4` 已推出很长时间，在某些地区已不再支持新数据库。此外，`Fsv2 系列`（计算优化型）和 `M 系列`（内存优化型）硬件选项最近已公开可用。另一个例子是，Azure SQL Database 无服务器版目前仅在 `Gen5` 硬件上可用。您的成本和性能需求可能会使某一代硬件成为更好的选择。您可以在此处查看最新的硬件选择和规格：[`https://aka.ms/sdstvhg`](https://aka.ms/sdstvhg)。

如果您正在寻找其他计算成本节约的机会，一个值得考虑的选项是使用 Azure SQL Database 预留容量 (`RI`) 预付计算资源以获得折扣。更多信息，请参阅：[`https://aka.ms/sdri`](https://aka.ms/sdri)。

如果您正在构建一个新应用，并从空的数据库或实例开始，Azure SQL 的弹性允许您根据需要增长和扩展计算与存储。如果您正巧要将现有工作负载迁移到 Azure SQL Database 或托管实例，将该工作负载转化为服务层级、计算级别和最大数据大小可能会很复杂。幸运的是，微软构建了一些可以提供帮助的工具。首先，您可以利用 **数据迁移助手** (`DMA`) 工具来识别任何阻碍因素并确定最佳的部署选项。具体到部署选择，`DMA` 工具还具有一项名为 **SKU 推荐器** 的功能，它会分析您现有的工作负载，并推荐服务层级、计算级别和最大数据大小。该工具会提供每月的估算成本，并在分析后创建一个可用于批量预配的 PowerShell 脚本。

## 网络

选择了计算和存储的起始点后，下一步是配置网络连接。Azure SQL Database（单数据库和弹性池）和 Azure SQL 托管实例（单实例和实例池）的网络连接选项是不同的。部署 Azure SQL Database 时，门户中的默认设置是“允许 Azure 服务和资源访问此服务器”选项设置为“是”，这意味着如果您建立连接，其他 Azure 服务（例如 Azure 数据工厂或 Azure 虚拟机）可以访问该数据库。此外，部署后，您可以更改 Azure SQL Database 防火墙规则以满足应用程序的要求。

对于 Azure SQL 托管实例，您将服务部署在 Azure **虚拟网络** (`VNet`) 和专用于托管实例的子网中。`VNet` 类似于您部署在本地环境中的网络，但特定于 Azure。子网只是在 `VNet` 内创建网段的一种方式。通过将 Azure SQL 托管实例部署到您拥有的 `VNet` 中的一个仅限托管实例的子网，您可以获得一个完全安全且私有的 IP 地址。此功能允许您将本地网络或其他本地数据存储连接到 Azure SQL 托管实例（例如，支持使用链接服务器）。您可以选择启用公共终结点，以便无需 VPN 即可从 Internet 连接到托管实例，但默认情况下此访问是禁用的。

Azure SQL 托管实例中通过 `VNet` 隔离实现私有终结点的原则，已通过 Azure 专用链接的形式应用到 Azure SQL Database 中：[`https://aka.ms/aplo`](https://aka.ms/aplo)。

## 连接方法

在部署过程中，对于 Azure SQL 托管实例，您还可以选择连接类型。对于 Azure SQL Database，您也可以选择连接类型，但只能在部署之后进行。您可以保留默认设置（`代理` 和 `重定向` 的组合），或更改为仅使用 `重定向` 或仅使用 `代理`。概括地说，在 `代理` 模式下，所有连接都通过 Azure SQL 网关进行代理；而在 `重定向` 模式下，利用网关建立连接后，连接将直接指向数据库或托管实例。直接连接 (`重定向`) 可以减少延迟并提高吞吐量，但也需要为 Azure 外部的服务打开额外的端口，这就是为什么它并非立即完全启用的原因；打开额外端口需要根据该决策带来的安全风险进行评估，因此 Azure 默认采用最安全的选项。

如前面章节所讨论的，在部署数据库（Azure SQL Database 的数据库或 Azure SQL 托管实例中的数据库）时，您可以选择使用来自另一个 Azure SQL Database 或 Azure SQL 托管实例中受管数据库的现有备份来设置 **数据源**（它们必须匹配；不能使用受管数据库来创建新的 Azure SQL Database）。仅对于 Azure SQL Database，您还可以选择使用现有的示例数据库 (`AdventureWorksLT`)。



## 排序规则

部署 Azure SQL 时，另一个应仔细考虑的选择是`collation`（排序规则）。Azure SQL 中的排序规则为数据库中的数据定义了排序规则、区分大小写和区分重音等属性。排序规则的设置会影响实例或数据库中许多操作的特性，因此应谨慎设置。在 Azure SQL 托管实例中，你可以在创建实例时设置服务器排序规则，但之后无法更改。这为该实例中的所有数据库设置了默认排序规则，但之后你可以在数据库和列级别修改排序规则。在 Azure SQL 数据库中，你无法设置服务器排序规则；它被固定为“`SQL_Latin1_General_CP1_CI_AS`”，这是最常见的排序规则。每个排序规则的名称格式类似：“`SQL`”表示它是 SQL Server 排序规则（而非 Windows 或二进制排序规则），“`Latin1_General`”指定了排序时使用的字母表和语言，“`CP1`”引用了排序规则使用的代码页，“`CI`”表示应不区分大小写，“`AS`”表示应区分重音，并且还有其他与宽度、UTF-8 等相关的选项可用 ([`https://aka.ms/srdc`](https://aka.ms/srdc))。

### 时区

`Time zone`（时区）设置，与排序规则设置一样，看似一个很小的决定，却可能对你的应用程序产生重大影响。微软的建议是为所有实例和数据库使用协调世界时（UTC）。在 Azure SQL 数据库中，你无法更改时区；你只能使用 UTC（不过别担心，任何 Azure SQL 数据库中都有丰富的内置功能来操作时区并处理转换为本地时间）。然而，对于 Azure SQL 托管实例，在部署时引入时区选择是为了满足许多客户的需求，这些客户的现有应用程序在特定时区的上下文中存储和调用日期时间值。

### 高级数据安全

从 Azure 门户部署（但也可以通过 PowerShell 完成）的最后一个可用选项是`Advanced data security`（高级数据安全，ADS）。在门户中，系统会提示你是否要开始免费试用并启用 ADS，它提供了与数据发现和分类、漏洞警报和威胁检测相关的功能。对于 Azure SQL 托管实例，可以在部署后配置 ADS。你可以在此处了解有关 ADS 的更多信息：[`https://aka.ms/sdads`](https://aka.ms/sdads)。

如你所见，有很多选项可供你选择，以使 Azure SQL 以高效且经济高效的方式满足你的需求。在本书中，将进一步探讨不同选项中可用的功能，让你对当前任何场景选择最佳选项有更深入的了解。

## 使用示例数据库

无论你是刚开始接触 Azure SQL 和 T-SQL，还是经验丰富的专业人士，总会有新的东西可以探索。探索一些新的或细微的功能和场景需要一个数据库，利用微软的一些示例数据库可以带来极大的帮助。此外，Azure SQL 和 SQL Server 社区非常庞大（只需在 Twitter 上搜索 #sqlfamily），社区成员不断贡献新内容和示例来探索最新功能或解释复杂主题。通常，你在网上或书中找到的示例都会与微软的某个示例数据库相关联，因为它们是丰富、易于查找和使用的样本。

在本章的第一部分，你学习了如何创建一个预加载了示例数据库的 Azure SQL 数据库。在本节中，你将了解更多可用的示例数据库，以及如何在 Azure SQL 数据库和 Azure SQL 托管实例上开始使用它们。

有两个最常用的示例数据库：`AdventureWorks` 和 `WideWorldImporters`。`AdventureWorks` 示例围绕虚构的零售公司 AdventureWorks Cycles 构建，而 `WideWorldImporters` 是一个虚构的、具有全球贸易业务的批发公司。`AdventureWorks` 示例数据库最初是为 SQL Server 2008 发布的，`WideWorldImporters` 是为 SQL Server 2016 发布的。正如你在本章第一部分看到的，使用 Azure CLI，你可以部署 `AdventureWorks` 数据库的“轻量”版本“`AdventureWorksLT`”。还有额外的数据库示例：用于 SQL Server 标准版功能的 `WideWorldImporters` 数据库“`WideWorldImportersStd`”和用于 SQL Server 企业版功能的 `WideWorldImporters` 数据库“`WideWorldImportersFull`”。多年来，这些示例通常会随着最新功能而更新。这些数据库需要手动部署，就像部署任何你可能已有的其他现有数据库一样。让我们看看如何操作。

将示例数据库导入 Azure SQL 数据库与导入 Azure SQL 托管实例的方法有所不同。对于 Azure SQL 数据库，你可以通过利用 bacpac 文件（一种扩展名为“.bacpac”的 Windows 文件，包含数据库的数据和架构）来完成。你可以从这里下载 `WideWorldImporters` 文件：[`https://aka.ms/wwi10`](https://aka.ms/wwi10)。

如果你未在业务关键型（或 DTU 模型中的高级版）服务层中运行，由于通用型层不支持内存优化表，样本中的一些依赖项将要求你使用以“Standard”或“std”结尾的样本。如果你在业务关键型（或高级版）中运行，那么你可以使用所有样本（包括以“Full”结尾的样本）。

你需要创建一个存储账户（[`https://aka.ms/sdrsac`](https://aka.ms/sdrsac)），然后将 bacpac 文件复制到 Azure Blob 容器中。复制 bacpac 文件的方法有很多，但使用 Azure Storage Explorer（[`http://aka.ms/storage-explorer`](http://aka.ms/storage-explorer)）和通过 Azure 门户上传是两种简单的方法。一旦 bacpac 文件被复制到 Azure Blob 容器中，你就可以利用以下代码，通过 Azure CLI 将数据库导入到你的 Azure SQL 数据库中。

```
az sql db import \
-g <resource-group-name> \
-s <server-name> \
-n <database-name> \
-u <admin-login> \
-p <admin-password> \
--storage-key-type StorageAccessKey \
--storage-key <storage-account-key> \
--storage-uri https://<storage-account-name>.blob.core.windows.net/<bacpac-file-name>.bacpac
```

（清单 2-8：将 bacpac 文件导入 Azure SQL 数据库）

在 PowerShell 中也有类似的功能，在 Azure 门户中也有 GUI 体验。在 .bak 文件可以导出为 bacpac 文件的情况下，这种方法非常适合将数据库迁移到 Azure SQL 数据库。


