# 第五章

## SQL Server 数据源

在本书中，我们探讨了从几个关系型数据库导入数据，以及它们如何用作 SQL Server 数据源的一些方法。然而，有一种关系型数据库我们尚未讨论，那就是 SQL Server 本身。因此，为了继续我们的“数据源之旅”，让我们研究一下可以在 SQL Server 数据库之间传输数据的一些方法。

此概述包括：

*   即席查询外部 SQL Server 实例
*   SQL Server 链接服务器
*   将数据从一个 SQL Server 数据库批量加载到另一个 SQL Server 数据库
*   将数据从旧版本的 SQL Server 加载到 SQL Server 2005、2008 和 2012
*   使用 `COPY_ONLY` 备份
*   快照复制
*   在数据库之间复制和粘贴少量数据
*   将数据加载到 SQL Server Azure

选择 SQL Server 作为数据源可能令人惊讶，但在许多企业中，有几十个——即使没有数百个——SQL Server，通常运行着不同版本的 Microsoft RDBMS。因此，你很可能需要了解在 SQL Server 版本之间获取数据有哪些选项。

不可能讨论 SQL Server 版本之间数据传输的方方面面，也必然有一些技术超出了本书的范围。由于我的重点是数据集成，并且非常关注 ETL，因此我不会研究 SQL Server 的众多高可用性选项，也不会涉及 Service Broker。我也不提导入/导出向导，因为这已经在第一章、第二章和第四章中广泛介绍。

不过，本章将探讨将数据迁移到 SQL Server Azure。虽然 Microsoft 的“云中”数据库无疑将取代许多供应商的现场数据库，但将其作为来自 Microsoft 数据库的数据的逻辑目标来讨论似乎最有效且易于理解。

关于遵循本章给出的示例，有几点需要注意。首先，许多示例需要另一个 SQL Server 2012 实例。这可以是一个单独的联网服务器，也可以是安装了定义实例名称的 SQL Server 的第二个安装。我在示例中使用的是 `ADAM02\AdamRemote` 实例。你需要将其替换为你正在使用的服务器以及可能的实例。你还需要将 `CarSales` 示例数据库部署到这个第二个实例上。除非指定了其他数据库，否则所有示例都假定你使用的是 `CarSales` 数据库。本章使用的任何示例文件都可以在 `C:\SQL2012DIRecipes\CH05` 目录中找到——前提是你已经从本书的配套网站下载了示例并按照附录 B 中的说明进行了安装。

### 5-1. 从其他 SQL Server 实例加载即席数据

#### 问题

你想要快速、轻松地从另一个 SQL Server 实例临时加载数据。

#### 解决方案

使用 `OPENROWSET` 和 `OPENDATASOURCE`。这允许你快速连接到源数据并使用 T-SQL 选择任何数据子集。

以下是使用 `OPENROWSET` 的代码 (`C:\SQL2012DIRecipes\CH05\OpenRowset.Sql`)：

```sql
SELECT Lnk.ClientName INTO MyTable
FROM OPENROWSET('SQLNCLI', 'Server=ADAM02;Trusted_Connection=yes;',
     'SELECT         ClientName
      FROM           CarSales.dbo.Client
      ORDER BY       ClientName') AS Lnk;
```

你可以像这样使用 `OPENDATASOURCE` (`C:\SQL2012DIRecipes\CH05\OpenDataSource.Sql`)：

```sql
SELECT ID, ClientName, Town INTO MyTable
FROM OPENDATASOURCE('SQLNCLI',
 'Data Source=ADAM02\AdamRemote;Integrated Security=SSPI').CarSales.dbo.Client
```

在这两种情况下，源数据都会被加载到目标表中。

#### 工作原理

如果你希望“一次性”导入数据，那么幸运的是，快速连接到另一个 SQL Server 实例极其容易。与大多数外部关系源一样，有两种建立连接的方式。它们是：

*   `OPENROWSET`：用于偶尔的查询。
*   `OPENDATASOURCE`：用于可能有朝一日发展为链接服务器的偶尔查询。

以下是相关的先决条件。

*   每个外部 SQL Server 实例上都必须安装 OLEDB 提供程序。诚然，这通常是 SQL Server 安装的一部分，但我倾向于说明显而易见的事情。
*   作为集群一部分的每个 SQL Server 上都必须安装 OLEDB 提供程序。
*   必须在运行查询的服务器上启用即席分布式查询。这是通过以下 T-SQL 片段完成的 (`C:\SQL2012DIRecipes\CH05\ConfigureDistributedQueries.Sql`)：

```sql
EXECUTE master.dbo.sp_configure 'show advanced options', 1;
GO
RECONFIGURE;
GO
EXECUTE master.dbo.sp_configure 'ad hoc distributed queries', 1;
GO
RECONFIGURE;
GO
```

对于偶尔的即席查询，你可能会发现 `OPENROWSET` 是最简单的解决方案。澄清一下，`OPENROWSET` 的参数基本上分为三部分：

*   OLEDB 提供程序
*   一个提供程序字符串，包含服务器和安全参数
*   一个用于检索数据的 T-SQL 查询

由于提供程序字符串仅指定服务器，因此最好使用三部分表示法来指定你希望从中获取数据的数据库、架构、表或视图。如果登录默认为该数据库和架构，那么当然没有问题；但我建议将其作为最佳实践习惯。如果你希望使用 SQL Server 安全性而不是可信连接，则将 `Trusted_Connection=yes` 替换为登录名和密码详细信息，如下所示：

```sql
'datasource=ADAM02\AdamRemote;user_id=AdamRemote;password=AdamRemPwd'
```

请注意，安全信息是第二个参数的一部分，参数元素用分号分隔。另外，冒着说明显而易见的风险，像这样以明文形式保留安全信息是极其危险的。如果你没有其他选择，那么你应该考虑将 `SELECT` 语句包装在使用 `WITH ENCRYPTION` 选项创建的存储过程中，这将对许多——但不是所有——窥探的眼睛隐藏存储过程的文本。或者，存储过程可以驻留在远程服务器上。在这种情况下，它需要由管理该服务器的团队创建。出于安全考虑，存储过程通常是更好的选择，因为你不会通过网络传递你的架构详细信息。

请记住，你使用的是纯 T-SQL，因此可以扩展 `SELECT` 子句（传递给外部服务器的以及包装 `OPENROWSET` 命令的代码）以包含 `WHERE`、`ORDER BY` 和 `GROUP BY` 子句以及列别名。这些技术在第 1-4 和 1-5 配方中有更详细的描述。

如果你使用 `OPENDATASOURCE`，你可以使用 SQL Server 安全性，以及以明文形式保留密码所暗示的所有警告。这里有一个片段来展示它：

```sql
SELECT   ClientID
FROM     OPENDATASOURCE('SQLNCLI',
                 'Data Source=ADAM02\AdamRemote;User ID=Adam;password=AdamRemPwd').CarSales.dbo.Client
```


#### 提示、技巧与陷阱
*   如果你怀疑一个临时查询将来可能成为更持久方案的一部分，那么使用 `OPENDATASOURCE` 来设置它，可以让你更轻松地更改为链接服务器。它使用 SQL Server 四部分表示法，允许你在后续用链接服务器引用替换 SQL 片段。
*   你不能在分布式查询中使用基于 CLR 的数据类型。实际上，这意味着不能读取（或转换其数据类型）`GEOGRAPHY`、`GEOMETRY` 或 `HIERARCHYID` 数据类型。但是，这些数据类型可以在直通查询中使用——因此你可以在使用 `OPENQUERY`（相对于四部分表示法）或 `OPENROWSET` 的链接服务器查询中使用它们。

