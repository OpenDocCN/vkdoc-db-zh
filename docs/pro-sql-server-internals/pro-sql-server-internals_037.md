# 第 2 章 ■ 表与索引：内部结构与访问方法

找到大量行，进而需要大量键查找操作，会导致大量 I/O 操作，这使得非聚集索引的使用效率低下。

导致非聚集索引效率低下的还有另一个重要因素。键查找操作需要从数据文件的不同位置读取数据。尽管来自根级别和中间索引级别的数据页通常会被缓存，仅引入逻辑读取，但访问叶级页会导致随机物理 I/O 活动。相比之下，索引扫描会触发顺序 I/O 活动，这比随机 I/O 更高效，尤其是在磁盘驱动器的情况下。

## 键查找与 RID 查找

在堆表上定义的非聚集索引引用行在数据文件中的实际位置。SQL Server 使用 `RID lookup` 操作从堆中获取数据行。理论上，`RID lookup` 似乎比键查找更高效，因为它可以直接读取行，而无需遍历聚集索引的根级别和中间级别。

然而，在实际中，读取非叶级聚集索引数据页的性能影响相对较小。这些页面通常缓存在缓冲池中，访问时不需要物理 I/O。逻辑读取仍然会带来一些开销；但与物理 I/O 和磁盘访问相比，这种开销通常微不足道。此外，堆表中的转发指针可能在单个 `RID lookup` 操作中引入多次物理读取，这会影响其性能。

因此，当预计需要大量键或 `RID lookup` 操作时，SQL Server 在选择非聚集索引时会非常保守。为了说明这一点，我们创建一个表并使用清单 2-10 所示的数据填充它。

### 清单 2-10. 非聚集索引用法：创建测试表

```sql
create table dbo.Books
(
    BookId int identity(1,1) not null,
    Title nvarchar(256) not null,
    -- International Standard Book Number
    ISBN char(14) not null,
    Placeholder char(150) null
);

create unique clustered index IDX_Books_BookId on dbo.Books(BookId);

-- 1,252,000 rows
;with Prefix(Prefix)
as
(
    select 100
    union all
    select Prefix + 1
    from Prefix
    where Prefix < 600
)
,Postfix(Postfix)
as
(
    select 100000001
    union all
    select Postfix + 1
    from Postfix
    where Postfix < 100002500
)
insert into dbo.Books(ISBN, Title)
select
    convert(char(3), Prefix) + '-0' + convert(char(9),Postfix)
    ,'Title for ISBN' + convert(char(3), Prefix) + '-0' + convert(char(9),Postfix)
from Prefix cross join Postfix
option (maxrecursion 0);

create nonclustered index IDX_Books_ISBN on dbo.Books(ISBN);
```

此时，表中有 1,252,000 行。`ISBN` 列的数据格式如下：`<Prefix>-<Postfix>`，其中前缀从 100 到 600，每个前缀有 2,500 个后缀。

让我们尝试选择其中一个前缀的数据，如清单 2-11 所示。

### 清单 2-11. 非聚集索引用法：为单个前缀选择数据

```sql
-- 2,500 rows
select * from dbo.Books where ISBN like '210%'
```

如图 2-20 所示，SQL Server 决定使用带键查找的 `nonclustered index seek` 作为执行计划。选择 2,500 行引入了 7,676 次逻辑读取。聚集索引 `IDX_Books_BookId` 有三个级别，这导致在键查找操作期间发生了 7,500 次读取。剩下的 176 次读取是在 SQL Server 遍历索引树并在范围扫描操作中读取页时，在非聚集索引上执行的。

### 图 2-20. 为单个前缀选择数据：执行计划

下一步，让我们选择五个不同前缀的数据。我们将运行两个不同的查询。在第一个查询中，我们将让 SQL Server 自由选择它想要的执行计划。在第二个查询中，


