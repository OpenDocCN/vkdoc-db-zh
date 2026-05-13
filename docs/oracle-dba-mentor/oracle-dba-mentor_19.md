# 12. 诊断

在前面的章节中，我们设置了一个数据库并开始使用它。在本章中，我们将利用诊断和跟踪文件来了解更多可能困扰 Oracle 数据库的问题。如果 Oracle 实例无法启动或崩溃，诊断文件是开始深入研究问题的绝佳起点。



## 诊断目标地

当今的 Oracle 数据库引擎将其诊断文件和跟踪文件整合到一个称为**诊断目标地**的统一位置。在 Oracle Database 11*g* 之前，这些文件分散在多个位置，使得查找所需内容更加困难。默认情况下，诊断目标地是 Oracle 基目录下的一个子目录。如果不确定其位置，可以使用一个简单的 SQL*Plus 命令 `show parameter` 来查找，如清单 12-1 所示。`show parameter` 命令可用于查看任何初始化参数的值。

```
SQL> show parameter diagnostic_dest
NAME                 TYPE        VALUE
-------------------- ----------- ----------------
diagnostic_dest      string      /u01/app/oracle
清单 12-1
诊断目标地
```

上面的输出对应于 Oracle 基目录的位置。诊断文件位于一个名为 `diag` 的子目录中。在 `/u01/app/oracle/diag`（或您系统上的类似位置）内，还有许多其他子目录，每个可能生成诊断和跟踪文件的组件都有一个对应的目录。例如，数据库引擎会将其跟踪文件写入 `rdbms` 子目录。Oracle 监听器会将其跟踪文件写入 `tnslsnr` 子目录。清单 12-2 展示了我们测试平台上 `/u01/app/oracle/diag` 下的多个子目录。

```
[oracle@dbamentor ~]$ cd /u01/app/oracle/diag
[oracle@dbamentor diag]$ ls -l
total 0
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 afdboot
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 apx
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 asm
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 asmtool
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 bdsql
drwxrwxr-x. 3 oracle oinstall 24 Jul 10 00:14 clients
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 crs
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 diagtool
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 dps
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 em
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 gsm
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 ios
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 lsnrctl
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 netcman
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 ofm
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 plsql
drwxrwxr-x. 2 oracle oinstall  6 Sep 29  2017 plsqlapp
drwxrwxr-x. 3 oracle oinstall 17 Sep 29  2017 rdbms
drwxrwxr-x. 4 oracle oinstall 38 Jul 30 10:30 tnslsnr
清单 12-2
诊断目标地内容
```

在您使用 Oracle 的职业生涯中，您会发现自己会经常访问其中的许多子目录，而其他的则永远不需要您的关注。如果您使用 Oracle 的自动存储管理器 (ASM)，那么您可能偶尔需要查看 `asm` 子目录。如果您从未使用过 ASM，那么您永远不需要此组件的跟踪文件，并且该目录很可能是空的。您可能访问的其他常见目录包括：

- `crs`：如果您安装了 Oracle 的 Grid Infrastructure (GI)，任何 GI 跟踪文件都将写入此处。在早期， GI 曾被称为集群就绪服务 (CRS)，因此目录名由此而来。
- `em`：如果您在此系统上安装并运行了 Enterprise Manager (EM)，您可以在此处找到其文件。
- `rdbms`：如前所述，此目录是 Oracle 数据库引擎写入其诊断跟踪文件的位置。
- `tnslsnr`：Oracle 监听器会将其文件写入此处。

`/u01/app/oracle/diag` 的其他子目录很少使用。

由于一台服务器可以支持多个 Oracle 数据库，每个 Oracle 实例会将跟踪文件写入类似于 `/u01/app/oracle/diag/rdbms/`*数据库名*`/`*实例名* 的目录路径中。

对于我们的测试平台，Oracle 实例名称与数据库名称相同。只有在运行 Oracle 真正应用集群 (RAC) 时，实例名才会与数据库名不同。由于我们的数据库不使用 RAC，因此我们数据库诊断和跟踪文件的目录路径是 `/u01/app/oracle/diag/rdbms/orcl/orcl`。

Oracle 监听器会将其诊断和跟踪文件写入类似于 `/u01/app/oracle/diag/tnslsnr/`*主机名*`/listener` 的路径中。

随着本章的推进，我们将看到更多这些文件。

## 告警日志

有一个特殊的文件，当您管理 Oracle 数据库时，您会很快熟悉它，那就是**告警日志**。很可能您在职业生涯早期就已经接触过这个文件。在我们的测试平台上，告警日志存储在 `/u01/app/oracle/diag/rdbms/orcl/orcl/trace` 中。第 6 章中的说明创建了一个名为 "orcl" 的数据库。如果您使用了不同的名称，那么目录路径会略有不同。

告警日志始终按照 `alert_`*实例名*`.log` 的约定命名，使用实例名。如前所述，除非您运行的是 Oracle RAC，否则实例名与数据库名相同。Oracle 还在 `/u01/app/oracle/diag/rdbms/orcl/orcl/alert` 中创建了一个基于 XML 的告警日志，其中日志文件的每个条目都包裹在 XML 标记标签中。我发现 XML 版本的日志文件难以阅读，但对于解析日志文件的工具来说，此文件非常有用。如果您与 Oracle 支持部门合作，他们可能会要求您发送基于 XML 的告警日志，以便他们用工具解析。否则，我从不查看基于 XML 的告警日志。

清单 12-3 显示了我们测试平台上的基于文本的告警日志。

```
[oracle@dbamentor ~]$ cd /u01/app/oracle/diag/rdbms/orcl/orcl/trace/
[oracle@dbamentor trace]$ ls -l alert*
-rw-r-----. 1 oracle oinstall 466338 Jul 30 14:09 alert_orcl.log
清单 12-3
告警日志位置
```

告警日志应该是 Oracle DBA 排查 Oracle 数据库问题的首选位置之一。Oracle 不会为每个会话的错误或问题都向此处写入信息。这样做会消耗过多资源。相反，告警日志是用于影响 Oracle 数据库及其整体实例的问题。我们将在告警日志中看到 Oracle 实例启动和关闭的消息。

当 Oracle 实例启动时，它会向告警日志写入大量信息性消息。在清单 12-4 中，我们可以看到告警日志中的一些信息。为简洁起见，许多信息已被移除。看看您是否能从包含的内容中发现以下信息：

- 实例启动的日期和时间
- 数据库服务器上的核心数
- Oracle 数据库版本
- Oracle 主目录
- 数据库服务器名称及其操作系统版本
- 参数文件 (spfile) 的位置
- 正在使用的非默认参数
- 实例开放业务的日期和时间


