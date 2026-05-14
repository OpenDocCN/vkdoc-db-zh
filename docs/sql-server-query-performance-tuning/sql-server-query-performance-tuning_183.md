# 第 23 章：内存优化的 OLTP 表与存储过程

除了硬件要求外，您还必须对数据库进行额外工作才能启用内存表。我将从一个新数据库开始进行说明。

```sql
CREATE DATABASE InMemoryTest ON PRIMARY
(NAME = N'InMemoryTest_Data',
FILENAME = N'D:\Data\InMemoryTest_Data.mdf',
SIZE = 5GB)
LOG ON
(NAME = N'InMemoryTest_Log',
FILENAME = N'L:\Log\InMemoryTest_Log.ldf');
```

为了使内存表保持持久性，它们必须既写入内存也写入磁盘，因为内存会随着断电而消失。持久性（关系数据集的 ACID 属性之一）意味着一旦事务提交，它就保持提交状态。您可以拥有一个**持久**的内存表，或者一个**非持久**的表。对于非持久表，即使您可能已提交事务，但仍可能丢失该数据，这与 SQL Server 中标准表的工作方式不同。非持久数据最常见的使用场景是会话状态或时间敏感的信息，例如电子购物车。总之，内存存储与标准关系表中的常规存储不同。因此，必须创建单独的文件组和文件。为此，您可以直接修改数据库，如下所示：

```sql
ALTER DATABASE InMemoryTest
ADD FILEGROUP InMemoryTest_InMemoryData
CONTAINS MEMORY_OPTIMIZED_DATA;

ALTER DATABASE InMemoryTest
ADD FILE (NAME='InMemoryTest_InMemoryData',
FILENAME ='D:\Data\InMemoryTest_InMemoryData.ndf')
TO FILEGROUP InMemoryTest_InMemoryData;
```

我本可以直接修改您一直在试验的 `AdventureWorks2012` 数据库，但使用内存优化表时另一个需要考虑的因素是，一旦创建了特殊文件组，就无法将其移除。您只能删除整个数据库。这就是为什么我仅使用一个单独的数据库进行试验。这样更安全。

使用内存 OLTP 的数据库在功能上有一些限制。

*   `DBCC CHECKDB`：您可以运行一致性检查，但会跳过内存优化表。如果您尝试运行 `DBCC CHECKTABLE`，将会收到错误。
*   `AUTO_CLOSE`：不支持此功能。
*   `DATABASE SNAPSHOT`：不支持此功能。
*   `ATTACH_REBUILD_LOG`：同样不支持此功能。

完成这些修改后，您现在可以开始在系统中创建内存表了。

### 创建表

数据库设置完成后，您现在就能够创建如前所述的内存优化表了。实际语法相当直接。我将尽可能复制 `AdventureWorks2012` 中的 `Person.Address` 表。

```sql
USE DATABASE InMemoryTest;
GO

CREATE TABLE dbo.Address
(
AddressID INT IDENTITY(1, 1)
NOT NULL
PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT = 50000),
AddressLine1 NVARCHAR(60) NOT NULL,
AddressLine2 NVARCHAR(60) NULL,
City NVARCHAR(30) NOT NULL,
StateProvinceID INT NOT NULL,
PostalCode NVARCHAR(15) NOT NULL,
--[SpatialLocation geography NULL,
--rowguid uniqueidentifier ROWGUIDCOL NOT NULL CONSTRAINT DF_Address_rowguid DEFAULT (newid()),
ModifiedDate DATETIME
NOT NULL
CONSTRAINT DF_Address_ModifiedDate DEFAULT (GETDATE())
)
WITH (
MEMORY_OPTIMIZED= ON,
DURABILITY = SCHEMA_AND_DATA);
```

这将在系统内存中创建一个持久表，使用您定义的磁盘空间来保留数据的持久副本，确保在断电情况下不会丢失数据。它有一个像常规 SQL Server 表一样的 `IDENTITY` 值作为主键（尽管在此版本的 SQL Server 中，若要使用 `IDENTITY` 而非 `SEQUENCE`，您将不得不放弃将定义设置为除 `(1,1)` 以外任何值的能力）。但是，索引定义不是聚集索引，而是 `NON-CLUSTERED HASH`。我接下来将讨论索引以及 `BUCKET_COUNT` 等内容。


