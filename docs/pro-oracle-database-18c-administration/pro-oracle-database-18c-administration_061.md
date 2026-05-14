# RMAN 数据库恢复概述

## 恢复过程

当您发出 `RESTORE` 命令时，RMAN 会自动决定如何从以下任何可用备份中提取数据文件：

*   全量数据库备份
*   增量 0 级备份
*   由 `BACKUP AS COPY` 命令生成的映像副本备份

文件从备份中恢复后，您需要通过 `RECOVER` 命令将重做日志应用于它们。当您发出 `RECOVER` 命令时，Oracle 会检查受影响数据文件中的 SCN，并判断是否有任何文件需要恢复。如果数据文件中的 SCN 小于控制文件中对应的 SCN，则需要进行介质恢复。

Oracle 检索数据文件的 SCN，然后在重做流中查找对应的 SCN，以确定从何处开始恢复过程。如果起始恢复 SCN 位于在线重做日志文件中，则不需要归档日志文件用于恢复。

在恢复期间，RMAN 会自动决定如何应用重做。首先，RMAN 会应用任何大于 0 级的可用增量备份，例如增量 1 级备份。接下来，应用磁盘上的任何归档日志文件。如果磁盘上不存在归档日志文件，RMAN 会尝试从备份集中检索它们。

## 执行完全恢复的条件

要能够执行完全恢复，所有以下条件都需要满足：

*   您的数据库处于归档日志模式。
*   您有一个良好的数据库基线备份。
*   您拥有自备份以来生成的任何必需重做（归档日志文件、在线重做日志文件或 RMAN 可以用于恢复而无需应用重做的增量备份）。

## 恢复与恢复场景

恢复和恢复的场景多种多样。如何恢复和恢复直接取决于您的备份策略以及哪些文件已损坏。以下是面对介质故障时的一般步骤：

1.  确定需要恢复哪些文件。
2.  根据损坏情况，将数据库模式设置为 `nomount`、`mount` 或 `open`。
3.  使用 `RESTORE` 命令从 RMAN 备份中检索文件。
4.  对需要恢复的数据文件使用 `RECOVER` 命令。
5.  打开您的数据库。

您特定的恢复和恢复场景可能不需要执行所有上述步骤。例如，您可能只想恢复您的 `spfile`，这不需要恢复步骤。

恢复与恢复过程的第一步是确定哪些文件经历了介质故障。您通常可以从以下来源确定需要恢复的文件：

*   屏幕上显示的错误消息，来自 RMAN 或 SQL*Plus
*   `Alert.log` 文件和对应的跟踪文件
*   数据字典视图

此外，除了前面列出的方法，您还应考虑使用数据恢复顾问来获取有关故障范围和相应纠正措施的信息。

## 数据恢复顾问

数据恢复顾问工具在 Oracle Database 11g 中引入。在发生介质故障时，此工具将显示故障的详细信息、建议纠正措施，如果您指定，它还可以执行推荐的措施。这就像在恢复和恢复情况下拥有另一双眼睛来提供反馈。数据恢复顾问有四种模式：

*   列出故障
*   建议纠正措施
*   运行命令以修复故障
*   更改故障状态

数据恢复顾问从 RMAN 中调用。您可以将数据恢复顾问视为一组在处理介质故障时可以协助您的 RMAN 命令。

### 列出故障

使用数据恢复顾问时，`LIST FAILURE` 命令用于显示数据文件、控制文件或在线重做日志的任何问题：

```
RMAN> list failure;
```

如果没有检测到故障，您将看到一条指示没有故障的消息。以下是一些示例输出，表明数据文件可能存在问题：

```
List of Database Failures
=========================
Failure ID Priority Status    Time Detected Summary
---------- -------- --------- ------------- -------
6222       CRITICAL OPEN      12-JAN-18     System datafile 1:
'/u01/dbfile/o18c/system01.dbf' is missing
```

要显示有关故障的更多信息，请使用 `DETAIL` 子句：

```
RMAN> list failure 6222 detail;
```

以下是此示例的附加输出：

```
Impact: Database cannot be opened
```

对于这种类型的故障，前面的输出表明数据库无法打开。

> 提示
> 如果您怀疑存在介质故障，但数据恢复顾问没有报告任何问题，请运行 `VALIDATE DATABASE` 命令来验证数据库是否完整。

### 建议纠正措施

`ADVISE FAILURE` 命令提供关于如何从数据恢复顾问检测到的潜在问题中恢复的建议。如果您的数据库有多个故障，您可以直接指定故障 ID 以获取对特定故障的建议，如下所示：

```
RMAN> advise failure 6222;
```

以下是此特定问题的输出片段：

```
Optional Manual Actions
=======================
1. If file /u01/dbfile/o18c/system01.dbf was unintentionally renamed or moved,
restore it
Automated Repair Options
========================
Option Repair Description
------ ------------------
1      Restore and recover datafile 1
Strategy: The repair includes complete media recovery with no data loss
Repair script: /ora01/app/oracle/diag/rdbms/o18c/o18c/hm/reco_4116328280.hm
```

在这种情况下，数据恢复顾问创建了一个可用于潜在修复问题的脚本。可以使用操作系统实用程序查看修复脚本的内容；例如，

```
$ cat /ora01/app/oracle/diag/rdbms/o18c/o18c/hm/reco_4116328280.hm
```

以下是此示例脚本的内容：

```
# restore and recover datafile
restore ( datafile 1 );
recover datafile 1;
sql 'alter database datafile 1 online';
```

查看脚本后，您可以决定手动运行建议的命令，或者让数据恢复顾问通过 `REPAIR` 命令运行该脚本（详见下一节）。

### 修复故障

如果您已识别故障并查看了建议，可以着手进行修复工作。如果您想在不实际运行命令的情况下检查 `REPAIR FAILURE` 命令将执行的操作，请使用 `PREVIEW` 子句：

```
RMAN> repair failure preview;
```

在运行 `REPAIR FAILURE` 命令之前，请确保您首先在同一连接会话中运行了 `LIST FAILURE` 和 `ADVISE FAILURE` 命令。换句话说，您所在的 RMAN 会话必须在同一会话中运行 `LIST` 和 `ADVISE` 命令，然后才能运行 `REPAIR` 命令。

如果您对修复建议满意，则运行 `REPAIR FAILURE` 命令：

```
RMAN> repair failure;
```

此时您将收到确认提示：

```
Do you really want to execute the above repair (enter YES or NO)?
```

输入 `YES` 以继续：

```
YES
```

如果一切顺利，您应该在输出中看到类似这样的最终消息：

```
repair failure complete
```

> 注意
> 您可以从 RMAN 命令提示符或 Enterprise Manager 运行数据恢复顾问命令。

通过这种方式，您可以使用 RMAN 命令 `LIST FAILURE`、`ADVISE FAILURE` 和 `REPAIR FAILURE` 来解决介质故障。

### 更改故障状态

关于数据恢复顾问的最后一点说明：如果您知道您发生了一个故障并且它不是关键性的（例如，从不再使用的表空间中丢失了数据文件），那么使用 `CHANGE FAILURE` 命令来更改故障的优先级。在此示例中，有一个属于非关键表空间的缺失数据文件。首先，通过 `LIST FAILURE` 命令获取故障优先级：

```
RMAN> list failure;
```

以下是一些示例输出：

