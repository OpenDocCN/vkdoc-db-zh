# 管理检查约束

## 工作原理

在讨论约束管理时，我并非涵盖所有约束。就 ETL 的目的而言，我建议将主键约束和唯一约束视为索引。尽管如此，你可能希望或需要像处理索引一样删除并重新创建检查约束。因此，这里提供了用于获取约束元数据并生成 `DROP` 和 `ADD` 脚本的脚本，你可以在暂存过程中使用。

它执行以下操作：

*   首先，它将 `DROP` 和 `ADD` 检查约束所需的元数据存储在表 `MetaData_CheckConstraints` 中。
*   其次，它利用这些信息生成并执行 `DROP` 脚本。
*   最后，它利用这些信息生成并执行 `ADD` 脚本。

这些脚本可以放置在 ETL 过程中的适当阶段，如示例 14-2 所示。

`check_constraints` 表与（不可避免地）来自 `sys.tables`、`sys.schemas` 和 `sys.columns` 的数据一起，用于获取元数据。然后，该元数据被塑造成用于管理检查约束的 `DROP` 和 `CREATE` 脚本。

```sql
definition
FROM       sys.schemas AS SCH
           INNER JOIN sys.tables AS TBL
           ON SCH.schema_id = TBL.schema_id
           INNER JOIN sys.check_constraints CHK
           ON TBL.object_id = CHK.parent_object_id
           INNER JOIN sys.columns COL
           ON CHK.parent_object_id = COL.object_id
           AND CHK.parent_column_id = COL.column_id
WHERE      CHK.is_disabled = 0 ;
```

## 3. 通过运行以下 T-SQL 代码段（`C:\SQL2012DIRecipes\CH14\DropCheckConstraints.Sql`）在目标数据库中删除检查约束：

```sql
-- 创建 DROP 语句
-- 删除用于保存脚本元素的表
IF OBJECT_ID('tempdb..#ScriptElements') IS NOT NULL
DROP TABLE tempdb..#ScriptElements;

CREATE TABLE #ScriptElements (ID INT IDENTITY(1,1), ScriptElement NVARCHAR(MAX))

INSERT INTO #ScriptElements
SELECT DISTINCT
'ALTER TABLE '
+ SCHEMA_NAME
+ '.'
+ TABLE_NAME
+ ' DROP CONSTRAINT '
+ CheckConstraintName
FROM       MetaData_CheckConstraints ;

-- 执行 DROP 脚本
DECLARE @DropFK NVARCHAR(MAX)

DECLARE  DropFK_CUR CURSOR
FOR
SELECT ScriptElement FROM #ScriptElements ORDER BY ID

OPEN DropFK_CUR
FETCH NEXT FROM DropFK_CUR INTO @DropFK
WHILE @@FETCH_STATUS <> −1
BEGIN
EXEC (@DropFK)
FETCH NEXT FROM DropFK_CUR INTO @DropFK
END ;

CLOSE DropFK_CUR;
DEALLOCATE DropFK_CUR;
```

## 4. 执行数据加载过程以更新目标数据库。

## 5. 使用类似以下代码，从持久化的元数据重新创建外键（`C:\SQL2012DIRecipes\CH14\CreateCheckConstraints.Sql`）：

```sql
-- 删除用于保存脚本元素的表
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
+ CheckConstraintName
+ ' CHECK '
+ [definition]
FROM MetaData_CheckConstraints ;

-- 执行 CREATE 脚本
DECLARE @CreateFK NVARCHAR(MAX)

DECLARE  CreateFK_CUR CURSOR
FOR
SELECT ScriptElement FROM #ScriptElements ORDER BY ID

OPEN CreateFK_CUR
FETCH NEXT FROM CreateFK_CUR INTO @CreateFK
WHILE @@FETCH_STATUS <> −1
BEGIN
EXEC (@CreateFK)
FETCH NEXT FROM CreateFK_CUR INTO @CreateFK
END ;

CLOSE CreateFK_CUR;
DEALLOCATE CreateFK_CUR;
```

