# `第 16 章`

## `参数嗅探`

在上一章中，我讨论了如何将执行计划放入缓存以及如何重用它们。这是一个值得称赞的目标，也是提高系统整体性能的众多方法之一。确保计划重用的最佳机制之一是参数化查询，可以通过存储过程、预处理语句或 `sp_executesql` 实现。所有这些机制都会创建一个参数，在创建计划时用该参数代替硬编码值。这些参数可以被优化器采样或“嗅探”，以便在创建执行计划时使用其中包含的值。当这种情况顺利进行时（大多数时候确实如此），你会受益于更准确的计划。但当它出错并变成糟糕的参数嗅探时，你可能会遇到严重的性能问题。

在本章中，我将涵盖以下主题：

-   参数嗅探背后有益的机制
-   参数嗅探如何变糟
-   处理糟糕参数嗅探的机制

## 参数嗅探

当参数化查询被发送到优化器，并且缓存中没有现有的计划时，优化器将执行其功能，为按 T-SQL 语句请求操作数据创建执行计划。当这个参数化查询被调用时，参数的值会被设置（通过你的程序或参数定义中的默认值）。无论哪种方式，那里都有一个值。优化器知道这一点。因此，它利用这一事实读取参数的值。这就是被称为 *参数嗅探* 的过程中“嗅探”的部分。

有了这些可用值，优化器随后将使用这些特定值来查看参数所引用数据的统计信息。有了特定值和一套准确的统计信息，你将得到更好的执行计划。这种有益的参数嗅探过程会自动地、持续不断地为所有参数化查询运行，无论它们来自哪里。

你也可以对局部变量进行嗅探。不过在深入之前，让我们先区分一下局部变量和参数，因为在 T-SQL 语句中，它们看起来可能一样。这个例子同时展示了局部变量和参数：

```sql
CREATE PROCEDURE dbo.ProductDetails
    (@ProductID INT)
AS
DECLARE @CurrentDate DATETIME = GETDATE();
SELECT p.Name,
    p.Color,
    p.DaysToManufacture,
    pm.CatalogDescription
FROM Production.Product AS p
JOIN Production.ProductModel AS pm
    ON pm.ProductModelID = p.ProductModelID
WHERE p.ProductID = @ProductID
    AND pm.ModifiedDate < @CurrentDate;
GO
```

前面查询中的参数是 `@ProductID`。局部变量是 `@CurrentDate`。区分它们很重要，因为当你看到 `WHERE` 子句时，它们看起来完全一样。

如果使用局部变量的任何语句发生重新编译，这些变量可以被优化器以与嗅探参数相同的方式嗅探。只需意识到这一点。除了重新编译的这种情况外，当优化器尝试编译计划时，局部变量对它来说是未知量。通常只有参数可以被嗅探。

为了观察参数嗅探的实际效果并展示其有用性，让我们从一个不同的存储过程开始。

```sql
IF (SELECT OBJECT_ID('dbo.AddressByCity')
) IS NOT NULL
DROP PROC dbo.AddressByCity;
GO

CREATE PROC dbo.AddressByCity @City NVARCHAR(30)
AS
SELECT a.AddressID,
    a.AddressLine1,
    AddressLine2,
    a.City,
    sp.[Name] AS StateProvinceName,
    a.PostalCode
FROM Person.Address AS a
JOIN Person.StateProvince AS sp
    ON a.StateProvinceID = sp.StateProvinceID
WHERE a.City = @City;
```

创建存储过程后，使用此参数运行它：

```sql
EXEC dbo.AddressByCity @City = N'London';
```

这将导致以下 I/O 和执行时间，以及如图 16-1 所示的查询计划。

表 'Address'。扫描计数 1，逻辑读取 216
表 'StateProvince'。扫描计数 1，逻辑读取 3


