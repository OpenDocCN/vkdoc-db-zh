# 恢复活动重做日志组所有成员丢失后的状态

```sql
alter database add logfile group 2
('/u01/oraredo/O12C/redo02a.rdo',
 '/u02/oraredo/O12C/redo02b.rdo') SIZE 50M reuse;
```

![Image](img/sq.jpg) `提示` 有关移动联机重做日志组的示例，请参见第 2 章。

如果您数据库中某个活动的联机重做日志组的所有成员都发生了介质故障，那么在恢复该活动联机重做日志组时，请执行以下步骤：

1.  确认成员损坏情况。
2.  确认其状态为 `ACTIVE`。
3.  尝试发出一个检查点。
4.  如果检查点成功，状态应变为 `INACTIVE`，然后您可以清除该日志组。
5.  如果被清除的日志组尚未归档，请立即备份您的数据库。
6.  如果检查点不成功，那么您将必须执行不完全恢复。

检查目标数据库的 `alert.log` 文件，确认损坏情况。您应该会在 `alert.log` 文件中看到一条标识损坏成员的消息：

```
ORA-00312: online log 2 thread 1: '/u01/oraredo/O12C/redo02a.rdo'
ORA-00312: online log 2 thread 1: '/u02/oraredo/O12C/redo02b.rdo'
```

接下来，验证受损的日志组状态是否为 `ACTIVE`，如下所示：

```
$ sqlplus / as sysdba
SQL> startup mount;
```

运行以下查询：

```sql
SQL> select group#, status, archived, thread#, sequence# from v$log;
```

以下是示例输出：

```
GROUP# STATUS           ARC  THREAD#  SEQUENCE#
------ ---------------- --- --------
     1 CURRENT          NO         1         92
     2 ACTIVE           YES        1         91
     3 INACTIVE         YES        1         90
```

如果状态是 `ACTIVE`，则尝试发出 `alter system checkpoint` 命令，如下所示：

```sql
SQL> alter system checkpoint;
System altered.
```

如果检查点成功完成，则活动的日志组应被标记为 `INACTIVE`。成功的检查点确保所有已修改的数据库缓冲区都已写入磁盘，此时，只有包含在 `CURRENT` 联机重做日志中的事务才需要用于崩溃恢复。

![Image](img/sq.jpg) `注意` 如果检查点不成功，您将必须执行不完全恢复。有关此场景下的完整选项列表，请参见本章“恢复当前重做日志组所有成员丢失后的状态”一节。

如果状态是 `INACTIVE` 且日志已归档，您可以使用 `clear logfile` 命令重新创建日志组，如下所示：

```sql
SQL> alter database clear logfile group <group#>;
```

如果状态是 `INACTIVE` 且日志组尚未归档，则使用 `clear unarchived logfile` 命令重新创建它，如下所示：

```sql
SQL> alter database clear unarchived logfile group <group#>;
```

如果被清除的日志组之前未归档，立即创建数据库备份至关重要。有关创建数据库完整备份的详细信息，请参见第 5 章。

状态为 `ACTIVE` 的联机重做日志组仍然需要用于崩溃恢复。如果某个活动的联机重做日志组的所有成员都发生了介质故障，那么您必须尝试发出检查点。如果检查点成功，则可以清除该日志组。如果检查点不成功，则必须执行不完全恢复。

如果检查点成功且日志组未归档，那么该日志可能仍需要用于介质恢复。在此情况下，请尽快备份您的数据库。如果上次数据库备份是在该日志中的重做信息创建之前进行的，那么未归档的日志组可能仍需要用于介质恢复。这意味着如果您尝试执行介质恢复，您将无法恢复损坏日志文件中的任何信息或在该日志之后创建的任何事务。

