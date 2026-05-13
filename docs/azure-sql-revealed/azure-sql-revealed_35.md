# 分布式事务 (DTC)

自本书第一版发布以来，`DTC` 支持已成为 Azure SQL 托管实例的一项新功能。此功能常被客户视为从 SQL Server 迁移的阻碍，因此托管实例工程团队在云端启用了它。

首先需要明确在什么情况下为托管实例使用 `DTC` 功能，以及在什么情况下支持跨 Azure 云服务的分布式数据库事务。

你可以执行跨 Azure SQL Database 或 Azure SQL 托管实例内数据库的分布式事务（使用 `.Net` 库），这通过一种称为分布式事务的 `原生支持` 概念实现。你无法跨 Azure SQL Database 和 Azure SQL 托管实例使用此功能。关于原生支持的更多信息请参阅：[`https://learn.microsoft.com/azure/azure-sql/database/elastic-transactions-overview`](https://learn.microsoft.com/azure/azure-sql/database/elastic-transactions-overview)。

Azure SQL 托管实例的 `DTC` 支持提供了在 `混合` 环境或 Azure SQL 托管实例之间（包括使用 T-SQL）运行分布式事务的能力。混合环境可以是任何支持分布式事务协议（如 `XA`）的系统，包括本地 SQL Server 实例。在这种情况下，Azure SQL 托管实例管理 `DTC` 的所有方面，就像由 Windows Server 本地管理的 `DTC` 服务一样。对于跨托管实例的分布式事务，你还需要利用一种称为 `服务器信任组` 的安全模型。这为使用 Microsoft Entra 的 `DTC`、`Service Broker` 和链接服务器等场景提供了跨实例的身份验证模型。了解更多关于服务器信任组的信息：[`https://learn.microsoft.com/azure/azure-sql/managed-instance/server-trust-group-overview`](https://learn.microsoft.com/azure/azure-sql/managed-instance/server-trust-group-overview)。为托管实例配置 `DTC` 需要考虑网络连接问题。了解更多：[`https://learn.microsoft.com/azure/azure-sql/managed-instance/distributed-transaction-coordinator-dtc`](https://learn.microsoft.com/azure/azure-sql/managed-instance/distributed-transaction-coordinator-dtc)。

