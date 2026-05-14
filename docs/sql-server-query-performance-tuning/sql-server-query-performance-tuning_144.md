# 查询重编译

#### 执行计划老化

SQL Server 通过维护缓存中执行计划的“年龄”来管理过程缓存的大小，正如你在第 15 章所见。如果一个存储过程长时间没有重新执行，其执行计划的年龄字段可能降至 0，并且该计划可能因内存压力而被从缓存中移除。当这种情况发生且存储过程被重新执行时，将生成一个新计划并缓存到过程缓存中。然而，如果系统中有足够的内存，未使用的计划在内存压力增加之前不会从缓存中移除。

#### 显式调用 sp_recompile

当架构更改或统计信息发生足够变化时，SQL Server 会自动重编译查询。它还提供了 `sp_recompile` 系统存储过程来手动将整个存储过程标记为待重编译。

可以在表、视图、存储过程或触发器上调用此存储过程。如果在存储过程或触发器上调用，则该存储过程或触发器将在下次执行时重编译。在表或视图上调用 `sp_recompile` 会将引用该表/视图的所有存储过程和触发器标记为在下次执行时重编译。

例如，如果在表 `Test1` 上调用 `sp_recompile`，所有引用表 `Test1` 的存储过程和触发器都将被标记为待重编译，并在下次执行时重编译，如下所示：

```sql
sp_recompile 'Test1';
```

在使用 `sp_executesql` 执行动态查询时，可以使用 `sp_recompile` 来取消对现有计划的重用。如前一章所述，对于变量部分的值范围可能需要不同处理策略的查询，不应将其参数化。例如，重新考虑相应的示例，你知道查询的第二次执行重用了为第一次执行生成的计划。为了便于参考，此处重复该示例：

```sql
DBCC FREEPROCCACHE; -- 清除过程缓存
GO
DECLARE @query NVARCHAR(MAX);
DECLARE @param NVARCHAR(MAX);
SET @query = N'SELECT soh.SalesOrderNumber, soh.OrderDate, sod.OrderQty, sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod ON soh.SalesOrderID = sod.SalesOrderID
WHERE soh.CustomerID >= @CustomerId;';
SET @param = N'@CustomerId INT';
EXEC sp_executesql @query, @param, @CustomerId = 1;
EXEC sp_executesql @query, @param, @CustomerId = 30118;
```

查询的第二次执行对 `SalesOrderHeader` 表执行索引扫描操作以检索数据。如第 8 章所述，对于第二次执行，可能更倾向于在 `SalesOrderHeader` 表上执行索引查找操作。你可以通过以下方式在 `SalesOrderHeader` 表上执行 `sp_recompile` 系统存储过程来实现这一点：

```sql
EXEC sp_recompile 'Sales.SalesOrderHeader'
```

现在，如果使用第二个参数值重新执行该查询，查询的计划将如前面的 `sp_recompile` 语句所标记的那样被重编译。这使得 SQL Server 能够为第二次执行生成最优计划。

嗯，这里有一个小问题：你可能会希望再次执行第一条语句。由于计划存在于缓存中，SQL Server 将为第一条语句重用该计划（在 `SalesOrderHeader` 表上的索引扫描操作），即使使用索引查找操作（利用过滤条件列 `soh.CustomerID` 上的索引）本来会更优。避免此问题的一种方法是为查询创建一个存储过程，并在语句上使用 `OPTION (RECOMPILE)` 子句。接下来我将介绍控制重编译的各种方法。

#### 显式使用 RECOMPILE

SQL Server 允许通过三种方式使用 `RECOMPILE` 命令显式重编译存储过程和查询：在 `CREATE PROCEDURE` 语句中、作为 `EXECUTE` 语句的一部分以及在查询提示中。这些方法会降低计划重用的效率，因此你应在以下部分解释的具体情况下才考虑它们。

##### RECOMPILE 子句与 CREATE PROCEDURE 语句

有时，存储过程的计划需求会随着传递给存储过程的参数值的变化而变化。在这种情况下，对不同参数值重用同一个计划可能会降低存储过程的性能。你可以通过在 `CREATE PROCEDURE` 语句中使用 `RECOMPILE` 子句来避免此问题。例如，对于上一节中的查询，你可以使用 `RECOMPILE` 子句创建一个存储过程。

```sql
IF (SELECT OBJECT_ID('dbo.CustomerList')) IS NOT NULL
    DROP PROC dbo.CustomerList;
GO
CREATE PROCEDURE dbo.CustomerList @CustomerId INT
WITH RECOMPILE
AS
SELECT soh.SalesOrderNumber,
       soh.OrderDate,
       sod.OrderQty,
       sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod
    ON soh.SalesOrderID = sod.SalesOrderID
WHERE soh.CustomerID >= @CustomerId;
GO
```

`RECOMPILE` 子句会阻止过程内每个语句的存储过程计划被缓存。每次执行存储过程时，都会生成新计划。因此，如果使用 `soh.CustomerID` 值 30118 或 1 执行存储过程：

```sql
EXEC CustomerList @CustomerId = 1;
EXEC CustomerList @CustomerId = 30118;
```

会在每次执行期间生成一个新计划，如图 17-12 所示。

##### RECOMPILE 子句与 EXECUTE 语句

如前所示，存储过程中的特定参数值根据其性质可能需要不同的计划。你可以将 `RECOMPILE` 子句从存储过程中移出，并在执行存储过程时根据具体情况使用它，如下所示：

```sql
EXEC dbo.CustomerList
    @CustomerId = 1
    WITH RECOMPILE;
```

当使用 `RECOMPILE` 子句执行存储过程时，会临时生成一个新计划。新计划不会被缓存，也不会影响现有计划。当不使用 `RECOMPILE` 子句执行存储过程时，计划照常缓存。与在 `CREATE PROCEDURE` 语句中使用 `RECOMPILE` 子句相比，这提供了对现有计划缓存重用性的某种程度控制。



由于带有 `RECOMPILE` 子句执行的存储过程的计划不会被缓存，因此每次使用 `RECOMPILE` 子句执行该存储过程时，都会重新生成计划。然而，为了获得更好的性能，与其使用 `RECOMPILE`，不如考虑为每个需要不同计划的参数值集分别创建独立的存储过程——前提是这些计划易于识别，且您只需处理少量可能的计划。

##### 使用 RECOMPILE 提示控制单个语句

虽然可以使用前述任一方法来重新编译整个过程，但如果过程包含多条命令，这可能会带来问题。使用前述任一方法时，过程中的所有语句都将被重新编译。对于某些查询，编译时间可能是执行过程中最昂贵的部分，因此应尽量避免重新编译。为此，一种更精细的方法是将重新编译隔离到仅需要它的语句。这可以通过使用 `RECOMPILE` 查询提示来实现，如下所示：

```sql
IF (SELECT OBJECT_ID('dbo.CustomerList')
) IS NOT NULL
DROP PROC dbo.CustomerList;
GO

CREATE PROCEDURE dbo.CustomerList @CustomerId INT
AS
SELECT soh.SalesOrderNumber,
       soh.OrderDate,
       sod.OrderQty,
       sod.LineTotal
FROM Sales.SalesOrderHeader AS soh
JOIN Sales.SalesOrderDetail AS sod
  ON soh.SalesOrderID = sod.SalesOrderID
WHERE soh.CustomerID >= @CustomerId
OPTION (RECOMPILE);
GO
```

此过程的表现看起来与将 `RECOMPILE` 应用于整个过程时相同，但如果您向此查询添加多个语句，则只有带有 `OPTION (RECOMPILE)` 查询提示的语句会在每次执行过程时进行编译。

## 避免重新编译

有时重新编译是有益的，但有时则值得避免。如果在查询的 `WHERE` 或 `JOIN` 子句引用的列上创建了新索引，那么重新生成引用该表的存储过程的执行计划是合理的，这样它们才能从使用索引中受益。但是，如果认为重新编译对性能有害（例如导致阻塞或消耗 CPU 等资源），您可以通过以下实现实践来避免它：

-   不要交错使用 DDL 和 DML 语句。
-   避免由统计信息更改引起的重新编译。
-   使用 `KEEPFIXED PLAN` 选项。
-   禁用表上的自动更新统计信息功能。
-   使用表变量。
-   避免在存储过程中更改 `SET` 选项。
-   使用 `OPTIMIZE FOR` 查询提示。
-   使用计划指南。

### 不要交错 DDL 和 DML 语句

在存储过程中，DDL 语句常用于创建本地临时表并更改其架构（包括添加索引）。这样做会影响现有计划的有效性，并可能导致在执行引用这些表的存储过程语句时发生重新编译。要理解对本地临时表使用 DDL 语句如何导致存储过程重复重新编译，请考虑以下示例：

```sql
IF (SELECT OBJECT_ID('dbo.TempTable')
) IS NOT NULL
DROP PROC dbo.TempTable
GO

CREATE PROC dbo.TempTable
AS
CREATE TABLE #MyTempTable (ID INT,Dsc NVARCHAR(50))
INSERT INTO #MyTempTable
(ID,
 Dsc)
SELECT pm.ProductModelID,
       pm.[Name]
FROM Production.ProductModel AS pm; --需要第 1 次重新编译

SELECT *
FROM #MyTempTable AS mtt;

CREATE CLUSTERED INDEX iTest ON #MyTempTable (ID);

SELECT *
FROM #MyTempTable AS mtt; --需要第 2 次重新编译

CREATE TABLE #t2 (c1 INT);

SELECT *
FROM #t2;
--需要第 3 次重新编译
GO

EXEC dbo.TempTable; --首次执行
```

此存储过程交错使用了 DDL 和 DML 语句。图 17-13 展示了此代码的扩展事件输出。


