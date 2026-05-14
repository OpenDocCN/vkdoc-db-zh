# 第 5 章 XML 索引

## XML 索引的性能考量

为了讨论 XML 索引的性能，让我们编写一个查询，该查询非常适合我们已在`OrderSummary`表上创建的 PATH 辅助 XML 索引。清单 5-6 中的查询针对`OrderSummary`表运行，并返回所有订购了`StockItemID`为 223 的“巧克力针鼹 250 克”产品的客户行。脚本的第一部分从缓冲区缓存中移除未更改的页并清除计划缓存，以确保测试的公平性。脚本的中间部分启用了时间统计，以便我们能准确记录查询执行所需的时间。

**提示** 性能会因服务器规格以及并发进程消耗的资源量而异。你应该始终在自己的环境中检查性能。

### 清单 5-5. 创建辅助 XML 索引

```sql
USE WideWorldImporters

GO

CREATE XML INDEX [SecondaryXmlIndex-OrderSummary-Path]
ON Sales.OrderSummary (OrderSummary)
USING XML INDEX [PrimaryXmlIndex-OrderSummary-OrderSummary] FOR PATH ;

GO
```

### 清单 5-6. 返回订购了 StockItemID 223 的客户的行

```sql
--清除缓冲区缓存和计划缓存
DBCC DROPCLEANBUFFERS
DBCC FREEPROCCACHE
GO

--启用时间统计以便随结果显示
SET STATISTICS TIME ON
GO

--运行查询
SELECT *
FROM Sales.OrderSummary
WHERE OrderSummary.exist('/SalesOrders/Order/OrderDetails/Product/.[@ProductID = 223]') = 1 ;
```

图 5-10 所示的统计信息显示，查询耗时 1.95 秒完成。

### 图 5-10. 使用 PATH 索引的查询结果

现在，让我们使用清单 5-7 中的脚本来删除 PATH 索引并再次运行查询。这次，只有主 XML 索引可用。

### 清单 5-7. 在没有 PATH 索引的情况下运行查询

```sql
DROP INDEX [SecondaryXmlIndex-OrderSummary-Path] ON Sales.OrderSummary ;
GO

DBCC DROPCLEANBUFFERS
DBCC FREEPROCCACHE
GO

SET STATISTICS TIME ON
GO

SELECT *
FROM Sales.OrderSummary
WHERE OrderSummary.exist('/SalesOrders/Order/OrderDetails/Product/.[@ProductID = 223]') = 1 ;
```

这一次，从图 5-11 的统计信息中我们可以看到，查询耗时超过 2.7 秒才完成。

### 图 5-11. 没有 PATH 索引的查询结果

最后，让我们使用清单 5-8 中的脚本来删除主 XML 索引，并在没有任何 XML 索引支持的情况下再次运行查询。

### 清单 5-8. 删除主 XML 索引并重新运行查询

```sql
DROP INDEX [PrimaryXmlIndex-OrderSummary-OrderSummary] ON Sales.OrderSummary ;
GO

DBCC DROPCLEANBUFFERS
DBCC FREEPROCCACHE
GO

SET STATISTICS TIME ON
GO

SELECT *
FROM Sales.OrderSummary
WHERE OrderSummary.exist('/SalesOrders/Order/OrderDetails/Product/.[@ProductID = 223]') = 1 ;
```

从图 5-12 的统计信息中你会注意到，查询执行时间现在已上升到超过 4 秒。虽然我们的表只有不到 700 行，并且你看到的结果会因机器性能而异，但这个例子说明了为何创建 XML 索引如此重要。

### 图 5-12. 没有 XML 索引的查询结果

## 小结

可以在 XML 列上创建专门的 XML 索引，以提升依赖于查询 XML 数据的查询性能。XML 索引有四种类型：主索引、辅助 PATH 索引、辅助 VALUE 索引和辅助 PROPERTY 索引。

除非表具有聚集主键（基于主键列构建的聚集索引），否则无法在 XML 列上创建主 XML 索引。除非 XML 列上已存在主 XML 索引，否则无法创建辅助 XML 索引。但是，可以在表填充数据之前创建 XML 索引。



