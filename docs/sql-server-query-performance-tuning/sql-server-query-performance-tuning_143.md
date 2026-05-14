# 扩展事件输出与存储过程重编译

## 因常规表导致的重编译

***图 17-9.** 显示因常规表而重编译的存储过程的扩展事件输出*
可以看到 `SELECT` 语句在第二次执行时被重新编译。在第一次执行期间于存储过程中删除表，并不会删除保存在计划缓存中的查询计划。在随后执行存储过程时，现有计划仍包含了对该表的处理策略。但是，由于在存储过程中重新创建了该表，SQL Server 将其视为表架构的更改。

因此，在执行后续的存储过程语句之前，SQL Server 会在执行 `SELECT` 语句前重新编译存储过程内的语句。相应的 `sql_statement_recompile` 事件的 `recompile_clause` 值反映了重编译的原因。

## 因本地临时表导致的重编译

大多数时候，在存储过程中你会创建本地临时表而非常规表。为了理解本地临时表如何不同地影响存储过程重编译，只需将前面示例中的常规表替换为本地临时表即可。

```sql
IF (SELECT OBJECT_ID('dbo.TestProc')) IS NOT NULL
DROP PROC dbo.TestProc;
GO

CREATE PROC dbo.TestProc
AS
CREATE TABLE #ProcTest1 (C1 INT); --确保表不存在
SELECT * FROM #ProcTest1; --导致重编译
DROP TABLE #ProcTest1;
GO

EXEC dbo.TestProc; --第一次执行
EXEC dbo.TestProc; --第二次执行
```

由于本地临时表在存储过程执行结束时会自动删除，因此没有必要显式删除临时表。但是，遵循良好的编程实践，你可以在其工作完成后立即删除本地临时表。图 17-10 显示了前面示例的扩展事件输出。

***图 17-10.** 显示因本地临时表而重编译的存储过程的扩展事件输出* 334

可以看到查询在第一次执行时被重新编译。如相应的 `recompile_cause` 值所示，重编译的原因与常规表上的重编译原因相同。然而，请注意，当存储过程被重新执行时，它不会像常规表那样被重新编译。

在存储过程后续执行期间，本地临时表的架构与之前执行期间保持相同。本地临时表在存储过程的范围之外不可用，因此其架构无法在多次执行之间以任何方式被更改。因此，SQL Server 在存储过程的后续执行期间安全地重用现有计划（基于先前的本地临时表实例），从而避免了重编译。

> **注意：** 为避免重编译，在存储过程中使用本地临时表来保存中间结果集，而不是使用临时创建的常规表，是有意义的。但是，这只有在你能避免数据偏斜的情况下才成立，数据偏斜可能导致其他糟糕的执行计划。在那种情况下，重编译可能不那么痛苦。

#### SET 选项更改

存储过程的执行计划依赖于环境设置。如果在存储过程中更改了环境设置，那么 SQL Server 会在每次执行时重新编译查询。例如，考虑以下代码：

```sql
IF (SELECT OBJECT_ID('dbo.TestProc')) IS NOT NULL
DROP PROC dbo.TestProc;
GO

CREATE PROC dbo.TestProc
AS
SELECT 'a' + NULL + 'b'; --第一次
SET CONCAT_NULL_YIELDS_NULL OFF;
SELECT 'a' + NULL + 'b'; --第二次
SET ANSI_NULLS OFF;
SELECT 'a' + NULL + 'b'; --第三次
GO

EXEC dbo.TestProc; --第一次执行
EXEC dbo.TestProc; --第二次执行
```



