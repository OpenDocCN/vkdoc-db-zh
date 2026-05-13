# 第四章 „ 错误与异常

## XACT_ABORT 与事务

正如在介绍 `XACT_ABORT` 及其对异常影响的部分所提到的，这个设置也会对事务产生影响——其名称可能暗示了这一点（读作 *事务中止*）。除了使异常表现为批处理级异常之外，该设置还会在发生异常时，导致任何活动事务立即回滚。这意味着以下 T-SQL 会导致活动事务计数变为 0：

```sql
SET XACT_ABORT ON;

BEGIN TRANSACTION;

GO

SELECT 1/0 AS DivideByZero;

GO

SELECT @@TRANCOUNT AS ActiveTransactionCount;

GO
```

现在的输出是

```text
ActiveTransactionCount
----------------------
0
```

`XACT_ABORT` 不会在存储过程中创建隐式事务，但它*确实*会导致在存储过程中显式事务内发生的任何异常引发回滚。以下 T-SQL 展示的存储过程行为比之前的示例更具原子性：

```sql
--清空表
TRUNCATE TABLE SomeData;
GO

--此过程将插入一行，然后引发除零异常
CREATE PROCEDURE XACT_Rollback
AS
BEGIN
    SET XACT_ABORT ON;
    BEGIN TRANSACTION;
    INSERT INTO SomeData VALUES (1);
    INSERT INTO SomeData VALUES (1/0);
    COMMIT TRANSACTION;
END;
GO

--执行存储过程
EXEC XACT_Rollback;
GO

--从表中选择行
SELECT *
FROM SomeData;
GO
```

此 T-SQL 产生如下输出，显示没有行被插入：

```text
Msg 8134, Level 16, State 1, Procedure XACT_Rollback, Line 10
Divide by zero error encountered.

SomeColumn
-----------
(0 row(s) affected)
```

`XACT_ABORT` 是一种非常简单但极其有效的手段，用于确保异常不会导致事务在只完成部分工作的情况下提交。我建议在任何使用显式事务的存储过程中开启此设置，以保证在发生异常时事务能够回滚。

### TRY/CATCH 与注定失败的事务

使用 `TRY/CATCH` 语法的一个有趣结果是，事务可能进入一种*只能*被回滚的状态。在这种情况下，事务不会像使用 `XACT_ABORT` 那样自动回滚；相反，SQL Server 会抛出一个异常，告知调用者该事务无法提交，必须手动回滚。这种状况被称为**注定失败的事务**，以下 T-SQL 展示了一种产生它的方法：

```sql
BEGIN TRANSACTION;
BEGIN TRY
    --在插入时抛出异常
    INSERT INTO SomeData VALUES (CONVERT(int, 'abc'));
END TRY
BEGIN CATCH
    --尝试提交...
    COMMIT TRANSACTION;
END CATCH
GO
```

这会产生如下输出：

```text
Msg 3930, Level 16, State 1, Line 10
The current transaction cannot be committed and cannot support
operations that write to the log file. Roll back the transaction.
```

一旦事务进入此状态，任何尝试提交事务或向前推进（做更多工作）的操作都会导致相同的异常。此异常会不断被抛出，直到事务被回滚。

为了确定一个活动事务是否可以提交或向前推进，请检查 `XACT_STATE()` 函数的值。该函数在无活动事务时返回 0，事务处于可继续工作状态时返回 1，事务注定失败时返回 –1。在任何涉及显式事务的 `CATCH` 块中检查 `XACT_STATE()` 是一个好习惯。

## 小结

这是每个开发者都会面对的现实：有时事情就是会出错。

扎实理解异常在 SQL Server 中的行为方式，能使处理它们变得容易得多。尤为重要的是语句级异常与批处理级异常之间的区别，以及在事务内抛出异常的后果。

SQL Server 的 `TRY/CATCH` 语法让处理异常容易了许多，但明智地使用此功能很重要。过度使用会使问题的检测和调试变得异常困难。并且，每当在 `CATCH` 块中处理事务时，请务必检查 `XACT_STATE()` 的值。

错误和异常总会发生，但通过仔细思考如何处理它们，你就可以轻松有效地应对。

