# 工作原理

如果我说这个方法的替代标题是“史上最短的 Oracle 数据典指南”，我希望我在这里描述的目标将是明确的。虽然有些数据库让您为获取足够的元数据而手忙脚乱，但在 Oracle 中，情况往往恰恰相反——您会感觉自己淹没在 Oracle 称为*数据字典*的信息海洋中。因此，这个方法提供了您可能获得的关于 Oracle 元数据的最短课程。我只能强调，获取 Oracle 元数据的方法有很多种。可用的信息如此之多，以至于我专注于一个极其简单的子集，这些是我过去发现有帮助的。我希望您会发现它们作为一个起点很有用，您可以在此基础上扩展和构建。

当然，对于接下来的每种方法，都有另外十几种方法可以获取您想要的东西——所以请继续搜索 Oracle 字典（在所有可用的资源中），因为您肯定会找到其他获取所需元数据的方法。

#### 8-13. 显示 DB2 元数据

## 问题

您需要查询即将从中导入数据的 DB2 数据库的元数据。

## 解决方案

使用 SSIS 从 DB2 的 `INFORMATION_SCHEMA` 表中捕获和存储 DB2 元数据。以下步骤描述了如何操作。

1.  使用以下 DDL 在 SQL Server 中创建表以保存 DB2 元数据：

    ```sql
    CREATE TABLE dbo.IS_DB2_Views
    (
     TABLE_CATALOG NVARCHAR(50) NULL,
     TABLE_SCHEMA NVARCHAR(50) NULL,
     TABLE_NAME NVARCHAR(50) NULL,
     VIEW_DEFINITION NVARCHAR(MAX) NULL,
     CHECK_OPTION NVARCHAR(10) NULL,
     IS_UPDATABLE NVARCHAR(5) NULL
    );
    GO

    CREATE TABLE dbo.IS_DB2_Tables
    (
     TABLE_CATALOG NVARCHAR(50) NULL,
     TABLE_SCHEMA NVARCHAR(50) NULL,
     TABLE_NAME NVARCHAR(50) NULL,
     TABLE_TYPE NVARCHAR(26) NULL
    );
    GO

    CREATE TABLE dbo.IS_DB2_TableConstraints
    (
     CONSTRAINT_CATALOG NVARCHAR(50) NULL,
     CONSTRAINT_SCHEMA NVARCHAR(50) NULL,
     CONSTRAINT_NAME NVARCHAR(50) NULL,
     TABLE_CATALOG NVARCHAR(50) NULL,
     TABLE_SCHEMA NVARCHAR(50) NULL,
     TABLE_NAME NVARCHAR(50) NULL,
     CONSTRAINT_TYPE NVARCHAR(50) NULL,
     IS_DEFERRABLE NVARCHAR(5) NULL,
     INITIALLY_DEFERRED NVARCHAR(5) NULL
    );
    GO

    CREATE TABLE dbo.IS_DB2_Schemas
    (
     CATALOG_NAME NVARCHAR(50) NULL,
     SCHEMA_NAME NVARCHAR(50) NULL,
     SCHEMA_OWNER NVARCHAR(50) NULL,
     DEFAULT_CHARACTER_SET_CATALOG NVARCHAR(50) NULL,
     DEFAULT_CHARACTER_SET_SCHEMA NVARCHAR(50) NULL,
     DEFAULT_CHARACTER_SET_NAME NVARCHAR(50) NULL,
     SQL_PATH VARCHAR(MAX) NULL
    );
    GO

    CREATE TABLE dbo.IS_DB2_Columns
    (
     TABLE_CATALOG NVARCHAR(50) NULL,
     TABLE_SCHEMA NVARCHAR(50) NULL,
     TABLE_NAME NVARCHAR(50) NULL,
     COLUMN_NAME NVARCHAR(50) NULL,
     ORDINAL_POSITION INT NULL,
     COLUMN_DEFAULT NVARCHAR(2002) NULL,
     IS_NULLABLE NVARCHAR(5) NULL,
     DATA_TYPE NVARCHAR(50) NULL,
     CHARACTER_MAXIMUM_LENGTH INT NULL,
     CHARACTER_OCTET_LENGTH INT NULL,
     NUMERIC_PRECISION INT NULL,
     NUMERIC_PRECISION_RADIX INT NULL,
     NUMERIC_SCALE INT NULL,
     DATETIME_PRECISION INT NULL,
     INTERVAL_TYPE NVARCHAR(50) NULL,
     INTERVAL_PRECISION INT NULL,
     CHARACTER_SET_CATALOG NVARCHAR(50) NULL,
     CHARACTER_SET_SCHEMA NVARCHAR(50) NULL,
     CHARACTER_SET_NAME NVARCHAR(50) NULL
    );
    GO
    ```

2.  创建一个新的 SSIS 包。在 Control Flow 选项卡上添加一个 Data Flow Task。双击打开此包并切换到 Data Flow 选项卡。
3.  为目标（SQL Server）数据库添加一个 OLEDB 连接管理器。
4.  在 Providers 列表中添加一个 ADO.NET 连接管理器。展开 .NET Providers for OleDb。选择 “.NET Providers for OleDb\IBM DB2 for i5/OS IBMDA400 OLE DB Provider”（或您正在使用的提供程序）。输入完全限定的服务器名称、用户名和密码。选择 Initial Catalog。
5.  添加一个 Execute SQL Task。将其配置为使用目标（SQL Server OLEDB）连接管理器。将 SQL Statement 设置为：

    ```sql
    TRUNCATE TABLE IS_DB2_Tables;
    TRUNCATE TABLE IS_DB2_Columns;
    TRUNCATE TABLE IS_DB2_Views;
    TRUNCATE TABLE IS_DB2_Schemas;
    TRUNCATE TABLE IS_DB2_TableConstraints;
    ```

6.  添加一个 Sequence Container，将 Execute SQL Task 连接到它，并在其中添加五个 Data Flow Task。将它们命名为 `Tables`、`Views`、`Columns`、`Schemas` 和 `Table Constraints`。将每个任务设置为使用 DB2 连接管理器，并将数据访问模式设置为 SQL Command。


