# 第 13 章：索引碎片化

## 理解碎片化的影响

为了获得更好的性能，通常优选使用顺序 I/O，因为这可以在一次磁盘 I/O 操作中读取整个区（八个 8KB 页）。相比之下，非连续的页面布局需要非顺序或随机 I/O 操作来从磁盘检索索引页，而一次随机 I/O 操作在一次磁盘操作中只能读取 8KB 数据（但是，如果您只检索一行，这可能是可以接受的）。硬盘驱动器，尤其是 SSD，速度的不断提升已经降低了这个问题的影响，但它仍然存在。

在内部碎片化的情况下，行稀疏地分布在大量页面上，增加了将索引页读入内存所需的磁盘 I/O 操作数量，并增加了从内存中检索多个索引行所需的逻辑读取次数。如前所述，尽管它增加了数据检索的成本，但少量的内部碎片化可能是有益的，因为它允许您执行 `INSERT` 和 `UPDATE` 查询而不会引起页拆分。对于那些不需要遍历一系列页面来检索数据的查询，碎片化可能影响极小。换句话说，从索引中检索单个值不会受到碎片化的影响；或者，最多可能会多一个需要向下遍历的 B-Tree 层级。

## 演示碎片化效果

为了理解碎片化如何影响查询成本，请创建一个带有聚集索引的测试表，并向表中插入一个高度碎片化的数据集。由于在有序数据集之间执行 `INSERT` 操作可能导致页拆分，您可以通过按以下顺序添加行来轻松创建碎片化数据集：

```sql
IF (SELECT OBJECT_ID('Test1')) IS NOT NULL
    DROP TABLE dbo.Test1;
GO

CREATE TABLE dbo.Test1
(
    C1 INT,
    C2 INT,
    C3 INT,
    c4 CHAR(2000)
);

CREATE CLUSTERED INDEX i1 ON dbo.Test1 (C1);

WITH Nums
AS (SELECT TOP (10000)
        ROW_NUMBER() OVER (ORDER BY (SELECT 1)) AS n
    FROM master.sys.All_Columns ac1
    CROSS JOIN master.sys.All_Columns ac2
    )
INSERT INTO dbo.Test1
    (C1, C2, C3, c4)
SELECT n,
    n,
    n,
    'a'
FROM Nums;

WITH Nums
AS (SELECT 1 AS n
    UNION ALL
    SELECT n + 1
    FROM Nums
    WHERE n < 100
    )
INSERT INTO dbo.Test1
    (C1, C2, C3, c4)
SELECT 41 - n,
    n,
    n,
    'a'
FROM Nums;
```

为了确定从这个碎片化表中检索小结果集和大结果集所需的逻辑读取次数，请在 `STATISTICS IO` 和 `TIME` 设置为 `ON` 的情况下执行这两个 `SELECT` 语句：

```sql
--读取 6 行
SELECT *
FROM dbo.Test1
WHERE C1 BETWEEN 21 AND 23;

--读取所有行
SELECT *
FROM dbo.Test1
WHERE C1 BETWEEN 1 AND 10000;
```

各个查询执行的逻辑读取次数分别如下：
`表 'Test1'。扫描计数 1，逻辑读取 8 次`
`CPU 时间 = 0 ms，已用时间 = 19 ms。`

`表 'Test1'。扫描计数 1，逻辑读取 2542 次`
`CPU 时间 = 0 ms，已用时间 = 317 ms。`

## 重建以分析性能

为了评估碎片化数据集如何影响逻辑读取次数，请通过重建聚集索引来物理重新排列索引叶级页。

```sql
ALTER INDEX i1 ON dbo.Test1 REBUILD;
```

在索引叶级页按正确顺序重新排列后，重新运行测试。前面两个 `SELECT` 语句所需的逻辑读取次数分别减少到 5 和 13。

`表 'Test1'。扫描计数 1，逻辑读取 6 次`
`CPU 时间 = 0 ms，已用时间 = 15 ms。`

`表 'Test1'。扫描计数 1，逻辑读取 2536 次`
`CPU 时间 = 0 ms，已用时间 = 297 ms。`

对于较小的数据集，性能有所改善，但对于较大的数据集变化不大，因为仅仅跳过几个页面不太可能产生那么大的影响。由于碎片化带来的成本开销通常随着检索行数的增加而线性增长，因为这涉及到读取更多无序的页面。对于 `点查询`（只检索一行的查询），碎片化通常无关紧要，因为该行仅从一个叶级页检索，但情况并非总是如此。由于索引的内部结构，碎片化甚至可能增加点查询的成本。

> **注意** 本节的经验是，为了获得更好的查询性能，分析索引中的碎片化量并在需要时重新排列它至关重要。

## 分析碎片化量

您可以使用 `sys.dm_db_index_physical_stats` 动态管理函数来分析索引的碎片化比率。对于具有聚集索引的表，聚集索引的碎片化与数据页的碎片化一致，因为聚集索引的叶级页和数据页是相同的。

`sys.dm_db_index_physical_stats` 也指示堆表（或没有聚集索引的表）中的碎片化量。由于堆表不需要任何行排序，页面的逻辑顺序对于堆表并不重要。

`sys.dm_db_index_physical_stats` 的输出显示有关索引（或表）的页面和区的信息。为索引中的 B-树的每一级返回一行。为堆中的每个分配单元返回一行。如前所述，在 SQL Server 中，八个连续的 8KB 页面被分组在一个大小为 64KB 的区中。对于小表（远小于 64KB），一个区中的页面可以属于多个索引或表——这些被称为 `混合区`。如果数据库中有许多小表，混合区有助于 SQL Server 节省磁盘空间。

随着表（或索引）增长并请求超过八个页面，SQL Server 会创建一个专用于该表（或索引）的区，并分配此区中的页面。这样的区称为 `统一区`，它为同一个表（或索引）提供最多八次页面请求。统一区有助于 SQL Server 将表（或索引）的页面连续排列。它们还将页面创建请求的数量减少了八分之一，因为一组八个页面是以区的形式创建的。存储在统一区中的信息仍然可能是碎片化的，但访问一个页面分配将高效得多。如果您有混合区，页面在多个对象之间共享，并且这些区内存在碎片化，访问信息就会变得更加成问题。但不会对混合区进行碎片整理。

为了分析索引的碎片化，让我们使用“碎片化开销”部分中使用的碎片化数据集重新创建表。您可以通过执行前面使用的针对 `sys.dm_db_index_physical_stats` 动态视图的查询来获取聚集索引的碎片化详细信息（图 13-11）。

*图 13-11. 碎片化统计信息*

```sql
SELECT
    ddips.avg_fragmentation_in_percent,
    ddips.fragment_count,
    ddips.page_count,
    ddips.avg_page_space_used_in_percent,
    ddips.record_count,
    ddips.avg_record_size_in_bytes
FROM sys.dm_db_index_physical_stats(DB_ID('AdventureWorks2012'),
    OBJECT_ID(N'dbo.Test1'),NULL,
    NULL,'Sampled') AS ddips;
```


