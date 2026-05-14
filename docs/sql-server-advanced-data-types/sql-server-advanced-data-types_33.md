# 第 3 章 使用 T-SQL 构建 XML

```xml
<Customers CustomerName="Agrita Abele">

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<LineItem Quantity="2" UnitPrice="13.00">

<Product StockItemName="开发者笑话马克杯——当你的锤子是 C++时（黑色）" />

</LineItem>

<LineItem Quantity="9" UnitPrice="13.00">

<Product StockItemName="开发者笑话马克杯——当你的锤子是 C++时（白色）" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<LineItem Quantity="4" UnitPrice="13.00">

<Product StockItemName="数据库管理员笑话马克杯——你可能是个数据库管理员（黑色）" />

</LineItem>

<LineItem Quantity="3" UnitPrice="13.00">

<Product StockItemName="开发者笑话马克杯——那是硬件问题（黑色）" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<LineItem Quantity="120" UnitPrice="18.00">

<Product StockItemName="&quot;The Gu&quot; 红色衬衫 XML 标签 T 恤（黑色）5XL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<LineItem Quantity="24" UnitPrice="18.00">

<Product StockItemName="&quot;The Gu&quot; 红色衬衫 XML 标签 T 恤（黑色）XXL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<LineItem Quantity="24" UnitPrice="18.00">

<Product StockItemName="万圣节僵尸面具（浅棕色）S" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<LineItem Quantity="60" UnitPrice="18.00">

<Product StockItemName="万圣节骷髅面具（灰色）M" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<LineItem Quantity="9" UnitPrice="32.00">

<Product StockItemName="毛绒大眼睛大猩猩拖鞋（黑色）XL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<LineItem Quantity="90" UnitPrice="32.00">

<Product StockItemName="胶带座（蓝色）" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72713">

<LineItem Quantity="20" UnitPrice="32.00">

<Product StockItemName="胶带座（红色）" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<LineItem Quantity="8" UnitPrice="25.00">

<Product StockItemName="超级英雄动作夹克（蓝色）S" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73350">

<LineItem Quantity="2" UnitPrice="25.00">

<Product StockItemName="遥控玩具轿车（红色）1/50 比例" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<LineItem Quantity="1" UnitPrice="25.00">

<Product StockItemName="超级英雄动作夹克（蓝色）3XS" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72770">

<LineItem Quantity="96" UnitPrice="4.10">

<Product StockItemName="黑橙相间小心轻放发货胶带 48mmx100m" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72671">

<LineItem Quantity="6" UnitPrice="35.00">

<Product StockItemName="外星军官连帽衫（黑色）4XL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72637">

<LineItem Quantity="96" UnitPrice="5.00">

<Product StockItemName="毛绒动物袜子（粉色）S" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72669">

<LineItem Quantity="84" UnitPrice="2.70">

<Product StockItemName="永久性记号笔黑色 5mm 笔尖（黑色）5mm" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73340">

<LineItem Quantity="2" UnitPrice="34.00">

<Product StockItemName="超级英雄动作夹克（蓝色）5XL" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-27" OrderID="73356">

<LineItem Quantity="192" UnitPrice="8.55">

<Product StockItemName="巧克力鲨鱼 250 克" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-19" OrderID="72787">

<LineItem Quantity="240" UnitPrice="8.55">

<Product StockItemName="新奇辣椒巧克力 250 克" />

</LineItem>

</SalesOrder>

<SalesOrder OrderDate="2016-05-18" OrderID="72671">
```



