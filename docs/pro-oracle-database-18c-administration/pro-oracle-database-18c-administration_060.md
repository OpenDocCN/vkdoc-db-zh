# 19. RMAN 恢复与还原

关于恢复操作，甚至每月为执行还原而进行的演练，我可以给你讲不少故事。备份是否有效，关键在于你能否成功还原；希望你永远用不上它，但还原过程必须被记录、测试和演练。通过演练，可以暴露错误并记录可能出现的异常情况，这样在压力下需要紧急完成还原的关键时刻，多年的练习将派上用场。这也有助于验证是否有可用的有效备份，正如合著者在一位 DBA 的故事中所描述的：

几年前的一个周六上午，我正在进行一次长距离的自行车骑行。骑行到一半时，手机响了。是数据中心的一位运维支持技术员。他告诉我一个关键任务数据库服务器行为异常，我应该尽快登录并确保一切正常。我告诉他大约 15 分钟后我才能登录。于是，我以最快速度赶回家检查生产服务器。到家后，我登录到数据库服务器，尝试启动`SQL*Plus`，立刻收到一个错误，指示`SQL*Plus`二进制文件已损坏。太棒了。我甚至无法登录`SQL*Plus`。情况不妙。

我让系统管理员从操作系统备份中恢复了 Oracle 二进制文件。然后启动了`SQL*Plus`。数据库已经崩溃，所以我尝试启动它。输出显示所有数据文件都存在介质故障。经过一些分析，发现存在一些文件系统问题，导致磁盘上的所有这些文件都已损坏：
*   数据文件
*   控制文件
*   归档重做日志
*   在线重做日志文件
*   RMAN 备份片

这几乎是一场全面的灾难。我的总监询问我们的选择。我回答说：“我们只需从最后一次磁带备份中还原数据库，我们会丢失那些尚未备份到磁带的归档重做日志中的数据。”

存储管理员被召集来，指示他们还原最近一次写入磁带的 RMAN 备份集。大约 15 分钟后，我们能听到磁带管理员们在压低声音交谈。其中一人说：“我们完蛋了。这台服务器上任何数据库的 RMAN 备份，我们都没有磁带备份。”

那是一个黑暗的时刻。最坏的情况是从 DDL 脚本重建数据库，并丢失 3 年的生产数据。这不是一个令人愉快的选择。

在检查生产服务器后，我发现之前的一位生产支持 DBA（讽刺的是，他因预算削减在几天前刚被解雇）已经实施了一个作业，将 RMAN 备份复制到生产环境中的另一台服务器。那台服务器上的 RMAN 备份完好无损。我得以从这些备份中还原并恢复了生产数据库。我们丢失了大约一天的数据（介于损坏的归档日志和停机时间之间，期间不允许传入任何交易），但在接到第一个电话后大约 20 个小时，我们成功完成了数据库的还原和恢复。那真是漫长的一天。

大多数需要还原和恢复的情况都不会像刚才描述的那样糟糕。即使在那些有安全保障且自然灾害很少的地方，似乎也总会发生一些事情，促使你想测试恢复过程并验证备份的有效性。然而，前面的场景确实突显了以下需求的必要性：
*   备份策略
*   具备备份与恢复技能的 DBA
*   还原与恢复策略，包括定期测试还原和恢复的要求

本章将引导你使用 RMAN 完成还原和恢复过程。本章涵盖了处理介质故障时你将必须执行的许多常见任务。

## 确定是否需要介质恢复

术语*介质恢复*是指还原因底层存储介质（通常是某种磁盘）故障或文件被意外删除而丢失或损坏的文件。通常，你会通过类似以下错误知道需要进行介质恢复：

```
ORA-01157: cannot identify/lock data file 1 - see DBWR trace file
ORA-01110: data file 1: '/u01/dbfile/o12c/system01.dbf'
```

在执行 DBA 任务（如停止和启动数据库）时，此错误可能会显示在屏幕上。或者，你可能在跟踪文件或`alert.log`文件中看到此类错误。也有可能由于文件未被写入或操作系统的原因，错误显示可能会延迟。如果你没有立即注意到问题，一旦发生严重的介质故障，数据库将停止处理交易，用户会开始打电话找你。

要理解 Oracle 如何确定需要介质恢复，你必须首先了解 Oracle 如何确定一切正常。当 Oracle 正常关闭（`IMMEDIATE`、`TRANSACTIONAL`、`NORMAL`）时，关闭过程的一部分是将内存中所有已修改的块刷新到磁盘，在每个数据文件的头部标记当前 SCN，并用当前 SCN 信息更新控制文件。

启动时，Oracle 会检查控制文件中的 SCN 是否与数据文件头部的 SCN 匹配。如果匹配，Oracle 会尝试打开数据文件和在线重做日志文件。如果所有文件都可用且可以打开，Oracle 将正常启动。以下查询比较控制文件中（针对每个数据文件）的 SCN 与数据文件头部的 SCN：

```
SQL> SET LINES 132
SQL> COL name             FORM a40
SQL> COL status           FORM A8
SQL> COL file#            FORM 9999
SQL> COL control_file_SCN FORM 999999999999999
SQL> COL datafile_SCN     FORM 999999999999999
--
SQL> SELECT
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

如果控制文件 SCN 值大于数据文件 SCN 值，则很可能需要进行介质恢复。如果你从备份中还原了数据文件，并且还原的数据文件的 SCN 小于当前控制文件中的数据文件 SCN，就会出现这种情况。

提示
`V$DATAFILE_HEADER`视图以磁盘上的物理数据文件作为其数据源。`V$DATAFILE`视图以控制文件作为其数据源。

你也可以直接查询`V$DATAFILE_HEADER`以获取更多信息。`ERROR`和`RECOVER`列报告任何潜在问题。例如，`RECOVER`列中的`YES`或`null`值表示存在问题：

```
SQL> select file#, status, error, recover from v$datafile_header;
```

这是一些示例输出：

```
FILE# STATUS  ERROR                REC
---------- ------- -------------------- ---
1 ONLINE  FILE NOT FOUND
2 ONLINE                       NO
3 ONLINE                       NO
```

## 确定要还原什么

介质恢复要求你执行手动任务，以使数据库恢复完整。这些任务通常涉及`RESTORE`和`RECOVER`命令的组合。如果由于某种原因（文件被意外删除、磁盘故障等），你的数据文件发生了介质故障，你将不得不发出 RMAN `RESTORE`命令。

## 过程如何工作


