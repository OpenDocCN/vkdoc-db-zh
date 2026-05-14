# 索引分析与压缩

#### 使用场景

报表系统最能从索引视图中获益。写入频繁的 OLTP 系统可能无法充分利用索引视图，因为维护索引视图及其底层基表的成本会增加。索引视图带来的净性能提升，是视图提供的总查询执行节省与存储和维护视图的成本之差。

> **注意**
> 如果查询连接设置与这些 ANSI 标准设置不匹配，在针对索引视图中使用的表进行插入/更新/删除操作时，你可能会看到错误。

如果你使用的是 SQL Server 企业版，查询优化器在执行查询时无需在查询语句中显式引用索引视图。这使得现有应用程序无需修改就能从新创建的索引视图中受益。否则，在非企业版的 SQL Server 上，你需要在 T-SQL 代码中直接引用它。查询优化器仅为成本显著的查询考虑索引视图。你可能还会发现，新的列存储索引可能比索引视图更适合你，尤其是在预聚合数据时。我将在本章后面的一节中介绍列存储索引。

让我们通过以下示例看看索引视图是如何工作的。考虑以下三个查询：

```sql
SELECT p.[Name] AS ProductName,
    SUM(pod.OrderQty) AS OrderOty,
    SUM(pod.ReceivedQty) AS ReceivedOty,
    SUM(pod.RejectedQty) AS RejectedOty
FROM Purchasing.PurchaseOrderDetail AS pod
JOIN Production.Product AS p
    ON p.ProductID = pod.ProductID
GROUP BY p.[Name];
```

```sql
SELECT p.[Name] AS ProductName,
    SUM(pod.OrderQty) AS OrderOty,
    SUM(pod.ReceivedQty) AS ReceivedOty,
    SUM(pod.RejectedQty) AS RejectedOty
FROM Purchasing.PurchaseOrderDetail AS pod
JOIN Production.Product AS p
    ON p.ProductID = pod.ProductID
GROUP BY p.[Name]
HAVING (SUM(pod.RejectedQty) / SUM(pod.ReceivedQty)) > .08;
```

```sql
SELECT p.[Name] AS ProductName,
    SUM(pod.OrderQty) AS OrderQty,
    SUM(pod.ReceivedQty) AS ReceivedQty,
    SUM(pod.RejectedQty) AS RejectedQty
FROM Purchasing.PurchaseOrderDetail AS pod
JOIN Production.Product AS p
    ON p.ProductID = pod.ProductID
WHERE p.[Name] LIKE 'Chain%'
GROUP BY p.[Name];
```

所有三个查询都对`PurchaseOrderDetail`表的列使用了聚合函数`SUM()`。因此，你可以创建一个索引视图来预计算这些聚合，并在查询执行期间最小化这些复杂计算的成本。

以下是这些查询执行的逻辑读次数：
- 查询 1：
  ```
  Table 'Workfile'. Scan count 0, logical reads 0
  Table 'Worktable'. Scan count 0, logical reads 0
  Table 'Product'. Scan count 1, logical reads 6
  Table 'PurchaseOrderDetail'. Scan count 1, logical reads 66
  CPU time = 0 ms, elapsed time = 128 ms.
  ```
- 查询 2：
  ```
  Table 'Workfile'. Scan count 0, logical reads 0
  Table 'Worktable'. Scan count 0, logical reads 0
  Table 'Product'. Scan count 1, logical reads 6
  Table 'PurchaseOrderDetail'. Scan count 1, logical reads 66
  CPU time = 0 ms, elapsed time = 158 ms.
  ```
- 查询 3：
  ```
  Table 'PurchaseOrderDetail'. Scan count 5, logical reads 894
  Table 'Product'. Scan count 1, logical reads 2, physical rea
  CPU time = 0 ms, elapsed time = 139 ms.
  ```

我将使用以下脚本创建索引视图来预计算开销大的计算并连接表：

```sql
IF EXISTS ( SELECT *
    FROM sys.views
    WHERE object_id = OBJECT_ID(N'[Purchasing].[IndexedView]') )
    DROP VIEW [Purchasing].[IndexedView];
GO

CREATE VIEW Purchasing.IndexedView
    WITH SCHEMABINDING
AS
    SELECT pod.ProductID,
        SUM(pod.OrderQty) AS OrderQty,
        SUM(pod.ReceivedQty) AS ReceivedQty,
        SUM(pod.RejectedQty) AS RejectedQty,
        COUNT_BIG(*) AS [Count]
    FROM Purchasing.PurchaseOrderDetail AS pod
    GROUP BY pod.ProductID;
GO

CREATE UNIQUE CLUSTERED INDEX iv
    ON Purchasing.IndexedView (ProductID);
GO
```

某些构造（如`AVG()`）是不允许的。（有关不允许的构造的完整列表，请参阅 SQL Server 联机丛书。）如果视图中包含聚合，如此示例，则必须默认包含`COUNT_BIG()`。

索引视图将聚合函数的结果具体化存储在磁盘上。这消除了在执行关注聚合输出的查询时计算聚合函数的需要。

例如，第三个查询请求`PurchaseOrderDetail`表中某些产品的`ReceivedQty`和`RejectedQty`之和。由于这些值已在索引视图中为`PurchaseOrderDetail`表中的每个产品具体化，你可以使用以下`SELECT`语句从索引视图中获取这些预聚合的值：

```sql
SELECT iv.ProductID,
    iv.ReceivedQty,
    iv.RejectedQty
FROM Purchasing.IndexedView AS iv;
```

如图 9-11 所示的执行计划中，`SELECT`语句直接从索引视图检索值，而无需访问基表（`PurchaseOrderDetail`）。

**图 9-11.** 使用索引视图的执行计划

索引视图不仅使直接基于该视图的查询受益，也可能使其他关注具体化数据的查询受益。例如，有了索引视图后，针对`PurchaseOrderDetail`的三个查询无需重写即可获益（第一个查询的执行计划见图 9-12），逻辑读次数减少如下：

- 使用索引视图后的查询 1：
  ```
  Table 'Product'. Scan count 1, logical reads 13
  Table 'IndexedView'. Scan count 1, logical reads 4
  CPU time = 0 ms, elapsed time = 88 ms.
  ```
- 使用索引视图后的查询 2：
  ```
  Table 'Product'. Scan count 1, logical reads 13
  Table 'IndexedView'. Scan count 1, logical reads 4
  CPU time = 0 ms, elapsed time = 0 ms.
  ```
- 使用索引视图后的查询 3：
  ```
  Table 'IndexedView'. Scan count 0, logical reads 10
  Table 'Product'. Scan count 1, logical reads 2
  CPU time = 0 ms, elapsed time = 41 ms.
  ```

**图 9-12.** 自动使用索引视图的执行计划

尽管查询未被修改以引用新的索引视图，优化器仍然使用索引视图来提高性能。因此，即使数据库应用程序中的现有查询也能从新的索引视图中受益，而无需修改查询。如果你需要与索引视图提供的聚合不同的聚合，那将无法实现。这里再次体现了列存储索引的优势。

请确保清理资源。

```sql
DROP VIEW Purchasing.IndexedView;
```

### 索引压缩

数据和索引压缩在 SQL Server 2008 中引入（在企业版和开发人员版中可用）。*压缩*索引意味着将更多的键信息容纳到单个页面上。这可以带来显著的性能改进，因为存储索引所需的页面更少，索引层级也更少。在 CPU 方面会有开销，因为索引中的键值会被压缩和解压缩，所以这可能不是所有索引的解决方案。内存也受益，因为压缩后的页面在内存中以压缩状态存储。



默认情况下，索引不会被压缩。你必须在创建索引时显式地调用压缩。压缩有两种类型：行级压缩和页级压缩。*行级压缩*识别可压缩的列（详见联机丛书）并压缩该列的存储，对每一行都执行此操作。*页级压缩*实际上先使用行级压缩，然后在此基础上添加额外的压缩，以减少存储在页上的非行元素的存储空间。在页类型下，索引的非叶页不接收任何压缩。

要查看索引压缩的实际效果，考虑以下索引：
```sql
CREATE NONCLUSTERED INDEX IX_Test
ON Person.Address(City ASC, PostalCode ASC);
```
该索引在本章前面已创建。如果按照此处定义重新创建它，则会为具有与第一个测试索引`IX_Test`相同两列的索引创建行类型压缩。

```sql
CREATE NONCLUSTERED INDEX IX_Comp_Test
ON Person.Address (City,PostalCode)
WITH (DATA_COMPRESSION = ROW);
```
再创建一个索引。
```sql
CREATE NONCLUSTERED INDEX IX_Comp_Page_Test
ON Person.Address (City,PostalCode)
WITH (DATA_COMPRESSION = PAGE);
```
为了检查存储的索引，修改针对`sys.dm_db_index_physical_stats`的原始查询，添加另一个列`compressed_page_count`。
```sql
SELECT i.Name,
i.type_desc,
s.page_count,
s.record_count,
s.index_level,
compressed_page_count
FROM sys.indexes i
JOIN sys.dm_db_index_physical_stats(DB_ID(N'AdventureWorks2012'),
OBJECT_ID(N'Person.Address'),NULL,
NULL,'DETAILED') AS s
ON i.index_id = s.index_id
WHERE i.OBJECT_ID = OBJECT_ID(N'Person.Address');
```
运行查询，得到图 9-13 所示的结果。
*图 9-13. `sys.dm_db_index_physical_stats`关于压缩索引的输出*
对于这个索引，你可以看到页压缩能够将索引从 106 页减少到 25 页，其中 25 页被压缩。此实例中的行类型压缩减少了索引中的页数，但效果远不如页压缩显著。

为了在不修改代码的情况下看到压缩为你工作，运行以下查询：
```sql
SELECT a.City,
a.PostalCode
FROM Person.Address AS a
WHERE a.City = 'Newton'
AND a.PostalCode = 'V2M1N7';
```
优化器选择在我的系统上使用`IXCompPageTest`索引。即使我强制它使用`IXTest`索引如下，性能也是相同的，尽管在第二个查询中多读取了一页：
```sql
SELECT a.City,
a.PostalCode
FROM Person.Address AS a WITH (INDEX = IX_Test)
WHERE a.City = 'Newton'
AND a.PostalCode = 'V2M1N7';
```
因此，虽然一个索引占用的空间大约是另一个的四分之一，页数也少得多，但这是在没有性能成本的情况下实现的。

压缩对 SQL Server 内的其他过程有一系列影响，因此在实施前应彻底探索可能的影响以及可能的益处。在大多数情况下，CPU 的成本完全被其他地方的收益所超越，但你应该测试并监控你的系统。

测试完成后清理索引。
```sql
DROP INDEX Person.Address.IX_Test;
DROP INDEX Person.Address.IX_Comp_Test;
DROP INDEX Person.Address.IX_Comp_Page_Test;
```

### 列存储索引
列存储索引在 SQL Server 2012 中引入，它按列而非按行来索引信息。
这在处理需要快速聚合和访问大量数据的数据仓库系统时特别有用。列存储索引中存储的信息按每列分组，并且这些分组是单独存储的。这使得对不同列集合的聚合变得极快，因为可以访问列存储索引，而无需访问大量行来聚合信息。

此外，你能获得更快的速度，因为存储是面向列的，因此你只会触及感兴趣的列的存储，而不是整行的列。最后，你会看到列存储带来的一些性能提升，因为列数据是压缩存储的。列存储有两种类型，类似于常规索引：聚集列存储和非聚集列存储。非聚集列存储不能被更新。你必须先删除它，然后重新创建它（或者，如果你正在使用分区，可以切换入和切换出不同的分区）。聚集列存储在 SQL Server 2014 中引入，并且仅在生产机器的 Enterprise 版本中可用。使用列存储索引有一些限制。

*   不能使用某些数据类型，如`binary`、`text`、`varchar(max)`、`uniqueidentifier`（在 SQL Server 2012 中，此数据类型在 SQL Server 2014 中有效）、`clr`数据类型、`xml`或精度大于 18 的`decimal`。
*   不能在稀疏列上创建列存储索引。
*   创建聚集列存储索引时，它可以是表上的唯一索引。
*   你想创建聚集列存储索引的表不能有任何约束，包括主键或外键约束。

有关限制的完整列表，请参阅联机丛书。

列存储主要设计用于数据仓库内部，因此在处理相关的存储样式（如星型模式）时效果最佳。在`AdventureWorks2012`数据库中，`Production.TransactionHistoryArchive`表是一种比许多其他结构更可能用于聚合查询的结构。由于它是一个存档表，其加载也是受控的，因此列存储索引可以成功地在这里使用。以这个查询为例：
```sql
SELECT tha.ProductID,
COUNT(tha.ProductID) AS CountProductID,
SUM(tha.Quantity) AS SumQuantity,
AVG(tha.ActualCost) AS AvgActualCost
FROM Production.TransactionHistoryArchive AS tha
GROUP BY tha.ProductID;
```
如果你对当前配置的表运行此查询，你会看到一个执行计划，如图 9-14 所示。
*图 9-14. `GROUP BY`查询的聚集索引扫描和哈希匹配聚合*
该查询的读取次数和执行时间如下：
```
Table 'Worktable'. Scan count 0, logical reads 0, physical reads 0
Table 'Workfile'. Scan count 0, logical reads 0, physical reads 0,
Table 'TransactionHistoryArchive'. Scan count 1, logical reads 628
CPU time = 16 ms, elapsed time = 126 ms.
```
有大量的读取，此查询使用了相当多的 CPU，并且执行速度不是特别快。我们有两种类型的列存储索引可供选择。如果你想只为现有表添加一个非聚集列存储索引，这也是可能的。
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX ix_csTest
ON Production.TransactionHistoryArchive
(ProductID,
Quantity,
ActualCost);
```


