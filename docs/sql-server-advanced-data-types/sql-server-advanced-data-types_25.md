# 第 3 章 用 T-SQL 构建 XML

`<row OrderDate="2016-05-18" CustomerName="Agrita Abele" OrderID="72671" StockItemName="Packing knife with metal insert blade (Yellow) 9mm" Quantity="35" UnitPrice="1.89" /> 57`

`<row OrderDate="2016-05-18" CustomerName="Agrita Abele" OrderID="72671" StockItemName="Shipping carton (Brown) 480x270x320mm" Quantity="150" UnitPrice="2.74" />`

`<row OrderDate="2016-05-19" CustomerName="Agrita Abele" OrderID="72770" StockItemName="Shipping carton (Brown) 356x229x229mm" Quantity="175" UnitPrice="1.14" />`

`<row OrderDate="2016-05-19" CustomerName="Agrita Abele" OrderID="72787" StockItemName="Ride on big wheel monster truck (Black) 1/12 scale" Quantity="4" UnitPrice="345.00" />`

`<row OrderDate="2016-05-18" CustomerName="Agrita Abele" OrderID="72669" StockItemName="10 mm Anti static bubble wrap (Blue) 10m" Quantity="70" UnitPrice="26.00" />`

`<row OrderDate="2016-05-27" CustomerName="Agrita Abele" OrderID="73340" StockItemName="32 mm Double sided bubble wrap 50m" Quantity="20" UnitPrice="112.00" />`

`<row OrderDate="2016-05-18" CustomerName="Agrita Abele" OrderID="72669" StockItemName="Office cube periscope (Black)" Quantity="20" UnitPrice="18.50" />`

`<row OrderDate="2016-05-27" CustomerName="Agrita Abele" OrderID="73356" StockItemName="Ride on vintage American toy coupe (Black) 1/12 scale" Quantity="10" UnitPrice="285.00" />`

`</SalesOrders>`

关于该文档，另一个需要注意的重要点是它是完全扁平的。不存在嵌套。这意味着文档的粒度位于行项目级别，这并不十分合理。

同样值得注意的是，所有数据都包含在属性中，而不是元素内。我们可以通过在`FOR XML`子句中使用`ELEMENTS`关键字来改变这种行为。`ELEMENTS`关键字将使所有数据都包含在子元素内，而不是属性中。这一点在清单 3-6 所示的修改后的查询中得到了演示。

### 清单 3-6. 使用 `ELEMENTS` 关键字

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
FOR XML RAW, ELEMENTS, ROOT('SalesOrders') ;
```

返回的格式良好的 XML 文档可以在清单 3-7 中看到。

### 清单 3-7. 使用 `ELEMENTS` 关键字

```xml
<SalesOrders>
    <row>
        <OrderDate>2016-05-19</OrderDate>
        <CustomerName>Agrita Abele</CustomerName>
        <OrderID>72787</OrderID>
        <StockItemName>Developer joke mug - when your hammer is C++ (Black)</StockItemName>
        <Quantity>2</Quantity>
        <UnitPrice>13.00</UnitPrice>
    </row>
    <row>
        <OrderDate>2016-05-19</OrderDate>
        <CustomerName>Agrita Abele</CustomerName>
        <OrderID>72787</OrderID>
        <StockItemName>Developer joke mug - when your hammer is C++ (White)</StockItemName>
        <Quantity>9</Quantity>
        <UnitPrice>13.00</UnitPrice>
    </row>
    <row>
        <OrderDate>2016-05-27</OrderDate>
        <CustomerName>Agrita Abele</CustomerName>
        <OrderID>73350</OrderID>
        <StockItemName>DBA joke mug - you might be a DBA if (Black)</StockItemName>
        <Quantity>4</Quantity>
        <UnitPrice>13.00</UnitPrice>
    </row>
    <row>
        <OrderDate>2016-05-27</OrderDate>
        <CustomerName>Agrita Abele</CustomerName>
        <OrderID>73350</OrderID>
        <StockItemName>Developer joke mug - that's a hardware problem (Black)</StockItemName>
        <Quantity>3</Quantity>
        <UnitPrice>13.00</UnitPrice>
    </row>
    <row>
        <OrderDate>2016-05-19</OrderDate>
        <CustomerName>Agrita Abele</CustomerName>
        <OrderID>72770</OrderID>
        <StockItemName>"The Gu" red shirt XML tag t-shirt (Black) 5XL</StockItemName>
        <Quantity>120</Quantity>
        <UnitPrice>18.00</UnitPrice>
    </row>
    <row>
        <OrderDate>2016-05-18</OrderDate>
        <CustomerName>Agrita Abele</CustomerName>
        <OrderID>72637</OrderID>
```



