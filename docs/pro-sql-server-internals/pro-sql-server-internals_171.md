# 第 33 章：基于列的存储与批处理模式执行

## 压缩与存储大小

我们已经了解到，列存储索引中的数据经过高度压缩，与页面压缩相比也能显著节省空间。此外，SQL Server 2014 引入了另一种称为 `归档压缩` 的压缩选项。它可以通过指定 `DATA_COMPRESSION=COLUMNSTORE_ARCHIVE` 列存储索引属性来应用于整个索引或单个分区，从而进一步减少存储空间。它使用 Xpress 8 压缩库，这是 Microsoft 对 LZ77 算法的内部实现。这种压缩直接处理二进制数据，无需了解底层 SQL Server 数据结构。

归档压缩与其他 SQL Server 功能透明协作。列存储数据在保存到磁盘时被压缩，在加载到内存之前被解压缩。

让我们比较一下不同压缩方法的结果。我创建了四个模式相同的表，如代码清单 33-9 所示。前两个表是堆表，没有定义非聚集索引。第一个表未压缩，第二个表使用页面压缩。第三和第四个表具有聚集列存储索引（下一章将详细介绍），分别使用 `COLUMNSTORE` 和 `COLUMNSTORE_ARCHIVE` 压缩方法进行压缩。每个表都有大约 6200 万行数据，这些数据基于 `AdventureWorksDW2012` 数据库中的 `dbo.FactResellerSales` 表生成。

### 代码清单 33-9. 测试表的模式

```sql
create table dbo.FactSalesBig
(
    ProductKey int not null,
    OrderDateKey int not null,
    DueDateKey int not null,
    ShipDateKey int not null,
    CustomerKey int not null,
    PromotionKey int not null,
    CurrencyKey int not null,
    SalesTerritoryKey int not null,
    SalesOrderNumber nvarchar(20) not null,
    SalesOrderLineNumber tinyint not null,
    RevisionNumber tinyint not null,
    OrderQuantity smallint not null,
    UnitPrice money not null,
    ExtendedAmount money not null,
    UnitPriceDiscountPct float not null,
    DiscountAmount float not null,
    ProductStandardCost money not null,
    TotalProductCost money not null,
    SalesAmount money not null,
    TaxAmt money not null,
    Freight money not null,
    CarrierTrackingNumber nvarchar(25) null,
    CustomerPONumber nvarchar(25) null,
    OrderDate datetime null,
    DueDate datetime null,
    ShipDate datetime null
)
```

表 33-5 比较了所有四种压缩方法的磁盘占用大小。

**表 33-5. 不同压缩方法对应的磁盘数据大小**

| 堆表（无压缩） | 堆表（页面压缩） | 列存储压缩 | 归档压缩 |
| :--- | :--- | :--- | :--- |
| 10,504 MB | 2,440 MB | 831 MB | 362 MB |

显然，不同的表模式和数据会导致不同的压缩结果；然而，在大多数情况下，实施归档压缩将实现**显著**更大的空间节省。归档压缩在压缩和解压缩时会带来额外的 CPU 开销。

代码清单 33-8 显示了一个查询，用于展示本章前面创建的 `dbo.FactSales` 表的分配单元。

### 代码清单 33-8. `dbo.FactSales` 表的分配单元

```sql
select i.name as [Index], p.index_id, p.partition_number as [Partition]
,p.data_compression_desc as [Compression], u.type_desc, u.total_pages
from sys.partitions p join sys.allocation_units u on
p.partition_id = u.container_id
join sys.indexes i on
p.object_id = i.object_id and p.index_id = i.index_id
where p.object_id = object_id(N'dbo.FactSales')
```

如图 33-20 所示，列存储索引存储为 `LOB_DATA`。值得注意的是，此索引具有 `IN_ROW_DATA` 分配单元；但是，这些分配单元不存储任何数据。如果没有 `IN_ROW_DATA` 分配，索引中就不可能存在 `LOB_DATA` 分配。

**图 33-20. `dbo.FactSales` 表的分配单元**

