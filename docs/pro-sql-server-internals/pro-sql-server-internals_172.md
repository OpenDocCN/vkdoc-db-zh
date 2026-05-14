# 列存储压缩与元数据查询

让我们运行一个查询，该查询在表的 20 个列上执行 `MAX()` 聚合。查询结果没有意义；然而，它迫使 SQL Server 从表中每个行组的 20 个不同列段中读取数据。清单 33-10 展示了该查询。

**清单 33-10.** 测试查询

```sql
select max(ProductKey),max(OrderDateKey),max(DueDateKey),max(ShipDateKey),max(CustomerKey)
,max(PromotionKey),max(CurrencyKey),max(SalesTerritoryKey),max(SalesOrderLineNumber)
,max(RevisionNumber),max(OrderQuantity),max(UnitPrice),max(ExtendedAmount)
,max(UnitPriceDiscountPct),max(DiscountAmount),max(ProductStandardCost)
,max(TotalProductCost)
,max(SalesAmount),max(TaxAmt),max(Freight)
from dbo.FactSalesBig;
```

表 33-6 说明了针对具有不同列存储压缩方法的表执行该查询的执行时间。即使使用归档压缩的数据在磁盘上占用的空间显著减少，但由于涉及解压缩开销，查询完成所需的时间更长。显然，结果会根据系统的 CPU 和 I/O 性能而有所不同。

**表 33-6.** 不同压缩方法的执行时间
*   **列存储压缩**（经过时间/CPU 时间）：1,458 ms / 4,733 ms
*   **列存储归档压缩**（经过时间/CPU 时间）：1,774 ms / 6,098 ms

归档压缩是静态、极少访问数据的绝佳选择，我想重申一下，它可以基于每个索引分区使用。数据仓库通常会保留长时间的数据，即使历史数据很少被访问。您可能希望考虑对存储旧数据的分区应用归档压缩，并从中获益于其节省的磁盘空间。

#### 元数据

SQL Server 提供了几个与列存储索引相关的目录和数据管理视图。接下来描述的两个目录视图在 SQL Server 2012-2016 中有效。我们将在下一章中查看其他视图。

### sys.column_store_segments

`sys.column_store_segments` 视图为每个列的每个段返回一行。

清单 33-11 展示了一个查询，该查询返回有关在 `dbo.FactSales` 表上定义的 `IDX_FactSales_ColumnStore` 列存储索引的信息。这里有几个需要注意的地方。首先，该视图不返回索引的 `object_id` 或 `index_id`。这不是问题，因为一个表只能定义一个列存储索引。但是，当需要时，您需要使用 `sys.partitions` 视图来获取 `object_id`。

其次，与常规 B-Tree 索引类似，非聚集列存储索引包含一个 *row-id*，它要么是堆表中行的地址，要么是聚集索引键值。在后一种情况下，即使您没有在 `CREATE COLUMNSTORE INDEX` 语句中显式定义它们，聚集索引的所有列也会包含在列存储索引中。但是，这些列不会存在于 `sys.index_columns` 视图中，如果您想获取列名，则需要使用外连接。

**清单 33-11.** 检查 `sys.column_store_segments` 视图

```sql
select p.partition_number as [partition], c.name as [column], s.column_id, s.segment_id
,p.data_compression_desc as [compression], s.version, s.encoding_type, s.row_count
, s.has_nulls, s.magnitude,s.primary_dictionary_id, s.secondary_dictionary_id,
, s.min_data_id, s.max_data_id, s.null_value
, convert(decimal(12,3),s.on_disk_size / 1024.0 / 1024.0) as [Size MB]
from sys.column_store_segments s join sys.partitions p on
p.partition_id = s.partition_id
join sys.indexes i on
p.object_id = i.object_id
left join sys.index_columns ic on
i.index_id = ic.index_id and
i.object_id = ic.object_id and
s.column_id = ic.index_column_id
left join sys.columns c on
ic.column_id = c.column_id and
ic.object_id = c.object_id
where i.name = 'IDX_FactSales_ColumnStore'
order by p.partition_number, s.segment_id, s.column_id
```

