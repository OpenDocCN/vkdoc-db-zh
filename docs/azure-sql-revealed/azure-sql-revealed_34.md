# 机器学习服务

在 SQL Server 2016 中，我们为 SQL Server 引入了 `R Services`。其概念是采用一种新架构，让你能够以安全、隔离和可扩展的方式，在 `与 SQL Server 同一台计算机` 上运行 R 程序。在 SQL Server 2017 中，我们增加了 Python 支持，并将此功能重新命名为 `Machine Learning Services`。

我们最终在 Azure SQL 托管实例中启用了此功能（在 Azure SQL Database 中不可用），但有几点注意事项：

*   你需要使用 `sp_configure` 的 `external scripts enabled` 选项来启用该功能。
*   Java 作为可扩展语言在 SQL Server 2019 中被添加，但在 Azure SQL 托管实例中不可用。
*   虽然预装了 R 和 Python 包，但如果你需要不同的包，可能需要使用一个名为 `sqlmlutils` 的工具。更多信息请参阅：[`https://learn.microsoft.com/azure/azure-sql/managed-instance/machine-learning-services-differences?view=azuresql#python-and-r-packages`](https://learn.microsoft.com/azure/azure-sql/managed-instance/machine-learning-services-differences?view=azuresql#python-and-r-packages)。
*   你无法对 R 和 Python 的执行使用自定义资源治理。

完整的差异列表请阅读：[`https://learn.microsoft.com/azure/azure-sql/managed-instance/machine-learning-services-differences`](https://learn.microsoft.com/azure/azure-sql/managed-instance/machine-learning-services-differences)。

