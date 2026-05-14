# 恢复与恢复操作

如果只有与根容器关联的数据文件受损，那么你可以在根级别进行恢复与恢复操作。在此示例中，正在恢复根容器的系统数据文件，因此数据库不得处于打开状态。以下命令通过关键字 `root` 指示 RMAN 仅恢复与根容器数据库关联的数据文件：

```bash
$ rman target /
RMAN> startup mount;
RMAN> restore database root;
RMAN> recover database root;
RMAN> alter database open;
```

在前面的代码中，`restore database root` 命令指示 RMAN 仅恢复与根容器数据库关联的数据文件。容器数据库打开后，你必须打开所有关联的可插拔数据库。你可以从根容器执行此操作，如下所示：

```sql
RMAN> alter pluggable database all open;
```

你可以通过此查询检查可插拔数据库的状态：

```sql
SQL> select name, open_mode from v$pdbs;
```

## 恢复与恢复可插拔数据库

你有两个选项来恢复与恢复可插拔数据库：

*   以容器根用户身份连接，并指定要恢复与恢复的可插拔数据库。
*   直接以特权可插拔级别用户身份连接到可插拔数据库，并发出 `RESTORE` 和 `RECOVER` 命令。

第一个示例连接到根容器，并恢复与 `salespdb` 可插拔数据库关联的数据文件。为此，可插拔数据库不得处于打开状态（因为可插拔数据库的系统数据文件也将被恢复与恢复）：

```bash
$ rman target /
RMAN> alter pluggable database salespdb close;
RMAN> restore pluggable database salespdb;
RMAN> recover pluggable database salespdb;
RMAN> alter pluggable database salespdb open;
```

你也可以直接连接到可插拔数据库并执行恢复与恢复操作。直接连接到可插拔数据库时，用户仅能访问与该可插拔数据库关联的数据文件：

```bash
$ rman target sys/foo@salespdb
RMAN> shutdown immediate;
RMAN> restore database;
RMAN> recover database;
RMAN> alter database open;
```

> **注意**
> 当你直接连接到可插拔数据库时，无法在 `RESTORE` 和 `RECOVER` 命令中指定可插拔数据库的名称。这种情况下，你会收到 `RMAN-07536: command not allowed when connected to a Pluggable Database` 错误。

前面的代码仅影响你所连接的可插拔数据库关联的数据文件。可插拔数据库需要关闭才能执行此操作。但是，根容器数据库可以处于打开或已装载状态。此外，你必须使用以特权用户身份连接到可插拔数据库时创建的备份。特权可插拔数据库用户无法访问由根容器数据库特权用户启动的数据文件备份。

## 恢复归档日志文件

RMAN 会在恢复过程中自动恢复其所需的任何归档日志文件。通常你无需手动恢复归档日志文件。但是，在以下任何情况下，你可能需要这样做：

*   你预期稍后执行恢复操作，需要先恢复归档日志文件；思路是如果归档日志文件已恢复，将加快恢复操作。
*   由于介质故障或存储空间问题，需要将归档日志文件恢复到非默认位置。
*   你需要恢复特定的归档日志文件，以便通过 LogMiner 检查它们。

如果你启用了 FRA，则 RMAN 默认将归档日志文件恢复到由初始化参数 `DB_RECOVERY_FILE_DEST` 定义的目标位置。否则，RMAN 使用 `LOG_ARCHIVE_DEST_N` 初始化参数（其中 `N` 通常为 1）来确定恢复归档日志文件的位置。
如果你将归档日志文件恢复到非默认位置，RMAN 会知道它们被恢复到的位置，并在你发出任何后续 `RECOVER` 命令时自动找到这些文件。RMAN 不会恢复它判定磁盘上已有的归档日志文件。即使你指定了非默认位置，如果文件已存在，RMAN 也不会将归档日志文件恢复到磁盘。在这种情况下，RMAN 仅返回一条消息，说明归档日志文件已恢复。使用 `FORCE` 选项可覆盖此行为。
如果你不确定恢复日志文件时要使用的序列号，可以查询 `V$LOG_HISTORY` 视图。

> **提示**
> 请记住，你无法恢复从未备份过的归档日志。另外，如果包含归档日志的备份文件不再可用，你也无法恢复该归档日志。运行 `LIST ARCHIVELOG ALL` 命令可查看磁盘上当前的归档日志，运行 `LIST BACKUP OF ARCHIVELOG ALL` 可验证哪些归档日志文件位于可用的 RMAN 备份中。

### 恢复到默认位置

以下命令将恢复 RMAN 已备份的所有归档日志文件：

```sql
RMAN> restore archivelog all ;
```

如果你想从指定的序列开始恢复，请使用 `FROM SEQUENCE` 子句。你可能希望先运行此查询以确定已生成的最新日志文件和序列号：

```sql
SQL> select sequence#, first_time from v$log_history order by 2;
```

此示例从序列 68 开始恢复所有归档日志文件：

```sql
RMAN> restore archivelog from sequence 68;
```

如果你想恢复一系列归档日志文件，请使用 `FROM SEQUENCE` 和 `UNTIL SEQUENCE` 子句或 `SEQUENCE BETWEEN` 子句，如下所示。以下命令使用线程 1，恢复从序列 68 到序列 78 的归档日志文件：

```sql
RMAN> restore archivelog from sequence 68 until sequence 78 thread 1;
RMAN> restore archivelog sequence between 68 and 78 thread 1;
```

默认情况下，如果归档日志文件已在磁盘上，RMAN 将不会恢复它。你可以使用 `FORCE` 覆盖此行为，如下所示：

```sql
RMAN> restore archivelog from sequence 1 force ;
```

### 恢复到非默认位置

如果你想将归档日志文件恢复到与默认位置不同的位置，请使用 `SET ARCHIVELOG DESTINATION` 子句。以下示例恢复到非默认位置 `/u01/archtemp`。`SET` 命令的选项必须在 RMAN `run{}` 块内执行。

```sql
run{
set archivelog destination to '/u01/archtemp';
restore archivelog from sequence 8 force;
}
```

空间是执行此操作的主要原因，但这类恢复是非常好的测试和实践案例，可用于体验此行为并为“以防万一”的场景记录文档。

## 恢复控制文件

如果你缺少一个控制文件，并且你有多个副本，那么你可以关闭数据库，并通过将一个好的控制文件复制到缺失控制文件的正确位置和名称来简单地恢复缺失或损坏的控制文件（详情参见第 5 章）。如果除了一个文件之外的所有文件都损坏，并且多个副本确实位于不同的磁盘上，此方法有效。如果存在磁盘或控制器故障，则至少有一个控制文件可能仍然可用。RMAN 策略的一部分是针对这些问题对控制文件进行备份。

下面列出了恢复控制文件时的三种典型场景：

*   使用恢复目录
*   使用自动备份
*   指定备份文件名

### 使用恢复目录


