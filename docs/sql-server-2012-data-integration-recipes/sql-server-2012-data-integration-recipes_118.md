# 使用 T-SQL 处理缓慢变化维度

## 1. 处理类型 1 缓慢变化维度

运行以下 T-SQL 代码片段（来自 `CarSales_Staging` 数据库，文件路径：`C:\SQL2012DIRecipes\CH09\SCD1.sql`）：

```sql
USE CarSales_Staging;
GO

MERGE CarSales_Staging.dbo.Client_SCD1 AS DST
USING CarSales.dbo.Client AS SRC
ON (SRC.ID = DST.BusinessKey)
WHEN NOT MATCHED THEN
    INSERT (BusinessKey, ClientName, Country, Town, County, Address1, Address2, ClientType, ClientSize)
    VALUES (SRC.ID, SRC.ClientName, SRC.Country, SRC.Town, SRC.County, SRC.Address1, SRC.Address2, SRC.ClientType, SRC.ClientSize)
WHEN MATCHED
AND (
    ISNULL(DST.ClientName,'') <> ISNULL(SRC.ClientName,'')
    OR ISNULL(DST.Country,'') <> ISNULL(SRC.Country,'')
    OR ISNULL(DST.Town,'') <> ISNULL(SRC.Town,'')
    OR ISNULL(DST.Address1,'') <> ISNULL(SRC.Address1,'')
    OR ISNULL(DST.Address2,'') <> ISNULL(SRC.Address2,'')
    OR ISNULL(DST.ClientType,'') <> ISNULL(SRC.ClientType,'')
    OR ISNULL(DST.ClientSize,'') <> ISNULL(SRC.ClientSize,'')
)
THEN UPDATE
SET
    DST.ClientName = SRC.ClientName
    ,DST.Country = SRC.Country
    ,DST.Town = SRC.Town
    ,DST.Address1 = SRC.Address1
    ,DST.Address2 = SRC.Address2
    ,DST.ClientType = SRC.ClientType
    ,DST.ClientSize = SRC.ClientSize
;
```

### 工作原理

自从 SQL Server 2008 引入 `MERGE` 命令后，就可以使用单个命令执行 `UPSERT` 操作——即插入、删除和更新。同样值得注意的是，处理缓慢变化维度的更新插入技术并不仅限于数据仓库领域。类型 1 SCD（缓慢变化维度）本质上就是一个 `UPSERT`，因此在无限多的场景中都很有用。

在所有 SCD 类型中，类型 1 缓慢变化维度是最容易处理的，因为它只是对现有数据进行简单的就地更新，不尝试跟踪变化的历史。

使用的 SQL 代码片段将基于业务键映射两个表。它还会执行以下操作：

*   如果业务键不存在于目标表中（`WHEN NOT MATCHED`），则向目标表插入一条新记录，并添加一个自增代理键。
*   如果源表和目标表之间的任何字段不同（`WHEN MATCHED AND...`），则更新其他字段。

这里，我以 `CarSales` 数据库中的 `dbo.Client` 表作为源数据，并在 `CarSales_Staging.dbo.Client` 表中更新经过适当转换的数据。此示例假设业务键是唯一的主键。我们假设不会使用 `WHEN NOT MATCHED` 来删除维度数据，而只关注 `UPSERT` 数据——即插入和更新维度数据。

#### 提示、技巧与陷阱

*   确保所有在 `WHEN MATCHED ... AND` 子句中用于比较数据的列，也出现在 `THEN UPDATE SET` 子句中。
*   在此示例中，我对位于同一服务器上的数据库执行更新插入。此技术可应用于同一数据库内的表或跨链接服务器。

## 2. 处理类型 2 缓慢变化维度

### 问题

当源数据在目标表中添加、插入和更新时，你需要随时间跟踪这些变化。

### 解决方案

使用 T-SQL 的 `MERGE` 命令执行作为类型 2 缓慢变化维度的 `UPSERT` 操作，从而随时间跟踪变化，如下所示。

1.  使用以下 DDL 创建一个表来保存类型 2 SCD 数据（文件路径：`C:\SQL2012DIRecipes\CH09\tblClient_SCD2.sql`）：

    ```sql
    CREATE TABLE CarSales_Staging.dbo.Client_SCD2
    (
     ClientID INT IDENTITY(1,1) NOT NULL,
     BusinessKey INT NOT NULL,
     ClientName VARCHAR(150) NULL,
     Country VARCHAR(50) NULL,
     Town VARCHAR(50) NULL,
     County VARCHAR(50) NULL,
     Address1 VARCHAR(50) NULL,
     Address2 VARCHAR(50) NULL,
     ClientType VARCHAR(20) NULL,
     ClientSize VARCHAR(10) NULL,
     ValidFrom INT NULL,
     ValidTo INT NULL,
     IsCurrent BIT NULL
    ) ;
    GO
    ```

2.  运行以下代码以执行类型 2 SCD 更新插入（文件路径：`C:\SQL2012DIRecipes\CH09\SCD2.sql`）：

    ```sql
    USE CarSales_Staging;
    GO

    -- 定义有效性使用的日期 - 假设为完整的 24 小时周期
    DECLARE @Yesterday INT = CAST(CAST(YEAR(DATEADD(dd,-1,GETDATE())) AS CHAR(4)) + RIGHT('0' + CAST(MONTH(DATEADD(dd,-1,GETDATE()))
    AS VARCHAR(2)),2) + RIGHT('0' + CAST(DAY(DATEADD(dd,-1,GETDATE())) AS VARCHAR(2)),2) AS INT)
    DECLARE @Today INT = CAST(CAST(YEAR(GETDATE()) AS CHAR(4)) + RIGHT('0' + CAST(MONTH(GETDATE())
    AS VARCHAR(2)),2) + RIGHT('0' + CAST(DAY(GETDATE()) AS VARCHAR(2)),2) AS INT);

    -- 用于对现有维度记录进行最新更新的插入语句
    INSERT INTO dbo.Client_SCD2 (BusinessKey, ClientName, Country, Town, Address1, Address2, ClientType, ClientSize, ValidFrom, IsCurrent)
    SELECT ID, ClientName, Country, Town, Address1, Address2, ClientType, ClientSize, @Today, 1
    FROM
    (
    -- 合并语句
    MERGE CarSales_Staging.dbo.Client_SCD2 AS DST
    USING CarSales.dbo.Client AS SRC
    ON (SRC.ID = DST.BusinessKey)
    WHEN NOT MATCHED THEN
        INSERT (BusinessKey, ClientName, Country, Town, County, Address1, Address2, ClientType, ClientSize, ValidFrom, IsCurrent)
        VALUES (SRC.ID, SRC.ClientName, SRC.Country, SRC.Town, SRC.County, SRC.Address1, SRC.Address2, SRC.ClientType, SRC.ClientSize, @Today, 1)
    WHEN MATCHED
    AND (
        ISNULL(DST.ClientName,'') <> ISNULL(SRC.ClientName,'')
        OR ISNULL(DST.Country,'') <> ISNULL(SRC.Country,'')
        OR ISNULL(DST.Town,'') <> ISNULL(SRC.Town,'')
        OR ISNULL(DST.Address1,'') <> ISNULL(SRC.Address1,'')
        OR ISNULL(DST.Address2,'') <> ISNULL(SRC.Address2,'')
        OR ISNULL(DST.ClientType,'') <> ISNULL(SRC.ClientType,'')
        OR ISNULL(DST.ClientSize,'') <> ISNULL(SRC.ClientSize,'')
    )
    -- 用于已更改维度记录的更新语句，标记为不再活跃
    THEN UPDATE
    SET DST.IsCurrent = 0, DST.ValidTo = @Yesterday
    OUTPUT SRC.ID, SRC.ClientName, SRC.Country, SRC.Town, SRC.Address1, SRC.Address2, SRC.ClientType, SRC.ClientSize, $Action AS MergeAction
    ) AS MRG
    WHERE MRG.MergeAction = 'UPDATE'
    ;
    ```

目标表将添加新记录，并将更改的记录添加并标记为最新版本。

### 工作原理

这里我们研究一种技术，当需要在 SQL Server 表中更新数据并维护更改历史记录时，该技术非常强大。由于我们处理的是 SCD（缓慢变化维度），我将假设不会使用 `WHEN NOT MATCHED` 来删除维度数据，而只关注 `UPSERT` 数据——即插入和更新维度数据。类型 2 SCD 是一种使用表中的历史记录进行变更跟踪的技术，同时指示有效日期范围和当前数据。这在数据仓库环境之外也有很多用途。

为了说明这一点，我以 `CarSales` 数据库中的 `dbo.Client` 表作为源数据，并将这些数据经过适当转换后导出到 `CarSales_Staging` 数据库。此示例假设业务键是唯一的主键。

处理类型 2 SCD 仅比类型 1 稍微复杂一些。




