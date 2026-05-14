# 第三章：数据库管理

## 索引管理

### 检查与创建索引

```sql
IF NOT EXISTS (SELECT * FROM sysindexes AS i JOIN sysobjects AS o on i.id = o.id WHERE o.name = 'chpt03_Location' and i.name = 'IX_chpt03_Location_Country')
BEGIN
    PRINT 'Adding index IX_chpt03_Location_Country'
    CREATE NONCLUSTERED INDEX IX_chpt03_Location_Country ON dbo.chpt03_Location
    (
        Country
    ) WITH( STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON)
    ON [PRIMARY]
END
GO
```

```sql
IF NOT EXISTS (SELECT * FROM sysindexes AS i JOIN sysobjects AS o on i.id = o.id WHERE o.name = 'chpt03_Person' and i.name = 'IX_chpt03_Person_BirthDate')
BEGIN
    PRINT 'Adding index IX_chpt03_Person_BirthDate'
    CREATE NONCLUSTERED INDEX IX_chpt03_Person_BirthDate ON dbo.chpt03_Person
    (
        BirthDate
    ) WITH( STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON)
    ON [PRIMARY]
END
GO
```

索引被轻松管理后，就该着手处理外键约束了。对于这类约束，同样没有现成的模板来在删除前检查现有约束。清单 3-8 中的脚本移除了 `Person` 和 `Location` 表之间的单个外键。

## 索引碎片整理

随着时间的推移，索引可能会产生碎片。有碎片的索引扫描起来比新创建的索引更耗时，因此有必要偶尔使用 `REORGANIZE` 命令来清理索引。该命令不会完全刷新索引，但能提升性能。`REBUILD` 命令可用于完全刷新索引。这些任务最好留给您的数据库管理员（DBA）。当您开始注意到性能下降时，可以考虑使用这种调优技术。

### 清单 3-8. `RemoveConstraints.sql`

```sql
IF EXISTS (SELECT * FROM dbo.sysobjects WHERE id = object_id(N'[dbo].[FK_chpt03_Person_chpt03_Location]') and OBJECTPROPERTY(id, N'IsForeignKey') = 1)
    ALTER TABLE [dbo].[chpt03_Person]
    DROP CONSTRAINT FK_chpt03_Person_chpt03_Location
```

然后，运行清单 3-9 中的约束来添加回约束。

### 清单 3-9. `AddConstraints.sql`

```sql
ALTER TABLE [dbo].[chpt03_Person] WITH CHECK ADD CONSTRAINT [FK_chpt03_Person_chpt03_Location] FOREIGN KEY([LocationId]) REFERENCES [dbo].[chpt03_Location] ([LocationId])
```

现在，每当表需要因更改而重新调整时，这个障碍就能轻松移除。如果您想添加列，可以更新表脚本，移除约束，运行表脚本，然后再添加回约束。

#### 性能考量

在介绍下一个示例之前，有一些事项需要考虑。在性能方面，有许多因素需要考量。要调优使用数据库的慢速应用程序，可以采取一些基本步骤，例如为表添加索引以及仅提取必要的数据。索引有助于更快地组装结果，而提取更少的数据则会减少传输的数据量。在结果集非常大的情况下，省略一些列会累积成可测量的差异。还可以进行许多其他更改来增强性能，但前两点是“唾手可得的果实”，通常能带来最大的收益。

当您将所有数据从数据库提取到 Web 应用程序以在 Web 窗体上显示后，数据绑定值将被序列化到 `ViewState` 中。如果数据量非常大，结果将是一个非常大的文件，网站用户将不得不下载它。

对于一个仅显示 100 行中的 10 行的 `GridView`，`ViewState` 仍然可以容纳所有 100 行的数据。虽然在开发过程中，当您与 Web 服务器在同一台计算机上工作时，具有如此大数据集的页面加载速度会很快，但对于远程访问网站的用户来说，这将会慢得令人痛苦。根据经验，网页大小必须小于 100 KB，最好小于 50 KB。

我曾经参与过一个网站项目，该网站使用了一个动态导航菜单，其 `ViewState` 和 `PostBack` 事件覆盖了网站层级的多个级别。它包含大量链接，这增加了 `ViewState` 的大小。典型的网页接近 300 KB，其中超过 90% 的页面内容是 `ViewState`。最初的开发者在他的笔记本电脑上工作，从未意识到这个问题，因为一切总是很快。他从不检查页面大小，也不关心从远程服务器加载页面需要多长时间。这类问题应该及早发现，那时更容易修复。

您可以随意调优表和查询，但如果页面是 300 KB，而您的许多用户仍在使用拨号上网甚至低端宽带速度，他们会发现您的网站很慢。相比之下，大多数主流新闻网站和信息门户的页面大小都在 30 KB 以下。在接下来的示例中，我们将通过一个简单的策略来解决 `ViewState` 问题。

**注意：** 数据传输时间只是导致网页加载缓慢的一个方面。包含许多行和单元格的大表格会增加网页的渲染时间。如果加载页面数据所需的时间不到一秒，渲染一个复杂表格可能仍然需要几秒钟。

更有效的方法是要求数据库仅返回将要显示的项目范围，而不是让数据库返回所有可能的项目。当仅显示十行时，从数据库向 Web 应用程序传输数千行数据是浪费的。

幸运的是，我们可以只请求该子集，但我们需要一个新的存储过程来实现这个新目标。清单 3-10 展示了创建这个特殊存储过程的脚本。

### 清单 3-10. `chpt03_GetPeopleSubSetSorted.sql`

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P' AND name = 'chpt03_GetPeopleSubSetSorted')
BEGIN
    DROP Procedure chpt03_GetPeopleSubSetSorted
END
GO

CREATE Procedure dbo.chpt03_GetPeopleSubSetSorted
(
    @sortExpression nvarchar(50),
    @startRowIndex int,
    @maximumRows int
)
AS
IF LEN(@sortExpression) = 0
    SET @sortExpression = 'PersonId'

-- 重置为基于 1 的索引
SET @startRowIndex = @startRowIndex + 1

-- 构建 sql
DECLARE @sql nvarchar(4000)
SET @sql = 'SELECT PersonId,FirstName,LastName,BirthDate,City,Country FROM
(SELECT p.PersonId,p.FirstName,p.LastName,p.BirthDate,l.City,l.Country,
ROW_NUMBER() over(ORDER BY ' + @sortExpression + ') AS RowNum
FROM chpt03_Person AS p
JOIN chpt03_Location AS l on l.LocationId = p.LocationId
) AS People
WHERE RowNum between ' + CONVERT(nvarchar(10), @startRowIndex) + ' and (' + CONVERT(nvarchar(10), @startRowIndex) + ' + ' + CONVERT(nvarchar(10), @maximumRows) + ') - 1'

-- 执行 SQL 查询
EXEC sp_executesql @sql
GO

GRANT EXEC ON chpt03_GetPeopleSubSetSorted TO PUBLIC
GO
```

## 添加到公共文件夹

清单 3-10 中的存储过程是一种技术，您可以多次应用它来减少数据库负载，同时加速应用程序层。此脚本可以放置在您 `Scripts` 子文件夹下的 `Common` 文件夹中，供未来项目引用（`D:\Projects\Common\Scripts\Database`）。

这个新的存储过程中发生了很多事情。它不返回 `Person` 表中的每一项。而是使用输入参数 `@startRowIndex` 和 `@maximumRows` 来定义要返回的项目范围。它还使用 `@sortExpression` 输入参数来提供排序功能。

SQL Server 2005 引入了新的 `Row_Number` 函数。给定选择的顺序，`Row_Number` 函数为每一行分配一个行号。此行号用于限制返回项的范围。


但 `@startRowIndex` 和 `@maximumRows` 这两个值必须有所来源。此信息由 `ObjectDataSource` 提供。当在 `ObjectDataSource` 上启用分页时，`SelectMethod` 必须引用一个具有这些参数的方法。我们必须向 `PersonDomain` 类添加这样一个方法。清单 3-11 展示了使用这些附加参数的示例方法。

### 清单 3-11. GetPeopleSubSetSortedDataSet 方法

```
[DataObjectMethod(DataObjectMethodType.Select)]
public DataSet GetPeopleSubSetSortedDataSet(int? startRowIndex, int? maximumRows)
{
    return GetPeopleSubSetSortedDataSet(null, startRowIndex, maximumRows);
}

[DataObjectMethod(DataObjectMethodType.Select)]
public DataSet GetPeopleSubSetSortedDataSet(
    string sortExpression, int? startRowIndex, int? maximumRows)
{
    DataSet ds = new DataSet();

    if (String.IsNullOrEmpty(sortExpression))
    {
        sortExpression = "";
    }

    if (!startRowIndex.HasValue)
    {
        startRowIndex = 0;
    }

    if (!maximumRows.HasValue)
    {
        maximumRows = 0;
    }

    using (DbCommand dbCmd = db.GetStoredProcCommand("chpt03_GetPeopleSubSetSorted"))
    {
        db.AddInParameter(dbCmd, "@sortExpression", DbType.String, sortExpression);
        db.AddInParameter(dbCmd, "@startRowIndex", DbType.Int64, startRowIndex);
        db.AddInParameter(dbCmd, "@maximumRows", DbType.Int64, maximumRows);
        ds = db.ExecuteDataSet(dbCmd);
    }

    //返回结果
    return ds;
}
```

在清单 3-11 中，有两个同名方法。区别在于第二个方法有额外的 `sortExpression` 参数。你可能会认出这是一种方法重载技术。根据 `ObjectDataSource` 是否支持排序，将使用此参数。第一个方法简单地以 `null` 值作为 `sortExpression` 调用第二个方法，从而引向清单 3-10 中的 `chpt03_GetPeopleSubSetSorted` 存储过程。

除了到目前为止我们一直在使用的 `SelectMethod` 属性外，`ObjectDataSource` 还有另一个属性，我们将使用它来使解决方案更高效地工作。这个名为 `SelectCountMethod` 的属性用于告诉数据绑定控件数据源返回的总项数。此值来自另一个存储过程。清单 3-12 展示了此存储过程。

### 清单 3-12. chpt03_GetPeopleRowCount.sql

```
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P' AND
name = 'chpt03_GetPeopleRowCount')
BEGIN
    DROP Procedure chpt03_GetPeopleRowCount
END
GO

CREATE Procedure dbo.chpt03_GetPeopleRowCount
(
    @Count int OUTPUT
)
AS
    SET @Count = (SELECT COUNT(*) AS [Count] FROM chpt03_Person)
GO

GRANT EXEC ON chpt03_GetPeopleRowCount TO PUBLIC
GO
```

此过程计算 `Person` 表中的所有记录，并在清单 3-13 所示的方法中使用。

### 清单 3-13. GetPeopleRowCount 方法

```
public long? GetPeopleRowCount()
{
    long? count = 0;

    using (DbCommand dbCmd = db.GetStoredProcCommand("chpt03_GetPeopleRowCount"))
    {
        db.AddOutParameter(dbCmd, "@Count", DbType.Int64, 0);
        db.ExecuteNonQuery(dbCmd);
        count = (long)db.GetParameterValue(dbCmd, "@Count");
    }

    return count;
}
```

最后，清单 3-14 中的 `ObjectDataSource` 将所有内容整合在一起，以便数据绑定控件可以使用。

### 清单 3-14. ObjectDataSource1

```
<asp:ObjectDataSource ID="ObjectDataSource1" runat="server"
    OldValuesParameterFormatString="original_{0}"
    TypeName="Chapter02.PersonDomain"
    SelectMethod="GetPeopleSubSetSortedDataSet"
    SelectCountMethod="GetPeopleRowCount"
    EnablePaging="True"
    SortParameterName="sortExpression">
</asp:ObjectDataSource>
```

有了这个 `ObjectDataSource`，你可以快速高效地分页浏览数千行数据。它还减少了从数据库移动到 Web 应用程序的数据量。

## 稳定性考虑

应用程序和数据库之间的联系常常被视为理所当然。因此，它没有得到足够的保护，也没有被视为真正的集成点。尽管我们希望数据库查询能像循环遍历内存中的对象数组一样可靠，但它并非如此。数据库的断开连接特性之一，是能够独立于另一方更改应用程序或数据库，尽管我们尽最大努力保持它们的兼容性，但很容易在不正确调整另一方的情况下更改一方。

对于类型化数据集，架构非常严格。单个列的最轻微更改都会导致应用程序中断。更改列名将导致非类型化数据集产生非预期行为。当应用程序和数据库之间的关系被视为集成点时，方法和假设就会改变。这种改变将允许采取额外的步骤来确保集成的稳定性。

在本章后面，你将看到处理数据的选项是什么，以及你可以做些什么来确保这条重要链接保持可靠。

## 隔离更改

任何软件项目的主要目标都应该是尽量减少维护量。当多个应用程序使用的数据库发生更改时，更新所有应用程序可能会产生大量工作。

但是，更改可以隔离到一个瓶颈点，所有应用程序都共享对此的依赖。考虑将你的类型化数据集和域类放在一个类库中，这将创建一个供所有应用程序使用的程序集。当数据库架构更改时，可以更新类库并将其部署到每个应用程序。在 `VARCHAR` 长度增长的情况下，它很容易被 .NET 吸收，因为 `String` 对象不关心大小，并且如果此类库突然开始返回更长的 `String` 值，也不会抛出异常。可能仍需要进行调整，例如验证控件属性，但这种方法通过将更改隔离到单个点，避免了在每个应用程序中更新类型化数据集引用。而且，通过这个单一的数据库访问点，我们有了测试类库的绝佳机会，以确保数据库中的更改已与代码同步。

## 数据的单元测试

不是每次更改都进行手动测试，而是可以使用单元测试，它可以自动运行一整套测试。这些测试可以确认每个存储过程和域方法都按照要求执行，并在某些内容确实中断时给你早期警告。早期警告让你有机会在破坏性更改后不久纠正问题，从而更容易识别和修复它。如果问题直到应用程序准备部署时才被注意到，识别原因将相当困难，这将使纠正变得更加困难。

.NET 最流行的单元测试框架是 NUnit。这是一个简单的框架，通过在类和方法上放置属性来表明它们是测试套件的一部分。NUnit 测试运行器将加载程序集，查找这些属性，并动态构建要运行的测试集合。在每个测试中，你将调用数据库，并使用 `Assert` 语句检查返回值，这些语句将成功或失败。当测试套件在所有测试中都成功时，指示灯全为绿色。但当测试失败时，它会亮红灯。

当单元测试就位并且测试套件全面覆盖你的代码库后，每个开发者都可以在将更改提交到源代码管理系统之前运行测试。



这样做可以将损坏的代码排除在源代码控制系统之外，否则这些损坏的代码会随着其他开发者拉取更新而蔓延到他们的系统中。但即使运行了测试，多个开发者的变更组合也可能破坏单元测试。为了防止这种情况，你可以设置一个构建服务器，定期从源代码控制中拉取更新、构建项目并运行测试。这通常被称为**持续集成**。

#### 持续集成

过去几年中最流行的持续集成解决方案一直是 `CruiseControl.NET`。它与 `NUnit` 等单元测试框架以及广泛的源代码控制系统兼容。它可以配置为每五分钟检查一次源代码控制系统是否有更新。当有更新时，`CruiseControl.NET` 会启动构建和测试过程。构建和测试的结果会通过构建报告记录下来，开发者可以通过电子邮件立即获得结果。

`CruiseControl.NET` 还有一个系统托盘应用程序，它与构建服务器保持联系，向开发者报告变更情况。当构建成功时，该应用程序会从系统托盘弹出一个通知气泡。当构建失败时，它会显示一个通知告知坏消息。`CruiseControl.NET` 还拥有一个网站，该网站显示每个已配置的项目及其所有构建报告列表。当构建失败时，你可以查看最新的构建报告，以确切了解问题所在。

它不仅会显示为最新构建而更改的文件，还会显示代码出错的具体行号。这些细节使得确定是谁破坏了构建以及是什么更改导致了破坏变得容易得多。自动化构建报告失败的一个附带好处是，它大大减少了针对个人的指责，开发者可以直接主动去修复自己的问题。我多次看到构建失败是因为一个文件遗漏了，没有与其他更改一起提交到源代码控制中。提交那个缺失的文件可以快速修复构建，大家就能恢复正常工作了。鉴于此类情况，最好提交你的更改并请求构建服务器立即运行构建，这样你就能确保在你的更改被包含后、但在你下班回家前，构建能成功运行。你最不希望发生的事情就是因为没有在最新更改中包含一个文件而被叫回公司。

接下来的章节将涵盖单元测试和持续集成的具体示例。它们将成为你日常工作流程的常规组成部分。

#### 总结

本章介绍了数据库管理流程，从在数据库项目中组织脚本，到创建用于管理存储过程、索引和约束的脚本。

最后回顾了可用于确保应用层与数据库持续良好协作的自动化测试技术。

`8601Ch04CMP3 8/25/07 1:07 PM Page 73`

## 第 4 章 数据绑定控件

`ASP.NET 2.0` 提供了许多有用的控件，这使得处理数据变得非常容易。微软提供的许多示例展示了如何仅需很少甚至无需代码就能快速组装一个数据驱动的网站。但有时，现实世界的需求会将你推出这个安全区，要求一个更复杂的解决方案。这些解决方案仍然可以是优雅的，并且只需很少的代码即可工作。你只需要更深入地了解如何利用现有控件，以及如何更复杂地组合使用这些控件。一旦你将你的方法分解成协同工作的小型组件，你就可以用可管理的组件来处理更复杂的需求。

本章涵盖以下内容：

• `DetailsView` 控件

• `FormView` 控件

• `GridView` 控件

• 编辑和验证字段

• 绑定输入参数

• 嵌入用户控件

• 创建数据绑定控件



