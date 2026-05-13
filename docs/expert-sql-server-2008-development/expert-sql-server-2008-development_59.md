# 第 4 章：错误与异常

在 SQL Server 内部。这意味着，如果托管代码抛出异常，它会被包装器捕获，然后生成一个错误。该错误消息包含原始异常的详细信息，以及其发生时的堆栈跟踪。

在这种情况下，原始 CLR 异常 `System.DivideByZeroException` 引发了一个 `6522` 错误，这是任何发生在 SQLCLR 函数内部的未处理异常的通用错误消息。

如前所述，处理此类异常的最佳方法是尽可能在最低层级解决。对于像本例中的用户定义函数（UDF）这样的情况，异常应该在 CLR 代码本身内部处理（例如使用 `try…catch`），这样就无需在 T-SQL 层级捕获它。

这引发了一个有趣的点：如何处理系统定义的 CLR 例程中产生的异常，例如由 `geometry`、`geography` 或 `hierarchyid` 类型定义的任何方法。考虑以下示例，该示例尝试使用无效值实例化 `hierarchyid` 和 `geography` 数据类型的变量：

```sql
DECLARE @HierarchyId hierarchyid = '/1/1';

DECLARE @Geography geography = 'POLYGON((0 51, 0 52, 1 52, 1 51 ,0 51))';
GO
```

这两条语句都将导致 CLR 异常，报告如下：

```text
Msg 6522, Level 16, State 2, Line 1
A .NET Framework error occurred during execution of user-defined routine or aggregate "hierarchyid":
Microsoft.SqlServer.Types.HierarchyIdException: 24001: SqlHierarchyId.Parse failed because the input string '/1/1' is not a valid string representation of a SqlHierarchyId node.
Microsoft.SqlServer.Types.HierarchyIdException:
   at Microsoft.SqlServer.Types.SqlHierarchyId.Parse(SqlString input).

Msg 6522, Level 16, State 1, Line 2
A .NET Framework error occurred during execution of user-defined routine or aggregate "geography":
Microsoft.SqlServer.Types.GLArgumentException: 24205: The specified input does not represent a valid geography instance because it exceeds a single hemisphere. Each geography instance must fit inside a single hemisphere. A common reason for this error is that a polygon has the wrong ring orientation.
Microsoft.SqlServer.Types.GLArgumentException:
   at Microsoft.SqlServer.Types.GLNativeMethods.ThrowExceptionForHr(GL_HResult errorCode)
   at Microsoft.SqlServer.Types.GLNativeMethods.GeodeticIsValid(GeoData g)
   at Microsoft.SqlServer.Types.SqlGeography.IsValidExpensive()
   at Microsoft.SqlServer.Types.SqlGeography.ConstructGeographyFromUserInput(GeoData g, Int32 srid)
   at Microsoft.SqlServer.Types.SqlGeography.GeographyFromText(OpenGisType type, SqlChars taggedText, Int32 srid)
   at Microsoft.SqlServer.Types.SqlGeography.STGeomFromText(SqlChars geometryTaggedText, Int32 srid)
   at Microsoft.SqlServer.Types.SqlGeography.Parse(SqlString s)
```

`注意` 如前面的示例代码所示，托管代码生成的异常是语句级异常——即使第一个语句已经产生了 `6522` 错误，第二个语句仍被允许运行。

我们如何创建特定的代码路径来处理此类异常？尽管它们涉及截然不同的情况，但由于两个异常都发生在托管代码内部，每种情况下生成的 T-SQL 错误是相同的——通用错误 `6522`。这意味着我们无法使用 `ERROR_NUMBER()` 来区分这些情况。此外，我们无法轻易地在原始函数代码中添加自定义错误处理，因为这些都是在预编译的 `Microsoft.SqlServer.Types.dll` 程序集中定义的系统方法。

一种方法是定义新的自定义 CLR 方法，包装 `SqlServer.Types.dll` 中的每个系统定义方法，这些方法在将结果传回 SQL Server 之前检查并处理任何 CLR 异常。以下代码清单展示了一个包装在 `geography` `Parse()` 方法外的此类包装器示例：

```csharp
[Microsoft.SqlServer.Server.SqlFunction()]
public static SqlGeography GeogTryParse(SqlString Input)
{
    SqlGeography result = new SqlGeography();
    try
    {
        result = SqlGeography.Parse(Input);
    }
    catch
    {
        // Exception Handling code here
        // Optionally, rethrow the exception
        // throw new Exception("An exception occurred that couldn't be handled");
    }
    return result;
}
```

或者，您可以创建依赖于解析 `ERROR_MESSAGE()` 内容的代码路径，以识别堆栈跟踪中指定的原始 CLR 异常的详细信息。系统定义的 CLR 类型生成的异常具有 `24000` 到 `24999` 范围内的五位数异常编号，因此可以使用 T-SQL `PATINDEX` 函数从 `ERROR_MESSSAGE()` 字符串中提取出来。以下代码清单展示了在应用于前面给出的 `hierarchyid` 示例时如何实现这种方法：

```sql
DECLARE @errorMsg nvarchar(max);

BEGIN TRY
    SELECT hierarchyid::Parse('/1/1');
END TRY
BEGIN CATCH
    SELECT @errorMsg = ERROR_MESSAGE();
    SELECT SUBSTRING(@errorMsg, PATINDEX('%: 24[0-9][0-9][0-9]%', @errorMsg) + 2, 5);
END CATCH
GO
```

得到的值 `24001` 对应于发生的具体 CLR 异常（“输入字符串不是 SqlHierarchyId 节点的有效字符串表示形式”），而不是通用的 T-SQL 错误 `6522`，并且可用于编写特定的代码路径来处理此类异常。

## 事务与异常

讨论 SQL Server 中的异常时，如果不提及事务与异常之间的相互作用，讨论就不完整。这是一个相当简单的领域，但常常让那些不太理解事务作用的开发者感到困惑。

SQL Server 是一个数据库管理系统（`DBMS`），因此其主要目标之一是对数据进行管理和操作。因此，SQL Server 中每个异常处理方案的核心都必须考虑到：这些不仅仅是异常——它们也是数据问题。

### 关于事务中止的误解

一些开发者犯的最大错误是假设如果在事务期间发生异常，该事务将被中止。默认情况下，这种情况几乎*从不*发生。即使在面对异常时，大多数事务也会继续存在，运行以下 T-SQL 即可证明：

```sql
BEGIN TRANSACTION;
GO
SELECT 1/0 AS DivideByZero;
GO
SELECT @@TRANCOUNT AS ActiveTransactionCount;
GO
```

此 T-SQL 的输出如下：

```text
DivideByZero
Msg 8134, Level 16, State 1, Line 1
Divide by zero error encountered.

ActiveTransactionCount
(1 row(s) affected)
```

另一个错误是认为存储过程代表某种形式的工作原子单元，拥有自己的隐式事务，在发生异常时会回滚。唉，这也不正确，如下的 T-SQL 所证明：

```sql
--Create a table for some data
CREATE TABLE SomeData
(
    SomeColumn int
);
GO

--This procedure will insert one row, then throw a divide-by-zero exception
CREATE PROCEDURE NoRollback
AS
BEGIN
    INSERT INTO SomeData VALUES (1);
    INSERT INTO SomeData VALUES (1/0);
END;
GO

--Execute the procedure
EXEC NoRollback;
GO

--Select the rows from the table
SELECT *
FROM SomeData;
GO
```

结果是，即使有错误，没有引发异常的那一行仍然在表中；存储过程没有产生隐式事务：

```text
SomeColumn
-----------
1
```

即使在插入之前在存储过程中开始一个*显式*事务，并在异常发生后提交，此示例仍将返回相同的输出。默认情况下，除非显式发出回滚，否则在大多数情况下，异常不会回滚任何内容。它只是作为一个消息，表明出了问题。

### XACT_ABORT：将误解变为（半）现实


