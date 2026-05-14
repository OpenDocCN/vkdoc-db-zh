# 第三章

![image](img/frontdot.jpg)

## XML 数据源

这已成老生常谈，但 XML 现在是数据交换的通用语言。在四、五年前，分隔文本文件还是常态，如今 XML 文件正取而代之。XML 当然有其缺点——尤其是与文本文件源数据相比，文件大小绝非小事。但仅就基于 XML 数据加载的可靠性和易于调试而言，我任何时候都倾向于使用 XML 文件而非平面文件作为源数据。

因此，本章将引导你了解将 XML 数据导入 SQL Server 的经典方法。然而，在开始之前，有几句警告。XML 极其复杂，是无数鸿篇巨著的主题——这些著作无疑又将成为其他更多著作的主题。本章绝不会以任何方式介绍 XML 及其所有微妙之处。相反，它将引导你了解将此类源数据加载到 SQL Server 中的标准方法。

### 3-1. 加载 XML 文件以存储在 SQL Server 中

#### 问题

你希望将一个 XML 文档——或多个 XML 片段——加载到 SQL Server 表中，并在其中“按原样”存储。

#### 解决方案

从 T-SQL 使用 `OPENROWSET (BULK)` 将 XML 文件推入 SQL Server，而无需将其“分解”到多个列中。

1.  使用以下代码创建目标表（`C:\SQL2012DIRecipes\CH03\tblXmlImportTest.Sql`）：
    ```sql
    CREATE TABLE XmlImportTest
    (
     Keyword NVARCHAR(50),
     XMLDataStore XML
    );
    GO
    ```
2.  找到一个 XML 源文件，如下面的简单示例（`C:\SQL2012DIRecipes\CH03\Clients.Xml`）：
    ```xml
    <?xml version = "1.0" encoding = "UTF-8"?>
    <CarSales>
      <Client>
        <ID > 3</ID>
        <ClientName> John Smith</ClientName>
        <Address1> 4, Grove Drive</Address1>
        <Town> Uttoxeter</Town>
        <County> Staffs</County>
        <Country> 1</Country>
      </Client>
    ```
    . . . 此处为节省空间省略了许多其他元素 . . .
    ```xml
      <Client>
        <ID> 7</ID>
        <ClientName> Slow Sid</ClientName>
        <Address1> 2, Rue des Bleues</Address1>
        <Town> Avignon</Town>
        <County> Vaucluse</County>
        <Country> 3</Country>
      </Client>
    </CarSales>
    ```
3.  运行以下 T-SQL 代码以导入引用的 XML 文件的内容，以及你希望存储在表中的任何其他数据（`C:\SQL2012DIRecipes\CH03\XmlImportTestInsert.Sql`）。
    ```sql
    INSERT INTO XmlImportTest(XMLDataStore, Keyword);
    SELECT XMLDATAToStore, 'Attribute-Centric' AS ColType
    FROM
    (
       SELECT   CONVERT(XML, XMLCol, 0)
       FROM    OPENROWSET (BULK 'C:\SQL2012DIRecipes\CH03\Clients_Simple.Xml',
                           SINGLE_BLOB) AS XMLSource (XMLCol)
                      ) AS XMLFileToImport (XMLDATAToStore);
    ```

#### 工作原理

XML 文件可以直接作为纯 XML 数据加载到 SQL Server 表中——也就是说，它们不会被“分解”（或拆解）成行和列。这是通过 `OPENROWSET (BULK)` 完成的，它本质上是将源文件读入 T-SQL 进程。该文件既可以是一个 XML “片段”——即不强制符合 XML 架构，也可以是一个“格式良好”的 XML 文档，后者符合 XML 架构定义。该技术在以下情况下非常有用：
*   当你希望将 XML 文档或片段存储在数据库中，而不是文件系统中时。
*   当你不希望将 XML 数据分解为完全关系型的 SQL Server 表结构时。

这样的数据加载本身可能就是一个目标，即将 XML 加载到数据库后，你可以使用 SQL Server 用来查询 XML 数据类型的 XQuery 语言子集来查询这些 XML 数据。或者，它可以作为数据暂存过程的一部分，即你希望先将 XML 片段存储在 SQL Server 中，然后再进行其他处理。

该技术同样可以轻松地扩展以更新（或更确切地说，替换）XML 片段或文档。所需的代码如下（`C:\SQL2012DIRecipes\CH03\LoadXMLToUpdateInTable.sql`）：
```sql
UPDATE XmlImportTest SET XMLDataStore =
                    (
                     SELECT XMLDATAToStore
                     FROM
                         (
                          SELECT   CONVERT(XML, XMLCol, 0)
                          FROM  OPENROWSET (BULK
                                            'C:\SQL2012DIRecipes\CH03\Clients_Simple.Xml',
                                            SINGLE_BLOB) AS XMLSource (XMLCol)
                         ) AS XMLFileToImport (XMLDATAToStore)
                    )
WHERE Keyword = 'Attribute-Centric'
```
可能你有很多 XML 片段或文档要加载，并且不想开发一个 SSIS 包来执行此任务。幸运的是，只需一点巧思（和一些动态 SQL），当前配方中使用的脚本就可以扩展为加载多个 XML 片段或文档。加载一系列文件的脚本如下（`C:\SQL2012DIRecipes\CH03\LoadMultiplexmlFiles.



