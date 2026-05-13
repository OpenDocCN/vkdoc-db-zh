# call script to source oracle OS variables
export ORACLE_SID=O12C
export ORACLE_HOME=/orahome/app/oracle/product/12.1.0.1/db_1
export PATH=$PATH:$ORACLE_HOME/bin
crit_var=$(
sqlplus -s <<EOF
/ as sysdba
SET HEAD OFF TERM OFF FEED OFF VERIFY OFF
COL value FORM A80
select value from v\$diag_info where name='Diag Trace';
EOF)
  if [ -r $crit_var/alert_$instance.log ]
  then
  grep -ic error $crit_var/alert_$instance.log
    if [ $? = 0 ]
    then
     mailx -s "Error in $instance log file" $MAILLIST <<EOF
Error in $crit_var/alert_$instance.log file on $BOX...
EOF
    fi # $?
  fi # -r
done # for instance
exit 0
```

您可以轻松修改以上脚本以适应您的环境要求。例如，您可能需要更改 Oracle 操作系统变量的加载方式、搜索的数据库、错误字符串以及电子邮件地址。这只是一个简单的示例，展示了使用 shell 脚本自动搜索文件中错误的强大功能。

## 丢失所有非活动重做日志组成员后的恢复

如果您丢失了某个非活动重做日志组的所有成员，请执行以下步骤：

1.  验证该组的所有成员是否都已损坏（通过检查`alert.log`文件）。
2.  验证日志组状态是否为`INACTIVE`。
3.  使用`clear logfile`命令重新创建该日志组。
4.  如果重新创建的日志组尚未归档，请立即备份数据库。

如果在线重做日志组的所有成员都已损坏，您将无法打开数据库。在这种情况下，Oracle 只允许您装载（mount）数据库。

首先检查您的`alert.log`文件，确认某个重做日志组的所有成员都已损坏。您应该会看到一条消息，指示某个在线重做日志组的所有成员已损坏，数据库无法打开：

```
ORA-00312: online log 2 thread 1: '/u01/oraredo/O12C/redo02a.rdo'
ORA-00313: open failed for members of log group 2 of thread 1
```

接下来，确保您的数据库处于装载（mount）模式：

```
$ sqlplus / as sysdba
SQL> startup mount;
```

然后，运行以下查询以验证损坏的日志组是否为`INACTIVE`状态，并确定其是否已归档：

```
SELECT group#, status, archived, thread#, sequence#
FROM v$log;
```

以下是一些示例输出：

```
    GROUP# STATUS           ARC    THREAD#  SEQUENCE#
---------- ---------------- --- ---------- ----------
         1 CURRENT          NO           1         25
         3 INACTIVE         NO           1         24
         2 INACTIVE         NO           1         23
```

如果状态为`INACTIVE`，则该日志组不再需要用于崩溃恢复（如表 7-3 所述）。因此，您可以使用`clear logfile`命令重新创建该日志组的所有成员。以下示例重新创建第 2 组的所有日志成员：

```
SQL> alter database clear logfile group 2;
```

如果该日志组尚未归档，那么您将需要使用`clear unarchived logfile`命令，如下所示：


SQL> alter database clear unarchived logfile group 2;
```

如果被清除的日志组之前未被归档，那么立即为你的数据库创建一个备份至关重要。关于如何对数据库进行完整备份的详细信息，请参阅第 5 章。

请记住，在前面的例子中，日志文件组编号为 2。你必须修改组号以匹配你实际场景中的组号。

如果在线重做日志组处于非活动状态并已归档，那么它的内容对于崩溃恢复或介质恢复来说就不是必需的。因此，可以使用 `clear logfile` 命令来重新创建一个组中的所有在线重做日志文件成员。

![Image](img/sq.jpg) **注意** `clear logfile` 命令会为你删除并重新创建日志组中的所有成员。即使你的数据库中只有两个日志组，你也可以执行此命令。

如果在线重做日志组尚未归档，那么它可能对介质恢复是必需的。在这种情况下，使用 `clear unarchived logfile` 命令来重新创建日志文件组成员。在此情况下，请尽快备份你的数据库。

如果上次数据库备份是在日志中的重做信息创建之前进行的，那么可能需要未归档的日志组来进行介质恢复。这意味着如果你尝试执行介质恢复，你将无法恢复损坏日志文件中的任何信息，或该日志之后创建的任何事务。

如果由于 I/O 错误导致 `clear logfile` 命令执行失败，并且这是一个永久性问题，那么你需要考虑删除该日志组并在其他位置重新创建。关于如何删除和重新创建日志文件组的指引，请参阅接下来的两个小节。

## 删除日志文件组

清除日志文件组（告诉 Oracle 重新创建日志文件）的替代方法是删除并重新创建日志文件组。如果你需要在不同位置重新创建日志文件组，因为原始位置已损坏或不可用，那么你可能需要这样做。

日志组必须处于非活动状态才能被删除。你可以按如下方式检查日志组的状态：

```sql
SQL> select group#, status, archived, thread#, sequence# from v$log;
```

你可以使用 `drop logfile group` 命令删除日志组：

```sql
SQL> alter database drop logfile group <group #>;
```

如果你尝试删除当前的在线日志组，Oracle 将返回 `ORA-01623` 错误，指出你不能删除当前组。使用 `alter system switch logfile` 命令来切换日志，使下一个组成为当前组。

日志切换后，之前是当前组的日志组只要包含 Oracle 执行崩溃恢复所需的重做信息，就会保持活动状态。如果你尝试删除一个处于活动状态的日志组，Oracle 将抛出 `ORA-01624` 错误，指出该日志组是崩溃恢复所必需的。发出 `alter system checkpoint` 命令以使日志组变为非活动状态。

此外，如果删除操作会导致你的数据库中只剩下一个日志组，那么你将无法发出 `drop logfile group` 命令。如果你尝试这样做，Oracle 将抛出 `ORA-01567` 错误，并告知你删除日志组是不被允许的，因为这将使你的数据库只剩下一个日志组（Oracle 最少需要两个日志组才能运行）。

## 添加日志文件组

你可以使用 `add logfile group` 命令添加新的日志组：

```sql
SQL> alter database add logfile group <group_#>
('/directory/file') SIZE <bytes> K|M|G;
```

你可以以字节、千字节、兆字节或千兆字节为单位指定日志文件的大小。以下示例添加了一个包含两个成员的日志组，每个成员大小为 50MB：

```sql
SQL> alter database add logfile group 2
('/u01/oraredo/O12C/redo02a.rdo',
 '/u02/oraredo/O12C/redo02b.rdo') SIZE 50M;
```

如果由于某些原因日志文件成员已存在于磁盘上，你可以使用 `reuse` 子句来覆盖它们：



