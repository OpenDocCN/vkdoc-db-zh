# RMAN 数据库恢复与还原指南

## 还原与恢复整个数据库

### `RESTORE DATABASE` 命令

`RESTORE DATABASE` 命令将还原数据库中的每个数据文件。例外情况是当 RMAN 检测到数据文件已被还原时；在那种情况下，它不会再次还原它们。如果你想覆盖该行为，请使用 `FORCE` 命令。

当你执行 `RECOVER DATABASE` 命令时，RMAN 会自动将重做日志应用到任何需要恢复的数据文件上。恢复过程包括应用在以下文件中找到的更改：

*   增量备份片（仅在使用增量备份时适用）
*   归档日志文件（自上次备份或应用的增量备份以来生成的）
*   在线重做日志文件（当前且未归档的）

在还原和恢复过程完成后，你可以打开你的数据库。完全数据库恢复仅在你拥有数据库的良好备份以及备份后生成的所有重做日志访问权限时才有效。你需要所有恢复数据库数据文件所需的重做日志。如果你没有所有必需的重做日志，那么你很可能必须执行不完全恢复（参见本章后面的“不完全恢复”部分）。

> **注意**
> 你的数据库必须至少处于 `MOUNT` 状态才能使用 RMAN 还原数据文件。这是因为在还原和恢复过程中，RMAN 从控制文件中读取信息。

你可以使用当前控制文件或备份控制文件来执行完整的数据库级恢复。

### 使用当前控制文件

你必须首先将数据库置于 `MOUNT` 模式以执行数据库范围的还原和恢复。这是因为 Oracle 不允许你在与 `SYSTEM` 表空间关联的数据文件正在被还原和恢复时以 `OPEN` 模式操作数据库。在这种情况下，以 `MOUNT` 模式启动数据库，发出 `RESTORE` 和 `RECOVER` 命令，然后打开数据库，如下所示：

```bash
$ rman target /
RMAN> startup mount;
RMAN> restore database;
RMAN> recover database;
RMAN> alter database open;
```

如果一切按预期进行，你最后应该看到的消息是：

```
Statement processed
```

### 使用备份控制文件

此技术使用从快速恢复区（FRA）检索的控制文件的自动备份（有关如何还原控制文件的更多示例，请参阅本章后面的“还原控制文件”部分）。在这种情况下，控制文件首先从备份中检索，然后再还原和恢复数据库：

```bash
$ rman target /
RMAN> startup nomount;
RMAN> restore controlfile from autobackup;
RMAN> alter database mount;
RMAN> restore database;
RMAN> recover database;
RMAN> alter database open resetlogs;
```

如果成功，你最后应该看到的消息是：

```
Statement processed
```

## 还原与恢复表空间

有时，你会遇到仅限于特定表空间或一组表空间的介质故障。在这种情况下，适合在表空间粒度级别进行还原和恢复。RMAN 的 `RESTORE TABLESPACE` 和 `RECOVER TABLESPACE` 命令将还原和恢复与指定表空间关联的所有数据文件。

### 数据库打开时还原表空间

如果你的数据库是打开的，那么你必须将要还原和恢复的表空间脱机。除了 `SYSTEM` 和 `UNDO` 表空间，你可以对任何表空间执行此操作。此示例在数据库打开时还原和恢复 `USERS` 表空间：

```bash
$ rman target /
RMAN> alter tablespace users offline immediate;
RMAN> restore tablespace users;
RMAN> recover tablespace users;
RMAN> alter tablespace users online;
```

在表空间联机后，你应该会看到类似这样的消息：

```
sql statement: alter tablespace users online
```

### 数据库处于 MOUNT 模式时还原表空间

通常，在执行还原和恢复时，DBA 会关闭数据库并以 `MOUNT` 模式重新启动它，为执行恢复做准备。将数据库置于 `MOUNT` 模式可确保没有用户连接到数据库，并且没有事务正在进行。

此外，如果你正在还原和恢复 `SYSTEM` 表空间，则必须以 `MOUNT` 模式启动数据库。Oracle 不允许在数据库打开时还原和恢复 `SYSTEM` 表空间数据文件。下一个示例在数据库处于 `MOUNT` 模式时还原 `SYSTEM` 表空间：

```bash
$ rman target /
RMAN> shutdown immediate;
RMAN> startup mount;
RMAN> restore tablespace system;
RMAN> recover tablespace system;
RMAN> alter database open;
```

如果成功，你最后应该看到的消息是：

```
Statement processed
```

### 还原只读表空间

当你发出 `RESTORE DATABASE` 命令时，RMAN 会与数据库的其余部分一起还原只读表空间。例如，以下命令将还原所有数据文件（包括那些处于只读模式的数据文件）：

```bash
RMAN> restore database;
```

> **注意**
> 如果你使用的是在只读表空间置于只读模式之后创建的备份，则无需对只读数据文件进行恢复。在这种情况下，自备份以来，只读表空间没有生成任何重做日志。

## 还原临时表空间

你不必还原或重新创建丢失的本地管理临时表空间临时文件。当你为使用而打开数据库时，Oracle 会自动检测并重新创建本地管理临时表空间临时文件。

当 Oracle 自动重新创建临时表空间时，它会将一条消息记录到你的目标数据库 `alert.log` 中，如下所示：

```
Re-creating tempfile
```

如果由于任何原因，你的临时表空间变得不可用，你也可以自己重新创建它。因为临时表空间中从来没有永久对象，所以你可以根据需要简单地重新创建它们。以下是如何创建本地管理临时表空间的示例：

```sql
SQL> CREATE TEMPORARY TABLESPACE temp TEMPFILE
'/u01/dbfile/o18c/temp01.dbf' SIZE 1000M
EXTENT MANAGEMENT
LOCAL UNIFORM SIZE 512K;
```

如果你的临时表空间存在，但临时数据文件丢失，你可以直接添加它们，如下所示：

```sql
SQL> alter tablespace temp
add tempfile '/u01/dbfile/o18c/temp02.dbf' SIZE 5000M REUSE;
```

## 还原与恢复数据文件

当介质故障仅限于一小部分数据文件时，数据文件级别的恢复和恢复非常有效。对于数据文件级别的恢复，你可以指示 RMAN 使用数据文件名或数据文件号进行还原和恢复。对于不与 `SYSTEM` 或 `UNDO` 表空间关联的数据文件，你可以在数据库保持打开状态时选择进行还原和恢复。但是，当数据库打开时，你必须首先将任何正在还原和恢复的数据文件脱机。

### 数据库打开时还原和恢复数据文件

使用 `RESTORE DATAFILE` 和 `RECOVER DATAFILE` 命令在数据文件级别进行还原和恢复。当数据库打开时，要求你将任何试图还原和恢复的数据文件脱机。此示例在数据库打开时还原和恢复数据文件：

```bash
RMAN> alter database datafile 4, 5 offline;
RMAN> restore datafile 4, 5;
RMAN> recover datafile 4, 5;
RMAN> alter database datafile 4, 5 online;
```

> **提示**
> 使用 RMAN 的 `REPORT SCHEMA` 命令列出数据文件名称和文件编号。你也可以查询 `V$DATAFILE` 的 `NAME` 和 `FILE#` 列来获取名称和编号。

你也可以指定要还原和恢复的数据文件的名称；例如，

