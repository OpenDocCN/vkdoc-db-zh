# 14-6. 存储元数据，然后为目标数据库 XML 索引编写和执行 DROP 与 CREATE 语句

## 问题

你在一个表中使用了 XML 索引，该表将作为大型 ETL 更新的目标，你需要作为 ETL 过程的一部分删除并重新创建这些索引。

## 解决方案

存储用于为 XML 索引创建 `DROP` 和 `CREATE` 语句所需的元数据。然后使用此信息来管理索引本身。

1.  创建以下表来保存 XML 索引的持久化元数据 (`C:\SQL2012DIRecipes\CH14\tblMetadata_XMLIndexes.Sql`):

    ```sql
    IF OBJECT_ID('dbo.Metadata_XMLIndexes') IS NOT NULL
        DROP TABLE dbo.Metadata_XMLIndexes;

    CREATE TABLE CarSales_Staging.dbo.Metadata_XMLIndexes
    (
        SERVER_NAME NVARCHAR(128) NULL,
        DATABASE_NAME NVARCHAR(128) NULL,
        SCHEMA_NAME NVARCHAR(128) NULL,
        TABLE_NAME NVARCHAR(128) NULL,
        PrimaryIndexName NVARCHAR(128) NULL,
        TableID INT NULL,
        XMLPrimaryIndexID INT NULL,
        XMLSecondaryaryIndexID INT NULL,
        INDEX_NAME NVARCHAR(128) NULL,
        SecondaryTypeDescription NVARCHAR(60) NULL,
        DataSpace NVARCHAR(128) NULL,
        Allow_Page_Locks BIT NULL,
        Allow_Row_Locks BIT NULL,
        Is_Padded BIT NULL,
        IsPrimaryXMLIndex BIT NULL,
        XMLColumnName NVARCHAR(128) NULL
    ) ;
    GO
    ```

2.  运行以下脚本以整理和存储 XML 索引元数据 (`C:\SQL2012DIRecipes\CH14\GatherXMLIndexMetadata.Sql`):

    ```sql
    INSERT INTO Metadata_XMLIndexes
    (
        SERVER_NAME
        ,DATABASE_NAME
        ,SCHEMA_NAME
        ,TABLE_NAME
        ,PrimaryIndexName
        ,XMLPrimaryIndexID
        ,DataSpace
        ,TableID
        ,[Allow_Page_Locks]
        ,Allow_Row_Locks
        ,Is_Padded
        ,IsPrimaryXMLIndex
        ,XMLColumnName
    )
    SELECT
        @SERVER_NAME
        ,@DATABASE_NAME
        ,SCH.name AS SCHEMA_NAME
        ,TBL.name AS TABLE_NAME
        ,XIN.name AS INDEX_NAME
        ,XIN.index_id AS IndexID
        ,DSP.name AS DataSpace
        ,TBL.object_id AS TableID
        ,XIN.allow_page_locks
        ,XIN.allow_row_locks
        ,XIN.is_padded
        ,1
        ,COL.name
    FROM    sys.tables TBL
            INNER JOIN sys.xml_indexes XIN
            ON TBL.object_id = XIN.object_id
            INNER JOIN sys.schemas SCH
            ON TBL.schema_id = SCH.schema_id
            INNER JOIN sys.data_spaces DSP
            ON XIN.
    ```

```sql
' + TABLE_NAME + ' (' + IndexFields_Main + ')' + ' WITH (' + CASE WHEN is_padded = 1 THEN ' PAD_INDEX  = OFF, ' ELSE ' PAD_INDEX  = ON, ' END + CASE WHEN IsNoRecompute = 1 THEN ' STATISTICS_NORECOMPUTE  = OFF, ' ELSE  'STATISTICS_NORECOMPUTE  = ON, ' END + CASE WHEN [ignore_dup_key] = 1 THEN ' IGNORE_DUP_KEY  = ON, ' ELSE ' IGNORE_DUP_KEY  = OFF, ' END + CASE WHEN [allow_page_locks] = 1 THEN ' ALLOW_PAGE_LOCKS  = ON, ' ELSE 'ALLOW_PAGE_LOCKS  = OFF, ' END + CASE WHEN fill_factor IS NOT NULL THEN ' FILLFACTOR  = ' + CAST(fill_factor AS VARCHAR(5)) + ',' ELSE '' END + CASE WHEN [allow_row_locks] = 1 THEN ' ALLOW_ROW_LOCKS  = ON, ' ELSE ' ALLOW_ROW_LOCKS  = OFF, ' END + CASE WHEN @SORT_IN_TEMPDB = 1 THEN ' SORT_IN_TEMPDB  = ON, ' ELSE ' SORT_IN_TEMPDB  = OFF, ' END + CASE WHEN @DROP_EXISTING = 1 THEN ' DROP_EXISTING  = ON, ' ELSE ' DROP_EXISTING  = OFF, ' END + CASE WHEN @ONLINE = 1 THEN ' ONLINE  = ON ' ELSE ' ONLINE  = OFF ' END + CASE WHEN @MAX_DEGREE_OF_PARALLELISM = 0 THEN '' ELSE ' MAXDOP = ' + CAST(@MAX_DEGREE_OF_PARALLELISM AS NVARCHAR(2)) END + ')' + ' ON [' + DataSpace + ']'
FROM        #Tmp_IndexData
WHERE       #Tmp_IndexData.is_primary_key = 0
            AND #Tmp_IndexData.is_unique_constraint = 0
            AND type_desc = 'CLUSTERED'

-- 创建唯一索引
INSERT INTO #ScriptElements (ScriptElement)
SELECT DISTINCT ' ALTER TABLE ' + SCHEMA_NAME + '.' + TABLE_NAME + ' ADD CONSTRAINT ' + INDEX_NAME + ' UNIQUE ' + CASE WHEN type_desc = 'CLUSTERED' THEN 'CLUSTERED ' ELSE 'NONCLUSTERED  ' END + ' (' + IndexFields_Main + ')' + ' ON [' + DataSpace + ']'
FROM      #Tmp_IndexData
WHERE     #Tmp_IndexData.is_unique_constraint = 1 ;

-- 创建非聚集索引
INSERT INTO #ScriptElements (ScriptElement)
SELECT DISTINCT 'CREATE ' + CASE WHEN is_unique = 1 THEN 'UNIQUE ' ELSE '' END + 'NONCLUSTERED INDEX ' + INDEX_NAME + ' ON ' + SCHEMA_NAME + '.' + TABLE_NAME + ' (' + IndexFields_Main + ')' + CASE WHEN IndexFields_Included IS NOT NULL THEN ' INCLUDE ' + ' (' + IndexFields_Included + ')' ELSE '' END + CASE WHEN has_filter = 1 THEN ' WHERE ' + filter_definition ELSE '' END + ' WITH (' + CASE WHEN is_padded = 1 THEN ' PAD_INDEX  = OFF, ' ELSE ' PAD_INDEX  = ON, ' END + CASE WHEN IsNoRecompute = 1 THEN ' STATISTICS_NORECOMPUTE  = OFF, ' -- 检查逻辑 ELSE  'STATISTICS_NORECOMPUTE  = ON, ' END + CASE WHEN [ignore_dup_key] = 1 THEN ' IGNORE_DUP_KEY  = ON, ' ELSE ' IGNORE_DUP_KEY  = OFF, ' END + CASE WHEN [allow_page_locks] = 1 THEN ' ALLOW_PAGE_LOCKS  = ON, ' ELSE 'ALLOW_PAGE_LOCKS  = OFF, ' END + CASE WHEN fill_factor IS NOT NULL THEN ' FILLFACTOR  = ' + CAST(fill_factor AS VARCHAR(5)) + ',' ELSE '' END + CASE WHEN [allow_row_locks] = 1 THEN ' ALLOW_ROW_LOCKS  = ON, ' ELSE ' ALLOW_ROW_LOCKS  = OFF, ' END + CASE WHEN @SORT_IN_TEMPDB = 1 THEN ' SORT_IN_TEMPDB  = ON, ' ELSE ' SORT_IN_TEMPDB  = OFF, ' END + CASE WHEN @DROP_EXISTING = 1 THEN ' DROP_EXISTING  = ON, ' ELSE ' DROP_EXISTING  = OFF, ' END + CASE WHEN @ONLINE = 1 THEN ' ONLINE  = ON ' ELSE ' ONLINE  = OFF ' END + CASE WHEN @MAX_DEGREE_OF_PARALLELISM = 0 THEN '' ELSE ' MAXDOP = ' + CAST(@MAX_DEGREE_OF_PARALLELISM AS NVARCHAR(2)) END + ')' + ' ON [' + DataSpace + ']'
FROM       #Tmp_IndexData
WHERE      #Tmp_IndexData.is_primary_key = 0
           AND #Tmp_IndexData.is_unique_constraint = 0
           AND type_desc = 'NONCLUSTERED'  ;

-- 创建并执行 CREATE 脚本
DECLARE @CreateIndex NVARCHAR(MAX)
DECLARE  CreateIndex_CUR CURSOR
FOR SELECT ScriptElement FROM #ScriptElements ORDER BY ID

OPEN CreateIndex_CUR
FETCH NEXT FROM CreateIndex_CUR INTO @CreateIndex
WHILE @@FETCH_STATUS <> −1
BEGIN
    EXEC (@CreateIndex)
    FETCH NEXT FROM CreateIndex_CUR INTO @CreateIndex
END ;

CLOSE CreateIndex_CUR ;
DEALLOCATE CreateIndex_CUR ;
```


