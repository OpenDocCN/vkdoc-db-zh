# 第 3 章 使用 T-SQL 构建 XML

## 示例 XML 结构

```xml
<Order>
  <OrderHeader>
    <CustomerName>Agrita Abele</CustomerName>
    <OrderDate>2016-05-27</OrderDate>
    <OrderID>73350</OrderID>
  </OrderHeader>
  <OrderDetails>
    <Product ProductID="21" ProductName="DBA joke mug - you might be a DBA if (Black)" Price="13.00" Qty="4" />
    <Product ProductID="33" ProductName="Developer joke mug - that's a hardware problem (Black)" Price="13.00" Qty="3" />
    <Product ProductID="142" ProductName="Halloween zombie mask (Light Brown) S" Price="18.00" Qty="24" />
    <Product ProductID="205" ProductName="Tape dispenser (Blue)" Price="32.00" Qty="90" />
    <Product ProductID="59" ProductName="RC toy sedan car with remote control (Red) 1/50 scale" Price="25.00" Qty="2" />
  </OrderDetails>
</Order>
<Order>
  <OrderHeader>
    <CustomerName>Agrita Abele</CustomerName>
    <OrderDate>2016-05-27</OrderDate>
    <OrderID>73356</OrderID>
  </OrderHeader>
  <OrderDetails>
    <Product ProductID="225" ProductName="Chocolate sharks 250g" Price="8.55" Qty="192" />
    <Product ProductID="74" ProductName="Ride on vintage American toy coupe (Black) 1/12 scale" Price="285.00" Qty="10" />
  </OrderDetails>
</Order>
```

使用 `FOR XML PATH` 来构建 XML 文档的查询一开始可能看起来有点复杂，但一旦你理解了原理，它们就变得相当简单。让我们检查每个需求以及如何实现所需的形状。

首先，我们需要一个名为 `<SalesOrders>` 的根节点，并且还需要将 `<row>` 元素重命名为 `<Order>`。这可以通过与我们探索 `RAW` 模式时相同的方式实现，使用 `<row>` 元素的可选参数以及 `FOR XML` 子句中的 `ROOT` 关键字和根节点，如清单 3-15 所示。

### 清单 3-15. 使用 `FOR XML PATH` 创建根节点并命名 `<row>` 元素

```sql
FOR XML PATH('Order'), ROOT ('SalesOrders') ;
```

下一个需求是创建一个名为 `<OrderHeader>` 的复杂元素，它将包含 `<CustomerName>`、`<OrderDate>` 和 `<OrderID>` 元素。这可以通过指定元素的路径以及元素名称来实现。如清单 3-16 所示，使用此技术时，`/` 字符表示层级结构中向下一步。

### 清单 3-16. 创建层级结构

```sql
SELECT
    CustomerName 'OrderHeader/CustomerName'
    , OrderDate 'OrderHeader/OrderDate'
    , OrderID 'OrderHeader/OrderID'
```

要创建嵌套的行项目，我们必须使用返回 `XML` 数据类型的子查询。为了使该过程正常工作，我们必须在子查询的 `FOR XML` 子句中使用 `TYPE` 关键字。这将导致结果以原生 `XML` 形式返回到外部查询，以便可以在服务器端进行处理。不使用此关键字将导致某些字符被控制字符序列替换。

因为我们希望将值存储为名为 `<Product>` 元素的属性，所以我们必须将 `<row>` 元素重命名，并在列别名前添加 `@` 符号。最后，为了确保 `<Product>` 元素嵌套在名为 `<OrderDetails>` 的元素下，我们将使用 `OrderDetails` 作为子查询返回的列的别名，如清单 3-17 所示。

### 清单 3-17. `OrderDetails` 子查询

```sql
(
    SELECT
        LineItems2.StockItemID '@ProductID'
        , StockItems.StockItemName '@ProductName'
        , LineItems2.UnitPrice '@Price'
        , Quantity '@Qty'
    FROM Sales.OrderLines LineItems2
    INNER JOIN Warehouse.StockItems StockItems
        ON LineItems2.StockItemID = StockItems.StockItemID
    WHERE LineItems2.OrderID = Base.OrderID
    FOR XML PATH('Product'), TYPE
) 'OrderDetails'
```

清单 3-18 综合了我们讨论的各个方面，并以所需格式返回一个 XML 文档。

### 清单 3-18. 综合示例

```sql
SELECT
```


