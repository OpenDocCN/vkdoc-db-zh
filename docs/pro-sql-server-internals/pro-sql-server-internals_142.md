# 第 31 章 ■ 备份与恢复

## 使用 STANDBY 进行恢复

当你使用 `NORECOVERY` 选项完成恢复过程后，数据库将保持在 `RESTORING` 状态，并且对用户不可用。`STANDBY` 选项允许你以只读模式访问数据库。

如前所述，SQL Server 在恢复过程的最后一步执行恢复的重做阶段。恢复的撤消阶段则会延迟，直到使用 `RECOVERY` 选项调用恢复时才会执行。`STANDBY` 选项强制 SQL Server 使用一个临时的*撤消文件*来执行撤消阶段，该文件用于存储撤消过程中生成的补偿日志记录。这些补偿日志记录不会成为数据库事务日志的一部分，如果需要，你可以恢复额外的日志备份或恢复数据库。

清单 31-11 展示了 `RESTORE WITH STANDBY` 操作符的用法。值得一提的是，在此模式下不应指定 `RECOVERY`/`NORECOVERY` 选项。

***清单 31-11.*** 使用 `STANDBY` 选项进行恢复

```sql
restore log MyDBCopy
from disk = N'C:\Backups\MyDB.trn'
with file = 1, stats = 5,
standby = 'C:\Backups\undo.trn';
```

`STANDBY` 选项可以与时间点恢复结合使用。这可以帮助你在需要确定用于 `STOPBEFOREMARK` 选项的 `LSN` 时避免不必要的恢复操作。设想一种情况，日志文件中有多个 `DROP OBJECT` 事务，而你不知道哪一个...


