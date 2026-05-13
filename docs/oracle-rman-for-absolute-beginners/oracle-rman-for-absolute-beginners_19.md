# 恢复当前重做日志组所有成员丢失后的状态

如果您数据库中当前联机重做日志组的所有成员都发生了介质故障，那么（不幸的是）当您丢失当前联机重做日志组的所有成员时，您的备选方案是有限的。以下是一些可能的选项：

*   执行不完全恢复到最后一个完好的 `SCN`。
*   如果启用了闪回，将数据库闪回到最后一个完好的 `SCN`。
*   如果您正在使用 Oracle Data Guard，则故障切换到您的物理或逻辑备用数据库。
*   联系 Oracle 支持服务寻求建议。

为不完全恢复做准备，首先通过查询 `V$LOG` 中的 `FIRST_CHANGE#` 列来确定最后一个完好的 `SCN`。在此场景中，您仅丢失了当前联机重做日志。因此，您可以执行不完全恢复，恢复到但不包括当前联机重做日志的 `FIRST_CHANGE#` `SCN`。

```
SQL> shutdown immediate;
SQL> startup mount;
```

现在发出此查询：

```sql
SELECT group#, status, archived,
thread#, sequence#, first_change#
FROM v$log;
```

以下是示例输出：

```
GROUP# STATUS           ARC  THREAD#  SEQUENCE# FIRST_CHANGE#
------ ---------------- --- -------- ---------- -------------
     1 INACTIVE         YES        1         86        533781
     2 INACTIVE         YES        1         85        533778
     3 CURRENT          NO         1         87        533784
```

在此情况下，您可以恢复并恢复到但不包括 `SCN` 533784。操作方法如下：

```
RMAN> restore database until scn 533784;
RMAN> recover database until scn 533784;
RMAN> alter database open resetlogs;
```

![Image](img/sq.jpg) `注意` 有关不完全恢复和/或闪回数据库的完整详细信息，请参见第 6 章。

丢失当前联机重做日志组的所有成员可以说是可能发生在您数据库上的最糟糕的事情。如果您遭遇当前联机组所有成员的介质故障，那么您很可能会丢失这些日志中包含的所有事务。在这种情况下，您必须先执行不完全恢复，然后才能打开数据库。

![Image](img/sq.jpg) `提示` 如果您迫切需要恢复受损的当前联机重做日志文件中丢失的事务，请联系 Oracle 支持服务以探索所有选项。

### 总结

理解如何处理联机重做日志故障至关重要。Oracle 没有提供在联机重做日志发生介质故障时自动修复问题的方法。因此，了解如何诊断和解决出现的任何问题非常重要。联机重做日志的故障很少见，但当它们确实发生时，您现在将能更好地做好准备，以高效有效地解决任何复杂情况。

