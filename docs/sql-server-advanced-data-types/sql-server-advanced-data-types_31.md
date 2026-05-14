# 第 3 章 使用 T-SQL 构建 XML

```xml
<LineItem Quantity="4" UnitPrice="13.00" />

</Product>

<Product StockItemName="开发者玩笑马克杯 - 那是硬件问题 (黑色)">

<LineItem Quantity="3" UnitPrice="13.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="&quot;The Gu&quot; 红色衬衫 XML 标签 T 恤 (黑色) 5XL">

<LineItem Quantity="120" UnitPrice="18.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="&quot;The Gu&quot; 红色衬衫 XML 标签 T 恤 (黑色) XXL">

<LineItem Quantity="24" UnitPrice="18.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="万圣节僵尸面具 (浅棕色) S">

<LineItem Quantity="24" UnitPrice="18.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="万圣节骷髅面具 (灰色) M">

<LineItem Quantity="60" UnitPrice="18.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="毛茸茸大眼猩猩拖鞋 (黑色) XL">

<LineItem Quantity="9" UnitPrice="32.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="胶带座 (蓝色)">

<LineItem Quantity="90" UnitPrice="32.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72713">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="胶带座 (红色)">

<LineItem Quantity="20" UnitPrice="32.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="超级英雄动作夹克 (蓝色) S">

<LineItem Quantity="8" UnitPrice="25.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="遥控玩具轿车 (红色) 1/50 比例">

<LineItem Quantity="2" UnitPrice="25.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="超级英雄动作夹克 (蓝色) 3XS">

<LineItem Quantity="1" UnitPrice="25.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="黑橙色小心轻放发货胶带 48mmx100m">

<LineItem Quantity="96" UnitPrice="4.10" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72671">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="外星军官连帽衫 (黑色) 4XL">

<LineItem Quantity="6" UnitPrice="35.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="毛茸茸动物袜子 (粉色) S">

<LineItem Quantity="96" UnitPrice="5.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="永久性记号笔黑色 5mm 笔尖 (黑色) 5mm">

<LineItem Quantity="84" UnitPrice="2.70" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73340">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="超级英雄动作夹克 (蓝色) 5XL">

<LineItem Quantity="2" UnitPrice="34.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="巧克力鲨鱼 250g">
```

```xml
<LineItem Quantity="192" UnitPrice="8.55" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Novelty chilli chocolates 250g">

<LineItem Quantity="240" UnitPrice="8.55" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72671">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Packing knife with metal insert
blade (Yellow) 9mm">

<LineItem Quantity="35" UnitPrice="1.89" />

</Product>

<Product StockItemName="Shipping carton (Brown)
480x270x320mm">

<LineItem Quantity="150" UnitPrice="2.74" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Shipping carton (Brown) 356x229x229mm">

<LineItem Quantity="175" UnitPrice="1.14" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Ride on big wheel monster truck
(Black) 1/12 scale">

<LineItem Quantity="4" UnitPrice="345.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="10 mm Anti static bubble wrap
(Blue) 10m">

<LineItem Quantity="70" UnitPrice="26.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73340">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="32 mm Double sided bubble wrap 50m">

<LineItem Quantity="20" UnitPrice="112.00" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Office cube periscope (Black)">

<LineItem Quantity="20" UnitPrice="18.50" />

</Product>

</Customers>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<Customers CustomerName="Agrita Abele">

<Product StockItemName="Ride on vintage American toy coupe
(Black) 1/12 scale">

<LineItem Quantity="10" UnitPrice="285.00" />

</Product>

</Customers>

</SalesOrder>
```

