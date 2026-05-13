# 第四章 - 错误与异常

`TRY/CATCH` 可以用来完全封装异常的一个有趣例子是处理死锁。虽然尝试找到并解决死锁的根源比通过编码绕开它更好，但这通常是一项困难且耗时的任务。因此，至少在临时情况下，通过让应用程序重新发出导致死锁的请求来处理死锁是很常见的。最终，死锁条件会自行解决（即，当另一个事务完成时），而 DML 操作将按预期执行。请注意，我不推荐将此作为解决反复发生的死锁情况的长期方案！

通过使用 T-SQL 的 `TRY/CATCH` 语法，应用程序不再需要重新发出请求，甚至不需要知道发生了问题。可以设置一个重试循环，其中易发生死锁的代码可以在 `TRY` 块中尝试，而死锁则在 `CATCH` 块中被捕获，以便再次尝试。

下面是一个重试循环的基本实现：

```sql
DECLARE @Retries int;

SET @Retries = 3;

WHILE @Retries > 0

BEGIN

    BEGIN TRY

        /*

        将易发生死锁的代码放在这里

        */

        --如果执行到达这里，表示成功

        BREAK;

    END TRY

    BEGIN CATCH

        IF ERROR_NUMBER() = 1205

        BEGIN

            SET @Retries = @Retries - 1;

            IF @Retries = 0

                RAISERROR('无法完成事务!', 16, 1);

        END

        ELSE

            RAISERROR('遇到非死锁条件', 16, 1);

        BREAK;

    END CATCH

END;

GO
```

在这个例子中，易发生死锁的代码会按照 `@Retries` 的值进行重试。每次循环中，代码都会被尝试执行。如果它成功执行而没有抛出异常，代码就会执行到 `BREAK` 语句，循环结束。否则，执行会跳转到 `CATCH` 块，在那里会检查错误编号是否为 1205（死锁牺牲品）。如果是，计数器会递减，以便可以再次尝试循环。如果异常不是死锁，则会抛出另一个异常，以便调用者知道出了问题。确保错误的异常不会触发重试非常重要。

## 异常处理与防御性编程

异常处理极其有用，它在 T-SQL 中的应用更是无价之宝。然而，我希望所有读者记住，异常处理不能替代在错误条件发生*之前*对其进行的适当检查。只要可能，就应编写防御性代码——主动寻找问题，如果问题既能被检测到又能被处理，就编写代码来应对它们。

请记住，通常处理异常比处理错误是更好的主意。如果你能在开发期间预测到一个条件并编写代码路径来处理它，这通常会提供比试图在异常发生时捕获并处理它更健壮的解决方案。

### 异常处理与 SQLCLR

.NET Framework 提供了其自己的异常处理机制，该机制与用于处理 T-SQL 中遇到的异常的机制完全不同。那么，当在 SQL Server 托管的 SQLCLR 进程中执行的 CLR 代码发生异常时，这两个系统如何交互呢？

让我们看一个例子——下面的 C# 代码展示了一个简单的 CLR 用户定义函数 (UDF)，用于将一个数除以另一个数：

```csharp
[Microsoft.SqlServer.Server.SqlFunction()]

public static SqlDecimal Divide(SqlDecimal x, SqlDecimal y)

{

    return x / y;

}
```

当编录并在 SQL Server 中被调用，且 `y` 参数值为 0 时，结果如下：

```
Msg 6522, Level 16, State 2, Line 1

在执行用户定义例程或聚合 "Divide" 期间发生了 .NET Framework 错误:

System.DivideByZeroException: 遇到除以零错误。

System.DivideByZeroException:

   at System.Data.SqlTypes.SqlDecimal.op_Division(SqlDecimal x, SqlDecimal y)

   at ExpertSQLServer.UserDefinedFunctions.Divide(SqlDecimal x, SqlDecimal y)

.
```

SQL Server 会自动在其进程中执行的任何托管代码周围包装一个异常处理器。


