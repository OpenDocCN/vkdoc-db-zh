# Azure SQL 数据库样本获取与连接查询

## 获取样本数据库

在 Azure SQL Database 中，获取样本数据库的选项包括使用 Azure 门户、Azure CLI 或 Azure PowerShell。在 Azure SQL 托管实例中，这些选项同样可用（语法略有不同），但您还可以使用 T-SQL 进行原生还原。首先，您需要有权访问备份的 `.bak` 文件。在此情况下，备份文件可以存储在 Azure Blob 容器中，方式与前面的 Azure SQL Database 示例相同。这次，您还需要生成一个共享访问签名（SAS）密钥（使用 Azure CLI 操作的指导：[`https://aka.ms/ascsas`](https://aka.ms/ascsas)），以便您的托管实例可以访问备份文件。部署实例后，您可以使用选择的 T-SQL 查询工具连接到该实例，以创建指向备份的凭据。如果您复制并粘贴以下任何 T-SQL 命令，可能需要重新键入单引号。

```
CREATE CREDENTIAL [https://]
WITH IDENTITY = 'SHARED ACCESS SIGNATURE'
, SECRET ''
Listing 2-9
Create a credential to a backup
```

要检查凭据是否有效，您可以运行以下 T-SQL 来获取备份文件列表。对于 “WideWorldImporters-Standard.bak”，您应该会看到返回的 mdf、ndf 和 ldf 文件。

```
RESTORE FILELISTONLY FROM URL = 'https://'
Listing 2-10
View the files in a backup
```

最后，您可以使用以下代码从 URL 还原数据库。

```
RESTORE DATABASE [Wide World Importers] FROM URL = 'https://'
Listing 2-11
Restore the database from a URL
```

此还原过程是异步且可重试的，这意味着即使连接中断或发生超时，Azure SQL 托管实例也会在后台不断尝试还原数据库。虽然此示例与 WideWorldImporters 相关，但使用 T-SQL 在 Azure SQL 托管实例中还原数据库的过程对于其他备份文件将是相同的。更多信息请参考：[`https://aka.ms/sdmirsdq`](https://aka.ms/sdmirsdq)。

## 延伸阅读

与本地相比，创建 Azure SQL 数据库或托管实例是一项简单的操作，但仍需要做出一些选择，以确保您不会为不需要的东西浪费资金。这或多或少与您在本地应用的概念相同。更大的区别在于向上扩展要容易得多。请记住，在云中，资源可以像盐一样管理：您可以随时添加更多。

以下是一些链接，深入探讨了本章讨论的几个概念：

*   在 Azure SQL 中选择正确的部署选项 – [`https://docs.microsoft.com/azure/sql-database/sql-database-paas-vs-sql-server-iaas`](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-paas-vs-sql-server-iaas)
*   什么是 PaaS？ – [`https://azure.microsoft.com/overview/what-is-paas/`](https://azure.microsoft.com/en-us/overview/what-is-paas/)
*   Azure SQL 服务层级 – [`https://docs.microsoft.com/azure/sql-database/sql-database-service-tiers-general-purpose-business-critical`](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-service-tiers-general-purpose-business-critical)
*   Azure SQL 数据库服务器及其管理 – [`https://docs.microsoft.com/azure/sql-database/sql-database-servers`](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-servers)
*   Azure SQL 功能 – [`https://docs.microsoft.com/azure/sql-database/sql-database-features`](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-features)
*   Azure SQL 基础 – [`https://aka.ms/azuresqlfundamentals`](https://aka.ms/azuresqlfundamentals)

