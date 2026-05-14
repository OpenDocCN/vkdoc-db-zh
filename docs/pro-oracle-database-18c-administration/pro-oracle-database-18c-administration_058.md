# 记录与报告 RMAN 操作

## 记录 RMAN 输出

在对 RMAN 输出进行故障排查或检查备份作业状态时，记录 RMAN 的执行内容及每条命令的状态至关重要。有多种方法可用于记录 RMAN 输出。其中一些是 Linux/Unix 操作系统的内置功能，另一些是 RMAN 特有的功能：

*   Linux/Unix 重定向输出到文件
*   Linux/Unix 日志记录命令
*   RMAN `SPOOL LOG` 命令
*   `V$RMAN_OUTPUT` 视图

这些日志记录功能将在后续章节中讨论。

### 将输出重定向到文件

Shell 脚本通常通过 `cron` 等调度工具自动运行。以这种方式运行 RMAN 命令时，可以通过指示 Shell 命令将标准输出消息和标准错误消息重定向到日志文件来捕获输出。这通过重定向 (`>`) 字符完成。此示例运行一个 Shell 脚本 (`rmanback.bsh`)，并将标准输出和标准错误输出都重定向到名为 `rmanback.log` 的日志文件：

```
$ rmanback.bsh 1>/home/oracle/bin/log/rmanback.log 2>&1
```

此处，`1>` 指示将标准输出重定向到指定文件。`2>&1` 指示 Shell 脚本将标准错误输出发送到与标准输出相同的位置。

> **提示**
> 有关 DBA 如何使用 Shell 脚本和 Linux 功能的更多详细信息，请参阅 Darl Kuhn 所著的 *Linux Recipes for Oracle DBAs* (Apress, 2008)。

### 使用 Linux/Unix 日志记录命令捕获输出

你可以指示 Linux/Unix 创建一个日志文件，以捕获同时显示在屏幕上的任何输出。这可以通过两种方式之一完成：

*   `tee`
*   `script`

#### 使用 tee 捕获输出

启动 RMAN 时，你可以使用 `tee` 命令将屏幕上看到的输出同时发送到操作系统文本文件：

```
$ rman | tee /tmp/rman.log
```

现在，你可以连接到目标数据库并运行命令。所有在屏幕上看到的输出都将被记录到 `/tmp/rman.log` 文件中：

```
RMAN> connect target /
RMAN> backup database;
RMAN> exit;
```

当你退出 RMAN 时，`tee` 会话停止写入日志文件。

#### 使用 script 捕获输出

`script` 命令很有用，因为它指示操作系统将终端上出现的任何输出记录到日志文件中。要捕获所有输出，请在连接到 RMAN 之前运行 `script` 命令：

```
$ script /tmp/rman.log
Script started, file is /tmp/rman.log
$ rman target /
RMAN> backup database;
RMAN> exit;
```

要结束 `script` 会话，请按 Ctrl+D 或键入 `exit`。`/tmp/rman.log` 文件将包含显示在屏幕上的所有输出。当你需要捕获特定时间范围内的所有输出时，`script` 命令非常有用。例如，你可能正在运行 RMAN 命令、退出 RMAN、运行 SQL*Plus 命令等等。`script` 会话从启动 `script` 开始，直到你按下 Ctrl+D 结束。

### 将输出记录到文件

捕获 RMAN 输出的一个简单方法是使用 `SPOOL LOG` 命令将输出发送到文件。此示例从 RMAN 内部启动日志文件：

```
RMAN> spool log to '/tmp/rmanout.log'
RMAN> set echo on;
RMAN>
RMAN> spool log off;
```

默认情况下，`SPOOL LOG` 命令会覆盖现有文件。如果要追加到日志文件，请使用关键字 `APPEND`：

```
RMAN> spool log to '/tmp/rmanout.log' append
```

你也可以在命令行启动 RMAN 时将输出定向到日志文件，这将覆盖现有文件：

```
$ rman target / log /tmp/rmanout.log
```

你也可以追加到日志文件，如下所示：

```
$ rman target / log /tmp/rmanout.log append
```

当你使用前面示例中所示的 `SPOOL LOG` 时，输出会进入文件而不是你的终端。因此，在交互式运行 RMAN 时，我很少使用 `SPOOL LOG`。该命令主要是在从脚本运行 RMAN 时捕获输出的工具。

### 在数据字典中查询输出

如果你没有捕获任何 RMAN 输出，仍然可以通过查询数据字典来查看最近的 RMAN 输出。`V$RMAN_OUTPUT` 视图包含 RMAN 最近报告的消息：

```
SQL> select sid, recid, output
from v$rman_output
order by recid;
```

`V$RMAN_OUTPUT` 视图是一个内存中对象，最多可保存 32,768 行。当你停止并重启数据库时，此视图中的信息会被清除。当你使用 RMAN `SPOOL LOG` 命令将输出假脱机到文件而无法在终端查看正在发生的情况时，此视图非常方便。

## RMAN 报告

有几种不同的方法来报告 RMAN 环境：

*   `LIST` 命令
*   `REPORT` 命令
*   通过数据字典视图查询元数据

初次学习 RMAN 时，`LIST` 和 `REPORT` 命令之间的区别可能令人困惑，因为两者的界限并非泾渭分明。通常，我使用 `LIST` 命令查看有关现有备份的信息，而使用 `REPORT` 命令来确定哪些文件需要备份，或显示有关过期或废弃备份的信息。

SQL 查询可以为专门的报告（无法通过 `LIST` 或 `REPORT` 获得）或自动化报告提供详细信息：例如，通常通过 Shell 脚本和 SQL 实现自动化检查，报告 RMAN 备份在过去一天内是否已运行。

### 使用 LIST

在调查 RMAN 备份问题时，我首先执行的任务之一通常是连接到目标数据库并运行 `LIST BACKUP` 命令。此命令允许你查看备份集、备份片以及备份中包含的文件：

```
RMAN> list backup;
```

该命令显示存储库中记录的所有 RMAN 备份。你可能希望将备份假脱机到输出文件，以便保存输出，然后使用操作系统编辑器在输出中搜索特定字符串。

要获取备份信息的汇总视图，请使用 `LIST BACKUP SUMMARY` 命令：

```
RMAN> list backup summary;
```

你也可以使用 `LIST` 命令仅报告映像副本信息：

```
RMAN> list copy;
```

要列出所有已备份的文件以及相关的备份集，请发出以下命令：

```
RMAN> list backup by file;
```

以下命令显示磁盘上的归档日志：

```
RMAN> list archivelog all;
RMAN> list copy of archivelog all;
```

此外，此命令列出归档日志的备份（以及哪些归档日志包含在哪个备份片中）：

```
RMAN> list backup of archivelog all;
```

运行 `LIST` 命令（同样，`REPORT` 命令将在下一节介绍）的方式有很多种。前面的方法是你大多数时间会使用的。有关完整选项列表，请参阅 Oracle 网站技术网络区域 ( [`http://otn.oracle.com`](http://otn.oracle.com) ) 提供的 *Oracle Database Backup and Recovery Reference Guide*。

### 使用 REPORT

RMAN `REPORT` 命令可用于报告各种详细信息。你可以快速查看与数据库关联的所有数据文件，如下所示：

```
RMAN> report schema;
```

`REPORT` 命令提供有关通过 RMAN 保留策略标记为废弃的备份的详细信息；例如，

```
RMAN> report obsolete;
```

你可以报告根据保留策略需要备份的数据文件，像这样：

```
RMAN> report need backup;
```

报告需要备份的数据文件有几种方式。以下是一些其他示例：

```
RMAN> report need backup redundancy 2;
RMAN> report need backup redundancy 2 datafile 2;
```

`REPORT` 命令也可用于从未备份过的数据文件，或可能包含由 `NOLOGGING` 操作创建的数据的数据文件。例如，假设你直接路径加载数据到一个表中，而该表所在的数据文件尚未备份。以下命令将检测这些情况：


