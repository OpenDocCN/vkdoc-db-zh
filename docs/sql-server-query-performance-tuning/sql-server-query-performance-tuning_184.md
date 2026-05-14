# 第 23 章 ■ 内存优化的 OLTP 表与存储过程

在下一节中，您还会注意到我不得不注释掉两列，`SpatialLocation`和`rowguid`。它们使用的数据类型在内存表中不可用。最后，`WITH`语句通过定义`MEMORY_OPTIMIZED=ON`来告知 SQL Server 将此表放置在何处。您可以通过修改`WITH`子句将`DURABILITY=SCHEMA_ONLY`，从而创建一个更快的表。这允许数据丢失，但使表速度更快，因为没有任何内容会被写入磁盘。

有许多不支持的数据类型可能会阻止您利用内存表。
*   `XML`
*   `ROWVERSION`
*   `SQL_VARIANT`
*   `HIERARCHYID`
*   `DATETIMEOFFSET`
*   `GEOGRAPHY`/`GEOMETRY`
*   用户定义的数据类型
*   LOB，包括`text`和`ntext`，以及所有的`varchar`和`binary`的`MAX`类型

除了数据类型，您还会遇到其他限制。我将在“内存索引”部分讨论索引要求。您无法创建指向内存表的外键引用。这意味着所有引用完整性都必须来自应用程序的编码端。

一旦表在内存中创建，您就可以像访问普通表一样访问它。如果我现在对它运行一个查询，它不会返回任何行，但它会正常运行。
```sql
SELECT a.AddressID
FROM dbo.Address AS a
WHERE a.AddressID = 42;
```
因此，为了在数据库中试验一些实际数据，请继续将 AdventureWorks 中存储在 `Person.Address` 中的信息加载到这个新数据库中存储在内存里的新表中。
```sql
CREATE TABLE dbo.AddressStaging(
    AddressLine1 nvarchar(60) NOT NULL,
    AddressLine2 nvarchar(60) NULL,
    City nvarchar(30) NOT NULL,
    StateProvinceID int NOT NULL,
    PostalCode nvarchar(15) NOT NULL
);
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

![](img/index-483_1.jpg)

```sql
INSERT dbo.AddressStaging
    (AddressLine1,
     AddressLine2,
     City,
     StateProvinceID,
     PostalCode
    )
SELECT a.AddressLine1,
       a.AddressLine2,
       a.City,
       a.StateProvinceID,
       a.PostalCode
FROM AdventureWorks2012.Person.Address AS a;

INSERT dbo.Address
    (AddressLine1,
     AddressLine2,
     City,
     StateProvinceID,
     PostalCode
    )
SELECT a.AddressLine1,
       a.AddressLine2,
       a.City,
       a.StateProvinceID,
       a.PostalCode
FROM dbo.AddressStaging AS a;

DROP TABLE dbo.AddressStaging;
```
您不能在跨数据库查询中组合使用内存表，因此我必须先将 19,000 行数据加载到一个临时表中，然后再将它们加载到内存表中。这并非为了展示性能示例，但值得注意的是，在我的系统上，将数据插入标准表用了近 850 毫秒，而将相同数据加载到内存表中只用了 2 毫秒。

但是，数据就位后，我可以重新运行查询并实际看到结果，如图 23-1 所示。

***图 23-1.** 第一个来自内存表的查询结果*

诚然，这并不那么令人兴奋。因此，为了有一些有意义的内容可供操作，我将创建其他几个表，以便您可以展示更多的查询行为。
```sql
CREATE TABLE dbo.StateProvince(
    StateProvinceID int IDENTITY(1,1) NOT NULL PRIMARY KEY NONCLUSTERED HASH WITH (BUCKET_COUNT=10000),
    StateProvinceCode nchar(3) COLLATE Latin1_General_100_BIN2 NOT NULL,
    CountryRegionCode nvarchar(3) NOT NULL,
    Name VARCHAR(50) NOT NULL,
    TerritoryID int NOT NULL,
    ModifiedDate datetime NOT NULL CONSTRAINT DF_StateProvince_ModifiedDate DEFAULT (getdate())
) WITH (MEMORY_OPTIMIZED=ON);

CREATE TABLE dbo.CountryRegion(
    CountryRegionCode nvarchar(3) NOT NULL,
    Name VARCHAR(50) NOT NULL,
    ModifiedDate datetime NOT NULL CONSTRAINT DF_CountryRegion_ModifiedDate DEFAULT (getdate()),
    CONSTRAINT PK_CountryRegion_CountryRegionCode PRIMARY KEY CLUSTERED
    (
        CountryRegionCode ASC
    )
);
```
这是一个额外的内存优化表和一个标准表。我也会向其中加载数据，以便您可以进行更有趣的查询。
```sql
SELECT sp.StateProvinceCode,
       sp.CountryRegionCode,
       sp.Name,
       sp.TerritoryID
```


