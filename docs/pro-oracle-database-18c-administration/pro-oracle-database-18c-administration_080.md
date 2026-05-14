# 自动化 DBA 任务示例

在当今往往混乱的商业环境中，自动化任务几乎是强制性的。如果不进行自动化，你可能会忘记执行某项任务；或者，如果手动执行工作，可能会在过程中引入错误。如果不自动化，你可能会发现自己被一个更高效或更便宜的 DBA 团队所取代。

当脚本失败时，收到一封电子邮件是合理的，但太多的成功任务邮件可能会造成干扰并导致错过失败通知。然而，无论成功或失败都收不到邮件，这实际上是一种失败，因为它处于一种不确定的状态。一种报告数据库运行脚本的方式是集中式报告，它从合并的日志或输出表的查询中显示成功或失败。审查每日邮件可以验证所有任务是否正常运行。

DBA 自动化执行各种各样的任务。几乎任何类型的环境都需要你创建某种操作系统脚本，这些脚本封装了操作系统命令、SQL 语句和 PL/SQL 块的组合。

本章中的以下脚本是 DBA 自动化的各种不同类型任务的一个示例。这套脚本绝非完整。其中许多脚本在你的环境中可能并不需要。重点是让你很好地了解自动化的任务类型以及完成特定任务所使用的技术。

**注意**
第 3 章包含 DBA 所需的一些核心脚本的基础示例。本节提供 DBA 通常自动化的任务和脚本示例。

## 启动和停止数据库及监听器

如果数据库服务器需要重启或重新启动，理想情况是 Oracle 数据库和监听器能随服务器自动重启。过去，这部分功能是通过`/etc/oratab`文件中的一个参数来实现数据库自动重启的。它会在`dbstart`和`dbshut`命令中被调用，设置为‘Y’表示重启，‘N’表示需要人工干预。

现在有了 Oracle Restart，它对于包含 RAC 和 ASM 的多组件环境尤其有用，但添加数据库作为重启的一部分也很简单。当使用`dbca`创建数据库时，数据库会被添加到 Oracle Restart 配置中。同样，当你使用`dbca`删除数据库时，数据库也会从 Oracle Restart 中移除。如果你使用 Oracle Net 配置助手（`netca`）创建或删除监听器，`netca`会将其添加到 Oracle Restart 或从中移除。

如果你不使用`dbca`，而是通过脚本或手动`createdb`步骤等其他方法添加数据库和监听器到 Oracle Restart，也有几种方式。数据库软件安装中包含一个版本的 Enterprise Manager 数据库控制，它只能将数据库和监听器添加到 Oracle Restart 配置。`srvctl`工具允许使用命令添加、修改和删除数据库及监听器。这些命令可以成为数据库创建脚本的一部分，从而使这个任务成为步骤之一，而不是之后的手动任务。

对于监听器，需要根据监听器是在 GRID 还是`ORACLE_HOME`中启动来设置`GRID_HOME`或`ORACLE_HOME`。默认监听器通过以下方式添加：

```bash
# cd $GRID_HOME/bin
# ./srvctl add listener
```

要添加另一个监听器，需要提供监听器的名称。

添加数据库：

```bash
# cd $ORACLE_HOME/bin
# ./srvctl add service -d o18c -o /u01/app/oracle/product/18.1.0/db_1
# -o 是 ORACLE_HOME 目录
# -d 是数据库名称
```

移除数据库：

```bash
# srvctl remove database -d o18c
```

也可以禁用某个服务，使其仍然可用但不随自动重启运行。禁用或启用数据库：

```bash
# srvctl disable database -d o18c
```

`srvctl`工具是 Oracle 软件的一部分，位于`ORACLE_HOME`中，因此不需要安装基础架构即可使用此方法将数据库和服务添加到 Oracle Restart。只要在重启配置中启用，Oracle Restart 将确保数据库在服务器重启时自动重启。

## 检查归档日志目标空间是否已满

有时，DBA 和系统管理员没有充分规划和实施磁盘上存储归档日志文件的位置。在这些情况下，通常需要一个脚本来检查主要位置的空间，并在归档日志目标空间即将填满前发出警告。此外，你可能还希望在脚本中实现，在归档日志位置自动启动一个 RMAN 任务来备份并删除归档日志以释放空间。

这类脚本在归档日志目标以不可预测的频率填满的混乱环境中非常有用。如果归档日志目标空间已满，数据库将挂起。在某些环境中，这是完全不可接受的。你可能会争辩说，永远不应该让自己陷入这种情况。因此，如果你被请来维护一个不可预测的环境，并且你是那个在凌晨 2 点接到电话的人，你可能需要考虑实现一个本节提供的脚本。

在使用以下脚本之前，请将脚本中的变量更改为与你的环境匹配。当空间低于`THRESH_GET_WORRIED`变量指定的数量时，脚本将发送警告邮件，并运行 RMAN 备份归档日志。

```bash
#!/bin/bash
PRG=`basename $0`
DB=$1
USAGE="Usage: ${PRG} "
if [ -z "$DB" ]; then
echo "${USAGE}"
exit 1
fi
# source OS 变量
. /var/opt/oracle/oraset ${DB}
# 设置开始关注的阈值
THRESH_GET_WORRIED=2000000 # df -k 显示的 2GB
MAILX="/bin/mailx"
MAIL_LIST="dkuhn@gmail.com "
BOX=`uname -a | awk '{print$2}'`
#
loc=`sqlplus -s <<EOF
CONNECT / AS sysdba
SET HEAD OFF FEEDBACK OFF
SELECT SUBSTR(destination,1,INSTR(destination,'/',1,2)-1)
FROM v\\$archive_dest WHERE dest_name='LOG_ARCHIVE_DEST_1';
EOF`
# df 的输出取决于你的 Linux/Unix 版本，
# 你可能需要根据输出调整下一行。
free_space=`df -k | grep ${loc} | awk '{print $4}'`
echo box = ${BOX}, sid = ${DB}, Arch Log Mnt Pnt = ${loc}
echo "free_space        = ${free_space} K"
echo "THRESH_GET_WORRIED= ${THRESH_GET_WORRIED} K"
#
if [ $free_space -le $THRESH_GET_WORRIED ]; then
$MAILX -s "Arch Redo Space Low ${DB} on $BOX" $MAIL_LIST <<EOF
Archive log dest space low running backup now,
box: $BOX, sid: ${DB}, free space: $free_space
EOF
# 运行 RMAN 备份归档日志并在备份后删除
rman nocatalog <<EOF
connect target /
backup archivelog all delete input;
EOF
else
echo no need to backup and delete, ${free_space} KB free on ${loc}
fi
#
exit 0
```

如果你使用 FRA（快速恢复区）作为归档日志文件的位置，可以从`V$ARCHIVED_LOG`视图中获取归档位置；例如，

```sql
SQL> select
substr(name,1,instr(name,'/',1,2)-1)
from v$archived_log
where first_time =
(select max(first_time) from v$archived_log);
```

使用 FRA 时，还有其他几种方法可以管理此空间，这绝对是使用 FRA 而不是仅仅设置目录的另一个原因。可以使用 SQL 从`v$recovery_file_dest`获取阈值，而不是查看文件系统。

```sql
SQL> select name, space_limit, space_used from v$recovery_file_dest;
NAME                SPACE_LIMIT  SPACE_USED
-------------------- ----------- ----------
/u02/oradata/FRA        1048576      48576
```



可以增加**快速恢复区**的大小，以容纳更多归档日志，直到通过备份和删除或清除旧备份释放空间。也可以更改快速恢复区的位置。自动化增加快速恢复区大小并运行备份的过程，以确保归档日志目录不会被填满，会更容易。可以在前面的脚本中的 RMAN 脚本之前插入以下内容：

```sql
SQL> alter system set DB_RECOVERY_FILE_DEST_SIZE=20G scope=both;
```

通常，检查归档日志目的地的脚本每小时运行一次。这是一个典型的 `cron` 条目（此条目实际上应为单行代码，但为了适应页面已放置在两行）：

```bash
38 * * * * /u01/oracle/bin/arch_check.bsh DWREP
1>/u01/oracle/bin/log/arch_check.log 2>&1
```

## 截断大型日志文件

有时日志文件可能变得非常大，并通过填满关键挂载点而导致问题。`listener.log` 将记录有关数据库传入连接的信息。`alert.log` 是另一个记录数据库错误、更改和活动的文件。对于活跃系统，此文件可能迅速增长到数千兆字节。对于许多环境，`listener.log` 文件中的信息无需保留。如果存在 Oracle Net 连接问题，则可以检查该文件以帮助排除故障。

`listener.log` 和 `alert.log` 文件正在被写入，因此不应直接删除它们。如果删除该文件，监听器进程不会重新创建该文件并开始再次写入；你必须停止并重新启动监听器以重新开始写入 `listener.log` 文件。`alert.log` 会被重新创建，但应按照以下 `listener.log` 示例中的相同方式处理。但是，你可以将 `listener.log` 文件置空或截断。在 Linux/Unix 环境中，通过以下技术完成：

```bash
$ cat /dev/null >listener.log
```

前面的命令将 `listener.log` 文件的内容替换为 `/dev/null` 的内容（Linux/Unix 系统上的一个默认文件，不包含任何内容）。此命令的结果是 `listener.log` 文件被截断，监听器可以继续主动写入它。

下面列出了一个 shell 脚本，该脚本在备份后截断默认的 `listener.log` 文件，备份保留至下次运行时被覆盖。此脚本依赖于设置操作系统变量 `ORACLE_BASE`。如果在环境中未设置该变量，则必须在脚本中硬编码目录路径：

```bash
#!/bin/bash
#
if [ $# -ne 1 ]; then
echo "Usage: $0 SID"
exit 1
fi
# 有关设置操作系统变量的详细信息，请参见第 2 章
# 使用 oraset 脚本设置 oracle 操作系统变量
. /etc/oraset $1
#
MAILX='/bin/mailx'
MAIL_LIST='dkuhn@gmail.com'
BOX=$(uname -a | awk '{print $2}' | cut -f 1 -d'.')
#
if [ -f $ORACLE_BASE/diag/tnslsnr/$BOX/listener/trace/listener.log ]; then
cp $ORACLE_BASE/diag/tnslsnr/$BOX/listener/trace/listener.log $ORACLE_BASE/diag/tnslsnr/$BOX/listener/trace/listener.bkup
cat /dev/null > $ORACLE_BASE/diag/tnslsnr/$BOX/listener/trace/listener.log
fi
if [ $? -ne 0 ]; then
echo "trunc list. problem" | $MAILX -s "trunc list. problem $1" $MAIL_LIST
else
echo "no problem..."
fi
exit 0
```

以下 `cron` 条目每月运行一次前面的脚本（此条目应全部在一行，但为了适应页面已放置在两行）：

```bash
30 6 1 * * /orahome/oracle/bin/trunc_log.bsh DWREP
1>/orahome/oracle/bin/log/trunc_log.log 2>&1
```

## 检查被锁定的生产账户

通常应有一个数据库配置文件，指定数据库账户在指定次数的失败登录尝试后被锁定。例如，将 `DEFAULT` 配置文件的 `FAILED_LOGIN_ATTEMPTS` 设置为 5。然而，有时恶意用户或开发人员会尝试猜测生产账户密码，经过五次尝试后，会锁定生产账户。发生这种情况时，需要尽快收到警报，以便调查是否存在安全事件或用户问题，然后解锁账户。

以下 shell 脚本检查 `DBA_USERS` 中生产数据库账户列表的 `LOCK_DATE` 值：

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
echo "Usage: $0 SID"
exit 1
fi
# 设置 oracle 操作系统变量
. /etc/oraset $1
#
crit_var=$(sqlplus -s <<EOF
/ as sysdba
SET HEAD OFF FEED OFF
SELECT count(*)
FROM dba_users
WHERE lock_date IS NOT NULL
AND username in ('CIAP','REPV','CIAL','STARPROD');
EOF)
#
if [ $crit_var -ne 0 ]; then
echo $crit_var
echo "locked acct. issue with $1" | mailx -s "locked acct. issue" dkuhn@sun.com
else
echo $crit_var
echo "no locked accounts"
fi
exit 0
```

此 shell 脚本由调度工具（如 `cron`）调用。例如，此 `cron` 条目指示作业每 10 分钟运行一次（此条目实际上应为单行代码，但为了适应页面已放置在两行）：

```bash
0,10,20,30,40,50 * * * * /home/oracle/bin/lock.bsh DWREP
1>/home/oracle/bin/log/lock.log 2>&1
```

这样，当其中一个生产数据库账户被锁定时，就会发出电子邮件通知。如果风险水平可接受，作为此脚本的一部分，应有一个步骤在设定的时间后解锁账户，并记录失败的登录尝试。

## 检查进程过多

在某些数据库服务器上，你可能有许多后台 SQL*Plus 作业。这些批处理作业可能执行诸如从远程数据库复制数据和大型每日更新作业等任务。在这些环境中，了解在任何给定时间数据库服务器上是否有异常数量的 shell 脚本或 SQL*Plus 进程运行是有用的。异常数量的作业可能表明某些内容已损坏或挂起。

下一个 shell 脚本中有两个检查：一个用于确定名称以 `bsh` 结尾的 shell 脚本的数量，另一个用于确定包含字符串 `sqlplus` 的进程数量：

```bash
#!/bin/bash
#
if [ $# -ne 0 ]; then
echo "Usage: $0"
exit 1
fi
#
crit_var=$(ps -ef | grep -v grep | grep bsh | wc -l)
if [ $crit_var -lt 20 ]; then
echo $crit_var
echo "processes running normal"
else
echo "too many processes"
echo $crit_var | mailx -s "too many bsh procs: $1" dkuhn@gmail.com
fi
#
crit_var=$(ps -ef | grep -v grep | grep sqlplus | wc -l)
if [ $crit_var -lt 30 ]; then
echo $crit_var
echo "processes running normal"
else
echo "too many processes"
echo $crit_var | mailx -s "too many sqlplus procs: $1" dkuhn@gmail.com
fi
#
exit 0
```

前面的 shell 脚本名为 `proc_count.bsh`，由 `cron` 作业每小时运行一次（此条目实际上应为单行代码，但为了适应页面已放置在两行）：

```bash
33 * * * * /home/oracle/bin/proc_count.bsh
1>/home/oracle/bin/log/proc_count.log 2>&1
```

## 验证 RMAN 备份的完整性

作为备份和恢复策略的一部分，你应定期验证备份文件的完整性。这也包含在使用 RMAN 备份数据库的过程中，但可以针对它们运行单独的作业以验证是否可以恢复。RMAN 提供了一个 `RESTORE...VALIDATE` 命令，用于检查备份文件内的物理损坏。以下脚本启动 RMAN 并假脱机一个日志文件。随后在日志文件中搜索关键字 error。如果日志文件中有任何错误，则会发送电子邮件：


```
#!/bin/bash
#
if [ $# -ne 1 ]; then
echo "用法: $0 SID"
exit 1
fi
# 加载 Oracle 操作系统变量
. /etc/oraset $1
#
date
BOX=`uname -a | awk '{print$2}'`
rman nocatalog <<EOF
connect target /
spool log to $HOME/bin/log/rman_val.log
set echo on;
restore database validate;
EOF
grep -i error $HOME/bin/log/rman_val.log
if [ $? -eq 0 ]; then
echo "RMAN 验证问题 $BOX, $1" | \
mailx -s "RMAN 验证问题 $BOX, $1" dkuhn@sun.com
else
echo "没有问题..."
fi
#
date
exit 0
```

`RESTORE...VALIDATE` 命令实际上不会恢复任何文件；它只验证恢复数据库所需的文件是否可用，并检查物理损坏。

如果还需要检查逻辑损坏，请指定 `CHECK LOGICAL` 子句。例如，要检查逻辑损坏，之前的 Shell 脚本中应包含这一行：

```
restore database validate check logical;
```

对于大型数据库，验证过程可能需要很长时间（因为脚本会检查备份文件中的每个数据块）。如果只想验证备份文件是否存在，请指定 `VALIDATE HEADER` 子句，如下所示：

```
restore database validate header;
```

此命令仅检查每个恢复和恢复所需文件的头部是否包含有效信息。

**Autonomous Database（自主数据库）**
前述的脚本和任务在调度时，仅仅是触及了 Oracle 数据库自动化作业的皮毛。开始获取这些脚本的结果并应用修复、执行下一步操作的流程，才更接近真正的自动化。Oracle 18c 数据库拥有许多允许环境配置更深度自动化的流程和钩子。Oracle 18c 本身并非自主数据库，但在 Oracle Cloud 环境中，它成为了 Oracle Autonomous Database。这就是所谓的自修复、自修补和自驱动数据库。

在 Oracle Cloud 中，数据库由自动化流程管理、备份，并在需要时执行故障转移和基本问题排查以处理问题。问题可能是需要增加处理能力或额外存储。如果数据库使用率不高，它可以缩减规模以节省成本。数据库了解活动发生的时间，并通过学习来监控性能问题并采取措施进行修复。

在 Oracle Cloud 中，默认会实施安全配置以及安全选项。Autonomous Database 启用了威胁检测和加密，以保护其存储的数据。修补也是自动化的，因此当出现漏洞时，修补流程可以应用修复。这些步骤无需人工干预即可发生，并且在高可用性环境中，数据库不会经历停机。

在这种云环境中，DBA 该做什么？在开发、数据集成、质量、安全和商业智能等其他领域，都有大量机会可以为企业增加价值。这不正是我们一直在讨论的——自动化基本任务并寻找方法来修复问题，以实现更一致、更稳定的数据库环境吗？

数据库有很多功能可以用于管理业务的其他领域，甚至还有应该构建到应用程序中的数据库新特性。DBA 可以成为推动这一切的人。本章不会深入探讨 Oracle Cloud 的细节，因为我们讨论的仍然是 Oracle 18c 数据库。区别在于运行数据库的位置或硬件。Oracle Cloud 和 Oracle Cloud Machine（在你的数据中心内）为数据库的配置和修补备份流程提供了监控、支持和自动化。

Oracle Cloud 数据库甚至可以使用 DBA 已经熟悉的工具（如 SQL Developer 和 Cloud Control，即前 Enterprise Manager）进行与本地数据库相同的管理。用户通过云服务进行管理，DBA 可以帮助管理这些资源，并为迁移到云环境提供意见。

随着 Autonomous Database 持续收集有关性能和活动的信息以进行安全预防，它会提供查询计划和索引的变更，以改进和检测安全异常。Oracle 18c 数据库正被用于改变数据库环境的实施方式。数据库只是可以按需为业务需求提供的服务。使该服务可用的流程和步骤需要减少人工操作，增加自动化流程以及修复能力。



