# 第四章 查询与分解 XML

### 表 4-3. 标志值

| 值 | 描述 |
| :--- | :--- |
| 0 (默认) | 默认使用以属性为中心的映射。除非指定了 `XML_ELEMENTS`，否则使用以属性为中心的映射。如果指定了 `XML_ELEMENTS`，则属性将首先被映射，然后是元素。 |
| 1 (`XML_ELEMENTS`) | 默认使用以元素为中心的映射，除非指定了 `XML_ATTRIBUTES`。如果指定了 `XML_ATTRIBUTES`，则元素将首先被映射，然后是属性。 |
| 2 (`XML_ATTRIBUTES`) | 与逻辑或（OR）结合使用。 |

除了函数的参数外，还可以指定一个 `WITH` 子句，该子句定义了要返回的结果集的模式。虽然 `WITH` 子句是可选的，但如果不指定模式，则将返回一个边缘表。该模式用于指定关系列名、关系数据类型以及相关节点的路径。

#### 注意
`WITH` 子句中的特定映射将覆盖标志设置。

`OPENXML()` 函数的一个限制是它本身不解析 XML 文档。相反，你必须在调用该函数之前解析文档。这可以通过使用 `sp_xml_preparedocument` 系统存储过程来实现。该过程使用 MSXML 解析器解析 XML 文档，并创建一个指向已解析文档版本的句柄，该版本已准备好供使用。

在 `OPENXML()` 函数执行完毕后，你应该使用 `sp_xml_removedocument` 系统存储过程将已解析的文档从内存中移除。重要的是要记住拆除已解析的文档，否则如果频繁调用 `sp_xml_preparedocument` 过程，你可能会开始遇到内存问题。

要查看 `OPENXML()` 函数的实际应用，请参阅清单 4-28 中的脚本。该脚本将一个销售订单文档传递到一个变量中，然后使用 `OPENXML()` 来分解数据。

#### 提示
为了更广泛地演示如何使用 `OPENXML()` 导航 XML 文档，此 XML 文档的结构已从本章前面的示例中更改。

### 清单 4-28. 使用 OPENXML()

```sql
DECLARE @SalesOrder XML ;
DECLARE @ParseDoc INT ;

SET @SalesOrder = '
<SalesOrders>
  <Order>
    <OrderDate>2013-01-02</OrderDate>
    <OrderHeader>
      <CustomerName>Camille Authier</CustomerName>
    </OrderHeader>
    <OrderDetails>
      <Product ProductID="45" ProductName="Developer joke mug - there are 10 types of people in the world (Black)" Price="13" Qty="7" />
      <Product ProductID="58" ProductName="RC toy sedan car with remote control (Black) 1/50 scale" Price="25" Qty="4" />
    </OrderDetails>
  </Order>
  <Order>
    <OrderDate>2013-01-02</OrderDate>
    <OrderHeader>
      <CustomerName>Camille Authier</CustomerName>
    </OrderHeader>
    <OrderDetails OrderID = "122">
      <Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="6" />
      <Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />
      <Product ProductID="111" ProductName="Superhero action jacket (Blue) M" Price="30" Qty="10" />
      <Product ProductID="116" ProductName="Superhero action jacket (Blue) 4XL" Price="34" Qty="4" />
    </OrderDetails>
  </Order>
</SalesOrders>' ;

EXEC sp_xml_preparedocument @ParseDoc OUTPUT, @SalesOrder ;

SELECT *
FROM OPENXML(@ParseDoc, '/SalesOrders/Order/OrderDetails/Product')
WITH (
    ProductID INT '@ProductID',
    ProductName NVARCHAR(200) '@ProductName',
    Quantity INT '@Qty',
    OrderID INT '../@OrderID',
    OrderDate DATE '../../OrderDate',
    CustomerName NVARCHAR(200) '../../OrderHeader/CustomerName'
) ;

EXEC sp_xml_removedocument @ParseDoc ;
```

当你检查此脚本中 `OPENXML()` 语句的 `WITH` 子句时，你会看到我们首先指定了关系列名。通过查看 `Quantity` 的映射，你会发现关系名不必与 XML 节点的名称匹配。然后我们指定一个

