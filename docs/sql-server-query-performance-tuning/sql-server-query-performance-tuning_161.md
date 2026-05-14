# 第 19 章 ■ 减少查询资源使用

此设置允许用户从数据库检索数据，但防止其修改数据。设置会立即生效。如果需要对数据库进行偶尔的修改，则可以临时将其转换为 `READWRITE` 模式。

`ALTER DATABASE <数据库名称> SET READ_WRITE`

`<数据库修改操作>`

`ALTER DATABASE <数据库名称> SET READONLY`

• 使用一种快照隔离级别。

SQL Server 提供了一种机制，在数据更新发生时将其版本存入 `tempdb`，从而显著减少读取操作的锁开销和阻塞。你可以使用 `ALTER` 语句更改数据库的隔离级别。

```sql
ALTER DATABASE AdventureWorks2012 SET TRANSACTION ISOLATION LEVEL READ_COMMITTED_SNAPSHOT;
```

• 防止 `SELECT` 语句请求任何锁。

```sql
SELECT * FROM <表名> WITH(NOLOCK)
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

这可以防止 `SELECT` 语句请求任何锁，并且仅适用于 `SELECT` 语句。

虽然 `NOLOCK` 提示不能直接在操作查询（`INSERT`、`UPDATE` 和 `DELETE`）所引用的表上使用，但它可以在操作查询的数据检索部分使用，如下所示：

```sql
DELETE Sales.SalesOrderDetail
FROM Sales.SalesOrderDetail sod WITH(NOLOCK)
JOIN Production.Product p WITH(NOLOCK)
ON sod.ProductID = p.ProductID
AND p.ProductID = 0;
```

需要了解的是，这会导致脏读，可能引起行重复或行丢失，因此被认为是控制锁的最后手段。最佳做法是将数据库标记为只读或使用一种快照隔离级别。

这是一个很大的话题，还有很多内容可以讨论。我将在下一章讨论不同类型的锁请求以及如何管理锁开销。如果你对数据库执行了本节中提出的任何更改，我建议从备份中恢复。

## 本章小结

正如本章所述，要提高数据库应用程序的性能，确保 SQL 查询设计得当以受益于索引、存储过程、数据库约束等性能增强技术至关重要。确保查询对资源友好，并且不妨碍索引的使用。在许多情况下，优化器能够生成具有成本效益的执行计划，无论查询结构如何，但首先正确设计查询仍然是良好的实践。即使你为单个查询设计了出色的性能，数据库应用程序的整体性能也可能不尽如人意。不仅提高单个查询的性能很重要，确保它们与其他查询良好协作而不引起严重的阻塞问题也同样重要。在下一章中，你将探讨数据库应用程序的不同阻塞方面。

[www.it-ebooks.info](http://www.it-ebooks.info/)

