# 第 22 章 ■ 逐行处理

使用游标逐行处理数据的数据库应用程序很常见。开发者倾向于以逐行方式思考数据处理。Oracle 甚至使用一种称为游标（cursor）的高速数据访问机制。但 SQL Server 中的游标不同。因为在 SQL Server 中通过游标进行数据操作会带来显著的额外开销，所以数据库应用程序应避免使用游标。T-SQL 和 SQL Server 设计为最适合处理数据集，而不是一次处理一行。Jeff Moden 将此类处理方式形象地称为 RBAR（发音为 ree-bar），意为“逐行痛苦地处理”。然而，如果必须使用游标，那么应使用开销最小的游标。

在本章中，我将涵盖以下主题：

*   游标的基础知识
*   游标不同特性的成本分析
*   默认结果集相对于游标的优势和劣势
*   最小化游标成本开销的建议

## 游标基础知识

当应用程序执行查询时，SQL Server 返回一组由行组成的数据。通常，应用程序无法一起处理多行；相反，它们通过遍历 SQL Server 返回的结果集来一次处理一行。此功能由游标提供，游标是一种从多行结果集中一次处理一行的机制。

T-SQL 游标处理通常涉及以下步骤：

1.  声明游标，将其与 `SELECT` 语句关联并定义游标的特性。
2.  打开游标以访问 `SELECT` 语句返回的结果集。
3.  从游标中检索一行。可选地，通过游标修改该行。
4.  处理完结果集中的所有行后，关闭游标并释放分配给游标的资源。

你可以使用 T-SQL 语句或用于连接到 SQL Server 的数据访问层创建游标。使用数据访问层创建的游标通常称为客户端游标。用 T-SQL 编写的游标称为服务器端游标。以下是一个处理表中查询结果的服务器端游标示例：

```sql
--将 SELECT 语句与游标关联并定义游标特性
USE AdventureWorks2012;
GO
SET NOCOUNT ON
DECLARE MyCursor CURSOR /*<游标特性>*/
FOR
SELECT adt.AddressTypeID,
       adt.Name,
       adt.ModifiedDate
FROM Person.AddressType adt;

--打开游标以访问 SELECT 语句返回的结果集
OPEN MyCursor;

--从 SELECT 语句返回的结果集中一次检索一行
DECLARE @AddressTypeId INT,
        @Name VARCHAR(50),
        @ModifiedDate DATETIME;
```



