# 密码文件

密码文件是一个可选文件，它允许远程以 `SYSDBA` 或管理员身份访问数据库。当你尝试启动 Oracle 时，没有可用的数据库来验证密码。当你在本地系统（即，不是通过网络，而是在数据库实例所在的机器上）启动 Oracle 时，Oracle 将使用操作系统（OS）进行身份验证。

Oracle 安装时，安装者会被要求为管理员指定一个操作系统组。通常，在 UNIX/Linux 上，这个组默认是 `DBA`，在 Windows 上是 `ORA_DBA`。然而，它可以是该平台上的任何合法组名。该组是“特殊”的，因为该组中的任何用户都可以“作为 `SYSDBA`”连接到 Oracle，而无需指定用户名或密码。

但是，如果你尝试通过网络连接到远程数据库作为 `SYSDBA`，会发生什么：

```
$ sqlplus sys/foo@localhost:1521/CDB as sysdba
ERROR:
ORA-01017: invalid username/password; logon denied
```

对于 `SYSDBA`，操作系统身份验证在网络连接中不起作用，即使（出于安全原因）不安全的参数 `REMOTE_OS_AUTHENT` 被设置为 true。因此，操作系统身份验证无法工作，而且，如前所述，如果你试图启动一个实例来挂载并打开一个数据库，那么根据定义，此时还没有数据库可供查询身份验证细节。这是一个典型的鸡生蛋、蛋生鸡的问题。

这时就轮到密码文件出场了。密码文件存储了允许通过网络远程以 `SYSDBA` 身份进行身份验证的用户名和密码列表。Oracle 必须使用此文件对他们进行身份验证，而不是使用存储在数据库中的常规密码列表。

那么，让我们来纠正我们的处境。首先，确认 `REMOTE_LOGIN_PASSWORDFILE` 参数设置为其默认值 `EXCLUSIVE`，这意味着只有一个数据库使用给定的密码文件：

```
SQL> show parameter remote_login_passwordfile
NAME                                 TYPE        VALUE
------------------------------------ ----------- ----------
remote_login_passwordfile            string      EXCLUSIVE
```

> 注意
> 此参数的其他有效值是 `NONE`（表示没有密码文件，没有远程 `SYSDBA` 连接）和 `SHARED`（多个数据库可以使用同一个密码文件）。

下一步是使用命令行工具（在 UNIX/Linux 和 Windows 上）`orapwd` 来创建并填充初始密码文件：

```
$ orapwd
Usage: orapwd file= entries= force= asm=
dbuniquename= format= sysbackup= sysdg=
syskm= delete= input_file=
Usage: orapwd describe file=
where
...
There must be no spaces around the equal-to (=) character.
```

当我们登录到拥有 Oracle 软件的操作系统账户时，将使用的命令是：

```
$ orapwd file=orapw$ORACLE_SID password=foo entries=20
```

在我的例子中（我的 `ORACLE_SID` 是 `CDB`），这会创建一个名为 `orapwCDB` 的密码文件。这是大多数 UNIX/Linux 平台上此文件的命名约定（有关您平台上此文件命名的详细信息，请参阅您的安装/操作系统管理员指南），它位于 `$ORACLE_HOME/dbs` 目录中。在 Windows 上，此文件名为 `PW%ORACLE_SID%.ora`，位于 `%ORACLE_HOME%\database` 目录中。你应该在运行创建该文件的命令之前导航到正确的目录，或者在之后将该文件移动到正确的目录。

现在，该文件中唯一的用户是 `SYS`，即使数据库上有其他 `SYSDBA` 账户（它们尚未在密码文件中）。然而，利用这一点，我们第一次可以作为 `SYSDBA` 通过网络连接：

```
$ sqlplus sys/foo@localhost:1521/CDB as sysdba
SQL> show user
USER is "SYS"
```

> 注意
> 如果在此步骤中遇到 ORA-12505 “TNS:listener does not currently know of SID given in connect Descriptor” 错误，这意味着数据库侦听器未为此服务器配置静态注册条目。当数据库实例未启动时，DBA 未允许远程 `SYSDBA` 连接。你需要在 `listener.ora` 配置文件中配置静态服务器注册。请在你所使用的数据库版本的 OTN（Oracle 技术网络）文档搜索页面上搜索带引号的“Configuring Static Service Information”，以获取有关配置此静态服务的详细信息。如果你遇到 ORA-12528 “TNS:listener: all appropriate instances are blocking new connections” 错误，你也可以在 tnsnames.ora 文件中配置 `UR=A` 参数，该参数将允许你连接到被阻止的实例。

我们已经通过身份验证，所以进去了。我们现在可以成功启动、关闭并使用 `SYSDBA` 账户远程管理此数据库。现在，我们有另一个用户 `TKYTE`，他已被授予 `SYSDBA`，但尚无法远程连接：

```
$ sqlplus tkyte/foobar@PDB1 as sysdba
ERROR:
ORA-01017: invalid username/password; logon denied
```

原因是 `TKYTE` 还不在密码文件中。为了将 `TKYTE` 添加到密码文件中，我们需要“重新授予”该账户 `SYSDBA` 权限：

```
$ sqlplus / as sysdba
SQL> alter session set container=PDB1;
SQL> grant sysdba to tkyte;
Grant succeeded.
SQL> exit
$ sqlplus tkyte/foobar@PDB1 as sysdba
SQL>
```

这为我们创建了密码文件中的一个条目，Oracle 现在将保持密码同步。如果 `TKYTE` 更改其密码，旧的密码将停止用于远程 `SYSDBA` 连接，新的密码将开始工作。对于任何是 `SYSDBA` 但尚未在密码文件中的用户，都重复相同的过程。



## 变更跟踪文件

变更跟踪文件是一个可选文件，适用于 Oracle 10*g* 企业版及更高版本。该文件的唯一目的是跟踪自上次增量备份以来哪些数据块已被修改。借助此文件，恢复管理器（`RMAN`）工具可以仅备份实际已修改的数据库块，而无需读取整个数据库。

在 Oracle 运行过程中，随着数据块被修改，Oracle 会可选地维护一个文件，用于告知 `RMAN` 哪些块已被更改。创建此变更跟踪文件相当简单，通过 `ALTER DATABASE` 命令即可完成：

```
$ mkdir -p /opt/oracle/product/btc
$ sqlplus / as sysdba
SQL> alter database enable block change tracking using file '/opt/oracle/product/btc/changed_blocks.btc';
Database altered.
```

**警告**

我将在本书中不时提醒：请谨记，设置参数、修改数据库或进行根本性更改的命令不应轻易执行，肯定应在您的“实际”系统上执行前进行测试。前面的命令实际上会导致数据库执行更多工作，它会消耗资源。

要关闭并移除块变更跟踪文件，需要再次使用 `ALTER DATABASE` 命令：

```
SQL> alter database disable block change tracking;
Database altered.
```

请注意，此命令将删除块变更跟踪文件。它不仅仅是禁用该功能——同时也删除了文件。

**注意**

在某些操作系统（如 Windows）上，您可能会发现，如果按照我的示例操作——创建一个块变更跟踪文件然后禁用它——该文件似乎仍然存在。这是一个特定于操作系统的问题——在许多操作系统上不会发生。只有当您从单个会话中 `CREATE` 和 `DISABLE` 变更跟踪文件时，才会发生这种情况。创建块变更跟踪文件的会话将保持该文件打开，某些操作系统不允许删除先前进程（例如创建该文件的会话进程）已打开的文件。这是无害的；您稍后只需自己删除该文件即可。

您可以在 `ARCHIVELOG` 或 `NOARCHIVELOG` 模式下启用这个新的块变更跟踪功能。但请记住，处于 `NOARCHIVELOG` 模式的数据库（其中每天生成的重做日志不保留）在发生介质（磁盘或设备）故障时无法恢复所有更改！`NOARCHIVELOG` 模式的数据库将来某天会丢失数据。我们将在第 9 章更详细地介绍这两种数据库模式。

## 闪回日志

闪回日志用于支持 `FLASHBACK DATABASE` 命令。闪回日志包含已修改数据库块的“前映像”，可用于将数据库恢复到之前某个时间点的状态。

### 闪回数据库

引入 `FLASHBACK DATABASE` 命令是为了加速原本缓慢的数据库时间点恢复过程。它可以替代完整的数据库还原和使用归档日志进行前滚操作，主要设计用于加速从“事故”中恢复。例如，让我们看看 DBA 可能如何从意外删除的模式中恢复——正确的模式被删除了，但只是在错误的数据库中（本意是在测试环境中删除）。DBA 立即意识到自己犯的错误，并立即关闭数据库。现在该怎么办？

在 `FLASHBACK DATABASE` 功能出现之前，可能会发生以下情况：

1.  DBA 会关闭数据库。
2.  DBA 会从磁带（通常）还原数据库的最后一个完整备份，这通常是一个漫长的过程。通常，这将通过 `RMAN` 使用 `RESTORE DATABASE UNTIL <时间点>` 来启动。
3.  DBA 会还原自备份以来生成的所有且在系统上不可用的归档重做日志。
4.  使用归档重做日志（以及可能在线重做日志中的信息），DBA 将前滚数据库，并在错误的 `DROP USER` 命令执行前的时间点停止前滚。此列表中的步骤 3 和 4 通常通过 `RMAN` 使用 `RECOVER DATABASE UNTIL <时间点>` 来启动。
5.  数据库将以 `RESETLOGS` 选项打开。

这是一个包含多个步骤的非平凡过程，通常会消耗大量时间（当然，这段时间无人可以访问数据库）。导致此类时间点恢复的原因有很多：升级脚本出错、升级失败、有权限的人无意中发布了命令（可能是最常见的原因），或者某个过程将数据完整性问题引入大型数据库（同样，是意外；可能它运行了两次而不是一次，或者它有 bug）。无论原因如何，最终结果都是长时间的停机。

假设您配置了闪回数据库功能，恢复步骤如下：

1.  DBA 关闭数据库。
2.  DBA 启动并加载数据库，然后发出闪回数据库命令，可以使用 `SCN`（Oracle 内部时钟）、还原点（指向 `SCN` 的指针）或时间戳（墙上时钟时间），其精度在几秒之内。
3.  DBA 使用 `resetlogs` 打开数据库。

闪回时，您可以闪回整个容器数据库（包括所有可插拔数据库）或仅闪回特定的可插拔数据库。当闪回可插拔数据库时，您将使用为整个数据库或特定可插拔数据库创建的还原点。闪回可插拔数据库不需要您使用 `resetlogs` 打开数据库，但要求您连接到已闪回的可插拔数据库后发出 `RECOVER DATABASE` 命令。

要使用闪回功能，您的数据库必须处于 `archivelog` 模式，并且必须设置快速恢复区（`FRA`）（因为闪回日志存储在 `FRA` 中）。要使用普通还原点，必须在数据库中启用闪回日志记录。保证还原点不需要数据库中的闪回日志记录。

要查看闪回日志记录状态，请运行以下查询：

```
SQL> select flashback_on from v$database;
FLASHBACK_ON
--------------------------------
NO
```

要在根容器中启用闪回日志记录，请执行以下操作：

```
$ sqlplus / as sysdba
SQL> alter database flashback on;
```

最后一点是，您需要在真正需要使用闪回功能之前就设置好它。这不是在损坏发生后可以启用的功能；您必须做出有意识的决定来使用它，无论是持续开启还是用于设置还原点。



