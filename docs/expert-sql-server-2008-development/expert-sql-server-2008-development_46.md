# 第三章 测试数据库例程

断言允许开发者自我记录代码例程所做的假设。如果一个例程期望某个变量在特定时间处于特定状态，则可以使用断言来帮助确保随着代码的成熟，该假设得以强制执行。如果在未来的任何时间，代码的更改使该假设无效，则进行更改的开发人员在测试或调试时触发该断言将会引发异常。

在单元测试中，断言有着非常相似的目的。它们允许测试人员控制使单元测试返回真或假的条件。如果在单元测试中有任何断言引发异常，则整个测试被视为失败。

几乎每种语言和平台都有单元测试框架，包括 T-SQL（例如，可从 http://sourceforge.net/projects/tsqlunit 获取的 TSQLUnit 项目）。就个人而言，我发现与其他语言相比，在 T-SQL 中进行单元测试很麻烦，因此更喜欢使用 .NET 单元测试框架 NUnit（http://www.nunit.org）在 .NET 语言中编写测试。

提供关于如何针对单元测试框架编码的深入指南超出了本书的范围，但鉴于许多开发人员对存储过程进行单元测试仍有些陌生，我将提供一套基本的遵循规则。编写存储过程单元测试时，可以遵循以下基本步骤：

1.  首先，确定应该对存储过程的接口做出哪些假设。将返回哪些结果集？列的数据类型是什么，会有多少列？契约是否保证了特定的行数？
2.  接下来，编写执行待测存储过程所需的代码。如果你使用 NUnit，我发现暴露相关输出的最简单方法是使用 ADO.NET 用存储过程的结果填充一个 `DataSet`，以便随后进行检查。在此阶段要小心；你想要测试的是存储过程，而不是你的数据访问框架。你可能会忍不住使用与应用程序本身相同的方法来调用存储过程。然而，这将是一个错误，因为你最终将同时测试存储过程和该方法。鉴于你只需要填充一个 `DataSet`，在单元测试中重新编码数据访问不应是主要负担，并且可以防止你测试到你本不打算测试的代码部分。
3.  最后，对你关于存储过程所做的每个假设使用一个断言；这意味着每个列名一个断言，每个列数据类型一个断言，必要时行数一个断言，等等。宁可多用断言——如果后来发现某个假设不正确，再将其移除，总比一开始就没有该假设，而在接口实际上工作不正确时单元测试却通过了要好。

以下代码清单展示了 NUnit 测试 `GetAggregateTransactionHistory` 存储过程可能的样子：

```csharp
[TestMethod]
public void TestAggregateTransactionHistory()
{
    // Set up a command object
    SqlCommand comm = new SqlCommand();

    // Set up the connection
    comm.Connection = new SqlConnection(
        @"server=serverName; trusted_connection=true;");

    // Define the procedure call
    comm.CommandText = "GetAggregateTransactionHistory";
    comm.CommandType = CommandType.StoredProcedure;
    comm.Parameters.AddWithValue("@CustomerId", 123);

    // Create a DataSet for the results
    DataSet ds = new DataSet();

    // Define a DataAdapter to fill a DataSet
    SqlDataAdapter adapter = new SqlDataAdapter();
    adapter.SelectCommand = comm;

    try
    {
        // Fill the dataset
        adapter.Fill(ds);
    }
    catch
    {
        Assert.Fail("Exception occurred!");
    }

    // Now we have the results -- validate them...

    // There must be exactly one returned result set
    Assert.IsTrue(
        ds.Tables.Count == 1,
        "Result set count != 1");
    DataTable dt = ds.Tables[0];

    // There must be exactly two columns returned
    Assert.IsTrue(
        dt.Columns.Count == 2,
        "Column count != 2");

    // There must be columns called TotalDeposits and TotalWithdrawals
    Assert.IsTrue(
        dt.Columns.IndexOf("TotalDeposits") > -1,
        "Column TotalDeposits does not exist");
    Assert.IsTrue(
        dt.Columns.IndexOf("TotalWithdrawals") > -1,
        "Column TotalWithdrawals does not exist");

    // Both columns must be decimal
    Assert.IsTrue(
        dt.Columns["TotalDeposits"].DataType == typeof(decimal),
        "TotalDeposits data type is incorrect");
    Assert.IsTrue(
        dt.Columns["TotalWithdrawals"].DataType == typeof(decimal),
        "TotalWithdrawals data type is incorrect");

    // There must be zero or one rows returned
    Assert.IsTrue(
        dt.Rows.Count <= 1,
        "Too many rows returned");
}
```

尽管单元测试的代码长度是它所测试的存储过程的两倍多，这一点可能令人不安，但请记住，大部分代码可以很容易地转换为模板以便快速重用。如前所述，你可能会忍不住将通用的单元测试代码重构到一个数据访问库中，但要小心，以免你最终测试的是你的测试框架，而不是你试图测试的实际例程。在调试正常工作的代码以试图弄清楚单元测试为何失败时，可能会浪费很多时间，而这实际上是因为单元测试所依赖的某些代码的故障。

单元测试允许对接口进行快速、自动化的验证。本质上，它们帮助你作为开发人员保证在对系统进行更改时没有破坏任何明显的东西。从这个意义上说，它们是无价的。在具有良好建立的单元测试集的系统上进行开发是一种乐趣，因为每个开发人员不再需要担心因接口更改而破坏其他组件。如果有任何东西需要修复，单元测试将会发出警报。

### 回归测试

当你为特定的应用程序建立一套单元测试时，这些测试最终将充当一个**回归测试套件**，这将有助于防范**回归错误**——即当开发人员破坏了原本能工作的功能时出现的错误。对接口的任何更改——无论有意与否——都会导致单元测试失败（假设测试编写正确）。对于有意的更改，解决方案是相应地重写单元测试。但正是这些无意的更改，才是我们创建单元测试的目的，也是回归测试所针对的目标。

经验表明，修复应用程序中的错误常常会引入其他错误。在真实的开发场景中，这种情况发生的频率可能难以证实，但有观点认为在某些情况下可能高达 50%。通过建立回归测试套件，修复这些“副作用”错误的成本大大降低。它们可以在开发阶段被发现和修复，而不是在应用程序部署后由最终用户报告。

回归测试也是一些较新的软件开发方法的关键，例如敏捷开发和极限编程（XP）。随着这些方法日益普及，其采用方式也渗透到数据库领域，可以预期数据库开发人员将开始更乐于采纳其中的一些技术。

## 实施数据库测试流程和程序的指南

在构成测试策略的所有可能要素中，成功的关键其实只有一个：一致性。测试必须是可重复的，并且每次必须以相同的方式运行，只有已知的（即已理解并有文档记录的）变量发生变化。不一致性，或者对测试之间可能发生变化的变量缺乏了解，可能意味着在测试中发现的任何问题都难以追踪。

