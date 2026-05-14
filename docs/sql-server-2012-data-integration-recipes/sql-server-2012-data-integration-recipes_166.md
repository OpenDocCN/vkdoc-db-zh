# 数据库示例与目录结构

## CarSales_DW 数据库

创建 `CarSales_DW` 数据库的代码位于 `C:\SQLDIRecipes\Databases\CarSales_DWDatabaseCreation.Sql`。

```sql
USE master
GO

IF db_id('CarSales_DW') IS NOT NULL DROP DATABASE CarSales_DW';
GO

CREATE DATABASE CarSales_DW
 CONTAINMENT = NONE
 ON PRIMARY ( NAME = N'CarSales_DW', FILENAME = N'C:\SQLDIRecipes\Databases\CarSales_DW.mdf' , SIZE = 33792KB , MAXSIZE = UNLIMITED, FILEGROWTH = 1024KB )
 LOG ON ( NAME = N'CarSales_DW_log', FILENAME = N'C:\SQLDIRecipes\Databases\CarSales_DW_log.ldf' , SIZE = 4224KB , MAXSIZE = 2048GB , FILEGROWTH = 10%)
GO
ALTER DATABASE CarSales_DW SET COMPATIBILITY_LEVEL = 110
GO
ALTER DATABASE CarSales_DW SET QUOTED_IDENTIFIER OFF
GO
ALTER DATABASE CarSales_DW SET RECOVERY SIMPLE
GO
ALTER DATABASE CarSales_DW SET READ_WRITE
GO
```

```sql
USE CarSales_DW
GO

CREATE TABLE dbo.Fact_Sales (
 SalesID int IDENTITY(1,1) NOT NULL,
 Sale_Price numeric(18, 2) NULL,
 InvoiceDate datetime NULL,
 Total_Discount numeric(18, 2) NULL,
 Delivery_Cost numeric(18, 2) NULL,
 Cost_Price decimal(18, 2) NULL,
 Make_X nvarchar(50) NULL,
 Marque_X nvarchar(50) NULL,
 Model_X nvarchar(50) NULL,
 Colour_X nvarchar(50) NULL,
 Product_Type_X nvarchar(50) NULL,
 Vehicle_Type_X nvarchar(50) NULL,
 ClientName_X nvarchar(150) NULL,
 Country_X varchar(50) NULL,
 Town_X nvarchar(50) NULL,
 County_X nvarchar(50) NULL,
 Client_ID int NULL,
 Geography_ID int NULL,
 Product_ID int NULL
) ;
GO

CREATE TABLE dbo.Dim_Products (
 ID int IDENTITY(1,1) NOT NULL,
 Product_Type nvarchar(50) NULL,
 Make nvarchar(50) NULL,
 Marque nvarchar(50) NULL,
 Colour nvarchar(50) NULL,
 Vehicle_Type nvarchar(50) NULL,
 Model nvarchar(50) NULL,
 CONSTRAINT PK_Dim_Products PRIMARY KEY CLUSTERED (
 ID ASC )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON PRIMARY
) ;
GO

CREATE TABLE dbo.Dim_Geography (
 ID int IDENTITY(1,1) NOT NULL,
 Country nvarchar(50) NULL,
 State_County nvarchar(50) NULL,
 Town nvarchar(50) NULL,
 CONSTRAINT PK_Dim_Geography PRIMARY KEY CLUSTERED (
 ID ASC )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON PRIMARY
) ;
GO

CREATE TABLE dbo.Dim_Clients (
 ID int IDENTITY(1,1) NOT NULL,
 Client_Name nvarchar(150) NULL,
 Client_Size varchar(10) NULL,
 Client_Type varchar(10) NULL,
 CONSTRAINT PK_Dim_Clients PRIMARY KEY CLUSTERED (
 ID ASC )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON PRIMARY
) ;
GO
```

该数据库包含一个存储过程 (`dbo.pr_FillDW`)，该过程从 `CarSales` 数据库获取数据并用于填充 `CarSales_DW` 数据库。相关脚本位于 `C:\SQLDIRecipes\Databases\CarSales_FillDW.Sql`。

```sql
CREATE procedure dbo.pr_FillDW
AS

-- Geography
TRUNCATE TABLE dbo.Dim_Geography
INSERT INTO dbo.Dim_Geography ( Country ,State_County ,Town )
SELECT DISTINCT C.CountryName_EN ,L.County ,L.Town
FROM               CarSales_Book.dbo.Countries C
                   INNER JOIN     CarSales_Book.dbo.Client L
                   ON             C.CountryID = L.Country
-- Clients
TRUNCATE TABLE dbo.Dim_Clients
INSERT INTO dbo.Dim_Clients ( Client_Name ,Client_Size ,Client_Type )
SELECT DISTINCT    ClientName, ClientSize, ClientType
FROM               CarSales_Book.dbo.Client

-- Products
TRUNCATE TABLE dbo.Dim_Products
INSERT INTO dbo.Dim_Products ( Product_Type ,Make ,Marque ,Model ,Colour ,Vehicle_Type )
SELECT DISTINCT   S.Product_Type, S.Make, S.Marque, S.Model, C.Colour, S.Vehicle_Type
FROM              CarSales_Book.dbo.Colours C
                  INNER JOIN    CarSales_Book.dbo.Stock S
                  ON            C.ColourID = S.Colour
-- Fact table
TRUNCATE TABLE dbo.Fact_Sales
INSERT INTO dbo.Fact_Sales ( Sale_Price ,InvoiceDate ,Total_Discount ,Delivery_Cost ,Cost_Price ,Make_X ,Marque_X ,Model_X ,Colour_X ,Product_Type_X ,Vehicle_Type_X ,ClientName_X ,Country_X ,Town_X ,County_X )
SELECT DISTINCT L.SalePrice, I.InvoiceDate, I.TotalDiscount, I.DeliveryCharge, S.Cost_Price, S.Make, S.Marque, S.Model, C.Colour, S.Product_Type, S.Vehicle_Type, CL.ClientName, CR.CountryName_EN, CL.Town, CL.County
FROM          CarSales_Book.dbo.Colours C
              INNER JOIN    CarSales_Book.dbo.Stock S
              ON            C.ColourID = S.Colour
              INNER JOIN    CarSales_Book.dbo.Invoice_Lines L
              ON            S.ID = L.StockID
              INNER JOIN    CarSales_Book.dbo.Invoice I
              ON            L.InvoiceID = I.ID
              INNER JOIN    CarSales_Book.dbo.Client CL
              ON            I.ClientID = CL.ID
              INNER JOIN    CarSales_Book.dbo.Countries CR
              ON            CL.Country = CR.CountryID

-- Set GeographyID
UPDATE       F
SET          F.Geography_ID = G.ID
FROM         dbo.Fact_Sales F
             INNER JOIN   dbo.Dim_Geography G
             ON           F.Country_X = G.Country
             AND          F.County_X = G.State_County
             AND          F.Town_X = G.Town
-- Set ProductID
UPDATE F
SET            F.Product_ID = P.ID
FROM           dbo.Fact_Sales F
               INNER JOIN   dbo.Dim_Products P
               ON           F.Product_Type_X = P.Product_Type
               AND          F.Make_X = P.Make
               AND          F.Marque_X = P.Marque
               AND          F.Model_X = P.Model
               AND          F.Vehicle_Type_X = P.Vehicle_Type

-- Set Client ID
UPDATE         F
SET            F.Client_ID = C.ID
FROM           dbo.Fact_Sales F
               INNER JOIN   dbo.Dim_Clients C
               ON           F.ClientName_X = C.Client_Name
GO
```

## CarSales SSAS 多维数据集

该多维数据集在结构上与 `CarSales_DW` 数据库基本相同。它包含以下部分：

| Sales:     | 核心（也是唯一的）事实表。 |
|------------|----------------------------|
| Products:  | 产品维度。                 |
| Clients:   | 客户维度。                 |
| Geography: | 地理维度。                 |

由于此处的目标是允许数据导出，而不是解释多维数据集开发，因此没有时间维度。

## 还原 CarSales_Cube SSAS 数据库

可以使用脚本 `C:\SQLDIRecipes\Databases\CarSales_CubeDatabaseRestore.Mdx` 还原 `CarSales_Cube` 数据库。用于还原此 Analysis Services 数据库的 XMLA 脚本确实太长，无法在此处复制——但你可以在本书的网站上找到它。

## CarSales_Logging

`CarSales_Logging` 数据库是一个非常简单的数据库，用于保存第 15 章中使用的日志和审核表。它最初不包含任何表——你选择使用哪些表取决于你选择遵循的配方。创建该数据库的代码位于 `C:\SQLDIRecipes\Databases\CarSales_LoggingDatabaseCreation.Sql`：

```sql
USE master
GO

IF db_id('CarSales_Logging') IS NOT NULL DROP DATABASE CarSales_Logging';
GO

CREATE DATABASE CarSales_Logging
 CONTAINMENT = NONE
 ON PRIMARY ( NAME = N'CarSales_Logging', FILENAME = N'C:\SQLDIRecipes\Databases\CarSales_Logging.mdf' , SIZE = 33792KB , MAXSIZE = UNLIMITED, FILEGROWTH = 1024KB )
 LOG ON ( NAME = N'CarSales_Logging_log', FILENAME = N'C:\SQLDIRecipes\Databases\CarSales_Logging_log.ldf' , SIZE = 4224KB , MAXSIZE = 2048GB , FILEGROWTH = 10%)
GO
ALTER DATABASE CarSales_Logging SET COMPATIBILITY_LEVEL = 110
GO
ALTER DATABASE CarSales_Logging SET QUOTED_IDENTIFIER OFF
GO
ALTER DATABASE CarSales_Logging SET RECOVERY SIMPLE
GO
ALTER DATABASE CarSales_Logging SET READ_WRITE
GO
```

## 示例文件的目录结构

`SQL2012DIRecipes.Zip` 文件中的目录结构见 表 B-1。

[表 B-1。


