# 第 2 章 ■ 数据库实现

## `PS3` 指示了 Bash `select` 命令要使用的提示符。

```bash
PS3='SID? '

select sid in ${SIDLIST}; do

if [ -n $sid ]; then

HOLD_SID=$sid

break

fi

done

else

if grep -v '#' ${OTAB} | grep -w "${1}:">/dev/null; then HOLD_SID=$1

else

echo "SID: $1 not found in $OTAB"

fi

shift

fi

#

export ORACLE_SID=$HOLD_SID

export ORACLE_HOME=$(grep -v '#' $OTAB|grep -w $ORACLE_SID:|cut -f2 -d:) export ORACLE_BASE=${ORACLE_HOME%%/product*}

export TNS_ADMIN=$ORACLE_HOME/network/admin

export ADR_HOME=$ORACLE_BASE/diag/rdbms/$(echo $HOLD_SID|tr A-Z a-z)/$HOLD_SID

export PATH=$ORACLE_HOME/bin:/usr/ccs/bin:/opt/SENSsshc/bin/\
:/bin:/usr/bin:.:/var/opt/oracle

export LD_LIBRARY_PATH=/usr/lib:$ORACLE_HOME/lib
```

你可以从命令行或启动文件（例如 `.profile`、`.bash_profile` 或 `.bashrc`）运行 `oraset` 脚本。要从命令行运行 `oraset`，请将 `oraset` 文件放在标准位置（如 `/var/opt/oracle`），并按如下方式运行：

```
$ . /var/opt/oracle/oraset
```

请注意，从命令行运行此命令的语法要求在点号 (`.`) 和命令的其余部分之间有一个空格。当你从命令行运行 `oraset` 时，应该会显示如下菜单：

```
1) O11R2
2) DEV1
```

在此示例中，你现在可以输入 `1` 或 `2` 来为要使用的任何数据库设置所需的 OS 变量。这允许你交互式地设置 OS 变量，而不管服务器上有多少个数据库安装。

你也可以从操作系统启动文件调用 `oraset` 文件。以下是在 `.bashrc` 文件中的示例条目：

```
. /var/opt/oracle/oraset
```

现在，每次登录服务器时，都会显示一个选项菜单，你可以使用它来指示要为其设置 OS 变量的数据库。

## 创建数据库

本节说明如何使用 `SQL*Plus` 的 `CREATE DATABASE` 语句手动创建 Oracle 数据库。创建数据库所需的步骤如下所列：

1.  设置操作系统变量。
2.  配置初始化文件。
3.  创建所需的目录。
4.  创建数据库。
5.  创建数据字典。

以下各小节将介绍每个步骤。

### 步骤 1. 设置操作系统变量

在运行 `SQL*Plus`（或任何其他 Oracle 实用程序）之前，你必须设置几个 OS 变量：

*   `ORACLE_HOME`
*   `PATH`
*   `ORACLE_SID`
*   `LD_LIBRARY_PATH`

`ORACLE_HOME` 变量定义了初始化文件的默认目录位置，在 Linux/Unix 上是 `ORACLE_HOME/dbs`。在 Windows 上，此目录通常是 `ORACLE_HOME\database`。`ORACLE_HOME` 变量也很重要，因为它定义了 Oracle 二进制文件（如 `sqlplus`）的目录位置，这些文件位于 `ORACLE_HOME/bin`。

`PATH` 变量指定了在操作系统提示符下键入命令时默认搜索的目录。在几乎所有情况下，你都需要将 `ORACLE_HOME/bin`（Oracle 二进制文件的位置）包含在你的 `PATH` 变量中。

`ORACLE_SID` 变量定义了你尝试创建的数据库的默认名称。`ORACLE_SID` 也用作初始化文件的默认名称，即 `init<ORACLE_SID>.ora`。

`LD_LIBRARY_PATH` 变量很重要，因为它指定了在 Linux/Unix 系统上搜索库的位置。此变量的值通常设置为包含 `ORACLE_HOME/lib`。

### 步骤 2: 配置初始化文件

Oracle 要求在尝试启动实例之前有一个初始化文件就位。初始化文件用于配置内存和控制文件位置等功能。你可以使用两种类型的初始化文件：

*   服务器参数二进制文件 (`spfile`)
*   `init.ora` 文本文件

Oracle 建议你使用 `spfile`，原因如下：

*   你可以使用 SQL `ALTER SYSTEM` 语句修改 `spfile` 的内容。
*   你可以使用远程客户端 SQL 会话启动数据库，而不需要本地（客户端）初始化文件。

这些都是使用 `spfile` 的好理由。然而，一些公司仍然使用传统的 `init.ora` 文件。`init.ora` 文件也有其优点：

*   你可以直接用操作系统文本编辑器编辑它。
*   你可以在其中包含修改历史的注释。

当我第一次创建数据库时，我发现使用 `init.ora` 文件更容易。如果需要，这个文件以后可以轻松转换为 `spfile`（通过 `CREATE SPFILE FROM PFILE` 语句）。以下是典型的 Oracle 数据库 `11g` `init.ora` 文件的内容：

```
db_name=O11R2
db_block_size=8192
memory_target=800M
memory_max_target=800M
processes=200
control_files=(/ora01/dbfile/O11R2/control01.ctl,/ora02/dbfile/O11R2/control02.ctl) job_queue_processes=10
open_cursors=300
fast_start_mttr_target=500
undo_management=AUTO
undo_tablespace=UNDOTBS1
remote_login_passwordfile=EXCLUSIVE
```

确保初始化文件命名正确并位于适当的目录中。启动实例时，Oracle 会在默认位置查找名为 `spfile<ORACLE_SID>.ora` 的二进制初始化文件。如果没有二进制 `spfile`，Oracle 会查找名为 `init<ORACLE_SID>.ora` 的文本文件。如果 Oracle 在默认位置找不到初始化文件（无论是 `spfile` 还是 `init.ora`），都会抛出错误。你可以通过指定 `STARTUP` 语句的 `PFILE` 子句来明确告诉 Oracle 使用哪个目录和文件，这允许你指定客户端（非服务器）初始化文件的非默认目录和名称。

在 Linux/Unix 系统上，初始化文件（文本 `init.ora` 或二进制 `spfile`）默认位于 `ORACLE_HOME/dbs` 目录中。在 Windows 上，默认目录是 `ORACLE_HOME\database`。

表 2–1 列出了配置 Oracle 初始化文件时的最佳实践。

### 表 2–1. 初始化文件最佳实践

| 最佳实践 | 原因 |
| :--- | :--- |
| Oracle 建议你使用二进制服务器参数文件 (`spfile`)。然而，在某些情况下我仍然使用旧的文本 `init.ora` 文件。 | 使用你习惯的任何类型的初始化参数文件。如果你有使用 `spfile` 的要求，那么一定要实现一个。 |
| 一般来说，如果你不确定初始化参数的预期用途，就不要设置它们。如有疑问，请使用默认值。 | 设置初始化参数可能在数据库性能方面产生深远影响。只有在你知道导致的行为后果时才修改参数。 |
| 对于 `11g`，设置 `memory_target` 和 `memory_max_target` 初始化参数。 | 这样做允许 Oracle 为你管理所有内存组件。 |
| 对于 `10g`，设置 `sga_target` 和 `sga_target_max` 初始化参数。 | 这样做让 Oracle 为你管理大部分内存组件。 |
| 对于 `10g`，设置 `pga_aggregate_target` 和 `workarea_size_policy`。 | 这样做允许 Oracle 管理用于排序空间的内存。 |
| 从 `10g` 开始，使用自动 UNDO 功能。这通过 `undo_management` 和 `undo_tablespace` 参数进行设置。 | 这样做允许 Oracle 管理 UNDO 表空间的大部分功能。 |
| 将 `open_cursors` 设置为比默认值更高的值。我通常将其设置为 `500`。活跃的在线事务处理 (OLTP) 数据库可能需要高得多的值。 | 默认值 50 几乎从来都不够。即使是一个单用户的小型应用程序也可能超过 50 个开放游标的默认值。 |
| 使用模式 `/<mount_point>/dbfile/<database_name>/control0N.ctl` 来命名控制文件。 | 这略微偏离了最优灵活架构 (OFA) 标准。我发现这个位置比位于 `ORACLE_BASE` 下更容易导航。 |
| 至少使用两个控制文件，最好使用位于不同磁盘的不同位置。 | 如果一个控制文件损坏，至少还有另一个可用的控制文件总是个好主意。 |

### 步骤 3: 创建所需的目录

在尝试创建数据库之前，必须在服务器上创建初始化文件或 `CREATE DATABASE` 语句中引用的任何目录。例如，在上一节的初始化文件中，控制文件定义为 `control_files=(/ora01/dbfile/O11R2/control01.ctl,/ora02/dbfile/O11R2/control02.ctl)`。根据前面的内容，确保你已经创建了目录 `/ora01/dbfile/O11R2` 和 `/ora02/dbfile/O11R2`（请根据你的环境进行修改）。在 Linux/Unix 中，你可以使用带有 `-p` 开关的 `mkdir` 命令创建目录以及任何所需的父目录：

```
$ mkdir -p /ora01/dbfile/O11R2
$ mkdir -p /ora02/dbfile/O11R2
```

同时确保你创建了 `CREATE DATABASE` 语句中引用的数据文件和在线重做日志所需的目录（请参阅“步骤 4: 创建数据库”部分）。对于本例，所需的目录如下：

```
$ mkdir -p /ora01/dbfile/O11R2
$ mkdir -p /ora01/dbfile/O11R2
$ mkdir -p /ora01/dbfile/O11R2
$ mkdir -p /ora02/oraredo/O11R2
$ mkdir -p /ora03/oarredo/O11R2
```

如果你以 root 用户身份创建了前面的目录，请确保 `oracle` 用户和 `dba` 组已正确设置为拥有这些目录、子目录和文件。此示例递归地更改以下目录的所有者和组：



## `chown -R oracle:dba /ora01`

## `chown -R oracle:dba /ora02`


