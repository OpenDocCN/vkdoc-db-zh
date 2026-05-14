# 启动和停止数据库

在启动和停止 Oracle 实例之前，你必须设置正确的操作系统变量（本章前面已介绍）。你还需要有权访问特权操作系统账户或特权数据库用户账户。以特权用户身份连接允许你执行管理任务，例如启动、停止和创建数据库。你可以使用操作系统认证或密码文件以特权用户身份连接到你的数据库。

## 理解操作系统认证

操作系统认证意味着，如果你能通过授权的操作系统账户登录到数据库服务器，你将被允许连接到你的数据库，而无需额外的密码。一个简单的示例演示了这一概念。首先，使用 `id` 命令显示 `oracle` 用户所属的操作系统组：

```bash
$ id
uid=500(oracle) gid=506(oinstall) groups=506(oinstall),507(dba),508(oper)
```

接下来，使用 `SYSDBA` 权限连接到数据库，特意使用了错误（无效）的用户名和密码：

```bash
$ sqlplus bad/notgood as sysdba
```

我现在可以验证已成功以 `SYS` 用户建立连接：

```sql
SYS@o18c> show user
USER is "SYS"
```



怎么可能使用错误的用户名和密码连接到数据库呢？实际上，这并非坏事（可能你最初会这样认为）。之前的连接之所以有效，是因为 Oracle 忽略了提供的用户名/密码，用户首先是通过 **操作系统认证** 进行验证的。在该示例中，`oracle` 操作系统用户属于 `dba` 操作系统组，因此无需提供正确的用户名和密码，即可使用 `SYSDBA` 权限建立与数据库的本地连接。关于操作系统组及其与相应数据库权限映射的完整描述，请参见第 1 章中的表 1-1。典型的组包括 `dba` 和 `oper`；这些组分别对应 `sysdba` 和 `sysoper` 数据库权限。`sysdba` 和 `sysoper` 权限允许您执行管理任务，例如启动和停止数据库。

在 Windows 环境中，会自动创建一个操作系统组（通常命名为 `ora_dba`），并将其分配给安装 Oracle 软件的操作系统用户。您可以按照以下方式验证哪些操作系统用户属于 `ora_dba` 组：选择 开始 ➤ 控制面板 ➤ 管理工具 ➤ 计算机管理 ➤ 本地用户和组 ➤ 组。您应该会看到一个名为 `ora_dba` 的组。您可以单击该组并查看哪些操作系统用户被分配到其中。此外，为了使操作系统认证在 Windows 环境中生效，您的 `sqlnet.ora` 文件中必须包含以下条目：

```
SQLNET.AUTHENTICATION_SERVICES=(NTS)
```

`sqlnet.ora` 文件通常位于 `ORACLE_HOME/network/admin` 目录中。

#### 启动数据库

启动和停止数据库是您需要频繁执行的任务。要启动/停止数据库，请使用具有 `sysdba` 或 `sysoper` 权限的用户帐户连接，并执行 `startup` 和 `shutdown` 语句。以下示例使用操作系统认证连接到数据库：

```
$ sqlplus / as sysdba
```

以特权帐户连接后，您可以启动数据库，如下所示：

```
SQL> startup;
```

要使前述命令生效，`ORACLE_HOME/dbs` 目录中需要存在 `spfile` 或 `init.ora` 文件。有关详细信息，请参阅本章前面的“步骤 2：配置初始化文件”部分。

## 注意

在 DBA 圈内，快速连续地停止并重启数据库俗称“重启数据库”。

当您的实例成功启动时，您应该会看到 Oracle 的消息，表明系统全局区 (`SGA`) 已分配。数据库将被挂载然后打开：

```
ORACLE instance started.
Total System Global Area  313159680 bytes
Fixed Size                  2259912 bytes
Variable Size             230687800 bytes
Database Buffers           75497472 bytes
Redo Buffers                4714496 bytes
Database mounted.
Database opened.
```

从前述输出可以看出，在打开 Oracle 数据库时，数据库启动操作经历了三个不同的阶段：

1.  启动实例

2.  挂载数据库

3.  打开数据库

在启动数据库时，您可以逐步执行这些阶段。首先，启动 Oracle 实例（后台进程和内存结构）：

```
SQL> startup nomount;
```

接下来，挂载数据库。此时，Oracle 会读取控制文件：

```
SQL> alter database mount;
```

最后，打开数据文件和联机重做日志文件：

```
SQL> alter database open;
```

此启动过程如图 2-2 所示。

![../images/214899_3_En_2_Chapter/214899_3_En_2_Fig2_HTML.jpg](img/214899_3_En_2_Fig2_HTML.jpg)

图 2-2 Oracle 启动阶段

当您发出不带任何参数的 `STARTUP` 语句时，Oracle 会自动逐步执行三个启动阶段（nomount、mount、open）。在大多数情况下，您将发出不带参数的 `STARTUP` 语句来启动数据库。表 2-3 描述了可与数据库 `STARTUP` 语句一起使用的参数的含义。

表 2-3 `startup` 命令可用参数

| 参数 | 含义 |
| --- | --- |
| `FORCE` | 在重启之前使用 `ABORT` 关闭实例；对于排查启动问题很有用；通常不使用 |
| `RESTRICT` | 仅允许具有 `RESTRICTED SESSION` 权限的用户连接到数据库 |
| `PFILE` | 指定启动实例时要使用的客户端参数文件 |
| `QUIET` | 在启动实例时抑制 `SGA` 信息的显示 |
| `NOMOUNT` | 启动后台进程并分配内存；不读取控制文件 |
| `MOUNT` | 启动后台进程，分配内存，并读取控制文件 |
| `OPEN` | 启动后台进程，分配内存，读取控制文件，并打开联机重做日志和数据文件 |
| `OPEN RECOVER` | 在打开数据库之前尝试进行介质恢复 |
| `OPEN READ ONLY` | 以只读模式打开数据库 |
| `UPGRADE` | 在升级数据库时使用 |
| `DOWNGRADE` | 在降级数据库时使用 |

#### 停止数据库

通常，您使用 `SHUTDOWN IMMEDIATE` 语句来停止数据库。`IMMEDIATE` 参数指示 Oracle 停止数据库活动并回滚所有未完成的事务：

```
SQL> shutdown immediate;
Database closed.
Database dismounted.
ORACLE instance shut down.
```

表 2-4 详细定义了可与 `SHUTDOWN` 语句一起使用的参数。在大多数情况下，`SHUTDOWN IMMEDIATE` 是一种可接受的关闭数据库方法。如果您发出不带参数的 `SHUTDOWN` 命令，则等同于发出 `SHUTDOWN NORMAL`。

表 2-4 `SHUTDOWN` 命令可用参数

| 参数 | 含义 |
| --- | --- |
| `NORMAL` | 等待用户退出活动会话后再关闭。 |
| `TRANSACTIONAL` | 等待事务完成，然后终止会话。 |
| `TRANSACTIONAL LOCAL` | 仅对本地实例执行事务性关闭。 |
| `IMMEDIATE` | 立即终止活动会话。未完成的事务将被回滚。 |
| `ABORT` | 立即终止实例。事务将被终止但不会回滚。 |

启动和停止数据库是一个相当简单的过程。如果环境设置正确，您应该能够连接到数据库并发出适当的 `STARTUP` 和 `SHUTDOWN` 语句。

## 提示

如果您在启动或停止数据库时遇到任何问题，请查看告警日志以获取详细信息。告警日志通常包含与任何问题相关的重要消息。

您很少需要使用 `SHUTDOWN ABORT` 语句。通常，`SHUTDOWN IMMEDIATE` 就足够了。话虽如此，使用 `SHUTDOWN ABORT` 并没有错。如果 `SHUTDOWN IMMEDIATE` 因任何原因无法工作，那么就使用 `SHUTDOWN ABORT`。同时请记住，在 `SHUTDOWN ABORT` 命令之后启动时，数据库将需要恢复介质文件，并且可能需要相当长的时间来处理这些文件。

在极少数情况下，`SHUTDOWN ABORT` 可能会失败。在这些情况下，您可以使用 `ps -ef | grep smon` 来定位 Oracle 系统监控进程，然后使用 Linux/Unix 的 `kill` 命令来终止实例。当您杀死必需的 Oracle 后台进程时，这会导致实例中止。显然，您应该只在最后 resort 时才使用操作系统 `kill` 命令。



