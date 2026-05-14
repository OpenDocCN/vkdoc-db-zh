# 12-2. 使用变更跟踪将更改拉取到目标表中

## 问题

使用变更跟踪，你希望将检测到的数据变更“拉取”而非“推送”到目标表中。

## 解决方案

应用“拉取”差异数据使用变更跟踪。以下 T-SQL 可以实现这一点——其中源数据位于名为 `R2` 的链接服务器上。

1.  运行方案 12-1 中的步骤 1 到 4，在源服务器 (`R2`) 上启用变更跟踪，并将目标表放置在本地服务器上。
2.  运行以下 T-SQL 以将数据从远程服务器同步到本地服务器 (`C:\SQL2012DIRecipes\CH11\PullChangeTracking.Sql`)：

    ```sql
    DECLARE @LAST_SYNC_VERSION BIGINT
    DECLARE @CURRENT_VERSION BIGINT
    DECLARE @MIN_VALID_VERSION BIGINT

    -- To get the LAST_SYNC_VERSION
    SELECT       @LAST_SYNC_VERSION = CAST(SEP.value AS BIGINT)
    FROM         R2.CarSales.sys.tables TBL
                 INNER JOIN   R2.CarSales.sys.schemas SCH
                 ON TBL.schema_id = SCH.schema_id
                 INNER JOIN   R2.CarSales.sys.extended_properties SEP
                 ON TBL.object_id = SEP.major_id
    WHERE        SCH.name = 'dbo'
    AND          TBL.name = 'client'
    AND          SEP.name = 'LAST_SYNC_VERSION'

    -- Gets maximum version in CHANGETABLE - so available for updating (use instead of CHANGE_TRACKING_CURRENT_VERSION)
    SELECT @CURRENT_VERSION = MaxValidVersion
    FROM OPENQUERY(R2, '
    SELECT CASE WHEN MAX(SYS_CHANGE_CREATION_VERSION) > MAX(SYS_CHANGE_VERSION)
                THEN MAX(SYS_CHANGE_CREATION_VERSION)
                ELSE MAX(SYS_CHANGE_VERSION)
           END AS MaxValidVersion
    FROM   CHANGETABLE(CHANGES Carsales.dbo.Client, 0) AS CT
    ')

    -- Gets minimum version This one works over a linked server!
    SELECT Min_Valid_Version
    FROM R2.CarSales.sys.change_tracking_tables

    IF @LAST_SYNC_VERSION >= @MIN_VALID_VERSION
    BEGIN
    -- Get all data for INSERTS/UPDATES/DELETES into temp tables
    -- Deletes
    DECLARE @DeleteSQL VARCHAR(8000) =
    'SELECT ID
    FROM OPENQUERY(R2, ''SELECT        ID
                       FROM       CHANGETABLE(CHANGES Carsales.dbo.Client,' +
                       CAST(@LAST_SYNC_VERSION AS VARCHAR(20)) + ') AS CT
                       WHERE       CT.
    ```



