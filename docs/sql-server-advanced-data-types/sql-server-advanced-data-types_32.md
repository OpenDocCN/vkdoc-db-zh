# 第 3 章 使用 T-SQL 构建 XML

你可以看到，当使用 `AUTO` 模式时，`FOR XML` 子句会根据查询中的 `JOIN` 子句自动嵌套数据。每个元素都被分配了一个名称，该名称基于其检索来源表的表别名。与 `RAW` 模式一样，我们可以使用 `ROOT` 关键字来添加根节点，使文档结构良好。我们还可以使用 `ELEMENTS` 关键字使文档以元素为中心。

如果你仔细查看清单 3-11 中的文档，你会注意到生成的层次结构并不理想。`<Customers>` 被嵌套在 `<SalesOrders>` 下，而 `<LineItem>` 被嵌套在 `<Product>` 下。然而，在这个文档中，你自然期望的层次结构应该是 `<Customers>` `<SalesOrders>` `<LineItem>` `<Product>`。要实现这一点，我们必须重写查询，如清单 3-12 所示。

## 清单 3-12

重写查询以正确嵌套数据

```sql
SELECT
    Customers.CustomerName
    , SalesOrder.OrderDate
    , SalesOrder.OrderID
    , LineItem.Quantity
    , LineItem.UnitPrice
    , Product.StockItemName
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

请注意，列的顺序已经改变，`Customers` 表在 `SalesOrders` 表之前被引用，而 `SalesOrders` 表又在 `OrderLines` 表之前被引用。`StockItems` 表是最后一个被引用的。你还会注意到，我们无需更改联接顺序，即可生成清单 3-13 所示的文档。

## 清单 3-13

重写查询的结果



