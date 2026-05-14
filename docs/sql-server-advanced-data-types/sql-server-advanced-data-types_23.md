# 第 3 章：使用 T-SQL 构建 XML

```
<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="73350" StockItemName="Tape dispenser (Blue)"
     Quantity="90" UnitPrice="32.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72713" StockItemName="Tape dispenser (Red)"
     Quantity="20" UnitPrice="32.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
     OrderID="72770" StockItemName="Superhero action jacket (Blue) S"
     Quantity="8" UnitPrice="25.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
     OrderID="73350" StockItemName="RC toy sedan car with remote control (Red) 1/50 scale"
     Quantity="2" UnitPrice="25.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72637" StockItemName="Superhero action jacket (Blue) 3XS"
     Quantity="1" UnitPrice="25.00" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
     OrderID="72770" StockItemName="Black and orange handle with care despatch tape 48mmx100m"
     Quantity="96" UnitPrice="4.10" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72671" StockItemName="Alien officer hoodie (Black) 4XL"
     Quantity="6" UnitPrice="35.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72637" StockItemName="Furry animal socks (Pink) S"
     Quantity="96" UnitPrice="5.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72669" StockItemName="Permanent marker black 5mm nib (Black) 5mm"
     Quantity="84" UnitPrice="2.70" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
     OrderID="73340" StockItemName="Superhero action jacket (Blue) 5XL"
     Quantity="2" UnitPrice="34.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
     OrderID="73356" StockItemName="Chocolate sharks 250g"
     Quantity="192" UnitPrice="8.55" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
     OrderID="72787" StockItemName="Novelty chilli chocolates 250g"
     Quantity="240" UnitPrice="8.55" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72671" StockItemName="Packing knife with metal insert blade (Yellow) 9mm"
     Quantity="35" UnitPrice="1.89" /> 53

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72671" StockItemName="Shipping carton (Brown) 480x270x320mm"
     Quantity="150" UnitPrice="2.74" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
     OrderID="72770" StockItemName="Shipping carton (Brown) 356x229x229mm"
     Quantity="175" UnitPrice="1.14" />

<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
     OrderID="72787" StockItemName="Ride on big wheel monster truck (Black) 1/12 scale"
     Quantity="4" UnitPrice="345.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72669" StockItemName="10 mm Anti static bubble wrap (Blue) 10m"
     Quantity="70" UnitPrice="26.00" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
     OrderID="73340" StockItemName="32 mm Double sided bubble wrap 50m"
     Quantity="20" UnitPrice="112.00" />

<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
     OrderID="72669" StockItemName="Office cube periscope (Black)"
     Quantity="20" UnitPrice="18.50" />

<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
     OrderID="73356" StockItemName="Ride on vintage American toy coupe (Black) 1/12 scale"
     Quantity="10" UnitPrice="285.00" />
```

我们应该注意到关于此 XML 实例的第一点是，它是一个 XML 片段，而不是一个格式良好的 XML 文档，因为它没有根节点。`<row>` 元素不能成为根节点，因为它会重复。这意味着我们无法根据模式来验证 XML。

因此，在使用 `FOR XML` 子句时，你应考虑使用 `ROOT` 关键字。这将强制在文档中创建一个根元素，其名称由你指定。清单 3-4 展示了这一点。

## 清单 3-4：添加根节点

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
```


