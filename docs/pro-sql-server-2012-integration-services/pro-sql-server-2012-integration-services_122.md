# 构建健壮的解决方案

```vb
Dim rdr As New TextFieldParser("C:\Robust\PeopleErrors.txt")

rdr.TextFieldType = FileIO.FieldType.Delimited

rdr.SetDelimiters(",")

Dim row As String()

While Not rdr.EndOfData

    Try

        row = rdr.ReadFields()

        MessageBox.Show(row(0).ToString)

    Catch ex As Microsoft.VisualBasic.FileIO.MalformedLineException

        MsgBox("Line " & ex.Message & _

        "is not valid and will be skipped.")

    End Try

End While

Dts.TaskResult = ScriptResults.Success

End Sub
```

`MessageBox` 返回行字符串数组中错误编号的索引值，可用于识别具体原因。显然，要动态处理错误需要额外代码，但有了错误代码和描述（在后续的 Tip 中提供），可以包含一个 `VB.NET Select Case` 语句来遍历错误代码，并根据错误采取适当的操作。

### 事件处理程序

事件处理程序提供了一种主动处理包和可执行文件上事件的方法，可用于为以下事件捕获任务特定信息：

- `OnError`
- `OnExecStatusChange`
- `OnExecution`
- `OnPostExecute`
- `OnPostValidate`
- `OnPreExecute`
- `OnPreValidate`
- `OnQueryCancel`
- `OnTaskFailed`
- `OnVariableValueChanged`
- `OnWarning`

[www.it-ebooks.info](http://www.it-ebooks.info/)

`OnError` 事件处理程序可用于捕获错误描述和代码，然后这些信息可用于动态处理错误以及记录有关特定问题的更深入信息。

由于事件处理程序仅在可执行文件和包上可用，这排除了在数据流内部使用它们的可能性，但可以捕获数据流可执行文件的失败。为了使 `OnError` 事件处理程序能够被触发，数据流任务必须首先失败，这将清空缓冲区中的所有数据，使其无法用于后续调查。

在事件处理程序内处理错误信息，最简单的方法是通过脚本任务传入 `System:ErrorCode` 和 `System:ErrorDescription` 作为只读变量。第 11 章概述了如何创建脚本任务来捕获错误信息，并演示了 `OnError` 事件从任务级别一直向上冒泡到包级别。显然，需要在脚本组件中纳入更复杂的逻辑类型，以主动处理特定错误。

上一节概述的包在数据流任务中使用了错误重定向，以防止错误出现并确保所有行都被定向到目标。如果不重定向行而创建相同的包，则可以使用 `OnError` 事件处理程序来捕获错误信息。在 `OnError` 事件中添加一个脚本任务，并将 `System:ErrorDescripton` 设置为 `ReadOnly` 变量，将允许捕获整个错误消息，并更深入地了解真正原因。使用以下 `VB.NET` 代码将显示数据流任务中发生的任何错误的消息：

```vb
MessageBox.Show(Dts.Variables(0).Value.ToString)
```

生成的消息框指定了截断错误以及受影响的具体列。错误重定向和 `OnError` 事件处理程序之间的主要区别在于，如果使用了错误重定向，则不会引发 `OnError` 事件处理程序。在前面的示例中，数据流任务在截断错误发生后停止，并且失败的行信息将不会被存储以供后续评估。

`TIP:` SSIS 错误消息和编号概述在 http://msdn.microsoft.com/en-us/library/ms345164.aspx。

### 动态性

从 SSIS 2005 开始，配置使包更具可移植性，并可以轻松地将包导入到 SQL 的多个实例中，而无需创建多个包或专门为环境配置包。在 SSIS 2012 中，随着参数的引入，这一功能得到了进一步优化。


环境允许将参数映射到其变量，并可用于在多种环境中执行单个包。通常需要在一个环境中开发包，在另一个环境中测试，最后部署到生产环境。这类场景体现了在 SSIS 中使用环境的真正灵活性和强大功能。

参数和环境在第 19 章概述，并提供了动态包执行的示例。

## 第 18 章 – 构建健壮的解决方案

### 可追溯性

在包部署并运行一段时间后，你经常需要查看每个任务“在实际运行中”的表现。我经常收到请求，要求查看通过单个数据流传递的总行数或平均行数，或者单个任务耗时多久。虽然对于在 SQL Server 代理中计划的包，其运行时间很容易确定，但要分离任务时间和计数却不那么简单。

#### 日志提供程序

第 11 章概述了如何配置日志提供程序以及将包执行信息保存到特定提供程序中，如 XML、SQL、CSV 等。日志提供程序也可用于提供通用的任务指标，例如执行时间。日志使你能够捕获事件处理程序引发的信息。例如，选择 `OnPreExecute` 和 `OnPostExecute` 事件提供了一种跟踪包内任务大致开始和结束时间的方法。

日志提供程序的配置在控制流窗格中完成，方法是右键单击并选择 `日志提供程序`。初始窗口需要定义要记录的可执行文件以及日志的写入格式，例如 XML、SQL 或任何支持的格式。配置好可执行文件和目标后，可以选择事件处理程序以及要收集的特定列。图 18-12 说明了通过日志提供程序可用的事件处理程序。

*图 18-12. 日志提供程序事件*

对选定事件使用 SQL 日志提供程序，可以查询数据以找到特定的任务持续时间。每个包都有自己的 `GUID`，每个任务也是如此，因此使用带有 `DATEDIFF` 函数的公用表表达式可以快速返回相关任务的执行时间。以下查询使用通过日志传送配置创建的表，查找由其 `GUID` 标识的特定包的开始和结束时间之间的差值。结果如图 18-13 所示。

```sql
WITH eventexecution
AS(
    SELECT sourceid,
           starttime
    FROM sysssislog
    WHERE executionid = 'B0885A09-A1BA-4638-9999-29698F357B40'
      AND event = 'OnPreExecute'
)
SELECT source,
       DATEDIFF(SECOND, c.starttime, endtime) AS 'seconds'
FROM sysssislog s JOIN eventexecution c
    ON s.sourceid = c.sourceid
WHERE event = 'OnPostExecute';
GO
```

*图 18-13. 任务持续时间查询结果*

#### 自定义日志记录

尽管日志提供程序可以捕获大量有关包执行的信息，但有些指标会被遗漏，例如数据流行数。另一个常见的请求是显示 `数据流` 任务随时间影响的总行数。此类计数未包含在 SSIS 日志提供程序中，但仍然可以使用自定义日志记录来捕获。

第一步是创建一个表来保存自定义日志信息，在本例中将包括包 `GUID`、包名称、任务名称、目标名称、行数以及任务运行的日期。例如：

```sql
IF NOT EXISTS(SELECT * FROM sys.tables WHERE name = 'SSISCounts')
BEGIN
    CREATE TABLE SSISCounts(
        ExecGUID NVARCHAR(50),
        PackageName NVARCHAR(50),
        TaskName NVARCHAR(50),
        DestinationName NVARCHAR(50),
        NumberRows INT,
        DateImported DATETIME2)
END;
GO
```

需要创建一个包级变量，该变量将保存所有 `数据流` 任务的行计数，数据类型为整数。每个 `数据流` 任务都应包含一个 `行计数` 转换，紧接在目标之前，该转换使用包级变量来记录导入的行数。

*图 18-14. 行计数转换*

最后，在 `数据流` 任务的 `PostExecute` 事件处理程序中，创建一个 `执行 SQL` 任务，该任务将使用系统变量和用户定义变量插入适当的信息。单击事件处理程序将打开 `事件处理程序` 窗格，并提供两个下拉列表：`可执行文件` 和 `事件处理程序`，如图 18-15 所示。选择可执行文件后，可以选择所需的事件处理程序，并将特定任务放置在窗格上。

*图 18-15. 执行 SQL 任务参数配置*

将使用 `执行 SQL` 任务，通过系统提供和用户提供的信息，从 `数据流` 任务插入计数信息。系统提供的信息是包 `GUID`、名称、任务和行计数，数据库和时间将在查询中提供。`SQLStatement` 属性应设置为使用将在任务中定义的参数，方法是使用问号 (`?`)，如下列代码所示：

```sql
INSERT SSISCounts
VALUES(?, ?, ?, 'AdventureWorks2012DW', ?, SYSDATETIME())
```

参数应按照它们在查询中出现的顺序，在 `执行 SQL 任务编辑器` 中以相同的序数位置进行组织。图 18-16 描绘了参数配置。请注意，参数名称被替换为变量在插入语句中出现的索引位置。

*图 18-16. 执行 SQL 任务参数配置*

在将参数映射到 SQL 语句属性中的 `?` 时通常存在困惑。根据上图，请记住，定义的第一个参数（在本例中为 `ExecutionInstanceGUID`）将出现在第一个问号出现的位置，每个参数将按顺序出现。为了便于演示，插入语句可以重写，以便更好地可视化这一点，如下列代码所示：

```sql
INSERT SSISCounts
VALUES(ExecutionInstanceGUID, PackageName, TaskName,
       'AdventureWorks2012DW', RowCount, SYSDATETIME())
```

前面的示例演示了，通过事件处理程序配合使用系统变量和用户定义变量，可以提供一种自定义日志记录的方法。`SSISCounts` 表中的信息可用于建立跟踪数据流添加的记录数以及跟踪特定任务的方法。

### 小结

本章介绍了确保 SSIS 包和任务的持久性、可移植性和弹性的方法。

本章讨论的许多方法都是本书通篇概述的功能的扩展。创建健壮解决方案的关键概念是：

-   为未知情况做计划。
-   主动处理可能发生的错误。
-   为数据流提供错误重定向。
-   配置包和任务以使其可移植。
-   记录错误和度量包及任务信息。

## 第 19 章 – 部署模型

*程序测试可用于显示错误的存在，但永远无法显示其不存在！*

—计算机科学家 艾兹格·迪科斯彻

SQL Server Integration Services 2012 版引入了一种全新的部署 ETL 过程的方式。



