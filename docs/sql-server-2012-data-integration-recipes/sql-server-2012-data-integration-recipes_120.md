# 使用 SSIS 处理类型 2 缓慢变化维度

## 问题

你需要作为 SSIS 数据流过程的一部分，在将源数据添加到目标表时，跟踪其随时间的变化。

## 解决方案

使用一个 SSIS 包，该包利用条件拆分和执行 SQL 任务，将此过程视为类型 2 SCD（缓慢变化维度）进行处理。以下步骤将指导你完成。

1.  在目标数据库 (`CarSales_Staging`) 中，为 `Client_SCDSSIS2` 类型 2 维度创建一个表。此表的 DDL 为 (`C:\SQL2012DIRecipes\CH09\tblClientSCDSSIS2.sql`)：

    ```sql
    USE CarSales_Staging;
    GO

    CREATE TABLE CarSales_Staging.dbo.Client_SCDSSIS2
    (
     SurrogateID INT IDENTITY(1,1) NOT NULL,
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
    );
    GO
    ```

2.  创建一个临时表——在暂存数据库中创建和测试包期间存放在磁盘上——该表将用于更新维度表。此表的 DDL 为：

    ```sql
    CREATE TABLE CarSales_Staging.dbo.Tmp_Client_SCDSSIS2
    (
     SurrogateID INT NOT NULL
    );
    GO
    ```

3.  创建一个新的 SSIS 包，命名为 `SCD_Type2`。添加两个 OLEDB 连接管理器——一个命名为 `CarSales_OLEDB`，连接到 `CarSales` 数据库；另一个命名为 `CarSales_Staging_OLEDB`，连接到 `CarSales_Staging` 数据库。将后者的 `RetainSameConnection` 属性设置为 True。
4.  添加以下两个变量：

    | **名称** | **作用域** | **数据类型** | **值** | **注释** |
    |---|---|---|---|---|
    | `TempTable` | `SCD_Type2` | String | `Tmp_Client_SCD` | 用于在数据库表和临时表之间切换。 |
    | `ValidFrom` | `SCD_Type2` | Int32 |  | 自动获取当前日期。 |

5.  将 `ValidFrom` 变量的 `EvaluateAsExpression` 属性设置为 True。设置表达式（这将获取当前日期作为整数）：

    ```text
    YEAR( GETDATE()) * 100000 + MONTH( GETDATE()) * 1000 + DAY( GETDATE())
    ```

6.  在“控制流”选项卡中，添加一个执行 SQL 任务，并按如下配置：

    | **名称** | **Create Temp Table for Session** |
    |---|---|
    | 连接： | `CarSales_Staging_OLEDB` |
    | SQL 语句： | `CREATE TABLE ##Tmp_Client_SCDSSIS2 (SurrogateID INT NOT NULL)` |

7.  即使这个临时表要等到包调试并运行成功后才使用，现在创建它也是有益的。单击“确定”以确认更改。
8.  添加一个数据流任务，并将其命名为 `Main SCD Type 2 Process`。将上一个执行 SQL 任务（Create Temp Table for Session）连接到它。双击以进行编辑。
9.  在数据流窗格中添加一个 OLEDB 源连接，将其命名为 `Source Data`，并按如下配置：

    | OLEDB 连接管理器： | `CarSales_OLEDB` |
    |---|---|
    | 数据访问模式： | SQL 命令 |
    | SQL 命令文本： | `SELECT ID, ClientName, Country, Town, County, Address1, Address2, ClientType, ClientSize FROM dbo.Client ORDER BY ID` |

10. 单击“确定”确认。
11. 右键单击 `Source Data` OLEDB 源连接，选择“显示高级编辑器”。选择“输入和输出属性”选项卡，单击“OLEDB 源输出”，并在对话框右侧将 `IsSorted` 属性设置为 True。
12. 展开“OLEDB 源输出”并展开“输出列”。单击 `ID` 列，并将 `SortKeyPosition` 属性设置为 1。
13. 添加一个查找转换，并将刚刚创建的源连接到它。双击以进行编辑。在“常规”选项卡中，设置为使用 OLEDB 连接管理器，缓存模式为完全缓存，并将不匹配的行重定向到“NoMatch 输出”。在“连接”窗格中，将 OLEDB 连接管理器设置为 `CarSales_Staging_OLEDB`，并使用以下 SQL 查询的结果：

    ```sql
    SELECT     SurrogateID, BusinessKey, ClientName, Country, Town, County, Address1,
               Address2, ClientType, ClientSize, ValidFrom, ValidTo, IsCurrent
    FROM       dbo.Client_SCDSSIS2 WITH (NOLOCK)
    ORDER BY   BusinessKey
    ```

14.


