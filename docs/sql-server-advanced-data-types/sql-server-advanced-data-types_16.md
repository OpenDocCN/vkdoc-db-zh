# 第 2 章 理解 XML

清单 2-1 中的 XML 文档提供了一个示例，其中包含了一个虚构组织的销售订单详情。在此示例中，文档作者选择使用一个元素来存储每个销售订单，并使用嵌套元素来存储订单的每个行项目。每个销售订单和行项目的详细信息则存储在元素的属性中。

### 清单 2-1. 使用以属性为中心的方法存储的销售订单

```xml
<SalesOrders>
  <SalesOrder OrderDate="2013-03-07" CustomerID="57" OrderID="3168">
    <LineItem StockItemID="176" Quantity="5" UnitPrice="240.00" />
    <LineItem StockItemID="143" Quantity="108" UnitPrice="18.00" />
    <LineItem StockItemID="136" Quantity="3" UnitPrice="32.00" />
    <LineItem StockItemID="92" Quantity="48" UnitPrice="18.00" />
  </SalesOrder>
  <SalesOrder OrderDate="2013-03-22" CustomerID="57" OrderID="4107">
    <LineItem StockItemID="153" Quantity="40" UnitPrice="4.50" />
    <LineItem StockItemID="36" Quantity="9" UnitPrice="13.00" />
    <LineItem StockItemID="208" Quantity="108" UnitPrice="2.70" />
  </SalesOrder>
  <SalesOrder OrderDate="2013-04-09" CustomerID="57" OrderID="4980">
    <LineItem StockItemID="102" Quantity="10" UnitPrice="35.00" />
    <LineItem StockItemID="144" Quantity="24" UnitPrice="18.00" />
    <LineItem StockItemID="79" Quantity="36" UnitPrice="18.00" />
    <LineItem StockItemID="217" Quantity="10" UnitPrice="25.00" />
  </SalesOrder>
  <SalesOrder OrderDate="2016-01-09" CustomerID="57" OrderID="64608">
    <LineItem StockItemID="156" Quantity="40" UnitPrice="15.00" />
    <LineItem StockItemID="56" Quantity="7" UnitPrice="13.00" />
  </SalesOrder>
  <SalesOrder OrderDate="2016-05-25" CustomerID="57" OrderID="73148">
    <LineItem StockItemID="31" Quantity="7" UnitPrice="13.00" />
    <LineItem StockItemID="103" Quantity="2" UnitPrice="35.00" />
  </SalesOrder>
</SalesOrders>
```

清单 2-1 中的 XML 可以通过针对 `WideWorldImporters` 数据库运行清单 2-2 中的查询来生成。

### 清单 2-2. 生成以属性为中心的 XML

```sql
SELECT
    SalesOrder.OrderDate
    , SalesOrder.CustomerID
    , SalesOrder.OrderID
    , LineItem.StockItemID
    , LineItem.Quantity
    , LineItem.UnitPrice
FROM Sales.Orders SalesOrder
INNER JOIN Sales.OrderLines LineItem
    ON LineItem.OrderID = SalesOrder.OrderID
WHERE SalesOrder.OrderID IN
(
    3168,
    4107,
    4980,
    64608,
)
FOR XML AUTO, ROOT('SalesOrders') ;
```

