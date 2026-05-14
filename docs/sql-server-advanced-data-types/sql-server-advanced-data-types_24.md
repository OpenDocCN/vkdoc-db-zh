# 第 3 章 使用 T-SQL 构建 XML

ON `LineItem.OrderID` = `SalesOrder.OrderID`

INNER JOIN `Warehouse.StockItems` `Product`

ON `Product.StockItemID` = `LineItem.StockItemID`

INNER JOIN `Sales.Customers` `Customers`

ON `Customers.CustomerID` = `SalesOrder.CustomerID`

WHERE `customers.CustomerName` = 'Agrita Abele'

FOR XML RAW, ROOT('SalesOrders') ;

生成的格式良好 XML 文档的部分输出可在`清单 3-5`中找到。

## `清单 3-5`. 带根节点的 XML 文档输出

```xml
<SalesOrders>
<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
OrderID="72787" StockItemName="Developer joke mug - when your hammer is C++ (Black)" Quantity="2" UnitPrice="13.00" />
<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
OrderID="72787" StockItemName="Developer joke mug - when your hammer is C++ (White)" Quantity="9" UnitPrice="13.00" />
<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
OrderID="73350" StockItemName="DBA joke mug - you might be a DBA if (Black)" Quantity="4" UnitPrice="13.00" />
<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
OrderID="73350" StockItemName="Developer joke mug - that's a hardware problem (Black)" Quantity="3" UnitPrice="13.00" />
<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
OrderID="72770" StockItemName="&quot;The Gu&quot; red shirt XML
tag t-shirt (Black) 5XL" Quantity="120" UnitPrice="18.00" />
<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
OrderID="72637" StockItemName="&quot;The Gu&quot; red shirt XML
tag t-shirt (Black) XXL" Quantity="24" UnitPrice="18.00" />
<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
OrderID="73350" StockItemName="Halloween zombie mask (Light Brown) S" Quantity="24" UnitPrice="18.00" />
<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
OrderID="72669" StockItemName="Halloween skull mask (Gray) M"
Quantity="60" UnitPrice="18.00" />
<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
OrderID="72787" StockItemName="Furry gorilla with big eyes slippers (Black) XL" Quantity="9" UnitPrice="32.00" />
<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
OrderID="73350" StockItemName="Tape dispenser (Blue)"
Quantity="90" UnitPrice="32.00" />
<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
OrderID="72713" StockItemName="Tape dispenser (Red)"
Quantity="20" UnitPrice="32.00" />
<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
OrderID="72770" StockItemName="Superhero action jacket (Blue) S" Quantity="8" UnitPrice="25.00" />
<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
OrderID="73350" StockItemName="RC toy sedan car with remote control (Red) 1/50 scale" Quantity="2" UnitPrice="25.00" />
<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
OrderID="72637" StockItemName="Superhero action jacket (Blue) 3XS" Quantity="1" UnitPrice="25.00" />
<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
OrderID="72770" StockItemName="Black and orange handle with care despatch tape 48mmx100m" Quantity="96" UnitPrice="4.10" />
<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
OrderID="72671" StockItemName="Alien officer hoodie (Black) 4XL" Quantity="6" UnitPrice="35.00" />
<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
OrderID="72637" StockItemName="Furry animal socks (Pink) S"
Quantity="96" UnitPrice="5.00" />
<row OrderDate="2016-05-18" CustomerName="Agrita Abele"
OrderID="72669" StockItemName="Permanent marker black 5mm nib (Black) 5mm" Quantity="84" UnitPrice="2.70" />
<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
OrderID="73340" StockItemName="Superhero action jacket (Blue) 5XL" Quantity="2" UnitPrice="34.00" />
<row OrderDate="2016-05-27" CustomerName="Agrita Abele"
OrderID="73356" StockItemName="Chocolate sharks 250g"
Quantity="192" UnitPrice="8.55" />
<row OrderDate="2016-05-19" CustomerName="Agrita Abele"
OrderID="72787" StockItemName="Novelty chilli chocolates
250g" Quantity="240" UnitPrice="8.55" />
```



