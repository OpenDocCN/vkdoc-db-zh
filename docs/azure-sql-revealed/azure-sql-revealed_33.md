# 全文搜索

全文搜索在过去的多个 SQL Server 版本中一直是引擎的集成组件。全文搜索通过 T-SQL 提供针对数据库中存储的 `text` 数据的搜索功能。尽管全文搜索在技术上是 SQL Server 的实例级功能，但我们在 Azure SQL 托管实例和数据库中均支持此功能。你可以在这里阅读如何开始使用全文搜索：[`https://learn.microsoft.com/sql/relational-databases/search/full-text-search`](https://learn.microsoft.com/sql/relational-databases/search/full-text-search)。

在 Azure SQL 方面，全文搜索有几个限制：

*   不支持"第三方过滤器"（例如 Office 和 .pdf 过滤器）。
*   你无法管理支持全文的服务（如 `fdhost`）。
*   不支持语义搜索。

