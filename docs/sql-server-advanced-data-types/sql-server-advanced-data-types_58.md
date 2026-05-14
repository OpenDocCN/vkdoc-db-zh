# 第 7 章 从 T-SQL 构建 JSON

```sql
SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
FROM Sales.Orders
WHERE CustomerID = 1060 ;
```

此查询返回的结果如图 7-1 所示。

图 7-1. 销售订单的键和日期结果

如果我们希望以 JSON 格式返回这些结果，可以附加`FOR JSON`子句，如代码清单 7-2 所示。

## 代码清单 7-2. 以 JSON 格式返回结果

```sql
USE WideWorldImporters
GO
SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
FROM Sales.Orders
WHERE CustomerID = 1060
FOR JSON AUTO ;
```

此查询将产生如代码清单 7-3 所示的结果。

## 代码清单 7-3. 以 JSON 格式返回数据的结果

```json
[
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
```

## 使用根节点

由于我们的结果没有根节点，文档周围必须有一个数组包装器（以方括号的形式）。如果我们不希望结果被方括号包围，可以通过修改查询来移除它们，如代码清单 7-4 所示。

### 代码清单 7-4. 移除数组包装器

```sql
USE WideWorldImporters
GO
SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
FROM Sales.Orders
WHERE CustomerID = 1060
FOR JSON AUTO, WITHOUT_ARRAY_WRAPPER ;
```

这个修改后的查询将返回如代码清单 7-5 所示的结果。

### 代码清单 7-5. 没有数组包装器的结果

```json
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
```

或者，我们可以通过为文档添加一个根节点来解决这个问题。这可以通过使用代码清单 7-6 所示的修改后的查询来实现，这将导致一个名为`SalesOrders`的根节点被添加到结果文档中。

**注意** `ROOT`和`WITHOUT_ARRAY_WRAPPER`彼此不兼容。尝试同时使用两者将导致查询生成错误。

### 代码清单 7-6. 添加根节点

```sql
USE WideWorldImporters
GO
SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
FROM Sales.Orders
WHERE CustomerID = 1060
FOR JSON AUTO, ROOT('SalesOrders') ;
```

此查询产生了如代码清单 7-7 所示的结果。

### 代码清单 7-7. 添加根节点后的结果

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

## 处理 NULL 值

如果我们的数据集返回了`NULL`值会怎样？考虑代码清单 7-8 中的示例，其中`Comments`列已被添加到`SELECT`列表中。

### 代码清单 7-8. 向 SELECT 列表中添加 Comments

```sql
USE WideWorldImporters
GO
SELECT
    OrderID
    , CustomerID
    , SalespersonPersonID
    , OrderDate
    , Comments
```

