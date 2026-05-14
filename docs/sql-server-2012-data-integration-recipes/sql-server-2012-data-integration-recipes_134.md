# 使用变更跟踪进行增量数据同步

## SQL 实现代码

```sql
SYS_CHANGE_OPERATION = ''''D'''''')'

IF OBJECT_ID('tempdb..#ClientDeletes') IS NOT NULL DROP TABLE tempdb..#ClientDeletes
CREATE TABLE #ClientDeletes (ID INT)
INSERT INTO #ClientDeletes EXEC (@DeleteSQL)

-- 插入操作
DECLARE @InsertsSQL VARCHAR(8000) =
        'SELECT ID,ClientName,Country,Town,County,Address1,Address2,ClientType,ClientSize
          FROM OPENQUERY(R2,
                        ''SELECT SRC.ID,ClientName,Country,Town,County,Address1,Address2,
                          ClientType,ClientSize
                          FROM    Carsales.dbo.Client SRC
                          INNER JOIN  CHANGETABLE(
                                         CHANGES Carsales.dbo.Client, ' +
                                         CAST(@LAST_SYNC_VERSION
                                         AS VARCHAR(20)) + ') AS CT
                                  ON SRC.ID = CT.ID
                          WHERE       CT.SYS_CHANGE_OPERATION = ''''I'''''')'

IF OBJECT_ID('tempdb..#ClientInserts') IS NOT NULL DROP TABLE tempdb..#ClientInserts
CREATE TABLE #ClientInserts (
    ID INT NOT NULL,
    ClientName VARCHAR(150) NULL,
    Country VARCHAR(50) NULL,
    Town VARCHAR(50) NULL,
    County VARCHAR(50) NULL,
    Address1 VARCHAR(50) NULL,
    Address2 VARCHAR(50) NULL,
    ClientType VARCHAR(20) NULL,
    ClientSize VARCHAR(10) NULL
)
INSERT INTO #ClientInserts EXEC (@InsertsSQL)

-- 更新操作
DECLARE @UpdatesSQL VARCHAR(8000) =
                   'SELECT    ID,ClientName,Country,Town,County,Address1,Address2,ClientType,ClientSize
                    FROM      OPENQUERY(R2, ''SELECT       SRC.ID,ClientName,Country,Town,
                                                        County,Address1,Address2,
                                                        ClientType,ClientSize
                                              FROM         Carsales.dbo.Client SRC
                                              INNER JOIN   CHANGETABLE(CHANGES Carsales.dbo.Client,'
                                              + CAST(@LAST_SYNC_VERSION AS VARCHAR(20))
                                              + ')  AS CT
                                                        ON SRC.ID = CT.ID
                    WHERE   CT.SYS_CHANGE_OPERATION = ''''U'''''')' ;

IF OBJECT_ID('tempdb..#ClientUpdates') IS NOT NULL DROP TABLE tempdb..#ClientUpdates;
CREATE TABLE #ClientUpdates (
    ID INT NOT NULL,
    ClientName VARCHAR(150) NULL,
    Country VARCHAR(50) NULL,
    Town VARCHAR(50) NULL,
    County VARCHAR(50) NULL,
    Address1 VARCHAR(50) NULL,
    Address2 VARCHAR(50) NULL,
    ClientType VARCHAR(20) NULL,
    ClientSize VARCHAR(10) NULL
) ;

INSERT INTO   #ClientUpdates (ID,ClientName,Country,Town,County,Address1,Address2,ClientType,ClientSize)
EXEC (@UpdatesSQL) ;

-- 执行插入/更新/删除操作

-- 插入
INSERT INTO      CarSales_Staging.dbo.Client
                 (ID, ClientName, Country, Town, County, Address1, Address2,
                  ClientType, ClientSize)
SELECT           ID, ClientName, Country,Town, County, Address1, Address2, ClientType, ClientSize
FROM             #ClientInserts

-- 更新
UPDATE            DST
SET               DST.ClientName = UPD.ClientName
                    ,DST.Country = UPD.Country
                    ,DST.Town = UPD.Town
                    ,DST.County = UPD.County
                    ,DST.Address1 = UPD.Address1
                    ,DST.Address2 = UPD.Address2
                    ,DST.ClientType = UPD.ClientType
                    ,DST.ClientSize = UPD.ClientSize
FROM              CarSales_Staging.dbo.Client DST
INNER JOIN        #ClientUpdates UPD
                  ON DST.ID = UPD.ID

-- 删除
DELETE FROM  DST
FROM         CarSales_Staging.dbo.Client DST
INNER JOIN   #ClientDeletes DLT
             ON DLT.ID = DST.ID

-- 设置新的 LAST_SYNC_VERSION
EXEC R2.CarSales.sys.sp_updateextendedproperty
    @level0type = N'SCHEMA',
    @level0name = dbo,
    @level1type = N'TABLE',
    @level1name = Client,
    @name = LAST_SYNC_VERSION,
    @value = @CURRENT_VERSION ;
END
```

## 工作原理

你可以使用变更跟踪在链接服务器上执行“推送”式数据同步，但能否使用“拉取”方法呢？答案是一个有保留的“可以”。说有保留并非因为存在任何固有困难，而是因为你需要对开箱即用的变更跟踪函数（正如我们在配方 12-1 中看到的）的限制应用一系列解决方法。

那么潜在的问题是什么？

*   `CHANGETABLE` 函数无法通过链接（四部分命名）服务器工作。
*   `CHANGE_TRACKING_MIN_VALID_VERSION` 函数无法通过链接服务器工作。
*   将 `CHANGETABLE` 函数包装在源服务器上的表值用户定义函数中将不起作用，因为 UDF 在链接服务器上也有问题。
*   将 `CHANGETABLE` 函数包装在源服务器上的存储过程可能有效——但提取数据需要 MSDTC（分布式事务协调器）在源服务器上运行，因为它将创建一个隐式事务。此外，你需要为 RPC 配置链接服务器（你将针对远程服务器执行存储过程）。这两个限制对某些 DBA 来说可能是无法接受的——也是一个相当大的麻烦。因此，由于这些原因，我们将避免这种方法。

这就剩下 `OPENQUERY` 和动态 SQL 作为解决方案。它并不完美，但它允许我们返回所有所需的数据（最小更改版本、最大更改版本，以及所有与插入、更新和删除相关的数据），在服务器级别的配置方面没有任何麻烦。

## 提示、技巧和陷阱

*   如果你愿意，可以使用表变量代替临时表。对于较大的数据集，临时表可能更可取，因为它们可以被索引，这在有些情况下很有用。
*   这里也可以使用 `COLUMNPROPERTY` 函数进行特定列跟踪，如配方 12-1 的“工作原理”部分所述。

## 12-3. 将变更跟踪作为结构化 ETL 流程的一部分使用

### 问题

你想将变更跟踪作为结构化 ETL 流程的一部分使用。

### 解决方案

在 SSIS 包中使用变更跟踪信息，以确保源表中的数据修改被应用到目标表。由于这个包有点复杂，我建议你在开始创建之前先通读一遍，了解它的目标。话虽如此，以下步骤说明了如何操作。

1.  运行配方 12-1 中的步骤 1 到 4，在源服务器（R2）上启用变更跟踪，并将目标表放在本地服务器上（当然，除非你已经这样做了）。
2.  创建一个新的 SSIS 包，并添加以下三个连接管理器：
    *   `Source_OLEDB`: 到源数据库的 OLEDB 连接。
    *   `Source_ADONET`: 到源数据库的 ADO.NET 连接。
    *   `Destination_OLEDB`: 到目标数据库的 OLEDB 连接。将 `RetainSameConnection` 属性设置为 `False`。
3.  添加以下包作用域变量：
    | 变量名 | 类型 | 值 |
    | --- | --- | --- |
    | `CurrentVersion` | `Int64` | 0 |
    | `DeleteTable` | `String` | TMP_Deletes |
    | `LastSynchVersion` | `Int64` | 0 |
    | `MinValidVersion` | `Int64` | 0 |
    | `SQLDelete` | `String` | `SELECT ID FROM dbo.Client` |
    | `SQLInsert` | `String` | `SELECT ID, ClientName, Country, Town, County, Address1, Address2, ClientType, ClientSize FROM dbo.Client` |
    | `SQLUpdate` | `String` | `SELECT ID, ClientName, Country, Town, County, Address1, Address2, ClientType, ClientSize FROM dbo.Client` |
    | `UpdateTable` | `String` | TMP_Updates |



