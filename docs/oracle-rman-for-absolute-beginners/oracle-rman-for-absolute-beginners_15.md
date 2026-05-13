# source oracle OS variables
export ORACLE_SID=O12C
export ORACLE_HOME=/orahome/app/oracle/product/12.1.0.1/db_1
export PATH=$PATH:$ORACLE_HOME/bin
crit_var=$(sqlplus -s <<EOF
/ as sysdba
SET HEAD OFF FEEDBACK OFF
SELECT COUNT(*) FROM
(SELECT (sysdate - MAX(end_time)) delta
 FROM v\$rman_backup_job_details) a
WHERE a.delta > $2;
EOF)
#
if [ $crit_var -ne 0 ]; then
  echo "rman backups not running on $1" | mailx -s "rman problem" dkuhn@gmail.com
else
  echo "rman backups ran ok"
fi
#--------------------------------------------
crit_var2=$(sqlplus -s <<EOF
/ as sysdba
SET HEAD OFF FEEDBACK OFF
SELECT COUNT(*)
FROM
(
SELECT name
FROM v\$datafile
MINUS
SELECT DISTINCT
 f.name
FROM v\$backup_datafile d
    ,v\$datafile        f
WHERE d.file#     = f.file#
AND   d.completion_time > sysdate - $2);
EOF)
#
if [ $crit_var2 -ne 0 ]; then
  echo "datafile not backed up on $1" | mailx -s "backup problem" dkuhn@gmail.com
else
  echo "datafiles are backed up..."
fi
#
exit 0
```

例如，要检查备份在过去 2 天内是否成功运行，请运行该脚本（名为 `rman_chk.bsh`）：

```
$ rman_chk.bsh O12c 2
```

前面的脚本是基础的，但很有效。你可以根据你的 RMAN 环境需要对其进行增强。

### 总结

RMAN 为备份提供了许多灵活且功能丰富的选项。默认情况下，RMAN 仅备份数据库中已修改的块。增量功能允许你仅备份自上次备份以来修改过的块。这些增量功能在大型数据库环境中特别有用，可以减少备份的大小，因为这类环境中数据库中只有很小比例的数据在每次备份之间发生变化。

你可以指示 RMAN 通过映像副本备份每个数据文件中的每个块。映像副本是数据文件逐块相同的副本。映像副本的优点是能够直接从备份中恢复备份文件（无需使用 RMAN）。你可以使用增量更新备份功能来实现映像副本备份和增量备份的有效混合。

RMAN 包含用于报告备份许多方面的内置命令。`LIST` 命令报告备份活动。`REPORT` 命令对于根据保留策略确定哪些文件需要备份非常有用。

在你成功配置 RMAN 并创建备份之后，你就可以在发生介质故障时恢复和恢复你的数据库。恢复和恢复的主题将在下一章详细说明。

# 第六章

![image](img/frontdot.jpg)

## RMAN 恢复与恢复

几年前的一个周六早上，我正在进行一次长距离骑行。大约骑到一半时，我的手机响了。是数据中心的一个运维支持技术员。他告诉我一个关键业务数据库服务器行为异常，我应该尽快登录并确保一切正常。我告诉他，我大约 15 分钟后才能登录。于是，我尽可能快地赶回家检查那台生产机器。当我回到家登录到数据库服务器时，我尝试启动 SQL\*Plus，却立即收到一个错误，提示 SQL\*Plus 二进制文件已损坏。太好了。我甚至无法登录 SQL\*Plus。这可不妙。

![Image](img/sq.jpg)  **切记**  确保所有的自行车骑行都在手机信号覆盖范围之外进行。 – 编者注。

我让系统管理员从操作系统备份中恢复了 Oracle 二进制文件。我启动了 SQL\*Plus。数据库已经崩溃，所以我尝试启动它。输出表明所有数据文件都存在介质故障。经过一些分析，发现之前存在一些文件系统问题，磁盘上的所有这些文件都已损坏：

*   数据文件
*   控制文件
*   归档重做日志
*   在线重做日志文件
*   RMAN 备份片

这几乎是一场彻底的灾难。我的总监询问我们的选择。我回答说：“我们所要做的就是从上次的磁带备份中恢复数据库，我们将丢失那些尚未备份到磁带的归档重做日志中所包含的任何数据。”

存储管理员被召集进来，并被指示恢复最后写入磁带的一组 RMAN 备份。大约 15 分钟后，我们可以听到磁带管理员们在低声交谈。其中一人说：“我们完蛋了。这台机器上任何数据库的 RMAN 磁带备份都没有。”

那是一个黑暗的时刻。最坏的情况是从 DDL 脚本重建数据库并丢失 3 年的生产数据。这不是一个非常容易接受的选项。

在检查了生产机器之后，我发现之前的支持 DBA（讽刺的是，他几天前刚因预算削减被解雇）实现了一个作业，将 RMAN 备份复制到生产环境中的另一台服务器上。那台其他服务器上的 RMAN 备份是完好无损的。我能够从这些备份中恢复和恢复生产数据库。我们丢失了大约一天的数据（在损坏的归档日志和停机期间，期间不允许传入事务），但在最初的电话呼叫大约 20 小时后，我们成功地恢复并恢复了数据库。那是漫长的一天。

大多数需要你进行恢复和恢复的情况都不会像刚才描述的那么糟糕。然而，前面的场景确实凸显了对以下内容的需求：

*   一个备份策略
*   一个具备 B&R 技能的 DBA
*   一个恢复与恢复策略，包括定期测试恢复和恢复的要求


# 通过 RMAN 执行恢复与修复

本章将引导你使用 RMAN 完成恢复与修复操作。本章涵盖了处理介质故障时需要执行的许多常见任务。

### 确定是否需要介质恢复

术语**介质恢复**是指恢复因底层存储介质（通常是某种磁盘）故障或文件被意外删除而丢失或损坏的文件。通常，你会通过类似以下的错误得知需要介质恢复：

```
ORA-01157: cannot identify/lock data file 1 - see DBWR trace file
ORA-01110: data file 1: '/u01/dbfile/O12C/system01.dbf'
```

在执行数据库管理任务（如停止和启动数据库）时，此错误可能会显示在屏幕上。或者，你可能会在跟踪文件或 `alert.log` 文件中看到此类错误。如果你没有立即注意到问题，在严重的介质故障情况下，数据库将停止处理事务，用户就会开始给你打电话。

要理解 Oracle 如何确定需要介质恢复，你必须首先理解 Oracle 如何确定一切正常。当 Oracle 正常关闭（`IMMEDIATE`、`TRANSACTIONAL`、`NORMAL`）时，关闭过程的一部分是将所有已修改的块（在内存中）刷新到磁盘，在每个数据文件的头部标记当前的系统变更号（SCN），并用当前 SCN 信息更新控制文件。

启动时，Oracle 会检查控制文件中的 SCN 是否与数据文件头部的 SCN 匹配。如果匹配，Oracle 尝试打开数据文件和在线重做日志文件。如果所有文件都可用且可以打开，Oracle 将正常启动。以下查询比较了控制文件中（针对每个数据文件）的 SCN 与数据文件头部中的 SCN：

```
SET LINES 132
COL name             FORM a40
COL status           FORM A8
COL file#            FORM 9999
COL control_file_SCN FORM 999999999999999
COL datafile_SCN     FORM 999999999999999
--
SELECT
 a.name
,a.status
,a.file#
,a.checkpoint_change# control_file_SCN
,b.checkpoint_change# datafile_SCN
,CASE
   WHEN ((a.checkpoint_change# - b.checkpoint_change#) = 0) THEN 'Startup Normal'
   WHEN ((b.checkpoint_change#) = 0)                        THEN 'File Missing?'
   WHEN ((a.checkpoint_change# - b.checkpoint_change#) > 0) THEN 'Media Rec. Req.'
   WHEN ((a.checkpoint_change# - b.checkpoint_change#) < 0) THEN 'Old Control File'
   ELSE 'what the ?'
 END datafile_status
FROM v$datafile        a -- control file SCN for datafile
    ,v$datafile_header b -- datafile header SCN
WHERE a.file# = b.file#
ORDER BY a.file#;
```

如果控制文件 SCN 值大于数据文件 SCN 值，则很可能需要介质恢复。如果你从备份中还原了数据文件，并且还原的数据文件中的 SCN 小于当前控制文件中数据文件的 SCN，就会出现这种情况。

![Image](img/sq.jpg) **提示** `V$DATAFILE_HEADER` 视图使用磁盘上的物理数据文件作为其源。`V$DATAFILE` 视图使用控制文件作为其源。

你也可以直接查询 `V$DATAFILE_HEADER` 以获取更多信息。`ERROR` 和 `RECOVER` 列报告任何潜在问题。例如，`RECOVER` 列中的 `YES` 或 `null` 值表示存在问题：

```
SQL> select file#, status, error, recover from v$datafile_header;
```

以下是一些示例输出：

```
     FILE# STATUS  ERROR                REC
---------- ------- -------------------- ---
         1 ONLINE  FILE NOT FOUND
         2 ONLINE                       NO
         3 ONLINE                       NO
```

### 确定要恢复的内容

介质恢复需要你执行手动任务以使数据库恢复完整。这些任务通常涉及 `RESTORE` 和 `RECOVER` 命令的组合使用。如果由于某种原因（文件被意外删除、磁盘故障等）导致数据文件发生介质故障，你将需要发出 RMAN `RESTORE` 命令。

### 流程工作原理

当你发出 `RESTORE` 命令时，RMAN 会自动决定如何从以下任何可用备份中提取数据文件：

*   完全数据库备份
*   增量级别 0 备份
*   由 `BACKUP AS COPY` 命令生成的映像副本备份

文件从备份中还原后，你需要通过 `RECOVER` 命令对其应用重做。当你发出 `RECOVER` 命令时，Oracle 会检查受影响数据文件中的 SCN，并确定其中任何文件是否需要恢复。如果数据文件中的 SCN 小于控制文件中对应的 SCN，则将需要介质恢复。

Oracle 检索数据文件 SCN，然后在重做流中查找对应的 SCN，以确定从何处开始恢复过程。如果起始恢复 SCN 在线重做日志文件中，则不需要归档重做日志文件进行恢复。

在恢复期间，RMAN 会自动确定如何应用重做。首先，RMAN 应用任何大于级别 0 的可用增量备份，例如增量级别 1。接下来，应用磁盘上的任何归档重做日志文件。如果归档重做日志文件在磁盘上不存在，RMAN 会尝试从备份集中检索它们。

要能够执行完全恢复，需要满足以下所有条件：

*   你的数据库处于归档日志模式。
*   你拥有数据库的良好基线备份。
*   你拥有自备份以来生成的任何所需重做（归档重做日志文件、在线重做日志文件，或 RMAN 可以用于恢复而不是应用重做的增量备份）。

存在多种多样的恢复场景。你如何恢复直接取决于你的备份策略以及哪些文件已损坏。以下是面对介质故障时应遵循的一般步骤：

1.  确定需要还原的文件。
2.  根据损坏情况，将数据库模式设置为 `nomount`、`mount` 或 `open`。
3.  使用 `RESTORE` 命令从 RMAN 备份中检索文件。
4.  对需要恢复的数据文件使用 `RECOVER` 命令。
5.  打开你的数据库。

你特定的恢复场景可能不需要执行所有上述步骤。例如，你可能只想还原 `spfile`，这就不需要恢复步骤。

恢复过程的第一步是确定哪些文件经历了介质故障。你通常可以从以下来源确定需要还原的文件：

*   屏幕上显示的错误消息，来自 RMAN 或 SQL*Plus
*   `Alert.log` 文件及相应的跟踪文件
*   数据字典视图

如果你使用的是 Oracle 11g 或更高版本，除了前面列出的方法外，你应考虑使用数据恢复顾问来获取有关故障范围及相应纠正措施的信息。

### 使用数据恢复顾问

数据恢复顾问工具在 Oracle 11g 中引入。在发生介质故障时，此工具将显示故障详情、建议纠正措施，并在你指定时执行建议的措施。这就像在恢复情况下拥有另一双眼睛提供反馈一样。数据恢复顾问有四种模式：

*   列出故障
*   建议纠正措施
*   运行命令修复故障
*   更改故障状态

数据恢复顾问从 RMAN 中调用。你可以将数据恢复顾问视为一组在处理介质故障时可以协助你的 RMAN 命令。

### 列出故障

使用数据恢复顾问时，`LIST FAILURE` 命令用于显示数据文件、控制文件或在线重做日志的任何问题：

```
RMAN> list failure;
```



![Image](img/sq.jpg) **提示** 如果您怀疑存在介质故障，但数据恢复顾问（Data Recovery Advisor）并未报告任何问题，可以运行 `VALIDATE DATABASE` 命令来验证数据库的完整性。

如果没有检测到故障，您将看到一条消息，表明没有故障。以下是一些示例输出，表明数据文件可能存在问题：

```
List of Database Failures
=========================

Failure ID Priority Status    Time Detected Summary
---------- -------- --------- ------------- -------
6222       CRITICAL OPEN      12-JAN-14     System datafile 1:
 '/u01/dbfile/O12C/system01.dbf' is missing
```

要显示有关故障的更多信息，请使用 `DETAIL` 子句：

```
RMAN> list failure 6222 detail;
```

以下是此示例的附加输出：

```
Impact: Database cannot be opened
```

对于此类故障，前面的输出表明数据库无法打开。

**提出纠正措施**

`ADVISE FAILURE` 命令提供有关如何恢复数据恢复顾问检测到的潜在问题的建议。如果数据库存在多个故障，您可以直接指定故障 ID 以获取针对特定故障的建议，如下所示：

```
RMAN> advise failure 6222;
```

以下是针对此特定问题的部分输出：

```
Optional Manual Actions
=======================
1\. If file /u01/dbfile/O12C/system01.dbf was unintentionally renamed or moved,
restore it

Automated Repair Options
========================
Option Repair Description
------ ------------------
1      Restore and recover datafile 1
  Strategy: The repair includes complete media recovery with no data loss
  Repair script: /ora01/app/oracle/diag/rdbms/O12C/O12C/hm/reco_4116328280.hm
```

在这种情况下，数据恢复顾问创建了一个可用于潜在修复问题的脚本。可以使用操作系统实用程序查看修复脚本的内容；例如，

```
$ cat /ora01/app/oracle/diag/rdbms/O12C/O12C/hm/reco_4116328280.hm
```

以下是此示例的脚本内容：

```
