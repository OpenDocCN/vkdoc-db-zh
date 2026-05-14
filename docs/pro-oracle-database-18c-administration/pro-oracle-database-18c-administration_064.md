# Oracle RMAN 数据文件恢复与介质恢复技术

## 在数据库未打开时恢复与恢复数据文件

在此场景中，数据库首先被关闭，然后在 `mount` 模式下启动。你可以在数据库未打开时恢复和恢复任何数据文件。此示例展示了恢复与 `SYSTEM` 表空间相关联的数据文件 1：

```bash
$ rman target /
RMAN> shutdown abort;
RMAN> startup mount;
RMAN> restore datafile 1;
RMAN> recover datafile 1;
RMAN> alter database open;
```

在执行数据文件恢复时，你也可以指定文件名：

```bash
$ rman target /
RMAN> shutdown abort;
RMAN> startup mount;
RMAN> restore datafile '/u01/dbfile/o18c/system01.dbf';
RMAN> recover datafile '/u01/dbfile/o18c/system01.dbf';
RMAN> alter database open ;
```

## 将数据文件恢复到非默认位置

有时会发生故障，导致与某个挂载点关联的磁盘无法操作。在这些情况下，你需要将数据文件恢复和恢复到与其原始位置不同的位置。另一个需要将数据文件恢复到非默认位置的典型场景是，将文件恢复到不同的数据库服务器上，该服务器的挂载点与备份来源服务器的挂载点完全不同。

使用 `SET NEWNAME` 和 `SWITCH` 命令将数据文件恢复到非默认位置。这两个命令都必须在 RMAN `run{}` 块内运行。你可以将使用 `SET NEWNAME` 和 `SWITCH` 视为重命名数据文件的一种方式（类似于 SQL*Plus 的 `ALTER DATABASE RENAME FILE` 语句）。

此示例展示了在恢复和恢复时更改数据文件的位置。首先，将数据库置于 `mount` 模式：

```bash
$ rman target /
RMAN> startup mount;
```

然后，运行以下 RMAN 代码块：

```bash
run{
set newname for datafile 4 to '/u02/dbfile/o18c/users01.dbf';
set newname for datafile 5 to '/u02/dbfile/o18c/users02.dbf';
restore datafile 4, 5;
switch datafile all; # 使用新数据文件位置更新仓库。
recover datafile 4, 5;
alter database open;
}
```

以下是部分输出列表：

```text
datafile 4 switched to datafile copy
input datafile copy RECID=79 STAMP=804533148 file name=/u02/dbfile/o18c/users01.dbf
datafile 5 switched to datafile copy
input datafile copy RECID=80 STAMP=804533148 file name=/u02/dbfile/o18c/users02.dbf
```

如果数据库是打开的，你可以将数据文件脱机，然后设置它们的新名称以进行恢复和恢复，如下所示：

```bash
run{
alter database datafile 4, 5 offline;
set newname for datafile 4 to '/u02/dbfile/o18c/users01.dbf';
set newname for datafile 5 to '/u02/dbfile/o18c/users02.dbf';
restore datafile 4, 5;
switch datafile all; # 使用新数据文件位置更新仓库。
recover datafile 4, 5;
alter database datafile 4, 5 online;
}
```

## 执行块级恢复

块级损坏很少发生，通常由某种 I/O 错误引起。它可以帮助你避免对数据文件进行完整的恢复。我以前确实遇到过这个问题，现在有了块检查和 ASM 自动修复，这种情况可能变得更加罕见。但是，如果你在一个大型数据文件中确实存在孤立的损坏块，能够选择执行块级恢复是很有用的。块级恢复在数据文件中只有少量块损坏时非常有用。如果整个数据文件都需要介质恢复，则块恢复不合适。

每当运行 `BACKUP`、`VALIDATE` 或 `BACKUP VALIDATE` 命令时，RMAN 会自动检测损坏的块。有关损坏块的详细信息可以在 `V$DATABASE_BLOCK_CORRUPTION` 视图中查看。在以下示例中，常规备份作业在输出中报告了一个损坏块：

```text
ORA-19566: exceeded limit of 0 corrupt blocks for file...
```

查询 `V$DATABASE_BLOCK_CORRUPTION` 视图可以指示哪个文件包含损坏：

```sql
SQL> select * from v$database_block_corruption;
FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO     CON_ID
---------- ---------- ---------- ------------------ --------- ----------
4         20          1                  0 ALL ZERO           0
```

执行块级恢复时，数据库可以处于 `mount` 或 `open` 状态。你不必将要恢复的数据文件脱机。你可以指示 RMAN 恢复在 `V$DATABASE_BLOCK_CORRUPTION` 中报告的所有块，如下所示：

```bash
RMAN> recover corruption list;
```

如果成功，将显示以下消息：

```text
media recovery complete...
```

另一种恢复块的方法是指定数据文件和块号，如下所示：

```bash
RMAN> recover datafile 4 block 20;
```

最好使用 `RECOVER CORRUPTION LIST` 语法，因为它将从 `V$DATABASE_BLOCK_CORRUPTION` 视图中清除已恢复的任何块。

> 注意：RMAN 无法对数据文件头（块 1）执行块级恢复。

块级介质恢复允许你保持数据库可用，并减少了平均恢复时间，因为在恢复期间只有损坏的块处于脱机状态。你的数据库必须处于归档日志模式才能执行块级恢复。RMAN 可以从闪回日志（如果可用）中恢复块。如果闪回日志不可用，RMAN 将尝试从完整备份、0 级备份或由 `BACKUP AS COPY` 命令生成的映像副本备份中恢复块。块恢复后，必须有所需的归档日志才能恢复块。RMAN 无法使用增量 1 级（或更高）备份执行块介质恢复。

## 恢复容器数据库及其关联的可插拔数据库

从 Oracle Database 12c 开始，你可以在一个容器数据库中创建可插拔数据库（详见第 22 章）。在处理容器和关联的可插拔数据库时，有三种基本场景：

*   所有数据文件都经历了介质故障（容器根数据文件以及所有关联的可插拔数据库数据文件）。
*   仅与容器根数据库关联的数据文件经历了介质故障。
*   仅与某个可插拔数据库关联的数据文件经历了介质故障。

前面的场景将在以下部分中介绍。

### 恢复和恢复所有数据文件

要恢复和恢复与容器数据库关联的所有数据文件（包括根容器、种子容器和所有关联的可插拔数据库），请使用 RMAN 以具有 `sysdba` 或 `sysbackup` 权限的用户身份连接到容器数据库。由于正在恢复与根系统表空间关联的数据文件，数据库必须在 `mount` 模式（而不是 `open`）下启动：

```bash
$ rman target /
RMAN> startup mount;
RMAN> restore database;
RMAN> recover database;
RMAN> alter database open;
```

请记住，当你打开容器数据库时，默认情况下不会打开关联的可插拔数据库。你可以从根容器中打开它们，如下所示：

```bash
RMAN> alter pluggable database all open;
```

### 恢复和恢复根容器数据文件

