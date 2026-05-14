# 第 31 章 ■ 备份与还原

你可以指定多个目标备份文件，并允许 SQL Server 将备份**条带化**分布到所有这些文件中。如果备份驱动器的 I/O 性能成为瓶颈，这可以提高备份以及后续还原操作的性能。

`COPY_ONLY` 选项允许你在不中断日志链的情况下执行备份。该选项的一个可能用例是，当你需要将数据库副本迁移到开发环境中时。

SQL Server 将关于服务器实例上每个备份和还原操作的信息，存储在 `msdb` 数据库中定义的一组表中。这些表的描述超出了本书的范围。你可以在在线丛书中阅读文章“备份历史记录和标头信息”来了解更多细节，网址为：[`msdn.microsoft.com/en-us/library/ms188653.aspx`](http://msdn.microsoft.com/en-us/library/ms188653.aspx)。

最后，SQL Server 会将每个备份的信息写入错误日志文件。如果备份运行频繁，这可能会导致日志文件大小迅速膨胀。你可以使用跟踪标志 `T3226` 来禁用此行为。这会使错误日志更紧凑，但代价是需要查询 `msdb` 来获取备份历史记录。

## 还原数据库

你可以使用 `RESTORE DATABASE` 命令来还原数据库。你可以在代码清单 31-4 中看到此命令的实际示例。它在新的目的地还原 `OrderEntryDB` 数据库（由 `MOVE` 选项控制此操作），并在之后应用差异备份和事务日志备份。

### 代码清单 31-4. 还原数据库

```sql
-- 初始完整备份
restore database OrderEntryDbDev
from disk = N'C:\Backups\OrderEntry.bak' with file = 1,
move N'OrderEntryDB' to N'c:\backups\OrderEntryDB.mdf',
move N'OrderEntryDB_log' to N'c:\backups\OrderEntryDB_log.ldf',
norecovery, nounload, stats = 5;

-- 差异备份
restore database OrderEntryDbDev
from disk = N'C:\Backups\OrderEntry.bak' with file = 2,
norecovery, nounload, stats = 5;

-- 事务日志备份
restore log OrderEntryDbDev
from disk = N'C:\Backups\OrderEntry.trn'
with nounload, norecovery, stats = 10;

restore database OrderEntryDbDev with recovery;
```

当备份文件存储了多个备份时，你应该使用 `WITH FILE` 选项来指定文件编号。正如我之前提到的，使用此方法时要小心，并确保你的备份例程不会意外覆盖文件中的现有备份。

每个 `RESTORE` 操作都应指定一个数据库恢复选项。当使用 `RECOVERY` 选项还原备份时，SQL Server 会通过执行重做和撤销两个恢复阶段来恢复数据库，并使其可供用户使用。之后不能再还原其他备份。或者，`NORECOVERY` 选项仅执行数据库恢复的重做阶段，并使数据库保持在 `RESTORING` 状态。这允许你从日志链中还原更多的备份。

