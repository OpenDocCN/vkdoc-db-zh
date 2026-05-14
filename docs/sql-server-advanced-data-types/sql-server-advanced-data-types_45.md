# 第 4 章 查询与分解 XML

来定义粒度。接着，我们为包含中间结果集的表和列定义任意名称。

`nodes()`方法也可与`query()`方法结合使用，从而将一个 XML 文档分解为更小的 XML 文档。例如，请看**代码清单 4-30**中的查询。此查询以 XML 格式从 XML 文档中提取`Product`元素。

***代码清单 4-30.*** 结合使用`nodes()`方法与`query()`方法
```sql
DECLARE @SalesOrder XML ;

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

SELECT
    TempCol.query('.') AS Product
FROM @SalesOrder.nodes('SalesOrders/Order/OrderDetails/Product') TempTable(TempCol) ;
```

此查询的结果可见于**图 4-11**。

***图 4-11.*** 结合使用`nodes()`方法与`query()`方法的结果

`nodes()`方法相对于`OPENXML()`的最大优势在于其针对表的易用性。`CROSS APPLY`操作符可用于将`nodes()`方法应用于表内的多行。**代码清单 4-31**中的查询阐释了这一点，它对`Sales.CustomerOrderSummary`表中的`OrderSummary`列调用了`nodes()`方法。该查询将为`CustomerID`为 814 的客户所下每个销售订单上的每个产品返回一行。结果随后按每个产品的数量进行排序。

***代码清单 4-31.*** 对表使用`nodes()`方法
```sql
USE WideWorldImporters
GO

SELECT
    CustomerID
    , TempCol.value('@Qty', 'INT') AS Quantity
    , TempCol.value('@ProductName', 'NVARCHAR(70)') AS ProductName
    , TempCol.query('.') AS Product
FROM Sales.CustomerOrderSummary
CROSS APPLY OrderSummary.nodes('SalesOrders/Order/OrderDetails/Product') TempTable(TempCol)
WHERE CustomerID = 841
ORDER BY TempCol.value('@Qty', 'INT') DESC ;
```

此查询的部分结果可见于**图 4-12**。

***图 4-12.*** 对表使用`nodes()`方法的结果

## 使用架构

如**第 3 章**所讨论的，XML 文档可以绑定到一个架构，以确保其结构符合客户端约定。在 SQL Server 中，我们可以通过创建`XML SCHEMA COLLECTION`来定义 XSD 架构。为了演示这一点，假设我们希望将`Sales.CustomerOrderSummary`表中的`OrderSummary`列绑定到一个架构。第一步是创建`XML SCHEMA COLLECTION`。我们可以使用**代码清单 4-32**中的代码来实现这一点。

***代码清单 4-32.*** 创建`XML SCHEMA COLLECTION`
```sql
USE WideWorldImporters
GO

CREATE XML SCHEMA COLLECTION OrderSummary AS
N'<?xml version="1.0" encoding="utf-16"?>
<xs:schema attributeFormDefault="unqualified"
elementFormDefault="qualified" xmlns:xs="http://www.w3.org/2001/XMLSchema">
  <xs:element name="SalesOrders">
    <xs:complexType>
      <xs:sequence>
        <xs:element maxOccurs="unbounded" name="Order">
          <xs:complexType>
```

