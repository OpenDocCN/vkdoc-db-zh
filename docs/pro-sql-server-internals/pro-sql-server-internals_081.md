# 第 26 章 ■ 计划缓存

#### 即席查询的计划缓存

```sql
case when @Number is not null then N' and Number=@Number' else N'' end +

case when @Name is not null then N' and Name=@Name' else N'' end;

exec sp_executesql @Sql, N'@Number varchar(32), @Name varchar(100)'

,@Number=@Number, @Name=@Name;
```

■ `重要提示` 始终在 `sp_executesql` 存储过程中使用参数，以避免 SQL 注入。

在使用筛选索引时请记住此行为。在某些参数值组合下无法使用筛选索引时，SQL Server 将不会生成并缓存使用该筛选索引的计划。清单 26-14 展示了一个示例。当 `@Processed` 参数设置为零时，SQL Server 不会生成使用 `IDX_Data_UnprocessedData` 索引的计划，因为这个计划对于非零的 `@Processed` 参数值是无效的。

***清单 26-14.*** 计划重用：筛选索引（非功能性演示）

```sql
create unique nonclustered index IDX_Data_UnprocessedData
on dbo.RawData(ID)
include(Processed)
where Processed = 0;

-- 此查询的缓存计划将不会使用筛选索引
select top 100 *
from dbo.RawData
where ID > @ID and Processed = @Processed
order by ID;
```

SQL Server 会缓存那些在 WHERE 子句中使用常量而非参数的即席查询（和批处理）的计划。清单 26-15 展示了即席查询的示例。

***清单 26-15.*** 即席查询

```sql
select * from dbo.Customers where LastName='Smith'
go
select * from dbo.Customers where LastName='Smith'
go
SELECT * FROM dbo.Customers WHERE LastName='Smith'
go
select * from dbo.Customers where LastName = 'Smith'
go
```

SQL Server 仅在查询完全相同且字符完全匹配的情况下才会重用即席查询的计划。例如，清单 26-15 中的四个查询会引入三个不同的计划。第一个和第二个查询是相同的，共享一个计划。其他两个查询由于 WHERE 子句中关键字大小写不匹配以及等号运算符周围的额外空格字符，将不会重用该计划。

由于即席查询的性质，它们不经常重用计划。不幸的是，即席查询的缓存计划可能会消耗大量内存。让我们看一个例子，运行 1000 个简单的即席批处理，如清单 26-16 所示，之后检查计划缓存状态。该脚本使用 `DBCC FREEPROCCACHE` 命令清除缓存内容；请勿在生产服务器上运行此命令。

***清单 26-16.*** 即席查询的内存使用情况：运行即席查询

```sql
dbcc freeproccache
go

declare
    @SQL nvarchar(max)
    ,@I int = 0

while @I < 1000
begin
    select @SQL =
        N'declare @C int;select @C=ID from dbo.Employees where ID='
        + convert(nvarchar(10),@I);

    exec(@SQL);

    select @I += 1;
end
go

select
    p.usecounts, p.cacheobjtype, p.objtype, p.size_in_bytes, t.[text]
from
    sys.dm_exec_cached_plans p cross apply
    sys.dm_exec_sql_text(p.plan_handle) t
where
    p.cacheobjtype like 'Compiled Plan%' and
    t.[text] like '%Employees%'
order by
    p.objtype desc;
```

如图 26-9 所示，缓存了 1000 个计划，每个计划使用 32 KB 内存，总计 32 MB。可以想见，在繁忙系统中，即席查询可能导致计划缓存内存使用过度。

***图 26-9.** 查询执行后的计划缓存内容*

SQL Server 2008 引入了一个名为 `优化即席工作负载` 的服务器端配置设置。启用此设置后，SQL Server 会缓存小于 300 字节的小型结构，称为 `已编译计划存根`，而不是实际的编译计划。已编译计划存根是一个占位符，用于跟踪哪些即席查询已被执行。当相同的查询第二次运行时，SQL Server 会用实际的编译计划替换已编译计划存根，并在后续运行中重用它。


