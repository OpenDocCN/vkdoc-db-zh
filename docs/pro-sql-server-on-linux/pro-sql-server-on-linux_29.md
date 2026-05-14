# 第三章 - 构建数据库与 T-SQL 基础

创建架构的语法可以在示例脚本 `` createschemas.sql `` 中找到。

执行此脚本或以下 T-SQL 语句来创建所有架构：

```sql
USE [WideWorldImporters]
GO

IF EXISTS (select * from sys.schemas where name = 'Application')
DROP SCHEMA [Application]
GO

CREATE SCHEMA [Application]
GO

IF EXISTS (select * from sys.schemas where name = 'Sales')
DROP SCHEMA [Sales]
GO

CREATE SCHEMA [Sales]
GO

IF EXISTS (select * from sys.schemas where name = 'Sequences')
DROP SCHEMA [Sequences]
GO

CREATE SCHEMA [Sequences]
GO
```

##### 创建序列

序列对象提供了一种能力，可以在表中的数据每次需要时生成唯一的键值。SQL Server 中另一个类似的功能称为标识列。序列对象独立于表创建，并且具有一些有益于性能的缓存特性。序列对象也用于其他数据库引擎，使它们成为一个很好的可移植特性。

虽然序列对象可以跨表共享，但我会为将要在此数据库中创建的两个表分别专用一个序列对象（一个用于 People 表，一个用于 Customers 表）。执行 `` createsequences.sql `` 示例脚本或以下 T-SQL 代码来创建序列对象：

```sql
USE [WideWorldImporters]
GO

DROP SEQUENCE IF EXISTS [Sequences].[PersonID]
GO

CREATE SEQUENCE [Sequences].[PersonID]
AS [int]
START WITH 1
INCREMENT BY 1
MINVALUE 0
MAXVALUE 2147483647
CACHE
GO

DROP SEQUENCE IF EXISTS [Sequences].[CustomerID]
GO

CREATE SEQUENCE [Sequences].[CustomerID]
AS [int]
START WITH 0
INCREMENT BY 1
MINVALUE 0
MAXVALUE 2147483647
CACHE
GO
```

`[PersonID]` 的 `SEQUENCE` 对象定义表明从值 1 开始，并允许最大值为 `int` 数据类型的最大值。`CACHE` 关键字有助于提高 `SEQUENCE` 值的性能。你可以在我们的文档中阅读更多关于 `SEQUENCE` 对象的信息：[`docs.microsoft.com/sql/t-sql/statements/create-sequence-transact-sql`](https://docs.microsoft.com/sql/t-sql/statements/create-sequence-transact-sql)。注意最小值是 0，但序列从 1 开始。这使我能够为 PersonID 列使用值 0，而无需使用 `SEQUENCE` 对象。`[CustomerID]` 的定义从值 0 开始。

在创建表的语句中，我将向你展示序列对象如何在向这两个表中的任何一个插入行时用于提供唯一的键值。

##### 最终创建表

现在我已经创建了架构和序列，我准备好创建我的两个表了。执行以下 T-SQL 语句或示例脚本 `` createpeople.sql `` 来创建 People 表：

```sql
USE [WideWorldImporters]
GO

-- [Application].[People] 表的表定义
--
DROP TABLE IF EXISTS [Application].[People]
GO

CREATE TABLE [Application].People NOT NULL,
    [PreferredName] nvarchar NOT NULL,
    --     [SearchName]  AS (concat([PreferredName],N' ',[FullName])) PERSISTED NOT NULL,
    [IsPermittedToLogon] [bit] NOT NULL,
    [LogonName] nvarchar NULL,
    [IsExternalLogonProvider] [bit] NOT NULL,
    [HashedPassword] varbinary NULL,
    [IsSystemUser] [bit] NOT NULL,
    [IsEmployee] [bit] NOT NULL,
    [IsSalesperson] [bit] NOT NULL,
    [UserPreferences] nvarchar NULL,
    [PhoneNumber] nvarchar NULL,
    [FaxNumber] nvarchar NULL,
    [EmailAddress] nvarchar NULL,
    [Photo] varbinary NULL,
    [CustomFields] nvarchar NULL,
    --    [OtherLanguages]  AS (json_query([CustomFields],N'$.OtherLanguages')),
    [LastEditedBy] [int] NOT NULL,
    --    [ValidFrom] datetime2 GENERATED ALWAYS AS ROW START NOT NULL,
    --    [ValidTo] datetime2 GENERATED ALWAYS AS ROW END NOT NULL
    CONSTRAINT [PK_Application_People] PRIMARY KEY CLUSTERED
    (
        [PersonID] ASC
    )
)
GO
```



