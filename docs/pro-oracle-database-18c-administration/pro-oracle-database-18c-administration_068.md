# 执行基于 SCN 的恢复

基于 SCN 的不完全数据库恢复适用于您知道希望恢复会话结束的 SCN 值的情况。RMAN 将恢复到但不包括指定的 SCN。当恢复过程达到指定的 SCN 时，RMAN 会自动终止恢复过程。

您可以通过多种方式查看数据库 SCN 信息：

*   使用 LogMiner 确定与 DDL 或 DML 语句关联的 SCN
*   查看 `alert.log` 文件
*   查看您的跟踪文件
*   查询 `V$LOG`、`V$LOG_HISTORY` 和 `V$ARCHIVED_LOG` 的 `FIRST_CHANGE#` 列

确定要恢复到的 SCN 后，使用 `UNTIL SCN` 子句恢复到但不包括指定的 SCN。以下示例恢复所有 SCN 小于 95019865425 的事务：

```
$ rman target /
RMAN> startup mount;
RMAN> restore database until scn 95019865425;
RMAN> recover database until scn 95019865425;
RMAN> alter database open resetlogs;
```

如果一切顺利，您应该会看到类似以下的输出：

```
Statement processed
```

