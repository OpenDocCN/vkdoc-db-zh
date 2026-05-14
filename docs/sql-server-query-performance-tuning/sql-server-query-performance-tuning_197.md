# 第 25 章 ■ 数据库工作负载优化

```
FROM Purchasing.PurchaseOrderHeader AS poh
JOIN Purchasing.PurchaseOrderDetail AS pod
  ON poh.PurchaseOrderID = pod.PurchaseOrderID
JOIN Production.Product AS p
  ON pod.ProductID = p.ProductID
JOIN HumanResources.Employee AS e
  ON poh.EmployeeID = e.BusinessEntityID
JOIN Person.Person AS per
  ON e.BusinessEntityID = per.BusinessEntityID
WHERE per.LastName LIKE @LastName AND
      poh.VendorID = COALESCE(@VendorID, poh.VendorID)
ORDER BY per.LastName,
         per.FirstName;

GO
```

创建这些过程后，你可以使用以下脚本执行它们：

```
EXEC dbo.PurchaseOrderBySalesPersonName
@LastName = 'Hill%';

GO
```

```
EXEC dbo.ShoppingCart
@ShoppingCartID = '20621';

GO
```

```
EXEC dbo.ProductBySalesOrder
@SalesOrderID = 43867;

GO
```

```
EXEC dbo.PersonByFirstName
@FirstName = 'Gretchen';

GO
```

```
EXEC dbo.ProductTransactionsSinceDate
@LatestDate = '9/1/2004',
@ProductName = 'Hex Nut%';

GO
```

```
EXEC dbo.PurchaseOrderBySalesPersonName
@LastName = 'Hill%',
@VendorID = 1496;

GO
```


