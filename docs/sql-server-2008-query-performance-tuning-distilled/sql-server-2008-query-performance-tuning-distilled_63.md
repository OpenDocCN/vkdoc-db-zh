# 27. 数据库工作负载优化

到目前为止，你已经了解了可能影响查询性能的多个方面、可用于分析查询性能的工具以及可用于提高查询性能的优化技术。接下来，你将学习如何应用这些信息来分析、诊断和优化数据库工作负载的性能。我将引导你完成一个调优过程，包括可能尝试一两个错误的方向，因此请耐心跟随我们导航这个过程。

在本章中，我将涵盖以下主题：

*   数据库工作负载的特征
*   数据库工作负载优化涉及的步骤
*   如何识别工作负载中的高开销查询
*   如何测量高开销查询的基线资源使用情况和性能
*   如何分析影响高开销查询性能的因素
*   如何应用技术来优化高开销查询
*   如何分析查询优化对整个工作负载的影响

## 工作负载优化基础

优化数据库工作负载通常符合 80/20 法则：80% 的工作负载消耗大约 20% 的服务器资源。尝试优化大部分工作负载的性能通常效率不高。因此，工作负载优化的第一步是找出消耗 80% 服务器资源的那 20% 的工作负载。

优化工作负载需要一组工具来衡量工作负载不同部分的资源消耗和响应时间。正如你在第 4 章和第 5 章中所见，SQL Server 提供了一套工具和实用程序来分析数据库工作负载和单个查询的性能。

除了使用这些工具外，了解如何运用不同技术来优化工作负载也很重要。关于工作负载优化需要记住的最重要一点是，没有一种优化技术能保证解决每一个性能问题。许多优化技术针对特定的数据库应用程序设计和数据库环境。因此，对于每一种优化技术，你都需要在应用前后测量工作负载每个部分（即每个单独查询）的性能。你可以使用第 26 章讨论的技术来实现这一点。

发现一种优化技术对工作负载的其他部分影响甚微，甚至产生负面影响，从而损害工作负载的整体性能，这种情况并不少见。例如，为优化 `SELECT` 语句而添加的非聚集索引可能会损害修改索引列值的 `UPDATE` 语句的性能。`UPDATE` 语句除了更新数据行外，还必须更新索引行。然而，正如第 6 章所演示的，有时索引也可以提高操作查询的性能。因此，提高特定查询的性能可能有益于也可能损害整个工作负载的性能。一如既往，你最好的行动方针是通过测试来验证任何假设。



## 工作负载优化步骤

优化数据库工作负载的过程遵循一系列特定的步骤。作为此过程的一部分，你将使用前面章节介绍的一套优化技术。由于每个性能问题都是一个新的挑战，你可以使用不同的优化技术组合来排查不同的性能问题。请记住，第一步始终是确保服务器配置良好且在可接受的限制范围内运行，正如第 2 章和第 3 章所定义的那样。

为了理解查询优化过程，你将使用一组查询来模拟一个样本工作负载。

查询调优的核心归结为几个步骤。

1.  确定要调优的查询。
2.  查看执行计划以了解资源使用情况和行为。
3.  修改查询或修改结构以提高性能。

大多数情况下，答案就是修改查询。简而言之，这就是进行查询调优所需做的全部。然而，这假设了你对系统有大量了解，并且你已经查看了诸如统计信息之类的历史数据。当你首次进行查询调优或在一个新系统上工作时，这个过程会详细得多。为了对调优查询所需的步骤给出一个全面而完整的定义，以下是你要执行的操作。这些是你在优化样本工作负载时将遵循的优化步骤：

1.  捕获工作负载。
2.  分析工作负载。
3.  识别开销最大/调用最频繁/运行时间最长的查询。
4.  量化开销最大查询的基线资源使用情况。
5.  确定总体资源使用情况。
6.  汇编关于资源使用的详细信息。
7.  分析和优化外部因素。
8.  分析索引的使用情况。
9.  分析应用程序使用的批处理级别选项。
10. 分析统计信息的有效性。
11. 评估碎片整理的必要性。
12. 分析开销最大查询的内部行为。
13. 分析查询执行计划。
14. 识别执行计划中开销最大的操作符。
15. 分析处理策略的有效性。
16. 优化开销最大的查询。
17. 分析更改对数据库工作负载的影响。
18. 在多个优化阶段中迭代进行。

正如第 1 章所解释的，性能调优是一个迭代过程。因此，你应该多次迭代执行性能优化步骤，直到达到期望的应用程序性能目标。一段时间后，你将需要重复该过程，以应对数据和数据库更改对工作负载造成的影响。此外，当你在一段时间内负责维护一台服务器时，你可能会跳过许多前面的步骤，因为你已经验证了事务方法、统计信息维护或其他步骤。你不必严格遵循这个列表。它旨在作为一个指南。关于调优查询所需的步骤的图形化表示，我建议你参考第 1 章。

## 样本工作负载

要对 SQL Server 性能问题进行故障排除，你需要了解在服务器上执行的 SQL 工作负载。然后，你可以分析工作负载以确定性能不佳的原因和适用的优化步骤。理想情况下，你应该在面临性能问题的 SQL Server 上捕获工作负载。在本章中，你将使用一组查询来模拟一个样本工作负载，以便你可以遵循上一节列出的优化步骤。你将使用的样本工作负载由良好查询和不良查询组合而成。

### 注意

我建议你恢复一个干净的 `AdventureWorks2017` 数据库副本，以便完全删除先前章节留下的任何工件。

简单的测试工作负载由以下一组示例存储过程模拟；你使用第二个脚本在 `AdventureWorks2017` 数据库上执行它们：

```sql
USE AdventureWorks2017;
GO
CREATE OR ALTER PROCEDURE dbo.ShoppingCart @ShoppingCartId VARCHAR(50)
AS
--provides the output from the shopping cart including the line total
SELECT sci.Quantity,
p.ListPrice,
p.ListPrice * sci.Quantity AS LineTotal,
p.Name
FROM Sales.ShoppingCartItem AS sci
JOIN Production.Product AS p
ON sci.ProductID = p.ProductID
WHERE sci.ShoppingCartID = @ShoppingCartId;
GO
CREATE OR ALTER PROCEDURE dbo.ProductBySalesOrder @SalesOrderID INT
AS
/*provides a list of products from a particular sales order,
and provides line ordering by modified date but ordered
by product name*/
SELECT ROW_NUMBER() OVER (ORDER BY sod.ModifiedDate) AS LineNumber,
p.Name,
sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod
ON soh.SalesOrderID = sod.SalesOrderID
JOIN Production.Product AS p
ON sod.ProductID = p.ProductID
WHERE soh.SalesOrderID = @SalesOrderID
ORDER BY p.Name ASC;
GO
CREATE OR ALTER PROCEDURE dbo.PersonByFirstName @FirstName NVARCHAR(50)
AS
--gets anyone by first name from the Person table
SELECT p.BusinessEntityID,
p.Title,
p.LastName,
p.FirstName,
p.PersonType
FROM Person.Person AS p
WHERE p.FirstName = @FirstName;
GO
CREATE OR ALTER PROCEDURE dbo.ProductTransactionsSinceDate
@LatestDate DATETIME,
@ProductName NVARCHAR(50)
AS
--Gets the latest transaction against
--all products that have a transaction
SELECT p.Name,
th.ReferenceOrderID,
th.ReferenceOrderLineID,
th.TransactionType,
th.Quantity
FROM Production.Product AS p
JOIN Production.TransactionHistory AS th
ON p.ProductID = th.ProductID
AND th.TransactionID = (   SELECT TOP (1)
th2.TransactionID
FROM Production.TransactionHistory AS th2
WHERE th2.ProductID = p.ProductID
ORDER BY th2.TransactionID DESC)
WHERE th.TransactionDate > @LatestDate
AND p.Name LIKE @ProductName;
GO
CREATE OR ALTER PROCEDURE dbo.PurchaseOrderBySalesPersonName
@LastName NVARCHAR(50),
@VendorID INT = NULL
AS
SELECT poh.PurchaseOrderID,
poh.OrderDate,
pod.LineTotal,
p.Name AS ProductName,
e.JobTitle,
per.LastName + ', ' + per.FirstName AS SalesPerson,
poh.VendorID
FROM Purchasing.PurchaseOrderHeader AS poh
JOIN Purchasing.PurchaseOrderDetail AS pod
ON poh.PurchaseOrderID = pod.PurchaseOrderID
JOIN Production.Product AS p
ON pod.ProductID = p.ProductID
JOIN HumanResources.Employee AS e
ON poh.EmployeeID = e.BusinessEntityID
JOIN Person.Person AS per
ON e.BusinessEntityID = per.BusinessEntityID
WHERE per.LastName LIKE @LastName
AND poh.VendorID = COALESCE(@VendorID,
poh.VendorID)
ORDER BY per.LastName,
per.FirstName;
GO
CREATE OR ALTER PROCEDURE dbo.TotalSalesByProduct @ProductID INT
AS
--retrieve aggregation of sales based on a productid
SELECT SUM((isnull((sod.UnitPrice*((1.0)-sod.UnitPriceDiscount))*sod.OrderQty,(0.0)))) AS TotalSales,
AVG(sod.OrderQty) AS AverageQty,
AVG(sod.UnitPrice) AS AverageUnitPrice,
SUM(sod.LineTotal)
FROM Sales.SalesOrderDetail AS sod
WHERE sod.ProductID = @ProductID
GROUP BY sod.ProductID;
GO
```

请记住，这只是一个说明性的示例，而不是实际施加在服务器上的真实负载。真实的存储过程通常要复杂得多，但我们在设置模拟生产负载方面只能投入这么多篇幅。在这些过程就绪后，你可以使用以下脚本执行它们：


```
EXEC dbo.PurchaseOrderBySalesPersonName @LastName = 'Hill%';
GO
EXEC dbo.ShoppingCart @ShoppingCartId = '20621';
GO
EXEC dbo.ProductBySalesOrder @SalesOrderID = 43867;
GO
EXEC dbo.PersonByFirstName @FirstName = 'Gretchen';
GO
EXEC dbo.ProductTransactionsSinceDate @LatestDate = '9/1/2004',
@ProductName = 'Hex Nut%';
GO
EXEC dbo.PurchaseOrderBySalesPersonName @LastName = 'Hill%',
@VendorID = 1496;
GO
EXEC dbo.TotalSalesByProduct @ProductID = 707;
GO
```

我知道我在重复自己，但我想说清楚。这是一个极其简单的工作负载，仅用于说明流程。在典型的系统中，您将看到跨越更多存储过程和即席查询的成百上千个额外调用。然而，尽管它很简单，这个示例工作负载却包含了通常在`SQL Server`上执行的不同类型的查询。

*   使用聚合函数的查询

*   仅检索单行或少量行的点查询

*   连接多个表的查询

*   检索窄范围行的查询

*   执行额外结果集处理的查询，例如提供排序输出

第一个优化步骤是捕获工作负载，即查看这些查询的性能表现，如下一节所述。

