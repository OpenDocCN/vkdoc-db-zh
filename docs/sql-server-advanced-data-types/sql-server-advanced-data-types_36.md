# 第 3 章 使用 T-SQL 构建 XML

## 使用 FOR XML EXPLICIT

`FOR XML EXPLICIT` 是 `FOR XML` 模式中最复杂的，但也提供了极致的灵活性。它使用标签和父节点的概念来控制嵌套。标签指定节点在层次结构中的位置，而父节点指示其上方节点的标签。因此，顶层节点的标签为 1，父节点为 `NULL`。每个节点在单独的查询中指定，这些查询使用 `UNION ALL` 进行连接。

例如，假设我们希望扩展清单 3-18 中的示例，使得我们返回一个包含两个子元素的 `OrderDetails` 元素。第一个将包含接单销售人员的姓名。第二个将是一个重复元素，详细列出订单中的行项目。我们可以通过使用清单 3-19 中的查询来实现这一点。

`清单 3-19.` 使用 XML Explicit 的订单详情

```sql
SELECT
    1 AS Tag
    , 0 AS Parent
    , SalesOrder.OrderID AS
      [OrderDetails!1!SalesOrderID]
    , SalesOrder.OrderDate AS
      [OrderDetails!1!OrderDate]
    , SalesOrder.CustomerID AS
      [OrderDetails!1!CustomerID]
    , NULL AS [SalesPerson!2!SalesPersonName]
    , NULL AS [LineItem!3!LineTotal!ELEMENT]
    , NULL AS [LineItem!3!ProductName!ELEMENT]
    , NULL AS [LineItem!3!OrderQty!ELEMENT]
FROM Sales.Orders SalesOrder
INNER JOIN Sales.Customers Customers
    ON Customers.CustomerID = SalesOrder.CustomerID
WHERE customers.CustomerName = 'Agrita Abele'

UNION ALL

SELECT
    2 AS Tag
    , 1 AS Parent
    , SalesOrder.OrderID
    , NULL
    , NULL
    , People.FullName
    , NULL
    , NULL
    , NULL
FROM Sales.Orders SalesOrder
INNER JOIN Sales.Customers Customers
    ON Customers.CustomerID = SalesOrder.CustomerID
INNER JOIN Application.People People
    ON People.PersonID = SalesOrder.SalespersonPersonID
WHERE customers.CustomerName = 'Agrita Abele'

UNION ALL

SELECT
    3 AS Tag
    , 1 AS Parent
    , SalesOrder.OrderID
    , NULL
    , NULL
    , People.FullName
    , LineItem.UnitPrice
    , Product.StockItemName
    , LineItem.Quantity
FROM Sales.Orders SalesOrder
INNER JOIN Sales.OrderLines LineItem
    ON SalesOrder.OrderID = LineItem.OrderID
INNER JOIN Sales.Customers Customers
    ON Customers.CustomerID = SalesOrder.CustomerID
INNER JOIN Warehouse.StockItems Product
    ON Product.StockItemID = LineItem.StockItemID
INNER JOIN Application.People People
    ON People.PersonID = SalesOrder.SalespersonPersonID
WHERE customers.CustomerName = 'Agrita Abele'
ORDER BY
    [OrderDetails!1!SalesOrderID]
    , [SalesPerson!2!SalesPersonName]
    , [LineItem!3!LineTotal!ELEMENT]
FOR XML EXPLICIT, ROOT('SalesOrders');
```

关于清单 3-19 中的查询，我们需要注意几点。首先，每个复杂的节点都在其自己的查询中，这些查询通过 `UNION ALL` 连接。就像每次使用 `UNION ALL` 一样，每个查询必须具有相同数量的列。因此，当查询所定义的复杂节点中某个节点不存在时，会使用 `NULL` 值。另外，请注意 `OrderID` 列在所有三个查询中都使用了。该列用于定义哪些销售人员和行项目明细应归属于每个销售订单之下。

```sql
CustomerName 'OrderHeader/CustomerName'
, OrderDate 'OrderHeader/OrderDate'
, OrderID 'OrderHeader/OrderID'
, (
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
FROM (
    SELECT DISTINCT
        Customers.CustomerName
        , SalesOrder.OrderDate
        , SalesOrder.OrderID
    FROM Sales.Orders SalesOrder
    INNER JOIN Sales.OrderLines LineItem
        ON SalesOrder.OrderID = LineItem.OrderID
    INNER JOIN Sales.Customers Customers
        ON Customers.CustomerID = SalesOrder.CustomerID
    WHERE customers.CustomerName = 'Agrita Abele'
) Base
FOR XML PATH('Order'), ROOT ('SalesOrders');
```


因为 `SalesPerson` 和 `LineItem` 节点是同级节点，所以第二和第三个查询的父节点都配置为 `1`，即第一个查询的标签。这意味着两者都将嵌套在 `OrderDetails` 节点下，处于层级的同一级别。定义 `OrderDetails` 节点的查询将其父节点配置为 `NULL`。这表示它位于层级的顶部，根节点除外，根节点是在 `FOR XML` 子句中定义的。

与其他 `UNION ALL` 查询一样，列名源自第一个查询，节点名也取自第一个查询。如果你检查第一个查询中的列别名，你将清楚地看到节点名的结构是如何定义的。名称部分用 `!` 分隔。第一个名称部分定义了复合节点的名称。第二部分描述了正在命名的标签的 ID。名称的第三部分定义了将存储列中数据的节点的名称。默认情况下，`FOR XML EXPLICIT` 是以属性为中心的。因此，如果你需要一个节点作为元素，你必须在名称中添加第四部分，称为 `ELEMENT` 指令。

查询 3-19 生成了 3-20 中的 XML 文档。

### 列表 3-20. FOR XML EXPLICIT 结果
```xml
<SalesOrders>
    <OrderDetails SalesOrderID="72637" OrderDate="2016-05-18"
        CustomerID="1061">
        <SalesPerson SalesPersonName="Taj Shand" />
        <LineItem>
            <LineTotal>5.00</LineTotal>
            <ProductName>Furry animal socks (Pink) S</ProductName>
            <OrderQty>96</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>18.00</LineTotal>
            <ProductName>"The Gu" red shirt XML tag t-shirt (Black)
                XXL</ProductName>
            <OrderQty>24</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>25.00</LineTotal>
            <ProductName>Superhero action jacket (Blue) 3XS</ProductName>
            <OrderQty>1</OrderQty>
        </LineItem>
    </OrderDetails>
    <OrderDetails SalesOrderID="72669" OrderDate="2016-05-18"
        CustomerID="1061">
        <SalesPerson SalesPersonName="Hudson Onslow" />
        <LineItem>
            <LineTotal>2.70</LineTotal>
            <ProductName>Permanent marker black 5mm nib (Black) 5mm
            </ProductName>
            <OrderQty>84</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>18.00</LineTotal>
            <ProductName>Halloween skull mask (Gray) M</ProductName>
            <OrderQty>60</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>18.50</LineTotal>
            <ProductName>Office cube periscope (Black)</ProductName>
            <OrderQty>20</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>26.00</LineTotal>
            <ProductName>10 mm Anti static bubble wrap (Blue) 10m
            </ProductName>
            <OrderQty>70</OrderQty>
        </LineItem>
    </OrderDetails>
    <OrderDetails SalesOrderID="72671" OrderDate="2016-05-18"
        CustomerID="1061">
        <SalesPerson SalesPersonName="Hudson Hollinworth" />
        <LineItem>
            <LineTotal>1.89</LineTotal>
            <ProductName>Packing knife with metal insert blade
                (Yellow) 9mm</ProductName>
            <OrderQty>35</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>2.74</LineTotal>
            <ProductName>Shipping carton (Brown) 480x270x320mm
            </ProductName>
            <OrderQty>150</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>35.00</LineTotal>
            <ProductName>Alien officer hoodie (Black) 4XL</ProductName>
            <OrderQty>6</OrderQty>
        </LineItem>
    </OrderDetails>
    <OrderDetails SalesOrderID="72713" OrderDate="2016-05-18"
        CustomerID="1061">
        <SalesPerson SalesPersonName="Hudson Onslow" />
        <LineItem>
            <LineTotal>32.00</LineTotal>
            <ProductName>Tape dispenser (Red)</ProductName>
            <OrderQty>20</OrderQty>
        </LineItem>
    </OrderDetails>
    <OrderDetails SalesOrderID="72770" OrderDate="2016-05-19"
        CustomerID="1061">
        <SalesPerson SalesPersonName="Hudson Onslow" />
        <LineItem>
            <LineTotal>1.14</LineTotal>
            <ProductName>Shipping carton (Brown) 356x229x229mm
            </ProductName>
            <OrderQty>175</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>4.10</LineTotal>
            <ProductName>Black and orange handle with care despatch
                tape 48mmx100m</ProductName>
            <OrderQty>96</OrderQty>
        </LineItem>
        <LineItem>
            <LineTotal>18.00</LineTotal>
```



