# 7-30. 使用 T-SQL 将数据导出到其他关系型数据库

## 问题

你希望使用 T-SQL 将数据导出到第三方关系型数据库管理系统（RDBMS）。

## 解决方案

使用 `OPENROWSET`、`OPENDATASOURCE` 或链接服务器。以下步骤展示了如何使用 `OPENROWSET` 将数据导出到 Oracle。

1.  按照“配方 4-4”中的脚本配置链接服务器。
2.  在目标 Oracle 数据库的 `SCOTT` 模式下创建一个名为 `Client_Dest` 的表：

```sql
CREATE TABLE "SCOTT"."Client_Dest"
(
    "ID" INTEGER,
    "ClientName" VARCHAR2(150),
    "Country" VARCHAR2(50),
    "Town" VARCHAR2(50),
    "County" VARCHAR2(50),
    "Address1" VARCHAR2(50),
    "Address2" VARCHAR2(50),
    "ClientType" VARCHAR2(20),
    "ClientSize" VARCHAR2(10)
);
```

3.  使用以下 SQL 代码片段将数据插入 Oracle 表 `Client_Dest`：

```sql
INSERT INTO OPENROWSET('OraOLEDB.Oracle', 'ORCL';'SYSTEM';'Me4B0ss', 'select ID,ClientName,Country,Town from SCOTT.Client_Dest')
 SELECT      ID,ClientName,Country,Town
 FROM        dbo.Client;
```

## 工作原理

得益于 OLEDB 和 .NET 提供程序对一系列关系型数据库的成熟支持，将数据导出到 Oracle、DB2、Sybase、Informix 和 PostgreSQL（仅举几例）变得异常简单——前提是所需的驱动程序以及在必要时所需的客户端软件已正确安装。我认为这一点再怎么强调也不为过。根据我的经验，绝大多数跨数据库导入导出问题都归结于这两个基本要素。一旦它们就位并经过测试，剩下的就是小菜一碟。因此，我鼓励你在进行任何数据导出尝试之前，确保这些链接的基本方面已正确设置。

然而，即使正确建立了链接基础设施，也并非总能保证执行过程万无一失。

你需要扎实了解目标表的数据类型以及关系结构所施加的任何限制——即任何唯一约束、外键、非空字段等等。这能让你确保导出的数据适用于目标数据结构并能成功加载。

`OPENROWSET` 命令需要表 7-9 中所示的参数。

| 描述 | 示例 |
| --- | --- |
| Oracle OLEDB 提供程序 | `OraOLEDB.Oracle` |
| TNSNames.Ora 文件中的 Oracle 地址 | `ORCL` |
| Oracle 登录名 | `SYSTEM` |
| Oracle 密码 | `Me4B0ss` |
| 目标字段 | `select ID,ClientName,Country,Town` |

在本配方的示例中，我使用 Oracle 作为外部数据目标，因为据我所知，它似乎是从 SQL Server 导出数据时最常使用的数据库。即将描述的技术适用于大多数其他关系型数据源，并且（除了少数罕见例外）只需为目标数据库更改提供程序名称即可。其他数据库提供程序及其一些特性在第 4 章中有详细描述。

当使用 Microsoft 或 Oracle 提供程序时，使用 `OPENROWSET` 和 `OPENDATASOURCE` 并不困难。这些示例使用的是较新的 Oracle 提供程序。

如果你更喜欢使用 `OPENDATASOURCE`，那么以下 SQL 代码片段将把数据插入 Oracle 表 `Client_Dest`：

```sql
SELECT * FROM OPENDATASOURCE('OraOLEDB.Oracle', 'Data Source = ORCL;User ID = SYSTEM;Password = Me4B0ss')..SCOTT.EMP;
```

假设你已经设置了一个名为 `MyOracleDatabase` 的链接服务器（如“配方 4-4”所述），那么将数据插入链接服务器是一个标准的 `INSERT...SELECT` 操作，只是使用四部分名称，类似于这样：

```sql
INSERT INTO MyOracleDatabase.CarSales.dbo.Client_Dest
(
    ID ,
    ClientName ,
    Country ,
    Town
)
SELECT
    ID ,
    ClientName ,
    Country ,
    Town
FROM dbo.Client;
```

## 提示、技巧和陷阱

-   确保源服务器和目标服务器之间的数据类型（和字段长度）兼容是你的责任。这可能意味着使用 `CAST` 和/或 `CONVERT` 来确保数据类型兼容，以及使用 `LEFT` 来截断字符数据。
-   在导出数据时，确保数据类型和长度允许你成功完成导出过程是你的责任。第 8 章包含获取链接服务器元数据的信息。附录 A 提供了数据库间数据类型映射的详细信息。

