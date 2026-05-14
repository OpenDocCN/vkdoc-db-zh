# 10-4. 更快地配置外部数据

## 问题
您希望在最短时间内配置来自 SQL Server 以外来源的数据。

## 解决方案
使用 T-SQL 和 `OPENROWSET`，同时最小化读取数据集的次数。一种方法是使用临时表，如下面的示例所示 (`C:\SQL2012DIRecipes\CH10\Stock.Txt`)：
```sql
DECLARE @Cost_Price             INT
DECLARE @Registration_Year      INT
DECLARE @ROWCOUNT               INT
DECLARE @Mileage_MAX            INT
DECLARE @Mileage_MIN            INT
DECLARE @Registration_Year_NULL INT
DECLARE @Cost_Price_NULL        INT

SELECT
    CASE WHEN Registration_Year IS NULL THEN 1 ELSE 0 END AS Registration_Year,
    CASE WHEN Cost_Price IS NULL THEN 1 ELSE 0 END AS Cost_Price
INTO   #NullSourceRecords
FROM   OPENROWSET('MSDASQL',
           'Driver={Microsoft Access Text Driver (*.txt, *.csv)};
            DefaultDir= C:\SQL2012DIRecipes\CH10;',
           'select Registration_Year, Cost_Price from Stock.txt')
WHERE  Registration_Year IS NULL OR Cost_Price IS NULL

SELECT
    @ROWCOUNT = COUNT(*),
    @Mileage_MAX = MAX(Mileage),
    @Mileage_MIN = MIN(Mileage)
FROM OPENROWSET('MSDASQL',
    'Driver={Microsoft Access Text Driver (*.txt, *.csv)};
     DefaultDir= C:\SQL2012DIRecipes\CH10;',
    'select Registration_Year, Cost_Price from Stock.')
```

