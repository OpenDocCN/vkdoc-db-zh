# 第 15 章：执行计划缓存行为

通过访问各种动态管理对象，你可以获取关于过程缓存中执行计划的大量信息。用于处理执行计划的初始 DMO 是 `sys.dm_exec_cached_plans`。

```sql
SELECT *
FROM sys.dm_exec_cached_plans;
```

表 15-1 展示了 `sys.dm_exec_cached_plans` 提供的一些有用信息（在网格视图下更易阅读）。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 表 15-1。`sys.dm_exec_cached_plans`

| 列名 | 描述 |
| :--- | :--- |
| `refcounts` | 这表示缓存中引用此计划的其他对象的数量。 |
| `usecounts` | 这是此对象自添加到缓存以来被使用的次数。 |
| `size_in_bytes` | 这是存储在缓存中的计划的大小。 |
| `cacheobjtype` | 这指定了计划的类型；有多种类型，但特别值得关注的有以下几种：<br>• `Compiled plan`：一个完整的执行计划。<br>• `Compiled plan stub`：用于即席查询的标记（你可以在本章的“即席工作负载”小节中找到更多细节）。<br>• `Parse tree`：为访问视图而存储的计划。 |
| `objtype` | 这是生成该计划的对象类型。同样有多种，但特别值得关注的有：<br>• `Proc`<br>• `Prepared`<br>• `Ad hoc`<br>• `View` |
| `plan_handle` | 这是内存中此计划的标识符；用于检索查询文本和执行计划。 |

单独使用 DMV `sys.dm_exec_cached_plans` 只能让你获得一小部分信息。DMO 最好与其他 DMO 和其他系统视图结合使用。例如，将动态管理函数 `sys.dm_exec_query_plan(plan_handle)` 与 `sys.dm_exec_cached_plans` 结合使用，还会返回 XML 执行计划本身，这样你就可以显示它并进行操作。如果你再引入 `sys.dm_exec_sql_text(plan_handle)`，你还可以检索原始查询文本。在你为这里的示例运行已知查询时，这可能看起来没用，但当你进入生产系统并开始从缓存中提取执行计划时，能够获取原始查询可能会很方便。要获取有关缓存计划的聚合性能指标，你可以使用 `sys.dm_exec_query_stats` 来返回该数据。其中包括查询哈希和查询计划哈希等信息都存储在此 DMF 中。最后，要查看当前正在执行的查询的执行计划，你可以使用 `sys.dm_exec_requests`。

在接下来的小节中，我将通过实际查询这些 DMO 来探讨计划缓存的工作原理。

## 执行计划重用

当提交一个查询时，SQL Server 会检查过程缓存中是否有匹配的执行计划。如果没有找到，那么 SQL Server 会执行查询编译和优化以生成一个新的执行计划。但是，如果该计划存在于过程缓存中，它将与私有执行上下文一起被重用。这节省了本该花费在计划生成上的 CPU 周期。

查询提交给 SQL Server 时会带有用于限制结果集大小的过滤条件。相同的查询经常使用不同的过滤条件值重新提交。例如，考虑以下查询：

```sql
SELECT soh.SalesOrderNumber,
       soh.OrderDate,
       sod.OrderQty,
       sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod
ON soh.SalesOrderID = sod.SalesOrderID
WHERE soh.CustomerID = 29690
  AND sod.ProductID = 711;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

当此查询被提交时，优化器会创建一个执行计划并将其保存在过程缓存中，以备将来重用。如果此查询使用不同的过滤条件值重新提交——例如 `soh.CustomerID = 29500`——重用先前提供的过滤条件值的现有执行计划将是有益的。但是，为一个过滤条件值创建的执行计划是否可以重用于另一个过滤条件值，取决于查询是如何提交给 SQL Server 的。

提交给 SQL Server 的查询（或工作负载）可以大致分为两类，这决定了当查询的可变部分的值发生变化时，执行计划是否可重用。

*   即席
*   预编译

**提示：** 要测试本章中 `sys.dm_exec_cached_plans` 的输出，有时需要通过执行 `DBCC FREEPROCCACHE` 来从缓存中移除计划。不要在你的生产服务器上运行此命令，因为清空缓存将要求所有执行计划在执行时重新构建，这会对你的生产系统造成无谓的严重压力。你可以使用 `DBCC FREEPROCCACHE(plan_handle)` 来针对特定的计划。使用我已经讨论过的 DMO 来检索 `plan_handle`。

## 即席工作负载

查询可以在没有显式地将变量与查询分离的情况下提交给 SQL Server。这些没有显式地将查询的可变部分转换为参数来执行的查询类型被称为 `即席工作负载`（或查询）。到目前为止本书中的大多数示例都是即席查询。再举一个例子，考虑这个查询：

```sql
SELECT soh.SalesOrderNumber,
       soh.OrderDate,
       sod.OrderQty,
       sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod
ON soh.SalesOrderID = sod.SalesOrderID
WHERE soh.CustomerID = 29690
  AND sod.ProductID = 711;
```

如果查询按原样提交，没有显式地将任何一个硬编码值转换为（可以在执行时提供给查询的）参数，那么该查询就是即席查询。使用 `DECLARE` 语句将值设置为局部变量与参数不同。

在这个查询中，过滤条件值嵌入在查询本身中，没有显式地参数化以将其与查询分离。这意味着你无法重用此查询的执行计划，除非你使用相同的值并且所有空格和回车符都完全相同。然而，查询中使用值的地方可以通过三种不同的方式显式地参数化，这些方式共同归类为预编译工作负载。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 预编译工作负载

`预编译工作负载`（或查询）显式地参数化查询的可变部分，使得查询计划不与可变部分的值绑定。在 SQL Server 中，可以使用以下三种方法将查询作为预编译工作负载提交：

*   **存储过程：** 允许保存可接受和返回用户提供参数的一组 SQL 语句。
*   **`sp_executesql`：** 允许执行可能包含用户提供参数的 SQL 语句或 SQL 批处理，而不保存该 SQL 语句或批处理。
*   **准备/执行模型：** 允许 SQL 客户端请求生成一个查询计划，该计划可以在后续使用不同参数值执行查询时重用，而无需在 SQL Server 中保存 SQL 语句。

例如，前面显示的 `SELECT` 语句可以使用存储过程显式地参数化，如下所示：

```sql
IF (SELECT OBJECT_ID('dbo.BasicSalesInfo')) IS NOT NULL
    DROP PROC dbo.BasicSalesInfo;
GO
CREATE PROC dbo.BasicSalesInfo
    @ProductID INT,
    @CustomerID INT
AS
SELECT soh.SalesOrderNumber,
       soh.OrderDate,
       sod.OrderQty,
       sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod
ON soh.SalesOrderID = sod.SalesOrderID
WHERE soh.CustomerID = @CustomerID
  AND sod.ProductID = @ProductID;
```



