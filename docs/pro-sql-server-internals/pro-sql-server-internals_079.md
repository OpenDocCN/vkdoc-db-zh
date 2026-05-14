# 第 26 章：计划缓存

#### 参数嗅探

在存储过程中。此时，表中不存在 `Country='USA'` 的行，重新编译会产生次优的执行计划，如图 26-5 所示。另外值得注意的是，由于更新操作引入的索引碎片，该查询的读取次数比之前更多。

**图 26-5.** 参数嗅探：`OPTIMIZE FOR` 与数据分布变化

SQL Server 2008 引入了另一个优化提示 `OPTIMIZE FOR UNKNOWN`，它有助于处理此类情况。使用此提示，SQL Server 会基于表中统计上最常见的值进行优化。代码清单 26-8 展示了执行此操作的代码。

**代码清单 26-8.** 参数嗅探：`OPTIMIZE FOR UNKNOWN` 提示

```sql
alter proc dbo.GetAverageSalary @Country varchar(64)
as
select Avg(Salary) as [Avg Salary]
from dbo.Employees
where Country = @Country
option (optimize for(@Country UNKNOWN));
go

exec dbo.GetAverageSalary @Country='Canada';
```

图 26-6 展示了执行计划。*Germany* 是表中统计上最常见的值，因此 SQL Server 生成了一个针对该参数值最优的执行计划。

**图 26-6.** 参数嗅探：`OPTIMIZE FOR UNKNOWN` 提示

## 使用局部变量

你可以通过使用局部变量代替参数，达到与 `OPTIMIZE FOR UNKNOWN` 提示相同的效果。此方法在 SQL Server 2005 中也有效，因为该版本不支持 `OPTIMIZE FOR UNKNOWN` 提示。代码清单 26-9 说明了这种方法。它引入的执行计划与图 26-6 中的相同——即包含 *聚集索引扫描* 的计划。

**代码清单 26-9.** 参数嗅探：使用局部变量

```sql
alter proc dbo.GetAverageSalary @Country varchar(64)
as
declare
@CountryTmp varchar(64) = @Country;
select Avg(Salary) as [Avg Salary]
from dbo.Employees
where Country = @CountryTmp;
```

## 数据库作用域配置与查询存储

SQL Server 2016 允许你通过数据库作用域配置，在数据库级别控制参数嗅探，使用的是 `ALTER DATABASE SCOPED CONFIGURATION SET PARAMETER_SNIFFING` 命令。禁用参数嗅探等同于对所有查询使用 `OPTIMIZE FOR UNKNOWN` 提示。另一个命令 `ALTER DATABASE SCOPED CONFIGURATION CLEAR PROCEDURE_CACHE` 可用于清除数据库的存储过程计划缓存。

你可以通过 `sys.dm_exec_query_stats` 视图和 `sys.dm_exec_query_plan` 函数分析缓存的计划，来排查由参数嗅探引发的问题。我们将在本章后面及第 28 章中更详细地讨论这一点，包括如何获取当前正在运行的语句的执行计划。

SQL Server 2016 引入了一个名为 *查询存储* 的新组件，它允许你捕获系统中查询的执行计划和运行时统计信息。此外，它通过允许你为查询强制指定特定的执行计划，来帮助你避免参数嗅探问题。我们将在本书第 29 章详细讨论查询存储。

#### 计划重用

SQL Server 缓存的计划对于未来调用该计划时的所有参数组合都必须是有效的。在某些情况下，这可能导致缓存的计划对于特定的参数值组合而言是次优的。

经常导致这种情况的代码模式之一，是实现基于一组可选参数搜索数据的存储过程。此类存储过程的一个典型实现如代码清单 26-10 所示。此代码还在 `dbo.Employees` 表上创建了两个非聚集索引。

**代码清单 26-10.** 计划重用：创建存储过程和索引

```sql
create proc dbo.SearchEmployee
( @Number varchar(32) = null, @Name varchar(100) = null )
as
select Id, Number, Name, Salary, Country
from dbo.Employees
where
```

