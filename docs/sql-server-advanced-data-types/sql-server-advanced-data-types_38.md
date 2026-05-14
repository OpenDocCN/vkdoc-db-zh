# 第 4 章 查询与分解 XML

为了让开发者能够从 `SQL Server` 内部查询和导航 XML 文档，可以将 `XQuery` 语言与 `T-SQL` 查询结合使用。在本章中，我将讨论如何使用 `XQuery` 来过滤、提取和修改 XML。我还将讨论分解 XML 的过程，即将 XML 数据转换为关系结果集的过程。最后，提供了如何将 `XSD` 架构绑定到数据类型为 `XML` 的列的概述。

## 查询 XML

`XQuery` 是一种用于查询 XML 的语言，就像 `SQL` 是用于查询关系数据的语言一样。该语言构建在 `XPath` 之上，但进行了增强以实现更好的迭代和排序。它还允许构建 XML。`XQuery` 标准由 `W3C`（万维网联盟）与 Microsoft 和其他主要关系数据库管理系统（`RDBMS`）供应商共同开发。

`SQL Server` 中的 `XQuery` 支持针对 `XML` 数据类型的五种方法。这些方法的概述见表 4-1，本章将逐一演示这些方法。`XQuery` 还支持使用 `XSD` 架构，以帮助处理复杂文档和 `FLWOR`（发音同“flower”）语句。`FLWOR` 是 `for`、`let`、`where`、`order by` 和 `return` 的首字母缩写。

`for` 语句允许您遍历节点序列。`let` 语句将序列绑定到变量。`where` 语句基于布尔表达式过滤节点。`order by` 语句在返回节点前对其进行排序。`return` 语句指定应返回的内容。

**提示** `XSD` 架构的结构和优势在第 2 章中有解释。

为了演示本章中 `XQuery` 方法的使用，我们将在 `WideWorldImporters` 数据库中创建一个名为 `Sales.CustomerOrderSummary` 的表。可以使用清单 4-1 中的脚本来创建此表。

### 表 4-1. XQuery 方法

| `exist()` | 检查 XML 文档中是否存在具有指定值的节点。如果存在具有指定值的节点，则返回 1，否则返回 0。 |
| :--- | :--- |
| `modify()` | 对 XML 文档执行数据修改语句。可接受的 `DML` 操作是 `insert`、`delete` 和 `replace value of`。 |
| `nodes()` | 返回一个包含原始 XML 实例副本的行集。用于将 XML 分解为关系结果集。 |
| `query()` | 以 XML 格式返回 XML 文档的子集。 |
| `value()` | 从 XML 文档返回单个标量值，映射到 `SQL Server` 数据类型。 |

### 清单 4-1. 创建 Sales.CustomerOrderSummary 表

```sql
USE WideWorldImporters
GO
CREATE TABLE Sales.CustomerOrderSummary
(
    ID INT NOT NULL IDENTITY,
    CustomerID INT NOT NULL,
    OrderSummary XML
) ;

INSERT INTO Sales.CustomerOrderSummary (CustomerID, OrderSummary)
SELECT
    CustomerID,
    (
        SELECT
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
```

© Peter A. Carter 2018
P. A. Carter, *SQL Server 高级数据类型*, [`doi.org/10.1007/978-1-4842-3901-8_4`](https://doi.org/10.1007/978-1-4842-3901-8_4)


## 使用 exist()

`exist()` 方法用于检查具有指定值的节点是否存在。例如，请考虑清单 4-2 中的脚本。该查询将以编程方式检查包含销售订单详细信息的 XML 文档中是否包含“巧克力鲨鱼 250 克”产品。

XQuery 的第一部分定义了要评估节点的路径，在本例中是 `StockItemName` 属性。我们通过在节点名前添加 `@` 符号来定义该节点是一个属性。然后，我们指定 `eq` 作为比较运算符，再提供我们要用来验证节点的值。如果满足条件，`exist()` 将返回 `1`。如果不满足条件，则返回 `0`。

***清单 4-2.*** 检查值的存在性

```sql
DECLARE @SalesOrders XML;
SET @SalesOrders =
'<SalesOrder OrderDate="2016-05-27" OrderID="73356">
    <Customers CustomerName="Agrita Abele">
        <Product StockItemName="Chocolate sharks 250g">
            <LineItem Quantity="192" UnitPrice="8.55" />
        </Product>
    </Customers>
</SalesOrder>';

SELECT @SalesOrders.exist('SalesOrder/Customers/Product[(@StockItemName) eq "Chocolate sharks 250g"]');
```

你可以看到 `exist()` 方法如何轻松地用于 `WHERE` 子句中，针对数据类型为 `XML` 的列。例如，考虑清单 4-3 中的脚本，它在 `WideWorldImporters` 数据库的 `Sales.CustomerOrderSummary` 表的 `WHERE` 子句中使用了 `exist()` 方法，仅返回 Tailspin Toys (Absecon, NJ) 的订单摘要。

你会注意到，因为我们评估的是元素的值，而不是属性，所以我们使用了 `text()` 方法来提取值。`[1]` 表示将返回一个单例值。

***清单 4-3.*** 使用 `exist()` 方法过滤表

```sql
SELECT
    CustomerID,
    OrderSummary
FROM WideWorldImporters.Sales.CustomerOrderSummary
WHERE OrderSummary.exist('SalesOrders/Order/OrderHeader/CustomerName[(text()[1]) eq "Tailspin Toys (Absecon, NJ)"]') = 1;
```

## 使用 value()

`value()` 方法用于从 XML 文档中提取单个标量值，并将其映射到 SQL Server 数据类型。例如，考虑清单 4-4 中的脚本。该脚本使用了与清单 4-1 相同的 XML 文档，但这次是从文档中提取客户名称。

与使用 `exist()` 方法时一样，我们必须使用 `@` 符号作为属性的前缀。我们还必须指明将返回一个单例值。

***清单 4-4.*** 使用 `value()` 方法

```sql
DECLARE @SalesOrders XML;
SET @SalesOrders =
'<SalesOrder OrderDate="2016-05-27" OrderID="73356">
    <Customers CustomerName="Agrita Abele">
        <Product StockItemName="Chocolate sharks 250g">
            <LineItem Quantity="192" UnitPrice="8.55" />
        </Product>
    </Customers>
</SalesOrder>';

SELECT @SalesOrders.value('(/SalesOrder/Customers/@CustomerName)[1]', 'nvarchar(100)') AS CustomerName;
```



