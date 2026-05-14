# 第 2 章 安装与配置

许多设置需要重启 `mssql-server` 服务。在大多数情况下，当您设置一个选项时，如果该设置需要重启，系统应会提示您执行重启操作。

例如，以下是启用 SQL Server 代理（从 SQL Server 2017 CU4 开始）的方法：
```
sudo /opt/mssql/bin/mssql-conf set sqlagent.enabled true
```

以下是您可能需要考虑的一些设置：

`filelocation.defaultdatadir`：这是创建数据库时存储数据库文件的默认目录。标准默认值是 `/var/opt/mssql/data`。如果更改此默认目录，必须将所有权更改为 `mssql` 组和用户。

`filelocation.defaultlogdir`：这是创建数据库时存储事务日志文件的默认目录。标准默认值是 `/var/opt/mssql/data`。如果更改此默认目录，必须将所有权更改为 `mssql` 组和用户。

`filelocation.errorlogfile`：这是存储 `ERRORLOG` 和其他“日志”文件的默认目录。标准默认值是 `/var/opt/mssql/log`。如果更改此默认目录，必须将所有权更改为 `mssql` 组和用户。

`network.tcpport`：这将成为 SQL Server 侦听的新 TCP 端口号。端口 1433 是默认值，通常没有其他应用程序使用它，但可能存在冲突，因此此选项允许您更改默认值。如果更改了默认 TCP 端口，在连接到 SQL Server 时需要指定端口。例如，如果将 TCP 端口更改为 1401，则需要使用 `sqlcmd` 等工具按如下方式连接到本地 Linux 服务器上的 SQL Server：
```
sqlcmd -Usa -S localhost,1401
```

**注意** 与端口 1433 一样，请确保配置 `firewalld` 以打开 SQL Server 使用的新端口。

`filelocation.defaultbackupdir`：这是使用 `T-SQL BACKUP` 命令时存储 SQL Server 备份的默认目录。标准默认值是 `/var/opt/mssql`。如果更改了备份的默认目录，必须将所有权更改为 `mssql` 组和用户。

`telemetry.customerfeedback`：默认情况下，SQL Server 收集有关 SQL Server 配置和性能的信息。这有助于 Microsoft 改进当前和未来版本的 SQL Server 开发。此反馈不收集任何客户数据。在付费版本的 SQL Server 上，您可以使用此 `mssql-conf` 设置禁用此信息收集。有关客户反馈的完整透明讨论，请参阅此文档：[`docs.microsoft.com/sql/linux/sql-server-linux-customer-feedback`](https://docs.microsoft.com/sql/linux/sql-server-linux-customer-feedback)。

我建议您花时间查看所有可能的配置设置，看看其他设置是否适合您对 SQL Server 的使用。

`unset`：允许您将使用 `set` 选项所做的更改恢复为该设置的默认值。您也可以使用 `set` 参数将值设置回其默认值，但您可能不记得那些默认设置。

例如，基于前面启用 SQL Server 代理的示例，您可以使用 `set` 参数将 `sqlagent.enabled` 值设置为 `false`，或者您可以执行：
```
sudo /opt/mssql/bin/mssql-conf unset sqlagent.enabled
```

**注意** 许多 `unset` 选项需要重启 `mssql-server` 服务才能生效，但截至 SQL Server 2017 CU4，系统不会提示您执行此操作。经验法则是，如果等效的 `set` 选项需要重启，则重启 SQL Server。

`traceflag`：跟踪标志是在许多不同级别影响 SQL Server 行为的“旋钮”。您将在本书中了解到可用于所有类型的


场景。某些跟踪标志需要在 SQL Server 启动时启用，或启用后使其全局应用于所有 SQL Server 会话。对于这些场景，请使用 `mssql-conf` 的 `traceflag` 参数。例如，若希望将关于死锁的诊断信息捕获到 SQL Server 错误日志中，您可以这样开启 `traceflag 1222`：

`sudo /opt/mssql/bin/mssql-conf traceflag 1222 on`

## 第 2 章   安装与配置

有关 SQL Server 跟踪标志的完整列表，请参阅此文档：[`docs.microsoft.com/sql/t-sql/database-console-commands/dbcc-traceon-trace-flags-transact-sql`](https://docs.microsoft.com/sql/t-sql/database-console-commands/dbcc-traceon-trace-flags-transact-sql)。

以下是其他参数的简要说明：

- `list`：转储出可与 `set` 或 `unset` 参数一起使用的有效选项。

- `set-sa-password`：用于重置 `sa` 密码。

- `set-collation`：用于更改 SQL Server 中数据库的默认排序规则。有关如何使用此选项的完整说明，请参阅我们的文档：[`docs.microsoft.com/sql/linux/sql-server-linux-configure-mssql-conf#collation`](https://docs.microsoft.com/sql/linux/sql-server-linux-configure-mssql-conf#collation)。

- `set-edition`：使用此选项可更改您在安装期间指定的 SQL Server 版本。系统将提示您选择版本，与使用 `mssql-conf` 的 `setup` 选项时相同。

- `validate`：每当您使用 `mssql-conf` 的任何选项进行配置更改时，这些值会存储在名为 `/var/opt/mssql/mssql.conf` 的文件中。这是一个文本文件，SQL Server 在启动时读取它，以使用除默认值外的各种选项。该文件的完整格式在我们的文档中有描述：[`docs.microsoft.com/sql/linux/sql-server-linux-configure-mssql-conf?#mssql-conf-format`](https://docs.microsoft.com/sql/linux/sql-server-linux-configure-mssql-conf?#mssql-conf-format)。您可以手动修改此文件，而不用 `mssql-conf` 脚本，但我们建议您使用脚本以避免任何错误。`mssql-conf` 的 `validate` 参数可用于确保文件格式和条目正确。

**注意**：`mssql.conf` 文件的一种可能用途是在 Docker 容器中，前提是将此文件存储在持久卷上。

## 第 2 章   安装与配置

##### SQL Server 实例配置

让我们花点时间看看 SQL Server 引擎支持的 SQL Server 实例配置还有哪些其他选项。这包括 T-SQL 系统存储过程 `sp_configure` 和 T-SQL 语句 `ALTER SERVER CONFIGURATION`。这些语句用于控制应用于整个 SQL Server 实例的配置选项。

#### sp_configure

`sp_configure` 是一个 T-SQL 系统命令，称为*系统存储过程*，可用于配置 SQL Server 引擎内的各种选项。这些选项持久化在名为 `master` 的系统数据库中。默认情况下，只有大约 20 个左右的配置选项可供选择。有一些高级选项，只有在您先将 "show advanced options" 选项设置为 1 后才会可见。

任何用户都可以运行此命令来查看可能的值，但默认情况下，只有系统管理员（例如 SQL Server 中的 `sa` 登录名）可以更改选项，因为这些更改可能对整个服务器产生影响。在本书的剩余章节中，我将讨论各种配置选项供您考虑。

所有配置更改都需要执行 T-SQL `RECONFIGURE`。



为使某些更改生效，即使在执行了 `RECONFIGURE` 后，其中一些仍需要重启 SQL Server 服务。

要查看完整的选项列表和运行此 T-SQL 命令的语法，请参阅我们文档中的此页面：[sp_configure (Transact-SQL)](https://docs.microsoft.com/sql/relational-databases/system-stored-procedures/sp-configure-transact-sql)。

### ALTER SERVER CONFIGURATION

`sp_configure` 存储过程适用于需要简单数值的单值配置选项。某些服务器范围的配置选项可能需要更复杂的值或选项。因此，专门创建了 `ALTER SERVER CONFIGURATION` T-SQL 命令用于此目的。

一个例子是 `PROCESS AFFINITY`，它可用于控制 SQL Server 将其线程调度到哪些 CPU 或 NUMA 节点上执行。

### 第 2 章：安装与配置

例如，在多节点系统上，要仅将 SQL Server 线程调度到 NUMA 节点 0 上，您可以运行此 T-SQL 命令：

```sql
ALTER SERVER CONFIGURATION SET PROCESS AFFINITY NUMANODE=0
```

与 `sp_configure` 选项一样，我将在本书后续章节中讨论其中的几种。与 `sp_configure` 类似，这些设置大多需要系统管理员才能进行更改。有关完整选项列表、语法和权限，请参阅我们的文档页面：[ALTER SERVER CONFIGURATION (Transact-SQL)](https://docs.microsoft.com/sql/t-sql/statements/alter-server-configuration-transact-sql)。

##### Linux 上的 Windows 配置选项

在 Windows 上安装 SQL Server 时，通常会使用一些配置选项和选择。以下是这些选项的摘要，以及它们如何适用于 Linux 上的 SQL Server。

#### 锁定页面

Windows 支持一个概念，即如果应用程序使用地址窗口扩展 (`AWE`) API，则可以避免内存工作集修剪。如果使用企业版或标准版，并且 SQL Server 服务帐户具有 `内存中锁定页面` 权限，则默认在 SQL Server 上启用此功能。

Linux 没有 `AWE` 或锁定页面的概念，因此此选项不适用于 Linux 上的 SQL Server。相反，Linux 有根据内存消耗对进程进行分页甚至终止进程的概念。Linux 上的 SQL Server 提供了选项来防止此类情况。我将在第 6 章讨论这个主题。

###### 即时文件初始化

为加快大文件大小的初始化，Windows 提供了一个名为 `SetFileValidData()` 的 API。如果 SQL Server 服务帐户被授予了 `执行卷维护任务` 权限，SQL Server 将使用此 API 创建或修改数据库文件。

Linux 默认执行此类初始化，因为 Host Extension 层中的 SQL Server 使用了 `fallocate()` Linux API。因此，对于 Linux 上的 SQL Server，无需配置此选项。

### 第 2 章：安装与配置

###### 大页面

Windows 上的 SQL Server 在启用跟踪标志 `834` 时，可以使用称为大页面的概念。Linux 上的 SQL Server 依赖于称为透明大页面 (`THP`) 的概念进行内存分配。虽然跟踪标志 `834` 在 Linux 上的 SQL Server 中仍然存在，但我们仅建议在特定场景的高端系统上使用此标志。此跟踪标志的指导方针包含在 Microsoft 支持文章中：[在高性能工作负载中运行时 SQL Server 的调优选项](https://support.microsoft.com/help/920093/tuning-options-for-sql-server-when-running-in-high-performance-workloads)。



[performance-workloa.](https://support.microsoft.com/help/920093/tuning-options-for-sql-server-when-running-in-high-performance-workloa)

