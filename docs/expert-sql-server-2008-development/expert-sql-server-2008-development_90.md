# 第 7 章：SQLCLR：架构与设计考量

不幸的是，满足此查询所需的过程效率并不高。假设表的 `x` 列上存在索引，上述查询会生成如图 7-2 所示的执行计划。

**图 7-2. 在 T-SQL 中用于创建累计和的嵌套索引查找**

要累加列中所有先前的值，需要一个包含索引查找的嵌套循环。随着处理的行数增加，此查找返回的行数呈指数级增长。在处理第一行时，此查找只需累加一个值，但要计算 100 行数据的累计和，则需要总共读取 5,050 行。对于 200 行的数据集，查询处理器需要处理 20,100 行——这是满足前一个查询所需工作量的四倍。因此，随着表中行数的增加，这种计算累计聚合的方法性能会迅速下降。

一个能带来显著性能提升的替代方案是使用游标。在 SQL Server 开发界普遍存在一种看法，认为游标是糟糕的东西，但它们确实有其合理的用例，而此场景可能就是其中之一。然而，有许多充分的理由让许多开发人员不愿意使用游标，当然，我也并非在一般情况下提倡使用它们。

更好的方法是使用 SQLCLR 循环遍历并使用局部变量存储累计值，然后通过 `SqlPipe` 逐行流式传输结果。以下代码清单给出了这样一个解决方案的示例：

```csharp
[Microsoft.SqlServer.Server.SqlProcedure]
public static void RunningSum()
{
    using (SqlConnection conn = new SqlConnection("context connection=true;"))
    {
        SqlCommand comm = new SqlCommand();
        comm.Connection = conn;
        comm.CommandText = "SELECT x FROM T ORDER BY x";
        
        SqlMetaData[] columns = new SqlMetaData[2];
        columns[0] = new SqlMetaData("Value", SqlDbType.Int);
        columns[1] = new SqlMetaData("RunningSum", SqlDbType.Int);
        
        int RunningSum = 0;
        SqlDataRecord record = new SqlDataRecord(columns);
        
        SqlContext.Pipe.SendResultsStart(record);
        
        conn.Open();
        SqlDataReader reader = comm.ExecuteReader();
        
        while (reader.Read())
        {
            int Value = (int)reader[0];
            RunningSum += Value;
            
            record.SetInt32(0, (int)reader[0]);
            record.SetInt32(1, RunningSum);
            
            SqlContext.Pipe.SendResultsRow(record);
        }
        
        SqlContext.Pipe.SendResultsEnd();
    }
}
```

我曾在多个场合使用此解决方案，发现它非常高效且易于维护，并且避免了像其他替代方案那样需要使用任何 `temp` 表来存储累计和。在针对包含 100,000 行的表进行测试时，SQLCLR 查询的平均执行时间为 2.7 秒，而等效的 T-SQL 查询则需要超过 5 *分钟*。

### 字符串操作

为了比较 T-SQL 和 SQLCLR 之间字符串处理函数的性能，我想设计一个公平、实用的测试。问题在于，处理字符串数据有大量巧妙的技术：在 T-SQL 中，一些性能最佳的方法使用一个或多个公共表表达式 (`CTEs`)、`CROSS APPLY` 运算符或数字表；或者将文本字符串转换为 `XML` 以执行复杂的字符数据操作。同样，在 SQLCLR 中，可用的技术差异很大，具体取决于您使用的是原生的 `String` 方法还是 `StringBuilder` 类提供的方法。

我决定，与其尝试定义一个需要特定字符串处理技术的场景，唯一公平的测试是直接比较两种环境中提供等效功能的两个内置方法。我决定选用 T-SQL 的 `CHARINDEX` 和 .NET 的 `String.IndexOf()`，它们都在一个字符串中搜索并返回另一个字符串的位置。为了测试的目的，我创建了不同长度的 `nvarchar(max)` 字符串，每个字符串完全由重复的字符 `a` 组成。然后，我在每个字符串的末尾附加一个字符 `x`，并计时测量相应方法在 10,000 次迭代中找到该字符位置的性能。

**图 7-3. 比较 CHARINDEX 与 String.IndexOf() 的性能**

以下代码清单展示了 T-SQL 方法：

```sql
CREATE PROCEDURE SearchCharTSQL
(
    @needle nchar(1),
    @haystack nvarchar(max)
)
AS BEGIN
    PRINT CHARINDEX(@needle, @haystack);
END;
```

而这是 CLR 等效方法：

```csharp
[SqlProcedure]
public static void SearchCharCLR(SqlString needle, SqlString haystack)
{
    SqlContext.Pipe.Send(
        haystack.ToString().IndexOf(needle.ToString()).ToString()
    );
}
```

请注意，`CHARINDEX` 的起始位置是基于 1 的，而 `IndexOf()` 使用的索引编号是基于 0 的。因此，每个方法的结果会不同，但它们获得该结果所完成的工作量是相同的。我对每个存储过程进行了如下测试，通过为 `REPLICATE` 方法替换不同的参数值来更改要搜索的字符串长度：

```sql
DECLARE @needle nvarchar(1) = 'x';
DECLARE @haystack nvarchar(max);

SELECT @haystack = REPLICATE(CAST('a' AS varchar(max)), 8000) + 'x';

EXEC dbo.SearchCharTSQL @needle, @haystack;
```

与前面给出的质数筛示例类似，字符串搜索、匹配和替换所需的逻辑最适合由 .NET 基础类库提供的高效例程来处理。如果您当前的代码逻辑严重依赖于 T-SQL 字符串功能，包括 `CHARINDEX`、`PATINDEX` 或 `REPLACE`，我强烈建议您研究一下通过 SQLCLR 可用的替代选项——您可能会对所获得的性能提升感到惊讶。

## 使用 SQLCLR 增强 Service Broker 横向扩展能力

在讨论了使用 SQLCLR 背后的一些理论并给出了一些孤立的性能比较之后，现在让我们将注意力转向一个更详细的、将这些想法付诸实践的例子。Service Broker 经常被提及为帮助横向扩展数据库服务的绝佳选择。其中一个更引人注目的用例是 Service Broker 服务，可用于异步地从远程系统请求数据。在这种情况下，请求消息将从本地存储过程发送到远程数据服务，该存储过程在等待响应（即请求的数据）返回的同时可以执行其他工作。

构建这样一个系统的方法有很多，并且考虑到 Service Broker 允许以二进制或 `XML` 格式发送消息，我想知道哪种方式能提供最佳的整体性能以及从代码复用角度看最有价值。在接下来的章节中，我将引导您了解我对使用 SQLCLR 进行 `XML` 和二进制序列化的研究。

### XML 序列化

我开始使用 `AdventureWorks2008` 数据库中的 `HumanResources.Employee` 表作为示例数据集，想象一个远程数据服务请求员工列表及其属性。经过一些实验，我确定 `FOR XML RAW` 选项是将表序列化为 `XML` 格式的最简单方法，并且我使用 `ROOT` 选项使 `XML` 有效：

```sql
DECLARE @x xml;

SET @x = (
    SELECT *
    FROM HumanResources.Employee
    FOR XML RAW, ROOT('Employees')
);
GO
```

众所周知，`XML` 是一种极其冗长的数据交换格式，而我并未




