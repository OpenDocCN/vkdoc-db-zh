# 第 3 章：使用 T-SQL 构建 XML

```xml
<ProductName>"The Gu" 红色衬衫 XML 标签 T 恤（黑色）5XL</ProductName>
<OrderQty>120</OrderQty>
</LineItem>
<LineItem>
<LineTotal>25.00</LineTotal>
<ProductName>超级英雄动感夹克（蓝色）S</ProductName>
<OrderQty>8</OrderQty>
</LineItem>
</OrderDetails>
```

```xml
<OrderDetails SalesOrderID="72787" OrderDate="2016-05-19" CustomerID="1061">
<SalesPerson SalesPersonName="Jack Potter" />
<LineItem>
<LineTotal>8.55</LineTotal>
<ProductName>新奇辣椒巧克力 250 克</ProductName>
<OrderQty>240</OrderQty>
</LineItem>
<LineItem>
<LineTotal>13.00</LineTotal>
<ProductName>开发者玩笑马克杯 - 当你的锤子是 C++（白色）</ProductName>
<OrderQty>9</OrderQty>
</LineItem>
<LineItem>
<LineTotal>13.00</LineTotal>
<ProductName>开发者玩笑马克杯 - 当你的锤子是 C++（黑色）</ProductName>
<OrderQty>2</OrderQty>
</LineItem>
<LineItem>
<LineTotal>32.00</LineTotal>
<ProductName>毛茸茸的大眼猩猩拖鞋（黑色）XL</ProductName>
<OrderQty>9</OrderQty>
</LineItem>
<LineItem>
<LineTotal>345.00</LineTotal>
<ProductName>大轮怪兽卡车（黑色）1/12 比例</ProductName>
<OrderQty>4</OrderQty>
</LineItem>
</OrderDetails>
```

```xml
<OrderDetails SalesOrderID="73340" OrderDate="2016-05-27" CustomerID="1061">
<SalesPerson SalesPersonName="Taj Shand" />
<LineItem>
<LineTotal>34.00</LineTotal>
<ProductName>超级英雄动感夹克（蓝色）5XL</ProductName>
<OrderQty>2</OrderQty>
</LineItem>
<LineItem>
<LineTotal>112.00</LineTotal>
<ProductName>32 毫米 双面气泡膜 50 米</ProductName>
<OrderQty>20</OrderQty>
</LineItem>
</OrderDetails>
```

```xml
<OrderDetails SalesOrderID="73350" OrderDate="2016-05-27" CustomerID="1061">
<SalesPerson SalesPersonName="Sophia Hinton" />
<LineItem>
<LineTotal>13.00</LineTotal>
<ProductName>DBA 玩笑马克杯 - 如果你符合这些你可能就是 DBA（黑色）</ProductName>
<OrderQty>4</OrderQty>
</LineItem>
<LineItem>
<LineTotal>13.00</LineTotal>
<ProductName>开发者玩笑马克杯 - 那是硬件问题（黑色）</ProductName>
<OrderQty>3</OrderQty>
</LineItem>
<LineItem>
<LineTotal>18.00</LineTotal>
<ProductName>万圣节僵尸面具（浅棕色）S</ProductName>
<OrderQty>24</OrderQty>
</LineItem>
<LineItem>
<LineTotal>25.00</LineTotal>
<ProductName>遥控玩具轿车（红色）1/50 比例</ProductName>
<OrderQty>2</OrderQty>
</LineItem>
<LineItem>
<LineTotal>32.00</LineTotal>
<ProductName>胶带座（蓝色）</ProductName>
<OrderQty>90</OrderQty>
</LineItem>
</OrderDetails>
```

```xml
<OrderDetails SalesOrderID="73356" OrderDate="2016-05-27" CustomerID="1061">
<SalesPerson SalesPersonName="Anthony Grosse" />
<LineItem>
<LineTotal>8.55</LineTotal>
<ProductName>巧克力鲨鱼 250 克</ProductName>
<OrderQty>192</OrderQty>
</LineItem>
<LineItem>
<LineTotal>285.00</LineTotal>
<ProductName>复古美式玩具双门轿车（黑色）1/12 比例</ProductName>
<OrderQty>10</OrderQty>
</LineItem>
</OrderDetails>
</SalesOrders>
```

## 总结

`FOR XML` 子句可用于从 T-SQL 查询构建 XML 数据。`FOR XML` 子句可以在四种模式下使用：`RAW`、`AUTO`、`PATH` 和 `EXPLICIT`。

当在 `RAW` 模式下使用时，`FOR XML` 将生成一个扁平的 XML 实例，没有嵌套。它使用非常简单，并允许 XML 实例是以属性为中心或以元素为中心的。通过添加根节点，它也可以生成格式良好的 XML。然而，它对格式的控制非常有限。

`FOR XML AUTO` 使用起来非常简单，允许开发者创建嵌套的 XML 实例。与 `FOR XML RAW` 一样，`FOR XML AUTO` 可以生成以元素为中心或以属性为中心的文档，通过添加根节点可以使其格式规范。XML 文档中的数据将根据表连接自动嵌套。节点的名称将源自表别名。

当在 `PATH` 模式下使用时，`FOR XML` 提供了对 XML 文档布局更细粒度的控制。通过包含或省略 `@` 前缀，可以将每列定义为元素或属性。


## 第 3 章 使用 T-SQL 构建 XML

`FOR XML EXPLICIT` 模式是 `FOR XML` 模式中最复杂但也最强大的一种。它的实现方式是将每个复杂节点定义在独立的查询中，并通过 `UNION ALL` 子句连接这些查询。

层次结构位置可以通过列别名来定义，复杂的嵌套需求可以通过子查询来实现。

层次结构定位通过 `tag` 和 `parent` 列来定义，它们始终分别是 `SELECT` 列表中的第一列和第二列。列别名允许声明节点名称，并定义每个节点应以元素为中心还是以属性为中心。

