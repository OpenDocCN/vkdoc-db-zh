# Oracle 数据库控制文件与 SPFILE 恢复

## 控制文件恢复

当您连接到恢复目录时，即使目标数据库处于 `nomount` 模式，您也可以查看控制文件的备份信息。要列出控制文件的备份，请使用 `LIST` 命令，如下所示：

```bash
$ rman target / catalog rcat/foo@rcat
RMAN> startup nomount;
RMAN> list backup of controlfile;
```

如果您丢失了所有控制文件，并且正在使用恢复目录，请发出 `STARTUP NOMOUNT` 和 `RESTORE CONTROLFILE` 命令：

```sql
RMAN> startup nomount;
RMAN> restore controlfile;
```

RMAN 将控制文件恢复到由 `CONTROL_FILES` 初始化参数定义的位置。您应该会看到一条消息，指示您的控制文件已成功从 RMAN 备份片复制回来。现在，您可以将数据库更改为 `mount` 模式，并执行数据库所需的任何其他恢复和恢复命令。

> **注意**
> 当您从备份还原控制文件时，需要对整个数据库执行介质恢复，并使用 `OPEN RESETLOGS` 命令打开数据库，即使您没有还原任何数据文件。您可以通过查询 `V$DATABASE` 视图的 `CONTROLFILE_TYPE` 列来确定您的控制文件是否是备份。

## 使用自动备份

当您启用控制文件的自动备份并使用快速恢复区（FRA）时，还原控制文件相当简单。首先，连接到目标数据库，然后发出 `STARTUP NOMOUNT` 命令，接着是 `RESTORE CONTROLFILE FROM AUTOBACKUP` 命令，如下所示：

```bash
$ rman target /
RMAN> startup nomount;
RMAN> restore controlfile from autobackup;
```

RMAN 将控制文件恢复到由 `CONTROL_FILES` 初始化参数定义的位置。您应该会看到一条消息，指示您的控制文件已成功从 RMAN 备份片复制回来。以下是输出片段：

```
channel ORA_DISK_1: control file restore from AUTOBACKUP complete
```

现在，您可以将数据库更改为 `mount` 模式，并执行数据库所需的任何其他恢复和恢复命令。在测试环境中练习此示例的一种方法是将控制文件移动到另一个目录，并逐步完成还原选项。复制文件允许您通过将其移回来快速恢复运行，但这确实能让您练习这些还原操作。

## 指定备份文件名

当将数据库还原到不同的服务器时，这些通常是过程中的最初几个步骤：对目标数据库进行备份，复制到远程服务器，然后从 RMAN 备份还原控制文件。在这些场景中，已知包含控制文件的备份片名称。以下是一个示例，其中您指示 RMAN 从特定的备份片文件还原控制文件：

```sql
RMAN> startup nomount ;
RMAN> restore controlfile from
'/u01/O18C/rman/rman_ctl_c-3423216220-20130113-01.bk';
```

控制文件将被恢复到由 `CONTROL_FILES` 初始化参数定义的位置。

## 恢复 SPFILE

您可能出于几个不同的原因想要恢复 `spfile`：

*   您意外地在 `spfile` 中设置了一个导致实例无法启动的值。
*   您意外删除了 `spfile`。
*   您需要查看 `spfile` 在过去某个时间点的样子。

一个场景（这在我身上发生过不止一次）是您正在使用 `spfile`，而您团队中的一位 DBA 做了一些莫名其妙的操作，例如：

```sql
SQL> alter system set processes=1000000 scope=spfile;
```

该参数在磁盘上的 `spfile` 中被更改，但在内存中没有。一段时间后，数据库因维护而停止。尝试启动数据库时，您甚至无法让实例在 `nomount` 状态下启动。这是因为一个参数被设置为一个荒谬的值，将消耗盒子上的所有内存。在这种情况下，实例可能会挂起，或者您可能会看到以下一条或多条消息：

```
ORA-01078: failure in processing system parameters
ORA-00838: Specified value of ... is too small
```

如果您有一个 RMAN 备份可用，其中包含修改前的 `spfile` 副本，您可以简单地恢复 `spfile`。如果您使用的是恢复目录，以下是恢复 `spfile` 的过程：

*   如果您不使用恢复目录，有多种方法可以恢复您的 `spfile`。您采取的方法取决于几个变量，例如您是否使用快速恢复区（FRA）。
*   您已为自动备份配置了通道备份位置。
*   您使用自动备份的默认位置。

```bash
$ rman target / catalog rcat/foo@rcat
RMAN> startup nomount;
RMAN> restore spfile;
```

这只是一个概述，显示了需要采取的步骤，但并非所有细节都列在这里。确定包含 `spfile` 备份的备份片的位置并执行还原，如下所示：

```sql
RMAN> startup nomount force;
RMAN> restore spfile to '/tmp/spfile.ora'
from '/u01/O18C/rman/rman_ctl_c-3423216220-20130113-00.bk';
```

您应该会看到类似这样的消息：

```
channel ORA_DISK_1: SPFILE restore from AUTOBACKUP complete
```

在此示例中，`spfile` 被恢复到 `/tmp` 目录。恢复后，您可以将 `spfile` 以正确的名称复制到 `ORACLE_HOME/dbs`。对于我的环境（数据库名称：`o18c`），操作如下：

```bash
$ cp /tmp/spfile.ora $ORACLE_HOME/dbs/spfileo18c.ora
```

> **注意**
> 有关所有可能的 `spfile` 和控制文件恢复场景的完整描述，请参阅 Darl Kuhn、Sam Alapati 和 Arup Nanda 所著的 *RMAN Recipes for Oracle Database 12c, second edition*（Apress, 2013）。

## 不完全恢复

术语 *不完全数据库恢复* 意味着您无法恢复所有已提交的事务。*不完全* 意味着您不应用所有重做日志来恢复到最后一个已提交事务发生的点。换句话说，您是在还原和恢复到过去的一个时间点。因此，不完全数据库恢复也称为数据库时间点恢复（DBPITR）。通常，您出于以下原因之一执行不完全数据库恢复：

*   您没有执行完全恢复所需的所有重做日志。您丢失了完全恢复所需的归档日志文件或在线重做日志文件。这种情况可能由于所需的重做文件损坏或丢失而发生。
*   您故意希望将数据库回滚到过去的一个时间点。例如，如果有人意外截断了一个表，而您希望将数据库回滚到发出 `truncate table` 命令之前，您就会这样做。

不完全数据库恢复包括两个主要步骤：还原和恢复。还原步骤重新创建数据文件，恢复步骤应用重做日志直到指定的时间点。可以通过以下几种方式在 RMAN 中启动还原过程：

*   `RESTORE DATABASE UNTIL`
*   `FLASHBACK DATABASE`（根据 `UNDO` 信息可能不需要还原）

对于大多数不完全数据库恢复的情况，您可以使用 `RESTORE DATABASE UNTIL` 命令来指示 RMAN 从 RMAN 备份文件中检索数据文件。本章本节主要关注这种类型的不完全数据库恢复。Flashback Database 功能将在本章后面的“闪回数据库”一节中介绍。



