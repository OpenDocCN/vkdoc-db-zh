# 第 31 章 ■ 备份与恢复

你可以使用 `STOPAT` 选项将数据库恢复到某个特定时间点。该选项接受一个日期/时间值或变量作为参数，并将数据库恢复至该时刻的状态。或者，你也可以使用 `STOPATMARK` 和 `STOPBEFOREMARK` 选项，它们允许你通过停在某个特定的 LSN（日志序列号）或命名事务来恢复数据库。

这些选项的一个常见用例是恢复意外删除的对象。让我们看一下代码清单 31-5 中的示例，创建一个包含表 `dbo.Invoices` 的数据库，填充一些数据，并执行一次完整的数据库备份。

## 代码清单 31-5. 时间点恢复：数据库创建

```sql
create database MyDB
go
create table MyDB.dbo.Invoices(InvoiceId int not null);
insert into MyDB.dbo.Invoices values(1),(2),(3) ;
go
backup database MyDB
to disk = N'c:\backups\MyDB.bak'
with noformat, init,
name = N'MyDB-Full Database Backup', stats = 5;
```

现在，假设有人意外地使用 `DROP TABLE dbo.Invoices` 命令删除了 `dbo.Invoices` 表。如果数据库处于活动状态，并且其他数据随时间已被修改，那么最佳操作是从备份中恢复数据库的另一个副本到该表被删除的时间点，然后将数据从新恢复的副本复制到原始数据库。

作为恢复过程的第一步，让我们按代码清单 31-6 所示备份事务日志。显然，在真实系统中，你可能已经拥有覆盖表被删除时刻的日志备份。

## 代码清单 31-6. 时间点恢复：备份日志

```sql
backup log MyDB
to disk = N'c:\backups\MyDB.trn'
with noformat, init,
name = N'MyDB-Transaction Log Backup', stats = 5;
```

棘手的部分是找出表被删除的时间。你有一个选择是分析系统默认跟踪，它捕获了此类事件。你可以使用 `fn_trace_gettable` 系统函数，如代码清单 31-7 所示。

## 代码清单 31-7. 时间点恢复：分析系统跟踪

```sql
declare
@TraceFilePath nvarchar(2000)
select @TraceFilePath = convert(nvarchar(2000),value)
from ::fn_trace_getinfo(0)
where traceid = 1 and property = 2;
select
StartTime, EventClass
,case EventSubClass
when 0 then 'DROP'
when 1 then 'COMMIT'
when 2 then 'ROLLBACK'
end as SubClass
,ObjectID, ObjectName, TransactionID
from ::fn_trace_gettable(@TraceFilePath, default)
where EventClass = 47 and DatabaseName = 'MyDB'
order by StartTime desc
```

如图 31-2 所示，输出中有两行。其中一行对应对象被删除的时间。另一行则与事务提交的时间相关。

## 图 31-2. 默认系统跟踪的输出

你可以使用输出中的时间来指定 `RESTORE` 命令的 `STOPAT` 参数，如代码清单 31-8 所示。也可以在 Management Studio 的数据库恢复 UI 中执行时间点恢复。然而，该选项不允许你在 `STOPAT` 值中指定毫秒。

## 代码清单 31-8. 时间点恢复：使用 STOPAT 参数

```sql
restore database MyDBCopy
from disk = N'C:\Backups\MyDB.bak' with file = 1,
move N'MyDB' to N'c:\db\MyDBCopy.mdf',
move N'MyDB_log' to N'c:\db\MyDBCopy.ldf',
norecovery, stats = 5;
restore log MyDBCopy
from disk = N'C:\Backups\MyDB.trn' with file = 1,
norecovery, stats = 5,
stopat = N'2016-03-10T06:21:27.697';
restore database MyDBCopy with recovery;
```

虽然系统默认跟踪是一个非常简单的选项，但它也有一个缺点。跟踪中事件的时间不够精确，可能与你需要指定的时间相差几毫秒。



作为 `STOPAT` 值。因此，无法保证你能恢复到删除时刻最新的表数据。此外，`DROP OBJECT` 事件可能已被覆盖，或者服务器上的跟踪功能可能已被禁用。

针对此问题的一个变通方法是使用一个未公开的系统函数 `fn_dump_dblog`，它会返回事务日志备份文件的内容。你需要找到属于 `DROP TABLE` 语句的 `LSN`，然后使用 `STOPBEFOREMARK` 选项来恢复数据库的一个副本。清单 31-9 展示了调用 `fn_dump_dblog` 函数的代码。图 31-3 展示了查询的输出结果。

***清单 31-9.*** 时间点恢复：使用 `fn_dump_dblog` 函数

```sql
select [Current LSN], [Begin Time], Operation,[Transaction Name], [Description]
from fn_dump_dblog
( default, default, default, default, 'C:\backups\mydb.trn',default, default, default
,default, default, default, default, default, default, default, default, default, default
,default, default, default, default, default, default, default, default, default, default
,default, default, default, default, default, default, default, default, default, default
,default, default, default, default, default, default, default, default, default, default
,default, default, default, default, default, default, default, default, default, default
,default, default, default, default, default, default, default, default, default, default )
where [Transaction Name] = 'DROPOBJ';
```

***图 31-3.** `fn_dump_dblog` 输出*

清单 31-10 展示了一个使用上述输出中 `LSN` 的 `RESTORE` 语句。你应该在 `STOPBEFOREMARK` 参数中指定 `lsn:0x` 前缀。这告诉 SQL Server 你使用的是十六进制格式的 `LSN`。

***清单 31-10.*** 时间点恢复：使用 `STOPBEFOREMARK` 参数

```sql
restore log MyDBCopy
from disk = N'C:\Backups\MyDB.trn'
with file = 1, norecovery, stats = 5,
stopbeforemark = 'lsn:0x 00000024:00000178:0001';
```

分析事务日志记录是一项繁琐且耗时的工作。然而，它能提供最准确的结果。此外，当数据因 `DELETE` 语句被意外删除时，你可以使用这种技术。此类操作不会被系统默认跟踪记录，分析事务日志内容是唯一可用的选择。幸运的是，有一些第三方工具可以简化在日志中搜索操作 `LSN` 的过程。

