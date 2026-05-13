# 第 4 章 错误与异常

在`BEGIN TRY`和`END TRY`之间发生的任何类型异常——连接或服务器级别异常除外——都将导致`BEGIN CATCH`和`END CATCH`之间的代码立即执行，绕过`TRY`块中剩余的任何代码。

作为第一个示例，考虑以下 T-SQL：

```sql
BEGIN TRY
    SELECT 1/0 AS DivideByZero;
END TRY
BEGIN CATCH
    SELECT 'Exception Caught!' AS CatchMessage;
END CATCH
```

运行此批处理会产生以下输出：

```
DivideByZero
CatchMessage
Exception Caught!
```

这里需要注意的是，首先，没有报告异常。我们可以看到发生了异常，因为代码执行跳到了`CATCH`块，但异常被成功处理了，客户端并不知道发生了异常。其次，请注意导致异常的`SELECT`语句返回了一个空结果集。如果异常没有被处理，则不会返回任何结果集。通过返回空结果集，`SELECT`语句的隐含契约得到了履行（或多或少，取决于客户端实际期望什么）。

虽然已经提到过，但需要强调的是，当使用`TRY/CATCH`时，在`TRY`块中遇到的所有异常都将立即中止`TRY`块剩余部分的执行。

因此，以下 T-SQL 的输出与上一个示例完全相同：

```sql
BEGIN TRY
    SELECT 1/0 AS DivideByZero;
    SELECT 1 AS NoError;
END TRY
BEGIN CATCH
    SELECT 'Exception Caught!' AS CatchMessage;
END CATCH
```

## 获取`CATCH`块中的扩展错误信息

除了能够捕获异常之外，SQL Server 2008 还提供了一系列可在`CATCH`块中使用的附加函数。这些函数（列表如下）使开发人员能够编写代码来检索`TRY`块中发生的异常的信息。

- `ERROR_MESSAGE`
- `ERROR_NUMBER`
- `ERROR_SEVERITY`
- `ERROR_STATE`
- `ERROR_LINE`
- `ERROR_PROCEDURE`

这些函数不接受输入参数，并且根据其名称来看，其含义相当直观。

然而，重要的是要指出，与`@@ERROR`不同，这些函数返回的值在每个语句之后不会被重置。它们在整个`CATCH`块中是持久的。因此，以下 T-SQL 中使用的逻辑是有效的：

```sql
BEGIN TRY
    SELECT CONVERT(int, 'ABC') AS ConvertException;
END TRY
BEGIN CATCH
    IF ERROR_NUMBER() = 123
        SELECT 'Error 123';
    ELSE
        SELECT ERROR_NUMBER() AS ErrorNumber;
END CATCH
```

正如预期的那样，在这种情况下，错误编号被正确报告：

```
ConvertException
ErrorNumber
```

这些函数，特别是`ERROR_NUMBER`，允许为某些异常编写特定的代码路径。

例如，如果开发人员知道某段代码可能会引发可以程序化修复的异常，则可以在`CATCH`块中检查该异常编号。

#### 重新抛出异常

大多数具有`try/catch`功能的编程语言中有一个常见特性，即能够从`catch`块中`rethrow`异常。这意味着最初在`try`块中发生的异常将被再次引发，就像它根本没有被处理一样。当你需要对异常进行一些处理，但同时也要让调用者知道例程中出现了问题时，这很有用。

T-SQL 不包含任何内置的`rethrow`功能。然而，基于`CATCH`块错误函数，结合`RAISERROR`，很容易创建这种行为。以下示例展示了 T-SQL 中`rethrow`的基本实现：

```sql
BEGIN TRY
    SELECT CONVERT(int, 'ABC') AS ConvertException;
END TRY
BEGIN CATCH
    DECLARE
        @ERROR_SEVERITY int = ERROR_SEVERITY(),
        @ERROR_STATE int = ERROR_STATE(),
        @ERROR_NUMBER int = ERROR_NUMBER(),
        @ERROR_LINE int = ERROR_LINE(),
        @ERROR_MESSAGE varchar(245) = ERROR_MESSAGE();
    RAISERROR('Msg %d, Line %d: %s',
              @ERROR_SEVERITY,
              @ERROR_STATE,
              @ERROR_NUMBER,
              @ERROR_LINE,
              @ERROR_MESSAGE);
END CATCH
GO
```

由于`RAISERROR`不能用于抛出低于 13000 的异常，在这种情况下“重新抛出”异常需要引发一个用户定义的异常，并将数据以特殊格式的错误消息发回。由于函数不允许在`RAISERROR`的调用中使用，因此必须在调用`RAISERROR`重新抛出异常之前定义变量并赋值错误函数的值。以下是此 T-SQL 的输出消息：

```
(0 row(s) affected)
Msg 50000, Level 16, State 1, Line 19
Msg 245, Line 2: Conversion failed when converting the varchar value 'ABC'
to data type int.
```

请记住，根据你的接口需求，你可能并不总是希望重新抛出最初捕获的相同异常。在许多情况下，捕获初始异常，然后抛出一个对调用者更相关（或更有帮助）的新异常可能更有意义。

例如，如果你正在使用链接服务器，并且该服务器由于某种原因没有响应，你的代码将抛出超时异常。传回一个通用的“数据不可用”异常可能比向调用者暴露问题的实际原因更有意义。这应该根据具体情况决定，以便你为存储过程接口设计出最佳方案。

## 何时应该使用`TRY/CATCH`？

如前所述，在 T-SQL 例程（例如存储过程中）处理异常的一般用例是，尽可能在较低级别进行封装，以简化应用程序的整体代码。这方面的一个主要例子是记录数据库异常。与其将无法正确处理的异常发送回应用层（在那里它将被记录回数据库），不如在数据库例程的范围内记录它，这样可能更有意义。

另一个用例涉及对源于应用程序代码的问题进行临时修复。例如，应用程序可能由于一个错误，偶尔向存储过程传递无效的键，而该存储过程本应将它们插入表中。通过在数据库中简单地捕获异常，而不是将其抛回给应用程序导致用户收到错误消息，来*临时*“修复”问题，可能更简单。将此类快速修复到位通常比重建和重新部署整个应用程序要便宜得多。

考虑何时*不*封装异常也很重要。确保不要过度处理安全问题、严重的数据错误以及其他应用程序——最终是用户——可能应该被告知的异常。绝对存在“过度处理异常”这种事，陷入这个陷阱可能意味着问题会被隐藏，直到它们引起足够的骚动而变得无法忽视。

隐藏在异常处理器背后的长期问题通常会以无法修复的数据损坏形式暴露出来。这些情况通常因缺乏可行的备份而凸显，因为问题已经存在了很长时间，并不可避免地以业务损失和开发人员更新简历寻找工作而告终。幸运的是，避免这个问题相当容易。只需运用一点常识，不要为了压制所有异常而走向极端。

## 使用`TRY/CATCH`构建重试逻辑


