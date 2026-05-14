# 第 25 章 ■ 数据库工作负载优化

## 迭代优化阶段

**图 25-13.** 显示优化代价最高查询对完整工作负载影响的扩展事件输出

根据此跟踪，表 25-5 总结了所考虑查询的资源使用情况和响应时间（即持续时间）。

**表 25-5.** 优化前后，优化查询的资源使用情况和响应时间

| 列名 | 优化前 | 优化后 |
| :--- | :--- | :--- |
| 读取次数 | | |
| 写入次数 | | |
| CPU | 16 ms | 0 ms |
| 持续时间 | 1313 ms | 19.4 ms |

■ **请注意**，绝对值不如优化前与对应优化后值之间的相对差异重要。数值间的相对差异表明了性能的提升。

优化性能最差的查询可能会损害工作负载中某些其他查询的性能。然而，只要工作负载的整体性能得到提升，就可以保留对该查询执行的优化。在本例中，其他查询未受到影响。但现在有一个查询比其他查询花费时间更长。它可能也需要优化，于是整个过程再次开始。

需要记住的一个重要点是，你需要多次迭代优化步骤。在每次迭代中，你可以识别一个或多个性能不佳的查询，并对这些查询进行优化以提高整体工作负载的性能。你必须持续迭代优化步骤，直到获得足够的性能或满足你的服务级别协议（`SLA`）。

除了分析资源密集型查询的工作负载外，你还必须分析工作负载是否存在错误条件。例如，如果你尝试向一个包含受唯一约束保护的列的表中插入重复行，`SQL Server` 将拒绝新行并向应用程序报告错误条件。尽管数据未被输入表中，也没有执行有用的工作，但宝贵的资源却被用于确定数据无效并必须被拒绝。

要识别由数据库请求引起的错误条件，你需要在 `扩展事件` 会话中包含以下内容（或者，你可以创建一个新会话，在错误或警告类别中查找这些事件）：

- `error_reported`
- `execution_warning`
- `hash_warning`
- `missing_column_statistics`
- `missing_join_predicate`
- `sort_warning`

例如，考虑以下 SQL 查询：

```sql
INSERT INTO Purchasing.PurchaseOrderDetail
(
    PurchaseOrderID,
    DueDate,
    OrderQty,
    ProductID,
    UnitPrice,
    ReceivedQty,
    RejectedQty,
    ModifiedDate
)
VALUES
(
    1066,
    '1/1/2009',
    1,
    42,
    98.6,
    5,
    4,
    '1/1/2009'
);
GO
```

```sql
SELECT p.[Name], psc.[Name]
FROM Production.Product AS p, Production.ProductSubCategory AS psc;
GO
```

图 25-14 显示了相应的会话输出。

**图 25-14.** 显示 SQL 工作负载引发错误的扩展事件输出

从图 25-14 的 `扩展事件` 输出中，你可以看到我故意生成的两个错误发生了：

- `error_reported`
- `missing_join_predicate`

`error_reported` 错误是由 `INSERT` 语句引起的，该语句尝试插入的数据未通过引用完整性检查；具体来说，它尝试插入 `ProductId = 42`，但 `Production.Product` 表中不存在该值。从 `error_number` 列中，你可以看到错误编号是 `547`。`message` 列显示了错误的完整描述。不过值得注意的是，`error_reported` 可能相当“健谈”，返回大量数据，并非所有数据都有用。

第二种错误 `missing_join_predicate` 是由 `SELECT` 语句引起的。

```sql
SELECT p.[Name], c.[Name]
FROM Production.Product AS p, Production.ProductSubCategory AS c;
GO
```

如果你仔细查看 `SELECT` 语句，会发现该查询没有在两张表之间指定 `JOIN` 子句。表之间缺少连接谓词通常会导致不准确的结果集和高成本的查询计划。这就是所谓的 `笛卡尔连接`，它会导致 `笛卡尔积`，即一张表的每一行与另一张表的每一行组合。你必须在“错误和警告”部分识别引起此类事件的查询，并实施必要的修复。例如，在上面的 `SELECT` 语句中，你不应该将 `Production.ProductCategory` 表的每一行与 `Production.Product` 表的每一行连接起来——你只应该连接具有匹配 `ProductCategoryID` 的行，如下所示：

```sql
SELECT p.[Name], c.[Name]
FROM Production.Product AS p
JOIN Production.ProductSubCategory AS c
    ON p.ProductSubcategoryID = c.ProductSubcategoryID;
```

即使在你彻底分析并优化了工作负载之后，你也必须记住，工作负载优化不是一个一次性的过程。数据库上的工作负载或数据分布会随时间变化，因此你应该定期检查你的查询是否针对当前情况进行了优化。你也可能发现数据库本身设计上的缺陷。过度规范化导致的过多连接或不当反规范化导致的过多列，都可能导致查询性能不佳，且没有真正的优化机会。在这种情况下，你将需要考虑重新设计数据库以获得更优化的结构。

## 总结

正如你在本章所学，优化数据库工作负载需要一系列工具、实用程序和命令来分析工作负载中涉及的查询的不同方面。你可以使用 `扩展事件` 来分析工作负载的全局情况并识别高成本查询。一旦识别出高成本查询，你可以使用查询窗口和各种 `SQL` 命令来排查与这些高成本查询相关的问题。根据检测到的高成本查询问题，你可以应用一组或多组优化技术来提高查询性能。对高成本查询的优化应能提升工作负载的整体性能；如果未能实现这一点，你应该回滚所做的更改。

在下一章中，我将简明扼要地总结与性能相关的最佳实践。你将能够把这些信息作为快速简便的参考。

