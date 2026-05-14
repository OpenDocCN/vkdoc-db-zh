# 8-7. 处理 CLR 数据类型

### 问题

你希望在 JSON 输出中包含 CLR 数据类型。

### 解决方案

`ToString()` 函数将 CLR 数据类型值转换为 `nvarchar` 数据类型。代码清单 8-28 演示了一个查询，其中 `Customers.DeliveryLocation` `geography` 数据类型列是 JSON 输出的一部分。代码清单 8-29 显示了 JSON 输出。

```sql
SELECT TOP (2) Customers.CustomerName,
People.FullName AS PrimaryContact,
ap.FullName AS AlternateContact,
Customers.PhoneNumber,
Cities.CityName AS CityName,
Customers.DeliveryLocation.ToString() AS DeliveryLocation
FROM Sales.Customers AS Customers
JOIN [Application].People AS People
ON Customers.PrimaryContactPersonID = People.PersonID
JOIN [Application].People AS ap
ON Customers.AlternateContactPersonID = ap.PersonID
JOIN [Application].Cities AS Cities
ON Customers.DeliveryCityID = Cities.CityID
FOR JSON PATH, ROOT('Customers');
代码清单 8-28. 将 CLR 值转换为字符串
```

```json
{"Customers": [
{
"CustomerName": "Tailspin Toys (Head Office)",
"PrimaryContact": "Waldemar Fisar",
"AlternateContact": "Laimonis Berzins",
"PhoneNumber": "(308) 555-0100",
"CityName": "Lisco",
"DeliveryLocation": "POINT (-102.6201979 41.4972022)"
},
{
"CustomerName": "Tailspin Toys (Sylvanite, MT)",
"PrimaryContact": "Lorena Cindric",
"AlternateContact": "Hung Van Groesen",
"PhoneNumber": "(406) 555-0100",
"CityName": "Sylvanite",
"DeliveryLocation": "POINT (-115.8743507 48.7163356)"
}
]
}
代码清单 8-29. 在 JSON 输出中显示
```

### 工作原理

`ToString()` 函数隐式地将 CLR 数据类型 `geography`、`geometry` 和 `hierarchyid` 转换为 `nvarchar` 数据类型。`ToString()` 函数的默认输出如表 8-5 所示。

**表 8-5.** 显示 `ToString()` 函数转换输出

| 数据类型 | 转换为数据类型 |
| --- | --- |
| `hierarchyid` | `nvarchar(4000)` |
| `geography` | `nvarchar(max)` |
| `geometry` | `nvarchar(max)` |

> **注意**
> `hierarchyid` 数据类型是此限制的一个例外。当包含 `hierarchyid` 数据类型的列是 JSON 输出的一部分时，`FOR JSON` 子句不会引发错误。

当 `FOR JSON` 子句引用未转换的 `geography` 和 `geometry` 数据类型时，SQL Server 会引发如代码清单 8-30 所示的错误。

```text
Msg 13604, Level 16, State 1, Line 1
FOR JSON cannot serialize CLR objects. Cast CLR types explicitly into one of the supported types in FOR JSON queries.
代码清单 8-30. 显示未转换 CLR 时的错误
```

`ToString()` 函数区分大小写，这对于大多数 SQL Server 内置函数来说并不常见。因此，任何与正确大小写的 `ToString()` 函数不同的拼写都会引发错误。例如，当函数拼写为 `tostring()` 时，SQL Server 会引发错误。代码清单 8-31 演示了错误消息。

```text
Msg 6506, Level 16, State 10, Line 5 Could not find method 'tostring' for type 'Microsoft.SqlServer.Types.SqlGeography' in assembly 'Microsoft.SqlServer.Types'
代码清单 8-31. 显示 `ToString()` 函数拼写错误时的错误消息
```

作为替代方案，`CAST()` 和 `CONVERT()` 函数可以显式地将 `geography` 和 `geometry` 转换为 `nvarchar` 和 `varchar` 数据类型。代码清单 8-32 演示了一个查询，其中使用 `CAST()` 函数将具有 `geography` 数据类型的列 `Customers.DeliveryLocation` 显式转换为 `nvarchar(1000)`。

```sql
SELECT TOP (2) Customers.CustomerName,
People.FullName AS PrimaryContact,
ap.FullName AS AlternateContact,
Customers.PhoneNumber,
Cities.CityName AS CityName,
CAST(Customers.DeliveryLocation as nvarchar(1000)) AS DeliveryLocation
FROM Sales.Customers AS Customers
JOIN Application.People AS People
ON Customers.PrimaryContactPersonID = People.PersonID
JOIN Application.People AS ap
ON Customers.AlternateContactPersonID = ap.PersonID
JOIN Application.Cities AS Cities
ON Customers.DeliveryCityID = Cities.CityID
FOR JSON PATH, ROOT('Customers');
代码清单 8-32. 使用 `CAST()` 函数处理具有 geography 数据类型的列
```

## 总结

SQL Server 2016 引入了 JSON 集成功能。本章介绍了如何构建高效且有效的 JSON 输出。这是专门讨论 JSON 的第一章，你可以看到 `FOR JSON` 子句与 `FOR XML` 子句有许多相似之处。两者都有 `AUTO` 和 `PATH` 模式。`FOR JSON` 子句中的 `ROOT` 选项与 `FOR XML` 子句中的相同。根据经验，如果你熟悉 XML，那么学习 JSON 应该会很简单。

下一章将介绍如何将 JSON 值转换为行和列。



