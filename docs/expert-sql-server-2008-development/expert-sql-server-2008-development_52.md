# 第四章 错误与异常

异常，然后有意引发这些异常，以此作为控制应用程序流程的一种方法。这种方法有一些可能的好处：引发异常是立即跳出例程（无论该例程在调用栈中嵌套得多深）的有效方式；而且，异常可用于描述那些用方法的正常返回值难以表达的情况。然而，尽管在这些场景中使用异常似乎很方便，但它们显然滥用了异常处理代码的预期目的（请记住，其目的是处理*异常*情况）。此外，大量调用异常会使你的代码难以阅读，并且可能使调试变得更加困难，因为调试器通常在遇到异常时会实现特殊行为。因此，与依赖更标准控制流结构的代码相比，充满异常的代码更难以维护。

在我看来，你只应在描述真正异常情况时引发异常。此外，由于异常可能导致中止条件，因此应谨慎使用。然而，使用异常而非错误确实有一个好处，那就是调用者更难忽略异常，因为如果未得到正确处理，它将导致代码中止。如果你正在设计一个需要确保调用者在某个条件发生时绝对能看到它的接口，那么使用异常而非错误可能是有意义的。

与几乎所有情况一样，这个决定很大程度上取决于你应用程序的上下文，所以不必觉得必须遵循我的意见！

## SQL Server 中异常的工作原理

理解如何在 SQL Server 中处理错误和异常的第一步，是看看服务器本身如何处理错误条件。与许多其他编程语言不同，SQL Server 的异常模型对不同类型的异常有不同的行为。当错误条件确实发生时，这可能导致意外行为，因此在处理 T-SQL 异常时，谨慎编程至关重要。

首先，考虑连接到 SQL Server 并发出一些 T-SQL。首先，你必须通过提供登录凭据来建立与服务器的`连接`。该连接还确定哪个数据库将用作作用域解析（即查找对象——稍后会详细介绍）的默认数据库。

一旦连接上，你就可以发出一批 T-SQL。一批由一个或多个 T-SQL`语句`组成，这些语句将被一起编译以形成一个执行计划。

SQL Server 抛出的异常行为大多遵循相同的模式：根据异常类型，一条语句、一个批处理或整个连接可能会被中止。让我们看一些实际例子来观察这种行为。

### 语句级异常

语句级异常只中止在一批 T-SQL 中当前正在运行的语句，允许该批中的任何后续语句运行。要查看此行为，你可以使用 SQL Server Management Studio 执行一个包含异常，后跟一条`PRINT`语句的批处理。

例如：

```sql
SELECT POWER(2, 32);
PRINT 'This will print!';
GO
```

运行此批处理会产生以下输出：

```text
Msg 232, Level 16, State 3, Line 1
Arithmetic overflow error for type int, value = 4294967296.000000.
This will print!
```

当运行此批处理时，尝试计算`POWER(2, 32)`导致了整数溢出，从而抛出了异常。然而，只有`SELECT`语句被中止。批处理的其余部分继续运行，在本例中，这意味着`PRINT`语句仍然打印了其消息。

### 批处理级异常

与语句级异常不同，批处理级异常不允许批处理的其余部分继续运行。引发异常的语句将被中止，批处理中剩余的任何语句都不会运行。一个批处理中止异常的例子是无效转换，如下所示：

```sql
SELECT CONVERT(int, 'abc');
PRINT 'This will NOT print!';
GO
```

此批处理的输出如下：

```text
Msg 245, Level 16, State 1, Line 1
Conversion failed when converting the varchar value 'abc' to data type int.
```

在这种情况下，转换异常发生在`SELECT`语句中，这在该点中止了批处理。`PRINT`语句未被允许运行，尽管如果批处理在异常发生*之前*包含任何有效语句，这些语句本应已成功执行。

批处理级异常很容易与连接级异常（它们会断开与服务器的连接）混淆，但在批处理级异常之后，连接仍然可以自由发送其他批处理。例如：

```sql
SELECT CONVERT(int, 'abc');
GO
PRINT 'This will print!';
GO
```

在此情况下，有两个批处理被发送到 SQL Server，由批处理分隔符`GO`分隔。第一个批处理抛出了转换异常，但第二个批处理仍然运行。这导致以下输出：

```text
Msg 245, Level 16, State 1, Line 2
Conversion failed when converting the varchar value 'abc' to data type int.
This will print!
```

批处理级异常不仅影响异常发生的作用域。异常将`向上冒泡`到下一级执行，中止堆栈中的每一个调用。这可以通过创建以下存储过程来说明：

```sql
CREATE PROCEDURE ConversionException
AS
BEGIN
    SELECT CONVERT(int, 'abc');
END;
GO
```

运行此存储过程后跟一条`PRINT`语句表明，即使异常发生在内部作用域（存储过程中），外部批处理仍然被中止：

```sql
EXEC ConversionException;
PRINT 'This will NOT print!';
GO
```

此批处理的结果与未使用存储过程时相同：

```text
Msg 245, Level 16, State 1, Line 4
Conversion failed when converting the varchar value 'abc' to data type int.
```

## 解析和作用域解析异常

在解析或编译的作用域解析阶段发生的异常，起初表现得与批处理级异常一模一样。然而，它们的行为实际上略有不同。如果异常与批处理的其余部分发生在同一作用域中，这些异常的行为将与批处理级异常完全一样。另一方面，如果异常发生在较低级别的作用域中，这些异常的行为将与语句级异常完全一样——至少，就外部批处理而言是这样。

作为一个例子，考虑以下批处理，其中包含一个格式错误的`SELECT`语句（这是一个解析异常）：

```sql
SELECTxzy FROM SomeTable;
PRINT 'This will NOT print!';
GO
```

在这种情况下，`PRINT`语句没有运行，因为在解析阶段整个批处理被丢弃。输出是以下异常消息：

```text
Msg 156, Level 15, State 1, Line 1
Incorrect syntax near the keyword 'FROM'.
```

要看到行为上的差异，可以使用`EXEC`函数将`SELECT`语句作为动态 SQL 执行。这导致`SELECT`语句在不同的作用域中执行，将异常行为从类似批处理变为类似语句。尝试运行以下 T-SQL 来观察变化：

```sql
EXEC('SELECTxzy FROM SomeTable');
PRINT 'This will print!';
GO
```

现在，`PRINT`语句被执行了，尽管异常发生了：

```text
Msg 156, Level 15, State 1, Line 1
Incorrect syntax near the keyword 'FROM'.
This will print!
```

