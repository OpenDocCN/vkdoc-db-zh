# 14-9. 管理外键约束

## 问题

你希望在加载数据前删除外键约束，并在数据加载后重新启用它们。

## 解决方案

使用 SQL Server 元数据创建所需的 `DROP` 和 `CREATE` 脚本。以下脚本收集并存储外键约束的元数据。然后在数据加载前运行 `DROP` 脚本。加载完成后，重新应用外键约束。

### 1. 使用类似以下 DDL 创建用于保存外键约束持久化元数据的表（`C:\SQL2012DIRecipes\CH14\tblTmp_Metadata_ForeignKeys.Sql`）：

```sql
IF OBJECT_ID('dbo.Tmp_Metadata_ForeignKeys') IS NOT NULL
DROP TABLE dbo.Tmp_Metadata_ForeignKeys

CREATE TABLE CarSales_Staging.dbo.Tmp_Metadata_ForeignKeys
(
 SCHEMA_NAME SYSNAME NOT NULL,
 TABLE_NAME SYSNAME NOT NULL,
 FOREIGN_KEY_NAME SYSNAME NOT NULL,
 COLUMN_NAME SYSNAME NULL,
 REFERENCED_TABLE_SCHEMA_NAME SYSNAME NOT NULL,
 REFERENCED_TABLE_NAME SYSNAME NOT NULL,
 REFERENCED_COLUMN_NAME SYSNAME NULL,
 ColumnList NVARCHAR(300) NULL
);
GO
```

### 2. 使用以下代码段填充外键约束的元数据表（`C:\SQL2012DIRecipes\CH14\ForeignKeyMetadata.Sql`）：

```sql
INSERT INTO Tmp_Metadata_ForeignKeys
SELECT
SCH.name AS SCHEMA_NAME
,OBJ.name AS TABLE_NAME
,FRK.name AS FOREIGN_KEY_NAME
,COL.name AS COLUMN_NAME
,RefSCH.name AS REFERENCED_TABLE_SCHEMA_NAME
,RefTBL.name AS REFERENCED_TABLE_NAME
,RefCOL.name AS REFERENCED_COLUMN_NAME
,CAST(NULL AS NVARCHAR(300)) AS ColumnList
FROM       sys.foreign_keys FRK
           INNER JOIN sys.objects OBJ
           ON FRK.parent_object_id = OBJ.object_id
           AND FRK.schema_id = OBJ.schema_id
           INNER JOIN sys.schemas SCH
           ON OBJ.schema_id = SCH.schema_id
           INNER JOIN sys.foreign_key_columns FKC
           ON FRK.object_id = FKC.constraint_object_id
           AND FRK.parent_object_id = FKC.parent_object_id
           INNER JOIN sys.columns COL
           ON FKC.constraint_column_id = COL.column_id
           AND FKC.parent_object_id = COL.object_id
           INNER JOIN sys.columns AS RefCOL
           ON FKC.referenced_object_id = RefCOL.object_id
           AND FKC.referenced_column_id = RefCOL.column_id
           INNER JOIN sys.objects AS RefTBL
           ON FKC.referenced_object_id = RefTBL.object_id
           INNER JOIN sys.schemas AS RefSCH
           ON RefTBL.schema_id = RefSCH.schema_id
WHERE      FRK.is_ms_shipped = 0
           AND FRK.is_not_trusted = 0 ;

-- 定义（连接）键列列表
IF OBJECT_ID('TempDB..#Tmp_IndexedFields') IS NOT NULL
DROP TABLE TempDB..#Tmp_IndexedFields

;
WITH Core_CTE ( FOREIGN_KEY_NAME, Rank, COLUMN_NAME )
             AS ( SELECT FOREIGN_KEY_NAME,
ROW_NUMBER() OVER( PARTITION BY FOREIGN_KEY_NAME ORDER BY
FOREIGN_KEY_NAME),
CAST( COLUMN_NAME AS VARCHAR(MAX) )
FROM Tmp_Metadata_ForeignKeys),

Root_CTE ( FOREIGN_KEY_NAME, Rank, COLUMN_NAME )
AS ( SELECT FOREIGN_KEY_NAME, Rank, COLUMN_NAME
FROM   Core_CTE
WHERE  Rank = 1 ),

Recursion_CTE ( FOREIGN_KEY_NAME, Rank, COLUMN_NAME )
AS
  ( SELECT FOREIGN_KEY_NAME, Rank, COLUMN_NAME
    FROM Root_CTE
    UNION ALL
    SELECT Core_CTE.FOREIGN_KEY_NAME, Core_CTE.Rank,
    Recursion_CTE.COLUMN_NAME + ', ' + Core_CTE.COLUMN_NAME
    FROM Core_CTE
    INNER JOIN Recursion_CTE
    ON Core_CTE.FOREIGN_KEY_NAME = Recursion_CTE.FOREIGN_KEY_NAME
    AND Core_CTE.Rank = Recursion_CTE.Rank + 1
  )
```

### 3. 删除目标数据库中的外键

运行以下 T-SQL 代码片段 (`C:\SQL2012DIRecipes\CH14\DropForeignKeys.Sql`) 以删除目标数据库中的外键：

```sql
-- 创建临时表用于存放脚本元素
IF OBJECT_ID('tempdb..#ScriptElements') IS NOT NULL
    DROP TABLE tempdb..#ScriptElements;

CREATE TABLE #ScriptElements (ID INT IDENTITY(1,1), ScriptElement NVARCHAR(MAX));

INSERT INTO #ScriptElements
SELECT DISTINCT
    'ALTER TABLE '
    + SCHEMA_NAME
    + '.'
    + TABLE_NAME
    + ' ADD CONSTRAINT '
    + FOREIGN_KEY_NAME
    + ' FOREIGN KEY ('
    + ColumnList
    + ') REFERENCES '
    + REFERENCED_TABLE_SCHEMA_NAME
    + '.'
    + REFERENCED_TABLE_NAME
    + ' ('
    + ColumnList
    + ')'
FROM Tmp_Metadata_ForeignKeys;

-- 执行 DROP 脚本
DECLARE @DropFK NVARCHAR(MAX);
DECLARE DropIndex_CUR CURSOR
FOR
SELECT ScriptElement FROM #ScriptElements ORDER BY ID;

OPEN DropIndex_CUR;
FETCH NEXT FROM DropIndex_CUR INTO @DropFK;

WHILE @@FETCH_STATUS <> -1
BEGIN
    EXEC (@DropFK);
    FETCH NEXT FROM DropIndex_CUR INTO @DropFK;
END;

CLOSE DropIndex_CUR;
DEALLOCATE DropIndex_CUR;
```

### 4. 执行数据加载过程以更新目标数据库。

### 5. 使用以下代码，根据持久化的元数据重新创建外键 (`C:\SQL2012DIRecipes\CH14\CreateForeignKeys.Sql`)：

```sql
-- 创建临时表用于存放脚本元素
IF OBJECT_ID('tempdb..#ScriptElements') IS NOT NULL
    DROP TABLE tempdb..#ScriptElements;

CREATE TABLE #ScriptElements (ID INT IDENTITY(1,1), ScriptElement NVARCHAR(MAX));

INSERT INTO #ScriptElements
SELECT DISTINCT
    'ALTER TABLE '
    + SCHEMA_NAME
    + '.'
    + TABLE_NAME
    + ' ADD CONSTRAINT '
    + FOREIGN_KEY_NAME
    + ' FOREIGN KEY ('
    + ColumnList
    + ') REFERENCES '
    + REFERENCED_TABLE_SCHEMA_NAME
    + '.'
    + REFERENCED_TABLE_NAME
    + ' ('
    + ColumnList
    + ')'
FROM Tmp_Metadata_ForeignKeys;

-- 执行 CREATE 脚本
DECLARE @CreateFK NVARCHAR(MAX);
DECLARE DropIndex_CUR CURSOR
FOR
SELECT ScriptElement FROM #ScriptElements ORDER BY ID;

OPEN DropIndex_CUR;
FETCH NEXT FROM DropIndex_CUR INTO @CreateFK;

WHILE @@FETCH_STATUS <> -1
BEGIN
    EXEC (@CreateFK);
    FETCH NEXT FROM DropIndex_CUR INTO @CreateFK;
END;

CLOSE DropIndex_CUR;
DEALLOCATE DropIndex_CUR;
```

### 工作原理
`sys.objects`、`sys.foreign_keys`、`sys.schemas`、`sys.foreign_key_columns` 和 `sys.columns` 表被用于获取元数据。这比检查约束所需的元数据稍微复杂一些，但它有效！然后基于此元数据编写 `DROP` 和 `CREATE` 脚本。你只需要复制并执行生成的脚本即可。它执行以下操作（与配方 14-8 非常相似）：

*   首先，将删除 (`DROP`) 和添加 (`ADD`) 外键所需的元数据存储在 `Tmp_Metadata_ForeignKeys` 表中。
*   其次，使用此信息生成并执行 `DROP` 脚本。
*   最后，使用此信息生成并执行 `ADD` 脚本。

这些脚本可以像配方 14-2 中演示的那样，放置在 ETL 过程的适当阶段。

### 提示、技巧和陷阱
*   在删除约束之前，始终对暂存数据库进行脚本编写并制作备份副本。
*   在 ETL 过程的各个阶段添加外键约束时，你可以使用约束创建脚本，并使用相关的表名进行筛选，以根据需要创建特定的外键约束。
*   此脚本假设元数据中不需要引号标识符。如果你的数据库命名约定不是这样，则需要处理引号标识符。
*   你可能更愿意先测试外键是否工作（使用一段 T-SQL 来隔离不符合要求的记录）。

### 14-10. 优化批量加载

#### 问题
你希望批量加载尽可能快地运行。

#### 解决方案
确保使用 `BulkLoad` API 执行加载。然后验证在数据加载期间是否仅使用了最小日志记录，以及是否适当地处理了索引和约束。以下是使用 SSIS 的示例。

1.  使用以下 DDL 创建目标表 (`C:\SQL2012DIRecipes\CH14\tblFastLoadClients.Sql`)：

```sql
CREATE TABLE CarSales_Staging.dbo.FastLoadClients
(
    ID INT,
    ClientName NVARCHAR (150),
    Address1 VARCHAR (50),
    Address2 VARCHAR (50),
    Town VARCHAR (50),
    County VARCHAR (50),
    PostCode VARCHAR (10),
    ClientType VARCHAR (20),
    ClientSize VARCHAR(10),
    ClientSince DATETIME,
    IsCreditWorthy BIT,
    DealerGroup BINARY(892)
) ;
GO
```

2.  创建一个新的 SSIS 包。
3.  添加两个 OLEDB 连接管理器，第一个命名为 `CarSales_OLEDB`，连接到 CarSales 数据库；第二个命名为 `CarSales_Staging_OLEDB`，连接到 CarSales_Staging 数据库。
4.  创建以下元数据表：`CarSales_Staging.dbo.IndexList`（配方 14-3，步骤 1）、`CarSales_Staging.dbo.Metadata_XMLIndexes`（配方 14-6，步骤 1）、`CarSales_Staging.dbo.MetaData_CheckConstraints`（配方 14-8，步骤 1）和 `CarSales_Staging.dbo.Tmp_Metadata_ForeignKeys`（配方 14-9，步骤 1）。
5.  添加一个名为 `GatherIndexMetadata` 的执行 SQL 任务。将其配置为使用 `CarSales_Staging_OLEDB` 连接管理器。将 SQL 源类型设置为“直接输入”，并将 SQL 语句设置为配方 14-3 步骤 2 中的代码。
6.  添加一个名为 `GatherXMLIndexMetadata` 的执行 SQL 任务。将其连接到先前创建的任务（`GatherIndexMetadata`）。将其配置为使用 `CarSales_Staging_OLEDB` 连接管理器。将 SQL 源类型设置为“直接输入”，并将 SQL 语句设置为配方 14-6 步骤 2 中的代码。
7.  添加一个名为 `GatherCheckConstraintMetadata` 的执行 SQL 任务。将其连接到先前创建的任务（`GatherXMLIndexMetadata`）。将其配置为使用 `CarSales_Staging_OLEDB` 连接管理器。将 SQL 源类型设置为“直接输入”，并将 SQL 语句设置为配方 14-8 步骤 2 中的代码。
8.  添加一个名为 `GatherForeignKeyMetadata` 的执行 SQL 任务。将其配置为使用 `CarSales_Staging_OLEDB` 连接管理器。将其连接到先前创建的任务（`GatherCheckConstraintMetadata`）。将 SQL 源类型设置为“直接输入”，并将 SQL 语句设置为配方 14-9 步骤 2 中的代码。
9.  添加一个名为 `DropForeignKeys` 的执行 SQL 任务。将其配置为使用 `CarSales_Staging_OLEDB` 连接管理器。将其连接到先前创建的任务（`GatherForeignKeyMetadata`）。将 SQL 源类型设置为“直接输入”，并将 SQL 语句设置为配方 14-9 步骤 3 中的代码。
10. 添加一个名为 `DropCheckConstraints` 的执行 SQL 任务。将其配置为使用 `CarSales_Staging_OLEDB` 连接管理器。将其连接到先前创建的任务（`DropForeignKeys`）。将 SQL 源类型设置为“直接输入”，并将 SQL 语句设置为配方 14-8 步骤 3 中的代码。
11. 添加一个名为 `DropXMLIndexes` 的执行 SQL 任务。将其配置为使用 `CarSales_Staging_OLEDB` 连接管理器。


将其连接到先前创建的任务（**DropCheckConstraints**）。将 SQL 源类型设置为**直接输入**，并将**SQL 语句**设置为配方 14-6 步骤 3 中给出的代码。

## 添加和配置任务

12. 添加一个名为`DropIndexes`的执行 SQL 任务。将其配置为使用`CarSales_Staging_OLEDB`连接管理器。将其连接到先前创建的任务（**DropXMLIndexes**）。将 SQL 源类型设置为**直接输入**，并将**SQL 语句**设置为配方 14-4 中给出的代码。
13. 添加一个名为`Set BulkLogged Recovery`的执行 SQL 任务。将其配置为使用`CarSales_Staging_OLEDB`连接管理器。将其连接到先前创建的任务（**DropIndexes**）。将 SQL 源类型设置为**直接输入**，并将**SQL 语句**设置为如下所示的代码：
    ```sql
    ALTER DATABASE CarSales
    SET RECOVERY BULK_LOGGED;
    ```
14. 添加一个数据流任务，将其连接到名为`Set BulkLogged Recovery`的执行 SQL 任务。双击进行编辑。
15. 添加一个 OLEDB 源，将其配置为使用`CarSales_OLEDB`连接管理器和源查询：
    ```sql
    SELECT        ID, ClientName, Address1, Address2, Town, County, PostCode, ClientType
                   ,ClientSize, ClientSince, IsCreditWorthy, DealerGroup
    FROM          Client
    ```
16. 添加一个 OLEDB 目标，将其连接到 OLEDB 源。双击进行编辑。
17. 选择`CarSales_Staging_OLEDB`连接管理器和`dbo.FastLoadClients`表，使用**表或视图-快速加载**数据访问模式。
18. 确保已勾选**表锁定**。
19. 点击“确定”完成配置。
20. 返回到数据流窗格。
21. 添加一个名为`CreateIndexes`的执行 SQL 任务。将其配置为使用`CarSales_Staging_OLEDB`连接管理器。将其连接到先前创建的数据流任务。将 SQL 源类型设置为**直接输入**，并将**SQL 语句**设置为配方 14-5 中给出的代码。
22. 添加一个名为`CreateXMLIndexes`的执行 SQL 任务。将其配置为使用`CarSales_Staging_OLEDB`连接管理器。将其连接到先前创建的任务（**CreateIndexes**）。将 SQL 源类型设置为**直接输入**，并将**SQL 语句**设置为配方 14-6 步骤 5 中给出的代码。
23. 添加一个名为`CreateCheckConstraints`的执行 SQL 任务。将其配置为使用`CarSales_Staging_OLEDB`连接管理器。将其连接到先前创建的任务（**CreateXMLIndexes**）。将 SQL 源类型设置为**直接输入**，并将**SQL 语句**设置为配方 14-8 步骤 5 中给出的代码。
24. 添加一个名为`CreateForeignKeys`的执行 SQL 任务。将其配置为使用`CarSales_Staging_OLEDB`连接管理器。将其连接到先前创建的任务（**CreateCheckConstraints**）。将 SQL 源类型设置为**直接输入**，并将**SQL 语句**设置为配方 14-9 步骤 5 中给出的代码。
25. 添加一个名为`Set BulkLogged Recovery`的执行 SQL 任务。将其配置为使用`CarSales_Staging_OLEDB`连接管理器。将其连接到先前创建的任务（**CreateForeignKeys**）。将 SQL 源类型设置为**直接输入**，并将**SQL 语句**设置为以下代码：
    ```sql
    ALTER DATABASE CarSales
    SET RECOVERY FULL;
    ```

现在您可以运行加载过程了。完成后，您应*立即*备份数据库。

## 工作原理

在本配方中，我们的目标是两个有些相互依赖的目标：批量加载和最小化日志记录。尝试实现这些目标的原因很简单：

*   在数据加载过程中使用`SQL Server BulkLoad API`将按批次写入数据，而不是逐行写入。这不可避免地会显著缩短过程持续时间，因为使用批量加载 API 总是比“正常”的逐行写入表更快。它总是需要表锁定。
*   *最小化日志记录的操作*通常比完整日志记录的操作更快，因为其开销更低。这是因为最小化日志记录的操作只跟踪区分配和元数据更改。这意味着更低的内存需求、更少的磁盘 I/O 以及降低的处理器负载。最小化日志记录的操作还减少了一个潜在的风险领域，即日志空间。即使您的日志优化得非常完美，一次大型数据加载也可能填满日志——或磁盘——从而停止加载过程。如果在加载过程中必须调整日志大小，这将导致加载速度显著减慢，因为需要添加更多日志空间。

然而，许多批量加载操作要求数据库处于`SIMPLE`或`BULK_LOGGED`恢复模式才能使用批量加载 API。一个操作可以是批量加载操作而不进行最小化日志记录，但如果加载速度是主要目标，确保两者结合始终是首选结果。

然而，要达到数据加载的极致速度这一理想状态，并非仅仅是设置恢复模式并确保设置了`TABLOCK`提示那么简单。还需要考虑另外两个基本方面：复制和索引。

第一个方面很简单：如果表上启用了复制，那么根据定义，最小化日志记录是不可能的，因为复制过程会使用事务日志。

索引要复杂得多，因为结果可能取决于索引类型（聚集或非聚集）以及表是否为空。让我们更详细地看看这些可能性。

*   *堆表（无索引）*：这是最简单的情况，数据页被最小化日志记录。
*   *堆表（带非聚集索引）*：数据页被最小化日志记录。然而，当表为空时，索引页被最小化日志记录；但如果表中有数据，则索引页被完整日志记录。此规则有一个例外——加载到空表中的第一个批次。对于这第一个批次，数据和索引都被最小化日志记录。对于第二个及后续批次，只有数据被最小化日志记录——索引页被完整日志记录。
*   *聚集索引*：如果表为空，数据和索引页被最小化日志记录。如果表包含数据，则数据和索引页都被完整日志记录。

您会注意到，在本配方中，我在加载期间切换到了**大容量日志恢复模式**。加载完成后我还备份了数据库。这是一个公认简单化的情况，即目标数据库显然是暂存区，并且在加载过程中没有事务运行。我知道这在许多其他情况下可能并非如此，这可能导致这种方法不可行。

> **注意** 高效的批量加载可能比看起来更复杂，因为核心要求可以有许多种组合方式。所有这些都在 Microsoft 白皮书*数据加载性能指南* (`http://msdn.microsoft.com/en-us/library/dd425070(v=sql.100).aspx`) 中得到了详尽的描述。这份白皮书可能是为 SQL Server 2008 编写的，但其见解同样适用于 SQL Server 2012。然而，要领会使用 SQL Server 进行高性能数据加载的所有细微差别和相互作用，可能需要数月的经验积累。

本配方未涉及（但在第 13 章的许多配方中已详尽涵盖）的一个批量插入方面是数据的并行加载。根据我的经验，并行加载总是比单次加载更快。然而，这意味着要能够执行并行加载，这暗示着：

*   *目标表未被索引*——如此处所示，所有三种批量导入命令都支持并行数据导入：`BCP`、`BULK INSERT`和`INSERT... SELECT * FROM OPENROWSET(BULK...)`。这是因为当表上存在索引时，仅通过应用`TABLOCK`提示无法执行并行加载操作。因此，在批量导入操作之前，必须删除索引。


