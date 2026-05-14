# 第 25 章 ■ 数据库工作负载优化

## 示例工作负载

要对 SQL Server 性能进行故障排查，你需要了解在服务器上执行的 SQL 工作负载。然后，你可以分析工作负载以确定性能不佳的原因和适用的优化步骤。理想情况下，你应该在面临性能问题的 SQL Server 上捕获工作负载。在本章中，你将使用一组查询来模拟一个示例工作负载，以便遵循上一节列出的优化步骤。你将使用的示例工作负载由好坏查询组合而成。

■ `注意` 我建议你恢复一个干净的`AdventureWorks2012`数据库副本，以便完全移除前面章节遗留的任何对象。

简单的测试工作负载由以下一组示例存储过程模拟；你使用第二个脚本在`AdventureWorks2012`数据库上执行它们：

```
USE AdventureWorks2012;

GO

CREATE PROCEDURE dbo.ShoppingCart
@ShoppingCartId VARCHAR(50)
AS
--提供购物车输出，包括行合计
SELECT sci.Quantity,
       p.ListPrice,
       p.ListPrice * sci.Quantity AS LineTotal,
       p.[Name]
FROM Sales.ShoppingCartItem AS sci
JOIN Production.Product AS p
  ON sci.ProductID = p.ProductID
WHERE sci.ShoppingCartID = @ShoppingCartId;

GO
```

```
CREATE PROCEDURE dbo.ProductBySalesOrder
@SalesOrderID INT
AS
/*提供特定销售订单的产品列表，
按修改日期提供行排序，但按产品名称排序*/
SELECT ROW_NUMBER() OVER (ORDER BY sod.ModifiedDate) AS LineNumber,
       p.[Name],
       sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod
  ON soh.SalesOrderID = sod.SalesOrderID
JOIN Production.Product AS p
  ON sod.ProductID = p.ProductID
WHERE soh.SalesOrderID = @SalesOrderID
ORDER BY p.[Name] ASC;

GO
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

