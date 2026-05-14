# 第 18 章 重编译：问题与解决方案

执行`sp_create_plan_guide`创建计划指南。

```
EXECUTE sp_create_plan_guide @name = N'MyBadSQLGuide',
@stmt = N'SELECT  soh.SalesOrderNumber,
soh.OrderDate,
sod.OrderQty,
sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
join Sales.SalesOrderDetail AS sod
ON soh.SalesOrderID = sod.SalesOrderID
WHERE   soh.CustomerID >= @CustomerID',
@type = N'SQL',
@module_or_batch = NULL,
@params = N'@CustomerID int',
@hints = N'OPTION  (TABLE HINT(soh,  FORCESEEK))';
```

当运行`SELECT`查询时，你仍然会得到相同的执行计划。这是因为查询看起来与计划指南中键入的内容不同。有几处差异，例如间距和`JOIN`语句的大小写。你可以使用 T-SQL 语句删除这个错误的计划指南。

```
EXECUTE sp_control_plan_guide @operation = 'Drop',
@name = N'MyBadSQLGuide';
```

输入正确的语法将创建一个新的计划。

```
EXECUTE sp_create_plan_guide @name = N'MyGoodSQLGuide',
@stmt = N'SELECT soh.SalesOrderNumber,
soh.OrderDate,
sod.OrderQty,
sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod
ON soh.SalesOrderID = sod.SalesOrderID
WHERE soh.CustomerID >= 1;',
@type = N'SQL',
@module_or_batch = NULL,
@params = NULL,
@hints = N'OPTION  (TABLE HINT(soh,  FORCESEEK))';
```

现在运行查询时，会创建一个完全不同的计划，如图 18-20 所示。

![../images/323849_5_En_18_Chapter/323849_5_En_18_Fig20_HTML.jpg](img/323849_5_En_18_Fig20_HTML.jpg)

图 18-20

计划指南强制对同一查询使用新的执行计划

当你缓存中有一个计划，并且你认为它的执行方式符合预期时，还存在另一个选项。你可以将该计划捕获到计划指南中，以确保下次运行查询时执行相同的计划。这通过运行`sp_create_plan_guide_from_handle`来完成。

为了测试，首先清除过程缓存（假设我们不在生产实例上运行），以便精确控制使用哪个查询计划。

```
ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE_CACHE
```

在过程缓存清空且现有计划指南`MyGoodSQLGuide`就位的情况下，重新运行查询。它将使用计划指南来获得如图 18-18 所示的执行计划。要查看此计划是否可以保持，首先删除强制`Index Seek`操作的计划指南。

```
EXECUTE sp_control_plan_guide @operation = 'Drop',
@name = N'MyGoodSQLGuide';
```

如果现在重新运行查询，它将恢复到其原始计划。然而，当前在计划缓存中，你有如图 18-18 所示的计划。要保留它，请运行以下脚本：

```
DECLARE @plan_handle VARBINARY(64),
@start_offset INT;
SELECT @plan_handle = deqs.plan_handle,
@start_offset = deqs.statement_start_offset
FROM sys.dm_exec_query_stats AS deqs
CROSS APPLY sys.dm_exec_sql_text(sql_handle)
CROSS APPLY sys.dm_exec_text_query_plan(deqs.plan_handle,
deqs.statement_start_offset,
deqs.statement_end_offset) AS qp
WHERE text LIKE N'SELECT soh.SalesOrderNumber%'
EXECUTE sp_create_plan_guide_from_handle @name = N'ForcedPlanGuide',
@plan_handle = @plan_handle,
@statement_start_offset = @start_offset;
GO
```

这基于当前缓存中存在的执行计划创建了一个计划指南。为了确保其有效，再次清除缓存。这样，查询必须生成一个新计划。重新运行查询，并观察执行计划。由于使用`sp_create_plan_guide_from_handle`创建的计划指南，它将与图 18-19 所示的计划相同。

计划指南是控制 SQL 查询和存储过程行为的有用机制，但你应该仅在彻底理解执行计划、数据和系统结构时才使用它们。

### 使用查询存储强制计划

与`OPTIMIZE FOR`和计划指南一样，通过查询存储强制计划实际上并不会减少系统上的重编译次数。然而，它将允许你更好地控制这些重编译的结果。类似于计划指南的工作原理，随着数据随时间变化，你可能需要重新评估已强制的计划（参见第 11 章）。

## 小结

正如你在本章所学，查询重编译既可能有益也可能损害性能。生成更好计划的重编译提高了存储过程的性能。然而，重新生成相同计划的重编译会消耗额外的 CPU 周期，而没有任何处理策略的改进。因此，你应该仔细检查重编译以确定其有用性。你可以使用扩展事件来识别是哪个存储过程语句导致了重编译，并且你可以从扩展事件输出中的`recompile_clause`数据列值确定原因。一旦确定了重编译的原因，你就可以应用不同的技术来避免不必要的重编译。

到目前为止，你已经了解了如何从正确的索引和计划缓存中受益。然而，这些技术的性能优势取决于查询的设计方式。SQL Server 的成本优化器处理了许多查询设计问题。但是，在设计查询时，你应该采用一些最佳实践。在下一章中，我将介绍一些影响性能的常见查询设计问题。

