# 使用 RMAN 管理 Oracle 数据库

## 故障优先级更改

```
Failure ID Priority Status    Time Detected Summary
---------- -------- --------- ------------- -------
5          HIGH     OPEN      12-JAN-18     One or more non-system datafiles
are missing
```

接下来，使用 `CHANGE FAILURE` 命令将优先级从 `HIGH` 更改为 `LOW`：

```
RMAN> change failure 5 priority low;
```

系统将提示您确认是否真的要更改优先级：

```
Do you really want to change the above failures (enter YES or NO)?
```

如果您确实要更改优先级，请键入 `YES` 并按 Enter 键。如果您再次运行 `LIST FAILURE` 命令，您将看到优先级现已更改为 `LOW`：

```
RMAN> list failure low ;
```

## 使用 RMAN 停止/启动 Oracle

您可以使用 RMAN 以几乎与 SQL*Plus 相同的方法来停止和启动数据库。在执行还原和恢复操作时，通常在 RMAN 内部停止和启动数据库更为方便。以下 RMAN 命令可用于停止和启动数据库：

*   `SHUTDOWN`
*   `STARTUP`
*   `ALTER DATABASE`

### 关闭数据库

RMAN 中的 `SHUTDOWN` 命令与 SQL*Plus 中的工作方式相同。有四种类型的关闭模式：`ABORT`、`IMMEDIATE`、`NORMAL` 和 `TRANSACTIONAL`。我通常首先尝试使用 `SHUTDOWN IMMEDIATE` 来停止数据库。以下是一些示例：

```
RMAN> shutdown immediate;
RMAN> shutdown abort;
```

如果您未指定关闭选项，则默认为 `NORMAL`。使用 `NORMAL` 模式关闭数据库很少可行，因为此模式会等待当前连接的用户在闲暇时断开连接。

#### 启动数据库

与 SQL*Plus 一样，您可以组合使用 RMAN 中的 `STARTUP` 和 `ALTER DATABASE` 命令来逐步引导数据库通过启动阶段，如下所示：

```
RMAN> startup nomount;
RMAN> alter database mount;
RMAN> alter database open;
```

这是另一个示例：

```
RMAN> startup mount;
RMAN> alter database open;
```

如果要以受限访问模式启动数据库，请使用 `DBA` 选项：

```
RMAN> startup dba;
```

提示：从 Oracle Database 12c 开始，您可以直接在 RMAN 内运行所有 SQL 语句，而无需指定 RMAN `sql` 命令。

## 完整恢复

如第 16 章所述，术语 `complete recovery` 表示您可以还原在故障发生前已提交的所有事务。`Complete recovery` 并不意味着要还原和恢复数据库中的所有数据文件。例如，如果您有一个数据文件发生介质故障，并且您还原并恢复了该数据文件，那么您正在执行的就是完全恢复。对于完全恢复，必须满足以下条件：

*   您的数据库处于归档日志模式。
*   您拥有发生介质故障的数据文件的良好基线备份。
*   您拥有自上次备份以来生成的任何必需重做。
*   所有归档重做日志从上次备份开始时起都存在。
*   RMAN 可用于恢复的任何增量备份均可用（如果使用）。
*   包含尚未归档的事务的联机重做日志可用。

如果您遇到了介质故障，并且拥有执行完全恢复所需的文件，那么您就可以还原和恢复数据库。

## 测试恢复与恢复

您可以在实际执行还原和恢复之前，确定 RMAN 将使用哪些文件进行还原和恢复。您还可以指示 RMAN 验证将用于还原和恢复的备份文件的完整性。

### 预览恢复所用备份

使用 `RESTORE...PREVIEW` 命令列出 RMAN 将用于还原和恢复数据库数据文件的备份和归档重做日志文件。`RESTORE...PREVIEW` 命令不会实际还原任何文件，而是列出将用于还原操作的备份文件。此示例详细预览了整个数据库还原和恢复所需的备份：

```
RMAN> restore database preview;
```

您还可以在摘要详细信息级别预览所需的备份文件：

```
RMAN> restore database preview summary;
```

以下是部分输出片段：

```
List of Backup Sets
===================
BS Key  Type LV Size       Device Type Elapsed Time Completion Time
------- ---- -- ---------- ----------- ------------ ---------------
224     Full    775.37M    DISK        00:02:22     12-JAN-18
BP Key: 229  Status: AVAILABLE  Compressed: NO  Tag: TAG20130112T120713
Piece Name: /u02/O18C/rman/r29gnv7q7i_1_1.bk
List of Datafiles in backup set 224
File LV Type Ckp SCN    Ckp Time  Name
---- -- ---- ---------- --------- ----
1       Full 4586940    12-JAN-18 /u01/dbfile/o18c/system01.dbf
3       Full 4586940    12-JAN-18 /u01/dbfile/o18c/undotbs01.dbf
4       Full 4586940    12-JAN-18 /u01/dbfile/o18c/users01.dbf
```

以下是预览还原和恢复所需备份的更多示例：

```
RMAN> restore tablespace system preview;
RMAN> restore archivelog from time 'sysdate -1' preview;
RMAN> restore datafile 1, 2, 3 preview ;
```

### 恢复前验证备份文件

您可以在不实际还原任何内容的情况下，对备份文件执行多个级别的验证。如果您只想让 RMAN 验证文件是否存在并检查文件头，请使用 `RESTORE...VALIDATE HEADER` 命令，如下所示：

```
RMAN> restore database validate header;
```

此命令仅验证备份文件的存在并检查文件头。您可以通过 `RESTORE...VALIDATE` 命令（不带 `HEADER` 子句）进一步指示 RMAN 验证还原数据库数据文件所需的备份文件中块的完整性。同样，RMAN 在此模式下不会还原任何数据文件：

```
RMAN> restore database validate;
```

此命令仅检查备份文件中的物理损坏。您还可以检查逻辑损坏（以及物理损坏），如下所示：

```
RMAN> restore database validate check logical;
```

以下是使用 `RESTORE...VALIDATE` 的其他一些示例：

```
RMAN> restore datafile 1,2,3 validate;
RMAN> restore archivelog all validate;
RMAN> restore controlfile validate;
RMAN> restore tablespace system validate;
```

### 测试介质恢复

前面的部分介绍了报告和验证还原操作。您还可以通过 `RECOVER...TEST` 命令指示 RMAN 验证恢复过程。在执行测试恢复之前，您需要确保要恢复的数据文件处于脱机状态。对于处于测试模式下恢复的任何联机数据文件，Oracle 都会抛出错误。

在此示例中，首先还原表空间 `USERS`，然后执行试用恢复：

```
RMAN> connect target /
RMAN> startup mount;
RMAN> restore tablespace users;
RMAN> recover tablespace users test;
```

如果恢复所需的任何归档重做日志缺失，则会抛出以下错误：

```
RMAN-06053: unable to perform media recovery because of missing log
RMAN-06025: no backup of archived log for thread 1 with sequence 6 ...
```

如果恢复测试成功，您将看到类似以下的消息，表明重做的应用已通过测试但未实际应用：

```
ORA-10574: Test recovery did not corrupt any data block
ORA-10573: Test recovery tested redo from change 4586939 to 4588462
ORA-10572: Test recovery canceled due to errors
ORA-10585: Test recovery can not apply redo that may modify control file
```

以下是测试恢复过程的一些其他示例：

