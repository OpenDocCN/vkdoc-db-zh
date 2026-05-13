# 5. 非聚集索引

本章讨论非聚集索引，这是 `In-Memory OLTP Engine` 支持的第二种索引类型。它展示了如何定义非聚集索引，讨论了它们的 `SARGability` 规则，并解释了它们的内部结构。

最后，本章讨论了针对内存优化表的几种索引策略和设计注意事项。

### 使用非聚集索引

非聚集索引是 `In-Memory OLTP Engine` 支持的另一种索引类型。与针对点查找相等搜索进行优化的哈希索引相反，非聚集索引帮助你基于值的范围来搜索数据。它们的结构与基于磁盘的表上的常规索引有些相似。然而，它们并非完全相同，我将在本章后面深入讨论它们的内部实现。

### 术语问题

非聚集索引在 `SQL Server 2014 CTP 2` 中引入，该版本的文档和白皮书使用术语 `range indexes` 来指代它们。但是，在 `SQL Server 2014` 的正式发布版中，Microsoft 将术语更改为 `nonclustered indexes`。尽管如此，你仍然可以在文档和数据管理视图中找到 `range indexes` 这个术语。

这种术语可能会令人困惑，因为哈希索引也非聚集的。事实上，聚集索引的概念不能应用于 `In-Memory OLTP`。数据行在内存中不以任何特定顺序存储。

同样值得注意的是，`In-Memory OLTP` 索引的最小 `index_id` 值为 `2`，这对应于基于磁盘的表中的非聚集索引。

### 创建非聚集索引

非聚集索引作为 `CREATE TABLE` 语句的一部分内联创建。其语法与哈希索引创建类似；但是，你应该省略关键字 `HASH`，并且不需要在索引属性中指定存储桶的数量。

清单 5-1 中的代码创建了一个内存优化表，其中包含两个非聚集索引，一个是复合索引，另一个是单列索引。

```sql
create table dbo.Customers
(
CustomerId int identity(1,1) not null
constraint PK_Customers
primary key nonclustered
hash with (bucket_count=1024),
FirstName varchar(32) not null,
LastName varchar(64) not null,
FullName varchar(97) not null,
index IDX_LastName_FirstName
nonclustered(LastName, FirstName),
index IDX_FullName
nonclustered(FullName)
)
with (memory_optimized=on, durability=schema_only);
```
清单 5-1. 创建带有两个非聚集索引的表

### 使用非聚集索引

与基于磁盘的表中的 `B-Tree` 索引类似，非聚集索引中的数据根据索引键列的值排序。因此，非聚集索引在大量用例中非常有益。当查询谓词允许 `SQL Server` 定位并隔离一部分索引键进行处理时，它们可以导致 `Index Seek` 操作。除极少数例外情况，非聚集索引的 `SARGability` 规则与在基于磁盘的表上定义的索引规则相匹配。

清单 5-2 展示了对 `dbo.Customers` 表的几个查询。`SQL Server` 能够对所有查询使用 `Index Seek` 操作。

```sql
-- 指定索引中所有列的点查找
select CustomerId, FirstName, LastName
from dbo.Customers
where LastName = 'White' and FirstName = 'Paul';
-- 使用最左侧索引列的点查找
select CustomerId, FirstName, LastName
from dbo.Customers
where LastName = 'White';
-- 使用 ">", ">=", "<", "<="
select CustomerId, FirstName, LastName
from dbo.Customers
where LastName > 'W';
-- 前缀搜索
select CustomerId, FirstName, LastName
from dbo.Customers
where LastName like 'Wh%';
-- IN 列表
select CustomerId, FirstName, LastName
from dbo.Customers
where LastName in ('White','Isakov');
```
清单 5-2. 导致 Index Seek 操作的查询

与 `B-Tree` 索引类似，当查询谓词不允许你隔离一部分索引键进行处理时，`Index Seek` 操作是不可能的。清单 5-3 展示了此类查询的几个示例。

```sql
-- 省略最左侧的索引列
select CustomerId, FirstName, LastName
from dbo.Customers
where FirstName = 'Paul';
-- 子串搜索
select CustomerId, FirstName, LastName
from dbo.Customers
where LastName like '%hit%';
-- 使用函数
select CustomerId, FirstName, LastName
from dbo.Customers
where len(LastName) = 5;
```
清单 5-3. 导致 Index Scan 操作的查询

与基于磁盘的表上的 `B-Tree` 索引相反，非聚集索引是单向的，`SQL Server` 无法按照与排序顺序相反的方向扫描索引键。在定义索引和为列选择排序顺序时，你应该牢记此行为。

让我们用一个例子来说明；我们将创建一个结构与 `dbo.Customers` 相同的基于磁盘的表，并用相同的数据填充这两个表。清单 5-4 展示了执行此操作的代码。

```sql
create table dbo.Customers_OnDisk
(
CustomerId int identity(1,1) not null,
FirstName varchar(32) not null,
LastName varchar(64) not null,
FullName varchar(97) not null,
constraint PK_Customers_OnDisk
primary key clustered(CustomerId)
);
create nonclustered index IDX_Customers_OnDisk_LastName_FirstName
on dbo.Customers_OnDisk(LastName, FirstName);
create nonclustered index IDX_Customers_OnDisk_FullName
on dbo.Customers_OnDisk(FullName);
go
;with FirstNames(FirstName)
as
(
select Names.Name
from
(
values('Andrew'),('Andy'),('Anton'),('Ashley')
,('Boris'),('Brian'),('Cristopher'),('Cathy')
,('Daniel'),('Don'),('Edward'),('Eddy'),('Emy')
,('Frank'),('George'),('Harry'),('Henry'),('Ida')
,('John'),('Jimmy'),('Jenny'),('Jack'),('Kathy')
,('Kim'),('Larry'),('Mary'),('Max'),('Nancy'),
('Olivia'),('Paul'),('Peter'),('Patrick'),('Robert'),
('Ron'),('Steve'),('Shawn'),('Tom'),('Timothy'),
('Uri'),('Victor')
) Names(Name)
)
,LastNames(LastName)
as
(
select Names.Name
from
(
values('Smith'),('Johnson'),('Williams'),('Jones')
,('Brown'),('Davis'),('Miller'),('Wilson')
,('Moore'),('Taylor'),('Anderson'),('Jackson')
,('White'),('Isakov')
) Names(Name)
)
insert into dbo.Customers(LastName, FirstName, FullName)
select LastName, FirstName, FirstName + ' ' + LastName
from FirstNames cross join LastNames;
insert into dbo.Customers_OnDisk(LastName, FirstName, FullName)
select LastName, FirstName, FullName
from dbo.Customers;
```
清单 5-4. 非聚集索引与排序顺序：基于磁盘的表创建

