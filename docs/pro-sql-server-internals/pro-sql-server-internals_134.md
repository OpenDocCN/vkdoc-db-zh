# 第 30 章 ■ 事务日志内部机制

最早的未提交事务，无论当前使用的是何种数据库恢复模型。

清单 30-3 中的查询返回当前数据库中五个最早未提交事务的列表。它返回事务开始的时间、会话信息以及日志使用统计信息。顺带一提，如果你需要找出消耗最多日志空间的事务，可以使用相同的查询，并将 `order by` 子句改为按 `Log Used` 列排序。

***清单 30-3.*** 查找系统中五个最早的活动事务

```sql
select top 5
ses_tran.session_id as [Session Id], es.login_name as [Login], es.host_name as [Host]
,es.program_name as [Program], es.login_time as [Login Time]
,db_tran.database_transaction_begin_time as [Tran Begin Time]
,db_tran.database_transaction_log_record_count as [Log Records]
,db_tran.[database_transaction_log_bytes_used] as [Log Used]
,db_tran.[database_transaction_log_bytes_reserved] as [Log Rsrvd]
,sqlText.text as [SQL], qp.query_plan as [Plan]
from
sys.dm_tran_database_transactions db_tran join
sys.dm_tran_session_transactions ses_tran on
db_tran.transaction_id = ses_tran.transaction_id
join sys.dm_exec_sessions es on
es.[session_id] = ses_tran.[session_id]
left outer join sys.dm_exec_requests er on
er.session_id = ses_tran.session_id
join sys.dm_exec_connections ec on
ec.session_id = ses_tran.session_id
cross apply
sys.dm_exec_sql_text (ec.most_recent_sql_handle) sqlText
cross apply
sys.dm_exec_query_plan (er.plan_handle) qp
where
db_tran.database_id = DB_ID()
order by
db_tran.database_transaction_begin_time;
```

正如我已经提到的，SQL Server 有许多读取事务日志的进程，例如事务复制、变更数据捕获、数据库镜像、AlwaysOn 可用性组等。当存在积压时，这些进程中的任何一个都可能阻止事务日志截断。虽然在一切正常运行时这种情况很少发生，但如果出现错误，你可能会遇到此问题。

这种情况的一个常见例子是可用性组或数据库镜像会话中存在无法访问的辅助节点。尚未发送到辅助节点的日志记录将保留在活动事务日志中。这会阻止其截断。`Log_reuse_wait_desc` 列的值将指示此情况。

■ **注意** 你可以在 [`technet.microsoft.com/en-us/library/ms178534.aspx`](http://technet.microsoft.com/en-us/library/ms178534.aspx) 查看可能的 `log_reuse_wait_desc` 值列表。

如果你遇到 9002 *事务日志已满* 错误，关键点是不要惊慌。你能做的最糟糕的事情是执行导致数据库事务不一致的操作。例如，关闭 SQL Server 或分离数据库后删除事务日志文件就会导致这种情况。如果数据库未被干净地关闭，SQL Server 可能无法恢复它，因为事务日志将丢失。

创建另一个日志文件可能是解决此问题最快、最简单的方法；然而，从长远来看，这几乎不是最佳选择。多个日志文件会使数据库管理复杂化。此外，很难删除日志文件。如果日志文件存储了日志的活动部分，SQL Server 不允许你删除它们。

你必须了解事务日志无法被截断的原因，并采取相应的措施。根据问题的根本原因，你可以执行日志备份、识别并终止保持未提交活动事务的会话，或从可用性组中移除无法访问的辅助节点。

#### 事务日志管理

手动管理事务日志大小比允许 SQL Server 自动增长它更好。不幸的是，确定最佳日志大小并不总是容易的。一方面，你希望事务日志足够大


## SQL Server 事务日志管理最佳实践

事务日志的大小应足以避免自动增长事件的发生。另一方面，您又希望保持日志较小，以节省磁盘空间，并减少从备份还原数据库时零初始化日志所需的时间。

如果您使用任何高可用性或其他依赖于事务日志记录的技术，还应该在日志文件中保留一些空间。如果那些进程出现问题，SQL Server 将无法在日志备份期间截断事务日志。此外，您应实施监控和告警框架，以便在事务日志填满之前向您发出此类情况的警报，并给您留出时间做出反应。

另一个重要因素是日志文件中的 `VLF` 数量。您应避免事务日志过度碎片化并包含大量小型 `VLF` 的情况。同样，您也应避免日志中 `VLF` 数量过少但每个都非常大的情况。

对于需要大型事务日志的数据库，您可以使用 4,000 `MB` 的块预分配空间，这会生成 16 个每个 250 `MB` 的 `VLF`。如果数据库不需要大型（超过 4,000 `MB`）事务日志，您可以根据大小需求通过一次操作预分配日志空间。

> **注意：** 在 SQL Server 2005–2008R2 中存在一个错误，如果事务日志大小是 4 `GB` 的倍数，它会错误地增长。您可以改用 4,000 `MB` 的倍数。此错误已在 SQL Server 2012 以及旧版 SQL Server 的累积更新/服务包中得到修复。

在紧急情况下，您仍然应该允许 SQL Server 自动增长事务日志。然而，选择合适的自动增长大小是棘手的。对于具有大型事务日志的数据库，明智的做法是使用 4,000 `MB` 以减少 `VLF` 的数量。但是，零初始化 4,000 `MB` 的新分配空间可能非常耗时。请记住，即使启用了`Instant File Initialization`，SQL Server 也始终会对事务日志进行零初始化。在自动增长过程中，所有写入日志文件的数据库活动都会被阻止。这是手动管理事务日志大小的另一个理由。

> **提示：** 使用多大的自动增长大小取决于 I/O 子系统的性能。您应该分析零初始化需要多长时间，并找到一个平衡点，使自动增长时间和生成的 `VLF` 大小都是可接受的。在许多情况下，1 `GB` 的自动增长可能适用。

同样值得注意的是，`Management Studio` 的数据库创建对话框使用了低效的默认事务日志自动增长参数，这导致数据库中出现过多的 `VLF`。通过 `Management Studio` 创建新数据库时，您需要更改这些参数。幸运的是，这个问题在 SQL Server 2016 中得到了解决。

在数据修改的情况下，SQL Server 会同步写入事务日志。对于数据易变且事务日志活动繁重的 `OLTP` 系统，应将事务日志存储在具有良好写入性能和低延迟的磁盘阵列上。当数据是静态时（例如在数据仓库系统中），事务日志的 I/O 性能就不那么重要了。但是，您应该考虑它如何影响在那里刷新数据的进程的性能和持续时间。

最佳实践建议您将事务日志存储在专为顺序写入性能优化的专用磁盘阵列上。对于底层 I/O 子系统有足够能力支持多个高性能磁盘阵列的情况，这是一个很好的建议。然而，在某些情况下，当面临预算限制且没有足够磁盘驱动器时，通过将数据和日志文件存储在单个磁盘阵列上，您可以获得更好的 I/O 性能。然而，您应该记住，在磁盘阵列发生故障时，将数据和日志文件保存在同一个磁盘阵列上可能会导致数据丢失。

另一个重要因素是数据库的数量。当您放置来自多个...

