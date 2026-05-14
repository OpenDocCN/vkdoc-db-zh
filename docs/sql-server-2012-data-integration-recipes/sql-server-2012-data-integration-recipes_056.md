# SQL 数据库

你需要加载到 SQL Server 的大量数据可能已经存储在关系数据库中。虽然你当然可以从目前大多数可用的基于 SQL 的商业数据库导出到文本或 XML 文件，然后再导入 SQL Server，但也有标准的方法可以直接连接到大多数 RDBMS（关系数据库管理系统），然后将数据从中提取到 Microsoft RDBMS 中。

因此，在本章中，我们将研究一些可用的方法，用于摄取和链接来自以下数据库的数据：

-   Oracle
-   DB2
-   MySQL
-   Sybase
-   Teradata
-   PostgreSQL

正如你将看到的，对于我们将涵盖的所有数据库，用于摄取数据的技术非常相似。它们包括（取决于源数据库）：

-   SSIS
-   即席连接
-   链接服务器

它们都有一个共同点，即你需要在 SQL Server 目标服务器上安装并配置好用于源数据库的提供程序。在大多数情况下，你可以使用 OLEDB 或 ODBC 提供程序（或者对于某些情况下的 SSIS，可以使用 .NET 提供程序）。从那时起，差异基本上是次要的。因此，我们将逐一查看每个数据库，并向你展示如何连接到源数据库以及如何将数据从那里加载到 SQL Server。

一旦我们建立了这些基本方法，还有一些其他有趣的事情值得关注，特别是如何使用适用于 Oracle、MySQL 和 Sybase 的 SQL Server 迁移助手 (SSMA)。

你可能已经意识到，现实世界中的数据库间数据连接有其应有的挑战，你将不得不处理以下部分或全部问题：

-   安装和配置 OLEDB 和 ODBC 提供程序
-   网络和防火墙问题
-   数据库安全性
-   数据类型映射

在理想的世界中，这些元素是清晰且文档完善的。在我们大多数人居住的现实中，情况有些模糊，你最终可能需要了解多个 IT 系统，或者至少需要精确了解几个系统中某些极其具体的方面。这始终是一个挑战，并且可能需要深入研究各种文档。

## 前言：安装和配置 OLEDB 和 ODBC 提供程序

如果你打算直接从另一个关系数据库导入数据，关键是要有一个正常工作且经过验证的提供程序。这说起来容易，但到目前为止，这可能是数据加载中最困难的部分，原因有几个。考虑以下你必须回答的问题：

-   你安装哪种类型的提供程序，从哪个供应商处获取？
-   它提供什么级别的数据库兼容性？
-   它是 64 位还是 32 位？
-   可获得什么级别的支持？

现在，使用哪个提供程序、哪种类型（OLEDB、ODBC 或 .NET）以及你偏好哪个供应商，完全取决于你。SQL Server 自带以下 OLEDB 提供程序，以方便连接到某些其他关系数据库：

-   用于 Oracle 的 Microsoft OLEDB 提供程序 (`MSDAORA.1`)，它是 SQL Server 安装的一部分。
-   用于 DB2 的 Microsoft OLEDB 提供程序（随功能包提供，但需要 SQL Server 企业版）。
-   用于 Oracle 的 Attunity OLEDB 提供程序（需要 SQL Server 企业版）。
-   用于 Teradata 的 Attunity OLEDB 提供程序（需要 SQL Server 企业版）。

当然，你还可以使用数据库供应商自己提供的 OLEDB 和 .NET 提供程序。在撰写本文时——针对本章中提到的数据源——以下是一些可用的提供程序：

-   Oracle Corporation 的 Oracle Provider for OLE DB 11.2.0.3.0。
-   Oracle Corporation 的 Oracle Data Provider for .NET 4 11.2.0.3.0。
-   IBM DB2 for I5/OS `IBMDA400` OLEDB Provider
-   IBM DB2 for I5/OS `IBMDARLA` OLEDB Provider
-   IBM DB2 for I5/OS `IBMDASQL` OLEDB Provider
-   IBM OLEDB Provider (`IBMDADB2`)
-   IBM OLE DB .NET 7 Data Provider
-   Sybase ASE OLEDB Provider
-   Sybase ASE ODBC Provider
-   Sybase ASE .NET Provider
-   MySQL ODBC Provider
-   PostgreSQL Native OLEDB Provider (`PGNP`)

鉴于该领域的不断发展，我不会指定你应该使用哪个版本。显然，在几乎所有情况下，最新版本都是首选。


此外，如果您在生产环境中使用这些驱动程序，您需自行确保遵守任何许可要求。

当然，市面上还有许多其他商业可用的提供程序，我只能建议您自行上网搜索。它们来自许多不同的来源。同样五花八门的是每个提供程序对其所谓优越性的宣称。我在此绝不会做出任何评判。我这里将只解释那些可从 Microsoft 或我们在本章所探讨的 RDBMS 供应商本身获得的提供程序。这绝非批评或评判，仅仅是本章范围的一种自愿性限制。

下载并安装后，这些提供程序应该在 SSIS 连接管理器的 `OLEDB` 源列表中可见，并且在您展开 `SQL Server Management Studio` 中的 `Server Objects` ![image](img/arrow.jpg) `Linked Servers` ![image](img/arrow.jpg) `Providers` 时也可见。

#### 网络与防火墙问题

务必确保与您组织内的网络架构师搞好关系，因为您将需要他们的帮助。或者，获取关于网络架构的完整文档。读到上一句时，我想大多数开发人员和 DBA 都会发出一声苦笑，并得出结论：需要施展魅力才能确保他们的 `SQL Server` 至少能看到其他数据库主机，因为这是建立服务器间数据库连接的起点。

#### 数据库安全

一旦与基础设施人员建立了友谊，下一个“魅力攻势”无疑将针对源系统的 DBA。您需要源数据库的登录和 `SELECT` 权限（这是最低要求）——而如果您需要检查源数据库元数据，则需要更广泛的权限。实际上，正如您将看到的，这是使用 `SSMA` 的先决条件。

#### 数据类型映射

幸运的是，`SQL Server` 开发团队多年来已为各大主流竞争数据库定义了一套稳健的数据类型对应关系。其中许多在附录 A 中给出。`SSMA` 也有预定义（且可配置）的数据类型映射方案，您不仅可以在使用此工具加载数据时使用，也可以将其作为建议映射的参考。幸运的是，根据我的经验，这些建议非常可靠，使得该领域的难题成为例外而非规则。当问题确实发生时，您可能别无选择，只能参考源系统的文档。

在开始本章的实践指南之前，有一件事我必须立即澄清。我意识到，讨论半打主流数据库及将它们连接到 `SQL Server` 的方法，有可能成为一个庞大的主题。因此，我将极其有选择性地讨论哪些产品和哪些连接方法。由于不可能讨论从已知宇宙中所有关系型数据库加载数据的所有方法的方方面面，我选择在本章刻意保持简洁，并专注于 RDBMS 市场中的主要参与者——这些产品的数据加载问题我有幸在过去多年中与之“搏斗”过。不可避免地，许多内容将无法涵盖，但我恐怕必须在某个地方划定界限。

此外，我将始终使用我们所探讨的每个 RDBMS 自带的示例数据库。如果某个数据源没有标准的示例数据库，我将使用 `INFORMATION_SCHEMA` 数据或源数据库中作为标准存在的任何表。

![image](img/sq.jpg) **注意** 从其他 `SQL` RDBMS 导入数据的难点在于，在某些情况下，您需要对源数据库具备一定程度的基础知识。在其他情况下，`SQL` 知识就足够了。

因此，在本章中，我假定您对外部数据库有基本的了解——或者至少具备一个可以相当快获得的初始理解水平。

我假定您已从本书的配套网站下载了本章的示例文件，并将其安装在 `C:\SQL2012DIRecipes\CH02\` 目录中。同样地，如果您要跟随示例操作，您需要按照附录 B 中的描述创建 `CarSales` 和 `CarSales_Staging` 数据库。

## 4-1. 配置服务器以连接到 Oracle

### 问题

您需要确保您将用于导入 Oracle 数据的 `SQL Server` 已正确配置，并且能够连接到 Oracle 数据库。

### 解决方案

在 32 位 `SQL Server` 上安装 32 位 Oracle 客户端，或在 64 位 `SQL Server` 上安装 64 位 Oracle 客户端。具体操作步骤如下：

1.  下载 Oracle 11G 完全客户端。按照 Oracle 指南进行安装。确保安装 Oracle `OLEDB`、`.NET` 和 `ODBC` 提供程序。
2.  通过编辑 `TNSNames.ora` 文件来配置 Oracle 访问。我将在“工作原理”部分解释这一点。
3.  重新启动安装了 Oracle 驱动程序的 `SQL Server`。

### 工作原理

作为全球市场份额最大的数据库，我们将首先探讨如何连接到 Oracle 数据库。尽管本指南中给出的说明假定您仅具备非常基础的 Oracle 知识，但在建立与 Oracle 服务器的连接时，您可能需要 Oracle DBA 的协助。这仅仅是因为连接到 Oracle 数据库时可能面临各种广泛的潜在场景。这个主题如此庞大，无法在此详细描述，因此我只展示如何使用 `TNS`（透明网络底层）连接。

请注意，在 `TNSNames.ora` 文件中，`Address` 是连接的首要元素。您稍后连接时将需要它。因此，使用一个纯粹假设的示例，以下 `TNSNAMES` 条目的 `Address` 名称是 `MyOracle`：

```text
MyOracle =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = aa.calidra.co.uk)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = MyOracleReally)
    )
  )
```

然而，我只能建议您考虑在 64 位服务器上同时安装 32 位和 64 位客户端，因为这不仅允许您在 64 位环境中连接到 Oracle，还允许您在 `SSDT`（SQL Server Data Tools）/`BIDS`（Business Intelligence Development Studio）中进行开发、调整、最终确定和测试——请记住，这两者都是作为 32 位应用程序运行的。具体步骤如下：

1.  移除任何现有的 Oracle 客户端。重新启动。
2.  下载 Oracle 11G 完全客户端。按照 Oracle 指南安装 32 位客户端。确保安装 Oracle `OLEDB`、`.NET` 和 `ODBC` 提供程序。仔细定义它使用的 Oracle 主目录和目录路径，例如 `C:\Oracle\product\11.2.0\Client32`。
3.  在“可用产品组件”中选择用于 `OLEDB` 的 `Oracle Windows Interfaces 11.x.x` 组件。如果愿意，您也可以添加 `.NET` 提供程序。
4.  重新启动服务器。
5.  下载 Oracle 11G 完全客户端。按照 Oracle 指南安装 64 位客户端。确保安装 Oracle `OLEDB`、`.NET` 和 `ODBC` 提供程序。定义它使用的 Oracle 主目录，目录路径可以是类似 `C:\Oracle\product\11.2.0\Client64`。
6.  在“可用产品组件”中选择用于 `OLEDB` 的 `Oracle Windows Interfaces 11.x.x` 组件。如果愿意，您也可以添加 `.NET` 提供程序。
7.  重新启动服务器。
8.  为 32 位和 64 位环境分别编辑 `TNSNames.ora` 文件以配置 Oracle 访问。这意味着要修改 `C:\Oracle\product\11.2.0\Client32\Network\Admin` 和 `C:\Oracle\product\11.2.0\Client64\Network\Admin` 这两个目录中的 `TNSNames.ora` 文件。



