# Service Broker

早在 SQL Server 2005 时代，我们就从客户那里了解到，他们希望使用异步消息传递技术构建面向服务的应用程序。我们设计并构建了一个名为 `Service Broker` 的系统，它利用 SQL Server 表、编程和通信的威力，在 SQL Server 引擎内部实现了一个消息传递系统。如果你从未使用过 `Service Broker`，可以从这里开始：[`https://learn.microsoft.com/sql/database-engine/configure-windows/sql-server-service-broker`](https://learn.microsoft.com/sql/database-engine/configure-windows/sql-server-service-broker)。

`Service Broker` 是实例级功能的又一个例子，我们在 Azure SQL Database 中不支持此功能。`CloudLifter` 再次来帮忙。我们在 Azure SQL 托管实例中支持 `Service Broker` 应用程序。一个主要的限制是我们仅支持在托管实例之间使用 `Service Broker`。完整的差异列表请参阅：[`https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#service-broker`](https://learn.microsoft.com/azure/azure-sql/managed-instance/transact-sql-tsql-differences-sql-server?view=azuresql#service-broker)。

