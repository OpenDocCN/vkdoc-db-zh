# 第 3 章 使用 T-SQL 构建 XML

`<LineItem>`
`<OrderDate>2016-05-19</OrderDate>`
`<CustomerName>Agrita Abele</CustomerName>`
`<OrderID>72770</OrderID>`
`<StockItemName>运输纸箱（棕色）356x229x229 毫米</StockItemName>`
`<Quantity>175</Quantity>`
`<UnitPrice>1.14</UnitPrice>`
`</LineItem>`

`<LineItem>`
`<OrderDate>2016-05-19</OrderDate>`
`<CustomerName>Agrita Abele</CustomerName>`
`<OrderID>72787</OrderID>`
`<StockItemName>大轮怪兽卡车玩具（黑色）1:12 比例</StockItemName>`
`<Quantity>4</Quantity>`
`<UnitPrice>345.00</UnitPrice>`
`</LineItem>`

`<LineItem>`
`<OrderDate>2016-05-18</OrderDate>`
`<CustomerName>Agrita Abele</CustomerName>`
`<OrderID>72669</OrderID>`
`<StockItemName>10 毫米防静电气泡膜（蓝色）10 米</StockItemName>`
`<Quantity>70</Quantity>`
`<UnitPrice>26.00</UnitPrice>`
`</LineItem>`

`<LineItem>`
`<OrderDate>2016-05-27</OrderDate>`
`<CustomerName>Agrita Abele</CustomerName>`
`<OrderID>73340</OrderID>`
`<StockItemName>32 毫米双面气泡膜 50 米</StockItemName>`
`<Quantity>20</Quantity>`
`<UnitPrice>112.00</UnitPrice>`
`</LineItem>`

`<LineItem>`
`<OrderDate>2016-05-18</OrderDate>`
`<CustomerName>Agrita Abele</CustomerName>`
`<OrderID>72669</OrderID>`
`<StockItemName>办公室方块潜望镜（黑色）</StockItemName>`
`<Quantity>20</Quantity>`
`<UnitPrice>18.50</UnitPrice>`
`</LineItem>`

`<LineItem>`
`<OrderDate>2016-05-27</OrderDate>`
`<CustomerName>Agrita Abele</CustomerName>`
`<OrderID>73356</OrderID>`
`<StockItemName>复古美国玩具双座小轿车（黑色）1:12 比例</StockItemName>`
`<Quantity>10</Quantity>`
`<UnitPrice>285.00</UnitPrice>`
`</LineItem>`

`</SalesOrders>`

## 使用 `FOR XML AUTO`

与 `FOR XML RAW` 不同，`FOR XML AUTO` 可以返回嵌套结果。它使用起来也异常简单，因为它会根据查询中的连接自动嵌套数据。`清单 3-10` 中修改后的查询使用 `AUTO` 模式来返回一个分层的 XML 文档。

