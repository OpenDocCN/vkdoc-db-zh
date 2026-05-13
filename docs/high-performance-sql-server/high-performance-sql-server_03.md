# 使用提示的注意事项

```
Msg 8622, Level 16, State 1, Line 119
Query processor could not produce a query plan because of the hints defined in this query. Resubmit the query without specifying any hints and without using SET FORCEPLAN.
```

在这个例子中，有趣且具有讽刺意味的是，查询优化器抱怨了一个提示。这些跟踪标志是未公开的，并且如前所述，出于任何原因都不应在生产环境中使用。具有讽刺意味的是，有一个文档记录且支持的方法可以以类似的方式改变优化器行为：使用提示。这是提示只应在没有其他解决方案可用的极端情况下使用的原因之一。如果你想通过一个提示获得与我们之前文本类似的行为，这是有文档记录且完全允许的，你可以使用以下查询：

```
SELECT c.CustomerID, COUNT(*)
FROM Sales.Customer c JOIN Sales.SalesOrderHeader o
ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID
OPTION (FORCE ORDER)
```

你将得到与使用未公开且不受支持的查询版本禁用 `GbAggBeforeJoin` 相同的计划，如下所示：

```
SELECT c.CustomerID, COUNT(*)
FROM Sales.Customer c JOIN Sales.SalesOrderHeader o
ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID
OPTION (RECOMPILE, QUERYRULEOFF GbAggBeforeJoin)
```

使用提示可能具有相同或非常相似的行为，但这次它是被允许且有文档记录的；但是，除非你有非常好的理由，否则不建议这样做。顺便说一下，`FORCE ORDER` 提示指定了在查询优化期间保留由查询语法指示的连接顺序和聚合位置。

### 备忘录

如前所述，备忘录是用于存储查询优化器生成和分析的备选方案的内存结构。每个优化过程都会创建一个新的备忘录结构，并且它只在优化过程中保留。最初，在解析和绑定期间创建的原始逻辑树将存储在备忘录中。随着优化过程中应用转换规则，新的逻辑和物理操作将存储在这里。

正如预期的那样，备忘录将存储在应用转换规则时创建的所有逻辑和物理操作，如之前所讨论的。成本估算组件将估算存储在此处的物理操作的成本。无需估算逻辑操作的成本。

你可以使用未公开的跟踪标志 `8608` 来显示初始备忘录结构，以及跟踪标志 `8615` 来显示其最终结构，从而可能看到备忘录结构的内容。让我们仅仅为了学术目的尝试一个快速示例。让我们处理以下查询：

```
DBCC TRACEON(3604)
SELECT ProductID, ListPrice FROM Production.Product
WHERE ListPrice > 90
OPTION (RECOMPILE, QUERYTRACEON 8606)
```

通过使用未公开的跟踪标志 `8606`，我们可以看到我们有以下最终逻辑树（这是我们之前展示的树表示，我们还没有检查备忘录结构）：

```
LogOp_Select
LogOp_Get TBL: Production.Product Production.Product TableID=482100758 TableReferenceID=0 IsRow: COL: IsBaseRow1000
ScaOp_Comp x_cmpGt
ScaOp_Identifier QCOL: [AdventureWorks2017].[Production].[Product].ListPrice
ScaOp_Const TI(money,ML=8) XVAR(money,Not Owned,Value=(10000units)=(900000))
```

同样地，我们可以使用未公开的跟踪标志 `8608` 查看这些初始操作符中的每一个是如何存储在备忘录中的。运行以下查询：

```
DBCC TRACEON(3604)
SELECT ProductID, ListPrice FROM Production.Product
WHERE ListPrice > 90
OPTION (RECOMPILE, QUERYTRACEON 8608)
```

你将得到以下输出，它显示了来自原始逻辑树的每个操作，例如 `LogOp_Select` 或 `LogOp_Get`，现在都存储在备忘录结构中：

```
DBCC execution completed. If DBCC printed error messages, contact your system administrator.
--- Initial Memo Structure ---
Root Group 4: Card=216 (Max=10000, Min=0)
0 LogOp_Select 3 2 (Distance = 0)
Group 3: Card=504 (Max=10000, Min=0)
0 LogOp_Get (Distance = 0)
Group 2:
0 ScaOp_Comp  0 1 (Distance = 0)
Group 1:
0 ScaOp_Const  (Distance = 0)
Group 0:
0 ScaOp_Identifier  (Distance = 0)
```

你也可以在自行承担风险的情况下，使用未公开的跟踪标志 `8615` 尝试相同的查询以显示最终的备忘录结构。

### 完全优化

我们刚刚回顾了转换规则如何工作。现在让我们定义完整的优化过程是如何工作的。如前所述，查询优化使用转换规则来生成替代的等效计划和操作，并使用启发式方法来清除或限制考虑的计划数量。还需要一个成本估算组件来选择最佳计划。记住，在我们之前的例子中，一个简单的优化，比如在连接之前移动分组聚合，就将计划的成本从 `0.336582` 改变为了 `0.309581`（稍后会详细介绍成本估算和这个成本值的含义）。

我们还看到，`sys.dm_exec_query_transformation_stats` DMV 列出了查询优化器可能使用的所有转换规则。显然，查询优化器不会为每个查询运行所有转换；它会根据查询特性运行所需的转换。它将对执行连接的查询应用与连接相关的转换，仅在查询使用聚合时才应用关于聚合的转换规则，依此类推。对于我们之前的示例，永远不会需要应用星型连接查询、列存储索引或窗口函数的规则，因为查询根本没有使用它们。

更进一步，它也不会同时执行所有适用的规则。并非所有的转换规则都在同一时间应用，相反，查询优化过程最多分为三个步骤或阶段执行，并且不需要全部执行。这意味着查询处理器可以在第一阶段找到一个好的执行计划并完成优化过程。如果在第一阶段没有找到好的计划，它将进入第二阶段，再次尝试找到优化器认为好的执行计划。如果仍然没有找到计划，则还会有第三阶段，此时将返回找到的最佳计划。这些阶段如下所示：

*   **Search 0** 或 **事务处理阶段**：此阶段将尝试尽快找到一个计划，而不尝试复杂的转换。此阶段对于通常出现在事务处理系统中的小型查询是最佳的，并且用于至少有三个表的查询。
*   **Search 1** 或 **快速计划**：`Search 1` 使用额外的转换规则、有限的连接重新排序，更适用于复杂查询。
*   **Search 2** 或 **完全优化**：`Search 2` 用于从复杂到非常复杂的查询。在此阶段会考虑更大一组的潜在转换规则、并行操作符和其他高级优化策略。

有几种方法，一些是有文档记录的，一些是没有的，可以查看在特定查询优化期间执行了哪些阶段。让我们从有文档记录的方法开始，或者更确切地说是部分有文档记录的，即使用 `sys.dm_exec_query_optimizer_info` DMV。

**注意**

`sys.dm_exec_query_optimizer_info` DMV 在其他许多方面也非常有趣，所以我也会在本节后面讨论其中的一些方面。

运行以下语句：

```
SELECT * from sys.dm_exec_query_optimizer_info
```

这是我访问的一台生产系统当前的输出。



