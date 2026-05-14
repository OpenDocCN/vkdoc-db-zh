# SQL Server XML 数据导出技巧

#### 提示、技巧与陷阱

- 可以通过当前最主流的解压工具，或使用 `GZIPStream` 类的 SSIS 脚本任务来解压文件。
- BCP 可执行文件的路径通常在安装时被添加到 `PATH` 系统变量中。如果未完成此操作，你可能会发现手动添加它是值得的。
- 像这样的简单脚本可以通过使用变量来保存表名等方式进行增强。如果你愿意，可以自行扩展它。
- 此方法不能替代有效的备份策略。

## 7-10. 导出小型数据集为 XML

### 问题
你需要将 SQL Server 表中的非常小的数据集导出为 XML 文件，这很可能是为了响应用户完全“临时”的请求。

### 解决方案
在 T-SQL `SELECT` 子句中手动构建 XML，然后将输出复制到文本编辑器中保存。

1.  使用以下代码片段，将来自示例 CarSales 数据库中 `Client` 表的数据，添加到 XML 文件（`C:\SQL2012DIRecipes\CH07\SimpleXMLExport.sql`）中，生成格式良好的 XML：
    ```sql
    CREATE VIEW CarSales.dbo.vw_XMLTest
    AS
        SELECT XMLCol, SortKey FROM
        (
            SELECT '<Root>' AS XMLCol, 0 AS SortKey
            UNION
            SELECT
                '<Client>'
                + '<ID>' + ISNULL(CAST(ID AS VARCHAR(12)),'') + '</ID>'
                + '<ClientName>' + ISNULL(ClientName,'') + '</ClientName>'
                + '</Client>' AS XMLCol
                ,1 AS SortKey
            FROM dbo.Client
            UNION
            SELECT '</Root>' AS XMLCol, 2 AS SortKey
        ) MainXML ;
    GO
    ```

2.  在 SSMS 查询窗口中运行以下代码：
    ```sql
    SELECT * FROM CarSales.dbo.vw_XMLTest;
    ```
3.  将输出复制并粘贴到文本文件中。
4.  保存文本文件。

### 工作原理
导出简单 XML 的最简单方法是手工构建，如此配方所示。这仅适用于你导出的 XML 确实非常简单的情况——本质上，是指你将一个表或视图导出为“扁平化”的 XML，没有层级结构。然而，如果是这种情况——那么你很幸运，因为无需将输出分解成块，你可以快速轻松地输出 XML，每条数据库记录成为一条输出记录。

在生产环境中，你可能会使用 `FOR XML`，所以我必须承认，这主要是一个理论上的例子。这里我将 XML 包装在一个视图中，但这在并非所有情况下都是必要的。将其包装在视图中的原因是为了包含 `<Root>` 元素以形成有效的 XML。通过使用硬编码的 `SortKey` 列强制了输出的正确顺序。如果需要对 XML 记录排序，请使用辅助排序键。你可能更喜欢将 XML `SELECT` 子句包装在存储过程中——这也可以从 BCP 运行。

这个配方引出了一个问题：“什么是小型数据集？”不幸的是，答案只能是“视情况而定”。对我来说，这里使用的技术仅适用于导出少量列（考虑到手工构建 XML 很繁琐）的简单 XML 格式，并且行集足够小，可以复制和粘贴。然而，你可以根据你的需求决定何时——或何时不——使用它。由于此方法确实仅为更临时的 XML 导出而设计，我假设简单的剪切和粘贴足以将输出保存为文件。关于更完整的导出 XML 数据的方法，请参阅配方 7-11。

## 7-11. 导出较大数据集为 XML

### 问题
你希望在单个进程中将中等大小的数据集从 SQL Server 表导出为 XML 文件。如果可能，你想使用 T-SQL。

### 解决方案
创建一个 BCP 格式文件，然后与 BCP 一起使用以导出 XML 文件。

以下代码片段将把包含来自 `Client` 表数据的格式良好的 XML 添加到 XML 文件中。

1.  创建以下 BCP 格式文件（`C:\SQL2012DIRecipes\CH07\XmlExport.Fmt`）：
    ```
    11.0
    1
    1       SQLBINARY     0       0       ""    1     XmlData          ""
    ```

2.  使用以下 T-SQL 将 XML 输出到表中（`C:\SQL2012DIRecipes\CH07\ExportXMLFromTSQL.sql`）：
    ```sql
    IF OBJECT_ID('dbo.XMLOutput') IS NOT NULL DROP TABLE XMLOutput;

    CREATE TABLE XMLOutput (XMLCol XML);

    INSERT INTO XMLOutput (XMLCol)

    SELECT XMLOut FROM
    (
        SELECT  C.ID, ClientName, Country, Town
        FROM    CarSales.dbo.Client C
        FOR XML PATH('Client'), ROOT('RootElement')
    ) A (XMLOut);

    EXEC Master.dbo.xp_cmdshell 'BCP CarSales.dbo.XMLOutput OUT C:\SQL2012DIRecipes\CH07\XMLOut.xml –w –SADAM02 -T'
    ```

3.  从命令窗口中运行以下命令，以导出你在前面代码片段中创建的表：
    ```
    BCP XMLOutput OUT C:\SQL2012DIRecipes\CH07\Clients.Xml -T -N -f C:\SQL2012DIRecipes\CH07\XmlFormat.Fmt
    ```

4.  使用以下代码行删除包含 XML 的表：
    ```sql
    DROP TABLE XMLOutput;
    ```

### 工作原理
自 SQL Server 2005 出现以来，XML 已成为 Microsoft 旗舰 DBMS 的组成部分，因此，从 SQL Server 数据库导出 XML 数据相对容易。然而，XML 是一个庞大的概念，导出数据的塑形方式可能相当复杂。因此，在涉及 XML 输出的配方中，我保持简单的 XML 输出示例，因为形成复杂 XML 本身就是一个足够庞大的主题，值得整章——甚至整本书来讨论。

由于 XML 本质上只是文本，你会看到导出除最简单 XML 之外的任何内容时，基本上有两个主要难点：

- 定义（塑形）XML
- 将通常只是一个大记录的内容导出到文本文件

关于定义 XML，在大多数情况下我使用 `FOR XML` 子句，因为它比尝试手工构建除最简单 XML 之外的所有内容要可取得多。建议阅读配方 7-14，简要介绍基本的 `FOR XML`。

此技术使用 `FOR XML PATH` 从源表创建 XML，然后使用 BCP 导出它。值得注意的几点是：首先，目标文件不需要存在——如果存在将被覆盖。其次，必须启用 `xp_CmdShell` 才能从 T-SQL 调用 BCP。最后，不能使用临时表，因此在使用 BCP 时必须使用“真实”表。这是因为临时表对 BCP 不可见——即使你将此代码包装在存储过程中并从 BCP 调用它。

当涉及到将结果输出到磁盘时，如果 XML 输出很小，通常很容易。当 XML 较大时，情况可能变得更复杂，因为 SSIS 和 BCP 会因 `FOR XML` 生成的单记录输出而遇到问题。然而，几乎所有问题都可以通过使用各种“分块”例程来解决，这些例程允许你将 `FOR XML` 生成的单块单记录 XML 输出分解为多条记录。XML 输出的这个附加要求可能看起来很复杂，但对于大型 XML 文件来说是基础性的。更多内容将在下一个配方中介绍。

#### 提示、技巧与陷阱

- 如果你将查询在 SSMS 中测试好再复制到 BCP 命令行，可以避免不必要的麻烦。
- 目标文件不需要存在——如果存在将被覆盖。
- 你总可以将 BCP 命令创建为 `.cmd` 文本文件，然后通过双击运行它，而不是在命令窗口中操作。
- 如果你更喜欢使用 CLR 存储过程写入磁盘，请参阅配方 10-15。


