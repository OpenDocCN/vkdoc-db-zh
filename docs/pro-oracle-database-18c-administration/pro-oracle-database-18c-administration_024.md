# 16. 用户管理的备份与恢复

所有 DBA 都应知道如何备份数据库。更为关键的是，DBA 必须能够恢复数据库。当发生介质故障时，所有人都会指望 DBA 来让数据库重新上线运行。Oracle 有两种常见但截然不同的备份与恢复方法：

*   用户管理的方法
*   RMAN 方法

用户管理的备份恰如其名，因为与备份或恢复（或两者）相关的所有步骤都由你手动执行。用户管理的备份有两种类型：冷备份和热备份。冷备份有时也称为离线备份，因为在备份过程中数据库处于关闭状态。热备份也称为在线备份，因为在备份过程中数据库是可用的。

```
SQL> insert into dept values(10);
SQL> insert into dept values(20);
SQL> insert into dept values(30);
SQL> insert into emp values(1,10);
SQL> insert into emp values(2,20);
SQL> insert into emp values(3,10);
SQL> commit;
```

打开两个终端会话。从一个会话中，从子表中删除一条记录（不提交）：

```
SQL> delete from emp where dept_id = 10;
```

然后，尝试从父表中删除一些不受子表删除影响的数据：

```
SQL> delete from dept where dept_id = 30;
```

从父表的删除操作会挂起，直到子表的事务被提交。如果在子表的外键列上没有常规的 B 树索引，那么任何时候尝试在子表中进行插入或删除操作，都会在父表上放置一个表级锁；这会阻止在父表中进行删除或更新，直到子表事务完成。

现在，运行之前的实验，但这次额外为子表的外键列创建一个索引：

```
SQL> create index emp_fk1 on emp(dept_id);
```

你应该能够独立运行之前的两个删除语句。当在外键列上有 B 树索引时，如果从子表中删除数据，Oracle 不会过度锁定父表中的所有行。
```

# RMAN 与用户管理备份

RMAN 是 Oracle 的备份与恢复工具。它简化并管理了备份与恢复的大部分方面。对于 Oracle 备份与恢复，您应该使用 RMAN。RMAN 会自动确定哪些数据文件需要备份、备份位置以及如何恢复与修复。那么，当这种方法已经尘封十多年时，为什么还要开设一章关于用户管理备份的内容呢？以下是理解用户管理备份与恢复的几点原因：

*   仍然有企业在使用用户管理备份与恢复技术。因此，您需要精通这项技术。
*   在备份或恢复失败时，使用这种方法可能有助于故障排查或解决备份问题。
*   巩固您对 Oracle 备份与恢复架构的理解。这在排查任何备份与恢复工具的问题时大有裨益，并为`RMAN`和`Data Guard`等关键 Oracle 工具奠定核心知识基础。
*   您将更充分地领略`RMAN`及其功能价值。
*   老 DBA 们讲述的噩梦般的数据库恢复故事，现在也会变得合理。

基于这些原因，您应该熟悉用户管理备份与恢复技术。手动演练本章中的场景，将极大加深您对哪些文件被备份以及它们在恢复中如何使用的理解。您将能更好地使用`RMAN`。`RMAN`使得大部分备份与恢复工作变得自动化、一键式。然而，了解如何手动备份和恢复数据库，有助于您思考和排查任何备份技术的问题。

本章从冷备份开始。这类备份被视为最简单的用户管理备份形式，因为即使是系统管理员也能实施。接下来，本章将讨论热备份。您还将研究几个常见的恢复与修复场景。这些示例将构建您对 Oracle 备份与恢复内部原理的基础知识。

> **提示**
> 在 Oracle Database 12c 中，您可以对可插拔数据库执行用户管理热备份和冷备份；用户管理备份与恢复技术在此环境下运行良好。与其他 Oracle 数据库一样，强烈建议您在可插拔环境中使用`RMAN`来管理备份与恢复。用户管理备份任务很快会变得难以驾驭，因为 DBA 必须为根容器以及可能众多的可插拔数据库管理这些信息。

## 实施冷备份策略

您通过在数据库关闭后复制文件来执行用户管理冷备份。此类备份也称为离线备份。进行冷备份时，数据库可以处于`noarchivelog`模式或`archivelog`模式。

DBA 倾向于认为冷备份等同于`noarchivelog`模式下数据库的备份。这是不正确的。您可以对`archivelog`模式下的数据库进行冷备份，这在数据中心迁移或需要停机的重大迁移之前可能会用到。冷备份是在数据库关闭时进行的备份，这意味着数据库处于`archivelog`还是`noarchivelog`模式没有区别。它是在某个时间点的备份，如果用于恢复，则只能恢复到该时间点。

### 对数据库进行冷备份

对`noarchivelog`模式下的数据库进行冷备份的一个主要原因，是为您提供一种将数据库恢复到过去某个时间点的方法。仅当您不需要恢复备份之后发生的事务时，才应使用此类备份。无法对`noarchivelog`模式下的数据库执行热备份（即在数据库启动并运行时进行备份），因为更改无法作为备份的一部分被捕获。只有当您的业务要求允许数据丢失和停机时，此类备份和恢复策略才是可接受的。对于生产业务数据库，很少会实施此类备份恢复解决方案，因为生产数据库通常处于`archivelog`模式。

话虽如此，实施此类备份也有一些合理的理由。一个常见用途是对开发/测试/培训数据库进行冷备份，并定期将数据库重置回基线。这为您提供了一种用相同的时点快照重新开始性能测试或培训课程的方法。

> **提示**
> 考虑使用“Flashback Database”功能将数据库设置回过去的某个时间点（更多详情请参见第 19 章）。

本节示例向您展示如何备份数据库中的每个关键文件：所有控制文件、数据文件和在线重做日志文件。使用此类备份，您可以轻松地将数据库恢复到进行备份的时间点。这种方法的主要优点是概念简单且易于实施。以下是对数据库进行冷备份所需的步骤：

1.  确定将备份文件复制到何处以及需要多少空间。
2.  确定要复制的数据库文件的位置和名称。
3.  使用`IMMEDIATE`、`TRANSACTIONAL`或`NORMAL`子句关闭数据库。
4.  将文件（步骤 2 中确定）复制到备份位置（步骤 1 中确定）。
5.  重新启动数据库。

以下各节将详细阐述这些步骤。

#### 步骤 1：确定将备份文件复制到何处以及需要多少空间

理想情况下，备份位置应位于与您实时数据文件位置分开的一组磁盘上。然而，在许多企业中，您可能没有选择权，并且可能会被告知数据库要使用的挂载点。对于此示例，备份位置是目录`/u01/cbackup/o18c`。要粗略了解存储备份副本所需的空间，您可以运行此查询：

```sql
SQL> select sum(sum_bytes)/1024/1024 m_bytes
from(
select sum(bytes) sum_bytes from v$datafile
union
select (sum(bytes) * members) sum_bytes from v$log
group by members);
```

您可以使用 Linux/Unix 的`df`命令来验证操作系统有多少磁盘空间可用。确保操作系统的可用磁盘空间大于上一个查询返回的总和：

```bash
$ df -h
```

> **提示**
> 仅当数据库版本早于 10g 时，才需要备份临时文件。

#### 步骤 2：确定要复制的数据库文件的位置和名称

运行此查询以列出包含在`noarchivelog`模式数据库冷备份中的文件的名称和路径：

```sql
SQL> select name from v$datafile
union
select name from v$controlfile
union
select member from v$logfile;
```


# 是否需要备份在线重做日志？

是否需要备份在线重做日志？不需要；你**永远不需要**在任何类型的备份中备份在线重做日志。那么，为什么数据库管理员会在冷备份中备份在线重做日志呢？一个原因是它使得在`非归档模式`场景下的还原过程稍微容易一些。在线重做日志是正常打开数据库所必需的。

如果你备份了所有文件（包括在线重做日志），那么为了将数据库恢复到备份时的状态，你需要还原所有文件（包括在线重做日志），然后启动数据库。

## 步骤 3. 关闭数据库

以 `SYS`（或具有 `SYSDBA` 权限的用户）身份连接到你的数据库，并使用 `IMMEDIATE`、`TRANSACTIONAL` 或 `NORMAL` 选项关闭数据库。在几乎所有情况下，使用 `IMMEDIATE` 是首选方法。此模式会断开用户连接、回滚未完成的事务并关闭数据库：

```bash
$ sqlplus / as sysdba
SQL> shutdown immediate;
```

## 步骤 4. 创建文件的备份副本

对于步骤 2 中识别的每个文件，使用操作系统实用程序将这些文件复制到备份目录（在步骤 1 中指定）。在这个简单的例子中，所有数据文件、控制文件、临时数据库文件和在线重做日志都在同一个目录中。在生产环境中，你很可能会将文件分散在几个不同的目录中。

此示例使用 Linux/Unix 的 `cp` 命令将数据库文件从 `/u01/dbfile/o18c` 目录复制到 `/u01/cbackup/o18c` 目录：

```bash
$ cp /u01/dbfile/o18c/*.*  /u01/cbackup/o18c
```

## 步骤 5. 重新启动数据库

所有文件复制完成后，你可以启动数据库：

```bash
$ sqlplus / as sysdba
SQL> startup;
```

# 在非归档模式下使用在线重做日志还原冷备份

下一个示例说明如何从`非归档模式`数据库的冷备份中进行还原。如果你在冷备份中包含了在线重做日志，则可以在还原文件时将它们包括在内。此过程涉及以下步骤：

1.  关闭实例。
2.  将数据文件、在线重做日志和控制文件从备份复制回活动数据库的数据文件位置。
3.  启动数据库。

这些步骤将在以下各节中详细说明。

## 步骤 1. 关闭实例

如果实例正在运行，则将其关闭。在此场景中，如何关闭数据库并不重要，因为你正在还原到某个时间点（不进行事务恢复）。活动数据库目录位置中的任何文件在备份文件复制回来时都会被覆盖。如果你的实例正在运行，你可以强制中止它。以具有 `SYSDBA` 权限的用户身份，执行以下操作：

```bash
$ sqlplus / as sysdba
SQL> shutdown abort;
```

## 步骤 2. 从备份复制文件回来

此步骤执行与备份相反的操作：你正在将文件从备份位置复制到活动数据库文件位置。在此示例中，所有备份文件都位于 `/u01/cbackup/o18c` 目录中，并且所有文件都被复制到 `/u01/dbfile/o18c` 目录：

```bash
$ cp /u01/cbackup/o18c/*.*  /u01/dbfile/o18c
```

## 步骤 3. 启动数据库

以 `SYS`（或具有 `SYSDBA` 权限的用户）身份连接到你的数据库，并启动数据库：

```bash
$ sqlplus / as sysdba
SQL> startup;
```

完成这些步骤后，你应该拥有一个与制作冷备份时完全相同的数据库副本。这就像将你的数据库设置回制作备份的时间点一样。

# 在没有在线重做日志的情况下在非归档模式下还原冷备份

如前所述，在从冷备份还原时，你**永远不需要**在线重做日志。如果你对`非归档模式`的数据库进行了冷备份，并且没有将在线重做日志作为备份的一部分包含在内，那么还原步骤与上一节中的步骤几乎完全相同。主要区别在于最后一步需要你使用 `OPEN RESETLOGS` 子句打开数据库。步骤如下：

1.  关闭实例。
2.  从备份复制控制文件和数据文件回来。
3.  在加载模式下启动数据库。
4.  使用 `OPEN RESETLOGS` 子句打开数据库。

## 步骤 1. 关闭实例

如果实例正在运行，则将其关闭。在此场景中，如何关闭数据库并不重要，因为你正在还原到某个时间点。以具有 `SYSDBA` 权限的用户身份，执行以下操作：

```bash
$ sqlplus / as sysdba
SQL> shutdown abort;
```

## 步骤 2. 从备份复制文件回来

将控制文件和数据文件从备份位置复制到活动数据文件位置：

```bash
$ cp /*.*
```

## 步骤 3. 在加载模式下启动数据库

以 `SYS` 或具有 `SYSDBA` 权限的用户身份连接到你的数据库，并在加载模式下启动数据库：

```bash
$ sqlplus / as sysdba
SQL> startup mount
```

## 步骤 4. 使用 OPEN RESETLOGS 子句打开数据库

使用 `OPEN RESETLOGS` 子句打开你的数据库以供使用：

```sql
SQL> alter database open resetlogs;
```

如果你看到 `Database altered` 消息，则表示命令成功。但是，你可能会看到此错误：

```text
ORA-01139: RESETLOGS option only valid after an incomplete database recovery
```

在这种情况下，发出以下命令：

```sql
SQL> recover database until cancel;
```

你应该看到此消息：

```text
Media recovery complete.
```

现在，尝试使用 `OPEN RESETLOGS` 子句打开数据库：

```sql
SQL> alter database open resetlogs;
```

此语句指示 Oracle 重新创建在线重做日志。Oracle 使用控制文件中的信息来确定重做日志的位置、名称和大小。如果这些位置存在旧的在线重做日志文件，它们将被覆盖。

如果你在整个过程中监控 `alert.log`，你可能会看到 `ORA-00312` 和 `ORA-00313`。这意味着 Oracle 找不到在线重做日志文件；这是可以的，因为这些文件直到通过 `OPEN RESETLOGS` 命令重新创建后才在物理上可用。

# 编写冷备份和还原脚本

学习如何编写冷备份脚本具有指导意义。基本思想是动态查询数据字典以确定要备份的文件的位置和名称。这比在脚本中硬编码目录位置和文件名更可取。动态生成脚本更不易出错和出现意外情况（例如，向数据库添加了新的数据文件，但没有更新旧的硬编码备份脚本）。

> **注意**
> 本节中的脚本并非用于生产强度的备份和恢复脚本。相反，它们说明了编写冷备份及后续还原脚本的基本概念。

本节中的第一个脚本对数据库进行冷备份。在使用冷备份脚本之前，你需要修改脚本中的这些变量以匹配你的数据库环境：

*   `ORACLE_SID`
*   `ORACLE_HOME`
*   `cbdir`

`cbdir` 变量指定了备份目录的位置名称。该脚本创建一个名为 `coldback.sql` 的文件，该文件从 SQL*Plus 执行以启动数据库的冷备份：



# Oracle 数据库冷备份与恢复脚本指南

## 冷备份脚本

以下脚本用于生成执行数据库冷备份的 SQL*Plus 命令。

```bash
#!/bin/bash
ORACLE_SID=o18c
ORACLE_HOME=/u01/app/oracle/product/18.1.0.1/db_1
PATH=$PATH:$ORACLE_HOME/bin
#
sqlplus -s <<EOF
/ as sysdba
set head off pages0 lines 132 verify off feed off trimsp on
define cbdir=/u01/cbackup/o18c
spool coldback.sql
select 'shutdown immediate;' from dual;
select '!cp ' || name || ' ' || '&&cbdir'   from v\$datafile;
select '!cp ' || member || ' ' || '&&cbdir' from v\$logfile;
select '!cp ' || name || ' ' || '&&cbdir'   from v\$controlfile;
select 'startup;' from dual;
spool off;
@@coldback.sql
EOF
exit 0
```

此文件生成的命令将从 SQL*Plus 脚本中执行，以对数据库进行冷备份。您在 Unix `cp` 命令前放置一个感叹号（`!`）来指示 SQL*Plus 调出操作系统以运行 `cp` 命令。在 Linux/Unix Shell 脚本中引用 `v$` 数据字典视图时，还需要在每个美元符号（`$`）前放置一个反斜杠（`\`）。`\` 转义了 `$`，并告诉 Shell 脚本不要将 `$` 视为特殊字符（`$` 通常表示 Shell 变量）。

运行此脚本后，写入 `coldback.sql` 脚本的复制命令示例如下：

```
shutdown immediate;
!cp /u01/dbfile/o18c/system01.dbf /u01/cbackup/o18c
!cp /u01/dbfile/o18c/sysaux01.dbf /u01/cbackup/o18c
!cp /u01/dbfile/o18c/undotbs01.dbf /u01/cbackup/o18c
!cp /u01/dbfile/o18c/users01.dbf /u01/cbackup/o18c
!cp /u01/dbfile/o18c/tools01.dbf /u01/cbackup/o18c
!cp /u01/oraredo/o18c/redo02a.rdo /u01/cbackup/o18c
!cp /u02/oraredo/o18c/redo02b.rdo /u01/cbackup/o18c
!cp /u01/oraredo/o18c/redo01a.rdo /u01/cbackup/o18c
!cp /u02/oraredo/o18c/redo01b.rdo /u01/cbackup/o18c
!cp /u01/oraredo/o18c/redo03a.rdo /u01/cbackup/o18c
!cp /u02/oraredo/o18c/redo03b.rdo /u01/cbackup/o18c
!cp /u01/dbfile/o18c/control01.ctl /u01/cbackup/o18c
!cp /u01/dbfile/o18c/control02.ctl /u01/cbackup/o18c
startup;
```

## 恢复脚本

在进行冷备份时，您还应该生成一个脚本，该脚本提供将数据文件、日志文件和控制文件复制回其原始位置的命令。您可以使用此脚本从冷备份中恢复。本节中的下一个脚本动态创建 `coldrest.sql` 脚本，该脚本将文件从备份位置复制到原始数据文件位置。您需要以与修改冷备份脚本相同的方式修改此脚本（即，更改 `ORACLE_SID`、`ORACLE_HOME` 和 `cbdir` 变量以匹配您的环境）：

```bash
#!/bin/bash
ORACLE_SID=o18c
ORACLE_HOME=/u01/app/oracle/product/18.1.0.1/db_1
PATH=$PATH:$ORACLE_HOME/bin
#
sqlplus -s <<EOF
/ as sysdba
set head off pages0 lines 132 verify off feed off trimsp on
define cbdir=/u01/cbackup/o18c
define dbname=$ORACLE_SID
spo coldrest.sql
select 'shutdown abort;' from dual;
select '!cp ' || '&&cbdir/' || substr(name, instr(name,'/',-1,1)+1) ||
' ' || name   from v\$datafile;
select '!cp ' || '&&cbdir/' || substr(member, instr(member,'/',-1,1)+1) ||
' ' || member from v\$logfile;
select '!cp ' || '&&cbdir/' || substr(name, instr(name,'/',-1,1)+1) ||
' ' || name   from v\$controlfile;
select 'startup;' from dual;
spo off;
EOF
exit 0
```

此脚本创建一个名为 `coldrest.sql` 的脚本，该脚本生成复制命令，将您的数据文件、日志文件和控制文件恢复回其原始位置。运行此 Shell 脚本后，`coldrest.sql` 文件中的代码片段如下：

```
shutdown abort;
!cp /u01/cbackup/o18c/system01.dbf /u01/dbfile/o18c/system01.dbf
!cp /u01/cbackup/o18c/sysaux01.dbf /u01/dbfile/o18c/sysaux01.dbf
!cp /u01/cbackup/o18c/undotbs01.dbf /u01/dbfile/o18c/undotbs01.dbf
!cp /u01/cbackup/o18c/users01.dbf /u01/dbfile/o18c/users01.dbf
!cp /u01/cbackup/o18c/tools01.dbf /u01/dbfile/o18c/tools01.dbf
...
!cp /u01/cbackup/o18c/redo03b.rdo /u02/oraredo/o18c/redo03b.rdo
!cp /u01/cbackup/o18c/control01.ctl /u01/dbfile/o18c/control01.ctl
!cp /u01/cbackup/o18c/control02.ctl /u01/dbfile/o18c/control02.ctl
startup;
```



如果您需要使用此脚本从冷备份中恢复，请以 `SYS` 用户身份登录 SQL*Plus，并执行脚本：

1.  对于归档日志模式下数据库的恢复，无论是从冷备份还是热备份恢复，操作都是相同的。这些恢复示例将在本章后续的“执行归档日志模式数据库的完全恢复”和“执行归档日志模式数据库的不完全恢复”部分中介绍。

```
$ sqlplus / as sysdba
SQL> @coldrest.sql
```

## 理解其工作原理确实重要

了解热备份的工作原理也有助于理清并应对困难的 `RMAN` 场景。`RMAN` 是一个复杂的工具，却提供了简化的备份和恢复管理。只需几个命令，您就可以备份、还原和恢复您的数据库。然而，如果任何 `RMAN` 命令或步骤出现故障，理解 Oracle 底层的内部还原与恢复架构将带来巨大好处。详细了解如何从热备份中还原和恢复，能帮助您在遇到任何 `RMAN` 场景时进行逻辑思考并解决问题。

知道 `RMAN` 正从备份中还原文件并能执行恢复，这关联到在没有 `RMAN` 的情况下如何通过复制命令还原数据文件。与 `RMAN` 在还原过程中可能出错的少数情况，应列为测试文档的一部分，以便处理这些问题从而继续恢复；否则，通过命令行记录复制文件和恢复数据文件的命令。

同样，您为理解备份和恢复实现方式所付出的努力，从长远来看是值得的。实际上您需要记忆的内容更少——因为您对底层操作的理解使您能够思考并解决问题，这是检查清单无法做到的。

## 实施热备份策略

如前所述，对于任何类型的 Oracle 数据库备份（联机或脱机），`RMAN` 都应是您的首选工具。`RMAN` 比用户管理的备份更高效，它改进了流程，还提供了一个框架来安排和维护备份与恢复。内部做法是进行热备份，然后使用该备份来还原和恢复您的数据库。手动执行热备份所涉及的命令，接着进行还原和恢复，有助于您理解每种文件类型（控制文件、数据文件、归档重做日志、联机重做日志）在还原与恢复场景中的作用。

以下部分首先向您展示如何实施热备份。它们还提供了可用于自动化热备份过程的基本脚本。后面的章节解释了热备份的一些内部机制，并阐明了为什么必须在执行热备份之前将表空间置于备份模式。

### 进行热备份

进行热备份需要以下步骤：

1.  确保数据库处于归档日志模式。
2.  确定将备份文件复制到何处。
3.  确定需要备份哪些文件。
4.  记录联机重做日志的最大序列号。
5.  将数据库/表空间切换到备份模式。
6.  使用操作系统实用程序将数据文件复制到步骤 2 中确定的位置。
7.  将数据库/表空间切换出备份模式。
8.  归档当前的联机重做日志，并记录联机重做日志的最大序列号。
9.  备份控制文件。
10. 备份备份期间生成的任何归档重做日志。

这些步骤将在以下部分详细说明。

### 步骤 1. 确保数据库处于归档日志模式

运行以下命令以检查数据库的归档日志模式状态：

```
SQL> archive log list;
```

输出显示此数据库处于归档日志模式：

```
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            /u01/oraarch/o18c
```

如果您不确定如何启用归档，请参阅第 5 章了解详情。


## 步骤 2. 确定备份文件的复制位置

现在，确定备份位置。在本例中，备份位置是目录 `/u01/hbackup/o18c`。为了大致了解所需的空间大小，您可以运行此查询：
```
SQL> select sum(bytes) from dba_data_files;
```

理想情况下，备份位置应位于与您的活动数据文件不同的一组磁盘上。但在实践中，很多时候您被分配了存储区域网络（SAN）上的一部分空间，而对底层磁盘布局一无所知。在这些情况下，您需要依赖内置在 SAN 硬件中的冗余（RAID 磁盘、多个控制器等）来确保高可用性和可恢复性。

## 步骤 3. 确定需要备份哪些文件

对于此步骤，您只需要知道数据文件的位置：
```
SQL> select name from v$datafile;
```

当进行到步骤 5 时，您可能需要考虑一次将一个表空间切换到备份模式。如果采用这种方法，您需要知道哪些数据文件与哪个表空间相关联：
```
SQL> select tablespace_name, file_name
from dba_data_files
order by 1,2;
```

## 步骤 4. 记录在线重做日志的最大序列号

要成功使用热备份进行恢复，您至少需要备份期间生成的所有归档重做日志。因此，在开始热备份之前，您需要记录归档日志序列号：
```
SQL> select thread#, max(sequence#)
from v$log
group by thread#
order by thread#;
```

## 步骤 5. 将数据库/表空间切换到备份模式

您可以使用 `ALTER DATABASE BEGIN BACKUP` 语句将所有表空间同时切换到备份模式：
```
SQL> alter database begin backup;
```

如果这是一个活跃的 OLTP 数据库，这样做会极大地降低性能。这是因为当表空间处于备份模式时，Oracle 会将任何数据块（在首次修改时）的完整映像复制到重做流中（有关更多详细信息，请参阅本章后面的“理解拆分块问题”部分）。

另一种方法是每次只将一个表空间切换到备份模式。将表空间切换到备份模式后，您可以复制相关的数据文件（步骤 6），然后将表空间切换出备份模式（步骤 7）。您必须为每个表空间执行此操作：
```
SQL> alter tablespace <tablespace_name> begin backup;
```

## 步骤 6. 使用操作系统实用程序复制数据文件

使用操作系统实用程序（Linux/Unix 的 `cp` 命令）将数据文件复制到备份位置。在此示例中，所有数据文件都在一个目录中，并且全部复制到同一个备份目录：
```
$ cp /u01/dbfile/o18c/*.dbf  /u01/hbackup/o18c
```

## 步骤 7. 将数据库/表空间切换出备份模式

将所有数据文件复制到备份目录后，您需要将表空间切换出备份模式。此示例同时将所有表空间切换出备份模式：
```
SQL> alter database end backup;
```

如果您是每次将一个表空间切换到备份模式，那么在复制完其数据文件后，您需要将每个表空间切换出备份模式：
```
SQL> alter tablespace <tablespace_name> end backup;
```

如果您不将表空间切换出备份模式，可能会严重降低性能并损害数据库恢复的能力。我也见过 RMAN 失败导致表空间或数据文件处于备份模式，并导致后续备份也失败，而此检查将显示是否有任何内容仍处于备份模式。您可以通过以下查询验证没有数据文件处于 `ACTIVE` 状态：
```
SQL> alter session set nls_date_format = 'DD-MON-RRRR HH24:MI:SS';
SQL> select * from v$backup where status='ACTIVE';
```

注意：正确设置 `NLS_DATE_FORMAT` 参数将使您能够看到数据文件进入备份模式的确切日期/时间。在需要恢复数据文件时，这对于确定所需的归档日志的起始序列号很有用。

## 步骤 8. 归档当前的在线重做日志，并记录在线重做日志的最大序列号

以下语句指示 Oracle 归档所有未归档的在线重做日志并启动日志切换。这确保了一个备份结束标记被写入归档重做日志：
```
SQL> alter system archive log current;
```

另外，记录在线重做日志的最大序列号。如果在热备份之后立即发生故障，您需要热备份期间生成的所有归档重做日志来完全恢复您的数据库：
```
SQL> select thread#, max(sequence#)
from v$log
group by thread#
order by thread#;
```

## 步骤 9. 备份控制文件

对于热备份，您不能使用操作系统复制命令来备份控制文件。Oracle 的热备份过程指定您必须使用 `ALTER DATABASE BACKUP CONTROLFILE` 语句。此示例备份控制文件并将其放在与数据库备份文件相同的位置：
```
SQL> alter database backup controlfile
to '/u01/hbackup/o18c/controlbk.ctl' reuse;
```

`REUSE` 子句指示 Oracle 如果备份位置中已存在该文件，则覆盖它。

## 步骤 10. 备份备份期间生成的任何归档重做日志

备份热备份期间生成的归档重做日志。您可以使用操作系统复制命令来完成此操作：
```
$ cp /u01/arch/o18c/*.arc /u01/hbackup/o18c
```

此过程保证您拥有这些日志，即使热备份完成后不久发生故障。请确保不要备份归档器进程当前正在写入的归档重做日志——这样做会导致该文件的副本不完整。有时，DBA 通过检查 `V$ARCHIVED_LOG` 视图中的最大 `SEQUENCE#` 和最大 `RESETLOGS_ID` 来编写此过程的脚本。当 Oracle 完成将归档重做日志复制到磁盘时，会更新该视图。因此，出现在 `V$ARCHIVED_LOG` 视图中的任何归档重做日志文件都应该是安全可复制的。

## 脚本化热备份

本节中的脚本涵盖了热备份相关的最小任务集。对于生产环境，热备份脚本可能相当复杂。这里给出的脚本为您提供了一个基线，说明了热备份脚本中应包含的内容。您需要修改脚本中的这些变量才能使其在您的环境中工作：
*   `ORACLE_SID`
*   `ORACLE_HOME`
*   `hbdir`

`ORACLE_SID` 操作系统变量定义您的数据库名称。`ORACLE_HOME` 操作系统变量定义您安装 Oracle 软件的位置。SQL*Plus 的 `hbdir` 变量指向热备份的目录。
```
#!/bin/bash
ORACLE_SID=o18c
ORACLE_HOME=/u01/app/oracle/product/18.1.0.1/db_1
PATH=$PATH:$ORACLE_HOME/bin
#
sqlplus -s <<EOF
/ as sysdba
set head off pages0 lines 132 verify off feed off trimsp on
define hbdir=/u01/hbackup/o18c
spo hotback.sql
select 'spo &&hbdir/hotlog.txt' from dual;
select 'select max(sequence#) from v\$log;' from dual;
select 'alter database begin backup;' from dual;
select '!cp ' || name || ' ' || '&&hbdir' from v\$datafile;
select 'alter database end backup;' from dual;
select 'alter database backup controlfile to ' || "'" || '&&hbdir'
|| '/controlbk.ctl'  || "'" || ' reuse;' from dual;
select 'alter system archive log current;' from dual;
select 'select max(sequence#) from v\$log;' from dual;
select 'select member from v\$logfile;' from dual;
select 'spo off;' from dual;
spo off;
@@hotback.sql
EOF
```

该脚本生成一个 `hotback.sql` 脚本。此脚本包含执行热备份的命令。以下是一个测试数据库的 `hotback.sql` 脚本清单：


# Oracle 热备份操作与原理

首先，执行以下 SQL*Plus 脚本进行热备份：
```
spo /u01/hbackup/o18c/hotlog.txt
select max(sequence#) from v$log;
alter database begin backup;
!cp /u01/dbfile/o18c/system01.dbf /u01/hbackup/o18c
!cp /u01/dbfile/o18c/sysaux01.dbf /u01/hbackup/o18c
!cp /u01/dbfile/o18c/undotbs01.dbf /u01/hbackup/o18c
!cp /u01/dbfile/o18c/users01.dbf /u01/hbackup/o18c
!cp /u01/dbfile/o18c/tools01.dbf /u01/hbackup/o18c
alter database end backup;
alter database backup controlfile to '/u01/hbackup/o18c/controlbk.ctl' reuse;
alter system archive log current;
select max(sequence#) from v$log;
select member from v$logfile;
spo off;
```
你可以从 SQL*Plus 手动运行此脚本：
```
SQL> @hotback.sql
```

## Caution（注意事项）
如果之前的脚本在执行 `ALTER DATABASE END BACKUP` 之前的某条语句上失败，你必须手动运行 `ALTER DATABASE END BACKUP`（以 `SYS` 用户身份从 SQL*Plus 执行）来使你的数据库（表空间）退出备份模式。

在生成热备份脚本时，同时生成一个用于从备份目录复制数据文件的脚本是审慎的做法。你必须修改此脚本中的 `hbdir` 变量，以匹配你环境中热备份的位置。以下是一个生成复制命令的脚本：
```
#!/bin/bash
ORACLE_SID=o18c
ORACLE_HOME=/u01/app/oracle/product/18.1.0.1/db_1
PATH=$PATH:$ORACLE_HOME/bin
#
sqlplus -s <<EOF
/ as sysdba
set head off pages0 lines 132 verify off feed off trimsp on
define hbdir=/u01/hbackup/o18c/
define dbname=$ORACLE_SID
spo hotrest.sql
select '!cp ' || '&&hbdir' || substr(name,instr(name,'/',-1,1)+1)
|| ' ' || name from v\$datafile;
spo off;
EOF
#
exit 0
```
在我的环境中，生成了以下代码，可在发生故障时从 SQL*Plus 执行，以将数据文件从备份目录复制回来：
```
!cp /u01/hbackup/o18c/system01.dbf /u01/dbfile/o18c/system01.dbf
!cp /u01/hbackup/o18c/sysaux01.dbf /u01/dbfile/o18c/sysaux01.dbf
!cp /u01/hbackup/o18c/undotbs01.dbf /u01/dbfile/o18c/undotbs01.dbf
!cp /u01/hbackup/o18c/users01.dbf /u01/dbfile/o18c/users01.dbf
!cp /u01/hbackup/o18c/tools01.dbf /u01/dbfile/o18c/tools01.dbf
```
在此输出中，如果你更喜欢从操作系统提示符运行这些命令，可以删除每行开头的感叹号（`!`）。核心思想是，这些命令在发生故障时可用，这样你就知道哪些文件已备份到哪个位置以及如何将它们复制回来。

## Tip（提示）
不要对在线备份使用用户管理的热备份技术；请使用 RMAN。RMAN 不需要将表空间置于备份模式，并且自动化了几乎所有与备份和恢复相关的操作。

## Understanding the Split-Block Issue（理解裂块问题）
要执行热备份，一个关键步骤是在使用操作系统实用程序复制与表空间关联的任何数据文件之前，将表空间更改为备份模式。要理解为何必须将表空间更改为备份模式，你必须熟悉有时被称为裂块（split- 或 fractured- block）的问题。

回想一下，数据库块的大小通常与操作系统块的大小不同。例如，一个数据库块的大小可能为 8KB，而操作系统块的大小为 4KB。作为热备份的一部分，你使用操作系统实用程序复制活动的数据文件。当操作系统实用程序正在复制数据文件时，存在数据库写入器同时写入某个块的可能性。因为 Oracle 块和操作系统块的大小不同，可能会发生以下情况：
1. 操作系统实用程序复制了 Oracle 块的一部分。
2. 片刻之后，数据库写入器更新了整个块。
3. 转瞬之间，操作系统实用程序复制了 Oracle 块的后半部分。
这可能导致操作系统复制的块与 Oracle 写入操作系统的内容不一致。图 `16-1` 说明了这个概念。

![../images/214899_3_En_16_Chapter/214899_3_En_16_Fig1_HTML.png](img/214899_3_En_16_Fig1_HTML.png)

**图 16-1** 热备份裂块（或碎块）问题

查看图 `16-1`，在时间点 3 复制到磁盘的块，就 Oracle 而言，是损坏的。块的前半部分来自时间点 1，后半部分在时间点 3 复制。当你进行热备份时，你是在保证数据文件备份中存在块级别的损坏。

要理解 Oracle 如何解决裂块问题，首先考虑在正常模式（非备份模式）下运行的数据库。写入联机重做日志的重做信息仅仅是 Oracle 重新应用事务所需的内容。重做流不包含完整的数据块。Oracle 在重做流中只记录一个更改向量，该向量指定了哪个块被更改以及如何被更改。图 `16-2` 显示了在正常条件下运行的 Oracle。

![../images/214899_3_En_16_Chapter/214899_3_En_16_Fig2_HTML.png](img/214899_3_En_16_Fig2_HTML.png)

**图 16-2** Oracle 通常只将更改向量写入重做流

现在，考虑在热备份期间会发生什么。对于热备份，在复制与表空间关联的数据文件之前，必须先将表空间更改为备份模式。在此模式下，在 Oracle 修改一个块之前，整个块会被复制到重做流中。对该块的任何后续更改只需要将正常的重做更改向量写入重做流。如图 `16-3` 所示。

![../images/214899_3_En_16_Chapter/214899_3_En_16_Fig3_HTML.png](img/214899_3_En_16_Fig3_HTML.png)

**图 16-3** 完整的块被写入重做流

要理解为什么 Oracle 将整个块记录到重做流中，考虑在还原和恢复期间会发生什么。首先，还原来自热备份的备份文件。如前所述，这些备份文件包含损坏的块，这是由裂块问题导致的。但这无关紧要，因为一旦 Oracle 恢复了数据文件，对于在热备份期间被修改过的任何块，Oracle 都拥有该块在修改前的映像副本。Oracle 使用它在重做流中的块副本作为恢复（该块）的起点。此过程如图 `16-4` 所示。

![../images/214899_3_En_16_Chapter/214899_3_En_16_Fig4_HTML.png](img/214899_3_En_16_Fig4_HTML.png)

**图 16-4** 裂块的还原和恢复

这样，热备份文件中是否存在损坏的块并不重要。Oracle 总是从重做流中的一个块副本（修改前的状态）开始该块的恢复过程。

## Understanding the Need for Redo Generated During Backup（理解备份期间生成重做的必要性）
如果你在创建热备份后不久就遇到故障，会发生什么？Oracle 知道表空间何时被置于备份模式（begin backup 系统 SCN 被写入重做流），也知道表空间何时退出备份模式（end-of-backup 标记被写入重做流）。Oracle 需要在此时间范围内生成的所有归档重做日志来成功恢复数据文件。

图 `16-5` 显示，至少需要从序列号 100 到 102 的归档重做日志来恢复表空间。这些归档重做日志是在热备份期间生成的。

![../images/214899_3_En_16_Chapter/214899_3_En_16_Fig5_HTML.png](img/214899_3_En_16_Fig5_HTML.png)

**图 16-5** 恢复已应用

如果你尝试在 begin 和 end 标记之间的所有重做都应用到数据文件之前停止恢复过程，Oracle 会抛出此错误：
```
ORA-01195: online backup of file 1 needs more recovery to be consistent
```



# Oracle 表空间热备份的恢复处理与数据文件写入行为澄清

在对表空间进行热备份期间产生的所有重做日志必须在数据文件可以打开之前应用到这些文件上。Oracle 至少需要应用从备份开始 SCN 标记到备份结束标记之间的一切内容，以覆盖表空间处于备份模式时修改的每个数据块。这些重做记录存在于归档重做日志文件中；或者，如果故障恰好发生在备份结束后，部分重做日志可能尚未归档，而是保存在在线重做日志中。因此，你必须指示 Oracle 应用在线重做日志中的内容。

## 理解数据文件的更新机制

请注意，在图 16-2 和 16-3 中，数据库写入器的行为在整个备份过程中基本保持不变。无论数据库是否处于备份模式，数据库写入器都会持续将数据块写入数据文件。数据库写入器并不关心是否正在进行热备份；它的工作就是将缓冲区缓存中的块写入数据文件。

偶尔你会遇到这样的 DBA：他声称在用户管理的热备份期间，数据库写入器不会写入数据文件。这是一种普遍的误解。请运用常识思考：如果数据库写入器在热备份期间不写入数据文件，那么更改被写入到哪里了？如果事务被写入到数据文件以外的地方，那么在备份之后这些数据文件如何重新同步？这完全说不通。

一些 DBA 说：“数据文件头被冻结了，这意味着数据文件不会有更改。”Oracle 确实会冻结数据文件头中的 SCN 以标记热备份的开始，并且直到将该表空间退出备份模式才会更新该 SCN。这种“冻结的 SCN”并不意味着在备份期间没有数据块被写入数据文件。你可以通过以下步骤轻松证明在备份模式下数据文件会被写入：

1.  将一个表空间至于备份模式：
    ```
    SQL> alter tablespace users begin backup;
    ```

2.  创建一个包含字符字段的表：
    ```
    SQL> create table cc(cc varchar2(20)) tablespace users;
    ```

3.  向该表中插入一个字符串：
    ```
    SQL> insert into cc values('DBWR does write');
    ```

4.  强制执行一个检查点（这确保所有被修改的缓冲区都被写入磁盘）：
    ```
    SQL> alter system checkpoint;
    ```

5.  在操作系统层面，使用`strings`和`grep`命令在数据文件中搜索该字符串：
    ```
    $ strings /u01/dbfile/o18c/users01.dbf | grep "DBWR does write"
    ```

6.  以下是输出结果，证明数据库写入器确实将数据写入了磁盘：
    ```
    DBWR does write
    ```

7.  别忘了将表空间退出备份模式：
    ```
    SQL> alter tablespace users end backup;
    ```

## 执行归档日志模式数据库的完全恢复

术语*完全恢复*意味着你可以恢复故障发生前所有已提交的事务。*完全恢复*并不意味着你必须完全恢复和还原整个数据库。例如，如果只有一个数据文件发生了介质故障，你只需要还原并恢复该受损的数据文件即可完成完全恢复。

> **提示**：如果你可以访问一个测试或开发数据库，请花时间逐步执行后续示例中的每一个步骤。实践这些步骤可以比任何文档教会你更多关于备份和恢复的知识。

此处概述的步骤适用于任何在归档日志模式下备份的数据库。无论你进行的是冷备份还是热备份，只要备份期间数据库处于归档日志模式，还原和恢复数据文件的步骤都是相同的。进行完全恢复，你需要以下条件：

*   能够还原发生介质故障的数据文件
*   访问自上次备份开始以来产生的所有归档重做日志



# 完整恢复的基本步骤如下：

1.  将数据库置于装载模式；这可以防止正常的用户事务处理读取/写入正在恢复的数据文件。（如果您恢复的不是 `SYSTEM` 或 `UNDO` 表空间，您也可以选择打开数据库并在恢复前手动将数据文件脱机。如果您这样做，请确保在恢复完成后将数据文件重新联机。）

2.  使用操作系统复制工具恢复损坏的数据文件。

3.  执行相应的 SQL\*Plus `RECOVER` 命令，以应用归档重做日志和在线重做日志中所需的任何信息。

4.  将数据库改为打开状态。

接下来的几个部分将演示一些常见的完整恢复与恢复场景。您应该能够应用这些基本场景来诊断和恢复您所遇到的任何复杂情况。

## 在数据库离线时进行恢复与恢复

本节详细描述了一个简单的恢复与恢复场景。接下来描述的是模拟故障然后执行完整恢复与恢复的步骤。请在开发数据库中尝试此场景。确保您有一个良好的备份，并且不要在包含关键业务数据的数据库中尝试此实验。

在开始此示例之前，请创建一个表并插入一些数据。在完整恢复过程结束时将选择此表和数据，以演示成功的恢复：

```sql
SQL> create table foo(foo number) tablespace users;
SQL> insert into foo values(1);
SQL> commit;
```

现在，切换几次在线日志。这样做可确保您必须在恢复过程中应用归档重做日志：

```sql
SQL> alter system switch logfile;
```

正斜杠 (`/`) 会重新运行最近执行的 SQL 语句：

```sql
SQL> /
SQL> /
SQL> /
```

接下来，通过重命名与 `USERS` 表空间关联的数据文件来模拟介质故障。您可以使用此查询来确定该文件的名称：

```sql
SQL> select file_name from dba_data_files where tablespace_name='USERS';
FILE_NAME
--------------------------------------------------------------------------------
/u01/dbfile/o18c/users01.dbf
```

在操作系统中，重命名该文件：

```bash
$ mv /u01/dbfile/o18c/users01.dbf /u01/dbfile/o18c/users01.dbf.old
```

然后，尝试停止您的数据库：

```bash
$ sqlplus / as sysdba
SQL> shutdown immediate;
```

您应该会看到类似这样的错误：

```
ORA-01116: error in opening database file ...
```

如果这是一场真实的灾难，谨慎的做法是导航到数据文件目录，列出文件，并检查有问题的文件是否在其正确的位置。您还应该检查 `alert.log` 文件，看看 Oracle 是否在那里记录了任何相关信息。

既然您已经模拟了介质故障，接下来的几个步骤将引导您完成恢复和完整恢复。

### 步骤 1. 将数据库置于装载模式

在将数据库置于装载模式之前，您可能需要先使用 `ABORT` 参数关闭它：

```bash
$ sqlplus / as sysdba
SQL> shutdown abort;
SQL> startup mount;
```

### 步骤 2. 从备份中恢复数据文件

下一步是从备份中复制与发生故障的文件相对应的数据文件：

```bash
$ cp /u01/hbackup/o18c/users01.dbf /u01/dbfile/o18c/users01.dbf
```

此时，思考一下如果您尝试启动数据库，Oracle 会做什么是有启发意义的。当您发出 `ALTER DATABASE OPEN` 语句时，Oracle 会检查控制文件中每个数据文件的 SCN。您可以通过查询 `V$DATAFILE` 来检查此 SCN：

```sql
SQL> select checkpoint_change# from v$datafile where file#=4;
CHECKPOINT_CHANGE#
------------------
            3502290
```

Oracle 会将控制文件中的 SCN 与数据文件头中的 SCN 进行比较。您可以通过查询 `V$DATAFILE_HEADER` 来检查数据文件头中的 SCN；例如，

```sql
SQL> select file#, fuzzy, checkpoint_change#
  2  from v$datafile_header
  3  where file#=4;
FILE# FUZ CHECKPOINT_CHANGE#
----- --- ------------------
    4 YES            3502285
```

请注意，`V$DATAFILE_HEADER` 中记录的 SCN 小于同一数据文件在 `V$DATAFILE` 中的 SCN。如果您尝试打开数据库，Oracle 会抛出错误，指出需要介质恢复（意味着您需要应用重做）以使数据文件中的 SCN 与控制文件中的 SCN 同步。`FUZZY` 列被设置为 `YES`。这表示必须先对数据文件应用重做，然后才能打开供使用。以下是此时尝试打开数据库时发生的情况：

```sql
SQL> alter database open;
alter database open
*
ERROR at line 1:
ORA-01113: file 4 needs media recovery...
```

Oracle 不允许您打开数据库，直到所有数据文件头中的 SCN 与控制文件中相应的 SCN 匹配。

### 步骤 3. 执行相应的 RECOVER 语句

归档重做日志和在线重做日志包含将数据文件 SCN 追赶上控制文件 SCN 所需的信息。您可以通过执行以下 SQL\*Plus 语句之一，对需要介质恢复的数据文件应用重做：

*   `RECOVER DATAFILE`
*   `RECOVER TABLESPACE`
*   `RECOVER DATABASE`

因为在此示例中只有一个数据文件需要恢复，所以 `RECOVER DATAFILE` 语句是合适的。但是，请记住，您可以运行上述任何 `RECOVER` 语句，Oracle 会弄清楚需要恢复什么。在此特定场景中，记住包含已恢复数据文件的表空间名称可能比记住数据文件名称更容易。接下来，恢复 `USERS` 表空间中所有需要恢复的数据文件：

```sql
SQL> recover tablespace users;
```

此时，Oracle 使用数据文件头中的 SCN 来确定使用哪个归档重做日志或在线重做日志开始应用重做。您可以通过以下查询查看 RMAN 将用于开始恢复过程的起始日志序列号：

```sql
SQL> select
  2  HXFNM file_name
  3  ,HXFIL file_num
  4  ,FHTNM tablespace_name
  5  ,FHTHR thread
  6  ,FHRBA_SEQ sequence
  7  from X$KCVFH
  8  where FHTNM = 'USERS';
```

如果所有所需的重做都在在线重做日志中，Oracle 会应用该重做并显示此消息：

```
Media recovery complete.
```

如果 Oracle 需要应用仅包含在归档重做日志中的重做（意味着包含相应重做的在线重做日志已被覆盖），系统会提示您 Oracle 的建议，即首先应用哪个归档重做日志：

```
ORA-00279: change 3502285 generated at 11/02/2018 10:49:39 needed for thread 1
ORA-00289: suggestion : /u01/oraarch/o18c/1_1_798283209.dbf
ORA-00280: change 3502285 for thread 1 is in sequence #1
Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
```

您可以按 Enter 或回车键 (`<RET>`) 让 Oracle 应用建议的归档重做日志文件，指定文件名，指定 `AUTO` 指示 Oracle 自动应用所有建议的文件，或键入 `CANCEL` 以取消恢复操作。

在此示例中，指定 `AUTO`。Oracle 将应用所有归档重做日志文件和在线重做日志文件中的所有重做，以执行完整恢复：

```
AUTO
```

应用完所有必需的归档重做和在线重做后，最后显示的消息是：

```
Log applied.
Media recovery complete.
```

### 步骤 4. 将数据库改为打开状态

介质恢复完成后，您可以打开数据库：

```sql
SQL> alter database open;
```

您现在可以验证在介质故障之前提交的事务是否已恢复和恢复：

```sql
SQL> select * from foo;
       FOO
----------
         1
```

## 在数据库在线时进行恢复与恢复



# 数据库在线恢复与控制文件还原指南

## 在线恢复数据文件

如果您丢失了除 `SYSTEM` 和 `UNDO` 之外的表空间关联的数据文件，您可以在数据库保持在线的情况下恢复并修复受损的数据文件。为此，任何需要被恢复的数据文件必须先被置于离线状态。您可能会收到关于数据文件问题的警报，当用户尝试更新表时看到类似以下错误：

```
SQL> insert into foo values(2);
ORA-01116: error in opening database file ...
```

您导航到包含该数据文件的操作系统目录，发现它已被系统管理员错误地删除。

在本例中，与 `USERS` 表空间关联的数据文件被置于离线状态，随后在数据库其余部分保持在线的同时进行了恢复和修复。首先，将数据文件离线：

```
SQL> alter database datafile '/u01/dbfile/o18c/users01.dbf' offline;
```

现在，从备份位置恢复相应的数据文件：

```
$ cp /u01/hbackup/o18c/users01.dbf /u01/dbfile/o18c/users01.dbf
```

在这种情况下，您不能使用 `RECOVER DATABASE`。`RECOVER DATABASE` 语句尝试恢复数据库中的所有数据文件，而 `SYSTEM` 表空间是其中的一部分。`SYSTEM` 表空间在数据库在线时无法恢复。如果您使用 `RECOVER TABLESPACE`，则该表空间关联的所有数据文件都必须处于离线状态。在这种情况下，更合适的做法是在数据文件粒度级别进行恢复：

```
SQL> recover datafile '/u01/dbfile/o18c/users01.dbf';
```

Oracle 会检查数据文件头中的 `SCN`，并确定使用哪个归档重做日志或在线重做日志来开始应用重做。如果所有需要的重做都在在线重做日志中，您会看到此消息：

```
Media recovery complete.
```

如果重做的起点只包含在归档重做日志文件中，Oracle 会建议从哪个文件开始：

```
ORA-00279: change 3502285 generated at 11/02/2018 10:49:39 needed for thread 1
ORA-00289: suggestion : /u01/oraarch/o18c/1_1_798283209.dbf
ORA-00280: change 3502285 for thread 1 is in sequence #1
Specify log: {=suggested | filename | AUTO | CANCEL}
```

您可以键入 `AUTO` 让 Oracle 自动应用所有归档重做日志文件和在线重做日志文件中必需的重做：

```
AUTO
```

如果成功，您应该会看到此消息：

```
Log applied.
Media recovery complete.
```

现在您可以将数据文件重新联机：

```
SQL> alter database datafile '/u01/dbfile/o18c/users01.dbf' online;
```

如果成功，您应该看到：

```
Database altered.
```

## 还原控制文件

在处理用户管理的备份时，您通常在以下情况下还原控制文件：

*   一个控制文件受损，并且该文件是多路复用的。
*   所有控制文件都受损。

这两种情况将在以下部分中介绍。

### 当多路复用时还原受损的控制文件

如果您的数据库配置了多个控制文件，您可以关闭数据库并使用操作系统命令将现有的控制文件复制到丢失控制文件的位置。例如，从初始化文件中，您知道此数据库使用了两个控制文件：

```
SQL> show parameter control_files
NAME                         TYPE        VALUE
---------------------------- ----------- ------------------------------
control_files                string      /u01/dbfile/o18c/control01.ctl
,/u02/dbfile/o18c/control02.ctl
```

假设 `control02.ctl` 文件已受损。查询数据字典时 Oracle 会抛出此错误：

```
ORA-00210: cannot open the specified control file...
```

当有完好的控制文件可用时，您可以关闭数据库，移动旧的/受损的控制文件（这可以保留它，以备以后进行根本原因分析时需要），然后将现有的完好控制文件复制到受损控制文件的名称和位置：

```
SQL> shutdown abort;
$ mv /u02/dbfile/o18c/control02.ctl /u02/dbfile/o18c/control02.ctl.old
$ cp /u01/dbfile/o18c/control01.ctl /u02/dbfile/o18c/control02.ctl
```

现在，重新启动数据库：

```
SQL> startup;
```

这样，您就可以从现有的控制文件还原控制文件。

### 当所有控制文件都受损时进行还原

如果您丢失了所有控制文件，您可以从备份中还原一个，或者重新创建控制文件。只要您拥有所有数据文件和任何必需的重做（归档重做和在线重做），就应该能够完全恢复数据库。此场景的步骤如下：

1.  关闭数据库。
2.  从备份中还原控制文件。
3.  以 `MOUNT` 模式启动数据库，并使用 `RECOVER DATABASE USING BACKUP CONTROLFILE` 子句启动数据库恢复。
4.  对于完全恢复，手动应用在线重做日志中包含的重做。
5.  使用 `OPEN RESETLOGS` 子句打开数据库。

在本例中，数据库的所有控制文件都被意外删除，随后 Oracle 报告此错误：

```
ORA-00210: cannot open the specified control file...
```

#### 步骤 1：关闭数据库

首先，关闭数据库：

```
SQL> shutdown abort;
```

#### 步骤 2：从备份还原控制文件

此数据库配置为仅使用一个控制文件，您可以将其从备份位置复制回来，如下所示：

```
$ cp /u01/hbackup/o18c/controlbk.ctl /u01/dbfile/o18c/control01.ctl
```

如果使用多个控制文件，则必须将备份控制文件复制到 `CONTROL_FILES` 初始化参数中列出的每个控制文件及其位置名称。

#### 步骤 3：以 MOUNT 模式启动数据库并启动数据库恢复

接下来，以 `MOUNT` 模式启动数据库：

```
SQL> startup mount;
```

在控制文件和数据文件被复制回来后，您可以执行恢复。Oracle 知道控制文件来自备份（因为它是使用 `ALTER DATABASE BACKUP CONTROLFILE` 语句创建的），因此必须使用 `USING BACKUP CONTROLFILE` 子句执行恢复：

```
SQL> recover database using backup controlfile;
```

此时，系统会提示您应用归档重做日志文件：

```
ORA-00279: change 3584431 generated at 11/02/2018 11:48:46 needed for thread 1
ORA-00289: suggestion : /u01/oraarch/o18c/1_8_798283209.dbf
ORA-00280: change 3584431 for thread 1 is in sequence #8
Specify log: {=suggested | filename | AUTO | CANCEL}
```

键入 `AUTO` 以指示恢复过程自动应用所有归档重做日志：

```
AUTO
```

恢复过程应用所有可用的归档重做日志。恢复过程无法确定归档重做流结束的位置，因此会尝试应用一个不存在的归档重做日志，导致出现如下消息：

```
ORA-00308: cannot open archived log '/u01/oraarch/o18c/1_10_798283209.dbf'
ORA-27037: unable to obtain file status
```

之前的消息是预期的。现在，尝试打开数据库：

```
SQL> alter database open resetlogs;
```

Oracle 在此情况下会抛出以下错误：

```
ORA-01113: file 1 needs media recovery
ORA-01110: data file 1: '/u01/dbfile/o18c/system01.dbf'
```

#### 步骤 4：应用在线重做日志中包含的重做

Oracle 需要应用更多重做来同步控制文件中的 `SCN` 与数据文件头中的 `SCN`。在此场景中，在线重做日志仍然完好无损，并包含所需的重做。要应用在线重做日志中包含的重做，首先确定在线重做日志文件的位置和名称：

```
SQL> select a.sequence#, a.status, a.first_change#, b.member
from v$log a, v$logfile b
where a.group# = b.group#
order by a.sequence#;
```

以下是此示例的部分输出：


# Oracle 数据库恢复操作指南

## 使用备份控制文件进行恢复

以下是重做日志文件的状态信息：

```
SEQUENCE#  STATUS          FIRST_CHANGE#   MEMBER
---------- --------------- --------------- ----------------------
6  INACTIVE        3543960         /u01/oraredo/o12c/redo03a.rdo
6  INACTIVE        3543960         /u02/oraredo/o12c/redo03b.rdo
7  INACTIVE        3543963         /u02/oraredo/o12c/redo01b.rdo
7  INACTIVE        3543963         /u01/oraredo/o12c/redo01a.rdo
8  CURRENT         3583986         /u02/oraredo/o12c/redo02b.rdo
8  CURRENT         3583986         /u01/oraredo/o12c/redo02a.rdo
```

现在，重新启动恢复过程：

```
SQL> recover database using backup controlfile;
```

恢复过程提示需要一个不存在的归档重做日志：

```
ORA-00279: change 3584513 generated at 11/02/2018 11:50:50 needed for thread 1
ORA-00289: suggestion : /u01/oraarch/o18c/1_10_798283209.dbf
ORA-00280: change 3584513 for thread 1 is in sequence #10
Specify log: {=suggested | filename | AUTO | CANCEL}
```

不要向恢复过程提供归档重做日志文件，而是输入一个当前联机重做日志文件的名称（可能需要尝试每个联机重做日志，直到找到 Oracle 需要的那个）。这会指示恢复过程应用联机重做日志中的所有重做信息：

```
/u01/oraredo/o18c/redo01a.rdo
```

当应用了正确的联机重做日志时，您应该看到以下消息：

```
Log applied.
Media recovery complete.
```

## 步骤 5：使用 `RESETLOGS` 打开数据库

此时数据库已完全恢复。在这种类型的恢复之后，对数据库执行新的备份非常重要。然而，由于恢复过程使用了备份控制文件，因此必须使用 `RESETLOGS` 子句打开数据库：

```
SQL> alter database open resetlogs;
```

成功后，您将看到：

```
Database altered.
```

## 执行归档日志模式数据库的不完全恢复

不完全恢复意味着您没有恢复故障发生前已提交的所有事务。使用这种恢复类型，您是在恢复到过去的一个时间点，事务会丢失。这就是为什么不完全恢复也称为数据库时间点恢复（DBPITR）。

不完全恢复并不意味着您只恢复和还原一部分数据文件。事实上，在大多数不完全恢复场景中，您必须作为过程的一部分从备份中还原所有数据文件。如果您不想恢复所有数据文件，首先需要将您不打算参与不完全恢复过程的数据文件脱机。当您启动恢复时，Oracle 只会恢复在 `V$DATAFILE_HEADER` 的 `STATUS` 列中具有 `ONLINE` 值的数据文件。

您可能出于多种不同原因想要执行不完全恢复：

*   您尝试执行完全恢复，但缺少所需的归档重做日志或未归档的联机重做日志信息。
*   您希望将数据库恢复到过去某个错误操作（删除数据、删除表等）发生之前的时刻。
*   您有一个测试环境。（闪回数据库可能是一个选择。）

您可以通过三种方式执行用户管理的不完全恢复：

*   基于取消的恢复
*   基于 SCN 的恢复
*   基于时间的恢复

基于取消的恢复允许您应用归档重做，并在归档重做日志文件边界处暂停过程。例如，假设您正在尝试还原和恢复数据库，并且发现缺少一个归档重做日志。您必须在最后一个完好的归档重做日志点停止恢复过程。您可以使用 `RECOVER DATABASE` 语句的 `CANCEL` 子句启动基于取消的不完全恢复：

```
SQL> recover database until cancel;
```

如果您想恢复到并包括某个 SCN 号，请使用基于 SCN 的不完全恢复。您可能从警报日志或 LogMiner 的输出中得知想要恢复到某个特定 SCN。使用 `UNTIL CHANGE` 子句执行此类不完全恢复：

```
SQL> recover database until change 12345;
```

如果您知道希望停止恢复过程的时间，请使用基于时间的不完全恢复。例如，您可能知道某个表是在特定时间被删除的，并希望将数据库恢复并恢复到指定时间。基于时间的恢复格式始终如下：`YYYY-MM-DD:HH24:MI:SS`。这是一个例子：

```
SQL> recover database until time '2018-10-21:02:00:00';
```

当您执行不完全恢复时，必须还原所有您计划在不完全恢复完成后保持在线的数据文件。以下是执行不完全恢复的步骤：

1.  关闭数据库。
2.  从备份中还原所有数据文件。
3.  以 `MOUNT` 模式启动数据库。
4.  应用重做（前滚）到所需点，然后暂停恢复过程（使用基于取消、SCN 或时间的恢复）。
5.  使用 `OPEN RESETLOGS` 子句打开数据库。

以下示例执行基于取消的不完全恢复。如果数据库是打开的，请将其关闭：

```
$ sqlplus / as sysdba
SQL> shutdown abort;
```

接下来，从备份中复制所有数据文件（冷备份或热备份均可）。此示例从热备份还原所有数据文件。在此示例中，当前控制文件是完好的，无需还原。以下是用于还原数据库的操作系统复制命令片段：

```
cp /u01/hbackup/o18c/system01.dbf /u01/dbfile/o18c/system01.dbf
cp /u01/hbackup/o18c/sysaux01.dbf /u01/dbfile/o18c/sysaux01.dbf
cp /u01/hbackup/o18c/undotbs01.dbf /u01/dbfile/o18c/undotbs01.dbf
cp /u01/hbackup/o18c/users01.dbf /u01/dbfile/o18c/users01.dbf
cp /u01/hbackup/o18c/tools01.dbf /u01/dbfile/o18c/tools01.dbf
```

数据文件复制回去后，就可以启动恢复过程。此示例执行基于取消的不完全恢复：

```
$ sqlplus / as sysdba
SQL> startup mount;
SQL> recover database until cancel;
```

此时，Oracle 恢复过程建议应用一个归档重做日志：

```
ORA-00279: change 3584872 generated at 11/02/2018 12:02:32 needed for thread 1
ORA-00289: suggestion : /u01/oraarch/o18c/1_1_798292887.dbf
ORA-00280: change 3584872 for thread 1 is in sequence #1
Specify log: {=suggested | filename | AUTO | CANCEL}
```

应用日志到您希望停止的点，然后输入 `CANCEL`：

```
CANCEL
```

这将停止恢复过程。现在，您可以使用 `RESETLOGS` 子句打开数据库：

```
SQL> alter database open resetlogs;
```

数据库已打开到过去的一个时间点。由于并非所有重做都已应用，因此恢复被视为不完全恢复。

**提示**：现在是对数据库进行良好备份的好时机。如果您在打开数据库后很快发生故障，这将为您提供一个干净的点，以便启动还原和恢复。

# OPEN RESETLOGS 命令的用途

有时，您需要使用 `OPEN RESETLOGS` 子句打开数据库。在以下情况中可能会执行此操作：重新创建控制文件、使用备份控制文件执行还原与恢复，或执行不完全恢复。当您使用 `OPEN RESETLOGS` 子句打开数据库时，该命令会清除所有现有的联机重做日志文件；如果这些文件不存在，则会重新创建它们。您可以通过查询 `V$LOGFILE` 的 `MEMBER` 列来查看哪些文件涉及 `OPEN RESETLOGS` 操作。

为什么要清除联机重做日志中的内容呢？以不完全恢复为例：在此情况下，数据库被故意打开到过去的某个时间点。此时，联机重做日志中的 SCN 信息包含了永远不会被恢复的事务数据。Oracle 强制您使用 `OPEN RESETLOGS` 打开数据库，旨在特意清除这些信息。

当您使用 `OPEN RESETLOGS` 打开数据库时，您会创建数据库的一个新化身，并将日志序列号重置为 1。Oracle 需要一个新化身，以避免在需要另一次还原与恢复时，意外使用任何旧的归档重做日志（这些日志与数据库的单独化身相关联）。

## 总结

一些研究表明，过度依赖自动驾驶仪技术的飞机飞行员，在应对灾难性飞行中问题方面的能力，不如那些花费大量时间在无自动驾驶辅助情况下飞行的飞行员。过度依赖自动驾驶仪的飞行员在出现严重问题时，往往容易忘记关键操作程序；而那些不过度依赖自动驾驶仪的飞行员则更擅长诊断和解决压力下的飞行中故障。

同样地，理解如何使用用户管理技术手动备份、还原和恢复数据库的 DBA，比那些仅通过界面操作备份和恢复技术的 DBA，更精通于排查和解决严重的备份与恢复问题。这正是本章被纳入本书的原因。理解每个步骤发生的情况以及该步骤为何必需，对于全面掌握 Oracle 备份和恢复架构至关重要。在压力巨大的恢复场景中，拥有额外的知识和工具极具价值。这种意识会转化为关键的故障排除技能，当您使用诸如 RMAN（备份与恢复）、Enterprise Manager 和 Data Guard（灾难恢复、高可用性和复制）等 Oracle 工具时，这些技能将大有裨益。

本章介绍的用户管理备份与恢复技术现在已较少教授或使用。大多数 DBA 正在（也应该）使用 RMAN 来满足其 Oracle 备份和恢复需求。然而，理解冷备份和热备份的工作原理对您至关重要。您可能会发现自己就职于一个已采用旧技术的环境，需要还原和恢复数据库、排查故障或协助迁移到 RMAN。在这些场景中，您必须完全理解旧的备份技术。

现在您已经深入理解了 Oracle 备份和恢复机制，可以开始研究 RMAN 了。接下来的几章将探讨如何配置和使用 RMAN 进行生产级的备份与恢复。

# 9. 视图、同义词与序列

本章重点介绍视图、同义词和序列。视图在报表应用程序中被广泛使用，也用于向用户呈现数据的子集。同义词提供了一种透明的方式，允许用户显示和使用其他用户的对象。序列通常用于生成唯一的整数，以填充主键和外键值。

**注意**
尽管视图、同义词和序列看似不如表和索引重要，但事实是，理解它们几乎同样重要。任何具备一定复杂度的应用程序都会涵盖本章讨论的内容。

## 实现视图

从某种意义上说，您可以将视图视为存储在数据库中的 SQL 语句。从概念上讲，当您从视图中选择数据时，Oracle 会在数据字典中查找视图定义，执行视图所基于的查询，并返回结果。

除了从视图中选择数据外，在某些场景下，还可以对视图执行 `INSERT`、`UPDATE` 和 `DELETE` 语句，这会导致底层表数据的修改。因此，从这个意义上说，与其简单地将视图描述为存储的 SQL 语句，不如将其概念化为构建在其他表或视图（或两者兼有）之上的逻辑表更为准确。

综上所述，视图的常见用途如下：
*   创建一种高效存储 SQL 查询以供重用的方法。
*   在应用程序和物理表之间提供一个接口层。
*   向应用程序隐藏 SQL 查询的复杂性。
*   仅向用户报告列或行（或两者）的子集。

考虑到这些，下一步是创建一个视图并观察其部分特性。

## 创建视图

您可以在表、物化视图或其他视图上创建视图。要创建视图，您的用户帐户必须具有 `CREATE VIEW` 系统权限。如果您想在另一个用户的模式中创建视图，则必须具有 `CREATE ANY VIEW` 权限。

供参考，本节的视图创建示例依赖于以下基础表：
```
SQL> create table sales(
sales_id number primary key
,amnt    number
,state   varchar2(2)
,sales_person_id number);
```

同时假设该表最初插入了以下数据：
```
SQL> insert into sales values(1, 222, 'CO', 8773);
SQL> insert into sales values(20, 827, 'FL', 9222);
```

然后，使用 `CREATE VIEW` 语句创建视图。以下代码创建了一个视图（如果视图已存在则替换它），该视图从 `SALES` 表中选择列和行的子集：
```
SQL> create or replace view sales_rockies as
select sales_id, amnt, state
from sales
where state in ('CO','UT','WY','ID','AZ');
```

**注意**
如果您不想意外替换现有的视图定义，请使用 `CREATE VIEW view`，而不是 `CREATE OR REPLACE VIEW view`。如果视图已存在，`CREATE VIEW <view>` 语句将抛出 `ORA-00955` 错误；而 `CREATE OR REPLACE VIEW view` 会覆盖现有定义。

现在，当您从 `SALES_ROCKIES` 中选择数据时，它会执行视图查询，并根据需要从 `SALES` 表返回数据：
```
SQL> select * from sales_rockies;
```

根据视图查询，直观可见输出仅显示以下列和一行数据：
```
SALES_ID       AMNT ST
---------- ---------- --
1        222 CO
```

不太明显的是，您还可以对视图执行 `UPDATE`、`INSERT` 和 `DELETE` 语句，这会导致底层表数据的修改。例如，以下针对视图的插入语句会导致在 `SALES` 表中插入一条记录：
```
SQL> insert into sales_rockies(
sales_id, amnt, state)
values
(2,100,'CO');
```

此外，作为表和视图的所有者（或作为 DBA），您可以授予其他用户对该视图的 DML 权限。例如，您可以将视图的 `SELECT`、`INSERT`、`UPDATE` 和 `DELETE` 权限授予另一个用户，这将允许该用户通过视图选择和修改数据。然而，拥有视图上的权限并不会给予用户对底层表直接进行 SQL 操作的权限。

因此，任何被授予视图权限的用户都将能够通过视图操作数据，但不能对视图所基于的对象直接执行 SQL。

请注意，您可以向视图插入一个值，该值导致在底层表中生成一行数据，但该行可能无法通过该视图被选择出来：


## SQL 视图的数据操作与约束

### 通过视图进行数据操作

```sql
SQL> insert into sales_rockies(
sales_id, amnt, state)
values (3,123,'CA');
SQL> select * from sales_rockies;
SALES_ID       AMNT ST
---------- ---------- --
1        222 CO
2        100 CO
```

相比之下，对基础表的查询显示存在一些视图无法返回的行：

```sql
SQL> select * from sales;
SALES_ID       AMNT ST SALES_PERSON_ID
---------- ---------- -- ---------------
1        222 CO            8773
20        827 FL            9222
2        100 CO
3        123 CA
```

如果你希望视图只允许那些导致数据修改结果能被视图查询语句选中的 `INSERT` 和 `UPDATE` 语句，那么请使用 `WITH CHECK OPTION`（参见下一节“检查更新”）。

### 检查更新

你可以指定一个视图，仅当底层表数据能被该视图选中时才允许修改。这种行为通过 `WITH CHECK OPTION` 启用：

```sql
SQL> create or replace view sales_rockies as
select sales_id, amnt, state
from sales
where state in ('CO','UT','WY','ID','AZ')
with check option;
```

使用 `WITH CHECK OPTION` 意味着你只能插入或更新那些会被视图查询返回的行。例如，这个 `UPDATE` 语句可以成功，因为它并未以导致该行不被视图查询返回的方式更改底层数据：

```sql
SQL> update sales_rockies set state='ID' where sales_id=1;
```

然而，下一个更新语句会失败，因为它试图将 `STATE` 列更新为一个不被视图基础查询选中的值：

```sql
SQL> update sales_rockies set state='CA' where sales_id=1;
```

在此示例中，会抛出以下错误：

```
ORA-01402: view WITH CHECK OPTION where-clause violation
```

我很少见到 `WITH CHECK OPTION` 被使用。话虽如此，如果你的业务要求强制规定可更新视图只能更新视图查询可选中的数据，那么请务必使用此特性。

### 创建只读视图

如果你不希望用户能够对视图执行 `INSERT`、`UPDATE` 或 `DELETE` 操作，那么不要将该视图的这些对象权限授予该用户。此外，对于任何你不希望其底层表被修改的视图，你还应该使用 `WITH READ ONLY` 子句来创建视图。默认行为是视图是可更新的（假设存在对象权限）。

此示例使用 `WITH READ ONLY` 子句创建视图：

```sql
SQL> create or replace view sales_rockies as
select sales_id, amnt, state
from sales
where state in ('CO','UT','WY','ID','AZ')
with read only;
```

即使用户（包括所有者）拥有删除、插入或更新该视图的权限，如果尝试执行此类操作，也会抛出以下错误：

```
ORA-42399: cannot perform a DML operation on a read-only view
```

如果你将视图用于报告，并且从不打算将视图用作修改底层表数据的机制，那么你应该始终使用 `WITH READ ONLY` 子句创建视图。这样做可以防止通过从未打算用于修改数据的视图，意外修改底层表。

### 可更新连接视图

如果在视图所基于的 SQL 查询的 `FROM` 子句中定义了多个表，仍然可以更新底层表。这被称为可更新连接视图。

作为参考，以下是本节示例中使用的两个表的 `CREATE TABLE` 语句：

```sql
SQL> create table emp(
emp_id number primary key
,emp_name varchar2(15)
,dept_id number);
--
SQL> create table dept(
dept_id number primary key
,dept_name varchar2(15),
constraint emp_dept_fk
foreign key(dept_id) references dept(dept_id));
```

以及这两个表的一些种子数据：

```sql
SQL> insert into dept values(1,'HR');
SQL> insert into dept values(2,'IT');
SQL> insert into dept values(3,'SALES');
SQL> insert into emp values(10,'John',2);
SQL> insert into emp values(20,'Bob',1);
SQL> insert into emp values(30,'Craig',2);
SQL> insert into emp values(40,'Joe',3);
SQL> insert into emp values(50,'Jane',1);
SQL> insert into emp values(60,'Mark',2);
```

以下是一个可更新连接视图的示例，基于前述两个基表：

```sql
SQL> create or replace view emp_dept_v
as
select a.emp_id, a.emp_name, b.dept_name, b.dept_id
from emp a, dept b
where a.dept_id = b.dept_id;
```

关于允许执行 DML 操作的列，存在一些限制。例如，底层表中的列仅在以下条件为真时才能被更新：

*   DML 语句必须只修改一个底层表。
*   视图创建时不能带有 `READ ONLY` 子句。
*   被更新的列属于连接视图中的键保留表（一个连接视图中只有一个键保留表）。

如果表的主键也可用于唯一标识视图返回的行，则该表在视图中是键保留的。一个数据示例将有助于说明底层表是否是键保留的。在此场景中，`EMP` 表的主键是 `EMP_ID` 列；`DEPT` 表的主键是 `DEPT_ID` 列。以下是对本节前面列出的视图进行查询返回的一些示例数据：

```
EMP_ID EMP_NAME        DEPT_NAME          DEPT_ID
---------- --------------- --------------- ----------
10 John            IT                       2
20 Bob             HR                       1
30 Craig           IT                       2
40 Joe             SALES                    3
50 Jane            HR                       1
60 Mark            IT                       2
```

从视图的输出可以看出，`EMP_ID` 列始终是唯一的。因此，`EMP` 表是键保留的（并且其列可以被更新）。相比之下，视图的输出显示 `DEPT_ID` 列可能不唯一。因此，`DEPT` 表不是键保留的（并且其列不能被更新）。

当你更新视图时，任何导致映射到底层 `EMP` 表的列被修改的操作都应被允许，因为 `EMP` 表在此视图中是键保留的。例如，这个 `UPDATE` 语句是成功的：

```sql
SQL> update emp_dept_v set emp_name = 'Jon' where emp_name = 'John';
```

但是，导致更新 `DEPT` 表列的语句是不允许的。下一条语句试图更新视图中映射到 `DEPT` 表的一列：

```sql
SQL> update emp_dept_v set dept_name = 'HR West' where dept_name = 'HR';
```

以下是抛出的错误消息：

```
ORA-01779: cannot modify a column which maps to a non key-preserved table
```

总结来说，一个可更新连接视图可以从多个表中选择数据，但连接视图中只有一个表是键保留的。查询中表的主键和外键关系决定了哪个表是键保留的。


# 创建 INSTEAD OF 触发器

对于非只读视图，当你对视图发出 DML 语句时，Oracle 会尝试修改该视图所基于的表中的数据。也可以指示 Oracle 忽略该 DML 语句，转而执行一个 PL/SQL 块。此特性被称为 `INSTEAD OF` 触发器。它允许你以常规连接视图无法实现的方式修改底层基表。

我并不十分推崇 `INSTEAD OF` 触发器。
在我看来，如果你在考虑使用它们，你应该重新思考如何发出 DML 语句来修改基表。也许你应该允许应用程序直接对基表发出 `INSERT`、`UPDATE` 和 `DELETE` 语句，而不是尝试在视图上构建 PL/SQL `INSTEAD OF` 触发器。

思考一下你将如何维护和排查与 `INSTEAD OF` 触发器相关的问题。对于下一个 DBA 来说，弄清楚基表是如何被修改的是否会很困难？对于下一个 DBA 或开发人员来说，修改 `INSTEAD OF` 触发器是否容易？
当 `INSTEAD OF` 触发器抛出错误时，是什么代码抛出错误以及如何解决问题是否显而易见？

话虽如此，如果你确定需要在视图上使用 `INSTEAD OF` 触发器，请使用 `INSTEAD OF` 子句来创建它，并在其中嵌入所需的 PL/SQL。此示例在 `EMP_DEPT_V` 视图上创建了一个 `INSTEAD OF` 触发器：

```
SQL> create or replace trigger emp_dept_v_updt
instead of update on emp_dept_v
for each row
begin
update emp set emp_name=UPPER(:new.emp_name)
where emp_id=:old.emp_id;
end;
/
```

现在，当对 `EMP_DEPT_V` 发出更新时，DML 不会被执行，Oracle 会拦截该语句并运行 `INSTEAD OF` PL/SQL 代码；例如：

```
SQL> update emp_dept_v set emp_name='Jonathan' where emp_id = 10;
1 row updated.
```

然后，你可以通过查询数据来验证触发器是否正确更新了表：

```
SQL> select * from emp_dept_v;
EMP_ID EMP_NAME        DEPT_NAME          DEPT_ID
---------- --------------- --------------- ----------
10 JONATHAN        IT                       2
20 Bob             HR                       1
30 Craig           IT                       2
40 Joe             SALES                    3
50 Jane            HR                       1
60 Mark            IT                       2
```

这段代码是一个简单的例子，但它说明了你可以让 PL/SQL 在视图上运行的 DML 之前执行。再次强调，使用 `INSTEAD OF` 触发器时要谨慎；确保你有信心能够高效地诊断和解决可能出现的任何相关问题。

# 实现不可见列

从 Oracle Database 12c 开始，你可以创建或修改表或视图中的列为不可见（有关向表添加不可见列的详细信息，请参见第 7 章）。不可见列的一个很好的用途是确保向表或视图添加列不会破坏任何现有的应用程序代码。如果应用程序代码没有显式访问不可见列，那么对应用程序来说，就好像该列不存在一样。

一个小例子将演示不可见列的用途。假设你有一个创建并填充了某些数据的表，如下所示：

```
SQL> create table sales(
sales_id number primary key
,amnt    number
,state   varchar2(2)
,sales_person_id number);
--
SQL> insert into sales values(1, 222, 'CO', 8773);
SQL> insert into sales values(20, 827, 'FL', 9222);
```

并且，你有一个基于前面表的视图，创建方式如下：

```
SQL> create or replace view sales_co as
select sales_id, amnt, state
from sales where state = 'CO';
```

出于本示例的目的，假设你还有一个这样的报表表：

```
SQL> create table rep_co(
sales_id number
,amnt    number
,state   varchar2(2));
```

并且，它使用 `SELECT *` 的插入语句填充：

```
SQL> insert into rep_co select * from sales_co;
```

一段时间后，向视图中添加了一个新列：

```
SQL> create or replace view sales_co as
select sales_id, amnt, state, sales_person_id
from sales where state = 'CO';
```

现在，考虑一下插入到 `REP_CO` 中的语句会发生什么变化。因为它使用了 `SELECT *`，所以会中断，因为没有在 `REP_CO` 表中添加相应的列：

```
SQL> insert into rep_co select * from sales_co;
ORA-00913: too many values
```

之前的插入语句不再能够填充 `REP_CO` 表，因为该语句没有考虑到已添加到视图中的额外列。

现在，考虑相同的场景，但是使用不可见列将列添加到 `SALES_CO` 视图：

```
SQL> create or replace view sales_co
(sales_id, amnt, state, sales_person_id invisible)
as
select
sales_id, amnt, state, sales_person_id
from sales
where state = 'CO';
```

当视图列定义为不可见时，这意味着在描述视图或在 `SELECT *` 的输出中，该列不会出现。这确保了基于 `SELECT *` 的插入语句可以继续工作。
有人可能有力地争辩说，永远不应该创建基于 `SELECT *` 的插入语句，因此你永远不会遇到此问题。或者，有人可能争辩说本示例中的 `REP_CO` 表也应该添加一列以避免该问题。然而，在与第三方应用程序合作时，你通常无法控制编写拙劣的代码。在这种情况下，你可以向视图添加不可见列，而不必担心破坏任何现有代码。

话虽如此，不可见列并非完全不可见。如果你知道不可见列的名称，你可以直接从中选择；例如：

```
SQL> select sales_id, amnt, state, sales_person_id from sales_co;
```

从这个意义上说，不可见列只是对编写拙劣的应用程序代码或不知道该列存在的用户不可见。

# 修改视图定义

如果需要修改视图所基于的 SQL 查询，那么可以删除并重新创建视图，或者使用 `CREATE OR REPLACE` 语法，如前面的示例所示。例如，假设你向 `SALES` 表添加了一个 `REGION` 列：

```
SQL> alter table sales add (region varchar2(30));
```

现在，要将 `REGION` 列添加到 `SALES_ROCKIES` 视图，请运行以下命令以替换现有的视图定义：

```
SQL> create or replace view sales_rockies as
select sales_id, amnt, state, region
from sales
where state in ('CO','UT','WY','ID','AZ')
with read only;
```

使用 `CREATE OR REPLACE` 方法的优点是，你不必为先前已授予权限的用户重新建立对视图的访问权限。`CREATE OR REPLACE` 的替代方法是删除并重新创建具有新定义的视图。如果删除并重新创建视图，你必须重新授予先前被授予对已删除和重新创建对象访问权限的任何用户或角色的权限。因此，在更改视图结构时，我几乎从不使用删除并重新创建的方法。

如果你从表中删除一列，并且有一个对应的视图引用了被删除的列，会发生什么？例如：

```
SQL> alter table sales drop (region);
```

如果你尝试从视图中选择，将会收到 `ORA-04063` 错误。在修改底层表时，你可以通过编译视图来检查视图是否受到表更改的影响；例如：

```
SQL> alter view sales_rockies compile;
Warning: View altered with compilation errors.
```

通过这种方式，你可以主动确定表更改是否影响依赖的视图。在这种情况下，你应该重新创建视图，省略被删除的表列：


# SQL 视图与同义词管理

## 显示用于创建视图的 SQL

有时，当您排查视图返回的信息问题时，需要查看该视图所基于的 SQL 查询。视图定义存储在`DBA/USER/ALL_VIEWS`视图的`TEXT`列中。请注意，`DBA/USER/ALL_VIEWS`视图的`TEXT`列是`LONG`数据类型，并且默认情况下，SQL*Plus 仅显示该类型的 80 个字符。您可以按如下方式将其设置得更长：

```
SQL> set long 5000
```

现在，使用以下脚本显示特定用户视图关联的文本：

```
SQL> select view_name, text
from dba_views
where owner = upper('&owner')
and view_name like upper('&view_name');
```

您也可以查询`ALL_VIEWS`以获取您有权访问的任何视图的文本：

```
SQL> select text
from all_views
where owner='MV_MAINT'
and view_name='SALES_ROCKIES';
```

如果您想显示当前模式中存在的视图文本，请使用`USER_VIEWS`：

```
SQL> select text
from user_views
where view_name=upper('&view_name');
```

> **注意**
> `DBA/ALL/USER_VIEWS`的`TEXT`列不会隐藏有关定义为不可见的列的信息。

您也可以使用`DBMS_METADATA`包的`GET_DDL`函数来显示视图的代码。`GET_DDL`返回的数据类型是`CLOB`；因此，如果您从 SQL*Plus 运行它，请确保首先将`LONG`变量设置为足够大的尺寸以显示所有文本。例如，以下是如何将`LONG`设置为 5,000 个字符：

```
SQL> set long 5000
```

现在，您可以通过`SELECT`语句调用`DBMS_METADATA.GET_DDL`来显示视图定义，如下所示：

```
SQL> select dbms_metadata.get_ddl('VIEW','SALES_ROCKIES') from dual;
```

如果要显示当前连接用户所有视图的 DDL，请运行以下 SQL：

```
SQL> select dbms_metadata.get_ddl('VIEW', view_name) from user_views;
```

## 重命名视图

重命名视图有几个充分的理由。您可能希望更改名称以使其更好地符合标准名称，或者您可能希望在删除视图之前对其进行重命名，以便更好地确定它是否正在使用。使用`RENAME`语句来更改视图的名称。以下示例重命名了一个视图：

```
SQL> rename sales_rockies to sales_rockies_old;
```

您应该看到此消息：

```
Table renamed.
```

如果说“View renamed.”，前面的消息会更有意义；只是请注意，在这种情况下，消息与操作并不完全匹配。

## 删除视图

在删除视图之前，请考虑对其进行重命名。如果您确定某个视图不再使用，那么为了尽可能保持模式整洁并删除任何未使用的对象是有意义的。使用`DROP VIEW`语句来删除视图：

```
SQL> drop view sales_rockies_old;
```

请记住，当您删除视图时，任何依赖的视图、物化视图和同义词都会变得无效。此外，与已删除视图关联的任何权限也会被移除。

## 管理同义词

同义词提供了一种为对象创建替代名称或别名的机制。例如，假设`USER1`是当前连接的用户，并且`USER1`对`USER2`的`EMP`表具有选择访问权限。如果没有同义词，`USER1`必须从`USER2`的`EMP`表中进行选择，如下所示：

```
SQL> select * from user2.emp;
```

假设它具有`CREATE SYNONYM`系统权限，`USER1`可以执行以下操作：

```
SQL> create synonym emp for user2.emp;
```

现在，`USER1`可以透明地从`USER2`的`EMP`表中进行选择：

```
SQL> select * from emp;
```

您可以为以下类型的数据库对象创建同义词：

*   表
*   视图、对象视图
*   其他同义词
*   通过数据库链接的远程对象
*   PL/SQL 包、过程和函数
*   物化视图
*   序列
*   Java 类模式对象
*   用户定义的对象类型

创建一个指向另一个对象的同义词消除了指定模式所有者的需要，并且还允许您为同义词指定一个与对象名称不匹配的名称。这使您可以在对象和用户之间创建一个抽象层，通常称为对象透明性。同义词允许您透明地管理对象，并与访问对象的用户分开。您还可以将对象无缝地重新定位到不同的模式甚至不同的数据库。引用这些序列的应用程序代码不需要更改——只需要更改同义词的定义。现在，仅使用模式账户，同义词将允许引用对象，因为没有账户登录到这些模式中。

> **提示**
> 您可以使用同义词在一个数据库中设置多个应用程序环境。每个环境都有自己的同义词，指向不同用户的对象，允许您在一个数据库中的几个不同模式上运行相同的代码。您这样做可能是因为无法为开发、测试、质量保证、生产等构建单独的机器或数据库。

### 创建同义词

用户必须被授予`CREATE SYNONYM`系统权限才能创建同义词。一旦授予该权限，请使用`CREATE SYNONYM`命令为另一个数据库对象创建别名。如果您希望该语句在同义词不存在时创建它，或者在它存在时替换同义词定义，可以指定`CREATE OR REPLACE SYNONYM`。这通常是可接受的行为。

在此示例中，将创建一个提供对某个视图访问权限的同义词。首先，视图的所有者必须授予对该视图的选择访问权限。这里，视图的所有者是`MV_MAINT`：

```
SQL> show user;
USER is "MV_MAINT"
SQL> grant select on sales_rockies to app_user;
```

接下来，以将创建同义词的用户身份连接到数据库。

```
SQL> conn app_user/foo
```

以`APP_USER`身份连接时，创建一个指向`MV_MAINT`拥有的名为`SALES_ROCKIES`的视图的同义词：

```
SQL> create or replace synonym sales_rockies for mv_maint.sales_rockies;
```

现在，`APP_USER`可以直接引用`SALES_ROCKIES`视图：

```
SQL> select * from sales_rockies;
```

使用`CREATE SYNONYM`命令时，如果您未指定`OR REPLACE`（如示例所示），并且同义词已存在，则会抛出`ORA-00955`错误。如果可以覆盖任何先前存在的同义词定义，则指定`OR REPLACE`子句。

同义词的创建并不同时创建访问对象的权限。此类权限必须单独授予，通常在创建同义词之前授予（如示例所示）。
默认情况下，当您创建同义词时，它是私有同义词。这意味着它由创建同义词的用户拥有，并且除非其他用户被授予适当的对象权限，否则无法访问它。

### 创建公共同义词

您也可以将同义词定义为公共的（有关私有同义词的讨论，请参见前一节“创建同义词”），这意味着数据库中的任何用户都可以访问该同义词。有时，缺乏经验的 DBA 会执行以下操作：

```
SQL> grant all on sales to public;
SQL> create public synonym sales for mv_maint.sales;
```

现在，任何可以连接到数据库的用户都可以对`MV_MAINT`模式中存在的`SALES`表执行任何`INSERT`、`UPDATE`、`DELETE`或`SELECT`操作。您可能很想这样做，这样就不必为每个需要访问的模式单独设置权限和同义词。这几乎总是一个坏主意。使用公共同义词存在一些问题：

*   如果您不知道全局定义（公共）同义词，故障排除可能会很成问题；DBA 往往会忘记或不知道创建了公共同义词。



# 数据库同义词管理最佳实践

## 公共同义词的问题

*   共享一个数据库的应用程序，如果多个应用程序使用在数据库中非唯一的公共同义词，可能会在对象名称上发生冲突。

*   安全性应按需管理，而不是整体统一管理。

我通常尽量避免使用公共同义词。然而，可能存在一些场景需要使用它们。例如，当 Oracle 创建数据字典时，会使用公共同义词来简化对内部数据库对象的访问管理。要显示数据库中任何公共同义词，请运行此查询：

```sql
SQL> select owner, synonym_name
from dba_synonyms
where owner='PUBLIC';
```

## 动态生成同义词

有时，为需要私有同义词的模式动态生成所有表或视图的同义词很有用。以下脚本使用`SQL*Plus`命令来格式化并捕获 SQL 脚本的输出，该脚本为模式中的所有表生成同义词：

```sql
SQL> CONNECT &&master_user/&&master_pwd
--
SQL> SET LINESIZE 132 PAGESIZE 0 ECHO OFF FEEDBACK OFF
SQL> SET VERIFY OFF HEAD OFF TERM OFF TRIMSPOOL ON
--
SQL> SPO gen_syns_dyn.sql
--
SQL> select 'create or replace synonym ' || table_name ||
' for ' || '&&master_user..' ||
table_name || ';'
from user_tables;
--
SQL> SPO OFF;
--
SQL> SET ECHO ON FEEDBACK ON VERIFY ON HEAD ON TERM ON;
```

看看`SELECT`语句中带有两个点附加的`&&master_user`变量：双点语法的目的是什么？`&`变量末尾的一个点指示`SQL*Plus`将任何跟在单点之后的内容连接到该`&`变量。当你把两个点放在一起时，这告诉`SQL*Plus`将单个点连接到`&`变量中包含的字符串。

## 显示同义词元数据

`DBA/ALL/USER_SYNONYMS`视图包含数据库中同义词的信息。使用以下 SQL 查看当前连接用户的同义词元数据：

```sql
SQL> select synonym_name, table_owner, table_name, db_link
from user_synonyms
order by 1;
```

`ALL_SYNONYMS`视图显示所有私有同义词、所有公共同义词，以及当前连接用户具有底层基表`SELECT`访问权限的、由不同用户拥有的任何私有同义词。你可以通过查询`DBA_SYNONYMS`视图来显示数据库中所有私有和公共同义词的信息。

`DBA/ALL/USER_SYNONYMS`视图中的`TABLE_NAME`列有点用词不当，因为`TABLE_NAME`可以引用多种类型的数据库对象，例如另一个同义词、视图、包、函数、过程或物化视图。类似地，`TABLE_OWNER`指的是对象的拥有者（该对象不一定是一张表）。

当你诊断数据完整性问题时，有时首先需要识别正在访问的是哪个表或对象。你可能从看似表的对象中进行选择，但实际上，它可能是一个指向视图的同义词，而该视图又从另一个数据库中指向表的同义词中进行选择。

以下查询通常是判断一个对象是同义词、视图还是表的起点：

```sql
SQL> select owner, object_name, object_type, status
from dba_objects
where object_name like upper('&object_name%');
```

请注意，在此查询中使用通配符百分号（`%`）允许你输入对象的部分名称。因此，该查询有可能返回与你输入的文本字符串部分匹配的任何对象的信息。

你还可以使用`DBMS_METADATA`包的`GET_DDL`函数来显示同义词元数据。如果你想显示当前连接用户的所有同义词的 DDL，请运行此 SQL：

```sql
SQL> set long 5000
SQL> select dbms_metadata.get_ddl('SYNONYM', synonym_name) from user_synonyms;
```

你也可以显示特定用户的 DDL。你必须将对象类型、对象名称和模式作为输入提供给`GET_DDL`函数：


# 管理同义词与序列

## 重命名同义词

您可能希望重命名一个同义词，以使其符合命名标准，或者用于确定它是否正在被使用。可以使用 `RENAME` 语句来更改同义词的名称：

```
SQL> rename inv_s to inv_st;
```

请注意，输出会显示此消息：

```
Table renamed.
```

上面的消息有些误导性。它表示一个表已被重命名，而在本场景中，实际被重命名的是一个同义词。

## 删除同义词

如果您确定不再需要某个同义词，则可以删除它。未使用的同义词可能会对将来需要增强或调试现有应用程序的其他人造成困惑。使用 `DROP SYNONYM` 语句来删除一个私有同义词：

```
SQL> drop synonym inv;
```

如果要删除的是一个公有同义词，则需要在删除时指定 `PUBLIC`：

```
SQL> drop public synonym inv_pub;
```

如果成功，您将看到如下消息：

```
Synonym dropped.
```

# 管理序列

序列是一种数据库对象，用户可以通过访问它来选择唯一的整数。序列通常用于为填充主键和外键列生成整数。您可以通过在 `SELECT`、`INSERT` 或 `UPDATE` 语句中访问序列来使其递增。Oracle 保证在选择时序列号是唯一的；没有两个用户会话可以选择相同的序列号。

无法保证序列生成的数字中不会偶尔出现缺口。通常，一定数量的序列值会被缓存在内存中，如果发生实例故障（断电、异常关闭），内存中任何未使用的值都将丢失。即使您不缓存序列，也无法阻止用户在事务中获取一个序列号然后回滚该事务（事务被回滚，但序列不会回滚）。对于大多数应用程序来说，生成一个基本连续且唯一的整数生成器是可以接受的。只需意识到可能存在缺口即可。

## 创建序列

对于许多应用，创建一个序列可以像这样简单：

```
SQL> create sequence inv_seq;
```

默认情况下，起始数字为 1，增量为 1，默认缓存到内存中的序列数量为 20，最大值为 `10²⁷`。您可以通过此查询验证默认值：

```
SQL> select sequence_name, min_value, increment_by, cache_size, max_value
from user_sequences;
```

在创建序列时，您在更改其各个属性方面拥有很大的自由度。例如，以下命令创建一个起始值为 1,000、最大值为 1,000,000 的序列：

```
SQL> create sequence inv_seq2 start with 10000 maxvalue 1000000;
```

表 9-1 列出了创建序列时可用的各种选项。

表 9-1
序列创建选项

| 选项 | 描述 |
| :--- | :--- |
| `INCREMENT BY` | 指定序列号之间的间隔 |
| `START WITH` | 指定生成的第一个序列号 |
| `MAXVALUE` | 指定序列的最大值 |
| `NOMAXVALUE` | 将序列的最大值设置为一个非常大的数字（`10²⁸ -1`） |
| `MINVALUE` | 指定序列的最小值 |
| `NOMINVALUE` | 对于升序序列，将最小值设置为 1；对于降序序列，将值设置为 `–(10²⁸–1)` |
| `CYCLE` | 指定当序列达到最大值或最小值时，应重新从最小值开始生成数字（对于升序序列），或从最大值开始（对于降序序列） |
| `NOCYCLE` | 指示序列在达到最大值或最小值后停止生成数字 |
| `CACHE` | 指定要预分配并保留在内存中的序列号数量。如果未指定 `CACHE` 和 `NOCACHE`，则默认为 `CACHE 20` |
| `NOCACHE` | 指定序列号不应被缓存 |
| `ORDER` | 保证按请求的顺序生成数字 |
| `NOORDER` | 如果不需要保证按请求的顺序生成序列号时使用。这通常是可接受的，并且是默认设置 |
| `SCALE` | 通过使用消除所有重复项的数值偏移量来启用序列可伸缩性。建议在使用 `SCALE` 时不要使用 `ORDER` |
| `EXTEND` | 默认值为 6，是可伸缩偏移的值 |
| `NOEXTEND` | `SCALE` 子句的默认设置。序列宽度将仅限于序列中的最大位数 |
| `NOSCALE` | 禁用序列可伸缩性 |

## 使用序列伪列

创建序列后，您可以使用两个伪列来访问序列的值：
*   `NEXTVAL`
*   `CURRVAL`

您可以在任何 `SELECT`、`INSERT` 或 `UPDATE` 语句中引用这些伪列。要从 `INV_SEQ` 序列中检索一个值，请访问 `NEXTVAL` 值，如下所示：

```
SQL> select inv_seq.nextval from dual;
```

既然此会话已经检索到一个序列号，您可以通过访问 `CURRVAL` 值来多次使用它：

```
SQL> select inv_seq.currval from dual;
```

下面的示例使用一个序列来填充父表的主键值，然后使用相同的序列来填充子表中相应的外键值。可以在 `INSERT` 语句中直接访问序列。第一次访问序列时，请使用 `NEXTVAL` 伪列。

```
SQL> insert into inv(inv_id, inv_desc) values (inv_seq.nextval, 'Book');
```

如果想重用相同的序列值，可以通过 `CURRVAL` 伪列来引用它。接下来，向子表插入一条记录，该记录的外键列使用与其父主键值相同的值：

```
SQL> insert into inv_lines
(inv_line_id,inv_id,inv_item_desc)
values
(1, inv_seq.currval, 'Tome1');
--
SQL> insert into inv_lines
(inv_line_id,inv_id,inv_item_desc)
values
(2, inv_seq.currval, 'Tome2');
```

## 自增列

> **提示**
> 从 Oracle Database 12c 开始，您可以创建具有自增列的表，这些列会自动使用序列值填充。详情请参见第 7 章。

如果您无法使用自增列，则可以通过使用触发器来模拟这种自动递增功能。例如，假设您创建了一个表和序列，如下所示：

```
SQL> create table inv(inv_id number, inv_desc varchar2(30));
SQL> create sequence inv_seq;
```

接下来，在 `INV` 表上创建一个触发器，该触发器会自动从序列中为 `INV_ID` 列填充值：

```
SQL> create or replace trigger inv_bu_tr
before insert on inv
for each row
begin
select inv_seq.nextval into :new.inv_id from dual;
end;
/
```

现在，向 `INV` 表插入几条记录：

```
SQL> insert into inv (inv_desc) values( 'Book');
SQL> insert into inv (inv_desc) values( 'Pen');
```

从表中查询以验证 `INV_ID` 列是否确实由序列自动填充：

```
SQL> select * from inv;
INV_ID INV_DESC
---------- ------------------------------
1 Book
2 Pen
```

我通常不喜欢使用这种技术。是的，它使开发人员的工作更轻松，因为他们不必担心填充键列。然而，对于 DBA 来说，生成维护这些自增列所需的代码却意味着更多的工作。因为我是 DBA，我喜欢保持我所维护的数据库代码尽可能简单，所以我通常告诉开发人员，我们不会使用这种自增列方法，而是使用在 DML 语句中直接调用序列的技术（如前一节“使用序列伪列”所示）。

# 可扩展序列

一项新功能是可扩展序列，它允许为序列添加一个唯一的数值偏移量前缀。这与第 8 章讨论的反向键索引类似，有助于缓解热点问题，并让所有服务器使用相同的偏移量，使数字聚集在一起。

可扩展序列使用 `SCALE`、`EXTEND`、`NOEXTEND` 和 `NOSCALE` 选项。

```sql
SQL> create sequence inv_seq scale;
```

有关缩放选项的信息存储在序列字典列 `SCALE_FLAG` 和 `EXTEND_FLAG` 中。

```sql
SQL> select sequence_name, scale_flag, extend_flag from dba_sequences
where sequence_name='INV_SEQ';
SEQUENCE_NAME                  SCALE_FLAG   EXTEND_FLAG

INV_SEQ                        Y            N
SQL> select inv_seq.nextval from dual;
NEXTVAL

```

前三个数值基于实例，接下来的三个基于会话，最后一个数字是序列的下一个值。

# 无间隙序列

人们有时会过分担心在向表中插入行时，确保不丢失任何一个序列值。在少数情况下，我见过应用程序因序列值中存在间隙而失败。对于这些问题，我有两个想法：

*   如果你担心间隙，说明你没有正确思考你正在解决的问题。
*   如果你的应用程序因为间隙而失败，那么你的做法是错误的。

我的话可能很重，我知道，但几乎没有应用程序需要无间隙序列。如果你真的需要无间隙序列，那么使用 Oracle 序列对象是错误的方法。你必须自己实现一个序列生成器。你将不得不经历痛苦的扭曲操作来确保没有间隙存在。这些扭曲操作会损害代码的性能。而且，最终你可能会失败。

# 实现生成唯一值的多个序列

曾经有开发人员询问是否可以为应用程序创建多个序列，并保证每个序列生成的数字在所有序列中都是唯一的。如果你有这种类型的需求，你可以通过几种不同的方式处理：

*   如果你心情不好，可以告诉开发人员这是不可能的，标准是每个应用程序使用一个序列（这通常是我采取的方法）。
*   设置序列以不同的起始点和增量开始。
*   使用序列号范围。

如果你心情不错，你可以设置少量、有限数量的序列，这些序列始终生成唯一值，方法是指定奇数或偶数起始编号，然后以二递增序列。例如，你可以设置两个奇数和两个偶数序列生成器；例如，

```sql
SQL> create sequence inv_seq_odd start with 1 increment by 2;
SQL> create sequence inv_seq_even start with 2 increment by 2;
SQL> create sequence inv_seq_odd_dwn start with -1 increment by -2;
SQL> create sequence inv_seq_even_dwn start with -2 increment by -2;
```

这四个序列生成的数字应该永远不会交叉。然而，这种方法仅限于只能使用四个序列。

如果你需要超过四个唯一序列，你可以使用数字范围；例如，

```sql
SQL> create sequence inv_seq_low start with 1 increment by 1 maxvalue 10000000;
SQL> create sequence inv_seq_ml  start with 10000001 increment by 1 maxvalue 20000000;
SQL> create sequence inv_seq_mh  start with 20000001 increment by 1 maxvalue 30000000;
SQL> create sequence inv_seq_high start with 30000001 increment by 1 maxvalue 40000000;
```

通过这种技术，你可以设置大量不同的数字范围供每个序列使用。缺点是每个序列可以生成的唯一值数量是有限的。

# 创建一个序列还是多个序列

假设你有一个包含 20 个表的应用程序。一个问题是，你应该使用 20 个不同的序列来填充每个表的主键和外键列，还是只使用 1 个序列。

我建议只使用 1 个序列；1 个序列比多个序列更容易管理，这意味着需要管理的 DDL 代码更少，在出现问题时需要调查的地方也更少。

有时，开发人员会提出以下问题：

*   只使用 1 个序列的性能问题
*   序列号变得过高

如果你缓存序列值，访问序列通常不会有性能问题。序列的最大数字是 `10²⁸–1`，所以如果序列以一递增，你永远无法达到最大值（至少，这辈子不行）。

然而，在你为主表和子表生成代理键的场景中，有时使用多个序列会更方便。在这些情况下，可能需要为每个应用程序使用多个序列。当你使用这种方法时，必须记住在添加表时添加序列，并在删除表时可能删除序列。这不是什么大问题，但这意味着 DBA 需要进行更多的维护，开发人员必须确保为每个表使用正确的序列。

# 查看序列元数据

如果你拥有 DBA 权限，可以查询 `DBA_SEQUENCES` 视图以显示数据库中所有序列的信息。要查看你模式拥有的序列，请查询 `USER_SEQUENCES` 视图：

```sql
SQL> select sequence_name, min_value, max_value, increment_by
from user_sequences;
```

要查看重新创建序列所需的 DDL 代码，请访问 `DBMS_METADATA` 视图。如果你使用 SQL\*Plus 执行 `DBMS_METADATA`，请首先确保设置了 `LONG` 变量：

```sql
SQL> set long 5000
```

此示例提取 `INV_SEQ` 的 DDL：

```sql
SQL> select dbms_metadata.get_ddl('SEQUENCE','INV_SEQ') from dual;
```

如果你想显示当前连接用户的所有序列的 DDL，请运行此 SQL：

```sql
SQL> select dbms_metadata.get_ddl('SEQUENCE',sequence_name) from user_sequences;
```

你还可以通过提供 `SCHEMA` 参数来为特定用户拥有的序列生成 DDL：

```sql
SQL> select
dbms_metadata.get_ddl(object_type=>'SEQUENCE', name=>'INV_SEQ', schema=>'INV_APP')
from dual;
```

# 重命名序列

有时，你可能需要重命名序列。例如，序列可能以错误的名称创建，或者你可能想在从数据库中删除序列之前将其重命名。使用 `RENAME` 语句来完成此操作。此示例将 `INV_SEQ` 重命名为 `INV_SEQ_OLD`：

```sql
SQL> rename inv_seq to inv_seq_old;
```

你应该看到以下消息：

```
Table renamed.
```

在这种情况下，即使消息显示“表已重命名”，被重命名的实际上是序列。

# 删除序列

通常，你想删除序列要么是因为它未被使用，要么是想用新的起始编号重新创建它。要删除序列，请使用 `DROP SEQUENCE` 语句：

```sql
SQL> drop sequence inv_seq;
```

当对象被删除时，所有与该对象相关的授权也会被删除。因此，如果你需要重新创建序列，请记住重新向可能需要使用该序列的其他用户授予 select 权限。

> **提示**
> 有关删除和重新创建序列的替代方法，请参见下一节“重置序列”。

# 重置序列

你可能偶尔需要更改序列号的当前值。例如，你可能在测试环境中工作，开发人员希望定期将数据库重置为之前的状态。一个典型的情况是，开发人员有脚本可以截断表并用测试数据重新播种，作为该操作的一部分，他们希望将序列重置回像 1 这样的值。

Oracle 的文档指出，“要从一个不同的数字重新启动序列，必须删除并重新创建它。”这并不完全准确。在大多数情况下，应避免删除序列，因为你必须重新为当前对序列拥有选择权限的用户授予对象权限。这可能导致你的应用程序在追踪这些用户期间出现暂时停机。

以下技术展示了如何使用 `ALTER SEQUENCE` 语句将当前值设置为更高或更低的值。基本步骤如下：

1.  将 `INCREMENT BY` 更改为一个大数。
2.  从序列中选择以增加该大的正值或负值。
3.  将 `INCREMENT BY` 设置回其原始值（通常为 1）。

此示例将序列号的下一个值设置为比当前值高 1,000 个整数：

```
SQL> alter sequence myseq increment by 1000;
SQL> select myseq.nextval from dual;
SQL> alter sequence myseq increment by 1;
```

验证序列是否设置为你期望的值：

```
SQL> select myseq.nextval from dual;
```

你也可以使用此技术将序列号设置为比当前值低得多的数字。区别在于 `INCREMENT BY` 的设置是一个大的负数。例如，此示例将序列回退 1,000 个整数：

```
SQL> alter sequence myseq increment by -1000;
SQL> select myseq.nextval from dual;
SQL> alter sequence myseq increment by 1;
```

验证序列是否设置为你期望的值：

```
SQL> select myseq.nextval from dual;
```

此外，你可以通过 SQL 脚本自动将序列号重置回某个值的任务。接下来的几行 SQL 代码展示了此技术。该代码将提示你输入序列名称和你希望序列重置到的值：

```
SQL> UNDEFINE seq_name
SQL> UNDEFINE reset_to
SQL> PROMPT "sequence name" ACCEPT '&&seq_name'
SQL> PROMPT "reset to value" ACCEPT &&reset_to
SQL> COL seq_id NEW_VALUE hold_seq_id
SQL> COL min_id NEW_VALUE hold_min_id
--
SQL> SELECT &&reset_to - &&seq_name..nextval - 1 seq_id
FROM dual;
--
SQL> SELECT &&hold_seq_id - 1 min_id FROM dual;
--
SQL> ALTER SEQUENCE &&seq_name INCREMENT BY &hold_seq_id MINVALUE &hold_min_id;
--
SQL> SELECT &&seq_name..nextval FROM dual;
--
SQL> ALTER SEQUENCE &&seq_name INCREMENT BY 1;
```

为确保序列已设置为你想要的值，从中选择 `NEXTVAL`：

```
SQL> select &&seq_name..nextval from dual;
```

当你将应用程序在各种开发、测试和生产环境之间迁移时，这种方法可能非常有用。它允许你重置序列，而无需重新颁发对象授权。

## 总结

视图、同义词和序列在 Oracle 数据库应用程序中被广泛使用。这些对象（连同表和索引）为创建复杂的应用程序提供了技术基础。

视图提供了一种创建和存储复杂多表连接查询的方法，然后这些查询可以被数据库用户和应用程序使用。视图可用于更新底层基表，也可以创建为只读以满足报告需求。

同义词（连同适当的权限）提供了一种机制，可以透明地允许用户访问属于单独模式的对象。访问同义词的用户只需知道同义词名称，而不管底层对象类型和所有者如何。这使得应用程序设计者可以无缝地将对象所有者与访问对象的用户分开。

序列生成唯一的整数，应用程序经常使用这些整数来填充主键和外键列。Oracle 保证当访问序列时，它总是会向选择的用户返回一个唯一的值。

在安装了 Oracle 二进制文件并创建了数据库和表空间之后，通常你会创建一个应用程序，该应用程序由所属用户以及相应的表、约束、索引、视图、同义词和序列组成。关于这些对象的元数据内部存储在数据字典中。数据字典广泛用于监控、故障排除和诊断问题。你必须熟练掌握从数据字典中检索信息。检索和分析数据字典信息是下一章的主题。

# 10. 数据字典基础

本书前面的章节重点介绍了诸如创建数据库、策略性实施表空间、管理用户、基本安全、表、索引和约束等主题。在这些章节中，你接触到了几个 SQL 查询，这些查询访问数据字典视图以：

*   显示数据库中有哪些用户以及他们的密码是否已过期
*   显示每个表的所有者及相关权限
*   显示各种数据库参数的设置
*   确定哪些列定义了外键约束
*   显示表空间及相关数据文件和空间使用情况

在这方面，Oracle 的数据字典是庞大而稳健的。几乎所有可以想到的关于你数据库的信息都可以检索到。数据字典存储了关于数据库物理特性、用户、对象和动态性能指标的关键信息。高级 DBA 必须具备数据字典方面的专家知识。

本章是本书的一个转折点，将其分为基础 DBA 任务和更高级的主题。现在深入探讨数据字典内部工作机制的细节是合适的。了解这些工作机制将为你理解环境、提取相关信息以及完成工作提供基础。

本章的前几节详细介绍了数据字典的架构及其创建方式。同时展示了逻辑对象与物理结构之间的关系，以及它们如何与特定的数据字典视图相关联。这些理解将作为编写 SQL 查询的基础，以提取你成为更高效、更有效 DBA 所需的信息。最后，提供了几个示例，说明 DBA 如何使用数据字典。

## 数据字典架构

如果你接手一个数据库并被要求维护和管理它，通常你会检查数据字典的内容以确定数据库的物理结构，并查看当前正在发生的事件。为此，Oracle 提供了两大类只读数据字典视图：


# Oracle 数据字典视图

*   数据库内容，例如用户、表、索引、约束和权限。这些有时被称为静态 `CDB/DBA/ALL/USER` 数据字典视图，它们基于存储在 `SYSTEM` 表空间中的内部表。此处的“静态”一词，意指这些视图中的信息仅在您对数据库进行更改（如添加用户、创建表或修改列）时才会改变。

*   数据库活动的实时视图，例如连接到数据库的用户、当前正在执行的 SQL、内存使用情况、锁和 I/O 统计信息。这些视图基于虚拟内存表，被称为动态性能视图。当数据库中发生事件时，Oracle 会持续更新这些视图中的信息。这些视图有时也称为 `V$` 或 `GV$` 视图。

这些类型的数据字典视图将在接下来的两个章节中进一步详细描述。

## 静态视图

Oracle 将一部分数据字典视图称为静态视图。这些视图基于 Oracle 内部维护的物理表。Oracle 的文档指出，这些视图之所以是静态的，是因为它们包含的数据变化速度不快（至少，与动态的 `V$` 和 `GV$` 视图相比）。

“静态”一词有时可能是一种误称。例如，`DBA_SEGMENTS` 和 `DBA_EXTENTS` 视图会随着数据库中数据量的增长和收缩而动态变化。尽管如此，Oracle 做出了静态和动态的区分，在查询数据字典时理解这种架构上的细微差别非常重要。在 Oracle Database 12c 之前，只有三个级别的静态视图：

*   `USER`
*   `ALL`
*   `DBA`

从 Oracle Database 12c 开始，当使用容器/可插拔数据库功能时，引入了第四个级别：

*   `CDB`

### 静态视图级别

`USER` 视图包含当前用户可用的信息。例如，`USER_TABLES` 视图包含当前用户所拥有的表的信息。从 `USER` 级别视图中进行选择不需要特殊权限。

下一个级别是 `ALL` 静态视图。`ALL` 视图显示当前用户有权限访问的所有对象信息。例如，`ALL_TABLES` 视图显示当前用户可以执行任何类型 DML 操作的所有数据库表。查询 `ALL` 级别视图不需要特殊权限。

接下来是 `DBA` 静态视图。`DBA` 视图包含描述数据库中所有对象的元数据（无论所有权或访问权限如何）。要访问 `DBA` 视图，必须向当前用户授予 `DBA` 角色或 `SELECT_CATALOG_ROLE`。

`CDB` 级别的视图仅在您使用可插拔数据库功能时适用。此级别提供有关容器数据库中所有可插拔数据库的信息（因此缩写为 `CDB`）。您会注意到许多静态数据字典视图和动态性能视图都有一个名为 `CON_ID` 的新列。此列唯一标识容器数据库中的每个可插拔数据库。

**提示** 关于可插拔数据库的完整讨论，请参阅第 22 章。除非另有说明，本章重点介绍 `DBA/ALL/USER` 级别的视图。请记住，如果您正在使用可插拔数据库，在报告容器数据库中所有可插拔数据库的信息时，可能需要访问 `CDB` 级别的视图。

### 静态视图的创建与结构

静态视图基于 Oracle 的内部表，例如 `USER$`、`TAB$` 和 `IND$`。如果您有权访问 `SYS` 模式，可以通过 SQL 直接查看底层表。在大多数情况下，您只需要访问基于底层内部表的静态视图即可。

数据字典表（如 `USER$`、`TAB$`、`IND$`）在执行 `CREATE DATABASE` 命令期间创建。作为创建数据库的一部分，会执行 `sql.bsq` 文件，该文件构建这些内部数据字典表。`sql.bsq` 文件通常位于 `ORACLE_HOME/rdbms/admin` 目录中；您可以通过操作系统编辑工具（如 Linux/Unix 中的 `vi` 或 Windows 中的 Notepad）查看它。

静态视图在您运行 `catalog.sql` 脚本时创建（通常，在 `CREATE DATABASE` 操作成功后运行此脚本）。`catalog.sql` 脚本位于 `ORACLE_HOME/rdbms/admin` 目录中。图 10-1 展示了创建静态数据字典视图的过程。

![../images/214899_3_En_10_Chapter/214899_3_En_10_Fig1_HTML.jpg](img/214899_3_En_10_Fig1_HTML.jpg)
*图 10-1 创建静态数据字典视图*

### 查看创建脚本

您可以通过查询 `DBA_VIEWS` 的 `TEXT` 列来查看静态视图的创建脚本；例如，

```sql
SQL> set long 5000
SQL> select text from dba_views where view_name='DBA_VIEWS';
```

输出如下：

```sql
SQL> select u.name, o.name, v.textlength, v.text, t.typetextlength, t.typetext,
t.oidtextlength, t.oidtext, t.typeowner, t.typename,
decode(bitand(v.property, 134217728), 134217728,
(select sv.name from superobj$ h, "_CURRENT_EDITION_OBJ" sv
where h.subobj# = o.obj# and h.superobj# = sv.obj#), null),
decode(bitand(v.property, 32), 32, 'Y', 'N'),
decode(bitand(v.property, 16384), 16384, 'Y', 'N'),
decode(bitand(v.property/4294967296, 134217728), 134217728, 'Y', 'N'),
decode(bitand(o.flags,8),8,'CURRENT_USER','DEFINER')
from sys."_CURRENT_EDITION_OBJ" o, sys.view$ v, sys.user$ u, sys.typed_view$ t
where o.obj# = v.obj#
and o.obj# = t.obj#(+)
and o.owner# = u.user#
```

**注意** 如果您手动创建数据库（不使用 `dbca` 实用程序），在运行 `catalog.sql` 和 `catproc.sql` 脚本时，必须以 `SYS` 模式连接。`SYS` 模式是数据字典中所有对象的所有者。

## 动态性能视图

动态性能数据字典视图通常被称为 `V$` 和 `GV$` 视图。这些视图由 Oracle 不断更新，反映了实例和数据库的当前状况。动态视图对于诊断实时性能问题至关重要。

`V$` 和 `GV$` 视图间接基于底层的 `X$` 表，这些是启动 Oracle 实例时实例化的内部内存结构。一些 `V$` 视图在 Oracle 实例启动后立即可用。例如，`V$PARAMETER` 在执行 `STARTUP NOMOUNT` 命令后就包含有意义的数据，并且不需要数据库处于加载或打开状态。其他动态视图（如 `V$CONTROLFILE`）依赖于控制文件中的信息，因此只有在数据库加载后才包含重要信息。一些 `V$` 视图（如 `V$BH`）提供内核处理信息，因此仅在数据库打开后才有用的结果。

### 动态视图结构

在顶层，`V$` 视图实际上是同义词，指向底层的 `SYS.V_$` 视图。在下一层，`SYS.V_$` 对象是在另一层 `SYS.V$` 视图之上创建的视图。`SYS.V$` 视图又基于 `SYS.GV$` 视图。在底层，`SYS.GV$` 视图基于 `X$` 内存结构。



顶级 `V$` 同义词和 `SYS.V_$` 视图是在你运行 `catalog.sql` 脚本时创建的，你通常会在数据库初始创建后执行此脚本。图 10-2 展示了创建 `V$` 动态性能视图的过程。

![../images/214899_3_En_10_Chapter/214899_3_En_10_Fig2_HTML.jpg](img/214899_3_En_10_Fig2_HTML.jpg)

**图 10-2** 创建 `V$` 动态性能数据字典视图

通过最顶层的同义词访问 `V$` 视图通常足以满足动态性能信息的需求。在极少数情况下，你可能希望查询那些可能无法通过 `V$` 视图获取的内部信息。在这些情况下，理解底层的 `X$` 表至关重要。如果你使用 Oracle RAC，则应该熟悉 `GV$` 全局视图。这些视图提供关于集群中所有实例的全局动态性能信息（而 `V$` 视图是实例特定的）。`GV$` 视图包含一个 `INST_ID` 列，用于在集群环境中标识特定实例。

你可以通过查询 `V$FIXED_VIEW_DEFINITION` 视图的 `VIEW_DEFINITION` 列来显示 `V$` 和 `GV$` 视图的定义。例如，此查询显示 `V$CONTROLFILE` 的定义：

```
SQL> select view_definition
from v$fixed_view_definition
where view_name='V$CONTROLFILE';
```

输出如下：

```
select STATUS, NAME, IS_RECOVERY_DEST_FILE, BLOCK_SIZE, FILE_SIZE_BLKS,
CON_ID from GV$CONTROLFILE where inst_id = USERENV('Instance')
```

## 元数据的另一种视角

DBA 通常会面临以下类型的数据库问题：

*   向表中插入数据失败，因为表空间无法扩展。
*   数据库拒绝连接，因为超过了最大会话数。
*   应用程序挂起，显然是由于某种锁问题。
*   PL/SQL 语句执行失败，并出现内存错误。
*   RMAN 备份连续两天未成功。
*   用户尝试更新记录，但抛出唯一键约束冲突。
*   一条 SQL 语句的运行时间比正常情况长了数小时。
*   应用程序用户反映性能似乎很慢，数据库肯定有问题。

上述列表是 DBA 日常遇到的典型问题的一小部分样本。要能够高效地诊断和处理这类问题，需要具备一定的知识。其中的一个基础部分就是理解 Oracle 的物理结构及其对应的逻辑组件。

例如，如果一个表因为表空间已满而无法扩展，你依赖哪些知识来解决这个问题？你需要理解，当创建数据库时，它包含多个称为表空间的逻辑空间容器。每个表空间由一个或多个物理数据文件组成。每个数据文件由许多操作系统块组成。每个表对应一个段，每个段包含一个或多个区。当一个段需要空间时，它会在物理数据文件中分配额外的区。

一旦理解了所涉及的逻辑和物理概念，你就会直观地查看数据字典视图，如 `DBA_TABLES`、`DBA_SEGMENTS`、`DBA_TABLESPACES` 和 `DBA_DATA_FILES`，以定位问题并根据需要添加空间。在各种各样的故障排除场景中，你对各种逻辑和物理结构之间关系的理解，将使你能够专注于查询那些能帮助你快速解决当前问题的视图。

为此，请查看图 10-3。该图描述了 Oracle 数据库中逻辑和物理结构之间的关系。圆角矩形代表逻辑结构，而尖角矩形代表物理文件。

![../images/214899_3_En_10_Chapter/214899_3_En_10_Fig3_HTML.jpg](img/214899_3_En_10_Fig3_HTML.jpg)

**图 10-3** Oracle 数据库逻辑与物理结构关系

> **提示：** 逻辑对象只有在数据库启动后才能从 SQL 中查看。相比之下，即使实例未启动，也可以通过操作系统实用程序查看物理对象。

图 10-3 并未展示 Oracle 数据库所有逻辑和物理方面的全部关系。相反，它侧重于你日常最有可能遇到的组件。这个基础关系图为利用 Oracle 的数据字典基础设施奠定了基础。

在脑海中保持图 10-3 的图像；现在，将其与图 10-4 并列思考。

![../images/214899_3_En_10_Chapter/214899_3_En_10_Fig4_HTML.jpg](img/214899_3_En_10_Fig4_HTML.jpg)

**图 10-4** 常用数据字典视图的关系

瞧，这些数据字典视图与 Oracle 数据库几乎所有的逻辑和物理元素都紧密对应。图 10-4 并未展示每一个数据字典视图。实际上，该图只是触及了皮毛。然而，这张图确实为你提供了一个坚实的基础，让你在此基础上理解如何利用数据字典视图来获取工作所需的数据。

该图确实展示了视图之间的关系，但并未指明连接视图时应使用哪些列。你必须描述这些表，并对接如何连接视图做出合理的推测。例如，假设你想显示与非本地管理的表空间关联的数据文件。这需要将 `DBA_TABLESPACES` 与 `DBA_DATA_FILES` 连接起来。如果你检查这两个视图，会注意到每个视图都包含一个 `TABLESPACE_NAME` 列，这允许你编写如下查询：

```
SQL> select a.tablespace_name, a.extent_management, b.file_name
from dba_tablespaces a,
dba_data_files  b
where a.tablespace_name = b.tablespace_name
and a.extent_management != 'LOCAL';
```

通常，如何连接视图是相当明显的。使用该图作为指南，告诉你从哪里开始查找信息，以及如何编写 SQL 查询来提供问题的答案，并扩展你对 Oracle 内部架构和工作原理的知识。这将你的问题解决技能锚定在坚实的基础上。一旦你牢固理解了 Oracle 逻辑和物理组件之间的关系，以及这与数据字典的关联，你就能自信地处理任何类型的数据库问题。

> **注意：** 有数千个 `CDB/DBA/ALL/USER` 静态视图和超过 700 个 `V$` 动态性能视图。

## 数据字典的几种创造性用法

在本书的几乎每一章中，你都会找到几个如何利用数据字典来更好地理解概念和解决问题的 SQL 示例。话虽如此，展示一些 DBA 如何利用数据字典的非主流例子还是值得的。接下来的几节内容正是如此。请记住，这仅仅是冰山一角：DBA 们采用无数的查询和技术来提取和使用数据字典信息。

### 可推导的文档

有时，如果你在排除故障且压力山大，你需要快速从数据字典中提取信息来帮助解决问题。然而，你可能不知道数据字典视图的确切名称或其相关列。如果你像我一样，不可能把所有数据字典视图名和列名都记在脑子里。此外，我使用从版本 8 到 12c 的数据库，很难跟踪哪个特定的视图可能在 Oracle 的某个发行版中可用。



## 使用数据字典视图

书籍和海报可以提供信息，但如果你找不到完全符合需求的内容，可以使用数据字典本身包含的文档。你可以从三个视图中进行查询，特别是：

*   `DBA_OBJECTS`
*   `DICTIONARY`
*   `DICT_COLUMNS`

如果你大致知道要从中选择信息的视图名称，可以先从`DBA_OBJECTS`开始查询。例如，如果你在排查与物化视图相关的问题，并且不记得与物化视图关联的数据字典视图的确切名称，你可以这样做：

```sql
SQL> select object_name
from dba_objects
where object_name like '%MV%'
and owner='SYS';
```

这可能足以让你找到大致方向。但通常你需要关于每个视图的更多信息。这时，`DICTIONARY`和`DICT_COLUMNS`视图就非常宝贵了。`DICTIONARY`视图存储数据字典视图的名称。它有两个列：

```sql
SQL> desc dictionary
Name                                      Null?    Type
----------------------------------------- -------- ------------------------
TABLE_NAME                                         VARCHAR2(30)
COMMENTS                                           VARCHAR2(4000)
```

例如，假设你正在排查与物化视图相关的问题，并希望确定与物化视图功能相关的数据字典视图的名称。你可以运行如下查询：

```sql
SQL> select table_name, comments
from dictionary
where table_name like '%MV%';
```

以下是一部分输出：

```
TABLE_NAME             COMMENTS

DBA_MVIEW_LOGS         All materialized view logs in the database
DBA_MVIEWS             All materialized views in the database
DBA_MVIEW_ANALYSIS     Description of the materialized views accessible to dba
DBA_MVIEW_COMMENTS     Comments on all materialized views in the database
```

通过这种方式，你可以快速确定需要访问哪个视图。如果你想要关于该视图的更多信息，可以对其进行描述；例如：

```sql
SQL> desc dba_mviews
```

如果这没有提供足够的关于列名的信息，你可以查询`DICT_COLUMNS`视图。该视图提供了数据字典视图各列的注释；例如：

```sql
SQL> select column_name, comments
from dict_columns
where table_name='DBA_MVIEWS';
```

以下是部分输出：

```
COLUMN_NAME            COMMENTS
---------------------- ---------------------------------------------
OWNER                  Owner of the materialized view
MVIEW_NAME             Name of the materialized view
CONTAINER_NAME         Name of the materialized view container table
QUERY                  The defining query that the materialized view instantiates
```

通过这种方式，你可以生成并查看关于大多数数据字典对象的文档。该方法允许你快速识别在排查问题时可能有帮助的适当视图和列。

## 显示用户信息

你可能会发现自己处在一个包含数十台不同服务器上数百个数据库的环境中。在这种情况下，你希望确保不会运行错误的命令或连接到错误的数据库，或两者兼而有之。在执行 DBA 任务时，谨慎的做法是验证你是否以正确的账户连接到正确的数据库。你可以运行以下类型的 SQL 命令来验证当前连接的用户和数据库信息：

```sql
SQL> show user;
SQL> select * from user_users;
SQL> select name from v$database;
SQL> select instance_name, host_name from v$instance;
```

正如第 3 章所示，保持对环境感知的一种有效方法是通过`login.sql`脚本自动设置你的 SQL*Plus 提示符，以显示用户和实例信息。以下示例手动设置了 SQL 提示符：

```sql
SQL> set sqlprompt '&_USER.@&_CONNECT_IDENTIFIER.> '
```

现在 SQL 提示符看起来像这样：

```
SYS@O18C>
```

你还可以使用`SYS_CONTEXT`内置 SQL 函数从数据字典中提取有关当前连接会话的详细信息。该函数的通用语法如下：

```sql
SYS_CONTEXT('','',[length])
```

此示例显示用户、身份验证方法、主机和实例：

```sql
SYS@O18C> select
sys_context('USERENV','CURRENT_USER') usr
,sys_context('USERENV','AUTHENTICATION_METHOD') auth_mth
,sys_context('USERENV','HOST') host
,sys_context('USERENV','INSTANCE_NAME') inst
from dual;
```

`USERENV`是一个内置的 Oracle 命名空间。当与`SYS_CONTEXT`函数一起使用时，`USERENV`命名空间提供了超过 50 个可用参数。表 10-1 描述了一些更有用的参数。有关参数的完整列表，请参阅 Oracle SQL 语言参考指南，该指南可以从 Oracle 网站的技术网络区域（[`http://otn.oracle.com`](http://otn.oracle.com)）免费下载。

**表 10-1 与 `SYS_CONTEXT` 一起使用的有用 `USERENV` 参数**

| 参数名 | 描述 |
| :--- | :--- |
| `AUTHENTICATED_IDENTITY` | 身份验证中使用的身份 |
| `AUTHENTICATION_METHOD` | 身份验证的方法 |
| `CDB_NAME` | 返回 CDB 的名称；否则返回 null |
| `CLIENT_IDENTIFIER` | 返回由应用程序设置的标识符 |
| `CLIENT_INFO` | 用户会话信息 |
| `CON_ID` | 容器标识符 |
| `CON_NAME` | 容器名称 |
| `CURRENT_USER` | 当前活动会话的用户名 |
| `DB_NAME` | 由`DB_NAME`初始化参数指定的名称 |
| `DB_UNIQUE_NAME` | 由`DB_UNIQUE_NAME`初始化参数指定的名称 |
| `HOST` | 客户端发起数据库连接所在机器的主机名 |
| `INSTANCE_NAME` | 实例名称 |
| `IP_ADDRESS` | 客户端发起数据库连接所在机器的 IP 地址 |
| `ISDBA` | 如果用户通过操作系统或密码文件以 DBA 权限通过身份验证，则为`TRUE` |
| `NLS_DATE_FORMAT` | 会话的日期格式 |
| `OS_USER` | 客户端发起数据库连接所在机器的操作系统用户 |
| `SERVER_HOST` | 运行数据库实例的机器的主机名 |
| `SERVICE_NAME` | 连接的服务名称 |
| `SID` | 会话标识符 |
| `TERMINAL` | 客户端终端的操作系统标识符 |

## 确定环境的详细信息

有时，在通过各种开发、测试、测试版和生产环境部署代码时，提示你是否处于正确的环境中是很有帮助的。完成此操作的技术需要两个文件：`answer_yes.sql`和`answer_no.sql`。以下是`answer_yes.sql`的内容：

```sql
-- answer_yes.sql
PROMPT
PROMPT Continuing...
```

而`answer_no.sql`的内容如下：

```sql
-- answer_no.sql
PROMPT
PROMPT Quitting and discarding changes...
ROLLBACK;
EXIT;
```

现在，你可以将以下代码插入到部署脚本的开头部分；该代码将提示你是否处于正确的环境以及是否希望继续：

```sql
WHENEVER SQLERROR EXIT FAILURE ROLLBACK;
WHENEVER OSERROR EXIT FAILURE ROLLBACK;
select host_name from v$instance;
select name as db_name from v$database;
SHOW user;
SET ECHO OFF;
PROMPT
ACCEPT answer PROMPT 'Correct environment? Enter yes to continue: '
@@answer_&answer..sql
```

如果你输入`yes`，则会执行`answer_yes.sql`脚本，你将可以继续运行你调用的任何其他脚本。如果你输入`no`，则会运行`answer_no.sql`脚本，你将退出 SQL*Plus 并返回到操作系统提示符。如果你直接按回车键而没有输入任何内容，你也会退出并返回到操作系统提示符。

## 显示表行数



# 统计表行数

当你调查性能或空间问题时，显示每个表的行数很有用。要手动计算行数，你需要为你拥有的每个表编写如下查询：

```sql
SQL> select count(*) from ;
```

手动编写 SQL 耗时且容易出错。在这种情况下，使用 SQL 来生成解决问题所需的 SQL 效率更高。为此，下一个示例根据 `DBA_TABLES` 视图中的信息动态选择所需文本。生成动态 SQL 的输出文件被假脱机。以拥有 `DBA` 权限的用户身份运行以下 SQL 代码。请注意，此脚本包含 SQL*Plus 特定命令，例如 `UNDEFINE` 和 `SPOOL`。脚本每次都会提示你输入用户名：

```sql
UNDEFINE user
SPOOL tabcount_&&user..sql
SET LINESIZE 132 PAGESIZE 0 TRIMSPO OFF VERIFY OFF FEED OFF TERM OFF
SELECT
'SELECT RPAD(' || "" || table_name || "" ||',30)'
|| ',' || ' COUNT(*) FROM &&user..' || table_name || ';'
FROM dba_tables
WHERE owner = UPPER('&&user')
ORDER BY 1;
SPO OFF;
SET TERM ON
@@tabcount_&&user..sql
SET VERIFY ON FEED ON
```

此代码生成一个名为 `tabcount_<user>.sql` 的文件，其中包含从指定模式中所有表选择行数的 SQL 语句。如果你提供给脚本的用户名是 `INVUSER`，那么你可以手动运行生成的脚本，如下所示：

```sql
SQL> @tabcount_invuser.sql
```

请记住，如果表行数很高，此脚本可能需要很长时间运行（几分钟）。

开发人员和 DBA 经常使用 SQL 来生成 SQL 语句。当你需要将相同的 SQL 过程（重复地）应用于许多不同的对象（例如模式中的所有表）时，这是一项有用的技术。如果你无法访问 `DBA` 级别的视图，可以查询 `USER_TABLES` 视图；例如：

```sql
SPO tabcount.sql
SET LINESIZE 132 PAGESIZE 0 TRIMSPO OFF VERIFY OFF FEED OFF TERM OFF
SELECT
'SELECT RPAD(' || "" || table_name || "" ||',30)'
|| ',' || ' COUNT(*) FROM ' || table_name || ';'
FROM user_tables
ORDER BY 1;
SPO OFF;
SET TERM ON
@@tabcount.sql
SET VERIFY ON FEED ON
```

如果你有准确的统计信息，可以查询 `CDB/DBA/ALL/USER_TABLES` 视图的 `NUM_ROWS` 列。如果定期生成统计信息，此列通常具有接近的行数。以下查询从 `USER_TABLES` 视图中选择 `NUM_ROWS`：

```sql
SQL> select table_name, num_rows from user_tables;
```

最后一点：如果你有分区表并想按分区显示行数，请使用接下来的几行 SQL 和 PL/SQL 代码来生成所需的 SQL：

```sql
SQL> UNDEFINE user
SQL> SET SERVEROUT ON SIZE 1000000 VERIFY OFF
SQL> SPO part_count_&&user..txt
SQL> DECLARE
counter  NUMBER;
sql_stmt VARCHAR2(1000);
CURSOR c1 IS
SELECT table_name, partition_name
FROM dba_tab_partitions
WHERE table_owner = UPPER('&&user');
BEGIN
FOR r1 IN c1 LOOP
sql_stmt := 'SELECT COUNT(*) FROM &&user..' || r1.table_name
||' PARTITION ( '||r1.partition_name ||' )';
EXECUTE IMMEDIATE sql_stmt INTO counter;
DBMS_OUTPUT.PUT_LINE(RPAD(r1.table_name
||'('||r1.partition_name||')',30) ||' '||TO_CHAR(counter));
END LOOP;
END;
/
SPO OFF
```

# 手动生成统计信息

如果你想为表生成统计信息，请使用 `DBMS_STATS` 包。此示例为用户和表生成统计信息：

```sql
SQL> exec dbms_stats.gather_table_stats(ownname=>'MV_MAINT',-
tabname=>'F_SALES',-
cascade=>true,estimate_percent=>20,degree=>4);
```

你可以使用以下代码为用户的所有对象生成统计信息：

```sql
SQL> exec dbms_stats.gather_schema_stats(ownname => 'MV_MAINT',-
estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,-
degree => DBMS_STATS.AUTO_DEGREE,-
cascade => true);
```

前面的代码指示 Oracle 使用 `ESTIMATE_PERCENT` 参数，通过 `DBMS_STATS.AUTO_SAMPLE_SIZE` 估算要采样的表的百分比。Oracle 还使用 `DEGREE` 参数设置 `DBMS_STATS.AUTO_DEGREE` 选择适当的并行度。`CASCADE` 参数指示 Oracle 为索引生成统计信息。请记住，Oracle 可能不会选择最佳的自动样本大小。Oracle 可能选择 10%，但你可能有设置较低百分比（例如 5%）的经验，并且知道这是可接受的数字。在这些情况下，不要使用 `AUTO_SAMPLE_SIZE`；而是明确提供一个数字。

# 显示主键和外键关系

有时在诊断约束问题时，显示有关外键约束关联的主键约束的数据字典信息很有用。例如，也许你尝试向子表插入数据，但抛出错误指出父键不存在，而你想显示有关父键约束的更多信息。

以下脚本查询 `DBA_CONSTRAINTS` 数据字典视图以确定与子外键约束相关的父主键约束。你需要向脚本提供表的所有者和你希望显示主键约束的子表：

```sql
SQL> select
a.constraint_type cons_type
,a.table_name      child_table
,a.constraint_name child_cons
,b.table_name      parent_table
,b.constraint_name parent_cons
,b.constraint_type cons_type
from dba_constraints a
,dba_constraints b
where a.owner    = upper('&owner')
and a.table_name = upper('&table_name')
and a.constraint_type = 'R'
and a.r_owner = b.owner
and a.r_constraint_name = b.constraint_name;
```

前面的脚本提示你输入两个 SQL*Plus 的 `&` 变量（`OWNER`，`TABLE_NAME`）；如果你不使用 SQL*Plus，那么在运行脚本前可能需要使用适当的值修改脚本。

以下输出显示有两个外键约束。它还显示了父表主键约束：

```
C CHILD_TABLE     CHILD_CONS          PARENT_TABLE    PARENT_CONS         C
- --------------- ------------------- --------------- ------------------- -
R REG_COMPANIES   REG_COMPANIES_FK2   D_COMPANIES     D_COMPANIES_PK      P
R REG_COMPANIES   REG_COMPANIES_FK1   CLUSTER_BUCKETS CLUSTER_BUCKETS_PK  P
```

当 `DBA/ALL/USER_CONSTRAINTS` 的 `CONSTRAINT_TYPE` 列包含 `R` 值时，这表示该行描述的是引用完整性约束，意味着子表约束引用了主键约束。你通过连接到同一张表两次来检索主键约束信息。子约束列（`R_OWNER`，`R_CONSTRAINT_NAME`）与 `DBA_CONSTRAINTS` 视图中包含主键信息的另一行匹配。

你也可以对本节前面的查询做反向操作；对于主键约束，你想找到与之关联的外键列（如果有）。下一个脚本获取主键记录，并查看它是否有任何约束类型为 `R` 的子记录。运行此脚本时，系统会提示你输入主键表的所有者和名称：

```sql
SQL> select
b.table_name        primary_key_table
,a.table_name        fk_child_table
,a.constraint_name   fk_child_table_constraint
from dba_constraints a
,dba_constraints b
where a.r_constraint_name = b.constraint_name
and   a.r_owner           = b.owner
and   a.constraint_type   = 'R'
and   b.owner             = upper('&table_owner')
and   b.table_name        = upper('&table_name');
```

以下是一些示例输出：


# 显示对象依赖关系

```
主键表          外键子表          外键子表约束
-------------------- -------------------- ------------------------------
CLUSTER_BUCKETS      CB_AD_ASSOC          CB_AD_ASSOC_FK1
CLUSTER_BUCKETS      CLUSTER_CONTACTS     CLUSTER_CONTACTS_FK1
CLUSTER_BUCKETS      CLUSTER_NOTES        CLUSTER_NOTES_FK1
```

假设你需要 `drop` 一张表，但在执行 `drop` 操作前，你想显示任何依赖于它的对象。例如，你可能有一张表，其上存在同义词、视图、物化视图、函数、过程和触发器等依赖对象。在进行更改前，你想审查还有哪些其他对象依赖于该表。你可以使用 `DBA_DEPENDENCIES` 数据字典视图来显示对象依赖关系。下面的查询会提示你输入用户名和对象名：

```
SQL> select '+' || lpad(' ',level+2) || type || ' ' || owner || '.' || name  dep_tree
from dba_dependencies
connect by prior owner = referenced_owner and prior name = referenced_name
and prior type = referenced_type
start with referenced_owner = upper('&object_owner')
and referenced_name = upper('&object_name')
and owner is not null;
```

在输出中，列出的每个对象都对你输入的对象存在依赖关系。行会被缩进，以显示一个对象对前一行所列对象的依赖性：

```
DEP_TREE

+   TRIGGER STAR2.D_COMPANIES_BU_TR1
+   MATERIALIZED VIEW CIA.CB_RAD_COUNTS
+   SYNONYM STAR1.D_COMPANIES
+    SYNONYM CIA.D_COMPANIES
+     MATERIALIZED VIEW CIA.CB_RAD_COUNTS
```

在此示例中，正在分析的对象是一张名为 `D_COMPANIES` 的表。有多个同义词、物化视图和一个触发器依赖于这张表。例如，由 `CIA` 拥有的物化视图 `CB_RAD_COUNTS` 依赖于由 `CIA` 拥有的同义词 `D_COMPANIES`，而该同义词又依赖于由 `STAR1` 拥有的 `D_COMPANIES` 同义词。`DBA_DEPENDENCIES` 视图包含了 `OWNER`、`NAME` 和 `TYPE` 列与其引用的列名 `REFERENCED_OWNER`、`REFERENCED_NAME` 和 `REFERENCED_TYPE` 之间的层次关系。Oracle 提供了许多结构来执行层次查询。例如，`START WITH` 和 `CONNECT BY` 允许你在树中标识一个起点，并沿着层次关系向上或向下遍历。

本节前面的 SQL 查询仅针对一个对象进行操作。如果你想检查模式中的每个对象，可以使用 SQL 来生成 SQL，以创建脚本显示模式中所有对象的依赖关系。下一个示例中的代码就是这么做的。为了格式化和输出，该代码使用了 SQL*Plus 特有的构造，例如设置页面大小和行大小，以及假脱机输出：

```
SQL> UNDEFINE owner
SQL> SET LINESIZE 132 PAGESIZE 0 VERIFY OFF FEEDBACK OFF TIMING OFF
SQL> SPO dep_dyn_&&owner..sql
SQL> SELECT 'SPO dep_dyn_&&owner..txt' FROM DUAL;
--
SELECT
'PROMPT ' || '_____________________________'|| CHR(10) ||
'PROMPT ' || object_type || ': ' || object_name || CHR(10) ||
'SELECT ' || "" || '+' || "" || ' ' ||  '|| LPAD(' || "" || ' '
|| "" || ',level+3)' || CHR(10) || ' || type || ' || "" || ' ' || "" ||
' || owner || ' || "" || '.' || "" || ' || name' || CHR(10) ||
' FROM dba_dependencies ' || CHR(10) ||
' CONNECT BY PRIOR owner = referenced_owner AND prior name = referenced_name '
|| CHR(10) ||
' AND prior type = referenced_type ' || CHR(10) ||
' START WITH referenced_owner = ' || "" || UPPER('&&owner') || "" || CHR(10) ||
' AND referenced_name = ' || "" || object_name || "" || CHR(10) ||
' AND owner IS NOT NULL;'
FROM dba_objects
WHERE owner = UPPER('&&owner')
AND object_type NOT IN ('INDEX','INDEX PARTITION','TABLE PARTITION');
--
SELECT 'SPO OFF' FROM dual;
SPO OFF
SET VERIFY ON LINESIZE 80 FEEDBACK ON
```

现在你应该在运行脚本的同一目录下，得到一个名为 `dep_dyn_<owner>.sql` 的脚本。该脚本包含显示你所输入的属主下对象依赖关系所需的所有 SQL。运行该脚本来显示对象依赖关系。在此示例中，属主是 `CIA`：

```
SQL> @dep_dyn_cia.sql
```

当脚本运行时，它会假脱机一个格式为 `dep_dyn_<owner>.txt` 的文件。你可以使用操作系统编辑器打开该文本文件查看其内容。以下是此示例的输出样本：

```
TABLE: DOMAIN_NAMES
+    FUNCTION STAR2.GET_DERIVED_COMPANY
+    TRIGGER STAR2.DOMAIN_NAMES_BU_TR1
+    SYNONYM CIA_APP.DOMAIN_NAMES
```

此输出显示表 `DOMAIN_NAMES` 有三个依赖于它的对象：一个函数、一个触发器和一个同义词。

# DUAL 表

`DUAL` 表是数据字典的一部分。该表包含一行一列，当你想要返回恰好一行，且无需从特定表中检索数据时非常有用。换句话说，你只是想返回一个值。例如，你可以执行算术运算，如下所示：

```
SQL> select 34*.15 from dual;
34*.15

5.1
```

其他常见用法包括从 `DUAL` 表中选择以显示当前日期，或在 SQL 脚本中显示一些文本。

## 总结

有时，你会接手一个已运行多年的旧数据库，管理和维护它的任务就落到了你身上。在某些情况下，你没有任何关于数据库中用户和对象的文档。即使有文档，它也可能不准确或已过时。在这些情况下，数据字典很快就会成为你的文档来源。你可以使用它来提取用户信息、数据库物理结构、安全信息、对象和属主、当前连接的用户等等。Oracle 在数据字典中提供了静态和动态视图。静态视图包含数据库中对象的信息。你可以使用这些视图来确定哪些表占用空间最多、包含的行数最多、分配的区最多等等。动态性能视图提供了数据库中当前正在发生的事件的实时窗口。这些视图提供有关当前连接的用户、正在执行的 SQL、资源消耗位置等信息。DBA 广泛使用这些视图来监控和解决性能问题。本书现在将注意力转向专门的 Oracle 特性，例如大对象、分区、数据泵和外部表。这些主题将在接下来的几章中介绍。

# 11. 大对象

组织经常需要处理需要存储并由业务用户查看的大型文件。通常，LOB 是一种适合存储大型非结构化数据的数据类型，例如文本、日志、图像、视频、声音和空间数据。Oracle 支持以下类型的 LOB：

*   字符大对象 (`CLOB`)
*   国家字符大对象 (`NCLOB`)
*   二进制大对象 (`BLOB`)
*   二进制文件 (`BFILE`)


# Oracle 大对象（LOB）数据类型详解

## 废弃的数据类型

在 Oracle 8 之前，`LONG` 和 `LONG RAW` 数据类型是在列中存储大量数据的唯一选择。您应该不再使用这些数据类型。我提到 `LONG` 和 `LONG RAW` 的唯一原因是许多遗留应用程序（例如 Oracle 的数据字典）仍在使用它们。否则，您应该使用 `CLOB` 代替 `LONG`，使用 `BLOB` 代替 `LONG RAW`。

同时，不要混淆 `RAW` 数据类型和 `LONG RAW`。`RAW` 数据类型用于存储少量二进制数据。而 `LONG RAW` 数据类型已被废弃超过十年了。

另一个注意事项：不要不必要地使用 LOB 数据类型。例如，对于字符数据，如果您的应用程序需要少于 32,000 个单字节字符，请使用 `VARCHAR2` 数据类型（而不是 `CLOB`）。对于二进制数据，如果您处理的二进制数据少于 32,000 字节，请使用 `RAW` 数据类型（而不是 `BLOB`）。如果您仍然不确定您的应用程序需要哪种数据类型，请参阅第 7 章以了解 Oracle 数据类型的适当用途。

## 描述 LOB 类型

在深入探讨 LOB 的实现细节之前，有必要先回顾每种 LOB 数据类型及其适当用途。之后，将提供创建和使用 LOB 的示例以及您应理解的相关功能。

### 内部 LOB

自早期 Oracle 版本以来，使用 `CLOB`、`NCLOB`、`BLOB` 和 `BFILE` 数据类型在数据库中存储大文件的能力得到了极大改进。这些附加的 LOB 数据类型让您可以存储更多数据，并具有更强大的功能。

表 11-1 总结了可用的 Oracle LOB 类型及其描述。

**表 11-1 Oracle 大对象数据类型**

| 数据类型 | 描述 | 最大大小 |
| :--- | :--- | :--- |
| `CLOB` | 字符大对象，用于存储字符文档，如大型文本文件、日志文件、XML 文件等 | (4GB–1) * 块大小 |
| `NCLOB` | 国家字符大对象；以国家字符集格式存储数据；支持宽度可变的字符 | (4GB–1) * 块大小 |
| `BLOB` | 二进制大对象，用于存储非结构化位流数据（图像、视频等） | (4GB–1) * 块大小 |
| `BFILE` | 二进制文件大对象，存储在数据库外的文件系统上；只读 | 2⁶⁴–1 字节（操作系统可能强加小于此值的大小限制） |

- `CLOB` 用于存储 JSON、XML、文本和日志文件等。`NCLOB` 的处理方式与 `CLOB` 相同，但可以包含数据库的多字节国家字符集中的字符。
- `BLOB` 不是人类可读的。`BLOB` 的典型用途是电子表格、文字处理文档、图像以及音频和视频数据。
- `CLOB`、`NCLOB` 和 `BLOB` 被称为**内部 LOB**。这是因为这些数据类型存储在 Oracle 数据库的数据文件内部。内部 LOB 参与事务，并受 Oracle 数据库安全以及其备份和恢复功能的保护。

### 外部 LOB

`BFILE` 被称为**外部 LOB**。`BFILE` 列存储指向数据库外部操作系统上文件的指针。您可以将 `BFILE` 视为一种为数据库外部操作系统文件系统上的大型二进制文件提供只读访问的机制。

有时会出现这样的问题：应该使用 `BLOB` 还是 `BFILE`？`BLOB` 参与数据库事务，可以由 Oracle 备份、恢复和还原。`BFILE` 不参与数据库事务，是只读的，并且不受任何 Oracle 安全、备份和恢复、复制或灾难恢复机制的保护。`BFILE` 更适用于那些在应用程序运行期间为只读且不会更改的大型二进制文件。例如，您可能有数据库应用程序引用的大型二进制视频文件。在这种场景下，业务确定您不需要在应用程序实际只需要一个指针（存储在数据库中）指向磁盘上大型文件位置的情况下，创建和维护一个 500TB 的数据库。

## 图解 LOB 定位器、索引和块

内部 LOB（`CLOB`、`NCLOB`、`BLOB`）以称为**块**的片段存储数据。块是 LOB 的最小分配单元，由一个或多个数据库块组成。**LOB 定位器**存储在包含 LOB 列的行中。LOB 定位器指向一个 **LOB 索引**。LOB 索引存储有关 LOB 块位置的信息。当查询表时，数据库使用 LOB 定位器和关联的 LOB 索引来定位适当的 LOB 块。

图 11-1 显示了表、行、LOB 定位器以及 LOB 定位器关联的索引和块之间的关系。

![表、行、LOB 定位器、LOB 索引和 LOB 段的关系](img/214899_3_En_11_Fig1_HTML.png)

**图 11-1 表、行、LOB 定位器、LOB 索引和 LOB 段的关系**

`BFILE` 的 LOB 定位器存储操作系统上的目录路径和文件名。图 11-2 显示了一个引用操作系统上文件的 `BFILE` LOB 定位器。

![BFILE LOB 定位器包含在 OS 上定位文件的信息](img/214899_3_En_11_Fig2_HTML.png)

**图 11-2 `BFILE` LOB 定位器包含在操作系统上定位文件的信息**

> **注意**
> `DBMS_LOB` 包通过 LOB 定位器执行对 LOB 的操作。

## 区分 BasicFiles 和 SecureFiles

Oracle 对 LOB 做了几项重大改进。Oracle 现在区分两种不同类型的底层 LOB 架构：
- BasicFiles
- SecureFiles

以下部分将讨论这两种 LOB 架构。

### BasicFiles

BasicFiles 是 Oracle 给予 Oracle Database 11g 之前可用的 LOB 架构的名称。理解 BasicFiles LOB 仍然很重要，因为许多使用 Oracle 的场所使用的版本不支持 SecureFiles。请注意，在 Oracle Database 11g 中，默认的 LOB 类型仍然是 BasicFiles。然而，从 Oracle Database 11g 开始，默认的 LOB 类型是 SecureFiles，并应作为存储 LOB 的方式使用。

## SecureFiles

SecureFiles 是推荐使用的 LOB 架构选项。它包含以下增强功能（相对于 BasicFiles LOB）：
- 加密（需要 Oracle Advanced Security 选项）
- 压缩（需要 Oracle Advanced Compression 选项）


## 重用现有表空间

*   去重（需要 **Oracle Advanced Compression** 选项）

SecureFiles 加密允许你透明地加密 LOB 数据（就像其他数据类型一样）。压缩功能可以显著节省空间。去重功能会消除重复的 LOB，否则它们可能会被存储多次。

在使用 SecureFiles 之前，你需要做一些小规模的规划。具体来说，使用 SecureFiles 需要满足以下条件：

*   SecureFiles LOB 必须存储在使用 ASSM 的表空间中。
*   初始化参数 `DB_SECUREFILE` 控制是否可以使用 SecureFiles 文件，并定义了数据库的默认 LOB 架构。

SecureFiles LOB 必须创建在使用 ASSM 的表空间中。要创建启用了 ASSM 的表空间，请指定 `SEGMENT SPACE MANAGEMENT AUTO` 子句；例如：

```sql
SQL> create tablespace lob_data
datafile '/u01/dbfile/o18c/lob_data01.dbf'
size 1000m
extent management local
uniform size 1m
segment space management auto;
```

如果你已有表空间，可以通过查询 `DBA_TABLESPACES` 视图来验证 ASSM 的使用情况。对于任何你想与 SecureFiles 一起使用的表空间，`SEGMENT_SPACE_MANAGEMENT` 列的值都应为 `AUTO`：

```sql
select tablespace_name, segment_space_management
from dba_tablespaces;
```

此外，SecureFiles 的使用受数据库参数 `DB_SECUREFILE` 管理。你可以使用 `ALTER SYSTEM` 或 `ALTER SESSION` 来修改 `DB_SECUREFILE` 的值。表 11-2 描述了 `DB_SECUREFILE` 的有效值。

表 11-2
`DB_SECUREFILE` 设置说明

| **DB_SECUREFILE** 设置 | **描述** |
| :--- | :--- |
| `NEVER` | 无论是否指定了 `SECUREFILE` 选项，都将 LOB 创建为 BasicFiles 类型。 |
| `PERMITTED` | 允许创建 SecureFiles LOB。 |
| `PREFERRED` | 默认值；指定所有 LOB 都创建为 SecureFiles 类型，除非另有说明。 |
| `ALWAYS` | 将 LOB 创建为 SecureFiles 类型，除非底层表空间未使用 ASSM。 |
| `IGNORE` | 忽略 SecureFiles 选项以及任何 SecureFiles 设置。 |

## 创建包含 LOB 列的表

默认的底层 LOB 架构是 SecureFiles。建议将 LOB 创建为 SecureFiles。如前所述，SecureFiles 允许你使用压缩和加密等功能。

### 创建 BasicFiles LOB 列

要创建 LOB 列，你需要指定 LOB 数据类型。最好显式指定 `STORE AS BASICFILE` 子句，以避免对实现哪种 LOB 架构产生混淆。以下是一个示例：

```sql
SQL> create table patchmain(
patch_id   number
,patch_desc clob)
tablespace users
lob(patch_desc) store as basicfile;
```

在创建带有 LOB 列的表时，你必须了解一些技术基础。请查看以下列表，并确保你理解每一点：

*   在 Oracle Database 12c 之前，默认创建的 LOB 是 BasicFiles 类型。
*   Oracle 为每个 LOB 列创建一个 LOB 段和一个 LOB 索引。
*   LOB 段的名称格式为：`SYS_LOB<string>`。
*   LOB 索引的名称格式为：`SYS_IL<string>`。
*   `<string>` 对于每个 LOB 段及其关联的索引是相同的。
*   LOB 段和索引创建在与表相同的表空间中，除非你指定了不同的表空间。
*   直到记录插入到表中（即所谓的延迟段创建功能），才会创建 LOB 段和 LOB 索引。这意味着 `DBA/ALL/USER_SEGMENTS` 和 `DBA/ALL/USER_EXTENTS` 在行插入到表中之前没有相关信息。

Oracle 为每个 LOB 列创建一个 LOB 段和一个 LOB 索引。LOB 段存储数据。LOB 索引跟踪 LOB 数据的块物理存储位置以及它们应该被访问的顺序。


您可以查询 `DBA_LOBS`、`ALL_LOBS` 或 `USER_LOBS` 视图来显示 LOB 段和 LOB 索引的名称：

```sql
SQL> select table_name, segment_name, index_name, securefile, in_row
from user_lobs;
```

本示例的输出如下：

```
TABLE_NAME   SEGMENT_NAME              INDEX_NAME                SEC IN_
------------ ------------------------- ------------------------- --- ---
PATCHMAIN    SYS_LOB0000022332C00002$$ SYS_IL0000022332C00002$$  NO  YES
```

您也可以查询 `DBA_SEGMENTS`、`USER_SEGMENTS` 或 `ALL_SEGMENTS` 来查看有关 LOB 段的信息。如前所述，直到向表中插入行时才会创建初始段（延迟段创建）。这可能会令人困惑，因为您可能会期望在创建表后，`DBA_SEGMENTS`、`ALL_SEGMENTS` 或 `USER_SEGMENTS` 中会立即出现相关行：

```sql
SQL> select segment_name, segment_type, segment_subtype, bytes/1024/1024 meg_bytes
from user_segments
where segment_name IN ('&&table_just_created',
'&&lob_segment_just_created',
'&&lob_index_just_created');
```

前面的查询会提示输入段名。输出显示没有行：

```
no rows selected
```

接下来，向包含 LOB 列的表中插入一条记录：

```sql
SQL> insert into patchmain values(1,'clob text');
```

重新对 `USER_SEGMENTS` 运行查询，显示已创建三个段——一个用于表，一个用于 LOB 段，一个用于 LOB 索引：

```
SEGMENT_NAME              SEGMENT_TYPE       SEGMENT_SU  MEG_BYTES
------------------------- ------------------ ---------- ----------
PATCHMAIN                 TABLE              ASSM            .0625
SYS_IL0000022332C00002$$  LOBINDEX           ASSM            .0625
SYS_LOB0000022332C00002$$ LOBSEGMENT         ASSM            .0625
```

## 在特定表空间中实施 LOB

默认情况下，LOB 段与其表存储在同一个表空间中。您可以使用 `CREATE TABLE` 语句的 `LOB...STORE AS` 子句为 LOB 段指定一个独立的表空间。下一个建表脚本在某个表空间中创建表，并为 `CLOB` 和 `BLOB` 列创建单独的表空间：

```sql
SQL> create table patchmain
(patch_id   number
,patch_desc clob
,patch      blob
) tablespace users
lob (patch_desc) store as (tablespace lob_data)
,lob (patch)      store as (tablespace lob_data);
```

以下查询验证此表关联的表空间：

```sql
SQL> select table_name, tablespace_name, 'N/A' column_name
from user_tables
where table_name='PATCHMAIN'
union
select table_name, tablespace_name, column_name
from user_lobs
where table_name='PATCHMAIN';
```

输出如下：

```
TABLE_NAME           TABLESPACE_NAME      COLUMN_NAME
-------------------- -------------------- --------------------
PATCHMAIN            LOB_DATA             PATCH
PATCHMAIN            LOB_DATA             PATCH_DESC
PATCHMAIN            USERS                N/A
```

如果您认为 LOB 段需要不同的存储特性（例如大小和增长模式），则建议将 LOB 创建在与表数据分离的表空间中。这样可以将 LOB 列存储与常规表数据存储分开管理。

## 创建安全文件 LOB 列

如前所述，默认的 LOB 架构是安全文件。尽管如此，我建议您明确说明要实施的 LOB 架构，以避免任何混淆。如前所述，包含安全文件 LOB 的表空间必须由 ASSM 管理。以下是创建安全文件 LOB 的示例：

```sql
SQL> create table patchmain(
patch_id   number
,patch_desc clob)
lob(patch_desc) store as securefile (tablespace lob_data);
```

在查看有关 LOB 列的数据字典详细信息之前，请向表中插入一条记录以确保段信息可用（这是由于 Oracle 数据库 11g 第 2 版及更高版本中的延迟段分配特性）；例如，

```sql
SQL> insert into patchmain values(1,'clob text');
```

现在，您可以通过查询 `USER_SEGMENTS` 视图来验证 LOB 的架构：

```sql
SQL> select segment_name, segment_type, segment_subtype
from user_segments;
```

以下是一些示例输出，表明 LOB 段是安全文件类型：

```
SEGMENT_NAME              SEGMENT_TYPE       SEGMENT_SU
------------------------- ------------------ ----------
PATCHMAIN                 TABLE              ASSM
SYS_IL0000022340C00002$$  LOBINDEX           ASSM
SYS_LOB0000022340C00002$$ LOBSEGMENT         SECUREFILE
```

您也可以查询 `USER_LOBS` 视图来验证安全文件 LOB 架构：

```sql
SQL> select table_name, segment_name, index_name, securefile, in_row
from user_lobs;
```

输出如下：

```
TABLE_NAME   SEGMENT_NAME              INDEX_NAME                SEC IN_
------------ ------------------------- ------------------------- --- ---
PATCHMAIN    SYS_LOB0000022340C00002$$ SYS_IL0000022340C00002$$  YES YES
```

**注意：** 使用安全文件架构后，您不再需要指定以下选项：`CHUNK`、`PCTVERSION`、`FREEPOOLS`、`FREELIST` 和 `FREELIST GROUPS`。

## 实施分区 LOB

您可以创建包含 LOB 列的分区表。这样可以让 LOB 跨多个表空间分布。此类分区有助于平衡 I/O、维护以及备份和恢复操作。

您可以按 `RANGE`、`LIST` 或 `HASH` 对 LOB 进行分区。下一个示例创建了一个 `LIST` 分区表，其中 LOB 列数据存储在与表数据不同的表空间中：

```sql
SQL> CREATE TABLE patchmain(
patch_id   NUMBER
,region     VARCHAR2(16)
,patch_desc CLOB)
LOB(patch_desc) STORE AS (TABLESPACE patch1)
PARTITION BY LIST (REGION) (
PARTITION p1 VALUES ('EAST')
LOB(patch_desc) STORE AS SECUREFILE
(TABLESPACE patch1 COMPRESS HIGH)
TABLESPACE inv_data1
,
PARTITION p2 VALUES ('WEST')
LOB(patch_desc) STORE AS SECUREFILE
(TABLESPACE patch2 DEDUPLICATE NOCOMPRESS)
TABLESPACE inv_data2
,
PARTITION p3 VALUES (DEFAULT)
LOB(patch_desc) STORE AS SECUREFILE
(TABLESPACE patch3 COMPRESS LOW)
TABLESPACE inv_data3
);
```

请注意，每个 LOB 分区都是使用其自身的存储选项创建的（有关安全文件特性的详细信息，请参阅本章后面的“实施安全文件高级特性”部分）。您可以查看有关 LOB 分区的详细信息，如下所示：

```sql
SQL> select table_name, column_name, partition_name, tablespace_name
,compression, deduplication
from user_lob_partitions;
```

以下是一些示例输出：

```
TABLE_NAME   COLUMN_NAME     PARTITION_ TABLESPACE_NAME COMPRE DEDUPLICATION
------------ --------------- ---------- --------------- ------ ------------
PATCHMAIN    PATCH_DESC      P1         PATCH1          HIGH   NO
PATCHMAIN    PATCH_DESC      P2         PATCH2          NO     LOB
PATCHMAIN    PATCH_DESC      P3         PATCH3          LOW    NO
```

**提示：** 您也可以查看 `DBA_PART_LOBS`、`ALL_PART_LOBS` 或 `USER_PART_LOBS` 以获取有关分区 LOB 的信息。

您可以在分区 LOB 列创建后更改其存储特性。为此，请使用 `ALTER TABLE ... MODIFY PARTITION` 语句。此示例将 LOB 分区更改为具有高压缩度：

```sql
SQL> alter table patchmain modify partition p2
lob (patch_desc) (compress high);
```

下一个示例修改分区 LOB 以不保留重复值（通过 `DEDUPLICATE` 子句）：

```sql
SQL> alter table patchmain modify partition p3
lob (patch_desc) (deduplicate lob);
```

**注意：** 本章讨论的分区和高级压缩是额外付费选项，仅适用于 Oracle 企业版。

## 维护 LOB 列

以下部分描述了针对 LOB 列或涉及 LOB 列的一些常见维护任务，包括在表空间之间移动列以及向表中添加新的 LOB 列。



# 移动 LOB 列

如前所述，如果创建一个带有 LOB 列的表时未指定表空间，则默认情况下，该 LOB 会与它的表创建在同一个表空间中。这种情况有时发生在数据库管理员 (DBA) 没有提前做好规划的环境中；直到 LOB 列消耗了大量磁盘空间后，DBA 才会疑惑为什么表变得如此之大。

你可以使用 `ALTER TABLE...MOVE...STORE AS` 语句将 LOB 列移动到与表不同的表空间中。其基本语法如下：

```sql
SQL> alter table move lob() store as (tablespace <new_tablespace);
```

下一个示例将 LOB 列移动到 `LOB_DATA` 表空间：

```sql
SQL> alter table patchmain
move lob(patch_desc)
store as securefile (tablespace lob_data);
```

你可以通过查询 `USER_LOBS` 来验证 LOB 是否已移动：

```sql
SQL> select table_name, column_name, tablespace_name from user_lobs;
```

总而言之，如果 LOB 列填充了大量数据，你几乎总是希望将 LOB 存储在与表中其余数据分开的表空间中。在这些场景下，LOB 数据具有不同的增长和存储需求，最好在它们自己的表空间中进行维护。

# 添加 LOB 列

如果你有一个现有的表，并想向其添加 LOB 列，请使用 `ALTER TABLE...ADD` 语句。下一个语句向表中添加了 `INV_IMAGE` 列：

```sql
SQL> alter table patchmain add(inv_image blob);
```

此语句适用于在开发环境中快速添加 LOB 列。对于任何其他情况，你应该指定存储特性。例如，此命令指定在 `LOB_DATA` 表空间中创建一个 SecureFiles LOB：

```sql
SQL> alter table patchmain add(inv_image blob)
lob(inv_image) store as securefile(tablespace lob_data);
```

# 移除 LOB 列

你可能会遇到业务需求变更、不再需要某个列的情况。在移除列之前，考虑将其重命名，以便更好地识别是否有任何应用程序或用户仍在访问它：

```sql
SQL> alter table patchmain rename column patch_desc to patch_desc_old;
```

在确定无人使用该列后，使用 `ALTER TABLE...DROP` 语句将其删除：

```sql
SQL> alter table patchmain drop(patch_desc_old);
```

你也可以通过删除并重新创建表（不包含 LOB 列）来移除 LOB 列。当然，这也会永久删除所有数据。

另外请记住，如果你的回收站已启用，那么当你删除表时未使用 `PURGE` 子句，被删除的表仍然会占用空间。如果你想释放与该表关联的空间，请使用 `PURGE` 子句，或者在删除表后清空回收站。

# 缓存 LOB

默认情况下，在读写 LOB 列时，Oracle 不会将 LOB 缓存在内存中。你可以通过设置与缓存相关的存储选项来更改此默认行为。此示例指定 Oracle 应将 LOB 列缓存在内存中：

```sql
SQL> create table patchmain(
patch_id number
,patch_desc clob)
lob(patch_desc) store as (tablespace lob_data cache);
```

你可以使用以下查询验证 LOB 缓存：

```sql
SQL> select table_name, column_name, cache from user_lobs;
```

以下是一些示例输出：

```
TABLE_NAME           COLUMN_NAME          CACHE
-------------------- -------------------- ----------
PATCHMAIN            PATCH_DESC           YES
```

表 11-3 描述了与 LOB 相关的内存缓存设置。如果你有需要频繁读写的 LOB，请考虑使用 `CACHE` 选项。如果你的 LOB 列频繁读取但很少写入，那么 `CACHE READS` 设置更合适。如果 LOB 列很少进行读写操作，则 `NOCACHE` 设置是合适的。

## 表 11-3: 关于 LOB 列的缓存描述

`缓存设置` | `含义`
--- | ---



# Oracle LOB 存储与高级特性

## 缓存选项

`CACHE` |
 Oracle 应将 LOB 数据置于缓冲区缓存中以便更快访问。 |

`CACHE READS` |
 Oracle 应将 LOB 数据置于缓冲区缓存中用于读取，但不用于写入。 |

`NOCACHE` |
 不应将 LOB 数据置于缓冲区缓存中。这是 SecureFiles 和 BasicFiles LOBs 的默认设置。 |

## LOB 行内与行外存储

默认情况下，LOB 列最多约 4,000 个字符会与表行一起**行内**存储。如果 LOB 超过 4,000 个字符，Oracle 会自动将其存储在行数据之外。

将 LOB 存储在行内的主要优势是，较小的 LOB（少于 4,000 个字符）需要更少的 I/O，因为 Oracle 无需到行外去搜索 LOB 数据。

然而，将 LOB 数据存储在行内并不总是可取的。将 LOBs 存储在行内的缺点是表行的大小可能变长。这会影响全表扫描、范围扫描以及对 LOB 列以外的列进行更新的性能。在这些情况下，您可能希望禁用行内存储。例如，您可以使用 `DISABLE STORAGE IN ROW` 子句明确指示 Oracle 将 LOB 存储在行外：

```sql
SQL> create table patchmain(
patch_id number
,patch_desc clob
,log_file   blob)
lob(patch_desc, log_file)
store as (
tablespace lob_data
disable storage in row);
```

如果您希望将最多 4,000 个字符的 LOB 存储在表行中，请在创建表时使用 `ENABLE STORAGE IN ROW` 子句：

```sql
SQL> create table patchmain(
patch_id number
,patch_desc clob
,log_file   blob)
lob(patch_desc, log_file)
store as (
tablespace lob_data
enable storage in row);
```

> **注意**
> LOB 定位符始终与行一起行内存储。

在表创建后，您无法修改 LOB 的行内存储设置。更改行内存储的唯一方法是移动 LOB 列或删除并重新创建表。以下示例通过移动 LOB 列来更改行内存储设置：

```sql
SQL> alter table patchmain
move lob(patch_desc)
store as (enable storage in row);
```

您可以通过 `USER_LOBS` 的 `IN_ROW` 列来验证行内存储状态：

```sql
SQL> select table_name, column_name, tablespace_name, in_row
from user_lobs;
```

值为 `YES` 表示 LOB 存储在行内：

```
TABLE_NAME      COLUMN_NAME     TABLESPACE_NAME IN_ROW
--------------- --------------- --------------- ------
PATCHMAIN       LOG_FILE        LOB_DATA        YES
PATCHMAIN       PATCH_DESC      LOB_DATA        YES
```

## 实现 SecureFiles 高级功能

如前所述，SecureFiles LOB 架构允许您压缩 LOB 列、消除重复项并透明地加密 LOB 数据。这些功能提供了 LOB 的高性能和高可管理性。接下来的几节将介绍 SecureFiles 特有的功能。

### 压缩 LOBs

如果您使用的是 SecureFiles LOB，则可以指定压缩级别。好处是 LOB 在数据库中占用的空间将大大减少。缺点是读写 LOB 可能需要更长时间。有关压缩值的描述，请参见表 11-4。

**表 11-4 SecureFiles LOB 可用的压缩级别**

| **压缩类型** | **描述** |
| :--- | :--- |
| `HIGH` | 最高压缩级别；读写 LOB 时延迟较高 |
| `MEDIUM` | 中等级别压缩；如果指定了压缩但未指定级别，则为默认值 |
| `LOW` | 最低压缩级别；读写 LOB 时延迟最低 |

以下示例创建了一个具有**低**压缩级别的 `CLOB` 列：

```sql
SQL> CREATE TABLE patchmain(
patch_id   NUMBER
,patch_desc CLOB)
LOB(patch_desc) STORE AS SECUREFILE
(COMPRESS LOW)
TABLESPACE lob_data;
```

如果 LOB 已创建为 SecureFiles 类型，您可以更改其压缩级别。例如，此命令将压缩级别更改为 `HIGH`：

```sql
SQL> alter table patchmain modify lob(patch_desc) (compress high);
```



# Oracle SecureFiles 大型对象特性配置

## 压缩

如果你创建了一个启用了压缩的大型对象，但后来决定不再使用该功能，可以通过 `NOCOMPRESS` 子句修改大型对象以取消压缩：

```sql
SQL> alter table patchmain modify lob(patch_desc) (nocompress);
```

> **提示：** 尝试通过 `CREATE TABLE` 语句启用压缩、去重和加密。如果你使用 `ALTER TABLE` 语句，表在修改大型对象期间会被锁定。

### 去重

如果你的应用程序中，两个或多个行关联了相同的大型对象，你应该考虑使用 SecureFiles 去重功能。启用后，这会指示 Oracle 在将新大型对象插入表时检查该大型对象是否已存储在另一行中（针对同一大型对象列）。如果大型对象已存在，则 Oracle 会存储一个指向现有相同大型对象的指针。这可能为你的应用程序节省大量空间。

> **注意：** 去重需要企业版数据库的 Oracle Advanced Compression 选项许可。

以下示例创建一个大型对象列，并使用了去重功能：

```sql
SQL> CREATE TABLE patchmain(
patch_id   NUMBER
,patch_desc CLOB)
LOB(patch_desc) STORE AS SECUREFILE
(DEDUPLICATE)
TABLESPACE lob_data;
```

要验证去重功能是否生效，可以运行此查询：

```sql
SQL> select table_name, column_name, deduplication
from user_lobs;
```

示例输出如下：

```
TABLE_NAME      COLUMN_NAME     DEDUPLICATION
--------------- --------------- ---------------
PATCHMAIN       PATCH_DESC      LOB
```

如果现有表已有 SecureFiles 大型对象，则可以修改该列以启用去重：

```sql
SQL> alter table patchmain
modify lob(patch_desc) (deduplicate);
```

这是修改分区大型对象以启用去重的另一个示例：

```sql
SQL> alter table patchmain modify partition p2
lob (patch_desc) (deduplicate lob);
```

如果你决定不启用去重，可以使用 `KEEP_DUPLICATES` 子句：

```sql
SQL> alter table patchmain
modify lob(patch_desc) (keep_duplicates);
```

## 加密

你可以透明地加密 SecureFiles 大型对象列（就像加密任何其他列一样）。在使用加密功能之前，你必须设置一个加密钱包。本节末尾包含了一个侧边栏，详细说明了如何设置钱包。

> **注意：** SecureFiles 加密功能需要企业版数据库的 Oracle Advanced Security 选项许可。

`ENCRYPT` 子句启用 SecureFiles 加密，使用 Oracle 透明数据加密。传统的大型对象（不使用 SecureFiles）可以通过 `DBMS_CRYPTO` 工具进行加密。以下示例为 `PATCH_DESC` 大型对象列启用加密：

```sql
SQL> CREATE TABLE patchmain(
patch_id number
,patch_desc clob)
LOB(patch_desc) STORE AS SECUREFILE (encrypt)
tablespace lob_data;
```

当你描述该表时，大型对象列现在会显示加密已生效：

```sql
SQL> desc patchmain;
Name                                      Null?    Type
----------------------------------------- -------- ------------------------
PATCH_ID                                           NUMBER
PATCH_DESC                                         CLOB ENCRYPT
```

这是一个稍有不同的示例，它将 `ENCRYPT` 关键字与大型对象列内联指定：

```sql
SQL> CREATE TABLE patchmain(
patch_id   number
,patch_desc clob encrypt)
LOB (patch_desc) STORE AS SECUREFILE;
```

你可以通过查询 `DBA_ENCRYPTED_COLUMNS` 视图来验证加密详细信息：

```sql
SQL> select table_name, column_name, encryption_alg
from dba_encrypted_columns;
```

本例的输出如下：

```
TABLE_NAME           COLUMN_NAME          ENCRYPTION_ALG
-------------------- -------------------- --------------------
PATCHMAIN            PATCH_DESC           AES 192 bits key
```

如果你已经创建了表，可以修改列以启用加密：

```sql
SQL> alter table patchmain modify
(patch_desc clob encrypt);
```



# 使用安全文件 LOB

## 指定加密算法

您也可以指定加密算法；例如，

```sql
SQL> alter table patchmain modify
(patch_desc clob encrypt using '3DES168');
```

## 禁用加密

您可以通过 `DECRYPT` 子句为安全文件 LOB 列禁用加密：

```sql
SQL> alter table patchmain modify
(patch_desc clob decrypt);
```

## 启用 Oracle Wallet

Oracle Wallet 是 Oracle 用来实现加密的机制。钱包是一个包含加密密钥的操作系统文件。启用钱包需遵循以下步骤：

1.  修改 `SQLNET.ORA` 文件，包含钱包的位置：

```sql
    ENCRYPTION_WALLET_LOCATION=
    (SOURCE=(METHOD=FILE) (METHOD_DATA=
    (DIRECTORY=/ora01/app/oracle/product/18.1.0.0/db_1/network/admin)))
```

2.  使用 `ALTER SYSTEM` 命令创建钱包文件 (`ewallet.p18`)：

```sql
    SQL> alter system set encryption key identified by foo;
```

3.  启用加密：

```sql
    SQL> alter system set encryption wallet open identified by foo;
```

有关实现加密的完整详情，请参阅《Oracle 高级安全管理员指南》，该指南可从 Oracle 网站的技术网络区域免费下载 ([`http://otn.oracle.com`](http://otn.oracle.com))。

## 将基本文件迁移至安全文件

您可以通过以下方法之一将基本文件 LOB 数据迁移至安全文件：

*   创建新表，从旧表加载数据，然后重命名表
*   移动表
*   在线重定义表

以下各节将描述每种技术。

### 创建新表

下面是一个创建新表并从旧表加载数据的简要示例。在此示例中，`PATCHMAIN_NEW` 是正在创建的新表，使用安全文件 LOB。

```sql
SQL> create table patchmain_new(
patch_id number
,patch_desc clob)
lob(patch_desc) store as securefile (tablespace lob_data);
```

接下来，用旧表的数据填充新创建的表：

```sql
SQL> insert into patchmain_new select * from patchmain;
```

现在，重命名表：

```sql
SQL> rename patchmain to patchmain_old;
SQL> rename patchmain_new to patchmain;
```

使用此技术时，请确保任何指向旧表的授权都已为新表重新发放。

### 将表移动到安全文件架构

您也可以使用 `ALTER TABLE...MOVE` 语句将 LOB 的存储重新定义为安全文件类型；例如，

```sql
SQL> alter table patchmain
move lob(patch_desc)
store as securefile (tablespace lob_data);
```

您可以通过此查询验证该列现在是否为安全文件类型：

```sql
SQL> select table_name, column_name, securefile from user_lobs;
```

`SECUREFILE` 列现在的值为 `YES`：

```
TABLE_NAME      COLUMN_NAME     SEC
--------------- --------------- ---
PATCHMAIN       PATCH_DESC      YES
```

### 使用在线重定义进行迁移

您也可以通过 `DBMS_REDEFINITION` 包在表在线时对其进行重定义。使用以下步骤执行在线重定义：

1.  确保表具有主键。如果表没有主键，则创建一个：

```sql
    SQL> alter table patchmain
    add constraint patchmain_pk
    primary key (patch_id);
```

2.  创建一个新表，将 LOB 列定义为安全文件类型：

```sql
    SQL> create table patchmain_new(
    patch_id number
    ,patch_desc clob)
    lob(patch_desc)
    store as securefile (tablespace lob_data);
```

3.  映射列，并将数据从原始表复制到新表（如果行很多，这可能需要很长时间）：

```sql
    SQL> declare
    l_col_map varchar2(2000);
    begin
    l_col_map := 'patch_id patch_id, patch_desc patch_desc';
    dbms_redefinition.start_redef_table(
    'MV_MAINT','PATCHMAIN','PATCHMAIN_NEW',l_col_map
    );
    end;
    /
```

4.  克隆被重定义表的相关对象（授权、触发器、约束等）：

```sql
    SQL> set serverout on size 1000000
    SQL> declare
    l_err_cnt integer :=0;
    begin
    dbms_redefinition.copy_table_dependents(
    'MV_MAINT','PATCHMAIN','PATCHMAIN_NEW',1,TRUE, TRUE, TRUE, FALSE, l_err_cnt
    );
    dbms_output.put_line('Num Errors: ' || l_err_cnt);
    end;
    /
```

5.  完成重定义：

```sql
    SQL> begin
    dbms_redefinition.finish_redef_table('MV_MAINT','PATCHMAIN','PATCHMAIN_NEW');
    end;
    /
```

您可以通过此查询确认表已被重定义：

```sql
SQL> select table_name, column_name, securefile from user_lobs;
```

以下是此示例的输出：

```
TABLE_NAME           COLUMN_NAME          SECUREFILE
-------------------- -------------------- --------------------
PATCHMAIN_NEW        PATCH_DESC           NO
PATCHMAIN            PATCH_DESC           YES
```

## 查看 LOB 元数据

您可以使用任何 `DBA/ALL/USER_LOBS` 视图来显示数据库中 LOB 的信息：

```sql
SQL> select table_name, column_name, index_name, tablespace_name
from all_lobs
order by table_name;
```

同时请记住，LOB 段有一个对应的索引段。

```sql
SQL> select segment_name, segment_type, tablespace_name
from user_segments
where segment_name like 'SYS_LOB%'
or    segment_name like 'SYS_IL%';
```

这样，您就可以在 `DBA/ALL/USER_SEGMENTS` 视图中同时查询段和索引的 LOB 信息。

## 加载 LOB

加载 LOB 数据通常不是 DBA 的工作，但您应该熟悉用于填充 LOB 列的技术。开发人员可能会向您寻求故障排除、性能或空间相关问题的帮助。

### 加载 CLOB

首先，创建一个 Oracle 数据库目录对象，该对象指向存储 `CLOB` 文件的操作系统目录。此目录对象在加载 `CLOB` 时使用。在此示例中，Oracle 目录对象名为 `LOAD_LOB`，操作系统目录为 `/orahome/oracle/lob`：

```sql
SQL> create or replace directory load_lob as '/orahome/oracle/lob';
```

供参考，接下来列出的是用于创建加载 `CLOB` 文件的表的 DDL：

```sql
SQL> create table patchmain(
patch_id number primary key
,patch_desc clob
,patch_file blob)
lob(patch_desc, patch_file)
store as securefile (compress low) tablespace lob_data;
```

此示例还使用了一个名为 `PATCH_SEQ` 的序列。以下是序列创建脚本：

```sql
SQL> create sequence patch_seq;
```

下面的代码使用 `DBMS_LOB` 包将文本文件 (`patch.txt`) 加载到 `CLOB` 列中。在此示例中，表名为 `PATCHMAIN`，`CLOB` 列为 `PATCH_DESC`：

```sql
SQL> declare
src_clb bfile; -- 指向文件系统上的源 CLOB
dst_clb clob;  -- 表中的目标 CLOB
src_doc_name varchar2(300) := 'patch.txt';
src_offset integer := 1; -- 在源 CLOB 中的起始位置
dst_offset integer := 1;  -- 在目标 CLOB 中的起始位置
lang_ctx integer := dbms_lob.default_lang_ctx;
warning_msg number; -- 如果遇到坏字符则返回警告值
begin
src_clb := bfilename('LOAD_LOB',src_doc_name); -- 分配指向文件的指针
--
insert into patchmain(patch_id, patch_desc) -- 创建 LOB 占位符
values(patch_seq.nextval, empty_clob())
returning patch_desc into dst_clb;
--
dbms_lob.open(src_clb, dbms_lob.lob_readonly); -- 打开文件
--
-- 将文件加载到 LOB 中
dbms_lob.loadclobfromfile(
dest_lob => dst_clb,
src_bfile => src_clb,
amount => dbms_lob.lobmaxsize,
dest_offset => dst_offset,
src_offset => src_offset,
bfile_csid => dbms_lob.default_csid,
lang_context => lang_ctx,
warning => warning_msg
);
dbms_lob.close(src_clb); -- 关闭文件
--
dbms_output.put_line('Wrote CLOB: ' || src_doc_name);
end;
/
```

您可以将此代码放在一个文件中，并从 SQL 命令提示符执行。在此示例中，包含代码的文件名为 `clob.sql`：

```sql
SQL> set serverout on size 1000000
SQL> @clob.sql
```

预期输出如下：

```
Wrote CLOB: patch.txt
PL/SQL procedure successfully completed.
```



# 加载 BLOB

加载`BLOB`与加载`CLOB`类似。此示例使用了上一个示例（加载`CLOB`的示例）中的目录对象、表和序列。加载`BLOB`比加载`CLOB`更简单，因为您无需指定字符集信息。

此示例将一个名为`patch.zip`的文件加载到`PATCH_FILE BLOB`列中：

```sql
SQL> declare
src_blb bfile; -- 指向文件系统上的源 BLOB
dst_blb blob;  -- 表中的目标 BLOB
src_doc_name varchar2(300) := 'patch.zip';
src_offset integer := 1; -- 在源 BLOB 中开始的位置
dst_offset integer := 1;  -- 在目标 BLOB 中开始的位置
begin
src_blb := bfilename('LOAD_LOB',src_doc_name); -- 为文件分配指针
--
insert into patchmain(patch_id, patch_file)
values(patch_seq.nextval, empty_blob())
returning patch_file into dst_blb; -- 首先创建 LOB 占位符列
dbms_lob.open(src_blb, dbms_lob.lob_readonly);
--
dbms_lob.loadblobfromfile(
dest_lob => dst_blb,
src_bfile => src_blb,
amount => dbms_lob.lobmaxsize,
dest_offset => dst_offset,
src_offset => src_offset
);
dbms_lob.close(src_blb);
dbms_output.put_line('Wrote BLOB: ' || src_doc_name);
end;
/
```

您可以将此代码放在文件中，并从 SQL 命令提示符运行它。此处，包含代码的文件名为`blob.sql`：

```sql
SQL> set serverout on size 1000000
SQL> @blob.sql
```

预期的输出如下：

```
Wrote BLOB: patch.zip
PL/SQL procedure successfully completed.
```

## 测量 LOB 占用的空间

如前所述，LOB 由行内 LOB 定位器、LOB 索引以及由一个或多个块组成的 LOB 段构成。与 LOB 段所使用的空间相比，LOB 索引使用的空间通常可以忽略不计。您可以通过查询`DBA/ALL/USER_SEGMENTS`的`BYTES`列来查看段所消耗的空间（就像数据库中的任何其他段一样）。以下是一个示例查询：

```sql
SQL> select segment_name, segment_type, segment_subtype,
bytes/1024/1024 meg_bytes
from user_segments;
```

您可以通过连接到`USER_LOBS`视图来修改查询，以便仅报告 LOB 的信息：

```sql
SQL> select a.table_name, a.column_name, a.segment_name, a.index_name
,b.bytes/1024/1024 meg_bytes
from user_lobs a, user_segments b
where a.segment_name = b.segment_name;
```

您还可以使用`DBMS_SPACE.SPACE_USAGE`包和过程来报告 LOB 正在使用的块。此包仅适用于在 ASSM 管理的表空间中创建的对象。`SPACE_USAGE`过程有两种不同的形式：一种用于报告 BasicFiles LOB，另一种用于报告 SecureFiles LOB。

### BasicFiles 使用空间

以下是如何为 BasicFiles LOB 调用`DBMS_SPACE.SPACE_USAGE`的示例：

```sql
SQL> declare
p_fs1_bytes number;
p_fs2_bytes number;
p_fs3_bytes number;
p_fs4_bytes number;
p_fs1_blocks number;
p_fs2_blocks number;
p_fs3_blocks number;
p_fs4_blocks number;
p_full_bytes number;
p_full_blocks number;
p_unformatted_bytes number;
p_unformatted_blocks number;
begin
dbms_space.space_usage(
segment_owner      => user,
segment_name       => 'SYS_LOB0000024082C00002$$',
segment_type       => 'LOB',
fs1_bytes          => p_fs1_bytes,
fs1_blocks         => p_fs1_blocks,
fs2_bytes          => p_fs2_bytes,
fs2_blocks         => p_fs2_blocks,
fs3_bytes          => p_fs3_bytes,
fs3_blocks         => p_fs3_blocks,
fs4_bytes          => p_fs4_bytes,
fs4_blocks         => p_fs4_blocks,
full_bytes         => p_full_bytes,
full_blocks        => p_full_blocks,
unformatted_blocks => p_unformatted_blocks,
unformatted_bytes  => p_unformatted_bytes
);
dbms_output.put_line('Full bytes  = '||p_full_bytes);
dbms_output.put_line('Full blocks = '||p_full_blocks);
dbms_output.put_line('UF bytes    = '||p_unformatted_bytes);
dbms_output.put_line('UF blocks   = '||p_unformatted_blocks);
end;
/
```

在此 PL/SQL 中，您需要修改代码，以便报告您环境中 LOB 段的信息。

### SecureFiles 使用空间

以下是如何为 SecureFiles LOB 调用`DBMS_SPACE.SPACE_USAGE`的示例：

```sql
SQL> DECLARE
l_segment_owner         varchar2(40);
l_table_name            varchar2(40);
l_segment_name          varchar2(40);
l_segment_size_blocks   number;
l_segment_size_bytes    number;
l_used_blocks           number;
l_used_bytes            number;
l_expired_blocks        number;
l_expired_bytes         number;
l_unexpired_blocks      number;
l_unexpired_bytes       number;
--
CURSOR c1 IS
SELECT owner, table_name, segment_name
FROM dba_lobs
WHERE table_name = 'PATCHMAIN';
BEGIN
FOR r1 IN c1 LOOP
l_segment_owner := r1.owner;
l_table_name := r1.table_name;
l_segment_name := r1.segment_name;
--
dbms_output.put_line('-----------------------------');
dbms_output.put_line('Table Name         : ' || l_table_name);
dbms_output.put_line('Segment Name       : ' || l_segment_name);
--
dbms_space.space_usage(
segment_owner           => l_segment_owner,
segment_name            => l_segment_name,
segment_type            => 'LOB',
partition_name          => NULL,
segment_size_blocks     => l_segment_size_blocks,
segment_size_bytes      => l_segment_size_bytes,
used_blocks             => l_used_blocks,
used_bytes              => l_used_bytes,
expired_blocks          => l_expired_blocks,
expired_bytes           => l_expired_bytes,
unexpired_blocks        => l_unexpired_blocks,
unexpired_bytes         => l_unexpired_bytes
);
--
dbms_output.put_line('segment_size_blocks: '||  l_segment_size_blocks);
dbms_output.put_line('segment_size_bytes : '||  l_segment_size_bytes);
dbms_output.put_line('used_blocks        : '||  l_used_blocks);
dbms_output.put_line('used_bytes         : '||  l_used_bytes);
dbms_output.put_line('expired_blocks     : '||  l_expired_blocks);
dbms_output.put_line('expired_bytes      : '||  l_expired_bytes);
dbms_output.put_line('unexpired_blocks   : '||  l_unexpired_blocks);
dbms_output.put_line('unexpired_bytes    : '||  l_unexpired_bytes);
END LOOP;
END;
/
```

同样，在此 PL/SQL 中，您需要修改代码，以便报告您环境中带有 LOB 段的表的信息。

## 读取 BFILE

如前所述，`BFILE`数据类型只是表中的一个列，它存储指向操作系统文件的指针。`BFILE`为您提供对磁盘上二进制文件的只读访问权限。要访问`BFILE`，必须首先创建一个目录对象。这是一个存储操作系统目录位置的数据库对象。目录对象使 Oracle 知晓磁盘上`BFILE`的位置。

此示例首先创建一个目录对象，然后创建一个包含`BFILE`列的表，最后使用`DBMS_LOB`包访问二进制文件：

```sql
SQL> create or replace directory load_lob as '/orahome/oracle/lob';
```

接下来，创建一个包含`BFILE`数据类型的表：

```sql
SQL> create table patchmain
(patch_id   number
,patch_file bfile);
```

对于此示例，假设一个名为`patch.zip`的文件位于上述目录中。您通过使用目录对象和文件名向表中插入一条记录，使 Oracle 知晓该二进制文件：

```sql
SQL> insert into patchmain values(1, bfilename('LOAD_LOB','patch.zip'));
```

现在，您可以通过`DBMS_LOB`包访问`BFILE`。例如，如果您想验证文件是否存在或显示 LOB 的长度，可以按如下方式操作：

```sql
SQL> select dbms_lob.fileexists(bfilename('LOAD_LOB','patch.zip')) from dual;
SQL> select dbms_lob.getlength(patch_file) from patchmain;
```

通过这种方式，二进制文件的行为类似于`BLOB`。最大的区别在于，该二进制文件不存储在数据库内部。

**提示**
有关使用`DBMS_LOB`包的完整详细信息，请参阅《Oracle Database PL/SQL Packages and Types Reference》指南。该指南可在 [`http://otn.oracle.com`](http://otn.oracle.com) 获取。



## 概述

Oracle 允许通过各种 LOB 数据类型在数据库中存储大型对象。LOB 便于存储、管理和检索视频片段、图像、电影、文字处理文档、大型文本文件等。Oracle 可以将这些文件存储在数据库中，从而提供备份恢复和安全保护（就像对待任何其他数据类型一样）。`BLOB` 用于存储二进制文件，例如图像（JPEG、MPEG）、电影文件、声音文件等。如果将文件存储在数据库中不可行，可以使用 `BFILE` LOB。

Oracle 为 LOB 提供了两种底层架构：`BasicFiles` 和 `SecureFiles`。`BasicFiles` 是自 Oracle 版本 8 以来就可用的 LOB 架构。`SecureFiles` 功能在 Oracle Database 11g 中引入，现在是 LOB 的默认值。`SecureFiles` 具有许多高级选项，例如压缩、去重和加密（这些特定功能需要从 Oracle 获取额外许可）。

LOB 提供了一种管理非常大的文件的方法。Oracle 还有另一个特性——分区——它允许您管理非常大的表和索引。分区将在下一章详细讨论。

# 第 12 章 分区：分而治之

Oracle 提供了两个关键的可扩展性特性，即使对于非常大的数据库也能实现良好的性能：并行处理和分区。并行处理允许 Oracle 启动多个执行线程以利用多个硬件资源。分区允许表或索引的子集被独立管理（Oracle 的“分而治之”方法）。本章的重点是分区策略。

分区允许您创建一个逻辑表或索引，该表或索引由独立的段组成，每个段都可以由独立的执行线程访问和处理。表或索引的每个分区都具有相同的逻辑结构，例如列定义，但可以驻留在不同的容器中。换句话说，您可以将每个分区存储在自己的表空间和相关的数据文件中。这使您能够将一个大的逻辑对象作为一组更小、更易于维护的部分进行管理。

从分区中获得的主要好处如下：

*   更好的性能；在某些情况下，SQL 查询可以在单个分区或分区子集上操作，这允许更快的执行时间。
*   更高的可用性；一个分区中数据的可用性不受另一个分区中数据不可用的影响。
*   更容易的维护；按分区插入、更新、删除、截断、重建和重组数据，可实现高效的加载和归档操作，否则这些操作将困难且耗时。
*   分区管理；修改策略和管理分区可以是联机和自动化的活动。

仅仅实施分区并不意味着您会自动获得性能提升、实现高可用性并简化管理活动。您需要了解分区的工作原理以及如何利用各种特性来获得任何好处。本章的目标是解释分区概念以及如何实施分区，并就何时使用哪些特性提供指导。

在深入细节之前，您首先应该熟悉几个分区术语。表 12-1 描述了本章中使用的关键分区术语的含义。

**表 12-1 Oracle 分区术语**

| 术语 | 含义 |
| :--- | :--- |
| 分区 | 透明地将一个逻辑表或索引实现为许多独立的、较小的段 |
| 分区键 | 明确确定行存储在哪个分区中的一个或多个列 |
| 分区边界 | 分区之间的边界 |
| 单级分区 | 使用单一方法的分区 |
| 组合分区 | 使用组合方法的分区 |
| 子分区 | 分区内的分区 |
| 分区独立性 | 能够单独访问分区以执行维护操作而不影响其他分区的可用性 |
| 分区修剪 | 消除不必要的分区。Oracle 检测 SQL 语句需要访问哪些分区，并从考虑中移除任何不需要的分区。 |
| 分区智能连接 | 以分区大小的块执行连接，通过并行执行许多较小的任务而不是顺序执行一个大型任务来提高性能 |
| 本地分区索引 | 使用与表相同分区键的索引 |
| 全局分区索引 | 不使用与表相同分区键的索引 |
| 全局非分区索引 | 在分区表上创建的常规索引。索引本身未分区。 |

另外请记住，如果您主要处理小型 OLTP 数据库，则不需要创建分区表和索引。但是，如果您在 OLTP 数据库或数据仓库环境中处理非常大的对象，您很可能可以从分区中受益。分区是设计和构建可扩展、高可用的大型数据库系统的关键。

# 哪些表应该被分区？

以下是一些用于确定是否对表进行分区的经验法则。通常，您应考虑对以下表进行分区：

*   大于 10GB 的
*   行数超过 1000 万，并且随着更多数据的添加，SQL 操作变得越来越慢的
*   您知道会变得很大的（最好在创建表时就将其设为分区表，而不是在表增长导致性能开始下降后再重建为分区表）
*   其行可以以有利于并行操作（如插入、检索、删除以及备份和恢复）的方式进行划分的
*   您希望定期归档最旧的数据，或者您希望定期删除最旧的分区（因为数据变旧）

一条规则是，任何大于 10GB 的表都是潜在的候选分区对象。运行此查询以显示数据库中占用空间最多的前几个对象：

```
SQL> select * from (
select owner, segment_name, segment_type, partition_name
,sum(bytes)/1024/1024 meg_tot
from dba_segments
group by owner, segment_name, segment_type, partition_name
order by sum(extents) desc)
where rownum <= 10;
```

以下是该查询输出的片段：

```
OWNER     SEGMENT_NAME              SEGMENT_TYPE PARTITION_NAME     MEG_TOT
--------  -------------------------------------- --------------  ----------
MV_MAINT  F_SALES                   TABLE                             15281
MV_MAINT  F_SALES_IDX1              INDEX                              8075
```

此输出显示该数据库中的几个大型对象可能从分区中受益。对于此数据库，如果这些大型对象存在性能问题，那么分区可能会有所帮助。

如果您从 `SQL*Plus` 运行前面的查询，您需要对列应用一些格式化，以便在终端的有限宽度内合理地显示输出：

```
set lines 132
col owner form a10
col segment_name form a25
col partition_name form a15
```



除了观察对象的大小之外，如果你能以利于操作（如加载数据、查询、备份、归档和删除）的方式来划分数据，那么就应该考虑使用分区。例如，如果你处理的一个表包含大量行，并且这些行经常根据特定的时间范围（如按天、周、月或年）被访问，那么考虑分区就是有意义的。即使是按地区划分的数据，为了便于加载或访问而进行分区也可能很有意义，因为这正是数据主要被按地区列表、组或类别访问的方式。

大表的大小加上良好的业务需求，意味着你应该考虑分区。请记住，对表进行分区会带来更多的设置工作和维护工作。然而，如前所述，在设置时对表进行分区要比在表增长到难以处理的大小后再进行转换容易得多。在表随着在线维护而增长后，处理各个分区也更容易，因为分区策略可以被修改，或者分区可以被合并在一起。

> **注意**
> 分区是一项额外收费的选件，仅在 **Oracle 企业版** 中可用。你必须根据你的业务需求来判断分区是否值得这个成本。数据仓库和超大规模数据库（VLDB）应结合成本和数据访问需求来评估这个选件。

## 创建分区表

Oracle 提供了一套健壮的方法，用于将表和索引划分为更小的子集。例如，你可以按日期范围（如按月或年）来划分表的数据。表 12-2 概述了可用的分区策略。

表 12-2. 分区策略

| 分区类型 | 描述 |
| --- | --- |
| 范围 | 允许基于日期、数字或字符的范围进行分区 |
| 列表 | 当分区恰好可以归入一个值列表（如州或地区代码）时非常有用 |
| 哈希 | 当没有明显的分区键时，允许均匀分布行 |
| 组合 | 允许组合其他分区策略 |
| 间隔 | 通过在新的分区键值超过现有的上限范围时自动分配新分区，来扩展范围分区 |
| 引用 | 用于基于父表列对子表进行分区非常有用 |
| 虚拟 | 允许在虚拟列上进行分区 |
| 系统 | 允许插入数据的应用程序决定应使用哪个分区 |

以下各节展示了每种分区策略的示例。此外，你还将学习如何将分区放置在单独的表空间中；为了充分利用分区的所有优势，你需要了解如何将分区分配给其自己的表空间。

### 按范围分区

范围分区被频繁使用。此策略指示 Oracle 根据值的范围（例如日期、数字或字符）将行放入分区中。当数据插入到范围分区表中时，Oracle 会根据每个范围分区的上下界来确定将行放入哪个分区。

基于范围的分区键由 `CREATE TABLE` 语句中的 `PARTITION BY RANGE` 子句定义。这决定了使用哪一列来控制行所属的分区。你很快就会看到一些例子。

每个范围分区都需要一个 `VALUES LESS THAN` 子句来标识范围上限的非包含值。为范围定义的第一个分区没有下限。任何小于第一个分区 `VALUES LESS THAN` 子句中设置的值都会被插入到第一个分区中。对于第一个分区之外的分区，范围的下限由前一个分区的上限决定。

你可以选择使用 `MAXVALUE` 子句来创建范围分区表的最高分区。任何其分区键值不在较低范围内的行都将被插入到这个最高的 `MAXVALUE` 分区中。

#### 将 NUMBER 类型用作分区键列

让我们看一个例子来说明前面的概念。假设你正在数据仓库环境中工作，其中通常有一个事实表来存储关于事件的信息，如销售额、利润、注册量等。在事实表中，通常一列表示金额或计数，另一列表示时间点。

一些数据仓库架构师选择用数字来表示时间点列——其思路是，当连接到多个维度表时，数字数据类型效率更高。例如，值 20130101 被用来表示 2013 年 1 月 1 日，你使用这个数字值（代表一个日期）作为对事实表进行分区的列。下面的 SQL 语句创建了一个基于数字范围的、包含三个分区的表：

```sql
SQL> create table f_sales
(sales_amt number
,d_date_id number)
partition by range (d_date_id)(
partition p_2012 values less than (20130101),
partition p_2013 values less than (20140101),
partition p_max  values less than (maxvalue));
```

创建范围分区表时，你不必指定 `MAXVALUE` 分区。但是，如果你没有指定带有 `MAXVALUE` 子句的分区，并且你试图插入一行，而该行的值不属于任何其他已定义的范围，你将收到类似以下的错误：

```
ORA-14400: inserted partition key does not map to any partition
```

当你看到这个错误时，你必须添加一个能够容纳被插入的分区键值的分区，或者添加一个带有 `MAXVALUE` 子句的分区。

> **提示**
> 考虑使用间隔分区策略，在这种策略中，当超过上限范围值时，Oracle 会自动添加分区。请参阅本章后面的“按需创建分区”一节。

你可以通过运行以下查询来查看刚刚创建的分区表的信息：

```sql
SQL> select table_name, partitioning_type, def_tablespace_name
from user_part_tables
where table_name='F_SALES';
```

以下是输出的一小部分：

```
TABLE_NAME           PARTITION DEF_TABLESPACE_NAME
-------------------- --------- ------------------------------
F_SALES              RANGE     USERS
```

要查看表中分区的信息，可发出如下查询：

```sql
SQL> select table_name, partition_name, high_value
from user_tab_partitions
where table_name = 'F_SALES'
order by table_name, partition_name;
```

以下是一些示例输出：



表名                 分区名                高值
-------------------- -------------------- --------------------
F_SALES              P_2016               20170101
F_SALES              P_2017               20180101
F_SALES              P_MAX                MAXVALUE

在这个示例中，`D_DATE_ID` 列是分区键列。`VALUES LESS THAN` 子句创建了分区边界；这些边界定义了行将被插入到的分区。`MAXVALUE` 参数创建了一个分区，用于存储不符合其他已定义分区条件的行（包括 `NULL` 值）。

### 检测何时需要增加高值范围
当你按范围分区而没有指定 `MAXVALUE` 分区时，你可能无法准确预测何时需要添加一个新的高值分区。此外，数据字典中的 `HIGH_VALUE` 列是 `LONG` 数据类型，这意味着你无法应用 `MAX` SQL 函数来返回当前的高值。

下面列出了一个简单的 shell 脚本，它尝试插入一条包含未来日期的记录，以确定是否存在能接收它的分区。如果记录插入成功，脚本将回滚事务。如果记录插入失败，则会生成错误，脚本会给你发送一封电子邮件：

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
echo "Usage: $0 SID"
exit 1
fi
export ORACLE_SID=o18c
export ORACLE_HOME=/ora01/app/oracle/product/18.0.0.1/db_1
#
sqlplus -s <<EOF
mv_maint/foo
WHENEVER SQLERROR EXIT FAILURE
COL date_id NEW_VALUE hold_date_id
SELECT to_char(sysdate+30,'yyyymmdd') date_id FROM dual;
--
INSERT INTO mv_maint.f_sales(sales_amt, d_date_id)
VALUES (0, '&hold_date_id');
ROLLBACK;
EOF
#
if [ $? -ne 0 ]; then
mailx -s "Partition range issue: f_sales" dkuhn@gmail.com <<EOF
check f_sales high range.
EOF
else
echo "f_sales ok"
fi
exit 0
```

确保你不会无意中使用此类脚本向生产表添加数据。你必须仔细修改脚本，以匹配你的表和高值范围分区键列。

### 为分区键列实现 TIMESTAMP
如前所述，前一节中的示例为 `F_SALES` 表创建了 `D_DATE_ID` 列作为 `NUMBER` 数据类型，而不是 `DATE` 数据类型。与 `NUMBER` 数据类型相反，一些数据仓库架构师会主张为分区键使用 `DATE` 或 `TIMESTAMP` 数据类型。以下是一个使用 `DATE` 数据类型为 `D_DATE_DTT` 列创建 `F_SALES` 表的示例：

```sql
SQL> create table f_sales
(sales_amt  number
,d_date_dtt date
)
partition by range (d_date_dtt)(
partition p_2016 values less than (to_date('01-01-2017','dd-mm-yyyy')),
partition p_2017 values less than (to_date('01-01-2018','dd-mm-yyyy')),
partition p_max  values less than (maxvalue));
```

**提示**
使用 DD-MM-YYYY 日期格式可能比 01-MON-YYYY 等格式更可取。DD-MM-YYYY 格式避免了使用月份的字符名称（JAN、FEB 等），从而规避了不同字符集语言带来的问题。

如前面的代码所示，我建议你始终使用 `TO_DATE` 来明确指示 Oracle 如何解释日期。这样做也为任何支持数据库的人员提供了最低限度的文档。为分区键使用 `DATE` 数据类型与使用 `NUMBER` 字段一样有效。只需记住，设计数据仓库表的人可能对使用哪种技术有强烈的意见。

### 将分区置于表空间中
将分区放在其自身表空间中的好处，如今主要在于从其他表中识别出这些分区。如果每个分区都有自己的表空间，则备份、恢复以及设置联机/脱机状态的方式会得到简化，但即使分区不在自己的表空间中，表的联机操作和备份也是可用的。在某些情况下，甚至将表空间和分区设置为只读模式可能是有意义的，而拥有自己的表空间则允许这样做。

要理解为每个分区使用单独表空间的好处，首先考虑一个非分区表的场景。供参考，以下是本示例使用的 `CREATE TABLESPACE` 语句：

```sql
SQL> CREATE TABLESPACE p1_tbsp
DATAFILE '/u01/dbfile/o18c/p1_tbsp01.dbf' SIZE 100m
EXTENT MANAGEMENT LOCAL
UNIFORM SIZE 128K
SEGMENT SPACE MANAGEMENT AUTO;
```

此示例创建了一个分区表，但没有为分区指定表空间：

```sql
SQL> create table f_sales (
sales_amt number
,d_date_id number)
tablespace p1_tbsp
partition by range(d_date_id)(
partition y11 values less than (20120101)
,partition y12 values less than (20130101)
,partition y13 values less than (20140101));
```

图 12-1 说明了这种方法。
请注意，在这种情况下，所有分区都存储在同一个表空间中。

![../images/214899_3_En_12_Chapter/214899_3_En_12_Fig1_HTML.jpg](img/214899_3_En_12_Fig1_HTML.jpg)

**图 12-1** 仅有一个表空间的分区表

下一个示例将每个分区放在不同的表空间中：

```sql
SQL> create table f_sales (
sales_amt number
,d_date_id number)
tablespace p1_tbsp
partition by range(d_date_id)(
partition y11 values less than (20120101)
tablespace p1_tbsp
,partition y12 values less than (20130101)
tablespace p2_tbsp
,partition y13 values less than (20140101)
tablespace p3_tbsp);
```

现在，每个分区的数据物理上存储在它们自己的表空间和相应的数据文件中（见图 12-2）。

![../images/214899_3_En_12_Chapter/214899_3_En_12_Fig2_HTML.jpg](img/214899_3_En_12_Fig2_HTML.jpg)

**图 12-2** 存储在不同表空间中的分区

### 按列表分区
列表分区适用于对无序且不相关的数据集进行分区。例如，假设你有一个大表，并希望按州代码对其进行分区。为此，请使用 `CREATE TABLE` 语句的 `PARTITION BY LIST` 子句。此示例使用州代码创建三个基于列表的分区：

```sql
SQL> create table f_sales
(sales_amt  number
,d_date_id  number
,state_code varchar2(3))
partition by list (state_code)
( partition reg_west values ('AZ','CA','CO','MT','OR','ID','UT','NV')
,partition reg_mid  values ('IA','KS','MI','MN','MO','NE','OH','ND')
,partition reg_def  values (default));
```

列表分区表的分区键只能是一列。使用 `DEFAULT` 列表为不匹配列表中值的行指定一个分区。如果不指定 `DEFAULT` 列表，则当插入的行其值无法映射到已定义的分区时，将生成错误。运行此 SQL 语句可查看每个分区的列表值：

```sql
SQL> select table_name, partition_name, high_value
from user_tab_partitions
where table_name = 'F_SALES'
order by 1;
```

以下是此示例的输出：

```
TABLE_NAME  PARTITION_NAME   HIGH_VALUE
----------- ---------------- ----------------------------------------------
F_SALES     REG_DEF          default
F_SALES     REG_MID          'IA', 'KS', 'MI', 'MN', 'MO', 'NE', 'OH', 'ND'
F_SALES     REG_WEST         'AZ', 'CA', 'CO', 'MT', 'OR', 'ID', 'UT', 'NV'
```


# Oracle 数据库分区技术详解

## `HIGH_VALUE` 列与数据类型
`HIGH_VALUE` 列显示为每个分区定义的列表值。该列是 `LONG` 数据类型。如果使用 SQL*Plus，可能需要将 `LONG` 变量的值设置得比默认值（80B）更高，以显示该列的全部内容：
```sql
SQL> set long 1000
```

## 哈希分区
有时，一个大表可能没有明显的列可以用来进行范围或列表分区。例如，假设你使用序列为表填充代理主键，并希望行数据基于唯一的主键均匀分布在各个分区中。这样做可能是因为没有其他列可供分区，或者你主要关心插入操作的效率。

哈希分区根据内部算法将行映射到分区，该算法将数据均匀分布在所有定义的分区上。你无法控制哈希算法或 Oracle 分布数据的方式。你只需指定所需的分区数量，Oracle 就会根据哈希键列将数据均匀划分。

**提示**：Oracle 强烈建议你为哈希分区数量使用 2 的幂（2、4、8、16 等）。这样做可以实现行在整个分区中的最优分布。

要创建基于哈希的分区，请使用 `CREATE TABLE` 语句的 `PARTITION BY HASH` 子句。以下示例创建了一个被划分为两个分区的表；每个分区都在其自己的表空间中创建：
```sql
SQL> create table f_sales(
    sales_id  number primary key
    ,sales_amt number)
    partition by hash(sales_id)
    partitions 2 store in(p1_tbsp, p2_tbsp);
```
当然，你需要修改细节（例如表空间名称）以匹配你的环境。或者，你可以省略 `STORE IN` 子句，Oracle 会将所有分区放置在你的默认表空间中。如果你想同时命名表空间和分区，可以按如下方式指定：
```sql
SQL> create table f_sales(
    sales_id  number primary key
    ,sales_amt number)
    partition by hash(sales_id)
    (partition p1 tablespace p1_tbsp
    ,partition p2 tablespace p2_tbsp);
```

哈希分区有一些有趣的性能影响。所有哈希键值相同的行都被插入到同一个分区中。这意味着插入操作特别高效，因为哈希算法确保数据在各分区中均匀分布。此外，如果你通常按特定键值进行选择，Oracle 只需访问一个分区即可检索这些行。但是，如果你按值的范围进行搜索，Oracle 很可能需要搜索每个分区来确定要检索哪些行。因此，范围搜索在哈希分区表中的性能可能较差。

## 混合不同的分区方法
Oracle 允许你使用多种策略对表进行分区（复合分区）。例如，假设你有一个表，希望按数字范围进行分区，但同时希望按区域列表对每个分区进行细分。以下示例正是如此：
```sql
SQL> create table f_sales(
    sales_amt  number
    ,state_code varchar2(3)
    ,d_date_id  number)
    partition by range(d_date_id)
    subpartition by list(state_code)
    (partition p2016 values less than (20170101)
        (subpartition p1_north values ('ID','OR')
        ,subpartition p1_south values ('AZ','NM')),
    partition p2017 values less than (20180101)
        (subpartition p2_north values ('ID','OR')
        ,subpartition p2_south values ('AZ','NM')));
```
你可以通过运行以下查询来查看子分区信息：
```sql
SQL> select table_name, partitioning_type, subpartitioning_type
    from user_part_tables
    where table_name = 'F_SALES';
```
这是一些示例输出：
```
TABLE_NAME  PARTITION SUBPARTIT
----------- --------- ---------
F_SALES     RANGE     LIST
```
运行下一个查询以查看有关子分区的信息：
```sql
SQL> select table_name, partition_name, subpartition_name
    from user_tab_subpartitions
    where table_name = 'F_SALES'
    order by table_name, partition_name;
```
这是一部分输出片段：
```
TABLE_NAME  PARTITION_NAME   SUBPARTITION_NAME
----------- ---------------- --------------------
F_SALES     P2016            P1_SOUTH
F_SALES     P2016            P1_NORTH
F_SALES     P2017            P2_SOUTH
F_SALES     P2017            P2_NORTH
```
复合分区可以实现为范围-哈希（自版本 8i 可用）和范围-列表（自版本 9i 可用）。以下是可用的复合分区策略：
*   **范围-哈希**：适用于可以通过某种随机键细分的范围，例如 `D_DATE_ID` 的范围，然后对 `SALES_ID` 进行哈希。
*   **范围-列表**：当一个范围可以进一步按列表分区时非常有用，例如 `D_DATE_ID` 的范围，然后对 `STATE_CODE` 进行列表分区。
*   **范围-范围**：适用于有两个不同的分区范围值时，例如 `D_DATE_ID` 和 `SHIP_DATE`。
*   **列表-范围**：当一个列表可以按范围进一步细分时非常有用，例如对 `STATE_CODE` 进行列表分区，然后对 `D_DATE_ID` 进行范围分区。
*   **列表-哈希**：适用于通过某种随机键进一步分区列表，例如对 `STATE_CODE` 进行列表分区，然后对 `SALES_ID` 进行哈希。
*   **列表-列表**：适用于一个列表可以通过另一个列表进一步划分，例如 `COUNTRY_CODE` 然后 `STATE_CODE`。
*   **哈希-哈希**：当一个哈希可以通过另一个唯一值进一步细分时非常有用，例如 `SALES_ID` 和 `CUSTOMER_ID`。
*   **哈希-列表**：当一个哈希可以按列表进一步分区时非常有用，例如对 `SALES_ID` 进行哈希，然后对 `STATE_CODE` 进行列表分区。
*   **哈希-范围**：当一个哈希可以按范围进一步分区时非常有用，例如对 `SALES_ID` 进行哈希，然后对 `SHIP_DATE` 进行范围分区。

如你所见，复合分区在数据分区方式上提供了极大的灵活性。

## 按需创建分区
你可以指示 Oracle 自动向范围分区表添加分区。此功能称为间隔分区。当插入的数据超过范围分区表的最大边界时，Oracle 会动态创建一个新分区。新添加的分区基于你指定的间隔（因此称为间隔分区）。

**提示**：可以将间隔视为你提供的一条规则，说明你希望如何创建未来的分区。

### 按日期添加年度分区
例如，假设你有一个范围分区表，并希望当插入的值高于为最高范围定义的最大值时，Oracle 自动添加一个分区。你可以使用 `CREATE TABLE` 语句的 `INTERVAL` 子句来指示 Oracle 自动向范围分区表的高端添加分区。以下示例创建了一个表，该表最初有一个分区，其高值范围为 `01-01-2018`：
```sql
SQL> create table f_sales(
    sales_amt  number
    ,d_date_dtt date)
    partition by range (d_date_dtt)
    interval(numtoyminterval(1, 'YEAR'))
    store in (p1_tbsp, p2_tbsp, p3_tbsp)
    (partition p1 values less than (to_date('01-01-2018','dd-mm-yyyy'))
    tablespace p1_tbsp);
```
第一个分区在 `P1_TBSP` 表空间中创建。随着 Oracle 添加分区，它会将新分区分配给 `STORE IN` 子句中定义的表空间（程序本应以轮询方式存储它们，但并非总是一致）。

**注意**：使用间隔分区时，你只能指定表中的单个键列，并且它必须是 `DATE` 或 `NUMBER` 数据类型。因为间隔是数学方式添加到这些数据类型的。你不能使用 `VARCHAR2`，因为你不能将数字添加到 `VARCHAR2` 数据类型。

# Oracle 自动分区技术详解

## 基于年份的间隔分区

此示例中的间隔为一年，由 `INTERVAL(NUMTOYMINTERVAL(1, 'YEAR'))` 子句指定。如果向表中插入一条 `D_DATE_DTT` 值大于或等于 2018-01-01 的记录，Oracle 会自动在表高端添加一个新分区。

可以通过运行以下 SQL 语句来检查分区的详细信息：

```sql
SQL> set lines 132
col table_name form a10
col partition_name form a9
col part_pos form 999
col interval form a10
col tablespace_name form a12
col high_value form a30
--
SQL> select table_name, partition_name, partition_position part_pos
  2  ,interval, tablespace_name, high_value
  3  from user_tab_partitions
  4  where table_name = 'F_SALES'
  5  order by table_name, partition_position;
```

以下是一些示例输出（列标题已缩短，`HIGH_VALUE` 列被截短以便输出适应页面）：

```
TABLE_NAME PARTITION   PART_POS INTERVAL     TABLESPACE_N HIGH_VALUE
---------  ---------  -------- ---------     ------------------------------
F_SALES    P1          1        NO           P1_TBSP      TO_DATE(' 2018-01-01 00:00:00'
```

接下来，在最高分区的高端值之上插入数据：

```sql
SQL> insert into f_sales values(1, sysdate+1000);
```

此时从 `USER_TAB_PARTITIONS` 选择数据的输出如下：

```
TABLE_NAME PARTITION  PART_POS INTERVAL TABLESPACE_N HIGH_VALUE
---------- ---------  --------   --------  -----------------------------
F_SALES    P1          1        NO               P1_TBSP TO_DATE(' 2018-01-01 00:00:00'
F_SALES    SYS_P3344   2        YES              P1_TBSP TO_DATE(' 2021-01-01 00:00:00'
```

一个分区被自动创建，其高端值为 2021-01-01。如果不喜欢 Oracle 给分区的名字，可以重命名它：

```sql
SQL> alter table f_sales rename partition sys_p3344 to p2;
```

注意当插入一个落在两个分区之间年份间隔的值时会发生什么：

```sql
SQL> insert into f_sales values(1, sysdate+500);
```

`USER_TAB_PARTITIONS` 视图显示又创建了一个分区，因为插入的值落在现有分区未包含的年份间隔内：

```
TABLE_NAME PARTITION PART_POS INTERVAL TABLESPACE_N  HIGH_VALUE
---------- ---------  -------- -------- ------------ ------------------------------
F_SALES    P1               1 NO       P1_TBSP       TO_DATE(' 2018-01-01 00:00:00'
F_SALES    SYS_P3345        2 YES      P3_TBSP       TO_DATE(' 2020-01-01 00:00:00'
F_SALES    SYS_P3344        3 YES      P1_TBSP       TO_DATE(' 2021-01-01 00:00:00'
```

> **注意**
> 如果 `INTERVAL` 分区中 `INTERVAL` 值为 `NO` 的分区超过一个，则除了最后一个之外的所有分区都可以被删除。换句话说，如果只有一个 `INTERVAL` 值为 `NO` 的分区，则该分区无法被删除。例如，尝试从前面示例的表中删除分区 `P1` 会产生 `ORA-14758` 错误。

## 基于周添加分区

也可以让 Oracle 按其他时间增量添加分区，例如按周；例如：

```sql
SQL> create table f_sales(
    sales_amt  number
    ,d_date_dtt date)
  2 partition by range (d_date_dtt)
  3 interval(numtodsinterval(7,'day'))
  4 store in (p1_tbsp, p2_tbsp, p3_tbsp)
  5 (partition p1 values less than (to_date('01-01-2018', 'dd-mm-yyyy'))
  6 tablespace p1_tbsp);
```

当数据插入到未来的周时，将自动创建新的周分区；例如：

```sql
SQL> insert into f_sales values(100, sysdate+7);
SQL> insert into f_sales values(200, sysdate+14);
```

运行此查询可验证是否已自动添加分区：

```sql
SQL> select table_name, partition_name, partition_position part_pos
  2  ,interval, tablespace_name, high_value
  3  from user_tab_partitions
  4  where table_name = 'F_SALES'
  5  order by table_name, partition_position;
```

以下是一些示例输出：

```
TABLE_NAME PARTITION PART_POS INTERVAL TABLESPACE_N HIGH_VALUE
---------- ---------  -------- -------- ------------ ------------------------------
F_SALES    P1               1 NO       P1_TBSP      TO_DATE(' 2018-01-01 00:00:00'
F_SALES    SYS_P3725        2 YES      P3_TBSP      TO_DATE(' 2018-01-15 00:00:00'
F_SALES    SYS_P3726        3 YES      P1_TBSP      TO_DATE(' 2018-01-22 00:00:00'
```

通过这种方式，Oracle 自动管理向表添加每周分区。

## 基于数字添加每日分区

回顾本章前面“按范围分区”一节中，如何使用数字字段（`D_DATE_ID`）作为基于范围的分区键。假设你想在使用此类分区策略的表中自动创建每日间隔分区。在这种情况下，你需要指定 `INTERVAL` 为 1。这是一个示例：

```sql
SQL> create table f_sales(
    sales_amt number
    ,d_date_id number)
  2 partition by range (d_date_id)
  3 interval(1)
  4 (partition p1 values less than (20180101));
```

只要你的应用程序能正确使用代表有效日期的数字，就不应该有任何问题。随着每天新数据的插入，会创建一个新的每日分区。例如，假设插入了这些数据：

```sql
SQL> insert into f_sales values(100,20180130);
SQL> insert into f_sales values(50,20180131);
```

两个对应的分区被自动创建。可以通过此查询进行验证：

```sql
select table_name, partition_name, partition_position part_pos
  ,interval, tablespace_name, high_value
from user_tab_partitions
where table_name = 'F_SALES'
order by table_name, partition_position;
```

相应的输出如下：

```
TABLE_NAME PARTITION PART_POS INTERVAL   TABLESPACE_N HIGH_VALUE
---------- ---------  -------- ---------- ------------ --------------------
F_SALES    P1               1 NO         USERS        20180101
F_SALES    SYS_P3383        2 YES        USERS        20180131
F_SALES    SYS_P3384        3 YES        USERS        20180132
```

请注意，`HIGH_VALUE` 列可能包含映射到无效日期的数字。这是预期的行为。例如，当使用 `D_DATE_ID` 为 20180131 创建分区时，Oracle 会将上边界计算为值 20180132。高端边界值定义为小于（不等于）插入到分区中的任何值。我在这里提到这一点的唯一原因是，如果你试图对 `HIGH_VALUE` 中的值执行日期运算，则需要考虑可能映射到无效日期的数字。在这个具体示例中，你需要从 `HIGH_VALUE` 中的值减去一才能获得有效日期。

如前面章节所示，基于数字的每日间隔分区方案工作良好。但是，如果你想按月或年创建间隔分区，这种方案效果就不那么好了。这是因为没有哪个数字能一致地代表一个月或一年。如果你需要基于日期的间隔功能，那么请使用日期，而不是基于数字的间隔功能。

## 使用引用分区匹配父表

可以使用 `PARTITION BY REFERENCE` 子句指定子表应与其父表以相同方式进行分区。这允许子表继承其父表的分区策略。父表的任何分区维护操作都会自动应用于子记录表。

> **注意**
> 在引用分区功能出现之前，你必须在子表中物理复制和维护父表列。这样做不仅需要更多磁盘空间，而且在维护分区时也是错误的来源。


## 示例：引用分区

例如，假设你想创建一个父表 `ORDERS` 和一个子表 `ORDER_ITEMS`，它们通过 `ORDER_ID` 列上的主键和外键约束相关联。父表 `ORDERS` 将基于 `ORDER_DATE` 列进行分区。

即使子表 `ORDER_ITEMS` 不包含 `ORDER_DATE` 列，你仍想知道是否可以对其进行分区，使得记录的分布方式与父表 `ORDERS` 相同。此示例创建了一个在 `ORDER_ID` 上有主键约束、在 `ORDER_DATE` 上有范围分区的父表：

```
SQL> create table orders(
order_id    number
,order_date  date
,constraint order_pk primary key(order_id))
partition by range(order_date)
(partition p16  values less than (to_date('01-01-2017','dd-mm-yyyy'))
,partition p17  values less than (to_date('01-01-2018','dd-mm-yyyy'))
,partition pmax values less than (maxvalue));
```

接下来，你创建子表 `ORDER_ITEMS`。通过将外键约束命名为被引用对象来进行分区：

```
SQL> create table order_items(
line_id  number
,order_id number not null
,sku      number
,quantity number
,constraint order_items_pk  primary key(line_id, order_id)
,constraint order_items_fk1 foreign key (order_id) references orders)
partition by reference (order_items_fk1);
```

注意，外键列 `ORDER_ID` 必须定义为 `NOT NULL`。外键列必须被启用并强制执行。

你可以通过以下查询检查分区键列：

```
SQL> select name, column_name, column_position
from user_part_key_columns
where name in ('ORDERS','ORDER_ITEMS');
```

以下是此示例的输出：

```
NAME                 COLUMN_NAME          COLUMN_POSITION
-------------------- -------------------- ---------------
ORDERS               ORDER_DATE                         1
ORDER_ITEMS          ORDER_ID                           1
```

注意，子表是通过 `ORDER_ID` 列分区的。这确保了子记录与父记录以相同的方式分区（因为子记录通过 `ORDER_ID` 键列与父记录相关）。

当你创建引用分区的子表时，如果没有显式命名子表分区，Oracle 默认会使用与父表相同的分区名来创建子表分区。此示例显式命名了子表的引用分区：

```
SQL> create table order_items(
line_id  number
,order_id number not null
,sku      number
,quantity number
,constraint order_items_pk  primary key(line_id, order_id)
,constraint order_items_fk1 foreign key (order_id) references orders)
partition by reference (order_items_fk1)
(partition c16
,partition c17
,partition cmax);
```

从 Oracle Database 12c 开始，你还可以指定间隔-引用分区策略。这允许为父表和子表自动创建分区。以下是此功能的表创建脚本：

```
SQL> create table orders(
order_id    number
,order_date  date
,constraint order_pk primary key(order_id))
partition by range(order_date)
interval(numtoyminterval(1, 'YEAR'))
(partition p1 values less than (to_date('01-01-2018','dd-mm-yyyy')));
--
SQL> create table order_items(
line_id  number
,order_id number not null
,sku      number
,quantity number
,constraint order_items_pk  primary key(line_id, order_id)
,constraint order_items_fk1 foreign key (order_id) references orders)
partition by reference (order_items_fk1);
```

插入一些示例数据将演示分区是如何自动创建的：

```
SQL> insert into orders values(1,sysdate);
SQL> insert into order_items values(10,1,123,1);
SQL> insert into orders values(2,sysdate+400);
SQL> insert into order_items values(20,2,456,1);
```

现在，运行此查询以验证分区详情：

```
SQL> select table_name, partition_name, partition_position part_pos
,interval, tablespace_name, high_value
from user_tab_partitions
where table_name IN ('ORDERS','ORDER_ITEMS')
order by table_name, partition_position;
```

以下是输出的一个片段：

```
TABLE_NAME  PARTITION PART_POS  INTERVAL   TABLESPACE_N HIGH_VALUE

ORDERS      P1            1 NO     USERS   TO_DATE(' 2018-01-01 00:00:00'
ORDERS      SYS_P3761     2 YES    USERS   TO_DATE(' 2019-01-01 00:00:00'
ORDERS      SYS_P3762     3 YES    USERS   TO_DATE(' 2020-01-01 00:00:00'
ORDER_ITEMS P1            1 NO     USERS
ORDER_ITEMS SYS_P3761     2 YES    USERS
ORDER_ITEMS SYS_P3762     3 YES    USERS
```

## 基于虚拟列分区

你可以基于一个虚拟列进行分区。（关于虚拟列的讨论，请参见第 7 章）。以下是一个创建名为 `EMP` 表的示例脚本，该表包含虚拟列 `COMMISSION`，并为该虚拟列创建了对应的范围分区：

```
SQL> create table emp (
emp_id   number
,salary   number
,comm_pct number
,commission generated always as (salary*comm_pct)
)
partition by range(commission)
(partition p1 values less than (1000)
,partition p2 values less than (2000)
,partition p3 values less than (maxvalue));
```

此策略允许你对一个不存储在表中、但动态计算的列进行分区。当业务需求要求对表中未物理存储的列进行分区时，虚拟列分区是合适的。虚拟列背后的表达式可以是复杂的计算、返回列字符串的子集、组合列值等等，可能性是无限的。

例如，你可能有一个十字符的字符串列，其中前两位数字代表区域，后八位数字代表特定位置（这是一个糟糕的设计，但确实存在）。在这种情况下，从业务角度看，基于该列的前两位数字（按区域）进行分区可能是有意义的。

## 让应用程序控制分区

你可能遇到一种罕见的场景，希望插入记录到表中的应用程序显式控制它将数据插入到哪个分区。你可以使用 `PARTITION BY SYSTEM` 子句和 `INSERT` 语句来指定将数据插入到哪个分区。下一个示例创建了一个系统分区表，包含三个分区：

```
SQL> create table apps
(app_id number
,app_amnt number)
partition by system
(partition p1
,partition p2
,partition p3);
```

向此表插入数据时，必须指定一个分区。下一行代码将记录插入到分区 `P1` 中：

```
SQL> insert into apps partition(p1) values(1,100);
```

当执行更新或删除时，如果不指定分区，Oracle 会扫描系统分区表的所有分区以查找相关行。因此，在更新和删除时应指定分区以避免性能不佳。

系统分区表在需要显式控制记录插入到哪个分区的特殊情况下很有用。这允许你的应用程序代码管理记录在各分区间的分布。我建议你仅在无法使用 Oracle 的其他分区机制来满足业务需求时才使用此特性。

### 维护分区

使用分区时，你最终将需要执行某种维护操作。例如，你可能需要移动、交换、重命名、拆分、合并或删除分区。本节将描述各种分区维护任务。

### 查看分区元数据



当您维护分区时，查看有关分区对象的元数据信息会很有帮助。Oracle 提供了多种包含分区表和索引信息的数据字典视图。表 12-3 概述了这些视图。

**表 12-3：包含分区信息的数据字典视图**

| 视图 | 包含的信息 |
| --- | --- |
| `DBA/ALL/USER_PART_TABLES` | 显示分区表信息 |
| `DBA/ALL/USER_TAB_PARTITIONS` | 包含关于单个表分区的信息 |
| `DBA/ALL/USER_TAB_SUBPARTITIONS` | 显示关于存储和统计信息的子分区级别表信息 |
| `DBA/ALL/USER_PART_KEY_COLUMNS` | 显示分区键列 |
| `DBA/ALL/USER_SUBPART_KEY_COLUMNS` | 包含子分区键列 |
| `DBA/ALL/USER_PART_COL_STATISTICS` | 显示列级别统计信息 |
| `DBA/ALL/USER_SUBPART_COL_STATISTICS` | 显示子分区级别统计信息 |
| `DBA/ALL/USER_PART_HISTOGRAMS` | 包含分区的直方图信息 |
| `DBA/ALL/USER_SUBPART_HISTOGRAMS` | 显示子分区的直方图信息 |
| `DBA/ALL/USER_PART_INDEXES` | 显示分区索引信息 |
| `DBA/ALL/USER_IND_PARTITIONS` | 包含关于单个索引分区的信息 |
| `DBA/ALL/USER_IND_SUBPARTITIONS` | 显示子分区级别索引信息 |
| `DBA/ALL/USER_SUBPARTITION_TEMPLATES` | 显示子分区模板信息 |

请记住，`DBA`级别的视图包含数据库中所有分区对象的数据，`ALL`级别显示当前连接用户有权访问的分区信息，而`USER`级别提供有关当前连接用户拥有的分区对象的信息。
您会经常使用的两个视图是 `DBA_PART_TABLES` 和 `DBA_TAB_PARTITIONS`。`DBA_PART_TABLES` 视图包含表级别的分区信息，例如分区方法和默认存储设置。`DBA_TAB_PARTITIONS` 视图提供关于单个表分区的信息，例如分区名称和单个分区的存储设置。

## 移动分区

假设您创建了一个列表分区表，如下所示：

```sql
SQL> create table f_sales
(sales_amt  number
,d_date_id  number
,state_code varchar2(20))
partition by list (state_code)
( partition reg_west values ('AZ','CA','CO','MT','OR','ID','UT','NV')
,partition reg_mid  values ('IA','KS','MI','MN','MO','NE','OH','ND')
,partition reg_rest values (default));
```

同时，对于这个分区表，您决定创建一个本地分区索引，如下所示：

```sql
SQL> create index f_sales_lidx1 on f_sales(state_code) local;
```

您还决定创建一个非分区全局索引，如下所示：

```sql
SQL> create index f_sales_gidx1 on f_sales(d_date_id) global;
```

并且，您创建了一个全局分区索引列：

```sql
SQL> create index f_sales_gidx2 on f_sales(sales_amt)
global partition by range(sales_amt)
(partition pg1 values less than (25)
,partition pg2 values less than (50)
,partition pg3 values less than (maxvalue));
```

后来，您决定将一个分区移动到特定的表空间。在这种情况下，您可以使用 `ALTER TABLE...MOVE PARTITION` 语句来重新定位表分区。此示例将 `REG_WEST` 分区移动到一个新的表空间：

```sql
SQL> alter table f_sales move partition reg_west tablespace p1_tbsp;
```

将分区移动到不同的表空间是一个相当简单的操作。但是，每当您这样做时，请确保检查与该表关联的任何索引的状态：

```sql
SQL> select b.table_name, a.index_name, a.partition_name
,a.status, b.locality
from user_ind_partitions a
,user_part_indexes   b
where a.index_name=b.index_name
and table_name = 'F_SALES';
```

以下是一些示例输出：



您必须重建任何不可用的索引。与手动重建索引不同，在移动分区时，您可以通过 `UPDATE INDEXES` 子句指定重建与之关联的索引：

```sql
SQL> alter table f_sales move partition reg_west tablespace p1_tbsp update indexes;
```

从 Oracle Database 12c 开始，在移动分区时，您可以通过 `ONLINE` 子句指定更新所有索引：

```sql
SQL> alter table f_sales move partition reg_west online tablespace p1_tbsp;
```

上面这行代码指示 Oracle 在移动操作期间维护所有索引。

## 自动移动更新的行

默认情况下，Oracle 不允许您通过将分区键设置为超出该行当前分区的值来更新行。例如，下面的语句将分区键列（`D_DATE_ID`）更新为一个值，这将导致该行需要存在于不同的分区中：

```sql
SQL> update f_sales set d_date_id = 20130901 where d_date_id = 20120201;
```

您将收到以下错误：

```
ORA-14402: updating partition key column would cause a partition change
```

在此场景下，请使用 `ALTER TABLE` 语句的 `ENABLE ROW MOVEMENT` 子句，以允许对分区键进行会导致其所属分区发生改变的更新。对于此示例，首先修改 `F_SALES` 表以启用行移动：

```sql
SQL> alter table f_sales enable row movement;
```

现在，您应该能够将分区键更新为一个值，从而将该行移动到不同的段。您可以通过查询 `USER_TABLES` 视图的 `ROW_MOVEMENT` 列来验证行移动是否已启用：

```sql
SQL> select row_movement from user_tables where table_name='F_SALES';
```

您应该看到值 `ENABLED`：

```
ROW_MOVE
--------
ENABLED
```

要禁用行移动，请使用 `DISABLE ROW MOVEMENT` 子句：

```sql
SQL> alter table f_sales disable row movement;
```

## 对现有表进行分区

您可能有一个变得相当大的非分区表，并希望对其进行分区。将非分区表转换为分区表有几种方法。表 12-4 列出了各种技术的优缺点。

表 12-4 转换非分区表的方法

| 转换方法 | 优点 | 缺点 |
| :--- | :--- | :--- |
| `ALTER TABLE ... MODIFY PARTITION BY ... ONLINE` | 修改表和添加分区是 `ONLINE` 操作 | 索引方面需要额外考虑；可使用 `UPDATE INDEXES` |
| `CREATE <new_part_tab> AS SELECT * FROM <old_tab>` | 简单；可使用 `NOLOGGING` 和 `PARALLEL` 选项；直接路径加载 | 需要新表和旧表的空间 |
| `INSERT /*+ APPEND */ INTO <new_part_tab> SELECT * FROM <old_tab>` | 快速；简单；直接路径加载 | 需要新表和旧表的空间 |
| 数据泵 `EXPDP` 旧表；`IMPDP` 新表（如果使用较旧版本的 Oracle 则使用 `EXP IMP`） | 快速；所需空间较少；处理授权、权限等。可以按分区并通过筛选条件进行加载。 | 更复杂，因为您需要使用实用程序 |
| 创建分区表 `<new_part_tab>`；与 `<old_tab>` 交换分区 | 潜在停机时间更少 | 步骤多；复杂 |
| 使用 `DBMS_REDEFINITION` 包 | 内联转换现有表 | 步骤多；复杂 |
| 创建 CSV 文件或外部表；使用 SQL*Loader 加载 `<new_part_tab>` | 可以按分区进行加载。 | 步骤多；复杂 |



## 对现有表进行分区

如表 12-4 所示，对现有表进行分区最简单的方法之一是使用 `ALTER TABLE`；此功能自 Oracle 12c 起可用。执行此修改后，需要验证索引是本地索引还是全局索引，并且可能需要为新的策略创建索引，但此表操作是一个 `ONLINE` 操作，允许在 `ALTER` 表操作完成期间使用该表。

要将非分区表转换为分区表，需要列出分区策略以及分区。

```
SQL> alter table f_sales modify
partition by range (d_date_id)
(partition p2012 values less than(20130101),
partition p2013 values less than(20140101),
partition pmax values less than(maxvalue))
online;
```

另一种简单的方法是创建一个新的分区表，并从旧表加载数据。接下来列出了所需的步骤：

1.  如果这是一个活动生产数据库中的表，您应该为表安排一些停机时间，以确保在迁移过程中没有活动的事务发生。
2.  使用 `CREATE TABLE <new table> AS SELECT * FROM <old table>` 从旧表创建一个新的、分区的表。
3.  删除或重命名旧表。
4.  将步骤 2 中创建的表重命名为已删除/重命名表的名称。

例如，假设本章到目前为止使用的 `F_SALES` 表是作为一个非分区表创建的。以下语句创建了一个新的分区表，从旧的非分区表中获取数据：

```
SQL> create table f_sales_new
partition by range (d_date_id)
(partition p2012 values less than(20130101),
partition p2013 values less than(20140101),
partition pmax values less than(maxvalue))
nologging
as select * from f_sales;
```

现在，您可以删除（或重命名）旧的非分区表，并将新的分区表重命名为旧表的名称。在用 `PURGE` 选项删除旧表之前，请确保您不再需要它，因为这会永久删除该表：

```
SQL> drop table f_sales purge;
SQL> rename f_sales_new to f_sales;
```

最后，为新表创建任何必需的约束、授权、索引和统计信息。您现在应该拥有一个替换了旧的非分区表的分区表。

对于最后一步，如果原始表包含许多约束、授权和索引，您可能希望使用 Data Pump `expd` 导出原始表（不含数据）。然后，在创建新表后，使用 Data Pump `impdp` 为新表创建约束、授权和索引。同时考虑为新创建的表生成新的统计信息。另一种思路是，如果表有潜力增长到非常大，可以创建它来处理分区，然后根据需要创建额外的分区。即使在执行 `ALTER` 表命令时，也应进行测试，以确认使用几个分区创建分区，然后如下一节所讨论的那样添加和修改额外的分区。

## 添加分区

有时很难预测最初应该为表建立多少个分区。一个典型的例子是一个范围分区表，在创建时没有定义 `MAXVALUE` 分区。您创建了一个包含足够未来两年分区的分区表，然后您就忘记了这个表。未来的某个时候，应用程序用户报告抛出了以下消息：

```
ORA-14400: inserted partition key does not map to any partition
```

> 提示
> 考虑使用间隔分区，当超过上限时，它可以使 Oracle 自动添加范围分区。

### 范围

对于范围分区表，如果表的最高界限未使用 `MAXVALUE` 定义，您可以使用 `ALTER TABLE...ADD PARTITION` 语句在表的高端添加一个分区。如果您不确定当前的上限是什么，可以查询数据字典：

```
SQL> select table_name, partition_name, high_value
from user_tab_partitions
where table_name = UPPER('&&tab_name')
order by table_name, partition_name;
```

以下示例向范围分区表的高端添加了一个分区：

```
SQL> alter table f_sales add
partition p_2018 values less than (20190101) tablespace p18_tbsp;
```

从 Oracle Database 12c 开始，您可以同时添加多个分区；例如，

```
SQL> alter table f_sales add
partition p_2018 values less than (20190101) tablespace p18_tbsp
,partition p_2019 values less than (20200101) tablespace p19_tbsp;
```

> 注意
> 如果您有一个范围分区表，其高范围界限由 `MAXVALUE` 界定，则无法添加分区。在这种情况下，您必须拆分现有分区（参见本章后面的“拆分分区”一节）。

### 列表

对于列表分区表，只有在没有定义 `DEFAULT` 分区的情况下，才能添加新分区。下一个示例向列表分区表添加了一个分区：

```
SQL> alter table f_sales add partition reg_east values('GA');
```

从 Oracle Database 12c 开始，您可以用一条语句添加多个分区：

```
SQL> alter table f_sales add partition reg_mid_east values('TN'),
partition reg_north values('NY');
```

### 哈希

如果您有一个哈希分区表，请使用 `ADD PARTITION` 子句，如下所示，来添加一个分区：

```
SQL> alter table f_sales add partition p3 update indexes;
```

> 注意
> 当向哈希分区表添加分区时，如果您没有指定 `UPDATE INDEXES` 子句，则必须重建任何全局索引。此外，您必须为新添加的分区重建任何本地索引。

向哈希分区表添加分区后，请始终检查索引以确保它们都仍具有 `VALID` 状态：

```
SQL> select b.table_name, a.index_name, a.partition_name, a.status, b.locality
from user_ind_partitions a
,user_part_indexes   b
where a.index_name=b.index_name
and table_name = upper('&&part_table');
```

同时检查任何全局非分区索引的状态：

```
SQL> select index_name, status
from user_indexes
where table_name = upper('&&part_table');
```

我强烈建议您始终在非生产数据库中测试维护操作，以确定任何不可预见的副作用。

## 与现有表交换分区

交换分区是一种将新数据透明加载到大型分区表中的常用技术。该技术涉及获取一个独立表，并将其与现有分区（在已分区的表中）交换，允许您添加完全加载的新分区（及关联的索引），而不会影响对表中其他分区的操作的可用性或性能。

这个简单的例子说明了该过程。假设您有一个范围分区表，创建如下：

```
SQL> create table f_sales
(sales_amt number
,d_date_id number)
partition by range (d_date_id)
(partition p_2016 values less than (20170101),
partition p_2017 values less than (20180101),
partition p_2018 values less than (20190101));
```

您还在 `D_DATE_ID` 列上创建了一个本地位图索引：

```
SQL> create bitmap index d_date_id_fk1 on
f_sales(d_date_id) local;
```

现在，向表中添加一个新分区以存储新数据：

```
SQL> alter table f_sales add partition p_2019
values less than(20200101);
```

接下来，创建一个暂存表，并插入属于新添加分区值范围的数据：

```
SQL> create table workpart(
sales_amt number
,d_date_id number);
--
SQL> insert into workpart values(100,20190201);
SQL> insert into workpart values(120,20190507);
```

然后，在 `WORKPART` 表上创建一个与 `F_SALES` 上的位图索引结构相匹配的位图索引：

```
SQL> create bitmap index d_date_id_fk2
on workpart(d_date_id);
```

现在，交换 `WORKPART` 表与 `P_2019` 分区：


# Oracle 分区管理操作指南

## 分区交换

执行以下 SQL 命令交换分区：

```sql
SQL> alter table f_sales
exchange partition p_2019
with table workpart
including indexes without validation;
```

快速查询 `F_SALES` 表可以验证分区是否成功交换：

```sql
SQL> select * from f_sales partition(p_2019);
```

查询结果如下：

```
SALES_AMT  D_DATE_ID
---------  ---------
100   20190201
120   20190507
```

此查询显示所有索引是否仍可用：

```sql
SQL> select index_name, partition_name, status from user_ind_partitions;
```

您还可以验证新分区是否已创建本地索引段：

```sql
SQL> select segment_name, segment_type, partition_name
from user_segments
where segment_name IN('F_SALES','D_DATE_ID_FK1');
```

交换分区是一个极其强大的功能。它允许您将现有表中的分区转换为独立表，同时将一个独立表（可以在分区交换操作前完全填充）转换为分区表的一部分。当您交换分区时，Oracle 只需更新数据字典中的条目即可完成交换。

当您使用 `WITHOUT VALIDATION` 子句交换分区时，您指示 Oracle 不验证传入分区（或子分区）中的行是否符合定义的范围。这样做的好处是使交换操作非常快速，因为 Oracle 仅通过更新数据字典中的指针来完成交换操作。如果您使用 `WITHOUT VALIDATION`，则需要确保数据的准确性。

如果为分区表定义了主键，则被交换的表必须具有相同的主键结构。如果存在主键，`WITHOUT VALIDATION` 子句不会阻止 Oracle 强制执行唯一约束。

## 重命名分区

有时，您可能需要重命名表分区或索引分区。例如，您可能希望在删除分区前重命名它（以确保它未被使用）。此外，您可能希望重命名对象以符合标准。在这些场景中，请酌情使用 `ALTER TABLE` 或 `ALTER INDEX` 语句。

此示例使用 `ALTER TABLE` 语句重命名表分区：

```sql
SQL> alter table f_sales rename partition p_2018 to part_2018;
```

下一行代码使用 `ALTER INDEX` 语句重命名索引分区：

```sql
SQL> alter index d_date_id_fk1 rename partition p_2018 to part_2018;
```

您可以查询数据字典以验证有关重命名对象的信息。此查询显示分区表的名称：

```sql
SQL> select table_name, partition_name, tablespace_name
from user_tab_partitions;
```

类似地，此查询显示分区索引信息：

```sql
SQL> select index_name, partition_name, status
,high_value, tablespace_name
from user_ind_partitions;
```

## 拆分分区

假设您识别出一个包含过多行的分区，并希望将其拆分为两个分区。使用 `ALTER TABLE...SPLIT PARTITION` 语句来拆分现有分区。以下示例拆分了一个范围分区表中的分区：

```sql
SQL> alter table f_sales split partition p_2018 at (20180601)
into (partition p_2018_a, partition p_2018)
update indexes;
```

如果您不指定 `UPDATE INDEXES`，本地索引将变为 `UNUSABLE`，并且您需要重建与拆分分区相关的任何本地索引以及任何全局索引。您可以使用以下 SQL 验证分区索引的状态：

```sql
SQL> select index_name, partition_name, status from user_ind_partitions;
```

下一个示例拆分了一个列表分区。首先，这里是 `CREATE TABLE` 语句，它显示了列表分区最初是如何定义的：

```sql
SQL> create table f_sales
(sales_amt  number
,d_date_id  number
,state_code varchar2(3))
partition by list (state_code)
( partition reg_west values ('AZ','CA','CO','MT','OR','ID','UT','NV')
,partition reg_mid  values ('IA','KS','MI','MN','MO','NE','OH','ND')
,partition reg_rest values (default));
```

接下来，拆分 `REG_MID` 分区：

```sql
SQL> alter table f_sales split partition reg_mid values ('IA','KS','MI','MN') into
(partition reg_mid_a,
partition reg_mid_b)
update indexes;
```

`REG_MID_A` 分区现在包含值 IA、KS、MI 和 MN，而 `REG_MID_B` 被分配了剩余的值 MO、NE、OH 和 ND。

拆分分区操作允许您从单个分区创建两个新分区。每个新分区都有自己的段、物理属性和区段。与原始分区关联的段将被删除。

## 合并分区

创建分区时，有时很难预测分区最终将包含多少行。您可能有两个分区，其中包含的数据不足以保证它们作为单独的分区存在。在这种情况下，请使用 `ALTER TABLE...MERGE PARTITIONS` 语句来合并分区。

以下示例将两个分区合并为一个现有分区：

```sql
SQL> alter table f_sales merge partitions p_2017, p_2018 into partition p_2018;
```

在此示例中，分区按日期范围组织。您合并到的目标分区被定义为接受两个合并分区中较高范围的行。任何本地索引也会合并到新的单个分区中。

您可以通过查询数据字典来验证分区索引的状态：

```sql
SQL> select index_name, partition_name, tablespace_name, high_value,status
from user_ind_partitions
order by 1,2;
```

从 Oracle 18c 开始，现在可以使用 `ONLINE` 子句在线执行合并操作。这将允许在合并执行期间数据仍然可用，并且应使用 `UPDATE INDEXES` 子句来同时维护和重建任何关联的索引。

```sql
SQL> alter table f_sales merge partitions p_2017, p_2018 into partition p_2018
tablespace p2_tbsp
update indexes
online;
```

请记住，当您使用 `UPDATE INDEXES` 子句时，合并操作需要更长时间。如果您希望最小化合并操作的时间，请不要使用此子句。而是手动重建与合并分区关联的本地索引：

```sql
SQL> alter table f_sales modify partition p_2018 rebuild unusable local indexes;
```

您可以使用 `ALTER INDEX...REBUILD PARTITION` 语句重建全局索引的每个分区：

```sql
SQL> alter index f_glo_idx1 rebuild partition sys_p680;
SQL> alter index f_glo_idx1 rebuild partition sys_p681;
SQL> alter index f_glo_idx1 rebuild partition sys_p682;
```

您可以使用 `ALTER TABLE...MERGE PARTITIONS` 语句合并两个或更多分区。您合并到的分区的名称可以是您正在合并的分区之一的名称，也可以是一个全新的名称。

在合并两个（或更多）分区之前，请确保您合并到的目标分区在其表空间中有足够的空间来容纳所有合并后的行。如果没有足够的空间，您将收到表空间无法扩展到必要大小的错误。

### 删除分区

您偶尔需要删除分区。一种常见的情况是您有不再使用的旧数据，这意味着可以删除该分区。

首先，识别要删除的分区的名称。运行以下查询以列出当前连接用户的特定表的分区：

```sql
SQL> select segment_name, segment_type, partition_name
from user_segments
where segment_name = upper('&table_name');
```

接下来，使用 `ALTER TABLE...DROP PARTITION` 语句从表中删除分区。此示例从 `F_SALES` 表中删除了 `P_2018` 分区：


```markdown
## 重新划分标题层级

### 删除分区
当删除一个分区时，您需要重建任何全局索引。这可以在同一条 DDL 语句中完成，如下例所示：

```sql
SQL> alter table f_sales drop partition p_2018 update global indexes;
```

如果想删除子分区，请使用 `DROP SUBPARTITION` 子句：

```sql
SQL> alter table f_sales drop subpartition p2_south;
```

您可以查询 `USER_TAB_SUBPARTITIONS` 来确认子分区已被删除。

> **注意**：Oracle 不允许您删除组合分区表的所有子分区。每个分区必须至少有一个子分区。

删除分区后，无法执行“undrop”操作。因此，在执行此操作之前，请确保您处于正确的环境并且确实需要删除该分区。如果需要保留要删除分区中的数据，请将该分区与另一个分区合并，而不是删除它。
您不能从哈希分区表中删除分区。对于哈希分区表，您必须合并分区以移除一个。而且，您不能从引用分区表中显式删除分区。当父表分区被删除时，它也会从相应的子引用分区表中删除。

### 为分区生成统计信息
向分区加载大量数据后，您应该生成统计信息以反映新插入的数据。使用 `EXECUTE` 语句来运行 `DBMS_STATS` 包以为特定分区生成统计信息。在此示例中，所有者是 `STAR`，表是 `F_SALES`，正在分析的分区是 `P_2018`：

```sql
SQL> exec dbms_stats.gather_table_stats(ownname=>'MV_MAINT',-
tabname=>'F_SALES',-
partname=>'P_2018');
```

如果您正在处理大型分区，可能希望指定百分比采样大小和并行度，并为任何索引生成统计信息：

```sql
SQL> exec dbms_stats.gather_table_stats(ownname=>'MV_MAINT',-
tabname=>'F_SALES',-
partname=>'P_2018',-
estimate_percent=>dbms_stats.auto_sample_size,-
degree=>dbms_stats.auto_degree,-
cascade=>true);
```

对于分区表，您可以为单个分区或整个表生成统计信息。我建议您在分区数据发生重大变化时生成统计信息。您需要充分了解您的表和数据，以确定是否需要生成新的统计信息。

您可以指示 Oracle 在生成全局统计信息时仅扫描新添加的分区。此功能通过 `DBMS_STATS` 包启用：

```sql
SQL> exec DBMS_STATS.SET_TABLE_PREFS(user,'F_SALES','INCREMENTAL','TRUE');
```

您可以按如下方式验证表的首选项：

```sql
SQL> select dbms_stats.get_prefs('INCREMENTAL', tabname=>'F_SALES') from dual;
```

增量全局统计信息收集必须与 `DBMS_STATS.AUTO_SAMPLE_SIZE` 结合使用。这可以大大减少为大表新添加的分区收集增量统计信息所需的时间和资源。

### 从分区中删除行
您可以使用几种技术从分区中删除行。如果特定分区中的数据不再需要，请考虑删除该分区。如果想删除数据但保留分区原样，则可以对其进行截断或从中删除。截断分区会快速永久地删除数据。如果需要回滚删除记录的选项，则应使用删除（而不是截断）。截断和删除都在下文描述。

首先，确定要从中删除记录的分区的名称：

```sql
SQL> select segment_name, segment_type, partition_name
from user_segments
where partition_name is not null;
```

使用 `ALTER TABLE...TRUNCATE PARTITION` 语句从分区中删除所有记录。此示例截断 `F_SALES` 表中的一个分区：

```sql
SQL> alter table f_sales truncate partition p_2013;
```

前面的命令仅从指定的分区中删除数据，而不是整个表。还要记住，截断分区将使任何全局索引失效。您可以在发出 `TRUNCATE` 时更新全局索引，如下所示：

```sql
SQL> alter table f_sales truncate partition p_2013 update global indexes;
```

截断分区是删除大量数据的有效方法。但是，当您截断分区时，没有回滚机制。截断操作会永久删除分区中的数据。

如果需要回滚事务的选项，请使用 `DELETE` 语句：

```sql
SQL> delete from f_sales partition(p_2013);
```

这种方法的缺点是，如果有数百万条记录，`DELETE` 操作可能需要很长时间才能运行。此外，对于大量记录，`DELETE` 会生成大量回滚信息。这可能会导致争用资源的其他 SQL 语句出现性能问题。

### 操作分区内的数据
如果需要在一个分区内选择或操作数据，请在 SQL 语句中指定分区名称。例如，您可以从特定分区中选择行，如下所示：

```sql
SQL> select * from f_sales partition (p_2013);
```

如果想从两个（或多个）分区中选择，请使用 `UNION` 子句：

```sql
SQL> select * from f_sales partition (p_2013)
Union all
select * from f_sales partition (p_2014);
```

如果您是开发人员，并且无法访问数据字典来查看有哪些分区可用，则可以使用 `SELECT...PARTITION FOR <partition_key_value>` 语法。使用这种新语法，您提供一个分区键值，Oracle 会确定该键值所属的分区并返回该分区中的行；例如，

```sql
SQL> select * from f_sales partition for (20130202);
```

您还可以更新和删除分区行。此示例更新分区中的一个列：

```sql
SQL> update f_sales partition(p_2013) set sales_amt=200;
```

您可以将 `PARTITION FOR <partition_key_value>` 语法用于更新、删除和截断操作；例如，

```sql
SQL> update f_sales partition for (20130202) set sales_amt=200;
```

> **注意**：有关删除和截断分区的示例，请参阅前面的部分“从分区中删除行”。

### 分区索引
在当今的大型数据库环境中，索引可能增长到难以管理的规模。分区索引提供与分区表相同的益处：改进的性能、可扩展性和可维护性。
您可以创建一个使用其表分区策略的索引（本地），也可以创建一个与其表分区方式不同的索引（全局）。这两种技术将在以下部分中描述。

#### 使索引跟随其表进行分区
在分区表上创建索引时，您可以选择将其设为 `LOCAL` 数据类型。本地分区索引的分区方式与分区表相同。每个表分区都有一个对应的索引，该索引仅包含该表分区的 `ROWID` 值和索引键值。换句话说，本地分区索引中的 `ROWID` 值仅指向相应表分区中的行。

以下示例说明了本地分区索引的概念。首先，创建一个只有两个分区的表：

```sql
SQL> create table f_sales (
sales_id number
,sales_amt number
,d_date_id number)
tablespace p1_tbsp
partition by range(d_date_id)(
partition y12 values less than (20130101)
tablespace p1_tbsp
,partition y13 values less than (20140101)
tablespace p2_tbsp);
```

并且，假设向表中插入了五条记录，其中三条记录插入到分区 `Y12`，两条记录插入到分区 `Y13`：
```


SQL 插入示例：
```
SQL> insert into f_sales values(1,20,20120322);
SQL> insert into f_sales values(2,33,20120507);
SQL> insert into f_sales values(3,72,20120101);
SQL> insert into f_sales values(4,12,20130322);
SQL> insert into f_sales values(5,98,20130507);
```

## 本地索引

接下来，使用 `CREATE INDEX` 语句的 `LOCAL` 子句在分区表上创建本地索引。此示例在 `F_SALES` 表的 `D_DATE_ID` 列上创建了一个本地索引：
```
SQL> create index f_sales_fk1 on f_sales(d_date_id) local;
```

运行以下查询以查看有关分区索引的信息：
```
SQL> select index_name, table_name, partitioning_type
from user_part_indexes
where table_name = 'F_SALES';
```

示例输出如下：
```
INDEX_NAME                     TABLE_NAME PARTITION
------------------------------ ---------- ---------
F_SALES_FK1                    F_SALES    RANGE
```

现在，查询 `USER_IND_PARTITIONS` 表以查看有关本地分区索引的信息：
```
SQL> select index_name, partition_name, tablespace_name
from user_ind_partitions
where index_name = 'F_SALES_FK1';
```

请注意，已为表的每个分区创建了一个索引分区，并且索引是在与表分区相同的表空间中创建的：
```
INDEX_NAME           PARTITION_NAME       TABLESPACE_NAME
-------------------- -------------------- ---------------
F_SALES_FK1          Y12                  P1_TBSP
F_SALES_FK1          Y13                  P2_TBSP
```

图 12-3 概念性地展示了本地管理索引的构造方式。
![../images/214899_3_En_12_Chapter/214899_3_En_12_Fig3_HTML.jpg](img/214899_3_En_12_Fig3_HTML.jpg)
**图 12-3 本地管理索引的架构**

如果希望将本地索引分区创建在与表分区不同的表空间（或表空间组）中，请在创建索引时指定表空间：
```
SQL> create index f_sales_fk1 on f_sales(d_date_id) local
(partition y12 tablespace users
,partition y13 tablespace users);
```

查询 `USER_IND_PARTITIONS` 现在显示索引分区已创建在与表分区表空间分离的表空间中：
```
INDEX_NAME           PARTITION_NAME       TABLESPACE_NAME
-------------------- -------------------- ---------------
F_SALES_FK1          Y12                  USERS
F_SALES_FK1          Y13                  USERS
```

如果在构建本地分区索引时指定了分区信息，则分区数必须与构建索引的表上的分区数匹配。Oracle 会自动保持本地索引分区与表分区同步。您无法显式地向本地索引添加分区或从中删除分区。当您添加或删除表分区时，Oracle 会自动为本地索引执行相应的工作。Oracle 管理本地索引分区，无论本地索引如何分配给表空间。

本地索引在数据仓库和 DSS 环境中很常见。如果您经常使用分区列进行查询，本地索引是合适的。这种方法允许 Oracle 使用适当的索引和表分区来快速检索数据。

有两种类型的本地索引：本地前缀索引和本地非前缀索引。本地前缀索引是索引最左边的列与表分区键匹配的索引。本节前面的示例是一个本地前缀索引，因为其最左边的列 (`D_DATE_ID`) 也是表的分区键。

本地非前缀索引是其最左边的列不与用于分区相应表的分区键匹配的索引。例如，这是一个本地非前缀索引：
```
SQL> create index f_sales_idx1 on f_sales(sales_id) local;
```

该索引使用 `SALES_ID` 列进行分区，该列不是表的分区键，因此是一个非前缀索引。您可以通过查询 `USER_PART_INDEXES` 中的 `ALIGNMENT` 列来验证索引是否被视为前缀索引：
```
SQL> select index_name, table_name, alignment, locality
from user_part_indexes
where table_name = 'F_SALES';
```

示例输出如下：
```
INDEX_NAME           TABLE_NAME           ALIGNMENT    LOCALI
-------------------- -------------------- ------------ ------
F_SALES_FK1          F_SALES              PREFIXED     LOCAL
F_SALES_IDX1         F_SALES              NON_PREFIXED LOCAL
```

您可能会好奇为什么存在前缀和非前缀之间的区别。一个非前缀的本地索引在其索引定义中不包含分区键作为前导列。这可能会影响性能，因为访问非前缀索引的范围扫描可能需要搜索每一个索引分区。如果分区数量很大，这可能导致性能低下。

您可以通过将分区键列包含在索引的前导列中，选择将所有本地索引创建为前缀索引。例如，您可以将 `F_SALES_IDX2` 索引创建为前缀索引：
```
SQL> create index f_sales_idx2 on f_sales(d_date_id, sales_id) local;
```

前缀索引是否优于非前缀索引？这取决于您如何查询表。您必须为您使用的查询生成执行计划，并检查前缀索引是否比非前缀索引更能利用分区裁剪（消除要搜索的分区）。同时请记住，多列的本地前缀索引比本地非前缀索引消耗更多的空间和资源。

#### 全局索引

### 与表分区不同的索引分区
与其基表分区方式不同的索引称为全局索引。全局索引中的一个条目可以指向其基表的任何分区。您可以在任何类型的分区表上创建全局索引。
您可以创建范围分区或基于哈希的全局索引。使用关键字 `GLOBAL` 来指定该索引是使用与其对应表不同的分区策略构建的。在创建范围分区全局索引时，必须始终指定一个 `MAXVALUE`。

以下示例创建了一个基于范围的全局索引：
```
SQL> create index f_sales_gidx1 on f_sales(sales_amt)
global partition by range(sales_amt)
(partition pg1 values less than (25)
,partition pg2 values less than (50)
,partition pg3 values less than (maxvalue));
```

图 12-4 显示了使用全局索引时，索引的分区策略与表的分区策略并不一致。
![../images/214899_3_En_12_Chapter/214899_3_En_12_Fig4_HTML.jpg](img/214899_3_En_12_Fig4_HTML.jpg)
**图 12-4 全局索引的架构**

另一种类型的全局分区索引是基于哈希的。此示例创建了一个哈希分区全局索引：
```
SQL> create index f_sales_gidx2 on f_sales(sales_id)
global partition by hash(sales_id) partitions 3;
```

一般来说，全局索引比本地索引更难维护。我建议您尽量避免使用全局索引，并尽可能使用本地索引。

全局索引没有自动维护（而本地索引有）。对于全局索引，您负责添加和删除索引分区。此外，对基础分区表的许多维护操作要求重建全局索引分区。对堆组织表执行的以下操作会使全局索引失效：
*   `ADD (HASH)`
*   `COALESCE (HASH)`
*   `DROP`
*   `EXCHANGE`
*   `MERGE`
*   `MOVE`
*   `SPLIT`
*   `TRUNCATE`


## 使用 UPDATE INDEXES 子句

在执行维护操作时，请考虑使用 `UPDATE INDEXES` 子句。这样做可以在操作期间保持全局索引可用，并避免了重建的需要。使用 `UPDATE INDEXES` 的缺点是维护操作耗时更长，因为索引在操作期间需要被维护。

全局索引对于通过索引检索少量行的查询非常有用。在这些情况下，Oracle 可以消除（修剪）任何不必要的索引分区并高效地检索数据。例如，全局范围分区索引在 OLTP 环境中非常有用，因为在这种环境中你需要快速访问单个记录。

## 部分索引

从 Oracle Database 12c 开始，你可以指定索引分区最初以不可用状态创建。如果你有预创建的分区，但尚未有映射到未来日期的范围分区的数据，你可能希望这样做——其思路是你将在分区加载后（在未来的某个日期）构建索引。

你通过 `INDEXING ON|OFF` 子句来控制本地索引是否以可用状态创建。以下是一个示例，默认指定索引分区将不可用，除非显式开启：

```sql
SQL> create table f_sales (
sales_id number
,sales_amt number
,d_date_id number
)
indexing off
partition by range (d_date_id)
(partition p1 values less than (20170101)  indexing on,
partition p2 values less than (20180101)  indexing on,
partition p3 values less than (20190101)  indexing on,
partition p4 values less than (20200101)  indexing off);
```

接下来，在表上创建一个本地分区索引，指定应使用部分索引功能：

```sql
SQL> create index f_sales_lidx1 on f_sales(d_date_id)
local indexing partial;
```

你可以通过此查询验证哪些分区是可用（或不可用）的：

```sql
SQL> select a.index_name, a.partition_name, a.tablespace_name, a.status
from user_ind_partitions a, user_indexes b
where b.table_name = 'F_SALES'
and a.index_name = b.index_name;
```

这是此示例的一些示例输出：

```
INDEX_NAME           PARTITION_ TABLESPACE_NAME STATUS
-------------------- ---------- --------------- --------
F_SALES_LIDX1        P1         USERS           USABLE
F_SALES_LIDX1        P2         USERS           USABLE
F_SALES_LIDX1        P3         USERS           USABLE
F_SALES_LIDX1        P4         USERS           UNUSABLE
```

通过这种方式，你可以控制在向分区插入数据时是否维护索引。你可能最初不希望索引分区以可用状态创建，因为它会减慢数据的批量加载。在这种情况下，你会先加载数据，然后通过重建使其可用：

```sql
SQL> alter index f_sales_lidx1 rebuild partition p4;
```

## 分区修剪

SQL 查询中的分区修剪专门通过分区键访问表。Oracle 只搜索包含查询所需数据的分区（并且不访问任何不包含此类数据的分区——可以说是修剪了它们）。

例如，假设一个分区表定义如下：

```sql
SQL> create table f_sales (
sales_id  number
,sales_amt number
,d_date_id number)
tablespace p1_tbsp
partition by range(d_date_id)(
partition y17 values less than (20180101)
tablespace p1_tbsp
,partition y18 values less than (20190101)
tablespace p2_tbsp
,partition y19 values less than (20200101)
tablespace p3_tbsp);
```

此外，你在分区键列上创建一个本地索引：

```sql
SQL> create index f_sales_fk1 on f_sales(d_date_id) local;
```

然后，插入一些示例数据：

```sql
SQL> insert into f_sales values(1,100,20170202);
SQL> insert into f_sales values(2,200,20180202);
SQL> insert into f_sales values(3,300,20190202);
```

为了说明分区修剪的过程，启用 autotrace 工具：

```sql
SQL> set autotrace trace explain;
```

现在，执行一个基于分区键访问行的 SQL 语句：

```sql
SQL> select sales_amt from f_sales where d_date_id = '20180202';
```

Autotrace 会显示执行计划。为了使输出整齐地适应页面，已删除部分列：

```
| Id  | Operation                           | Name        | Pstart| Pstop |
|---  | ---                                 | ---         | ---   | ---   |
|   0 | SELECT STATEMENT                    |             |       |       |
|   1 |  PARTITION RANGE SINGLE             |             |     2 |     2 |
|   2 |   TABLE ACCESS BY LOCAL INDEX ROWID BATCHED | F_SALES     |     2 |     2 |
|*  3 |    INDEX RANGE SCAN                 | F_SALES_FK1 |     2 |     2 |
```

在此输出中，`Pstart` 显示起始访问分区是分区 2。`Pstop` 显示最后访问的分区是分区 2。在此示例中，分区 2 是用于检索数据的唯一分区；表中的其他分区根本未被查询访问。

如果执行的查询未使用分区键，则访问所有分区；例如，

```sql
SQL> select * from f_sales;
```

这是对应的执行计划：

```
| Id  | Operation           | Name    |    Rows| Pstart|  Pstop|
|---  | ---                 | ---     |     ---| ---   | ---   |
|   0 | SELECT STATEMENT    |         |     3  |       |       |
|   1 | PARTITION RANGE ALL |         |     3  |     1 |     3 |
|   2 | TABLE ACCESS FULL   | F_SALES |     3  |     1 |     3 |
```

注意在此输出中，起始分区是分区 1，停止分区是分区 3。这意味着分区 1 到 3 被此查询访问，没有进行分区修剪。这个例子虽然简单，但演示了分区修剪的概念。当你通过分区键访问表时，可以大幅减少 Oracle 需要检查和处理的行数。这对于能够修剪分区的查询具有巨大的性能优势。

## 修改分区策略

在 Oracle 18c 之前，如果你使用特定策略（如 hash）创建了分区表，然后想更改为其他策略，你将不得不使用迁移非分区表到分区表的方法之一来重新创建表。现在，将 hash 分区表更改为 composite range-hash 分区表，可以通过一个 `ALTER TABLE` 语句在线或离线执行。作为新策略前缀的索引将迁移到本地分区索引或自动转换为全局索引。

```sql
SQL> create table f_sales(
sales_id number
, sales_amt  number
,state_code varchar2(3)
,d_date_id  number)
partition by hash(sales_id);

SQL> alter table f_sales modify
partition by range(d_date_id)
subpartition by hash(sales_id)
subpartitions 8
(partition p2016 values less than (20170101),
partition p2017 values less than (20180101))
ONLINE
UPDATE INDEXES;
```


# 摘要

Oracle 提供了一个分区功能，这对于实现大型表和索引至关重要。分区对于构建高度可扩展和可维护的应用程序非常重要。此功能的工作原理是：在逻辑上创建一个对象（表或索引），但在物理上将该对象实现为几个独立的数据库段。分区对象允许您在分区的基础上进行构建、加载、维护和查询。维护操作，如删除、归档、更新和插入数据，变得易于管理，因为您只处理大型逻辑表的一小部分子集。

如果您在数据仓库环境中工作或处理大型数据库，您必须对分区概念有深入的了解。作为 DBA，您需要创建和维护分区对象。您必须就表分区策略以及在何处使用本地和全局索引提出建议。这些决策对系统的可用性和性能有巨大影响。许多新的分区功能支持在线操作，例如合并分区以及从非分区改为分区策略。这允许在操作这些大型表和按需管理分区策略时，对象和数据保持可用。

本书现在转向用于在不同环境之间复制和移动用户、对象和数据的工具。Oracle 的 Data Pump 和外部表功能即将介绍。

## 13. Data Pump

Data Pump 通常被描述为旧的`exp`/`imp`实用程序的升级版。这有点像将一部现代智能手机称为旧式旋转拨号座机的替代品。虽然旧的实用程序可靠且运行良好，但 Data Pump 在包含了这些功能的同时，为数据如何在环境之间提取和移动增添了全新的维度。

本章将帮助解释 Data Pump 如何使您当前的数据传输任务变得更轻松，并将展示如何以您未曾想过的方式移动信息和解决问题。

Data Pump 使您能够高效地备份、复制、保护和转换大量数据和元数据。您可以通过多种方式使用 Data Pump：

*   对整个数据库或数据子集执行时间点逻辑备份
*   为测试或开发复制整个数据库或数据子集
*   快速生成重新创建对象所需的 DDL
*   通过从旧版本导出并导入到新版本来升级数据库

有时，DBA 对旧的`exp`/`imp`实用程序有一种老式的依恋，因为 DBA 熟悉这些实用程序的语法，并且它们能快速完成工作。即使这些传统实用程序易于使用，您也应考虑今后使用 Data Pump。Data Pump 包含比旧的`exp`/`imp`实用程序更强大的功能：

*   处理大型数据集的性能，允许高效导出和导入千兆字节的数据
*   交互式命令行实用程序，允许您断开连接，然后稍后附加到活动的 Data Pump 作业，并监控作业进度
*   能够从远程数据库导出大量数据并直接导入到本地数据库，而无需创建转储文件
*   能够在导出到导入时动态更改模式、表空间、数据文件和存储设置
*   对对象和数据进行复杂过滤
*   用于执行可传输表空间导出
*   通过数据库控制的目录对象和数据目录
*   高级功能，如压缩和加密

本章的重点从 Data Pump 架构开始。还有其他在数据库之间移动数据的方法，如克隆和其他功能，将在后续章节中讨论。基本的导出和导入实用程序不应再使用。将数据写入客户端机器存在很高的安全风险，旧的导出和导入实用程序会这样做，而不是像 Data Pump 那样将文件写入服务器或安全的文件共享。需要采取安全措施以防止数据被放置在非安全区域。

### Data Pump 架构

Data Pump 包含以下组件：

*   `expdp`（Data Pump 导出实用程序）
*   `impdp`（Data Pump 导入实用程序）
*   `DBMS_DATAPUMP` PL/SQL 包（Data Pump 应用程序编程接口 [API]）
*   `DBMS_METADATA` PL/SQL 包（Data Pump 元数据 API）

`expdp`和`impdp`实用程序在导出和导入数据及元数据时使用`DBMS_DATAPUMP`和`DBMS_METADATA`内置 PL/SQL 包。`DBMS_DATAPUMP`包在数据库环境之间移动整个数据库或数据子集。`DBMS_METADATA`包导出和导入有关数据库对象的信息。

注意：您可以独立于`expdp`和`impdp`（在 SQL*Plus 中）调用`DBMS_DATAPUMP`和`DBMS_METADATA`包。`DBMS_DATAPUMP`可用于监控 Data Pump 作业。`DBMS_METADATA`对于检索 DDL 语句非常有用。更多详细信息，请参阅《Oracle Database PL/SQL Packages and Types Reference Guide》，该指南可从 Oracle 网站的技术网络区域下载（[`http://otn.oracle.com`](http://otn.oracle.com)）。

当您启动 Data Pump 导出或导入作业时，会在数据库服务器上启动一个主操作系统进程。此主进程名称的格式为`ora_dmNN_<SID>`。在 Linux/Unix 系统上，您可以使用`ps`命令从操作系统提示符查看此进程：
```
$ ps -ef | grep -v grep | grep ora_dm
oracle   14602     1  4 08:59 ?        00:00:03 ora_dm00_o12c
```

根据并行度和指定的工作量，还会启动一定数量的工作进程。如果未指定并行度，则只启动一个工作进程。主进程协调主进程和工作进程之间的工作。工作进程名称的格式为`ora_dwNN_<SID>`。

此外，当用户启动导出或导入作业时，数据库状态表会启动作业。此表仅在 Data Pump 作业期间存在。状态表的名称取决于您运行的作业类型。表的命名格式为`SYS_<OPERATION>_<JOB_MODE>_NN`，其中`OPERATION`是`EXPORT`或`IMPORT`。`JOB_MODE`可以是以下类型之一：

*   `FULL`
*   `SCHEMA`
*   `TABLE`
*   `TABLESPACE`
*   `TRANSPORTABLE`

例如，如果您正在导出一个模式，则会在您的帐户中创建一个名为`SYS_EXPORT_SCHEMA_NN`的表，其中`NN`是一个数字，使表名在用户模式中唯一。此状态表包含诸如导出/导入的对象、开始时间、已用时间、行数和错误计数等信息。状态表有超过 80 个列。

提示：Data Pump 状态表是在执行导出/导入的用户的默认永久表空间中创建的。因此，如果用户没有在默认表空间中创建表的权限，则 Data Pump 作业将失败，并出现`ORA-31633`错误。

# Data Pump 架构与基本操作

## 概述

`Data Pump` 的状态表在导出或导入作业成功完成后会被删除。如果您使用 `KILL_JOB` 交互式命令，主表也会被删除。如果您使用 `STOP_JOB` 交互式命令停止作业，该表不会被移除，可用于在您重启作业时使用。
如果作业异常终止，主表将被保留。如果您不计划重新启动作业，可以删除状态表。
当 `Data Pump` 运行时，它使用数据库目录对象来确定写入和读取转储文件及日志文件的位置。通常，您需要指定要让 `Data Pump` 使用的目录对象。如果您不指定目录对象，则将使用默认目录。默认目录路径由名为 `DATA_PUMP_DIR` 的数据目录对象定义。此目录对象在数据库首次创建时自动创建。在 Linux/Unix 系统上，此目录对象映射到 `ORACLE_HOME/rdbms/log` 目录。

## 架构组件

`Data Pump` 导出会创建导出文件和日志文件。导出文件包含被导出的对象。日志文件包含作业活动的记录。图 13-1 展示了与 `Data Pump` 导出作业相关的架构组件。

![../images/214899_3_En_13_Chapter/214899_3_En_13_Fig1_HTML.jpg](img/214899_3_En_13_Fig1_HTML.jpg)
*图 13-1 Data Pump 导出作业组件*

类似地，图 13-2 展示了 `Data Pump` 导入作业的架构组件。导出和导入之间的主要区别在于数据流的方向。导出将数据写出数据库，而导入将信息带入数据库。在本章中学习 `Data Pump` 示例和概念时，请参考这些图表。

![../images/214899_3_En_13_Chapter/214899_3_En_13_Fig2_HTML.jpg](img/214899_3_En_13_Fig2_HTML.jpg)
*图 13-2 Data Pump 导入作业组件*

对于每个 `Data Pump` 作业，您必须确保可以访问目录对象。导出和导入的基础知识将在接下来的几个小节中描述。

> **提示**：因为 `Data Pump` 内部使用 `PL/SQL` 来执行其工作，所以需要在共享池中有一些可用内存来保存 `PL/SQL` 包。如果共享池中没有足够的空间，`Data Pump` 将抛出 `ORA-04031: unable to allocate bytes of shared memory...` 错误并中止。如果收到此错误，请将数据库参数 `SHARED_POOL_SIZE` 至少设置为 50M。更多详情请参阅 MOS 注释 396940.1。

## 入门指南

现在您已经了解了 `Data Pump` 的架构，接下来是一个简单的示例，展示导出表、删除表，然后将表重新导入数据库所需的导出设置步骤。这将为本章涵盖的所有其他 `Data Pump` 任务奠定基础。

### 进行导出

运行 `Data Pump` 导出作业时需要进行少量设置。步骤如下：

1.  创建一个数据库目录对象，该对象指向您想要写入/读取 `Data Pump` 文件的操作系统目录。
2.  将目录对象的读写权限授予运行导出的数据库用户。
3.  在操作系统提示符下，运行 `expdp` 实用程序。

#### 步骤 1：创建数据库目录对象

在运行 `Data Pump` 作业之前，首先创建一个对应于磁盘上物理位置的数据库目录对象。此位置将用于保存导出和日志文件，并且应该是您知道有足够磁盘空间来容纳导出数据量的位置。

使用 `CREATE DIRECTORY` 命令来完成此任务。此示例创建一个名为 `dp_dir` 的目录，并指定它映射到磁盘上的 `/oradump` 物理位置：

```sql
SQL> create directory dp_dir as '/oradump';
```

要查看新创建目录的详细信息，请发出此查询：

```sql
SQL> select owner, directory_name, directory_path from dba_directories;
```

以下是一些示例输出：

```sql
OWNER      DIRECTORY_NAME  DIRECTORY_PATH
---------- --------------- --------------------
SYS        DP_DIR          /oradump
```

请记住，指定的目录路径必须物理存在于数据库服务器上。此外，该目录必须是 `oracle` 操作系统用户具有读/写访问权限的目录。最后，执行 `Data Pump` 操作的用户需要被授予对该目录对象的读/写访问权限（见步骤 2）。

如果在导出或导入时未指定 `DIRECTORY` 参数，`Data Pump` 将尝试使用默认数据库目录对象（如前所述，该对象映射到 `ORACLE_HOME/rdbms/log`）。不建议使用默认目录，原因如下：
*   如果您正在导出大量数据，最好在磁盘上有一个首选位置，在该位置您知道有足够的空间来满足您的磁盘空间要求。如果使用默认目录，您可能会无意中填满与 `ORACLE_HOME` 关联的挂载点，从而可能导致数据库挂起。
*   如果您授予非 DBA 用户进行导出的权限，您不希望他们在与 `ORACLE_HOME` 关联的位置创建大型转储文件，或者访问 `ORACLE_HOME` 目录。同样，您不希望与 `ORACLE_HOME` 关联的挂载点被填满，从而损害您的数据库。

#### 步骤 2：授予目录访问权限

您需要将数据库目录对象的权限授予想要使用 `Data Pump` 的用户。使用 `GRANT` 语句分配适当的权限。如果您希望用户能够从目录读取和写入，则必须授予安全访问权限。此示例将目录对象的访问权限授予名为 `MV_MAINT` 的用户：

```sql
SQL> grant read, write on directory dp_dir to mv_maint;
```

所有目录对象都归 `SYS` 用户所有。如果您使用的用户账户被授予了 `DBA` 角色，那么您对任何目录对象都拥有必要的读/写权限。权限可以为迁移、开发人员在测试中的数据刷新而授予，因此，授予对特定目录（可能是文件共享，而非数据库服务器本地目录）的权限是可能的。这也将数据安全地限制在只有特定用户组才能访问的目录中，并限制对其他模式和文件的访问。

### 旧版 exp 实用程序的安全问题

创建目录对象然后授予对物理存储位置的特定 I/O 访问权限背后的理念是，您可以更安全地管理哪些用户具有生成读写活动的能力，而这些活动通常是他们没有权限执行的。对于旧的 `exp` 实用程序，任何有权访问该工具的用户默认都有权向 `Oracle` 二进制文件所有者（通常是 `oracle`）可以访问的文件进行写入或读取。可以想象，一个恶意的非 `oracle` 操作系统用户可以尝试运行 `exp` 实用程序来故意覆盖关键的数据库文件。例如，以下命令可以由任何具有 `exp` 实用程序执行权限的非 `oracle` 操作系统用户运行：

```bash
$ exp heera/foo file=/oradata04/SCRKDV12/users01.dbf
```

但是，用户还必须在文件系统上具有写入文件的权限。如果没有文件系统上的权限，导出将会失败。

```bash
EXP-00028: failed to open /opt/oracle/x.dmp for write
Export file: expdat.dmp >
```


使用其他用户来运行这些实用工具，可能会允许未经授权的访问以从数据库提取数据，但如果该用户没有文件系统的写入权限，命令将会失败。
这里有一个重要的注意事项：**不要**将数据库软件目录 `ORACLE_HOME/bin` 用作客户端实用工具。应从 `ORACLE_HOME/bin` 中移除其他用户的执行权限，这样非 Oracle 用户就无法在数据库服务器上执行任何这些实用工具。
导出实用工具还允许用户直接导出到客户端机器，而不需要是配置了所需权限的文件系统。这一点在本章开头已经提及，也是不允许使用这些旧实用工具的主要原因之一。
为了防止此类问题，对于 Oracle 数据泵，你必须首先创建一个映射到特定目录的数据库对象目录，然后还要为该目录按用户分配读写权限。因此，数据泵没有旧的 `exp` 实用工具存在的安全问题。

## 步骤 3. 执行导出
当目录对象和权限设置到位后，你就可以使用数据泵从数据库导出信息了。本节的一个简单示例展示了如何导出一张表。本章后面的章节将详细描述导出数据的各种方式。这里的重点是完成一个示例，为理解后续更复杂的主题奠定基础。

以非 `SYS` 用户身份创建一个表，并填充一些数据：
```sql
SQL> create table inv(inv_id number);
SQL> insert into inv values (123);
```

接下来，以非 `SYS` 用户身份导出该表。此示例使用先前创建的、名为 `DP_DIR` 的目录。数据泵使用目录对象指定的目录路径作为磁盘上写入转储文件和日志文件的位置：
```bash
$ expdp mv_maint/foo directory=dp_dir tables=inv dumpfile=exp.dmp logfile=exp.log
```

`expdp` 实用工具在 `/oradump` 目录中创建一个名为 `exp.dmp` 的文件，其中包含重新创建 `INV` 表并以其导出时状态填充数据所需的信息。此外，一个名为 `exp.log` 的日志文件也会在 `/oradump` 目录中创建，其中包含与此导出作业相关的日志信息。
如果你不指定转储文件名，数据泵会创建一个名为 `expdat.dmp` 的文件。如果目录中已存在名为 `expdat.dmp` 的文件，数据泵将抛出错误。如果你不指定日志文件名，数据泵会创建一个名为 `export.log` 的文件。如果名为 `export.log` 的日志文件已存在，数据泵将覆盖它。

**提示：** 虽然可以使用 `SYS` 用户执行数据泵，但我不推荐这样做，原因有几个。首先，`SYS` 需要使用 `AS SYSDBA` 子句连接到数据库。这需要一个包含 `USERID` 参数并在连接字符串两边加上引号的数据泵参数文件。这很不方便。其次，`SYS` 拥有的大多数表无法导出（少数例外，例如 `AUD$`）。如果你尝试导出 `SYS` 拥有的表，数据泵会抛出 `ORA-39166` 错误并指出该表不存在。这很令人困惑。

`FULL` 导出不再导出系统方案 `SYS`、`ORDSYS` 或 `MDSYS`，即使使用 `SYS` 账户进行导出也是如此。

## 导入表
导出数据的关键原因之一是为了能够重新创建数据库对象。你可能希望将其作为备份策略的一部分，或者将数据复制到不同的数据库。数据泵导入使用导出转储文件作为输入，并重新创建导出文件中包含的数据库对象。导入的过程与导出类似：
1.  创建一个数据库目录对象，指向你想要从中读写数据泵文件的操作系统目录。
2.  将对该目录对象的读写权限授予执行导出或导入的数据库用户。
3.  在操作系统提示符下，运行 `impdp` 命令。

步骤 1 和 2 已在上一节“执行导出”中介绍，因此这里不再重复。

在运行导入作业之前，删除之前创建的 `INV` 表：
```sql
SQL> drop table inv purge;
```

接下来，根据导出的文件重新创建 `INV` 表：
```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=exp.dmp logfile=imp.log
```

现在，你应该已经重新创建了 `INV` 表，并以其导出时的状态填充了数据。现在是再次查看图 13-1 和 13-2 的好时机。确保你理解哪些文件是由 `expdp` 创建的，哪些文件被 `impdp` 使用。

## 使用参数文件
在许多情况下，将命令存储在文件中，然后在执行数据泵导出或导入时引用该文件，比在命令行中键入命令更好。使用参数文件使任务更易于重复，且不易出错。你可以将命令放入文件中一次，然后多次引用该文件。
此外，某些数据泵命令（例如 `FLASHBACK_TIME`）需要使用引号；在这些情况下，有时很难预测操作系统将如何解释它们。每当命令需要引号时，强烈建议使用参数文件。

要使用参数文件，首先创建一个包含用于控制作业行为的命令的操作系统文本文件。此示例使用 Linux/Unix 的 `vi` 命令创建一个名为 `exp.par` 的文本文件：
```bash
$ vi exp.par
```

现在，将以下命令放入 `exp.par` 文件中：
```text
userid=mv_maint/foo
directory=dp_dir
dumpfile=exp.dmp
logfile=exp.log
tables=inv
reuse_dumpfiles=y
```

接下来，导出操作通过 `PARFILE` 命令行选项引用参数文件：
```bash
$ expdp parfile=exp.par
```

数据泵处理文件中的参数，就如同它们是在命令行上键入的一样。如果你发现自己在重复键入相同的命令，或者使用需要引号的命令，或者两者兼有，那么考虑使用参数文件来提高效率。

**提示：** 不要将数据泵参数文件与数据库初始化参数文件混淆。数据泵参数文件指示数据泵以哪个用户身份连接到数据库、从哪个目录位置读写文件、在操作中包含哪些对象等等。相比之下，数据库参数文件在数据库启动时确定实例的特性。

## 细粒度导出和导入
回顾本章前面的“数据泵体系结构”一节，你可以调用导出/导入实用工具的几种不同模式。例如，你可以指示数据泵以以下模式导出/导入：
*   整个数据库
*   方案级别
*   表级别
*   表空间级别
*   可传输表空间级别

在深入研究数据泵的众多功能之前，讨论这些模式并确保你了解每种模式的工作方式是很有用的。这将为理解本章后面介绍的概念进一步奠定基础。

### 导出和导入整个数据库
当你导出整个数据库时，这有时被称为完全导出。在此模式下，生成的导出文件包含制作数据库副本所需的一切。除非受到过滤参数的限制（见本章后面的“过滤数据和对象”一节），完全导出包括以下内容：
*   重新创建表空间、用户、用户表、索引、约束、触发器、序列、存储的 PL/SQL 等所需的所有 DDL。
*   所有表数据（`SYS` 用户的表除外）。



# Oracle Data Pump 导出导入级别

## 全库级别

执行一次完整导出需要将 `FULL` 参数设置为 `Y`，并且必须由具有 DBA 权限或被授予了 `DATAPUMP_EXP_FULL_DATABASE` 角色的用户来完成。以下是一个导出整个数据库的示例：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=full.dmp logfile=full.log full=y
```

在导出执行期间，您应该在输出中看到此文本，表明正在进行全库级别的导出：

```
Starting "MV_MAINT"."SYS_EXPORT_FULL_01":
```

请注意，全库导出不会导出数据库中的所有内容：

*   `SYS` 模式的内容不会被导出（有一些例外，例如 `AUD$` 表）。试想一下，如果您能将一个数据库中 `SYS` 模式的内容导出并导入到另一个数据库会发生什么。`SYS` 模式的内容会覆盖内部数据字典表/视图，从而损坏数据库。因此，Data Pump 永远不会导出由 `SYS` 拥有的对象。

*   索引数据不会被导出，而是导出包含在后续导入期间重新创建索引所需 SQL 的索引 DDL。

一旦您有了完整导出，就可以使用其内容在原始数据库中重新创建对象（例如，在表被意外删除的情况下），或者将整个数据库或部分用户/表复制到不同的数据库。下一个示例假设转储文件已被复制**到**不同的数据库服务器，并现在用于将所有对象导入目标数据库：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=full.dmp logfile=fullimp.log full=y
```

**提示**
要启动全库导入，您必须具有 DBA 权限或被分配了 `DATAPUMP_IMP_FULL_DATABASE` 角色。

在屏幕上显示的输出中，您应该看到正在进行全库导入的指示：

```
Starting "MV_MAINT"."SYS_IMPORT_FULL_01":
```

运行全库导入数据库作业需要注意一些影响：

*   导入作业将首先尝试重新创建任何表空间。如果表空间已存在，或者表空间所依赖的目录路径不存在，则表空间创建语句将失败，导入作业将继续处理下一个任务。
*   接下来，导入作业将更改 `SYS` 和 `SYSTEM` 用户账户，使其包含导出时相同的密码。因此，从生产系统导入后，最好更改 `SYS` 和 `SYSTEM` 的密码，以反映新环境。
*   此外，导入作业随后将尝试创建导出文件中的任何用户。如果用户已存在，则会抛出错误，导入作业继续执行下一个任务。
*   用户将使用从原始数据库获取的相同密码导入。根据您的安全标准，您可能需要更改密码。
*   表将被重新创建。如果表已存在并包含数据，您必须指定希望导入作业如何处理此情况。您可以使导入作业跳过、追加、替换或截断该表（请参阅本章后面的“当对象已存在时导入”部分）。
*   在每张表被创建并填充数据后，会创建相关的索引。
*   如果有可用的统计信息，导入作业也会尝试导入。此外，对象授权也会被实例化。

如果一切运行顺利，最终结果将是一个在逻辑上与源数据库相同的数据库，包括表空间、用户、对象等。

## 模式级别

当您启动导出时，除非另有指定，Data Pump 会为运行导出作业的用户启动一个模式级别导出。用户级别导出常用于将一个或多个模式从一个环境复制到另一个环境。以下命令为 `MV_MAINT` 用户启动一个模式级别导出：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=mv_maint.dmp logfile=mv_maint.log
```

在屏幕上显示的输出中，您应该会看到一些文本表明已启动模式级别导出：

```
Starting "MV_MAINT"."SYS_EXPORT_SCHEMA_01"...
```

您还可以使用 `SCHEMAS` 参数为运行导出作业之外的其他用户启动模式级别导出。以下命令显示了多个用户的模式级别导出：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=user.dmp  schemas=heera,chaya
```

您可以通过引用使用模式级别导出创建的转储文件来启动模式级别导入：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=user.dmp
```

当您启动模式级别导入时，需要注意一些细节：

*   模式级别导出中不包含任何表空间。
*   导入作业会尝试重新创建转储文件中的任何用户。如果用户已存在，则会抛出错误，但导入作业会继续。
*   导入作业将根据导出的密码重置用户的密码。
*   由用户拥有的表将被导入并填充数据。如果表已存在，您必须通过 `TABLE_EXISTS_ACTION` 参数指示 Data Pump 如何处理。

您也可以在模式级别导入时使用全库导出的转储文件。为此，请指定您希望从全库导出中提取哪些模式：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=full.dmp schemas=heera,chaya
```

## 表级别

您可以通过 `TABLES` 参数指示 Data Pump 操作特定的表。例如，假设您想要导出：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=tab.dmp \
tables=heera.inv,heera.inv_items
```

您应该在输出中看到一些文本，表明正在进行表级别导出：

```
Starting "MV_MAINT"."SYS_EXPORT_TABLE_01...
```

类似地，您可以通过指定表级别创建的转储文件来启动表级别导入：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=tab.dmp
```

表级别导入仅尝试导入表和指定的数据。如果表已存在，则会抛出错误，导入作业继续。如果表已存在并包含数据，您必须指定希望导出作业如何处理此情况。您可以通过 `TABLE_EXISTS_ACTION` 参数使导入作业跳过、追加、替换或截断该表。

您也可以在使用全库导出或模式级别导出的转储文件时启动表级别导入。为此，请指定您希望从全库或模式级别导出中提取哪些表：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=full.dmp tables=heera.inv
```

## 表空间级别

表空间级别导出/导入操作特定表空间内包含的对象。此示例导出 `USERS` 表空间中包含的所有对象：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=tbsp.dmp tablespaces=users
```

输出中显示的文本应表明正在执行表空间级别导出：

```
Starting "MV_MAINT"."SYS_EXPORT_TABLESPACE_01"...
```

您可以通过指定使用表空间级别导出创建的导出文件来启动表空间级别导入：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=tbsp.dmp
```

您也可以通过使用全库导出但指定 `TABLESPACES` 参数来启动表空间级别导入：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=full.dmp tablespaces=users
```

表空间级别导入将尝试在表空间内创建任何表和索引。导入不会尝试重新创建表空间本身。由于 PDB 数据库将有自己的表空间，这可能是一种易于用于 PDB 导出的级别。

**注意**
还有一种可传输表空间模式导出。请参阅本章后面的“复制数据文件”部分。

## 传输数据



数据泵（Data Pump）的主要用途之一是将数据从一个数据库复制到另一个数据库。通常，源数据库和目标数据库位于相距数千英里的数据中心。数据泵提供了几个强大的功能来高效地复制数据：

*   网络链接
*   复制数据文件（可传输表空间）
*   外部表（参见章节 14）

使用网络链接可以让你执行导出操作并将其直接导入目标数据库，而无需创建转储文件。这是一种非常高效的数据移动方式。
Oracle 还提供了可传输表空间功能，允许你将数据文件从源数据库复制到目标数据库，然后使用数据泵传输相关的元数据。这两种技术将在后续章节中描述。

注意：关于使用外部表传输数据的讨论，请参见章节 14。

## 直接通过网络导出与导入

假设你有两个数据库环境——一个运行在 Solaris 服务器上的生产数据库，以及一个运行在 Linux 服务器上的测试数据库。你的老板向你提出以下要求：

*   在 Solaris 服务器上为生产数据库制作一个副本。
*   将该副本导入 Linux 服务器上的测试数据库。
*   在导入时更改模式（Schema）的名称，以符合测试数据库的命名标准。

首先，考虑一下使用旧的 `exp`/`imp` 工具将数据从一个数据库传输到另一个数据库所需的步骤。步骤大致如下：

1.  导出生产数据库（这会在数据库服务器上创建一个转储文件）。
2.  将转储文件复制到测试数据库服务器。
3.  将转储文件导入测试数据库。

你也可以使用数据泵执行相同的步骤。然而，数据泵提供了一种更高效、更透明的方法来执行这些步骤。如果你的生产数据库服务器和测试数据库服务器之间有直接的网络连接，你可以执行导出操作并直接将其导入目标数据库，而无需创建或复制任何转储文件。此外，你还可以在导入时动态重命名模式。另外，源数据库运行的操作系统与目标数据库不同也没有关系。

一个例子将有助于说明这是如何工作的。在这个例子中，生产数据库的用户是 `STAR2`、`CIA_APP` 和 `CIA_SEL`。你希望将这些用户移动到测试数据库中，并将他们重命名为 `STAR_JUL`、`CIA_APP_JUL` 和 `CIA_SEL_JUL`。
此任务需要以下步骤：

1.  在要导入到的测试数据库中创建用户。以下是一个在测试数据库中创建用户的示例脚本：

```
define star_user=star_jul
define star_user_pwd=star_jul_pwd
define cia_app_user=cia_app_jul
define cia_app_user_pwd=cia_app_jul_pwd
define cia_sel_user=cia_sel_jul
define cia_sel_user_pwd=cia_sel_jul_pwd
--
create user &&star_user identified by &&star_user_pwd;
grant connect,resource to &&star_user;
alter user &&star_user default tablespace dim_data;
--
create user &&cia_app_user identified by &&cia_app_user_pwd;
grant connect,resource to &&cia_app_user;
alter user &&cia_app_user default tablespace cia_data;
--
create user &&cia_sel_user identified by &&cia_app_user_pwd;
grant connect,resource to &&cia_app_user;
alter user &&cia_sel_user default tablespace cia_data;
```

2.  在你的测试数据库中，创建一个指向你的生产数据库的数据库链接。`CREATE DATABASE LINK` 语句中引用的远程用户必须在生产数据库中被授予 DBA 角色。以下是一个 `CREATE DATABASE LINK` 的示例脚本：

```
create database link dk
connect to darl identified by foobar
using 'dwdb1:1522/dwrep1';
```

3.  在你的测试数据库中，创建一个目录对象，指向你希望日志文件存放的位置：

```
SQL> create or replace directory engdev as '/orahome/oracle/ddl/engdev';
```

4.  在测试服务器上运行导入命令。此命令通过 `NETWORK_LINK` 参数引用远程数据库。该命令还指示数据泵将生产数据库用户名映射到测试数据库中已创建的新用户。

```
$ impdp darl/engdev directory=engdev network_link=dk \
schemas='STAR2,CIA_APP,CIA_SEL' \
remap_schema=STAR2:STAR_JUL,CIA_APP:CIA_APP_JUL,CIA_SEL:CIA_SEL_JUL
```

这种技术允许你在不同的数据库之间移动大量数据，而无需创建或复制任何转储文件或数据文件。你还可以通过 `REMAP_SCHEMA` 参数动态重命名模式。这是一个非常强大的数据泵功能，可让你快速高效地传输数据。

提示：在复制整个数据库时，也请考虑使用 RMAN 的 duplicate database 功能。

### 数据库链接与 NETWORK_LINK 参数

请不要将通过数据库链接连接到远程数据库进行的导出，与使用 `NETWORK_LINK` 参数进行的导出混淆。当通过数据库链接连接到远程数据库进行导出时，被导出的对象存在于远程数据库中，并且转储文件和日志文件会在远程服务器上由 `DIRECTORY` 参数指定的目录中创建。
例如，以下命令导出远程数据库中的对象并在远程服务器上创建文件：
`$ expdp mv_maint/foo@shrek2 directory=dp_dir dumpfile=sales.dmp`

相比之下，当你使用 `NETWORK_LINK` 参数导出时，你是在本地创建转储文件和日志文件，而被导出的数据库对象存在于远程数据库中；例如，

```
$ expdp mv_maint/foo network_link=shrek2 directory=dp_dir dumpfile=sales.dmp
```

## 复制数据文件

Oracle 提供了一种将数据文件从一个数据库复制到另一个数据库的机制，结合使用数据泵来传输相关的元数据。这被称为可传输表空间功能。此任务所需的时间取决于将数据文件复制到目标服务器所需的时间。这种技术适用于在 DSS 和数据仓库环境中移动数据。

提示：可传输表空间也可以（与 RMAN 的 `CONVERT TABLESPACE` 命令结合使用）用于将表空间移动到与主机平台不同的目标服务器。

按照以下步骤传输表空间：

1.  确保表空间是自包含的。以下是一些常见的违反自包含规则的情况：
    *   一个表空间中的索引不能指向不在要传输的表空间集中的另一个表空间中的表。
    *   在某个表空间中的表上定义了外键约束，该约束引用了不在要传输的表空间集中的另一个表空间中的表上的主键约束。

    运行以下检查，查看要传输的表空间集是否违反了任何自包含规则：

    ```
    SQL> exec dbms_tts.transport_set_check('INV_DATA,INV_INDEX', TRUE);
    ```

    现在，查看 Oracle 是否检测到任何违规：

    ```
    SQL> select * from transport_set_violations;
    ```

    如果没有任何违规，你应该看到：

    ```
    no rows selected
    ```

    如果你确实有违规，例如一个索引建立在某个未被传输的表空间中的表上，那么你将需要在该索引在要被传输的表空间中重建。

2.  将要传输的表空间设置为只读：

    ```
    SQL> alter tablespace inv_data read only;
    SQL> alter tablespace inv_index read only;
    ```

3.  使用数据泵导出要传输的表空间的元数据：


# 使用 Data Pump 进行可传输表空间迁移

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=trans.dmp \
transport_tablespaces=INV_DATA,INV_INDEX
```

4.  将 Data Pump 导出的转储文件复制到目标服务器。
5.  将数据文件复制到目标数据库。将文件放置在目标数据库服务器上你希望它们所在的目录中。文件名和目录路径必须与下一步中使用的导入命令相匹配。
6.  将元数据导入目标数据库。使用以下参数文件来导入正在传输的数据文件的元数据：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=trans.dmp
transport_datafiles=/ora01/dbfile/rcat/inv_data01.dbf,
/ora01/dbfile/rcat/inv_index01.dbf
```

如果一切顺利，你将看到一些指示成功的输出：

```
Job "MV_MAINT"."SYS_IMPORT_TRANSPORTABLE_01" successfully completed...
```

如果被传输的数据块大小与目标数据库不同，那么你必须修改初始化文件（或使用 `ALTER SYSTEM` 命令）并添加一个包含源数据库块大小的缓冲区池。例如，要添加一个 16KB 的缓冲区缓存，请在初始化文件中放置以下内容：

```
db_16k_cache_size=200M
```

你可以通过以下查询检查表空间的块大小：

```
SQL> select tablespace_name, block_size from dba_tablespaces;
```

可传输表空间机制允许你在数据库之间快速移动数据文件，即使数据库使用不同的块大小或不同的字节序格式。本节不讨论与可传输表空间相关的所有细节；本章的重点是展示如何使用 Data Pump 传输数据。有关可传输表空间的完整详细信息，请参阅可以免费从 Oracle 网站技术网络区域下载的 *Oracle Database Administrator's Guide* ([`http://otn.oracle.com`](http://otn.oracle.com))。

## 注意
要生成可传输表空间，你必须使用 Oracle 企业版。你可以使用 Oracle 的其他版本来导入可传输表空间。

## 用于操作存储的特性

Data Pump 包含许多灵活的特性，用于在导出和导入时操作表空间和数据文件。以下部分展示了在处理这些重要数据库对象时有用的 Data Pump 技术。

### 导出表空间元数据

有时，你可能需要复制一个环境——例如，将生产环境复制到测试环境。首要任务之一是复制表空间。为此，你可以使用 Data Pump 提取重新创建环境所需的表空间 DDL：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
full=y include=tablespace
```

`FULL` 参数指示 Data Pump 导出数据库中的所有内容。然而，当与 `INCLUDE` 一起使用时，Data Pump 仅导出该命令指定的对象。在此组合中，仅导出与表空间相关的元数据；数据文件内的任何数据都不包含在导出中。你可以在 `INCLUDE` 命令中添加 `CONTENT=METADATA_ONLY` 参数和值，但这是多余的。

现在，你可以使用 `SQLFILE` 参数来查看与导出的表空间关联的 DDL：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp sqlfile=tbsp.sql
```

当你使用 `SQLFILE` 参数时，不会导入任何内容。在此示例中，前面的命令仅创建一个名为 `tbsp.sql` 的文件，其中包含与表空间相关的 SQL 语句。你可以修改 DDL 并在目标数据库环境中运行它；或者，如果不需要更改，你可以通过将表空间导入目标数据库来直接使用转储文件。

### 指定不同的数据文件路径和名称

如前所述，你可以使用 `FULL` 和 `INCLUDE` 参数的组合来仅导出表空间元数据信息：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
full=y include=tablespace
```

如果你想在具有不同目录结构的单独数据库服务器上使用转储文件创建表空间，该怎么办？Data Pump 允许你在导入步骤中使用 `REMAP_DATAFILE` 参数更改数据文件目录路径和文件名。

例如，假设源数据文件存在于名为 `/ora03` 的挂载点上，但在要导入到的数据库上，挂载点以 `/ora01` 命名。以下是一个参数文件，它指定仅导入以字符串 `INV` 开头的表空间，并且其对应的数据文件名称应更改为反映新环境：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
full=y
include=tablespace:"like 'INV%'"
remap_datafile="'/ora03/dbfile/O18C/inv_data01.dbf':'/ora01/dbfile/O18C/tb1.dbf'"
remap_datafile="'/ora03/dbfile/O18C/inv_index01.dbf':'/ora01/dbfile/O18C/tb2.dbf'"
```

当 Data Pump 创建表空间时，对于任何匹配字符串第一部分（冒号 `:` 左边）的路径，该字符串将被替换为下一部分字符串（冒号右边）中的文本。

## 提示
当使用需要同时包含单引号和双引号的参数时，使用参数文件会得到可预测的行为。相反，如果你尝试在命令行上输入各种必需的引号，操作系统可能会解释并传递给 Data Pump 与你预期不同的内容。

### 导入到与原始表空间不同的表空间
你可能偶尔需要导出一个表，然后将其导入到不同的用户和不同的表空间中。源数据库可能与目标数据库不同，或者你可能只是尝试在同一数据库中的两个用户之间移动数据。你可以使用 `REMAP_SCHEMA` 和 `REMAP_TABLESPACE` 参数轻松处理此要求。

此示例重新映射了用户以及表空间。原始用户和表空间是 `HEERA` 和 `INV_DATA`。此命令将 `INV` 表导入 `CHAYA` 用户和 `DIM_DATA` 表空间：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp remap_schema=HEERA:CHAYA \
remap_tablespace=INV_DATA:DIM_DATA tables=heera.inv
```

`REMAP_TABLESPACE` 功能不会重新创建表空间。它仅指示 Data Pump 将对象放置在与它们导出时不同的表空间中。导入时，如果你放置对象的表空间不存在，Data Pump 会抛出错误。

### 更改数据文件的大小

你可以使用带有 `PCTSPACE` 选项的 `TRANSFORM` 参数在导入时更改数据文件的大小。假设你已经创建了一个仅包含表空间元数据的导出：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp full=y include=tablespace
```

现在，你希望在开发数据库中创建表空间名称中包含字符串 `DATA` 的表空间，但没有足够的磁盘空间来像源数据库中那样创建表空间。在这种情况下，你可以使用 `TRANSFORM` 参数来指定表空间按原始大小的百分比创建。

例如，如果你希望表空间按原始大小的 20% 创建，请发出以下命令：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
full=y
include=tablespace:"like '%DATA%'"
transform=pctspace:20
```


# 表空间与数据文件的创建

表空间创建时，其数据文件大小为原始大小的 20%。区分配大小也是其原始定义的 20%。这一点很重要，因为 Data Pump 不会检查存储属性是否满足数据文件的最低大小限制。这意味着如果计算出的更小尺寸违反了 Oracle 的最小尺寸（例如，统一区大小的五个块），则在导入期间将抛出错误。

此功能在将生产数据导出然后导入到更小的数据库时非常有用。在这些场景中，您可能会通过`SAMPLE`参数或`QUERY`参数（参见本章后面的“过滤数据和对象”部分）过滤掉部分生产数据。

## 更改段和存储属性

在导入时，您可以使用`TRANSFORM`参数来更改表的存储属性。此参数的通用语法如下：

```
TRANSFORM=transform_name:value[:object_type]
```

当您使用`SEGMENT_ATTRIBUTES:N`作为转换名称时，可以在导入期间删除以下段属性：

*   物理属性
*   存储属性
*   表空间
*   日志记录

当您导入到开发环境并且不希望表带有与生产数据库中相同的所有存储属性时，可能需要此功能。例如，在开发环境中，您可能只有一个表空间来存储所有表和索引，而在生产环境中，您将表和索引分布在多个表空间中。

以下是一个删除段属性的示例：

```
$ impdp mv_maint/foo directory=dp_dir  dumpfile=inv.dmp \
transform=segment_attributes:n
```

您可以使用`STORAGE:N`仅删除存储子句：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
transform=storage:n
```

## 过滤数据和对象

Data Pump 提供了大量机制来过滤数据和元数据。您可以通过以下方式影响 Data Pump 导出或导入中排除或包含的内容：

*   使用`QUERY`参数导出或导入数据的子集。
*   使用`SAMPLE`参数导出表中行的百分比。
*   使用`CONTENT`参数排除或包含数据和元数据。
*   使用`EXCLUDE`参数专门命名要排除的项目。
*   使用`INCLUDE`参数命名要包含的项目（从而排除未包含在列表中的其他非依赖项）。
*   使用诸如`SCHEMAS`之类的参数来指定您只需要数据库对象的子集（属于指定用户或用户的对象）。

以下各节描述了每种技术的示例。

> **注意：**
> 您不能同时使用`EXCLUDE`和`INCLUDE`。这些参数是互斥的。

### 指定查询

您可以使用`QUERY`参数指示 Data Pump 仅将满足特定条件的行写入转储文件。如果您正在重新创建测试环境并且只需要数据的子集，可能需要这样做。请记住，此技术不了解可能存在的任何外键约束，因此您不能在不考虑父-子关系的情况下盲目限制数据集。

`QUERY`参数包含查询的通用语法如下：

```
QUERY = [schema.][table_name:] query_clause
```

查询子句可以是任何有效的 SQL 子句。查询必须用双引号或单引号括起来。我建议使用双引号，因为您可能需要在查询中嵌入单引号来处理`VARCHAR2`数据。此外，您应使用参数文件，以避免操作系统对引号的解释产生混淆。

此示例使用参数文件并限制两个表的导出行。这是导出时使用的参数文件：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
tables=inv,reg
query=inv:"WHERE inv_desc='Book'"
query=reg:"WHERE reg_id <=20"
```

假设您将上述代码行放在名为`inv.par`的文件中。导出作业引用该参数文件，如下所示：

```
$ expdp parfile=inv.par
```

生成的转储文件仅包含由`QUERY`参数过滤的行。再次提醒，请注意任何父-子关系，并确保导出的内容不会在导入时违反任何约束。

您也可以在导入数据时指定查询。这是一个参数文件，用于限制导入到`INV`表中的行，基于`INV_ID`列：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
tables=inv,reg
query=inv:"WHERE inv_id > 10"
```

此文本被放置在名为`inv2.par`的文件中，并在导入期间被引用如下：

```
$ impdp parfile=inv2.par
```

`REG`表中的所有行都被导入。只有`INV`表中`INV_ID`大于 10 的行被导入。

### 导出数据的百分比

在导出时，`SAMPLE`参数指示 Data Pump 根据您提供的数字检索一定百分比的行。Data Pump 在导出时不跟踪父-子关系。因此，当您有通过外键约束链接的表并且尝试随机选择行的百分比时，此方法效果不佳。

此参数的通用语法如下：

```
SAMPLE=[[schema_name.]table_name:]sample_percent
```

例如，如果您想导出表中 10%的数据，请执行以下操作：

```
$ expdp mv_maint/foo directory=dp_dir tables=inv sample=10 dumpfile=inv.dmp
```

下一个示例导出两个表，但只导出`REG`表数据的 30%：

```
$ expdp mv_maint/foo directory=dp_dir tables=inv,reg sample=reg:30 dumpfile=inv.dmp
```

> **注意：**
> `SAMPLE`参数仅对导出有效。

### 从导出文件中排除对象

对于导出，`EXCLUDE`参数指示 Data Pump 不要导出指定的对象（而`INCLUDE`参数指示 Data Pump 仅在导出文件中包含特定对象）。`EXCLUDE`参数具有以下通用语法：

```
EXCLUDE=object_type[:name_clause] [, ...]
```

`OBJECT_TYPE`是数据库对象，例如`TABLE`或`INDEX`。要查看可以过滤哪些对象类型，请查看`DATABASE_EXPORT_OBJECTS`、`SCHEMA_EXPORT_OBJECTS`或`TABLE_EXPORT_OBJECTS`的`OBJECT_PATH`列。例如，如果您想查看可以过滤的模式级对象，请运行以下查询：

```
SELECT
object_path
FROM schema_export_objects
WHERE object_path NOT LIKE '%/%';
```

以下是输出片段：

```
OBJECT_PATH
--------------
STATISTICS
SYNONYM
SYSTEM_GRANT
TABLE
TABLESPACE_QUOTA
TRIGGER
```

`EXCLUDE`参数实例，例如，表示您正在导出表但希望排除索引和授权：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp tables=inv exclude=index,grant
```

您可以通过使用`NAME_CLAUSE`在更精细的级别进行过滤。`EXCLUDE`的`NAME_CLAUSE`选项允许您指定 SQL 过滤器。要排除名称以字符串“INV”开头的索引，您使用以下命令：

```
exclude=index:"LIKE 'INV%'"
```

前面的行要求您使用引号；在这些场景中，我建议您使用参数文件。这是一个包含`EXCLUDE`子句的参数文件：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
tables=inv
exclude=index:"LIKE 'INV%'"
```

`EXCLUDE`子句的某些方面可能看起来违反直觉。例如，考虑以下导出参数文件：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=sch.dmp
exclude=schema:"='HEERA'"
```


# 使用数据泵导出导入对象

如果尝试用这种方式排除用户，系统会抛出错误。
这是因为默认的导出模式是 `SCHEMA` 级别，而数据泵无法同时排除和包含一个模式。如果想从导出文件中排除某个用户，请指定 `FULL` 模式，并排除该用户：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=sch.dmp
exclude=schema:"='HEERA'"
full=y
```

## 排除统计信息

默认情况下，导出表对象时，相关的统计信息也会一并导出。你可以通过 `EXCLUDE` 参数来防止统计信息被导入。示例如下：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
tables=inv exclude=statistics
```

导入时，如果你试图从一个原本不包含统计信息的转储文件中排除统计信息，就会收到这个错误：

```
ORA-39168: Object path STATISTICS was not found.
```

如果导出的转储文件中的对象从未生成过统计信息，你也会收到此错误。如果迁移到不同环境，建议在导入后根据数据的新位置重新生成统计信息。

## 仅在导出文件中包含特定对象

使用 `INCLUDE` 参数可以在导出文件中仅包含特定的数据库对象。以下示例仅导出用户拥有的过程和函数：

```
$ expdp mv_maint/foo dumpfile=proc.dmp directory=dp_dir include=procedure,function
```

创建的 `proc.dmp` 文件只包含重新创建用户拥有的任何过程和函数所需的 DDL 语句。

使用 `INCLUDE` 时，也可以指定仅导出特定的 PL/SQL 对象：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=ss.dmp \
include=function:\"=\'IS_DATE\'\"
```

使用参数文件可以方便地捕获 PL/SQL 导出的具体细节。以下示例显示了用于导出特定对象的参数文件内容：

```
directory=dp_dir
dumpfile=ss.dmp
include=function:"='ISDATE'",procedure:"='DEPTREE_FILL'"
```

如果指定了一个不存在的对象，数据泵会抛出一个错误，但会继续执行导出操作：

```
ORA-39168: Object path FUNCTION was not found.
```

## 导出表、索引、约束和触发器 DDL

假设你想导出与数据库中表、索引、约束和触发器相关联的 DDL。为此，请使用 `FULL` 导出模式，指定 `CONTENT=METADATA_ONLY`，并且仅包含表：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=ddl.dmp \
content=metadata_only full=y include=table
```

导出对象时，数据泵也会导出任何依赖对象。因此，当导出一个表时，你也会得到与该表关联的索引、约束和触发器。

## 在导入时排除对象

通常，你可以使用与在导出中过滤对象相同的技术来从导入中排除对象。使用 `EXCLUDE` 参数来排除对象被导入。例如，要排除触发器和过程不被导入，请使用以下命令：

```
$ impdp mv_maint/foo dumpfile=inv.dmp directory=dp_dir exclude=TRIGGER,PROCEDURE
```

你可以通过添加 SQL 子句来进一步细化排除条件。例如，假设你不想导入以字母 `B` 开头的触发器。参数文件内容如下：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
schemas=HEERA
exclude=trigger:"like 'B%'"
```

## 在导入时包含对象

你可以使用 `INCLUDE` 参数来限制导入的内容。假设你有一个模式，你想从中导入以字母 `A` 开头的表。以下是参数文件：

```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
schemas=HEERA
include=table:"like 'A%'"
```

如果将上述文本放入名为 `h.par` 的文件中，则可以如下调用参数文件：

```
$ impdp parfile=h.par
```

在此示例中，`HEERA` 模式必须已存在。只有以字母 `A` 开头的表会被导入。

# 常见的数据泵任务

以下部分描述了你可以与数据泵一起使用的常见功能。其中许多功能是数据泵的标准功能，例如创建一致的导出，以及在导入的对象已存在于数据库中时采取行动。其他功能，例如压缩和加密，需要 Oracle 企业版或额外许可证，或两者兼需。在介绍数据泵相关元素时，我会指出这些要求（如果适用）。

## 估算导出作业的大小

如果你即将导出大量数据，可以在运行导出前估算数据泵创建的文件的大小。你可能想这样做是因为担心导出作业所需的空间量。

要估算大小，请使用 `ESTIMATE_ONLY` 参数。此示例估算整个数据库的导出文件大小：

```
$ expdp mv_maint/foo estimate_only=y full=y logfile=n
```

以下是输出片段：

```
Estimate in progress using BLOCKS method...
Total estimation using BLOCKS method: 6.75 GB
```

类似地，你可以指定模式名称来估算导出用户所需的空间大小：

```
$ expdp mv_maint/foo estimate_only=y schemas=star2 logfile=n
```

以下是一个估算两个表所需空间的示例：

```
$ expdp mv_maint/foo estimate_only=y tables=star2.f_configs,star2.f_installations \
logfile=n
```

## 列出转储文件的内容

数据泵有一个非常强大的方法来创建一个文件，其中包含导入作业运行时执行的所有 SQL 语句。数据泵使用 `DBMS_METADATA` 包来创建 DDL，你可以使用这些 DDL 在数据泵转储文件中重新创建对象。

使用数据泵导入的 `SQLFILE` 选项来列出数据泵导出文件的内容。此示例创建一个名为 `expfull.sql` 的文件，其中包含导入过程调用的 SQL 语句（文件放置在 `DPUMP_DIR2` 目录对象定义的目录中）：

```
$ impdp hr/hr DIRECTORY=dpump_dir1 DUMPFILE=expfull.dmp \
SQLFILE=dpump_dir2:expfull.sql
```

如果未指定单独的目录（例如前例中的 `dpump_dir2`），则 SQL 文件将写入 `DIRECTORY` 选项中指定的位置。

**提示**：你必须以具有 DBA 权限的用户或执行数据泵导出的模式身份运行前面的命令。否则，你将得到一个空的 SQL 文件，其中没有预期的 SQL 语句。

当你在导入时使用 `SQLFILE` 选项，`impdp` 进程不会导入任何数据；它只创建一个文件，其中包含导入过程将运行的 SQL 命令。出于以下原因，生成 SQL 文件有时很方便：

*   在运行导入前预览和验证 SQL 语句。
*   手动运行 SQL 以预创建数据库对象。
*   捕获重新创建数据库对象（用户、表、索引等）所需的 SQL。

关于最后一个要点，有时提交到源代码控制存储库的内容与实际应用到生产数据库的内容不符。此过程对于故障排除或记录数据库在某个时间点的状态非常方便。

## 克隆用户

假设你需要将用户的对象和数据移动到新数据库。作为迁移的一部分，你想重命名该用户。首先，创建一个包含要克隆用户的模式级导出文件。在此示例中，用户名为 `INV`：

```
$ expdp mv_maint/foo directory=dp_dir schemas=inv dumpfile=inv.dmp
```

现在，你可以使用数据泵导入来克隆该用户。如果想将用户移动到不同的数据库，请将转储文件复制到远程数据库，并使用 `REMAP_SCHEMA` 参数创建用户的副本。在此示例中，`INV` 用户被克隆为 `INV_DW` 用户：

```
$ impdp mv_maint/foo directory=dp_dir remap_schema=inv:inv_dw dumpfile=inv.dmp
```


此命令将 `INV` 用户中的所有结构和数据复制到 `INV_DW` 用户。生成的 `INV_DW` 用户在对象方面与 `INV` 用户完全相同。复制后的方案也包含与源方案相同的密码。

如果你只想将元数据从一个方案复制到另一个，请使用带有 `METADATA_ONLY` 选项的 `CONTENT` 参数：

```bash
$ impdp mv_maint/foo directory=dp_dir remap_schema=inv:inv_dw \
content=metadata_only dumpfile=inv.dmp
```

`REMAP_SCHEMA` 参数提供了一种高效复制方案（无论是否包含数据）的方法。在方案复制操作期间，如果你想更改对象所在的表空间，请同时使用 `REMAP_TABLESPACE` 参数。这允许你复制方案并将对象放置在与源对象不同的表空间中。

你也可以在不首先创建转储文件的情况下，将用户从一个数据库复制到另一个。为此，请使用 `NETWORK_LINK` 参数。有关直接从一个数据库复制数据到另一个的详细信息，请参阅本章前面的“直接通过网络导出和导入”部分。

## 创建一致的导出
一致的导出意味着导出文件中的所有数据在某个时间点或某个 SCN（系统变更号）是一致的。当你导出一个包含许多父子表的活动数据库时，应确保获得数据的一致快照。

你可以通过使用 `FLASHBACK_SCN` 或 `FLASHBACK_TIME` 参数来创建一致的导出。此示例使用 `FLASHBACK_SCN` 参数进行导出。要确定数据集当前的 SCN 值，请执行此查询：

```sql
SQL> select current_scn from v$database;
```

这是一些典型的输出：

```sql
CURRENT_SCN
-----------
    5715397
```

以下命令使用 `FLASHBACK_SCN` 参数对数据库进行一致的全量导出：

```bash
$ expdp mv_maint/foo directory=dp_dir full=y flashback_scn=5715397 \
dumpfile=full.dmp
```

之前的导出命令确保所有导出的数据与在指定 SCN 之前数据库中已提交的任何事务保持一致。当你使用 `FLASHBACK_SCN` 参数时，数据泵会确保导出文件中的数据与该指定 SCN 时刻一致。这意味着在指定 SCN 之后提交的任何事务都不包含在导出文件中。

**注意**
如果同时使用 `NETWORK_LINK` 和 `FLASHBACK_SCN` 参数，则导出将使用与数据库链接中引用的数据库一致的 SCN。

你也可以使用 `FLASHBACK_TIME` 来指定导出文件应基于指定时间的已提交事务创建。使用 `FLASHBACK_TIME` 时，Oracle 会确定与指定时间最匹配的 SCN，并使用该 SCN 生成一致的导出。使用 `FLASHBACK_TIME` 的语法如下：

```bash
FLASHBACK_TIME="TO_TIMESTAMP('24-jan-2013 07:03:00','dd-mon-yyyy hh24:mi:ss')"
```

对于某些操作系统，直接在命令行上出现的双引号必须用反斜杠（`\`）转义，因为操作系统将其视为特殊字符。因此，使用参数文件更为直接。以下是一个使用 `FLASHBACK_TIME` 的参数文件内容：

```par
directory=dp_dir
content=metadata_only
dumpfile=inv.dmp
flashback_time="to_timestamp('24-jan-2013 07:03:00','dd-mon-yyyy hh24:mi:ss')"
```

根据你的操作系统，上一个示例的命令行版本必须如下指定：

```bash
flashback_time=\"to_timestamp\(\'24-jan-2013 07:03:00\',
\'dd-mon-yyyy hh24:mi:ss\'\)\"
```

此行代码应在一行中指定。这里为了适应页面，将代码放在了两行上。

进行导出时，不能同时指定 `FLASHBACK_SCN` 和 `FLASHBACK_TIME`；这两个参数是互斥的。如果你尝试同时使用这两个参数，数据泵将抛出以下错误消息并停止导出作业：

```
ORA-39050: parameter FLASHBACK_TIME is incompatible with parameter FLASHBACK_SCN
```

## 对象已存在时的导入
在导出和导入数据时，你通常需要导入到已创建对象（表、索引等）的方案中。在这种情况下，你应该导入数据，但指示数据泵尽量不要创建已存在的对象。

你可以通过 `TABLE_EXISTS_ACTION` 和 `CONTENT` 参数来实现这一点。下一个示例通过 `TABLE_EXISTS_ACTION=APPEND` 选项指示数据泵将数据追加到任何已存在的表中。同时还使用了 `CONTENT=DATA_ONLY` 选项，它指示数据泵不运行任何 DDL 来创建对象（仅加载数据）：

```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
table_exists_action=append content=data_only
```

现有对象不会以任何方式被修改，转储文件中存在的任何新数据将被插入到表中。

你可能会想，如果只使用 `TABLE_EXISTS_ACTION` 选项而不与 `CONTENT` 选项结合会怎样：

```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
table_exists_action=append
```

唯一的区别是数据泵会尝试运行 DDL 命令来创建对象（如果它们存在）。这不会阻止作业运行，但你会在输出中看到一条错误消息，指示对象已存在。以下是上一个命令输出的一个片段：

```
Table "MV_MAINT"."INV" exists. Data will be appended ...
```

`TABLE_EXISTS_ACTION` 参数的默认值是 `SKIP`，除非你同时指定了 `CONTENT=DATA_ONLY` 参数。如果你使用 `CONTENT=DATA_ONLY`，那么 `TABLE_EXISTS_ACTION` 的默认值是 `APPEND`。

`TABLE_EXISTS_ACTION` 参数接受以下选项：
- `SKIP`（默认，如果未与 `CONTENT=DATA_ONLY` 结合使用）
- `APPEND`（默认，如果与 `CONTENT=DATA_ONLY` 结合使用）
- `REPLACE`
- `TRUNCATE`

`SKIP` 选项告诉数据泵如果对象存在则不处理。`APPEND` 选项指示数据泵不删除现有数据，而是在不修改任何现有数据的情况下向表中添加数据。`REPLACE` 选项指示数据泵删除并重新创建对象；当 `CONTENT` 参数与 `DATA_ONLY` 选项一起使用时，此参数无效。`TRUNCATE` 参数告诉数据泵通过 `TRUNCATE` 语句删除表中的行。

`CONTENT` 参数接受以下选项：
- `ALL`（默认）
- `DATA_ONLY`
- `METADATA_ONLY`

`ALL` 选项指示数据泵加载转储文件中包含的数据和元数据；这是默认行为。`DATA_ONLY` 选项告诉数据泵仅将表数据加载到现有表中；不创建任何数据库对象。`METADATA_ONLY` 选项仅创建对象；不加载数据。

## 重命名表
你可以在导入操作期间选择重命名表。在导入时重命名表可能有很多原因。例如，目标方案中可能有一个表与你要导入的表同名。你可以通过使用 `REMAP_TABLE` 参数在导入时重命名表。此示例将表从 `HEERA` 用户的 `INV` 表导入到 `HEERA` 用户的 `INVEN` 表：

```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp tables=heera.inv \
remap_table=heera.inv:inven
```

以下是重命名表的通用语法：

```bash
REMAP_TABLE=[schema.]old_tablename[.partition]:new_tablename
```

请注意，此语法不允许你将表重命名到不同的方案中。如果不小心，你可能会尝试执行以下操作（以为自己在一个操作中移动了表并重命名了它）：

```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp tables=heera.inv \
remap_table=heera.inv:scott.inven
```

在前面的示例中，你最终会在 `HEERA` 方案中得到一个名为 `SCOTT` 的表。这可能会令人困惑。


# 重映射数据

你可以在导出或导入过程中，应用一个 PL/SQL 函数来修改列值。例如，假设你有一个审计员需要查看数据，而其中一项要求是，你需要对敏感列应用一个简单的混淆函数。数据不需要被加密；只需要进行足够的改动，使得审计员无法轻易确定 `CUSTOMERS` 表中 `LAST_NAME` 列的值。

此示例首先创建一个用于混淆数据的简单程序包：

```
create or replace package obfus is
function obf(clear_string varchar2) return varchar2;
function unobf(obs_string varchar2) return varchar2;
end obfus;
/
--
create or replace package body obfus is
fromstr varchar2(62) := '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ' ||
'abcdefghijklmnopqrstuvwxyz';
tostr varchar2(62)   := 'defghijklmnopqrstuvwxyzabc3456789012' ||
'KLMNOPQRSTUVWXYZABCDEFGHIJ';
--
function obf(clear_string varchar2) return varchar2 is
begin
return translate(clear_string, fromstr, tostr);
end obf;
--
function unobf(obs_string varchar2) return varchar2 is
begin
return translate(obs_string, tostr, fromstr);
end unobf;
end obfus;
/
```

现在，当你将数据导入数据库时，对 `CUSTOMERS` 表的 `LAST_NAME` 列应用混淆函数：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=cust.dmp tables=customers  \
remap_data=customers.last_name:obfus.obf
```

从 `CUSTOMERS` 表中查询 `LAST_NAME`，可以看到它已以混淆的方式被导入：

```
SQL> select last_name from customers;
LAST_NAME

yYZEJ
tOXXSMU
xERX
```

你可以手动应用程序包的 `UNOBF` 函数来查看该列的真实值：

```
SQL> select obfus.unobf(last_name) from customers ;
OBFUS.UNOBF(LAST_NAME)

Lopuz
Gennick
Kuhn
```

## 抑制日志文件

默认情况下，Data Pump 在生成导出或导入时会创建一个日志文件。如果你确定不希望生成日志文件，可以通过指定 `NOLOGFILE` 参数来抑制它。示例如下：

```
$ expdp mv_maint/foo directory=dp_dir tables=inv nologfile=y
```

如果你选择不创建日志文件，Data Pump 仍会在输出设备上显示状态消息。一般来说，我建议你在每次 Data Pump 操作时都创建一个日志文件。这为你提供了操作的审计跟踪。

## 使用并行处理

使用 `PARALLEL` 参数来并行化一个 Data Pump 作业。例如，如果你知道一台服务器上有四个 CPU，并且希望将并行度设置为 4，可以按如下方式使用 `PARALLEL`：

```
$ expdp mv_maint/foo parallel=4 dumpfile=exp.dmp directory=dp_dir full=y
```

为了充分利用并行特性，请确保在导出时指定多个文件。以下示例为每个并行线程创建一个文件：

```
$ expdp mv_maint/foo parallel=4 dumpfile=exp1.dmp,exp2.dmp,exp3.dmp,exp4.dmp
```

你也可以使用 `%U` 替换变量来指示 Data Pump 自动创建与并行度匹配的转储文件。`%U` 变量的值从 01 开始，并随着分配更多转储文件而递增。此示例使用了 `%U` 变量：

```
$ expdp mv_maint/foo parallel=4 dumpfile=exp%U.dmp
```

现在，假设你需要从导出创建的转储文件中导入。你可以单独指定转储文件，或者，如果转储文件是使用 `%U` 变量创建的，则可以在导入时使用它：

```
$ impdp mv_maint/foo parallel=4 dumpfile=exp%U.dmp
```

在前面的示例中，导入过程会首先查找名为 `exp01.dmp` 的文件，然后是 `exp02.dmp`，依此类推。

## 提示

Oracle 建议并行度不要设置为服务器上可用 CPU 数量的两倍以上。另外，在 RAC 环境中使用并行处理时需要注意：请确保设置 `CLUSTER=N` 以避免跨节点的并行处理。

你也可以在作业运行时修改并行度。首先，以交互命令模式连接到你想要修改并行度的作业（参见本章后面的“交互命令模式”部分）。然后，使用 `PARALLEL` 选项。在此示例中，连接到的作业是 `SYS_IMPORT_TABLE_01`：

```
$ impdp mv_maint/foo attach=sys_import_table_01
Import> parallel=6
```

你可以通过 `STATUS` 命令检查并行度：

```
Import> status
```

以下是一些示例输出：

```
Job: SYS_IMPORT_TABLE_01
Operation: IMPORT
Mode: TABLE
State: EXECUTING
Bytes Processed: 0
Current Parallelism: 6
```

## 注意

`PARALLEL` 功能仅在 Oracle 企业版中可用。

## 指定额外的转储文件

如果主要的 Data Pump 位置空间不足，你可以动态指定额外的 Data Pump 位置。使用交互式命令提示符下的 `ADD_FILE` 命令。以下是添加额外文件的基本语法：

```
ADD_FILE=[directory_object:]file_name [,...]
```

此示例向一个已存在的 Data Pump 导出作业添加另一个输出文件：

```
Export> add_file=alt2.dmp
```

你也可以指定一个独立的数据库目录对象：

```
Export> add_file=alt_dir:alt3.dmp
```

## 重用输出文件名

默认情况下，Data Pump 不会覆盖现有的转储文件。例如，当你第一次运行此作业时，它会正常运行，因为所使用的目录中没有名为 `inv.dmp` 的转储文件：

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp
```

如果你尝试使用相同的目录和相同的数据泵名称再次运行前面的命令，将会抛出此错误：

```
ORA-31641: unable to create dump file "/oradump/inv.dmp"
```

你可以为导出作业指定一个新的数据泵名称，或者使用 `REUSE_DUMPFILES` 参数指示 Data Pump 覆盖现有的转储文件；例如，

```
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp reuse_dumpfiles=y
```

现在，无论输出目录中是否存在同名的现有转储文件，你应该都能够运行 Data Pump 导出了。当你将 `REUSE_DUMPFILES` 设置为 `y` 时，如果 Data Pump 找到同名的转储文件，它会覆盖该文件。

## 注意

`REUSE_DUMPFILES` 的默认值为 `N`。

## 创建每日 DDL 文件

有时，在数据库环境中，数据库对象会以意想不到的方式发生变化。你可能会有一个开发人员以某种方式获取了生产用户密码，并决定在不告知任何人的情况下即时进行更改。或者，一个 DBA 可能决定不遵循标准的发布流程，在排查问题时更改了某个对象。这些场景对于生产支持 DBA 来说可能令人沮丧。每当出现问题时，提出的第一个问题是：“什么发生了变化？”当你使用 Data Pump 时，创建一个包含数据库中所有对象重新创建所需 DDL 的文件是相当简单的。你可以通过 `CONTENT=METADATA_ONLY` 选项指示 Data Pump 仅导出或导入元数据。例如，在生产环境中，你可以设置一个每日作业来捕获这些 DDL。如果对什么发生了变化以及何时发生变化有疑问，你可以回头比较每日转储文件中的 DDL。

下面列出了一个简单的 Shell 脚本，它首先从数据库导出元数据内容，然后使用 Data Pump 导入从该导出创建 DDL 文件：

#!/bin/bash
# 源操作系统变量，详见第 2 章
. /etc/oraset o18c
#
DAY=$(date +%Y_%m_%d)
SID=DWREP
#---------------------------------------------------
# 首先仅使用元数据创建导出转储文件
expdp mv_maint/foo dumpfile=${SID}.${DAY}.dmp content=metadata_only \
directory=dp_dir full=y logfile=${SID}.${DAY}.log
#---------------------------------------------------
# 现在从导出转储文件创建 DDL 文件。
impdp mv_maint/foo directory=dp_dir dumpfile=${SID}.${DAY}.dmp \
SQLFILE=${SID}.${DAY}.sql logfile=${SID}.${DAY}.sql.log
#
exit 0

这段代码依赖于创建了一个数据库目录对象，该对象指向你希望每日转储文件被写入的位置。你可能还想设置另一个作业来定期删除超过一定时间的文件。

## 压缩输出

当使用 Data Pump 创建大文件时，你应该考虑压缩输出。
从 Oracle Database 11g 开始，`COMPRESSION` 参数可以是以下值之一：`ALL`、`DATA_ONLY`、`METADATA_ONLY` 或 `NONE`。如果你指定 `ALL`，那么输出文件中的数据和元数据都会被压缩。此示例导出一个表并压缩输出文件中的数据和元数据：

```sql
$ expdp dbauser/foo tables=locations directory=datapump \
dumpfile=compress.dmp compression=all
```

注意
`COMPRESS` 参数的 `ALL` 和 `DATA_ONLY` 选项需要 Oracle Advanced Compression 选项的许可。

Oracle Database 12c 新增功能，你可以指定压缩算法。
选项有 `BASIC`、`LOW`、`MEDIUM` 和 `HIGH`。下面是一个使用 `MEDIUM` 压缩的示例：

```sql
$ expdp mv_maint/foo dumpfile=full.dmp directory=dp_dir full=y \
compression=all compression_algorithm=MEDIUM
```

使用 `COMPRESSION_ALGORITHM` 参数在磁盘空间不足或通过网络连接导出时特别有用（因为它减少了需要传输的字节数）。

注意
`COMPRESSION_ALGORITHM` 参数需要 Oracle Advanced Compression 选项的许可。

## 在导入时更改表压缩特性

从 Oracle Database 12c 开始，你可以在导入表时更改表的压缩特性。此示例将作业中导入的所有表的压缩特性更改为 `COMPRESS FOR OLTP`。因为此示例中的命令需要引号，所以它被放置在参数文件中，如下所示：

```sql
userid=mv_maint/foo
dumpfile=inv.dmp
directory=dp_dir
transform=table_compression_clause:"COMPRESS FOR OLTP"
```

假设参数文件名为 `imp.par`。现在可以如下调用它：

```sql
$ impdp parfile=imp.par
```

导入作业中包含的所有表都将创建为 `COMPRESS FOR OLTP`，并且数据在加载时被压缩。

注意
表级压缩（用于 OLTP）需要 Oracle Advanced Compression 选项的许可。

## 加密数据

Data Pump 转储文件的一个潜在安全问题是，任何具有对输出文件操作系统访问权限的人都可以在文件中搜索字符串。
在 Linux/Unix 系统上，你可以使用 `strings` 命令执行此操作：

```bash
$ strings inv.dmp | grep -i secret
```

这是这个特定转储文件的输出：

```text
Secret Data<
top secret data<
corporate secret data<
```

这个命令允许你查看转储文件的内容，因为数据是常规文本且未加密。如果你要求数据被保护，你可以使用 Data Pump 的加密功能。

此示例使用 `ENCRYPTION` 参数来保护输出中的所有数据和元数据：

```sql
$ expdp mv_maint/foo encryption=all directory=dp_dir dumpfile=inv.dmp
```

要使此命令工作，你的数据库必须有一个已打开的加密钱包。有关如何创建和打开钱包的更多详细信息，请参阅 *Oracle Advanced Security Administrator's Guide*，可从 Oracle 网站的 Technology Network 区域下载（网址：`http://otn.oracle.com`）。

## 注意

数据泵的 `ENCRYPTION` 参数要求您使用 Oracle 数据库的企业版，并且还需要拥有 Oracle 高级安全选件的许可。

## `ENCRYPTION` 参数

`ENCRYPTION` 参数接受以下选项：

*   `ALL`
*   `DATA_ONLY`
*   `ENCRYPTED_COLUMNS_ONLY`
*   `METADATA_ONLY`
*   `NONE`

`ALL` 选项对数据和元数据都启用加密。`DATA_ONLY` 选项仅加密数据。`ENCRYPTED_COLUMNS_ONLY` 选项指定只有数据库中加密的列才会以加密格式写入转储文件。`METADATA_ONLY` 选项仅加密导出文件中的元数据。

## 将视图导出为表

从 Oracle 数据库 12c 开始，您可以导出视图，并在之后将其作为表导入。如果您需要将视图中包含的数据复制到历史报表数据库，可能需要这样做。

使用 `VIEWS_AS_TABLES` 参数将视图导出为表结构。此参数的语法如下：

```bash
VIEWS_AS_TABLES=[schema_name.]view_name[:template_table_name]
```

这是一个示例：

```bash
$ expdp mv_maint/foo directory=dp_dir dumpfile=v.dmp \
views_as_tables=sales_rockies
```

现在，该转储文件可用于将名为 `SALES_ROCKIES` 的表导入到其他方案或数据库中。

```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=v.dmp
```

如果您只想导入该表（该表在导出过程中由视图创建），可以按如下方式操作：

```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=v.dmp tables=sales_rockies
```

该表将拥有与视图定义相同的列和数据类型。此外，该表还将包含与从视图导出时所选内容相匹配的数据行。

## 在导入时禁用重做日志记录

从 Oracle 数据库 12c 开始，您可以指定对象以 nologging 方式加载重做日志。这是通过 `DISABLE_ARCHIVE_LOGGING` 参数实现的：

```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
transform=disable_archive_logging:Y
```

在执行导入期间，对象的日志记录属性被设置为 `NO`；导入后，日志记录属性会恢复为其原始值。对于数据泵可以通过直接路径执行的操作（例如向表中插入数据），这可以减少导入期间生成的重做日志量。

## 附加到正在运行的作业

数据泵的一个强大功能是，您可以附加到当前正在运行的作业，并查看其进度和状态。如果您拥有 DBA 权限，即使您不是作业所有者，也可以附加到该作业。您可以通过 `ATTACH` 参数附加到导入或导出作业。

数据泵不像旧的实用程序 `exp` 那样在前台运行。数据泵在后台运行，因此如果您按下 Control-C 来中断作业，数据泵作业会继续运行，只是命令行界面被中断。附加数据泵作业允许您对该作业执行操作。

在附加到作业之前，您必须首先确定数据泵作业名称（如果您不是作业所有者，还需要确定所有者名称）。运行以下 SQL 查询以显示当前正在运行的作业：

```sql
SQL> select owner_name, operation, job_name, state from dba_datapump_jobs;
```

以下是一些示例输出：

```sql
OWNER_NAME OPERATION       JOB_NAME              STATE
---------- --------------- --------------------  --------------------
MV_MAINT   EXPORT          SYS_EXPORT_SCHEMA_01  EXECUTING
```

在此示例中，`MV_MAINT` 用户可以直接附加到导出作业，如下所示：

```bash
$ expdp mv_maint/foo attach=sys_export_schema_01
```

如果您不是作业所有者，请通过指定所有者名称和作业名称来附加到作业：

```bash
$ expdp system/foobar attach=mv_maint.sys_export_schema_01
```

您现在应该会看到数据泵命令行提示符：

```bash
Export>
```

键入 `STATUS` 以查看当前附加作业的状态：

```bash
Export> status
```

## 停止和重启作业

如果您有一个正在运行的数据泵作业，并希望暂时停止它，可以先附加到交互式命令模式。您可能希望停止作业以解决空间问题或性能问题，然后在问题解决后重新启动作业。此示例附加到导入作业：

```bash
$ impdp mv_maint/foo attach=sys_import_table_01
```

现在，使用 `STOP_JOB` 参数停止作业：

```bash
Import> stop_job
```

您应该会看到以下输出：

```bash
Are you sure you wish to stop this job ([yes]/no):
```

键入 `YES` 以继续停止作业。您也可以指定立即停止作业：

```bash
Import> stop_job=immediate
```

当您使用 `IMMEDIATE` 选项停止作业时，可能会有一些与该作业相关的未完成任务。要重新启动作业，请附加到交互式命令模式，并发出 `START_JOB` 命令：

```bash
Import> start_job
```

如果您希望将作业输出继续记录到您的终端，请发出 `CONTINUE_CLIENT` 命令：

```bash
Import> continue_client
```

## 终止数据泵作业

您可以指示数据泵永久终止导出或导入作业。首先，在交互式命令模式下附加到作业，然后发出 `KILL_JOB` 命令：

```bash
Import> kill_job
```

系统将提示您以下输出：

```bash
Are you sure you wish to stop this job ([yes]/no):
```

键入 `YES` 以永久终止该作业。数据泵会立即将作业终止，并从运行导出或导入的用户中删除相关的状态表。

## 监控数据泵作业

当您有长时间运行的数据泵作业时，应不时检查作业状态，以确保其没有失败、暂停等。有几种方法可以监控数据泵作业的状态：

*   屏幕输出
*   数据泵日志文件
*   查询数据字典视图
*   数据库警报日志
*   查询状态表
*   交互式命令模式状态
*   使用进程状态 (`ps`) 操作系统实用程序
*   Oracle 企业管理器

监控作业最直接的方法是查看数据泵在作业运行时显示在屏幕上的状态。如果您已从命令模式断开连接，则屏幕上将不再显示状态。在这种情况下，您必须使用其他技术来监控数据泵作业。

### 数据泵日志文件

默认情况下，数据泵为每个作业生成一个日志文件。启动数据泵作业时，最好为其指定一个特定的日志文件名称：

```bash
$ impdp mv_maint/foo directory=dp_dir dumpfile=archive.dmp logfile=archive.log
```

此作业创建一个名为 `archive.log` 的文件，该文件放置在数据库对象 `DP` 引用的目录中。如果您没有明确指定日志文件名，数据泵导入会创建一个名为 `import.log` 的文件，数据泵导出会创建一个名为 `export.log` 的文件。

注意：日志文件包含的信息与您在运行数据泵作业时交互式显示在屏幕上的信息相同。

## 数据字典视图

确定数据泵作业是否正在运行的一种快速方法是检查 `DBA_DATAPUMP_JOBS` 视图中是否有状态为 `EXECUTING` 的作业在运行：

```sql
select job_name, operation, job_mode, state
from dba_datapump_jobs;
```

以下是一些示例输出：

```sql
JOB_NAME                  OPERATION            JOB_MODE   STATE
------------------------- -------------------- ---------- ---------------
SYS_IMPORT_TABLE_04       IMPORT               TABLE      EXECUTING
SYS_IMPORT_FULL_02        IMPORT               FULL       NOT RUNNING
```

您还可以通过以下查询 `DBA_DATAPUMP_SESSIONS` 视图来获取会话信息：

```sql
select sid, serial#, username, process, program
from v$session s,
dba_datapump_sessions d
where s.saddr = d.saddr;
```

以下是一些示例输出，显示正在使用多个数据泵会话。

# 数据泵监控与外部表

## 状态查询示例

```
SID     SERIAL#  USERNAME    PROCESS     PROGRAM
---------- -----------  ----------- ----------- ------------------------
1049        6451  STAGING     11306       oracle@xengdb (DM00)
1058       33126  STAGING     11338       oracle@xengdb (DW01)
1048       50508  STAGING     11396       oracle@xengdb (DW02)
```

## 数据库告警日志

如果某个任务执行时间远超预期，请检查数据库告警日志中是否有类似以下信息：

```
statement in resumable session 'SYS_IMPORT_SCHEMA_02.1' was suspended due to
ORA-01652: unable to extend temp segment by 64 in tablespace REG_TBSP_3
```

此消息表明一个数据泵导入作业已暂停，并等待向`REG_TBSP_3`表空间添加空间。添加空间后，数据泵作业会自动恢复处理。默认情况下，数据泵作业会等待 2 小时以添加空间。

**注意**
除了写入告警日志，对于每个数据泵作业，Oracle 还会在`ADR_HOME/trace`目录中创建一个跟踪文件。此文件包含会话 ID 和作业启动时间等信息。跟踪文件的命名格式为：`<SID>_dm00_<process_ID>.trc`。

## 状态表

每次启动数据泵作业时，都会在运行该作业的用户账户中自动创建一个状态表。对于导出作业，表名取决于所运行的导出类型。表名格式为`SYS_<OPERATION>_<JOB_MODE>_NN`，其中`OPERATION`为`EXPORT`或`IMPORT`。`JOB_MODE`可以是`FULL`、`SCHEMA`、`TABLE`、`TABLESPACE`等。

以下是查询状态表以获取当前运行作业详细信息的示例：

```
select name, object_name, total_bytes/1024/1024 t_m_bytes
,job_mode
,state ,to_char(last_update, 'dd-mon-yy hh24:mi')
from SYS_EXPORT_TABLE_01
where state='EXECUTING';
```

## 交互式命令模式状态

验证数据泵是否正在运行作业的一个快速方法是，以交互命令模式附加并发出`STATUS`命令；例如：

```
$ impdp mv_maint/foo attach=SYS_IMPORT_TABLE_04
Import> status
```

以下是部分示例输出：

```
Job: SYS_IMPORT_TABLE_04
Operation: IMPORT
Mode: TABLE
State: EXECUTING
Bytes Processed: 0
Current Parallelism: 4
```

您应该会看到状态为`EXECUTING`，这表明作业正在主动运行。输出中还需检查的其他项目是对象数量和已处理的字节数。这些数字应随着作业的进展而增加。

## 操作系统工具

您可以使用`ps`操作系统工具来显示服务器上运行的作业。例如，您可以搜索主进程和工作进程，如下所示：

```
$ ps -ef | egrep 'ora_dm|ora_dw' | grep -v egrep
```

以下是部分示例输出：

```
oracle 29871   717   5 08:26:39 ?          11:42 ora_dw01_STAGE
oracle 29848   717   0 08:26:33 ?           0:08 ora_dm00_STAGE
oracle 29979   717   0 08:27:09 ?           0:04 ora_dw02_STAGE
```

如果您多次运行此命令，您应该会看到当前作业的一个或多个处理时间（第七列）在增加。这是数据泵仍在执行和工作的一个很好指标。

## 总结

数据泵是一个极其强大且功能丰富的工具。如果您不常使用数据泵，那么我建议您花些时间重新阅读本章并演练示例。此工具极大地简化了将用户和数据从一个环境移动到另一个环境等任务。您可以使用一个命令来导出和导入用户子集、通过 SQL 和 PL/SQL 过滤和重映射数据、重命名用户和表空间、压缩、加密和并行化。它确实如此强大。

数据泵提供了在数据库之间移动数据所需的功能，甚至无需将数据存储在磁盘上进行传输。借助数据泵的功能，可以对数据进行过滤或重映射到其他模式和表。可以启动数据泵作业，并可以对其进行监控、暂停或停止。

尽管数据泵是将数据库对象和数据从一个环境移动到另一个环境的优秀工具，但有时您需要在操作系统平面文件之间传输大量数据。您可以使用外部表来实现此任务。这是本书下一章的主题。

# 14. 外部表

有时，DBA 和开发人员没有掌握外部表的实用性。Oracle 外部表功能使您能够执行以下操作：

*   透明地将操作系统文件中具有分隔符或固定字段的信息选择到数据库中。
*   创建可用于传输数据的平台无关的转储文件。您还可以将这些文件创建为压缩和加密格式，以实现高效和安全的数据传输。
*   允许在不创建数据库外部表的情况下，内联对文件数据运行 SQL。

**提示**
逗号分隔文件（CSV）是一种平面文件，有时可简称为平面文件。

外部表的一个常见用途是通过 SQL*Plus 从操作系统平面文件中选择数据。简而言之，外部表允许在无需先将数据加载到表中的情况下，从平面文件读取数据到数据库中。这将允许在将所需数据加载到数据库的同时对文件执行转换。使用外部表可以简化或增强 ETL（提取、转换和加载）过程。在此模式下使用外部表时，您必须指定文件中的数据类型以及数据的组织方式。您可以从外部表中选择，但不允许修改内容（无插入、更新或删除）。

您还可以使用外部表功能，使您能够从数据库中选择数据并将该信息写入二进制转储文件。外部表的定义决定了将使用哪些表和列来卸载数据。在此模式下使用外部表提供了一种将大量数据提取到平台无关文件的方法，您可以稍后将该文件加载到不同的数据库中。

启用外部表所需的全部条件是首先创建一个指定操作系统文件位置的数据库目录对象。然后，使用`CREATE TABLE...ORGANIZATION EXTERNAL`语句使数据库知晓可用作数据源或目标的操作系统文件。

本章首先比较使用 SQL*Loader（Oracle 的传统数据加载实用程序）与外部表将数据加载到数据库中。几个示例说明了将外部表用作加载和数据转换工具的灵活性和强大功能。本章最后提供了一个如何将数据卸载到转储文件的外部表示例。


# SQL*Loader 与外部表

## 概述

外部表的一个常见用途是使用 SQL 将操作系统文件中的数据加载到常规数据库表中。这有助于将大量数据从平面文件加载到数据库中。在 Oracle 的旧版本中，这种类型的加载是通过 `SQL*Loader` 或自定义的 `Pro*C` 程序完成的。

几乎任何可以使用 `SQL*Loader` 完成的操作，都可以通过外部表实现。一个重要区别在于，`SQL*Loader` 将数据加载到表中，而外部表无需这样做。外部表比 `SQL*Loader` 更灵活、更直观。此外，通过使用直接路径和并行特性，使用外部表加载数据时可以获得非常好的性能。

## 加载数据步骤对比

快速比较一下通过 `SQL*Loader` 和外部表将数据加载到数据库的方式，可以突显其用法。以下是使用 `SQL*Loader` 加载和转换数据的步骤：

1.  创建一个 `SQL*Loader` 用来解释操作系统文件中数据格式的参数文件。
2.  创建一个常规数据库表，`SQL*Loader` 将向其中插入记录。数据将暂存于此，直到可以进一步处理。
3.  运行 `SQL*Loader` 的 `sqlldr` 实用程序，将数据从操作系统文件加载到（步骤 2 中创建的）数据库表中。在加载数据时，`SQL*Loader` 提供了一些允许你转换数据的功能。此步骤有时令人沮丧，因为可能需要多次试错运行才能正确地将参数文件映射到表及相应列。
4.  创建另一个表，用于包含完全转换后的数据。
5.  运行 SQL 从暂存表（步骤 2 中创建）中提取数据，然后转换并将其插入到生产表（步骤 4 中创建）中。

将上述 `SQL*Loader` 列表与以下使用外部表加载和转换数据的步骤进行比较：

1.  执行 `CREATE TABLE...ORGANIZATION EXTERNAL` 脚本，将操作系统文件的结构映射到表列。运行此脚本后，你可以直接使用 SQL 查询操作系统文件的内容。
2.  创建一个常规表来保存完全转换后的数据。
3.  运行 SQL 语句，将数据从外部表（步骤 1 中创建）加载并完全转换到步骤 2 中创建的表中。

## 外部表的优势

对于许多公司来说，`SQL*Loader` 是大型数据加载操作的基础。它仍然是完成该任务的好工具。但是，你可能需要考虑使用外部表。外部表具有以下优势：

*   使用外部表加载数据更直接，需要的步骤更少。
*   创建和从外部表加载的接口是 `SQL*Plus`。许多 DBA/开发人员发现 `SQL*Plus` 比 `SQL*Loader` 的参数文件接口更直观、更强大。
*   你可以在数据加载到数据库表之前（通过 SQL）查看外部表中的数据。
*   你无需中间暂存表即可加载、转换和聚合数据。对于大量数据，这可以节省巨大的空间。

接下来的几个部分包含使用外部表从操作系统文件读取的示例。

## 将 CSV 文件加载到数据库

你可以使用外部表和 SQL 将小型或非常大的 CSV 平面文件加载到数据库中。图 14-1 显示了使用外部表查看和加载操作系统文件数据所涉及的架构组件。需要一个目录对象来指定操作系统文件的位置。`CREATE TABLE...ORGANIZATION EXTERNAL` 语句创建了一个数据库对象，`SQL*Plus` 可以使用它直接从操作系统文件中进行选择。

![外部表用于读取平面文件的架构组件](img/214899_3_En_14_Fig1_HTML.jpg)

**图 14-1** 外部表用于读取平面文件的架构组件

以下是使用外部表访问操作系统平面文件的步骤：

1.  创建一个指向 CSV 文件位置的数据库目录对象。
2.  将目录对象的读写权限授予创建外部表的用户。（即使使用具有 DBA 权限的账户更容易，但根据各种安全选项，该账户可能无法访问表和数据。需要验证并根据需要授予权限。）
3.  运行 `CREATE TABLE...ORGANIZATION EXTERNAL` 语句。
4.  使用 `SQL*Plus` 访问 CSV 文件的内容。

在此示例中，平面文件名为 `ex.csv`，位于 `/u01/et` 目录。它包含以下数据：

```
5|2|0|0|12/04/2011|Half
6|1|0|1|09/06/2012|Quarter
7|4|0|1|08/10/2012|Full
8|1|1|0|06/15/2012|Quarter
```

> **注意**
> 本章中的一些分隔文件示例由逗号以外的字符分隔，例如管道符 (`|`)。使用的字符取决于数据和平面文件的提供者。逗号并不总是有用的分隔符，因为被加载的数据可能包含作为数据内有效字符的逗号。也可以使用固定字段长度而不是使用分隔符。

### 创建目录对象并授予权限

首先，创建一个指向磁盘上平面文件位置的目录对象：

```sql
SQL> create directory exa_dir as '/u01/et';
```

此示例使用了一个被授予 DBA 角色的数据库账户；因此，你不需要将目录对象的 `READ` 和 `WRITE` 权限授予访问该目录对象的用户（你的账户）。如果你不是使用 DBA 账户从目录对象读取，则使用此对象将这些权限授予该账户：

```sql
SQL> grant read, write on directory exa_dir to reg_user;
```

### 创建外部表

然后，编写创建将引用该平面文件的外部表的脚本。`CREATE TABLE...ORGANIZATION EXTERNAL` 语句向数据库提供以下信息：

*   如何解释平面文件中的数据，以及文件中数据到数据库中列定义的映射
*   一个 `DEFAULT DIRECTORY` 子句，用于标识目录对象，该对象又指定了磁盘上平面文件的目录
*   `LOCATION` 子句，用于标识平面文件的名称

下一条语句创建一个看起来像表但能够直接从平面文件检索数据的数据库对象：

```sql
SQL> create table exadata_et(
exa_id        NUMBER
,machine_count NUMBER
,hide_flag     NUMBER
,oracle        NUMBER
,ship_date     DATE
,rack_type     VARCHAR2(32)
)
organization external (
type              oracle_loader
default directory exa_dir
access parameters
(
records delimited  by newline
fields  terminated by '|'
missing field values are null
(exa_id
,machine_count
,hide_flag
,oracle
,ship_date char date_format date mask "mm/dd/yyyy"
,rack_type)
)
location ('ex.csv')
)
reject limit unlimited;
```

当你执行此脚本时，会创建一个名为 `EXADATA_ET` 的外部表。现在，使用 `SQL*Plus` 查看平面文件的内容：

```sql
SQL> select * from exadata_et;
EXA_ID MACHINE_COUNT  HIDE_FLAG     ORACLE SHIP_DATE  RACK_TYPE
---------- ------------- ---------- ---------- ---------- ----------------
5             2          0          0 04-DEC-11  Half
6             1          0          1 06-SEP-12  Quarter
7             4          0          1 10-AUG-12  Full
8             1          1          0 15-JUN-12  Quarter
```

## 生成用于创建外部表的 SQL

如果你当前正在使用 `SQL*Loader` 并想转换为使用外部表，你可以使用 `SQL*Loader` 的 `EXTERNAL_TABLE` 选项来生成创建外部表所需的 SQL。一个小例子将有助于演示此过程。假设你有以下表 DDL：

```sql
SQL> create table books
(book_id number,
book_desc varchar2(30));
```



# 使用外部表将数据从平面文件加载到 Oracle 数据库中

在这种情况下，您需要将以下数据从 CSV 文件加载到 `BOOKS` 表中。数据位于名为 `books.dat` 的文件中，内容如下：

```
1|RMAN Recipes
2|Linux for DBAs
3|SQL Recipes
```

您还有一个名为 `books.ctl` 的 SQL*Loader 控制文件，其中包含以下数据：

```
load data
INFILE 'books.dat'
INTO TABLE books
APPEND
FIELDS TERMINATED BY '|'
(book_id,
book_desc)
```

## 生成外部表 DDL

您可以使用带有 `EXTERNAL_TABLE=GENERATE_ONLY` 子句的 SQL*Loader 来生成创建外部表所需的 SQL；例如，

```
$ sqlldr dk/f00 control=books.ctl log=books.log external_table=generate_only
```

前面这行代码不会加载任何数据。而是创建一个名为 `books.log` 的文件，其中包含创建外部表所需的 SQL。以下是生成代码的部分列表：

```
CREATE TABLE "SYS_SQLLDR_X_EXT_BOOKS"
(
"BOOK_ID" NUMBER,
"BOOK_DESC" VARCHAR2(30)
)
ORGANIZATION external
(
TYPE oracle_loader
DEFAULT DIRECTORY SYS_SQLLDR_XT_TMPDIR_00000
ACCESS PARAMETERS
(
RECORDS DELIMITED BY NEWLINE CHARACTERSET US7ASCII
BADFILE 'SYS_SQLLDR_XT_TMPDIR_00000':'books.bad'
LOGFILE 'books.log_xt'
READSIZE 1048576
FIELDS TERMINATED BY "|" LDRTRIM
REJECT ROWS WITH ALL NULL FIELDS
(
"BOOK_ID" CHAR(255)
TERMINATED BY "|",
"BOOK_DESC" CHAR(255)
TERMINATED BY "|"
)
)
location
(
'books.dat'
)
)REJECT LIMIT UNLIMITED;
```

## 创建目录和外部表

在运行前面的代码之前，请创建一个指向 `books.dat` 文件位置的目录；例如，

```
SQL> create or replace directory SYS_SQLLDR_XT_TMPDIR_00000
as '/u01/sqlldr';
```

现在，如果您运行 SQL*Loader 生成的 SQL 代码，您应该能够查看 `SYS_SQLLDR_X_EXT_BOOKS` 表中的数据：

```
SQL> select * from SYS_SQLLDR_X_EXT_BOOKS;
```

预期输出如下：

```
BOOK_ID BOOK_DESC
---------- ------------------------------
1 RMAN Recipes
2 Linux for DBAs
3 SQL Recipes
```

这是一个强大的技术，尤其当您已有现成的 SQL*Loader 控制文件，并希望确保在转换为外部表时语法正确。

## 查看外部表元数据

此时，您还可以查看有关外部表的元数据。查询 `DBA_EXTERNAL_TABLES` 视图获取详细信息：

```
SQL> select
owner
,table_name
,default_directory_name
,access_parameters
from dba_external_tables;
```

以下是部分输出列表：

```
OWNER      TABLE_NAME      DEFAULT_DIRECTORY_NA ACCESS_PARAMETERS
---------- --------------- -------------------- --------------------
SYS        EXADATA_ET      EXA_DIR              records delimited ...
```

此外，您可以从 `DBA_EXTERNAL_LOCATIONS` 表中选择，以获取有关外部表中引用的任何平面文件的信息：

```
SQL> select
owner
,table_name
,location
from dba_external_locations;
```

以下是一些示例输出：

```
OWNER      TABLE_NAME      LOCATION
---------- --------------- --------------------
SYS        EXADATA_ET      ex.csv
```

## 从外部表加载常规表

现在，您可以将外部表中包含的数据加载到常规数据库表中。这样做时，您可以利用 Oracle 的直接路径加载和并行特性。此示例创建一个常规数据库表，该表将从外部表加载数据：

```
SQL> create table exa_info(
exa_id        NUMBER
,machine_count NUMBER
,hide_flag     NUMBER
,oracle        NUMBER
,ship_date     DATE
,rack_type     VARCHAR2(32)
) nologging parallel 2;
```

您可以（通过 `APPEND` 提示）直接路径加载这个常规表，内容来自外部表，如下所示：

```
SQL> insert /*+ APPEND */ into exa_info select * from exadata_et;
```

您可以通过在提交数据前尝试从中选择来验证表是否已直接路径加载：

```
SQL> select * from exa_info;
```

预期错误如下：

```
ORA-12838: cannot read/modify an object after modifying it in parallel
```

提交数据后，您就可以从该表中选择了。



`SQL>` `commit`;
`SQL>` `select * from exa_info;`

注意：当使用外部表读写数据时可能会出现转换错误。数字到日期或字符字段的转换应该能被识别，但当收到这些错误时，可以在语句中显式创建转换。使用`TO_NUMBER`、`TO_DATE`和`TO_CHAR`有助于避免这些如果转换未隐式进行时会出现的问题。

另一种直接路径加载表的方法是使用`CREATE TABLE AS SELECT`（CTAS）语句。CTAS 语句会自动尝试进行直接路径加载。在此示例中，`EXA_INFO`表在一条语句中被创建并加载：

```sql
SQL> create table exa_info nologging parallel 2 as select * from exadata_et;
```

通过使用直接路径加载和并行性，您可以实现类似于 SQL*Loader 的加载性能。使用 SQL 从外部表创建表的优势在于，在构建常规数据库表（此示例中为`EXA_INFO`）时，您可以使用标准 SQL*Plus 功能执行复杂的数据转换。

任何 CTAS 语句都会自动以底层表定义的并行度进行处理。但是，当使用`INSERT AS SELECT`语句时，您需要为会话启用并行性：

```sql
SQL> alter session enable parallel dml;
```

作为最后一步，您应该为任何已加载大量数据的表生成统计信息。这里是一个示例：

```sql
SQL> exec dbms_stats.gather_table_stats( -
ownname=>'SYS', -
tabname=>'EXA_INFO', -
estimate_percent => 20, -
cascade=>true);
```

执行高级转换

Oracle 提供了复杂的数据转换技术。本节详细说明如何使用管道函数来转换外部表中的数据。以下是执行此操作的步骤：

1.  创建一个外部表。
2.  创建一个与外部表中列映射的记录类型。
3.  基于步骤 2 中创建的记录类型创建一个表。
4.  创建一个管道函数，用于在加载时检查每一行，并根据业务需求转换数据。
5.  使用一个`INSERT`语句，该语句从外部表中选择数据，并使用管道函数在数据加载时转换它们。

此示例使用本章前面“将 CSV 文件加载到数据库”一节中创建的相同外部表和 CSV 文件。回想一下，外部表名是`EXADATA_ET`，CSV 文件名是`ex.csv`。在创建外部表之后，创建一个映射到外部表中列名的记录类型：

```sql
SQL> create or replace type rec_exa_type is object
(
exa_id        number
,machine_count number
,hide_flag     number
,oracle_flag   number
,ship_date     date
,rack_type     varchar2(32)
);
```

接下来，基于前面的记录类型创建一个表：

```sql
SQL> create or replace type table_exa_type is table of rec_exa_type;
```

Oracle PL/SQL 允许您将函数用作 SQL 操作的行源。此特性称为管道化。它允许您使用复杂的转换逻辑，并结合 SQL*Plus 的功能。在此示例中，您创建一个管道函数来转换选定列的数据。具体来说，此函数为`ORACLE_FLAG`列随机生成一个数字：

```sql
SQL> create or replace function exa_trans
return table_exa_type pipelined is
begin
for r1 in
(select rec_exa_type(
exa_id, machine_count, hide_flag
,oracle_flag, ship_date, rack_type
) exa_rec
from exadata_et) loop
if (r1.exa_rec.hide_flag = 1) then
r1.exa_rec.oracle_flag := dbms_random.value(low => 1, high => 100);
end if;
pipe row (r1.exa_rec);
end loop;
return;
end;
/
```

现在，您可以使用此函数将数据加载到常规数据库表中。作为参考，这里是将要实例化的表的`CREATE TABLE`语句：



SQL> 创建表 exa_info(
exa_id        NUMBER
,machine_count NUMBER
,hide_flag     NUMBER
,oracle_flag   NUMBER
,ship_date     DATE
,rack_type     VARCHAR2(32)
) nologging 并行 2;
```

接下来，使用管道函数一步完成从外部表中选择数据、进行转换并插入到常规数据库表中：

```
SQL> insert into exa_info select * from table(exa_trans);
```

以下是本示例中加载到 `EXA_INFO` 表中的数据：

```
SQL> select * from exa_info;
```

以下是一些示例输出，显示了 `ORACLE_FLAG` 列中包含随机值的行：

```
EXA_ID MACHINE_COUNT  HIDE_FLAG    ORACLE_FLAG   SHIP_DATE   RACK_TYPE
---------- ------------- ---------- ---------------- ----------  ---------
5             2          1             32   03-JAN-17        Half
6             1          0              0   06-SEP-17     Quarter
7             4          0              0   10-AUG-17        Full
8             1          1             58   15-JUL-17     Quarter
```

尽管本节中的示例很简单，但您可以使用该技术来应用任何级别的转换逻辑。此技术允许您将转换需求嵌入到一个管道 PL/SQL 函数中，在加载每一行时修改数据。

## 从 SQL 查看文本文件

外部表允许您使用 SQL `SELECT` 语句从操作系统平面文件中检索信息。例如，假设您想要报告告警日志文件的内容。首先，创建一个指向告警日志位置的目录对象：

```
SQL> select value from v$diag_info where name = 'Diag Trace';
```

本示例的输出为：

```
/ora01/app/oracle/diag/rdbms/o18c/o18c/trace
```

接下来，创建一个指向诊断跟踪目录的目录对象：

```
SQL> create directory t_loc as '/ora01/app/oracle/diag/rdbms/o18c/o18c/trace';
```

现在，创建一个映射到数据库告警日志操作系统文件的外部表。在此示例中，数据库名称为 `o18c`，因此告警日志文件名为 `alert_o18c.log`：

```
SQL> create table alert_log_file(
alert_text varchar2(4000))
organization external
( type              oracle_loader
default directory t_loc
access parameters (
records delimited by newline
nobadfile
nologfile
nodiscardfile
fields terminated by '#$~=ui$X'
missing field values are null
(alert_text)
)
location ('alert_o18c.log')
)
reject limit unlimited;
```

您可以通过 SQL 查询来查询该表；例如，

```
SQL> select * from alert_log_file where alert_text like 'ORA-%';
```

这使您能够使用 SQL 查看和报告告警日志的内容。您可能会发现这是为原本无法访问的操作系统文件提供 SQL 访问的一种便捷方式。

如果您之前使用过 SQL*Loader，外部表的 `ORACLE_LOADER` 访问驱动程序的 `ACCESS PARAMETERS` 子句可能看起来很熟悉。表 14-1 描述了一些更常用的访问参数。有关访问参数的完整列表，请参阅《Oracle Database Utilities Guide》*（*可以从 Oracle 网站的技术网络区域免费下载 [`http://otn.oracle.com`](http://otn.oracle.com) ）。

### 表 14-1 `ORACLE_LOADER` 驱动程序的部分访问参数

| 访问参数 | 描述 |
| --- | --- |
| `DELIMITED BY` | 指示哪个字符是字段分隔符 |
| `TERMINATED BY` | 指示字段如何终止 |
| `FIXED` | 指定固定长度记录的大小 |
| `BADFILE` | 存储因错误而无法加载的记录的文件名 |
| `NOBADFILE` | 指定不应创建文件来保存因错误而无法加载的记录 |
| `LOGFILE` | 创建外部表时记录常规消息的文件名 |
| `NOLOGFILE` | 指定不应创建日志文件 |
| `DISCARDFILE` | 命名写入未通过 `LOAD WHEN` 子句的记录的文件 |


