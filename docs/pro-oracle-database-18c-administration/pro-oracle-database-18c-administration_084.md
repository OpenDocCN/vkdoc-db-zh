# 检查警报日志

处理数据库问题时，`alert.log` 文件应该是您检查相关错误消息的首批文件之一。您可以使用操作系统工具或 ADRCI 实用程序来查看 `alert.log` 文件和相应的跟踪文件。

## 通过操作系统工具查看警报日志

导航到包含 `alert.log` 的目录后，您可以通过查看文件末尾（最靠下的部分）来查看最新消息（换句话说，最新的消息写入文件的末尾）。要查看最后 50 行，请使用 `tail` 命令：
```
$ tail -50 alert_<ORACLE_SID>.log
```
您可以使用 `-f` 选项持续查看最新条目：
```
$ tail -f alert_<ORACLE_SID>.log
```
您还可以使用操作系统编辑器（如 `vi`）直接打开 `alert.log`：
```
$ vi alert_<ORACLE_SID>.log
```


有时，定义一个函数来打开 `alert.log` 文件会非常方便，无论你当前的工作目录在哪里。接下来的几行代码定义了一个函数，它能在 11g 或 10g 环境中定位并使用 `view` 命令打开 `alert.log`：

```
#-----------------------------------------------------------#
# 查看告警日志
function valert {
if [ "$ORACLE_SID" = "O18C" ]; then
view /orahome/app/oracle/diag/rdbms/o18c/O18C/trace/alert_O18C.log
elif [ "$ORACLE_SID" = "O12C" ]; then
view /orahome/app/oracle/diag/rdbms/o12c/O12C/trace/alert_O12C.log  
elif [ "$ORACLE_SID" = "O11R2" ]; then
view /orahome/app/oracle/diag/rdbms/o11r2/O11R2/trace/alert_O11R2.log
fi
} # valert
#-----------------------------------------------------------#
```

通常，上述代码会被放置在启动文件中，以便在你登录服务器时自动定义该函数。一旦函数定义好，你可以通过输入以下命令来查看 `alert.log`：

```
$ valert
```

在检查 `alert.log` 的末尾时，留意表明以下类型问题的错误：

*   归档进程因磁盘空间不足而挂起
*   文件系统空间耗尽
*   表空间空间耗尽
*   ORA-600 或 7445 错误
*   缓冲区高速缓存或共享池内存耗尽
*   表明数据文件缺失或损坏的介质错误
*   表明写归档日志存在问题的错误；例如，

```
ORA-19502: write error on file "/ora01/fra/o18c/archivelog/...
```

对于 `alert.log` 文件中列出的严重错误消息，几乎总有一个对应的跟踪文件。例如，上述错误消息的伴随信息如下：

```
Errors in file /orahome/app/oracle/diag/rdbms/o18c/O18C/trace/O18C_ora_5665.trc
```

检查跟踪文件通常（但不总是）能为问题提供更深入的洞察。

## 使用 ADRCI 工具查看 alert.log

你可以使用 ADRCI 工具来查看 `alert.log` 文件的内容。从操作系统运行以下命令来启动 ADRCI 工具：

```
$ adrci
```

你应该会看到一个提示符：

```
adrci>
```

使用 `SHOW ALERT` 命令来查看 `alert.log` 文件：

```
adrci> show alert
```

如果服务器上有多个 Oracle 主目录，系统将提示你选择要查看哪个 `alert.log`。`SHOW ALERT` 命令将使用操作系统设置为默认编辑器的工具来打开 `alert.log`。在 Linux/Unix 系统上，默认编辑器来源于操作系统 `EDITOR` 变量（通常设置为 `vi` 之类的工具）。

**提示**
当显示 `alert.log` 时，如果你不熟悉 `vi` 并想退出，首先按 Escape 键，然后按住 Shift 键的同时按下冒号 (`:`) 键。接着，输入 `q!`。这应该会退出 `vi` 编辑器并返回到 ADRCI 提示符。

你可以在 ADRCI 中使用 `SET EDITOR` 命令覆盖默认编辑器。此示例将默认编辑器设置为 `emacs`：

```
adrci> set editor emacs
```

你可以使用 `TAIL` 选项查看 `alert.log` 中最后的 `N` 行。以下命令显示 `alert.log` 的最后 50 行：

```
adrci> show alert -tail 50
```

如果你有多个 Oracle 主目录，可能会看到如下消息：

```
DIA-48449: Tail alert can only apply to single ADR home
```

ADRCI 工具不会假设你想在服务器上使用一个 Oracle 主目录而不是另一个。要专门为 ADRCI 工具设置 Oracle 主目录，首先使用 `SHOW HOMES` 命令显示所有可用的 Oracle 主目录：

```
adrci> show homes
```

以下是此服务器的一些示例输出：

```
diag/rdbms/o18c/O18C
diag/rdbms/o18cp/O18CP
```

现在，使用 `SET HOMEPATH` 命令。这将把 `HOMEPATH` 设置为 `diag/rdbms/o18c/O18C`：

```
adrci> set homepath  diag/rdbms/o18c/O18C
```

要持续显示文件末尾，请使用此命令：

```
adrci> show alert -tail -f
```

按 Ctrl+C 可以中断对 `alert.log` 文件的持续查看。要显示 `alert.log` 中包含特定字符串的行，请使用 `MESSAGE_TEXT LIKE` 命令。此示例显示包含 `ORA-27037` 字符串的消息：

```
adrci> show alert -p "MESSAGE_TEXT LIKE '%ORA-27037%'"
```

你将看到一个文件，其中包含 `alert.log` 中与指定字符串匹配的所有行。

**提示**
有关如何使用 ADRCI 工具的完整详细信息，请参阅《Oracle 数据库实用程序指南》。

## 通过操作系统工具识别瓶颈

在 Oracle 领域，有时人们倾向于假设你为一台 Oracle 数据库配备了专用机器。此外，这个数据库是最新的 Oracle 版本，已完全打补丁，并由复杂的图形化工具监控。这个数据库环境是完全自动化的，并通过可视化工具快速定位问题、高效隔离和解决问题来保持无故障运行。如果你生活在这样理想的世界中，那么你可能不需要本章的任何内容。

让我描绘一个略有不同的场景：一个环境，一台机器上运行着十几个数据库。有一个 MySQL 数据库；一个 PostgreSQL 数据库；以及 Oracle 11g、12c 和 18c 版本的混合数据库。此外，许多这些旧数据库使用的是 Oracle 的非终端版本，因此不受 Oracle 支持。由于业务无法承担可能破坏依赖这些数据库的应用程序的风险，没有计划升级任何这些不受支持的数据库。注意：此时也应理解对业务的安全风险；尽管如此，多数据库平台是可能的。

那么，当有人报告数据库应用程序性能不佳时，在这种环境中该怎么做？在这种情况下，通常是其他数据库中的某些东西导致同一台机器上的其他应用程序表现不佳。可能不是 Oracle 进程或 Oracle 数据库导致了问题。

在这种情况下，从使用操作系统工具开始调查问题几乎总是更有效。操作系统工具与数据库无关。操作系统性能工具有助于精确定位资源消耗最多的位置，无论数据库供应商或版本如何。

在 Linux/Unix 环境中，有多种工具可用于监控资源使用情况。表 21-1 总结了用于诊断性能问题的最常用操作系统实用程序。熟悉这些操作系统命令的工作原理以及如何解读输出，将使你能够更好地诊断服务器性能问题，尤其是当性能拖累盒上所有其他应用程序的是非 Oracle 甚至是非数据库进程时。

**表 21-1** 性能和监控实用程序

| 工具 | 用途 |
| --- | --- |
| `vmstat` | 监控进程、CPU、内存和磁盘 I/O 瓶颈 |
| `top` | 识别消耗资源最多的会话 |
| `watch` | 定期运行另一个命令 |
| `ps` | 识别消耗 CPU 和内存最多的会话；用于识别消耗系统资源最多的 Oracle 会话 |
| `mpstat` | 报告 CPU 统计信息 |
| `sar` | 显示当前和历史的 CPU、内存、磁盘 I/O 和网络使用情况 |
| `free` | 显示空闲和已使用的内存 |
| `df` | 报告空闲磁盘空间 |
| `du` | 显示磁盘使用情况 |
| `iostat` | 显示磁盘 I/O 统计信息 |
| `netstat` | 报告网络统计信息 |

在诊断性能问题时，确定操作系统在何处受限是很有用的。例如，尝试识别问题是否与 CPU、内存或 I/O 相关，或者是这些因素的组合。


