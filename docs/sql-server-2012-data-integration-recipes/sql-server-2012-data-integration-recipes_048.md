# SQL Server 中加载和处理 XML 数据

## 问题

需要使用 T-SQL 从目录加载多个 XML 文件，并将其作为 XML 数据类型存储到 SQL Server 表中。

## 解决方案

使用 `xp_cmdshell` 和动态 SQL 从文件系统读取文件，并利用 `OPENROWSET` 和 `BULK` 选项将整个文件加载为 XML。

```sql
DECLARE @SQL VARCHAR(8000);
-- 用于保存文件名的表变量
DECLARE @FileList TABLE (XMLFile VARCHAR(150));
-- 用于捕获"原始"文件列表的表变量
DECLARE @DirList TABLE (List VARCHAR(250));

-- 将 DIR 命令的输出加载到 @DirList
-- 表变量中
INSERT INTO @DirList EXEC master.dbo.xp_cmdshell 'Dir D:\BIProject\*.Xml';

-- 解析出文件名 - 注意事项：文件名中不能有空格，
-- 所有文件必须具有正确且相同的结构...
INSERT INTO @FileList
SELECT REVERSE(LEFT(REVERSE(List), CHARINDEX(' ', REVERSE(List))))
FROM @DirList
WHERE List LIKE '%.Xml%';

-- 游标用于循环遍历文件名并加载 xml 文件
DECLARE @FileName VARCHAR(150);

DECLARE FileLoad_CUR CURSOR FOR
SELECT XMLFile FROM @FileList

OPEN FileLoad_CUR
FETCH NEXT FROM FileLoad_CUR INTO @FileName

WHILE @@FETCH_STATUS <> -1
BEGIN
    -- XML 加载过程
    SET @SQL = 'INSERT INTO AA1 (XMLData) SELECT CAST(XMLSource AS XML) AS XMLSource FROM OPENROWSET(BULK ''' + @FileName + ''', SINGLE_BLOB) AS X (XMLSource)'
    EXEC (@SQL)

    FETCH NEXT FROM FileLoad_CUR INTO @FileName
END;

CLOSE FileLoad_CUR;
DEALLOCATE FileLoad_CUR;
```

用于加载多个文件的 T-SQL 工作原理如下：

*   首先，使用 `xp_cmdshell` 调用 `Dir`（目录列表）命令来定义待处理文件的列表。
*   然后，解析此列表以获得可用的文件名。
*   一个游标遍历文件列表并加载它们。

此处使用的动态 SQL 做了一些预设：

*   文件具有 `.Xml` 扩展名。
*   文件名中没有空格。

如果需要，可以扩展脚本以接受多个扩展名或包含空格的文件名。请注意，本方案仅旨在帮助你将 XML 数据作为 XML 加载到 SQL Server 表中。遗憾的是，提取和查询数据的多种方法超出了本书的范围。但是，如果你将 XML 数据存储为 XML 数据类型，请记住 SQL Server 允许你为 XML 列创建索引，以使用 SQL Server 支持的 XQuery 语言子集进行更快的查询。

这里使用的"关键字"当然是可选的，我保留它是为了展示 XML 和"普通"数据如何可以一起导入。重要的是要使用 `SINGLE_BLOB` 参数，以便整个文件作为一列的内容处理。此外，任何 XML 文档都不能大于 2 千兆字节。另请注意，此处使用的 `CONVERT` 参数 `0` 会丢弃无关的空白，并且不允许内部 DTD（文档类型定义）子集。我假设不会使用 DTD。

要查询 XML 列的内容，你可以使用以下 T-SQL 语法，例如：

```sql
SELECT XMLDataStore.query('(/ROOT/Customer/CustomerID)')
FROM XmlImport
```

## 提示、技巧和陷阱

*   动态 SQL 的前提是拥有使用 `xp_cmdshell` 的权限。许多 DBA 不会容忍这一点，因此你需要发掘巨大的魅力储备——或无可辩驳的技术论据——来获得允许。或者，你可以编写一个 CLR（公共语言运行时）例程来列出目录中的文件，并说服 DBA 这种方法更安全。
*   通过使用 `SINGLE_BLOB`（相对于 `SINGLE_CLOB` 或 `SINGLE_NCLOB`），你可以避免 XML 文档的编码（如 XML 编码声明中所给出）与服务器的代码页之间潜在的错配。
*   方案 3-2 和 3-3 展示了如何从 XML 列获取数据并将其分解到目标表的单独列中。
*   震惊和恐怖——一个游标！嗯，正如我在本书其他地方所说，游标并非必然是错的，并且在有些场合它们是易于使用和调试的最简单解决方案——而且资源影响可以忽略不计。整个方法的前提是你可能最多只加载几十个 XML 文件。如果你将加载更多文件，那么一个无游标的方案（方案 6-7）可以适应你的需求。

## 3-2. 将 XML 数据加载到行和列中

### 问题

你想将中小型 XML 文件中的数据加载到 SQL Server 表中，并正确地"分解"为行和列。

### 解决方案

使用 `OPENXML` 将以元素为中心和以属性为中心的 XML 文件加载并分解到 SQL Server 表中。

1.  将（本例中是以元素为中心的）XML 文件放在必需的源目录中。文件如下所示（`C:\SQL2012DIRecipes\CH03\ClientLite.Xml`）：
    ```xml
    <CarSales>
      <Client>
        <ID> 3</ID>
        <ClientName> John Smith</ClientName>
        <Country> 1</Country>
      </Client>
    ```
    为节省空间省略数据...
    ```xml
      <Client>
        <ID> 7</ID>
        <ClientName> Slow Sid</ClientName>
        <Country> 3</Country>
      </Client>
    </CarSales>
    ```

2.  使用以下 T-SQL 代码加载 XML 文件（`C:\SQL2012DIRecipes\CH03\ShredClientLite.Sql`）：
    ```sql
    DECLARE @DocID INT;
    DECLARE @DocXML VARCHAR(MAX);

    SELECT @DocXML = CAST(XMLSource AS VARCHAR(MAX))
    FROM OPENROWSET(BULK 'C:\SQL2012DIRecipes\CH03\ClientLite.xml', SINGLE_BLOB)
                    AS X (XMLSource);
    EXECUTE master.dbo.sp_xml_preparedocument @DocID OUTPUT, @DocXML;

    SELECT ID, ClientName, Country
    INTO XmlTable
    FROM OPENXML(@DocID, 'CarSales/Client', 2)
                    WITH (
                          ID VARCHAR(50)
                          ,ClientName VARCHAR(50)
                          ,Country VARCHAR(10)
                         );
    ```
    `EXECUTE master.dbo.sp_xml_removedocument @DocID;`

3.  查询 `XmlTable` 表，你应该会看到类似以下结果：
    ```
     ID     ClientName        Country
     3      John Smith           1
     4      Bauhaus Motors       2
     5      Honest Fred          3
     6      Fast Eddie           2
     7      Slow Sid             3
    ```

### 工作原理

自 SQL Server 2000 发布以来，`OPENXML` 一直帮助开发人员和 DBA 加载（或"分解"）XML 数据到关系表中。该技术在当前版本的 SQL Server 中仍然受支持，并且易于使用且高效。结果与方案 3-1 中获得的结果完全不同，因为这次源文件被分解为其组成的数据片段，每个元素或属性被加载到一个或多个表中的单独列。

首先，将源文件加载到 `@DocXML` 变量中。然后，对该变量运行 `sp_xml_preparedocument` 存储过程以准备 XML 并返回一个句柄，本例中为 `@DocID`。该句柄传递给 `OPENXML` 命令，该命令读取 XML 数据并将其分解到目标表的行和列中。然后，在 `OPENXML` 语句中，行模式（'`CarSales/Client`'）标识要处理的 `<Client>` 节点。

`WITH` 子句是 `ColPattern`，它允许你遍历 XML 的层次结构并选择任何你想要的元素。对于属性，你可以只使用像 `Invoice/@InvoiceNumber` 这样的 `ColPattern`。你可以使用（例如）`../Invoice` 遍历超出初始节点指定的 XML——假设行模式是 `'CarSales/Client'/Invoice`。如果未指定 `ColPattern`，则将采用默认映射（指定的以属性为中心或以元素为中心的映射）。

但请注意，尽管简单，`OPENXML` 非常消耗内存。此外，`OPENXML` 使用的是 XPath（而不是 XQuery）。



