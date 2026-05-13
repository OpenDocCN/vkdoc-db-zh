# 第四章：错误与异常

第二个参数 `severity` 可用于对异常的行为实施某种程度的控制，类似于 SQL Server 使用错误级别的方式。在大多数情况下，适用相同的异常范围：级别在 1 到 10 之间的异常产生警告，级别在 11 到 18 之间的异常被视为普通用户错误，而级别高于 18 的异常被视为严重异常，只能由 `sysadmin` 固定服务器角色的成员引发。用户引发的超过级别 20 的异常，就像 SQL Server 引发的异常一样，会导致连接中断。除了这些范围之外，用户引发的异常没有真正的控制权，所有异常都被视为语句级别——即使设置了 `XACT_ABORT` 也是如此。

**注意** `XACT_ABORT` 不会影响 `RAISERROR` 语句的行为。

`state` 参数可以是 1 到 127 之间的任何值，对异常的行为没有影响。它可用于向异常添加额外的编码信息——但在大多数情况下，将这些数据添加到错误消息本身可能同样简单。

使用 `RAISERROR` 的最简单方法是传入包含错误消息的字符串，并设置适当的错误级别和状态。对于一般异常，我通常使用严重级别 16 和状态值 1：

```sql
RAISERROR('General exception', 16, 1);
```

这会产生以下输出：

```
Msg 50000, Level 16, State 1, Line 1
General exception
```

请注意，此情况下生成的错误号是 50000，这是当向 `RAISERROR` 的第一个参数传入字符串时将使用的通用用户定义错误号。

**注意** 先前版本的 SQL Server 允许使用如下语法指定错误号和消息号：`RAISERROR 50000 'General exception'`。此语法在 SQL Server 2008 中已弃用，不应再使用。

### 格式化错误消息

定义错误消息时，通常以某种方式格式化文本是有用的。例如，思考一下你可能会如何编写代码，在循环中处理动态检索到的多个产品 ID。

你可能有一个名为 `@ProductId` 的局部变量，其中包含代码当前正在处理的产品 ID。如果是这样，你可能希望定义一个在发生问题时应引发的自定义异常——并且很可能最好随错误消息一起返回 `@ProductId` 的当前值。

在这种情况下，有几种方法可以将数据随异常一起发回。第一种是动态构建错误消息字符串：

```sql
DECLARE @ProductId int;
SET @ProductId = 100;
/* ... 问题出现 ... */
DECLARE @ErrorMessage varchar(200);
SET @ErrorMessage =
    'Problem with ProductId ' + CONVERT(varchar, @ProductId);
RAISERROR(@ErrorMessage, 16, 1);
```

执行此批处理会产生以下输出：

```
Msg 50000, Level 16, State 1, Line 10
Problem with ProductId 100
```

虽然这在此情况下有效，但动态构建错误消息并不是最优雅的开发实践。更好的方法是使用格式设计符并将 `@ProductId` 作为可选参数传递，如下面的代码清单所示：

```sql
DECLARE @ProductId int;
SET @ProductId = 100;
/* ... 问题出现 ... */
RAISERROR('Problem with ProductId %i', 16, 1, @ProductId);
```

执行此批处理会产生与之前相同的输出，但所需的代码量大大减少，并且你不必担心定义额外变量或构建繁琐的转换代码。嵌入在错误消息中的 `%i` 是一个格式设计符，表示“整数”。另一个最常用的格式设计符是 `%s`，表示“字符串”。

你可以在错误消息中嵌入任意数量的设计符，它们将按照附加可选参数的顺序进行替换。例如：

```sql
DECLARE @ProductId1 int;
SET @ProductId1 = 100;
DECLARE @ProductId2 int;
SET @ProductId2 = 200;
DECLARE @ProductId3 int;
SET @ProductId3 = 300;
/* ... 问题出现 ... */
RAISERROR('Problem with ProductIds %i, %i, %i',
    16, 1, @ProductId1, @ProductId2, @ProductId3);
```

这会产生以下输出：

```
Msg 50000, Level 16, State 1, Line 12
Problem with ProductIds 100, 200, 300
```

**注意** 熟悉 C 编程的读者会注意到，`RAISERROR` 使用的格式设计符与 C 语言的 `printf` 函数使用的相同。有关支持的设计符的完整列表，请参阅 SQL Server 2008 联机丛书中的“[RAISERROR (Transact-SQL)](https://docs.microsoft.com/en-us/sql/t-sql/language-elements/raiserror-transact-sql)”主题。

## 创建持久的自定义错误消息

使用格式设计符格式化消息而不是动态构建字符串是朝着正确方向迈出的一步，但它没有解决最后一个关键问题：如果你需要在多个地方使用相同的错误消息怎么办？你可以在需要该异常的每个例程中简单地使用完全相同的参数调用 `RAISERROR`，但如果你需要更改错误消息，这可能会导致维护难题。此外，每个异常都将只能使用默认的用户定义错误号 50000，这使得针对这些自定义异常的编程变得更加困难。

幸运的是，SQL Server 很好地解决了这些问题，它提供了一种机制，可以将自定义错误消息添加到 `sys.messages`。然后，可以使用 `RAISERROR` 并传入自定义错误号作为第一个参数来引发使用这些错误消息的异常。

要创建持久的自定义错误消息，请使用存储过程 `sp_addmessage`。此存储过程允许用户为大于 50000 的消息编号指定自定义消息。除了错误消息外，用户还可以指定默认严重级别。使用 `sp_addmessage` 添加的消息的作用域是服务器级别，因此如果你在同一服务器上托管多个应用程序，请注意它们是否定义了自定义消息以及是否存在任何重叠——你可能需要为一个或多个应用程序设置新的 SQL Server 实例，以允许它们创建自己的异常。在开发使用自定义消息的新应用程序时，请尝试选择一个定义明确的范围来创建你的消息，以避免在共享环境中与其他应用程序重叠。

请记住，你可以使用 50000 到 2147483647 之间的任何数字，不必局限于 50000 范围。

添加自定义消息就像调用 `sp_addmessage` 并定义消息编号和消息文本一样简单。以下 T-SQL 将上一节中的消息定义为错误消息编号 50005：

```sql
EXEC sp_addmessage
    @msgnum = 50005,
    @severity = 16,
    @msgtext = 'Problem with ProductIds %i, %i, %i';
GO
```

一旦执行此 T-SQL，就可以通过使用适当的错误号调用 `RAISERROR` 来引发使用此错误消息的异常：

```sql
RAISERROR(50005, 15, 1, 100, 200, 300);
```

这会导致以下输出被发送回客户端：

```
Msg 50005, Level 15, State 1, Line 1
Problem with ProductIds 100, 200, 300
```

请注意，在这种情况下调用 `RAISERROR` 时指定了严重级别 15，尽管自定义错误最初定义为严重级别 16。这引出了关于自定义错误严重级别的一点重要说明：在调用 `RAISERROR` 时指定的任何严重级别都将覆盖为该错误定义的严重级别。但是，如果你向 `RAISERROR` 的该参数传递一个负值，则将使用默认严重级别：

```sql
RAISERROR(50005, -1, 1, 100, 200, 300);
```

这会产生以下输出（请注意，`Level` 现在是 16，与创建错误消息时定义的一致）：

```
Msg 50005, Level 16, State 1, Line 1
Problem with ProductIds 100, 200, 300
```

建议，除非有特定原因需要覆盖严重级别，否则在引发自定义异常时，始终对严重性参数使用 -1。



