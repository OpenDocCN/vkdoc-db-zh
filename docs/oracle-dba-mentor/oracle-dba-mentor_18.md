# 其他工具和实用程序

本章介绍了 Oracle 数据库管理员可能使用的一些工具和实用程序。您的大部分时间将花在`SQL*Plus`和/或`SQL Developer`上。许多数据库管理员也整天使用`Enterprise Manager`。由于篇幅限制，本章无法更详细地讨论这些工具。毕竟，`Enterprise Manager`本身就值得一本书来介绍。与 Oracle 的任何事物一样，您必须自己探索和学习更多。

工具和实用程序并不限于本章到目前为止所见的内容。有许多实用程序您会时不时地用到。在本节中，我将讨论其中一些实用程序，并告知您它们的用途。重要的是了解有哪些工具，以及每个工具擅长执行什么工作。一旦知道使用哪个工具，您就可以立即集中精力到适当的领域。以下是各种工具和实用程序的列表：

*   `Database Upgrade Assistant (DBUA)`：一个 GUI 向导，引导您完成升级过程。未来的章节将更详细地展示此工具。
*   `Data Pump`：一组用于将数据从一个 Oracle 数据库导出和导入到另一个 Oracle 数据库的实用程序。
*   `SQL*Loader`：一个命令行实用程序，用于将文本文件数据导入到 Oracle 数据库中。
*   `Automatic Diagnostic Repository Command Interpreter (adrci)`：一个命令行实用程序，用于管理诊断跟踪文件。它也可用于打包跟踪文件，发送给 Oracle 支持部门进行分析。
*   `Database Verify (dbv)`：一个命令行实用程序，用于检查数据库的数据文件是否损坏。
*   `Character Set Scanner (csscan)`：一个命令行实用程序，用于在更改数据库字符集之前扫描数据库。此实用程序将有助于发现字符集转换导致的问题。
*   `Listener Control (lsnrctl)`：一个命令行实用程序，用于管理监听器。我们在本书中已经看到了许多示例。
*   `Oracle Error (oerr)`：一个命令行实用程序，用于显示特定错误代码的错误消息。例如，在终端窗口中，键入`oerr ora 100`以获取`ORA-00100`错误消息的文本帮助。
*   `Oracle Password Utility (orapwd)`：一个命令行实用程序，用于管理 Oracle 的密码文件，该文件用于`SYSDBA`和`SYSOPER`账户的外部身份验证。
*   `Orion`：一个命令行实用程序，用于基准测试磁盘性能。
*   `Relink`：一个命令行实用程序，用于重新编译 Oracle 软件。
*   `Recovery Manager (RMAN)`：一个命令行实用程序，用于备份和恢复 Oracle 数据库。`RMAN`在第 7 章中讨论过。
*   `tkprof`：一个命令行实用程序，可以将一些跟踪文件重新格式化为更易于人类阅读的输出。
*   `TNS Ping (tnsping)`：一个命令行实用程序，用于确定`TNS`别名是否能到达有效的监听器。此实用程序在第 9 章中讨论过。
*   `sqlcl`：一个命令行实用程序，是`SQL*Plus`的替代品。`SQL*Plus`多年来没有重大变化。`sqlcl`实用程序由 Oracle 创建`SQL Developer`的同一团队提供。它被宣传为“`SQL Developer`遇见`SQL*Plus`”。这是列表中默认未安装的唯一一个 Oracle 产品。
*   `Swingbench`：一个免费的实用程序，可从[`www.dominicgiles.com`](http://www.dominicgiles.com)获取，可用于对 Oracle 数据库进行性能测试。这不是 Oracle 产品，默认未安装。
*   `HammerDB`：一个开源实用程序，可用于对 Oracle 和其他数据库平台进行性能测试。这不是 Oracle 产品，默认未安装。

此列表中的大多数工具和实用程序由 Oracle Corporation 提供，在您安装 Oracle 数据库软件时即已存在。您也可以使用许多第三方工具。其中一些通常需要额外费用。您和您的公司将不得不评估这些产品，看看它们是否值得额外成本。

## 继续前行

本章花了一些时间讨论 Oracle DBA 可用的一些工具。在使用这些工具时，您将更多地了解它们以及如何使用它们来完成工作。

在下一章中，我们将转向 Oracle 诊断文件。数据库管理员知道如何利用诊断和跟踪文件来了解 Oracle 数据库出了什么问题，这一点很重要。

