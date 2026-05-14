# 第 4 章 查询与分解 XML

### 图 4-8. 使用 for 循环遍历产品元素的结果

清单 4-14 中的查询使用了`let`语句，而非`for`循环迭代，来检索 XML 文档中第一个订单内的第一个产品的名称。`return`语句要求一个单值，而`let`只是赋值（与迭代多个值相对），因此我们无法返回多个产品。

### 清单 4-14. 使用 let 查找产品名称

```xquery
SELECT @XML.query('
let $product := /SalesOrders/Order[1]/OrderDetails/Product/@ProductName
return string($product[1])
') ;
```

此查询的结果见图 4-9。

### 图 4-9. 使用 let 查找产品名称的结果

清单 4-15 中的查询结合了`for`语句和`let`语句，为每个售出的产品构建一个新的 XML 文档，其中包含客户名称和产品名称。

### 清单 4-15. 结合使用 for 和 let

```xquery
SELECT @XML.query('
for $product in /SalesOrders/Order/OrderDetails/Product/@ProductName
let $customer := /SalesOrders/Order/OrderHeader/CustomerName
return
<Customer>
  {$customer[1]}
  <OrderDetails>
    {$product}
  </OrderDetails>
</Customer>
') ;
```

生成的 XML 文档见清单 4-16。

### 清单 4-16. 结合使用 for 和 let 的结果

```xml
<Customer>
  <CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>
  <OrderDetails ProductName="恐龙电池动力拖鞋（绿色）M 码" />
</Customer>
<Customer>
  <CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>
  <OrderDetails ProductName="遥控玩具轿车（绿色）1/50 比例" />
</Customer>
<Customer>
  <CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>
  <OrderDetails ProductName="黑色与橙色玻璃注意运送胶带 48mmx100m" />
</Customer>
<Customer>
  <CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>
  <OrderDetails ProductName="外星军官连帽衫（黑色）3XL 码" />
</Customer>
<Customer>
  <CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>
  <OrderDetails ProductName="开发者玩笑马克杯 - 那是硬件问题（黑色）" />
</Customer>
<Customer>
  <CustomerName>Tailspin Toys (Absecon, NJ)</CustomerName>
  <OrderDetails ProductName="恐龙电池动力拖鞋（绿色）XL 码" />
</Customer>
```

清单 4-17 中的查询增强了清单 4-15 中的查询，添加了一个`where`语句来筛选产品，使得只返回恐龙拖鞋。该查询还使用了`order by`语句，以确保结果按产品名称排序。

### 清单 4-17. 使用 where 和 order by

```xquery
SELECT @XML.query('
for $product in /SalesOrders/Order/OrderDetails/Product/@ProductName
let $customer := /SalesOrders/Order/OrderHeader/CustomerName
where $product = "恐龙电池动力拖鞋（绿色）M 码"
   or $product = "恐龙电池动力拖鞋（绿色）XL 码"
order by $product
return
<Customer>
  {$customer[1]}
  <OrderDetails>
    {$product}
  </OrderDetails>
</Customer>
') ;
```

## 修改 XML 数据

可以使用 XQuery 的`modify`方法修改 XML 数据。使用`modify`时，开发者有三种选项：`insert`选项、`delete`选项或`replace value of`选项。`replace value of`选项用于替换 XML 文档中的现有值。

要理解`insert`选项的工作原理，请看清单 4-18 中的脚本。该脚本用一个空订单填充变量。然后使用`modify`方法向该销售订单中添加一个订单行。

### 清单 4-18. 使用 modify Insert

```xml
DECLARE @SalesOrder xml;
SET @SalesOrder = '
<Order>
  <OrderHeader>
    <CustomerName>Camille Authier</CustomerName>
    <OrderDate>2013-01-02</OrderDate>
    <OrderID>121</OrderID>
  </OrderHeader>
  <OrderDetails>
  </OrderDetails>
</Order>' ;

SET @SalesOrder.modify('
```


## 第 4 章 查询与解析 XML

insert `<Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="6" />`

into `(/Order/OrderDetails)[1]')` ;

`SELECT @SalesOrder` ;

执行此查询的结果可以在清单 4-19 中查看。

### 清单 4-19. 使用 modify Insert 的结果

```xml
<Order>
    <OrderHeader>
        <CustomerName>Camille Authier</CustomerName>
        <OrderDate>2013-01-02</OrderDate>
        <OrderID>121</OrderID>
    </OrderHeader>
    <OrderDetails>
        <Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="6" />
    </OrderDetails>
</Order>
```

但是，如果`OrderDetails`元素中已经存在`Product`元素，您可以通过使用`as first`、`as last`、`before`或`after`选项来精细控制元素的插入位置。例如，清单 4-20 中的查询将把新的`Product`元素作为`OrderDetails`元素中的第一个元素插入。

### 清单 4-20. 将元素作为第一个插入

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
        <Product ProductID="22" ProductName="DBA joke mug - it depends (White)" Price="13" Qty="6" />
    </OrderDetails>
</Order>' ;

SET @SalesOrder.modify('
insert <Product ProductID="2" ProductName="USB rocket launcher (Gray)" Price="25" Qty="9" /> as first
into (/Order/OrderDetails)[1]') ;

SELECT @SalesOrder ;
```

此查询的结果可以在清单 4-21 中查看。

### 清单 4-21. 将元素作为第一个插入的结果

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

或者，清单 4-22 中的查询演示了如何在另一个特定元素之后插入元素。在本例中，我们将在 USB 火箭发射器之后插入超级英雄行动夹克。

### 清单 4-22. 在另一个元素之后插入元素

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
SET @ProductName = 'USB rocket launcher (Gray)' ;

SET @SalesOrder.modify('
insert <Product ProductID="111" ProductName="Superhero action jacket (Blue) M" Price="30" Qty="10" />
after (/Order/OrderDetails/Product[@ProductName = sql:variable("@ProductName")])[1]') ;

SELECT @SalesOrder ;
```

此查询使用`sql:variable`从 T-SQL 变量传递产品名称，运用了本章“在 XQuery 中使用关系值”一节中学到的技术。查询结果可以在清单 4-23 中查看。

### 清单 4-23. 在另一个元素之后插入元素的结果

```xml
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
</Order>
```

`delete`选项可用于从 XML 变量或列中删除节点。



