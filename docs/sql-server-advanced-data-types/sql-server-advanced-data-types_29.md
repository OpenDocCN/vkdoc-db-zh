# 第 3 章 使用 T-SQL 构建 XML

### 清单 3-10.

使用 `FOR XML AUTO`

```sql
SELECT
SalesOrder.OrderDate
, Customers.CustomerName
, SalesOrder.OrderID
, Product.StockItemName
, LineItem.Quantity
, LineItem.UnitPrice
FROM Sales.Orders SalesOrder
INNER JOIN Sales.OrderLines LineItem
ON LineItem.OrderID = SalesOrder.OrderID
INNER JOIN Warehouse.StockItems Product
ON Product.StockItemID = LineItem.StockItemID
INNER JOIN Sales.Customers Customers
ON Customers.CustomerID = SalesOrder.CustomerID
WHERE customers.CustomerName = 'Agrita Abele'
FOR XML AUTO ;
```

此查询返回的 XML 片段可部分见于 `清单 3-11`。

### 清单 3-11.

使用 `FOR XML AUTO` 的结果

```xml
<SalesOrder OrderDate="2016-05-19" OrderID="72787">
<Customers CustomerName="Agrita Abele">
<Product StockItemName="开发者玩笑马克杯 - 当你的锤子是 C++ (黑色)">
<LineItem Quantity="2" UnitPrice="13.00" />
</Product>
```

