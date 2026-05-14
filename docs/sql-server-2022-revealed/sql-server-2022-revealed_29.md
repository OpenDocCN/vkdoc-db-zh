# 查询存储提示

在 2021 年夏天，我们为 Azure SQL Database 引入了查询存储提示的概念。查询存储提示的理念是在不更改应用程序代码的情况下塑造查询执行计划。

> **注意**
>
> 您可以在此期 Data Exposed 节目中阅读关于查询存储提示最初发布的信息，由 Anna Hoffman 和 Joe Sack 主讲：[`youtu.be/pYB6Uik_Q7A`](https://youtu.be/pYB6Uik_Q7A)。

*查询提示*的概念并不新鲜，您可能已经在使用它们。通过向 T-SQL 语句添加 `OPTION` 子句，您可以影响查询优化器构建查询计划。我见过两个常用的查询提示是 `RECOMPILE`（每次执行查询时重新编译）和 `MAXDOP`（在查询级别控制最大并行度）。您可以在[`docs.microsoft.com/sql/t-sql/queries/hints-transact-sql-query`](https://docs.microsoft.com/sql/t-sql/queries/hints-transact-sql-query)阅读所有可能的查询提示和示例。

如果您没有使用查询提示，也不必担心。查询提示是为那些由于某种原因您的应用程序与查询处理器默认编译的查询计划配合不佳的场景而设计的。

然而，出于实际生产工作负载的原因，确实需要这些提示。查询提示要求能够更改 T-SQL 语句。但是，如果 T-SQL 语句在应用程序代码中被调用，更改应用程序代码来应用提示不切实际或不可能，该怎么办？这正是查询存储提示可以发挥作用的地方，因为查询优化器可以基于查询存储中存储的内容应用提示，而无需更改应用程序代码。

## 如何使用查询存储提示？

以下是使用查询存储提示的步骤，假设查询存储已启用读写功能：

*   执行查询，其计划存储在查询存储中。
*   您使用查询存储目标视图找到要对其应用提示的查询的 `query_id`。
*   您执行存储过程 `sys.sp_query_store_set_hints`，指定 `query_id` 和要应用的提示。
*   您可以使用视图 `sys.query_store_query_hints` 查看现有的查询存储提示。

我们将提示保存在查询存储中，这样查询处理器将在下次执行该查询时使用提供的提示。由于提示是持久化的，它将在重启和计划缓存清除后依然存在。

并非所有查询提示都支持查询存储。有关支持的查询提示的完整列表，请查阅文档[`docs.microsoft.com/sql/relational-databases/system-stored-procedures/sys-sp-query-store-set-hints-transact-sql`](https://docs.microsoft.com/sql/relational-databases/system-stored-procedures/sys-sp-query-store-set-hints-transact-sql)。还有一些情况下，根据给定查询的编译规则，查询优化器无法使用您指定的查询提示。这被称为查询存储提示*失败*，并将记录在视图 `sys.query_store_query_hints` 中。

文档在[`docs.microsoft.com/sql/relational-databases/performance/query-store-hints?#examples`](https://docs.microsoft.com/sql/relational-databases/performance/query-store-hints?#examples)提供了一个非常好的端到端使用查询存储提示的示例。我将在第 5 章中向您展示一个系统生成的查询存储提示的示例，称为**基数估计 (CE) 模型反馈**。

## 我应该在什么时候使用查询存储提示？

虽然查询提示可能是您长期与应用程序一起使用的有用功能，但我见过的大多数查询提示用法都是临时措施。因此，当您想临时应用查询提示而不更改（或无法更改）应用程序代码时，查询存储提示可以是一个极其有用的功能。在您想覆盖应用程序代码中查询的提示时，查询存储提示也很方便。

> **注意**
>
> 在使用查询存储提示之前，您应该了解一些使用它们的注意事项，您可以在[`docs.microsoft.com/sql/relational-databases/performance/query-store-hints?#query-store-hints-and-feature-interoperability`](https://docs.microsoft.com/sql/relational-databases/performance/query-store-hints?#query-store-hints-and-feature-interoperability)阅读相关内容。

在使用查询存储提示之前，请确认您试图解决的问题无法通过其他方式完成。例如，许多人使用 `RECOMPILE` 或 `OPTIMIZE FOR` 提示来解决经典的**参数敏感计划**问题。在 SQL Server 2022 中，我们有一个针对此问题的新解决方案，可能让您避免使用那些查询提示。您可以在第 5 章中阅读更多关于**参数敏感计划 (PSP)** 优化的信息。

## 这与计划指南有何不同？

与查询存储提示类似，计划指南提供了一种在不更改应用程序代码的情况下塑造查询计划的方法。计划指南在 SQL Server 中已经存在了很多个版本。查询存储提示提供了一个更简单的接口来塑造查询计划，主要是因为您只需要向查询存储中存储的现有查询添加一个或多个查询提示。

计划指南可以通过提供应用程序中查询的原始 SQL 文本或使用缓存中计划的计划句柄来创建。因此，查询存储提示具有实现更简单的优势，并且可以用于随时间持久化的查询。计划指南优于查询存储提示的一个优势是它们不需要启用查询存储。

