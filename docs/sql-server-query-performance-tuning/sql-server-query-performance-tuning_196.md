# 第 25 章 ■ 数据库工作负载优化

```
CREATE PROCEDURE dbo.PersonByFirstName
@FirstName NVARCHAR(50)
AS
--从 Person 表中按名字获取任何人
SELECT p.BusinessEntityID,
       p.Title,
       p.LastName,
       p.FirstName,
       p.PersonType
FROM Person.Person AS p
WHERE p.FirstName = @FirstName;

GO
```

```
CREATE PROCEDURE dbo.ProductTransactionsSinceDate
@LatestDate DATETIME,
@ProductName NVARCHAR(50)
AS
--获取所有有交易的产品的最新交易
SELECT p.Name,
       th.ReferenceOrderID,
       th.ReferenceOrderLineID,
       th.TransactionType,
       th.Quantity
FROM Production.Product AS p
JOIN Production.TransactionHistory AS th
  ON p.ProductID = th.ProductID AND
     th.TransactionID = (SELECT TOP (1)
                            th2.TransactionID
                         FROM Production.TransactionHistory th2
                         WHERE th2.ProductID = p.ProductID
                         ORDER BY th2.TransactionID DESC
                        )
WHERE th.TransactionDate > @LatestDate AND
      p.Name LIKE @ProductName;

GO
```

```
ALTER PROCEDURE dbo.PurchaseOrderBySalesPersonName
@LastName NVARCHAR(50),
@VendorID INT = NULL
AS
SELECT poh.PurchaseOrderID,
       poh.OrderDate,
       pod.LineTotal,
       p.[Name] AS ProductName,
       e.JobTitle,
       per.LastName + ', ' + per.FirstName AS SalesPerson,
       poh.VendorID
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

