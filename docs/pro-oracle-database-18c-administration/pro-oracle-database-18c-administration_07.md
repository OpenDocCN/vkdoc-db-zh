# 第 5 步。使用 RESETLOGS 打开数据库

```
SEQUENCE#  STATUS          FIRST_CHANGE#   MEMBER
---------- --------------- --------------- ----------------------
6  INACTIVE        3543960         /u01/oraredo/o12c/redo03a.rdo
6  INACTIVE        3543960         /u02/oraredo/o12c/redo03b.rdo
7  INACTIVE        3543963         /u02/oraredo/o12c/redo01b.rdo
7  INACTIVE        3543963         /u01/oraredo/o12c/redo01a.rdo
8  CURRENT         3583986         /u02/oraredo/o12c/redo02b.rdo
8  CURRENT         3583986         /u01/oraredo/o12c/redo02a.rdo
```

现在，重新启动恢复过程：

```sql
SQL> recover database using backup controlfile;
```

恢复过程提示需要一个不存在的归档重做日志：

```
ORA-00279: change 3584513 generated at 11/02/2018 11:50:50 needed for thread 1
ORA-00289: suggestion : /u01/oraarch/o18c/1_10_798283209.dbf
ORA-00280: change 3584513 for thread 1 is in sequence #10
Specify log: {=suggested | filename | AUTO | CANCEL}
```

不要向恢复过程提供归档重做日志文件，而是输入当前联机重做日志文件的名称（您可能需要尝试每个联机重做日志，直到找到 Oracle 需要的那一个）。这会指示恢复过程应用联机重做日志中的任何重做信息：

```
/u01/oraredo/o18c/redo01a.rdo
```

当应用了正确的联机重做日志时，您应该看到此消息：

```
Log applied.
介质恢复完成。
```

此时数据库已完全恢复。在此类恢复之后，重要的是对数据库执行新的备份。但是，由于恢复过程使用了备份控制文件，因此必须使用 `RESETLOGS` 子句打开数据库：

```sql
SQL> alter database open resetlogs;
```

成功后，您应该看到：

```
Database altered.
```

## 执行归档日志模式数据库的不完全恢复

不完全恢复意味着您没有恢复故障发生前已提交的所有事务。对于此类恢复，您是恢复到过去的一个时间点，而事务将会丢失。这就是为什么不完全恢复也被称为数据库时间点恢复。

不完全恢复并不意味着您只恢复和还原部分数据文件。事实上，在大多数不完全恢复场景中，作为过程的一部分，您必须从备份中还原所有数据文件。如果您不想恢复所有数据文件，则首先需要将不打算参与不完全恢复过程的任何数据文件脱机。当您启动恢复时，Oracle 将仅恢复 `V$DATAFILE_HEADER` 的 `STATUS` 列中值为 `ONLINE` 的数据文件。

您可能出于许多不同原因希望执行不完全恢复：

*   您尝试执行完全恢复，但缺少所需的归档重做日志或未归档的联机重做日志信息。
*   您希望将数据库恢复到用户错误（如删除数据、删除表等）发生之前的某个时间点。
*   您有一个测试环境。（闪回数据库可能是此处的一个选项。）

您可以通过三种方式执行用户管理的不完全恢复：

### 基于取消的恢复

基于取消的恢复允许您应用归档重做，并在基于归档重做日志文件的边界处暂停过程。例如，假设您正在尝试还原和恢复数据库，并且意识到缺少一个归档重做日志。您必须在最后一个完好的归档重做日志处停止恢复过程。您使用 `RECOVER DATABASE` 语句的 `CANCEL` 子句启动基于取消的不完全恢复：

```sql
SQL> recover database until cancel;
```

## 基于 SCN 的恢复

如果您希望恢复到并包括某个 SCN 号，请使用基于 SCN 的不完全恢复。您可能从警报日志或 LogMiner 的输出中知道要恢复到的某个 SCN 点。使用 `UNTIL CHANGE` 子句执行此类不完全恢复：

```sql
SQL> recover database until change 12345;
```

### 基于时间的恢复

如果您知道要停止恢复过程的时间，请使用基于时间的不完全恢复。例如，您可能知道某个表在特定时间被删除，并希望将数据库恢复并恢复到指定时间。基于时间的恢复格式始终如下：`YYYY-MM-DD:HH24:MI:SS`。这是一个示例：

```sql
SQL> recover database until time '2018-10-21:02:00:00';
```

执行不完全恢复时，必须还原所有计划在不完全恢复完成后联机的数据文件。以下是进行不完全恢复的步骤：

1.  关闭数据库。
2.  从备份中还原所有数据文件。
3.  在加载模式下启动数据库。
4.  应用重做（前滚）到所需点，并停止恢复过程（使用基于取消、SCN 或时间的恢复）。
5.  使用 `OPEN RESETLOGS` 子句打开数据库。

以下示例执行基于取消的不完全恢复。如果数据库是打开的，请将其关闭：

```bash
$ sqlplus / as sysdba
SQL> shutdown abort;
```

接下来，从备份中复制所有数据文件（冷备份或热备份均可）。此示例从热备份还原所有数据文件。在此示例中，当前控制文件完好无损，无需还原。以下是用于还原数据库的操作系统复制命令片段：

```bash
cp /u01/hbackup/o18c/system01.dbf /u01/dbfile/o18c/system01.dbf
cp /u01/hbackup/o18c/sysaux01.dbf /u01/dbfile/o18c/sysaux01.dbf
cp /u01/hbackup/o18c/undotbs01.dbf /u01/dbfile/o18c/undotbs01.dbf
cp /u01/hbackup/o18c/users01.dbf /u01/dbfile/o18c/users01.dbf
cp /u01/hbackup/o18c/tools01.dbf /u01/dbfile/o18c/tools01.dbf
```

数据文件复制回来后，即可启动恢复过程。此示例执行基于取消的不完全恢复：

```bash
$ sqlplus / as sysdba
SQL> startup mount;
SQL> recover database until cancel;
```

此时，Oracle 恢复过程建议应用一个归档重做日志：

```
ORA-00279: change 3584872 generated at 11/02/2018 12:02:32 needed for thread 1
ORA-00289: suggestion : /u01/oraarch/o18c/1_1_798292887.dbf
ORA-00280: change 3584872 for thread 1 is in sequence #1
Specify log: {=suggested | filename | AUTO | CANCEL}
```

应用日志到您想要停止的点，然后键入 `CANCEL`：

```
CANCEL
```

这将停止恢复过程。现在，您可以使用 `RESETLOGS` 子句打开数据库：

```sql
SQL> alter database open resetlogs;
```

数据库已打开到过去的一个时间点。由于并非所有重做都已应用，因此该恢复被视为不完全恢复。

**提示**
现在是对数据库进行良好备份的好时机。如果您在打开数据库后不久发生故障，这将为您提供一个干净的点来启动还原和恢复。

## OPEN RESETLOGS 的用途

有时，你需要使用 `OPEN RESETLOGS` 子句来打开数据库。当你重建控制文件、使用备份控制文件执行恢复与还原，或者执行不完全恢复时，可能就需要这样做。当你使用 `OPEN RESETLOGS` 子句打开数据库时，它会清除所有现有的联机重做日志文件，或者如果文件不存在，则会重新创建它们。你可以查询 `V$LOGFILE` 的 `MEMBER` 列，以查看哪些文件涉及 `OPEN RESETLOGS` 操作。

为什么你会想要清除联机重做日志中的内容呢？以不完全恢复为例，在这种情况下，数据库被有意地打开到过去的某个时间点。此时，联机重做日志中的 SCN（系统变更号）信息包含了一些永远无法恢复的事务数据。Oracle 强制你使用 `OPEN RESETLOGS` 打开数据库，就是为了特意清除这些信息。

当你使用 `OPEN RESETLOGS` 打开数据库时，你创建了数据库的一个新化身（incarnation），并将日志序列号重置回 1。Oracle 要求一个新的化身，以避免在需要再次执行还原与恢复时，意外使用任何旧的归档重做日志（这些日志关联于数据库的另一个独立化身）。

## 概要

一些研究表明，过度依赖自动驾驶技术的飞机飞行员，在处理灾难性飞行中问题的能力上，不如那些花费大量时间在没有自动驾驶辅助下飞行的飞行员。过度依赖自动驾驶的飞行员在遇到严重问题时往往忘记关键程序，而那些不那么依赖自动驾驶的飞行员则更擅长诊断和解决令人紧张的飞行中故障。

同样，理解如何使用用户管理技术手动备份、还原和恢复数据库的 DBA，在故障排除和解决严重的备份与恢复问题方面，比那些仅通过屏幕界面操作备份与恢复技术的 DBA 更为熟练。这就是本章被纳入本书的原因。理解每个步骤发生的事情以及为什么需要该步骤，对于全面掌握 Oracle 备份与恢复架构至关重要。在紧张的恢复场景中，拥有额外的知识和工具极具价值。这种意识会转化为关键的故障排除技能，尤其是在使用 Oracle 工具如 RMAN（备份与恢复）、Enterprise Manager 和 Data Guard（灾难恢复、高可用性和复制）时。

本章涵盖的用户管理备份与恢复技术现在已较少被教授或使用。大多数 DBA 正在（并且应该）使用 RMAN 来满足其 Oracle 备份与恢复需求。然而，理解冷备份和热备份的工作原理对你来说至关重要。你可能会受雇于一个仍在使用旧技术的公司，需要还原和恢复数据库、进行故障排除，或协助迁移到 RMAN。在这些场景中，你必须完全理解旧的备份技术。

既然你已经深入理解了 Oracle 备份与恢复机制，现在已准备好研究 RMAN。接下来的几章将探讨如何配置和使用 RMAN 进行生产级备份与恢复。

