# 设置 Oracle 环境变量

## 安装设置

1. 将 oraset 文件置于 `/etc` (Linux 系统) 或 `/var/opt/oracle` (Solaris 系统) 目录中
2. 确保 `/etc` 或 `/var/opt/oracle` 目录位于 `$PATH` 环境变量中

## 使用方法

- 批处理模式：`. oraset`
- 菜单模式：`. oraset`

---

```
if [ -f /etc/oratab ]; then
OTAB=/etc/oratab
elif [ -f /var/opt/oracle/oratab ]; then
OTAB=/var/opt/oracle/oratab
else
echo 'oratab file not found.'
exit
fi
#
if [ -z $1 ]; then
SIDLIST=$(egrep -v '#|\*' ${OTAB} | cut -f1 -d:)
```



## PS3 指示了 Bash `select` 命令要使用的提示符。
`PS3='SID? '`
`select` sid in `${SIDLIST}`; do
if [ -n `$sid` ]; then
`HOLD_SID=$sid`
`break`
fi
`done`
`else`
if `egrep -v '#|\*' ${OTAB}` | `grep -w "${1}:">/dev/null`; then
`HOLD_SID=$1`
else
echo "SID: `$1` not found in `$OTAB"`
fi
`shift`
`fi`
#
`export ORACLE_SID=$HOLD_SID`
`export ORACLE_HOME=$(egrep -v '#|\*' $OTAB|grep -w $ORACLE_SID:|cut -f2 -d:)`
`export ORACLE_BASE=${ORACLE_HOME%%/product*}`
`export TNS_ADMIN=$ORACLE_HOME/network/admin`
`export ADR_BASE=$ORACLE_BASE/diag`
`export PATH=$ORACLE_HOME/bin:/usr/ccs/bin:/opt/SENSsshc/bin/\
:/bin:/usr/bin:.:/var/opt/oracle:/usr/sbin`
`export LD_LIBRARY_PATH=/usr/lib:$ORACLE_HOME/lib`

## 使用 oraset 脚本

你可以从命令行或启动文件（如 `.profile`、`.bash_profile` 或 `.bashrc`）中运行 `oraset` 脚本。要从命令行运行 `oraset`，请将 `oraset` 文件放置在标准位置，例如 `/var/opt/oracle`（Solaris）或 `/etc`（Linux），并按如下方式运行：

```
$ . /etc/oraset
```

请注意，从命令行运行此命令的语法要求点号 (`.`) 与命令的其余部分之间有一个空格。当你从命令行运行 `oraset` 时，应该会看到类似以下的菜单：

```
1) o18c
2) rcat
SID?
```

在此示例中，你现在可以输入 `1` 或 `2` 来为要使用的任何数据库设置所需的 OS 变量。这允许你以交互方式设置 OS 变量，无论服务器上有多少个数据库安装。

你也可以从 OS 启动文件中调用 `oraset` 文件。以下是 `.bashrc` 文件中的一个示例条目：

```
. /etc/oraset
```

现在，每次登录服务器时，系统都会显示一个选择菜单，你可以使用该菜单来指示要为其设置 OS 变量的数据库。如果希望 OS 变量自动设置为特定数据库，请在 `.bashrc` 文件中放入如下条目：

```
. /etc/oraset o18c
```

前面的行将为 `o18c` 数据库运行 `oraset` 文件，并相应地设置 OS 变量。

## 创建数据库

本节说明如何使用 SQL*Plus 的 `CREATE DATABASE` 语句手动创建 Oracle 数据库。创建数据库需要以下步骤：

1.  设置 OS 变量。
2.  配置初始化文件。
3.  创建所需的目录。
4.  创建数据库。
5.  创建数据字典。

以下各节将详细介绍这些步骤中的每一步。

### 步骤 1. 设置 OS 变量

如前所述，在运行 SQL*Plus（或任何其他 Oracle 实用程序）之前，必须设置几个 OS 变量。你可以手动设置这些变量，也可以结合使用文件和脚本来设置变量。以下是一个手动设置这些变量的示例：

```
$ export ORACLE_HOME=/u01/app/oracle/product/18.0.0.0/db_1
$ export ORACLE_SID=o18c
$ export LD_LIBRARY_PATH=/usr/lib:$ORACLE_HOME/lib
$ export PATH=$ORACLE_HOME/bin:$PATH
```

有关这些变量的完整说明以及设置它们的技术，请参阅本章前面的“设置 OS 变量”部分。

### 步骤 2. 配置初始化文件

Oracle 要求在尝试启动实例之前准备好一个初始化文件。初始化文件用于配置内存和控制文件位置等特性。你可以使用两种类型的初始化文件：

*   服务器参数二进制文件 (`spfile`)
*   `init.ora` 文本文件

Oracle 建议你使用 `spfile`，原因如下：

*   你可以使用 SQL `ALTER SYSTEM` 语句修改 `spfile` 的内容。
*   你可以使用远程客户端 SQL 会话启动数据库，而无需本地（客户端）初始化文件。
*   有更多的动态参数可以使用 spfile 设置，且无需任何停机时间。

这些是使用 `spfile` 的充分理由。然而，一些站点仍然使用传统的 `init.ora` 文件。`init.ora` 文件也有优点：

*   你可以使用 OS 文本编辑器直接编辑它。
*   你可以在其中添加注释，详细说明修改历史。

当我第一次创建数据库时，我发现使用 `init.ora` 文件更容易。如果需要，这个文件以后可以轻松转换为 `spfile`（通过 `CREATE SPFILE FROM PFILE` 语句）。在此示例中，我的数据库名称是 `o18c`，因此我将以下内容放入名为 `inito18c.ora` 的文件中，并将文件放在 `ORACLE_HOME/dbs` 目录中：

```
db_name=o18c
db_block_size=8192
memory_target=300M
memory_max_target=300M
processes=200
control_files=(/u01/dbfile/o18c/control01.ctl,/u02/dbfile/o18c/control02.ctl)
job_queue_processes=10
open_cursors=500
fast_start_mttr_target=500
undo_management=AUTO
undo_tablespace=UNDOTBS1
remote_login_passwordfile=EXCLUSIVE
```

确保初始化文件命名正确并位于适当的目录中。这很关键，因为在启动实例时，Oracle 首先在 `ORACLE_HOME/dbs` 目录中按以下顺序查找特定格式的参数文件：

*   `spfile<SID>.ora`
*   `spfile.ora`
*   `init<SID>.ora`

换句话说，Oracle 首先查找名为 `spfile<SID>.ora` 的文件。如果找到，则启动实例；如果未找到，Oracle 会查找 `spfile.ora`，然后是 `init<SID>.ora`。如果这些文件都未找到，Oracle 将抛出错误。如果你不知道 Oracle 查找哪些文件以及查找顺序，这可能会造成一些混淆。例如，你可能对 `init<SID>.ora` 文件进行了更改，并期望在停止并启动实例后实例化该参数。如果存在 `spfile<SID>.ora`，则 `init<SID>.ora` 将被完全忽略。

> **注意**
> 你可以使用 `startup` 命令的 `pfile=<directory/filename>` 子句手动指示 Oracle 在目录中查找文本参数文件；在正常情况下，你不需要这样做。你需要默认行为，即 Oracle 在 `ORACLE_HOME/dbs` 目录（对于 Linux/Unix）中查找参数文件。Windows 上的默认目录是 `ORACLE_HOME/database`。

表 2-1 列出了配置 Oracle 初始化文件时需要考虑的最佳实践。

**表 2-1：初始化文件最佳实践**

| 最佳实践 | 推理依据 |
| :--- | :--- |
| Oracle 建议你使用二进制服务器参数文件 (`spfile`)。 | Spfile 允许对参数进行动态更改。如果存在可接受的维护窗口，那么使用 init.ora 文件也是可以的。 |
| 一般来说，如果你不确定初始化参数的预期用途，就不要设置它们。如有疑问，请使用默认值。 | 设置初始化参数可能对数据库性能产生深远影响。只有在知道结果行为是什么的情况下才修改参数。 |
| 对于 11g 及更高版本，设置 `memory_target` 和 `memory_max_target` 初始化参数。 | 这样做允许 Oracle 为你管理所有内存组件。 |
| 对于 10g，设置 `sga_target` 和 `sga_target_max` 初始化参数。 | 这样做让 Oracle 为你管理大多数内存组件。 |
| 对于 10g，设置 `pga_aggregate_target` 和 `workarea_size_policy`。 | 这样做允许 Oracle 管理用于排序空间的内存。 |
| 从 10g 开始，使用自动 `UNDO` 功能。这通过 `undo_management` 和 `undo_tablespace` 参数设置。 | 这样做允许 Oracle 管理 `UNDO` 表空间的大多数特性。 |
| 将 `open_cursors` 设置为比默认值更高的值。我通常将其设置为 500。活跃的在线事务处理 (OLTP) 数据库可能需要高得多的值。 | 默认值 50 几乎永远不够。即使是一个小型、单用户应用程序也可能超过 50 个打开游标的默认值。 |
| 使用模式 `/<mount_point>/dbfile/<database_name>/control0N.ctl` 命名控制文件。 | 这与 OFA 标准略有不同。我发现这个位置比位于 `ORACLE_BASE` 下更容易导航。 |
| 使用至少两个控制文件，最好位于不同位置、使用不同的磁盘。 | 如果一个控制文件损坏，至少还有一个其他控制文件可用总是一个好主意。 |

### 步骤 3. 创建所需的目录

在尝试创建数据库之前，必须在服务器上创建参数文件或 `CREATE DATABASE` 语句中引用的任何 OS 目录。例如，在上一节的初始化文件中，控制文件定义为：

```
control_files=(/u01/dbfile/o18c/control01.ctl,/u02/dbfile/o18c/control02.ctl)
```

根据前一行，确保你已创建目录 `/u01/dbfile/o18c` 和 `/u02/dbfile/o18c`（根据你的环境进行修改）。在 Linux/Unix 中，你可以使用带有 `-p` 选项的 `mkdir` 命令创建目录（包括所需的任何父目录）：

```
$ mkdir -p /u01/dbfile/o18c
$ mkdir -p /u02/dbfile/o18c
```

同时确保为 `CREATE DATABASE` 语句中引用的数据文件和联机重做日志创建任何所需的目录（参见步骤 4）。对于此示例，需要额外创建以下目录：

```
$ mkdir -p /u01/oraredo/o18c
$ mkdir -p /u02/oraredo/o18c
```

如果你以 `root` 用户身份创建了上述目录，请确保 `oracle` 用户和 `dba` 组已正确设置为目录、子目录和文件的所有者。此示例递归地更改以下目录的所有者和组：



### 步骤 4. 创建数据库

在您设置了操作系统变量、配置了初始化文件并创建了所需的目录之后，现在就可以创建数据库了。本步骤将说明如何使用 `CREATE DATABASE` 语句来创建一个数据库。

在运行 `CREATE DATABASE` 语句之前，您必须通过 `STARTUP NOMOUNT` 语句启动后台进程并分配内存：

```bash
$ sqlplus / as sysdba
SQL> startup nomount;
```

当您发出 `STARTUP NOMOUNT` 语句时，SQL*Plus 会尝试读取 `ORACLE_HOME/dbs` 目录中的初始化文件（参见步骤 2）。`STARTUP NOMOUNT` 语句会实例化 Oracle 使用的后台进程和内存区域。此时，您拥有一个 Oracle 实例，但还没有数据库。

> **注意**
> Oracle 实例被定义为后台进程和内存区域。Oracle 数据库则被定义为磁盘上的物理文件（数据文件、控制文件、在线重做日志）。

下面列出的是一个典型的 Oracle `CREATE DATABASE` 语句：

```sql
CREATE DATABASE o18c
MAXLOGFILES 16
MAXLOGMEMBERS 4
MAXDATAFILES 1024
MAXINSTANCES 1
MAXLOGHISTORY 680
CHARACTER SET AL32UTF8
DATAFILE
'/u01/dbfile/o18c/system01.dbf'
SIZE 500M REUSE
EXTENT MANAGEMENT LOCAL
UNDO TABLESPACE undotbs1 DATAFILE
'/u01/dbfile/o18c/undotbs01.dbf'
SIZE 800M
SYSAUX DATAFILE
'/u01/dbfile/o18c/sysaux01.dbf'
SIZE 500M
DEFAULT TEMPORARY TABLESPACE TEMP TEMPFILE
'/u01/dbfile/o18c/temp01.dbf'
SIZE 500M
DEFAULT TABLESPACE USERS DATAFILE
'/u01/dbfile/o18c/users01.dbf'
SIZE 20M
LOGFILE GROUP 1
('/u01/oraredo/o18c/redo01a.rdo',
'/u02/oraredo/o18c/redo01b.rdo') SIZE 50M,
GROUP 2
('/u01/oraredo/o18c/redo02a.rdo',
'/u02/oraredo/o18c/redo02b.rdo') SIZE 50M,
GROUP 3
('/u01/oraredo/o18c/redo03a.rdo',
'/u02/oraredo/o18c/redo03b.rdo') SIZE 50M
USER sys    IDENTIFIED BY foo
USER system IDENTIFIED BY foo;
```

在此示例中，脚本被放置在一个名为 `credb.sql` 的文件中，并以 `SYS` 用户身份从 SQL*Plus 提示符运行：

```sql
SQL> @credb.sql
```

如果执行成功，您应该会看到以下消息：

```
Database created.
```

> **注意**
> 有关创建可插拔数据库的详细信息，请参见第 22 章。

如果在 `CREATE DATABASE` 语句运行时抛出任何错误，请检查警报日志文件。错误通常发生在所需目录不存在、内存分配不足或超出操作系统限制时。如果您不确定警报日志的位置，请执行以下查询：

```sql
SQL> select value from v$diag_info where name = 'Diag Trace';
```

即使数据库处于 nomount 状态，前面的查询也应该有效。另一种快速找到警报日志文件的方法是从操作系统入手：

```bash
$ cd $ORACLE_BASE
$ find . -name "alert*.log"
```

> **提示**
> 警报日志文件的默认命名格式为 `alert_<SID>.log`。

关于前面的 `CREATE DATABASE` 语句示例，有几点关键内容需要指出。请注意，`SYSTEM` 数据文件被定义为本地管理的。这意味着在此数据库中创建的任何表空间都必须是本地管理的（相对于字典管理）。如果您尝试在此数据库中创建字典管理的表空间，Oracle 将抛出错误。这是期望的行为。

字典管理的表空间使用 Oracle 数据字典来管理区和可用空间，而本地管理的表空间则使用每个数据文件中的位图来管理其区和可用空间。本地管理的表空间具有以下优点：

*   性能得到提升。
*   不需要合并。
*   减少了数据字典中的资源争用。
*   减少了递归空间管理。

另请注意，`TEMP` 表空间被定义为默认临时表空间。这意味着在此数据库中创建的任何用户都会自动分配 `TEMP` 表空间作为其默认临时表空间。在创建数据字典后（参见步骤 5），您可以通过以下查询验证默认临时表空间：

```sql
select *
from database_properties
where property_name = 'DEFAULT_TEMP_TABLESPACE';
```

最后，请注意 `USERS` 表空间被定义为任何未在 `CREATE USER` 语句中定义默认表空间的用户的默认永久表空间。在创建数据字典后（参见步骤 5），您可以通过运行此查询来确定默认表空间：

```sql
select *
from database_properties
where property_name = 'DEFAULT_PERMANENT_TABLESPACE';
```

表 2-2 列出了创建 Oracle 数据库时应考虑的最佳实践。

**表 2-2**
**创建 Oracle 数据库的最佳实践**

| 最佳实践 | 原因说明 |
| --- | --- |
| 谨慎使用 `REUSE` 子句。通常，仅应在重新创建数据库时使用它。 | `REUSE` 子句指示 Oracle 覆盖现有文件，无论它们是否正在使用中。这是危险的。 |
| 使用名称中包含 `TEMP` 的表空间创建默认临时表空间。 | 每个用户都应被分配一个类型为 `TEMP` 的临时表空间，包括 `SYS` 用户。如果未指定默认临时表空间，则将使用 `SYSTEM` 表空间。您绝不希望用户被分配到 `SYSTEM` 作为临时表空间。如果您的数据库没有默认临时表空间，请使用 `ALTER DATABASE DEFAULT TEMPORARY TABLESPACE` 语句来指定一个。 |
| 创建名为 `USERS` 的默认永久表空间。 | 这确保用户被分配一个除 `SYSTEM` 之外的默认永久表空间。如果您的数据库没有默认永久表空间，请使用 `ALTER DATABASE DEFAULT TABLESPACE` 语句来指定一个。 |
| 使用 `USER SYS` 和 `USER SYSTEM` 子句来指定非默认密码。 | 这样做会在创建数据库时为通常是黑客首要目标的数据库账户设置非默认密码。 |
| 创建至少三个重做日志组，每组两个成员。 | 至少三个重做日志组为归档进程在日志切换之间写出归档重做日志提供了时间。两个成员可以镜像在线重做日志成员，提供一定的容错能力。 |
| 给重做日志命名，例如 `redoNA.rdo`。 | 这与 OFA 标准略有不同，但我曾多次遇到扩展名为 `.log` 的文件被意外删除（这本不应发生，但确实发生了）。 |
| 使数据库名称具有一定意义，例如 `PAPRD`、`PADEV1` 或 `PATST1`。 | 这有助于您确定正在操作的是哪个数据库，以及它是生产、开发还是测试环境。 |
| 在创建数据字典时（参见步骤 5）使用 `?` 变量。不要硬编码目录路径。 | SQL*Plus 将 `?` 解释为操作系统 `ORACLE_HOME` 变量中包含的目录。这可以防止您意外地从错误的 `ORACLE_HOME` 版本运行脚本。 |

> **提示**
> 使用 `dbca` 会为您完成 `CREATE DATABASE` 中的许多此类设置。可以设置表空间或使用默认设置，同时确保用户不会在 `SYSTEM` 表空间中创建对象，且不使用默认密码。使用 `dbca` 会将新特性和安全选项作为数据库创建的一部分进行处理。



