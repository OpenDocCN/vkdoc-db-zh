# 处理 XML 索引元数据

## 用于填充元数据表的查询

以下是用于填充 `#Metadata_XMLIndexes` 临时表的查询：

```sql
SELECT
    @SERVER_NAME AS SERVER_NAME,
    @DATABASE_NAME AS DATABASE_NAME,
    SCH.name AS SCHEMA_NAME,
    TBL.name AS TABLE_NAME,
    SIX.name AS PrimaryIndexName,
    TBL.object_id AS TableID,
    SIX.index_id AS XMLPrimaryIndexID,
    XIN.index_id AS XMLSecondaryaryIndexID,
    XIN.name AS INDEX_NAME,
    XIN.secondary_type_desc AS SecondaryTypeDescription,
    DSP.name AS DataSpace,
    XIN.allow_page_locks AS AllowPageLocks,
    XIN.allow_row_locks AS AllowRowLocks,
    XIN.is_padded AS IsPadded,
    0,
    COL.name
FROM sys.indexes SIX
INNER JOIN sys.tables TBL ON SIX.object_id = TBL.object_id
INNER JOIN sys.xml_indexes XIN ON SIX.object_id = XIN.object_id
    AND SIX.index_id = XIN.using_xml_index_id
INNER JOIN sys.schemas SCH ON TBL.schema_id = SCH.schema_id
INNER JOIN sys.data_spaces DSP ON XIN.data_space_id = DSP.data_space_id
INNER JOIN sys.index_columns SIC ON XIN.index_id = SIC.index_id
    AND XIN.object_id = SIC.object_id
INNER JOIN sys.columns COL ON SIC.object_id = COL.object_id
    AND SIC.column_id = COL.column_id
WHERE XIN.secondary_type_desc IS NULL;
```

## 执行删除 XML 索引的脚本

执行以下代码以在选定的数据库中 `DROP` XML 索引（`C:\SQL2012DIRecipes\CH14\DropXMLIndexes.Sql`）：

```sql
-- 删除用于保存脚本元素的表
IF OBJECT_ID('tempdb..#ScriptElements') IS NOT NULL
    DROP TABLE tempdb..#ScriptElements;

CREATE TABLE #ScriptElements (ID INT IDENTITY(1,1), ScriptElement NVARCHAR(MAX))

-- 非聚集索引
IF OBJECT_ID('tempdb..#Tmp_IndexedFields') IS NOT NULL
    DROP TABLE tempdb..#Tmp_IndexedFields;

-- 二级 XML 索引
INSERT INTO #ScriptElements
SELECT DISTINCT
    'DROP INDEX '
    + INDEX_NAME
    + ' ON '
    + DATABASE_NAME + '.' + SCHEMA_NAME + '.' + TABLE_NAME
FROM Metadata_XMLIndexes
WHERE IsPrimaryXMLIndex = 0;

-- 主 XML 索引
INSERT INTO #ScriptElements
SELECT DISTINCT
    'DROP INDEX '
    + PrimaryIndexName
    + ' ON '
    + DATABASE_NAME + '.' + SCHEMA_NAME + '.' + TABLE_NAME
FROM Metadata_XMLIndexes
WHERE IsPrimaryXMLIndex = 1;

-- 删除并执行 DROP 脚本
DECLARE @DropIndex NVARCHAR(MAX)
DECLARE DropIndex_CUR CURSOR
FOR SELECT ScriptElement FROM #ScriptElements ORDER BY ID

OPEN DropIndex_CUR
FETCH NEXT FROM DropIndex_CUR INTO @DropIndex

WHILE @@FETCH_STATUS <> -1
BEGIN
    EXEC (@DropIndex)
    FETCH NEXT FROM DropIndex_CUR INTO @DropIndex
END;

CLOSE DropIndex_CUR;
DEALLOCATE DropIndex_CUR;
```

## 运行 ETL 并重新创建索引

现在，您可以运行您的 ETL 包，将源数据加载到目标或暂存数据库中。运行以下 T-SQL 在暂存数据库中重新创建 XML 索引（`C:\SQL2012DIRecipes\CH14\CreateXMLIndexes.Sql`）：

```sql
DECLARE @SORT_IN_TEMPDB BIT = 1
DECLARE @DROP_EXISTING BIT = 0

-- 创建表以保存脚本元素
IF OBJECT_ID('tempdb..#ScriptElements') IS NOT NULL
    DROP TABLE tempdb..#ScriptElements;

CREATE TABLE #ScriptElements (ID INT IDENTITY(1,1), ScriptElement NVARCHAR(MAX))

-- 非聚集索引
IF OBJECT_ID('tempdb..#Tmp_IndexedFields') IS NOT NULL
    DROP TABLE tempdb..#Tmp_IndexedFields;

INSERT INTO #ScriptElements
SELECT DISTINCT
    'CREATE PRIMARY XML INDEX '
    + PrimaryIndexName
    + ' ON '
    + DATABASE_NAME + '.' + SCHEMA_NAME + '.' + TABLE_NAME
    + ' (' + XMLColumnName + ') '
    + ' WITH ('
    + CASE WHEN is_padded = 1 THEN ' PAD_INDEX = OFF, ' ELSE ' PAD_INDEX = ON, ' END
    + CASE WHEN [allow_page_locks] = 1 THEN ' ALLOW_PAGE_LOCKS = ON, ' ELSE 'ALLOW_PAGE_LOCKS = OFF, ' END
    + CASE WHEN [allow_row_locks] = 1 THEN ' ALLOW_ROW_LOCKS = ON, ' ELSE ' ALLOW_ROW_LOCKS = OFF, ' END
    + CASE WHEN @SORT_IN_TEMPDB = 1 THEN ' SORT_IN_TEMPDB = ON, ' ELSE ' SORT_IN_TEMPDB = OFF, ' END
    + CASE WHEN @DROP_EXISTING = 1 THEN ' DROP_EXISTING = ON, ' ELSE ' DROP_EXISTING = OFF ' END
    + ')'
FROM Metadata_XMLIndexes
WHERE IsPrimaryXMLIndex = 1;

-- 二级 XML 索引
INSERT INTO #ScriptElements
SELECT
    'CREATE XML INDEX '
    + INDEX_NAME
    + ' ON '
    + DATABASE_NAME + '.' + SCHEMA_NAME + '.' + TABLE_NAME
    + ' (' + XMLColumnName + ') '
    + ' USING XML INDEX '
    + PrimaryIndexName
    + ' FOR ' + secondarytypedescription
    + ' WITH ('
    + CASE WHEN is_padded = 1 THEN ' PAD_INDEX = OFF, ' ELSE ' PAD_INDEX = ON, ' END
    + CASE WHEN [allow_page_locks] = 1 THEN ' ALLOW_PAGE_LOCKS = ON, ' ELSE 'ALLOW_PAGE_LOCKS = OFF, ' END
    + CASE WHEN [allow_row_locks] = 1 THEN ' ALLOW_ROW_LOCKS = ON, ' ELSE ' ALLOW_ROW_LOCKS = OFF, ' END
    + CASE WHEN @SORT_IN_TEMPDB = 1 THEN ' SORT_IN_TEMPDB = ON, ' ELSE ' SORT_IN_TEMPDB = OFF, ' END
    + CASE WHEN @DROP_EXISTING = 1 THEN ' DROP_EXISTING = ON, ' ELSE ' DROP_EXISTING = OFF ' END
    + ')'
FROM Metadata_XMLIndexes
WHERE IsPrimaryXMLIndex = 0
ORDER BY SERVER_NAME, DATABASE_NAME, SCHEMA_NAME, TABLE_NAME;

-- 创建并执行 CREATE 脚本
DECLARE @CreateIndex NVARCHAR(MAX)
DECLARE CreateIndex_CUR CURSOR
FOR SELECT ScriptElement FROM #ScriptElements ORDER BY ID

OPEN CreateIndex_CUR
FETCH NEXT FROM CreateIndex_CUR INTO @CreateIndex

WHILE @@FETCH_STATUS <> -1
BEGIN
    EXEC (@CreateIndex)
    FETCH NEXT FROM CreateIndex_CUR INTO @CreateIndex
END;

CLOSE CreateIndex_CUR;
DEALLOCATE CreateIndex_CUR;
```

### 工作原理

这个方法类似于配方 14-3、14-4 和 14-5 的总和，但将整个过程的各个部分整合在了一起。它执行以下操作（并且仅在您已经生成了“核心”索引元数据——特别是主键之后才能工作）：

*   首先，它将 `DROP` 和 `CREATE` XML 索引所需的元数据存储在 `Metadata_XMLIndexes` 表中。
*   其次，它使用此信息生成并执行 `DROP` 脚本。
*   最后，它使用此信息生成并执行 `CREATE` 脚本。

这些脚本可以按照配方 14-2 中演示的方式，在 ETL 过程的适当阶段放置。脚本创建的字段描述见 表 14-3。

**表 14-3。用于存储 XML 索引元数据的字段**

| 字段名 | 源表 | 描述 |
| --- | --- | --- |
| `SERVER_NAME` |  | 从当前环境获取服务器名称。 |



| | DATABASE_NAME |  | 从当前环境中获取数据库名称。 | | SCHEMA_NAME | sys.schemas | 架构名称。 | | TABLE_NAME | sys.tables | 表名。 | | PrimaryIndexName | sys.indexes | 主键的索引名称。 | | TableID | sys.tables | 系统表中的内部表 ID。 | | INDEX_NAME | sys.xml_indexes | 主 XML 索引名称。 | | XMLSecondaryaryIndexID | sys.xml_indexes | 辅助 XML 索引的内部 ID。 | | XMLPrimaryIndexID | sys.xml_indexes | 主 XML 索引的内部 ID。 | | SecondaryTypeDescription | sys.xml_indexes | 辅助索引的类型。 | | DataSpace | sys.dataspaces | 索引存储位置。 | | Allow_Page_Locks | sys.xml_indexes | 是否允许页锁？ | | Allow_Row_Locks | sys.xml_indexes | 是否允许行锁？ | | Is_Padded | sys.xml_indexes | 索引是否进行填充？ | | IsPrimaryXMLIndex | sys.xml_indexes | 这是否是主 XML 索引。 | | XMLColumnName | sys.xml_indexes | 存储 XML 数据的表中的列名。 |
![image](img/sq.jpg)

**注意** 除非表的主键已首先创建，否则无法重新创建 XML 索引。

## 提示、技巧和陷阱

*   用于存储 XML 索引元数据的脚本（包括 `DROP` 和随后的 `CREATE` 索引操作）可以转化为存储过程，并在 SSIS 包中的 SSIS 执行 SQL 任务中调用。
*   如果愿意，可以将 XML 索引元数据存储在单独的数据库或单独的架构中。

## 14-7. 查找缺失的索引

### 问题
你怀疑目标数据库中缺少索引，并希望推断并创建它们。

### 解决方案
使用缺失索引动态管理视图 (DMV)。这需要你运行以下 T-SQL 代码片段：

```sql
SELECT     TOP (100) PERCENT
           sys.objects.name AS TableName
          ,COALESCE(sys.dm_db_missing_index_details.equality_columns ,
                    sys.dm_db_missing_index_details.inequality_columns) AS IndexCols
          ,sys.dm_db_missing_index_details.included_columns
          ,user_seeks
          ,user_scans
FROM        sys.dm_db_missing_index_details
             INNER JOIN sys.objects
             ON sys.dm_db_missing_index_details.object_id = sys.objects.object_id
             INNER JOIN sys.dm_db_missing_index_groups
             ON sys.dm_db_missing_index_details.index_handle =
                sys.dm_db_missing_index_groups.index_handle
             INNER JOIN sys.dm_db_missing_index_group_stats
             ON sys.dm_db_missing_index_groups.index_group_handle =
                sys.dm_db_missing_index_group_stats.group_handle
ORDER BY    TableName, IndexCols ;
```

### 工作原理
使用 SQL Server 的缺失索引 DMV 来确定是否存在任何缺失的索引——以及这些索引是否有用。缺失索引 DMV 不仅能提供可用于编写必要 `CREATE INDEX` 语句的核心列和表信息，还能通过 `user_seeks` 和 `user_scans` 列指示索引被进程使用的次数。仅使用一次的索引可能不值得创建。如果使用了多次，则应认真考虑创建。

我的建议是——假设你是在开发服务器上运行 ETL 过程，且只有你连接——重启 SQL Server 实例，然后从头到尾运行你的 ETL 过程。这样，只有你（开发者）可以决定 ETL 过程中是否需要使用某个索引。SQL Server 提供了许多工具来建议索引；你可以决定是否应用它们。

我不会在此试图详尽描述索引需求——因为纸质和网络上已有成千上万页的相关出版物，这是不必要的。因此，我希望从 ETL 角度出发的简要概述就足够了。

如果你希望就特定查询获得 SQL Server 关于缺失索引的意见，那么使用 `STATISTICS XML` 选项通常是最快速、最简单的途径。你只需将你的查询包裹在 `SET STATISTICS XML ON` 和 `SET STATISTICS XML OFF` 之间，如下所示：

```sql
SET STATISTICS XML ON

SELECT ...
FROM ...
WHERE ...

SET STATISTICS XML OFF
GO
```

然后，如果存在潜在的缺失索引，你将在 XML 显示计划的顶部看到一个 XML 片段，其中指示了任何缺失的索引。这读起来并不特别困难；对于每个表，它会指示任何等式、不等式和包含列。你可以在 `SHOWPLAN` 语句之间“嵌套”一系列 SQL 语句，甚至是存储过程列表，并获得一系列 XML 结果，然后你可以独立分析这些结果。

另一种查看潜在缺失索引——以及创建这些索引的脚本——的方法是，在 SSMS/SSDT 中显示实际或估计的执行计划。除了执行计划，你还会以一种显眼的绿色看到缺失索引。右键单击缺失索引会提供 `Missing Index Details...`（缺失索引详细信息）选项。点击它可以生成 `CREATE INDEX` 语句脚本。

你需要知道，如果你同时请求，SSMS/SSDT 不会同时输出执行计划和 XML 显示计划。此外，除了工具栏按钮，你还可以使用“查询/显示估计的执行计划”和“包含实际执行计划”来显示执行计划。归根结底，这与输入 `SHOWPLAN` 语句的输出相同。

如果你更喜欢一种更稳健的方法来调整查询，你可能更愿意使用数据库引擎优化顾问。它提供了超出你所需的信息，但其中大部分是有用的——例如估计的查询大小。要使用数据库引擎优化顾问，请执行以下步骤。

1.  右键单击并在数据库引擎优化顾问中选择“分析查询”。
2.  连接到数据库引擎优化顾问。“查询”应在“常规”选项卡中被选中。
3.  选择你将用于查询的数据库。
4.  单击工具栏中的“开始分析”。
5.  在“建议”选项卡中，对于“建议目标”列包含“索引”一词的每条建议，在其“定义”列中右键单击以获取 `CREATE INDEX` 脚本。

### 提示、技巧和陷阱
*   再次记住——你不必使用建议的索引。
*   由于数据库引擎优化顾问的多个选项在联机丛书和网络上的许多优秀文章中都有详细解释，我在此不再赘述。

## 14-8. 管理 CHECK 约束

### 问题
你希望在加载数据前删除 CHECK 约束，并在数据加载后重新启用它们。

### 解决方案
使用 SQL Server 元数据创建所需的 `DROP` 和 `CREATE` 脚本。以下过程收集 CHECK 约束的元数据。然后执行 `DROP` 语句。数据加载后，你可以运行脚本来重新创建 CHECK 约束。

1.  使用如下 DDL 创建用于保存 CHECK 约束持久化元数据的表 (`C:\SQL2012DIRecipes\CH14\tblMetaData_CheckConstraints.Sql`)：

    ```sql
    IF OBJECT_ID('dbo.MetaData_CheckConstraints') IS NOT NULL
        DROP TABLE dbo.MetaData_CheckConstraints;

    CREATE TABLE CarSales_Staging.dbo.MetaData_CheckConstraints
    (
        SCHEMA_NAME NVARCHAR (128) NOT NULL,
        TABLE_NAME NVARCHAR (128) NOT NULL,
        COLUMN_NAME NVARCHAR (128) NOT NULL,
        CheckConstraintName NVARCHAR (128) NOT NULL,
        definition NVARCHAR(max) NULL,
    ) ;
    GO
    ```

2.  使用以下代码片段填充外键约束的元数据表 (`C:\SQL2012DIRecipes\CH14\CheckConstraintMetadata.Sql`)：

    ```sql
    INSERT INTO MetaData_CheckConstraints
    (
        SCHEMA_NAME
        ,TABLE_NAME
        ,COLUMN_NAME
        ,CheckConstraintName
        ,definition
    )
    SELECT
        SCH.name AS SCHEMA_NAME
        ,TBL.name AS TABLE_NAME
        ,COL.name
        ,CHK.name AS CheckConstraintName
        ,CHK.
    ```



