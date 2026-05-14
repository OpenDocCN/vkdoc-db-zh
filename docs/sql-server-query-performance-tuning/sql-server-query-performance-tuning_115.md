# SQL Server 索引碎片概述

## 碎片类型
这种类型的碎片称为**外部碎片**。外部碎片始终是不希望出现的。

碎片也可能发生在索引页内部。如果 `INSERT` 或 `UPDATE` 操作导致页面拆分，那么原始叶页中将会留下空闲空间。`DELETE` 操作也可能导致空闲空间。其净效应是减少叶页中包含的行数。例如，在图 13-3 中，由 `INSERT` 操作引发的页面拆分在第一个叶页中创建了空闲空间。这被称为**内部碎片**。

对于高度事务性的数据库，最好在叶页中故意留出一些空闲空间，以便可以在不引起页面拆分的情况下添加新行或更改现有行的大小。在图 13-3 中，第一个叶页内的空闲空间允许将索引键值 26 添加到该叶页，而不会引起页面拆分。

## 注意事项
■ **注意**：请注意，此索引碎片不同于磁盘碎片。无法仅通过运行磁盘碎片整理工具来修复索引碎片，因为只有 SQL Server 理解 SQL Server 文件内页面的顺序，操作系统并不理解。

## 堆碎片与 DBCC 命令
堆页可能以相同方式变得碎片化。不幸的是，由于堆的存储方式以及任何非聚集索引使用物理数据位置从堆中检索数据的方式，对堆进行碎片整理是相当成问题的。你可以使用 `ALTER TABLE` 的 `REBUILD` 命令来执行堆重建，但要明白，这将强制重建与该表关联的任何非聚集索引。

SQL Server 2014 通过一个名为 `sys.dm_db_index_physical_stats` 的动态管理视图公开叶页和非叶页以及其他数据。它存储索引大小和碎片信息。我将在下一节中更详细地介绍它。这个 DMV 比旧的 `DBCC SHOWCONTIG` 更容易使用。

现在，让我们看一下碎片化的机制。

## 通过 UPDATE 语句引发的页面拆分示例
为了展示由 `UPDATE` 语句引发的页面拆分时会发生什么，我将使用一个构造的表。这个小测试表将有一个聚集索引，该索引在一个叶（或数据）页内对行进行排序，如下所示：

```sql
USE AdventureWorks2012;
GO
IF (SELECT OBJECT_ID('Test1')) IS NOT NULL
    DROP TABLE dbo.Test1;
GO
CREATE TABLE dbo.Test1
(C1 INT,
 C2 CHAR(999),
 C3 VARCHAR(10)
)
INSERT INTO dbo.Test1
VALUES (100, 'C2', ''),
       (200, 'C2', ''),
       (300, 'C2', ''),
       (400, 'C2', ''),
       (500, 'C2', ''),
       (600, 'C2', ''),
       (700, 'C2', ''),
       (800, 'C2', '');
CREATE CLUSTERED INDEX iClust
ON dbo.Test1(C1);
```

聚集索引叶页中一行的平均大小（不包括内部开销）不仅仅是聚集索引列的平均大小之和；它是表中所有列的平均大小之和，因为聚集索引的叶页和表的数据页是同一个。因此，基于前面样本数据的聚集索引中一行的平均大小如下：

= (`C1` 的平均大小) + (`C2` 的平均大小) + (`C3` 的平均大小) 字节 = (`INT` 的大小) + (`CHAR(999)` 的大小) + (`C3` 中数据的平均大小) 字节

= 4 + 999 + 0 = 1,003 字节

SQL Server 中一行的最大大小是 8,060 字节。因此，如果内部开销不是很高，所有八行都可以容纳在一个 8KB 页面中。

要确定分配给 `iClust` 聚集索引的叶页数量，请针对 `sys.dm_db_index_physical_stats` 执行 `SELECT` 语句。

```sql
SELECT ddips.avg_fragmentation_in_percent,
       ddips.fragment_count,
       ddips.page_count,
       ddips.avg_page_space_used_in_percent,
       ddips.record_count,
       ddips.avg_record_size_in_bytes
FROM sys.dm_db_index_physical_stats(DB_ID('AdventureWorks2012'),
                                    OBJECT_ID(N'dbo.Test1'), NULL,
                                    NULL, 'Sampled') AS ddips;
```


