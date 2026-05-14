# 第 4 章：查询与分解 XML

## 使用 delete 选项

例如，考虑清单 4-24 中的脚本。这里，我们使用清单 4-22 中的查询所生成的 XML 文档，并移除我们添加到该文档中的最后一个订单行。

**清单 4-24.** 使用 delete 选项

```sql
DECLARE @SalesOrder xml;

SET @SalesOrder = '
<Order>
  <OrderHeader>
    <CustomerName>Camille Authier</CustomerName>
    <OrderDate>2013-01-02</OrderDate>
    <OrderID>121</OrderID>
  </OrderHeader>
  <OrderDetails>
    <Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />
    <Product ProductID="111" ProductName="Superhero action jacket (Blue) M" Price="30" Qty="10" />
    <Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="6" />
  </OrderDetails>
</Order>' ;

DECLARE @ProductName NVARCHAR(200) ;
SET @ProductName = 'Superhero action jacket (Blue) M' ;

SET @SalesOrder.modify('
  delete (/Order/OrderDetails/Product[@ProductName = sql:variable("@ProductName")])[1]') ;

SELECT @SalesOrder ;
```

你可以在清单 4-25 中看到生成的 XML 文档。

**清单 4-25.** 使用 delete 选项的结果

```xml
<Order>
  <OrderHeader>
    <CustomerName>Camille Authier</CustomerName>
    <OrderDate>2013-01-02</OrderDate>
    <OrderID>121</OrderID>
  </OrderHeader>
  <OrderDetails>
    <Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />
    <Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="6" />
  </OrderDetails>
</Order>
```

## 使用 replace value of 选项

`replace value of` 选项用于对现有节点执行更新操作。

例如，清单 4-26 中的脚本将 DBA Joke Mugs 的订购数量从 6 更新为 10。我们再次绑定 SQL 变量，以便传入我们希望更新的 `ProductName` 和新的数量。

请注意，我们使用 `replace value of` 来定义应更新的节点，并使用 `with` 来定义新值。

**清单 4-26.** 使用 replace value of

```sql
DECLARE @SalesOrder xml;

SET @SalesOrder = '
<Order>
  <OrderHeader>
    <CustomerName>Camille Authier</CustomerName>
    <OrderDate>2013-01-02</OrderDate>
    <OrderID>121</OrderID>
  </OrderHeader>
  <OrderDetails>
    <Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />
    <Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="6" />
  </OrderDetails>
</Order>' ;

DECLARE @ProductName NVARCHAR(200) ;
SET @ProductName = 'DBA joke mug - it depends (White)' ;

DECLARE @Quantity INT ;
SET @Quantity = 10

SET @SalesOrder.modify('
  replace value of (/Order/OrderDetails/Product[@ProductName = sql:variable("@ProductName")]/@Qty)[1]
  with "10"
') ;

SELECT @SalesOrder ;
```

该查询的结果见清单 4-27。

**清单 4-27.** 使用 replace value of 的结果

```xml
<Order>
  <OrderHeader>
    <CustomerName>Camille Authier</CustomerName>
    <OrderDate>2013-01-02</OrderDate>
    <OrderID>121</OrderID>
  </OrderHeader>
  <OrderDetails>
    <Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" />
    <Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="10" />
  </OrderDetails>
</Order>
```

## 分解 XML

分解 XML 是指获取 XML 结果集并将其转换为关系结果集的过程。在 SQL Server 中，有两种方法可以实现这一点：`OPENXML()` 函数和 `nodes()` XQuery 方法。

### 使用 OPENXML() 分解 XML

`OPENXML()` 是一个函数，它接受一个 XML 文档并将其转换为关系结果集。它接受的参数详见表 4-2。

**表 4-2.** `OPENXML()` 参数

| 参数 | 描述 |
|-------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| idoc | 指向 XML 文档内部表示的指针 |
| rowpattern | 应转换为行的 XML 文档最低级别的路径 |
| flags | 一个可选参数，用于指定 ... 之间的映射 |


