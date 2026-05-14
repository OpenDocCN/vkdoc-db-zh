# 第 30 章 ■ 事务日志内部机制

`图 30-15.` `log_reuse_wait_desc` 输出

对于处于 `FULL` 或 `BULK LOGGED` 恢复模型的数据库，事务日志未被截断的最常见原因之一是缺少日志备份。一个常见的误解是 `FULL` 数据库备份会截断事务日志。事实并非如此，你必须执行日志备份才能做到这一点。`log_reuse_wait_desc` 的值为 `LOG_BACKUP` 表示你需要执行日志备份以截断事务日志。

`log_reuse_wait_desc` 的值为 `ACTIVE_TRANSACTION` 表示系统中存在长时间运行和/或未提交的事务。`SQL Server` 无法截断事务日志中超过 LSN（日志序列号）



