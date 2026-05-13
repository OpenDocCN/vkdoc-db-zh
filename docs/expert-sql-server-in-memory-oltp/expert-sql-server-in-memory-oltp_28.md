# 内存优化的表类型与变量

SQL Server 允许你创建内存优化的表类型。这些类型的表变量被称为**内存优化表变量**。与常规的基于磁盘的表变量不同，内存优化表变量仅存在于内存中，不使用 `tempdb`。

内存优化表变量提供了卓越的性能。它们可以用来替代基于磁盘的表变量，并且在某些情况下，可以替代临时表。显然，它们与内存优化表具有相同的功能限制集合。

与基于磁盘的表类型相反，你可以在内存优化的表类型上定义索引；然而，类似于基于磁盘的表变量，SQL Server 不会在这些索引上维护统计信息。幸运的是，正如所讨论的，由于内存优化表上索引的特性，基数估计错误所带来的负面影响要比基于磁盘的表小得多。

**注意**

使用 `option (recompile)` 进行语句级重编译，可以让 SQL Server 估计内存优化表变量中的行数。但是，它不会为 SQL Server 提供关于其中数据分布的任何信息。

SQL Server 不支持内存优化表变量的内联声明。例如，代码清单 9-15 中所示的代码将无法编译，并会引发错误。此限制背后的原因是，SQL Server 会为每个内存优化表类型编译一个 DLL，而这种情况下的内联声明是无效的。

```sql
declare
@IDList table
(
ID int not null
primary key nonclustered hash
with (bucket_count=10000)
)
with (memory_optimized=on)
Msg 319, Level 15, State 1, Line 91
Incorrect syntax near the keyword 'with'. If this statement is a common table expression, an xmlnamespaces clause or a change tracking context clause, the previous statement must be terminated with a semicolon.
```

**代码清单 9-15.**
（无效的）内存优化表变量的内联声明

你应该定义并使用一个内存优化的表类型，如代码清单 9-16 所示。

```sql
create type dbo.mtvIDList as table
(
ID int not null
primary key nonclustered hash
with (bucket_count=16384)
)
with (memory_optimized=on)
go
declare
@IDList dbo.mtvIDList
```

**代码清单 9-16.**
创建内存优化表类型与内存优化表变量

你可以在本机编译的和常规的 T-SQL 模块中，将内存优化表变量用作表值参数（TVP）。与基于磁盘的表值参数一样，这是将一批行传递给 T-SQL 例程的一种高效方式。

**注意**

我将在第 13 章中更详细地讨论将一批行传递给 T-SQL 例程的场景，以及使用内存优化表变量作为临时表替代品的场景。

你可以使用内存优化表变量来模拟使用游标的逐行处理，而游标在本机编译的存储过程中是不被支持的。代码清单 9-17 展示了一个使用内存优化表变量模拟静态游标的例子。显然，如果可能的话，最好避免使用游标，而是采用基于集合的逻辑。

```sql
create type dbo.MODataStage as table
(
ID int not null
primary key nonclustered
hash with (bucket_count=1024),
Value int null
)
with (memory_optimized=on)
go
create proc dbo.CursorDemo
with native_compilation, schemabinding, execute as owner
as
begin atomic
(
transaction isolation level = snapshot
,language=N'English'
)
declare
@tblCursor dbo.MODataStage
,@ID int = -1
,@Value int
,@RC int = 1
/* 将数据暂存到临时表中以模拟静态游标 */
insert into @tblCursor(ID, Value)
select ID, Value
from dbo.MOData
while @RC = 1
begin
select top 1 @ID = ID, @Value = Value
from @tblCursor
where ID > @ID
order by ID
select @RC = @@rowcount
if @RC = 1
begin
/* 行处理 */
update dbo.MOData set Value = Value * 2 where ID = @ID
end
end
end
```

**代码清单 9-17.**
使用内存优化表变量模拟游标

## 小结

SQL Server 使用本机编译来最小化解释型 T-SQL 语言的处理开销。它为每个内存优化对象生成单独的 DLL，并将其加载到进程内存中。

SQL Server 支持常规 T-SQL 存储过程、标量用户定义函数和触发器的本机编译。它在创建时，或者在服务器或数据库重启的情况下，在第一次调用时将它们编译成 DLL。SQL Server 针对 `UNKNOWN` 值优化了本机编译的模块，并将执行计划嵌入到代码中。该计划永远不会改变，除非模块被重新编译——无论是显式重新编译，还是在 SQL Server 或数据库重启后重新编译。如果初始编译后数据分布发生了显著变化，你应该重新编译该模块。

你可以通过修改模块或调用 `sp_recompile` 存储过程来重新编译模块。修改模块会在后台执行重新编译，在繁忙的系统中对工作负载的影响较小。

虽然本机编译的模块速度极快，但它们支持的 T-SQL 语言特性有限。你可以通过使用解释型 T-SQL 代码来规避这些限制，这种代码通过 SQL Server 的查询互操作组件访问内存优化表。在此模式下，几乎所有 T-SQL 语言特性都受支持。

内存优化表类型和内存优化表变量是表类型和表变量的内存版本。它们仅存在于内存中，不使用 `tempdb`。你可以使用内存优化表变量作为数据的暂存区，并用于将一批行传递给 T-SQL 例程。内存优化表类型允许你创建类似于内存优化表的索引。

