# 检查统计信息

在此次查询中引用了五张表：`Purchasing.PurchaseOrderHeader`、`Purchasing.PurchaseOrderDetail`、`Person.Employee`、`Person.Person` 以及 `Production.Product`。你必须知道查询使用了哪些索引才能获取其统计信息。这一点可以通过查看执行计划来确定。现在，我将检查 `HumanResources.Employee` 表主键的统计信息，因为该表产生了最多的读取操作。现在运行以下查询：

```sql
DBCC SHOW_STATISTICS('HumanResources.Employee',
'PK_Employee_BusinessEntityID');
```

当上述查询完成后，你将看到如图 25-3 所示的输出。

**图 25-3.** `HumanResources.Employee` 的 `SHOW_STATISTICS` 输出

你可以看到索引的选择性非常高，因为如“全部密度”列所示，其密度相当低。在此情况下，统计信息不太可能是此查询性能不佳的原因。在可能的情况下，最好查看实际的执行计划并比较其中的估计行数与实际行数。你还可以检查“更新”列以确定此组统计信息的最后更新时间。如果统计信息的更新时间距今已超过几天，那么你需要检查你的统计信息维护计划，并应手动更新这些统计信息。就提供的数据而言，这些统计信息可能已经严重过时。

### 分析碎片需求的必要性

如第 13 章所述，碎片化的表会增加执行扫描的查询所需访问的页数，从而对性能产生不利影响。然而，对于点查询，碎片通常不是问题。因此，你应该确保查询中引用的数据库对象没有过度碎片化。

你可以通过运行一个针对 `sys.dm_db_index_physical_stats` 的查询来确定性能最差的查询所访问的五张表的碎片情况。首先针对 `HumanResources.Employee` 表运行查询。

```sql
SELECT s.avg_fragmentation_in_percent,
       s.fragment_count,
       s.page_count,
       s.avg_page_space_used_in_percent,
       s.record_count,
       s.avg_record_size_in_bytes,
       s.index_id
FROM sys.dm_db_index_physical_stats(DB_ID('AdventureWorks2012'),
                                    OBJECT_ID(N'HumanResources.Employee'),
                                    NULL, NULL, 'Sampled') AS s
WHERE s.record_count > 0
ORDER BY s.index_id;
```

图 25-4 展示了此查询的输出。

**图 25-4.** `HumanResources.Employee` 表的索引碎片情况

如果你针对其他四张表运行相同的查询（按顺序为 `Purchasing.PurchaseOrderHeader`、`Purchasing.PurchaseOrderDetail`、`Production.Product` 和 `Person.Person`），输出将如图 25-5 所示。

**图 25-5.** 问题查询中四张表的索引碎片情况

`Purchasing.PurchaseOrderHeader` 表的碎片非常轻微：28%。同时，许多索引的 `avg_page_space_used_in_percent` 都大于 90%。考虑到该表上索引的页数（42 或更少），如第 13 章详述，通过整理索引（假设可以整理）来提升性能的可能性非常低。

`Purchasing.PurchaseOrderDetail` 表也是如此，其碎片率很低，页数也很少。`Production.Product` 的碎片程度稍高；但同样，页数非常少，因此整理索引可能帮助不大。`Person.Employee` 有一个索引的碎片率为 **66**%；然而，它又只分布在三页上。最后，`Person.Person` 几乎没有碎片可言。



