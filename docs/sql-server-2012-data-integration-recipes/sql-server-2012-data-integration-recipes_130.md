# 10-5. 运行并记录完整的数据剖析

## 问题

你希望对源表进行一次完整的数据剖析，并将使用 10-1 至 10-4 节中描述的基于 T-SQL 的技术收集到的剖析结果存储起来。

## 解决方案

设计一个 T-SQL 脚本来剖析所有你认为至关重要的信息，并将输出写入一个 SQL Server 表。然后，你就可以运行剖析 T-SQL 并存储返回的剖析数据。下面是一个示例。

1.  使用以下 DDL 创建一个剖析数据日志表 (`C:\SQL2012DIRecipes\CH10\tblDataProfiling.Sql`)：
    ```sql
    CREATE TABLE CarSales_Staging.dbo.DataProfiling
    (
     ID int IDENTITY(1,1) NOT NULL,
     DateExecuted DATETIME NULL CONSTRAINT DF_Log_DataProfiling_DateExecuted
                            DEFAULT (getdate()),
     DataSourceObject varchar(250) NULL,
     DataSourceColumn varchar(250) NULL,
     ProfileName varchar(50) NULL,
     ProfileResult numeric(22, 4) NULL
    ) ;
    GO
    ```
2.  运行以下 T-SQL 来捕获并存储剖析数据 (`C:\SQL2012DIRecipes\CH10\LogProfileData.Sql`)：
    ```sql
    DECLARE @Rowcount               INT;
    DECLARE @Model_NULL              INT;
    DECLARE @Mileage_NULL            INT
    DECLARE @Model_PERCENTNULL       NUMERIC(8,4);
    DECLARE @Mileage_PERCENTNULL     NUMERIC(8,4);
    DECLARE @Model_MAXLENGTH         INT;
    DECLARE @Model_MINLENGTH         INT;
    DECLARE @Mileage_MAX             INT;
    DECLARE @Mileage_MIN             INT;

    SELECT    @Rowcount = COUNT(*),
              @Mileage_MAX = MAX(Mileage),
              @Mileage_MIN = MIN(Mileage),
              @Model_MAXLENGTH = MAX(LEN(Model)),
              @Model_MINLENGTH = MIN(LEN(Model))
    FROM      CarSales.dbo.Stock;

    SELECT   @Model_NULL = COUNT(*)
    FROM     CarSales.dbo.Stock
    WHERE    Model IS NULL;
    SELECT   @Mileage_NULL = COUNT(*)
    FROM     CarSales.dbo.Stock
    WHERE    Mileage IS NULL;

    SET        @Model_PERCENTNULL =
                 CAST(@Model_NULL AS NUMERIC(5,2)) / CAST(@Rowcount AS NUMERIC(5,2));
    SET        @Mileage_PERCENTNULL =
                 CAST(@Mileage_NULL AS NUMERIC(5,2)) / CAST(@Rowcount AS NUMERIC(5,2));

    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'ALL', 'NbRows', @Rowcount, 100);
    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'Mileage', 'MaxMileage', @Mileage_MAX, 100);
    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'Mileage', 'MinMileage', @Mileage_MIN, 100);
    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'Model', 'MaxLn_Model', @Model_MAXLENGTH, 100);
    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'Model', 'MinLn_Model', @Model_MINLENGTH, 100);
    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'Model', 'NbNulls_Model', @Model_NULL, 100);
    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'Mileage', 'NbNulls_Mileage', @Mileage_NULL, 100);
    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'Model', 'PctNulls_Model', @Model_PERCENTNULL, 100);
    INSERT INTO CarSales_Staging.dbo.DataProfiling (DataSourceObject, DataSourceColumn, ProfileName, ProfileResult)
    VALUES ('CarSales_Staging.dbo.Stock', 'Mileage', 'PctNulls_Mileage', @Mileage_PERCENTNULL, 100);
    ```

## 工作原理

收集剖析数据只是整个过程的一部分。你很可能希望做以下一项或两项事情：

*   将数据存储在数据表中（例如，为确保满足 SLA（服务水平协议）或跟踪数据剖析随时间的变化）。
*   利用对存储的剖析数据的分析来阻止或允许 ETL 过程运行。

因此，这里为你提供了一个如何使用你所收集数据的实际示例。


