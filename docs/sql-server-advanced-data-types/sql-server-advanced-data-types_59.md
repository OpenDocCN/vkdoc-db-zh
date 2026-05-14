# 第 7 章：从 T-SQL 构建 JSON

## 处理 JSON 输出中的空值

让我们考虑以下查询：

```sql
USE WideWorldImporters
GO

SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
    , Comments
FROM Sales.Orders
WHERE CustomerID = 1060;
```

此查询的结果如图 7-2 所示。

**图 7-2.** 添加注释后的查询结果

你会注意到返回的订单都没有关联的注释。让我们通过列表 7-9 中的查询来看看这对 JSON 有什么影响。

**列表 7-9.** 添加了空值注释的 JSON 查询

```sql
USE WideWorldImporters
GO

SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
    , Comments
FROM Sales.Orders
WHERE CustomerID = 1060
FOR JSON AUTO, ROOT('SalesOrders');
```

此查询的结果可以在列表 7-10 中找到。

**列表 7-10.** 包含空值注释的 JSON 结果

```json
{
  "SalesOrders": [
    {
      "OrderID": 72646,
      "CustomerID": 1060,
      "SalespersonPersonID": 14,
      "OrderDate": "2016-05-18"
    },
    {
      "OrderID": 72738,
      "CustomerID": 1060,
      "SalespersonPersonID": 14,
      "OrderDate": "2016-05-19"
    },
    {
      "OrderID": 72916,
      "CustomerID": 1060,
      "SalespersonPersonID": 6,
      "OrderDate": "2016-05-20"
    },
    {
      "OrderID": 73081,
      "CustomerID": 1060,
      "SalespersonPersonID": 8,
      "OrderDate": "2016-05-24"
    }
  ]
}
```

你可以看到，默认行为是从结果集中忽略任何值为 `NULL` 的名称/值对。然而，我们可以通过使用 `INCLUDE_NULL_VALUES` 选项来改变这一行为，如列表 7-11 所示。

**列表 7-11.** 在结果中包含 `NULL` 值

```sql
USE WideWorldImporters
GO

SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
    , Comments
FROM Sales.Orders
WHERE CustomerID = 1060
FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES;
```

此查询的结果可以在列表 7-12 中找到。你会注意到，这一次，值为 `NULL` 的名称/值对已被包含在结果中。

**列表 7-12.** 包含 `NULL` 值的结果

```json
{
  "SalesOrders": [
    {
      "OrderID": 72646,
      "CustomerID": 1060,
      "SalespersonPersonID": 14,
      "OrderDate": "2016-05-18",
      "Comments": null
    },
    {
      "OrderID": 72738,
      "CustomerID": 1060,
      "SalespersonPersonID": 14,
      "OrderDate": "2016-05-19",
      "Comments": null
    },
    {
      "OrderID": 72916,
      "CustomerID": 1060,
      "SalespersonPersonID": 6,
      "OrderDate": "2016-05-20",
      "Comments": null
    },
    {
      "OrderID": 73081,
      "CustomerID": 1060,
      "SalespersonPersonID": 8,
      "OrderDate": "2016-05-24",
      "Comments": null
    }
  ]
}
```

## 使用列别名

在考虑 JSON 输出格式时，也可以使用列别名，如列表 7-13 所示。

**列表 7-13.** 使用列别名

```sql
USE WideWorldImporters
GO

SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
    , Comments AS 'Orders Comments'
FROM Sales.Orders
WHERE CustomerID = 1060
FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES;
```

此查询返回的结果如列表 7-14 所示。

**列表 7-14.** 使用列别名的结果

```json
{
  "SalesOrders": [
    {
      "OrderID": 72646,
      "CustomerID": 1060,
      "SalespersonPersonID": 14,
      "OrderDate": "2016-05-18",
      "Orders Comments": null
    },
    {
      "OrderID": 72738,
      "CustomerID": 1060,
      "SalespersonPersonID": 14,
      "OrderDate": "2016-05-19",
      "Orders Comments": null
    },
    {
      "OrderID": 72916,
      "CustomerID": 1060,
      "SalespersonPersonID": 6,
      "OrderDate": "2016-05-20",
      "Orders Comments": null
    },
    {
      "OrderID": 73081,
      "CustomerID": 1060,
      "SalespersonPersonID": 8,
      "OrderDate": "2016-05-24",
      "Orders Comments": null
    }
  ]
}
```

你会注意到，`comments` 键现在使用的是名称 `Orders Comments`，而不是 `Comments`。如果你使用点分隔的别名，这种行为同样适用，如列表 7-15 中的示例。

**列表 7-15.** 使用点分隔的别名

```sql
USE WideWorldImporters
GO

SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
    , Comments AS 'Orders.Comments'
FROM Sales.Orders
WHERE CustomerID = 1060
FOR JSON AUTO, ROOT('SalesOrders'), INCLUDE_NULL_VALUES;
```


