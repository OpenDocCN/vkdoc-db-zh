# 第 3 章 ■ 统计信息

现在，假设你想运行一个仅基于 `FirstName` 参数选择数据的查询。该谓词对于 `IDX_Customers_LastName_FirstName` 索引不是 SARGable 的，因为 `LastName` 列（索引中最左侧的列）上没有 SARGable 谓词。

SQL Server 提供了两种不同的选项来执行查询。第一个选项是执行 `聚集索引扫描`。第二个选项是使用 `非聚集索引扫描`，同时为非聚集索引中 `FirstName` 值匹配参数的每一行执行 `键查找`。

非聚集索引的行大小远小于聚集索引。它使用更少的数据页，并且由于执行的 I/O 读取更少，扫描非聚集索引会比聚集索引扫描更高效。同时，当表中具有特定 `FirstName` 的行数很多并且需要大量键查找时，使用非聚集索引扫描的执行计划将不如聚集索引扫描高效。不幸的是，`IDX_Customers_LastName_FirstName` 索引的直方图仅存储 `LastName` 列的数据分布，SQL Server 并不了解 `FirstName` 的数据分布。

让我们运行 `Listing 3-3` 中所示的两个 SELECT 语句，并检查 `Figure 3-4` 中的执行计划。

### 清单 3-3. 列级统计信息：查询数据

```sql
select CustomerId, FirstName, LastName, Phone
from dbo.Customers
where FirstName = 'Brian';

select CustomerId, FirstName, LastName, Phone
from dbo.Customers
where FirstName = 'Victor';
```



