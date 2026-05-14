# 检查数据文件和备份中的损坏情况

你可以使用 RMAN 来检查数据文件、归档日志和控制文件中的损坏情况。你还可以验证备份集是否可以恢复。RMAN 的 `VALIDATE` 命令用于执行这类完整性检查。运行 `VALIDATE` 命令有三种方式：
*   `VALIDATE`
*   `BACKUP...VALIDATE`
*   `RESTORE...VALIDATE`

**注意**
独立的 `VALIDATE` 命令在 Oracle Database 11g 及更高版本中可用。`BACKUP...VALIDATE` 和 `RESTORE...VALIDATE` 命令在 Oracle Database 10g 及更高版本中可用。

## 使用 VALIDATE

`VALIDATE` 命令可以作为独立命令使用，用于检查数据库数据文件、归档日志文件、控制文件、`spfile` 以及备份集片中是否存在丢失文件或物理损坏。例如，此命令将验证所有数据文件和控制文件：
```
RMAN> validate database;
```

你也可以仅验证控制文件，如下所示：
```
RMAN> validate current controlfile;
```

你可以像这样验证归档日志文件：
```
RMAN> validate archivelog all;
```

你可能希望将所有先前的完整性检查合并到一个命令中，如下所示：
```
RMAN> validate database include current controlfile plus archivelog;
```

在正常情况下，`VALIDATE` 命令仅检查物理损坏。你可以通过使用 `CHECK LOGICAL` 子句来指定同时检查逻辑损坏：
```
RMAN> validate check logical database include current controlfile plus archivelog;
```

`VALIDATE` 有多种用途。这里还有一些例子：
```
RMAN> validate database skip offline;
RMAN> validate copy of database;
RMAN> validate tablespace system;
RMAN> validate datafile 3 block 20 to 30;
RMAN> validate spfile;
RMAN> validate backupset ;
RMAN> validate recovery area;
```

如果你正在使用 Oracle Database 12c 的可插拔数据库功能，你可以验证容器内的特定数据库。在以 `SYS` 身份连接到根容器时，验证任何关联的可插拔数据库：
```
RMAN> validate pluggable database salespdb;
```

如果 RMAN 检测到任何损坏的块，`V$DATABASE_BLOCK_CORRUPTION` 视图会被填充。此视图包含有关文件号、块号以及受影响块数的信息。你可以使用此信息来执行块级别的恢复（详见第 19 章）。

**注意**
物理损坏是指对块的更改导致其内容与 Oracle 预期的物理格式不匹配。默认情况下，RMAN 在备份、恢复和验证数据文件时会检查物理损坏。对于逻辑损坏，块的格式是正确的，但其内容与 Oracle 预期的不一致，例如在行片段或索引条目中。

## 使用 BACKUP...VALIDATE

`BACKUP...VALIDATE` 命令与 `VALIDATE` 命令非常相似，它也可以检查数据文件是否可用以及数据文件是否包含任何损坏的块；例如，
```
RMAN> backup validate database;
```

此命令实际上并不创建任何备份文件；它只读取数据文件并检查损坏情况。与 `VALIDATE` 命令类似，`BACKUP VALIDATE` 默认只检查物理损坏。你也可以指示它同时检查逻辑损坏，如下所示：
```
RMAN> backup validate check logical database;
```

以下是 `BACKUP...VALIDATE` 命令的一些变体：
```
RMAN> backup validate database current controlfile;
RMAN> backup validate check logical database current controlfile plus archivelog;
```

同样与 `VALIDATE` 命令类似，如果 `BACKUP...VALIDATE` 检测到任何损坏块，也会填充 `V$DATABASE_BLOCK_CORRUPTION`。此视图中的信息可用于确定哪些块可能通过块级别恢复来还原（详见第 19 章）。

## 使用 RESTORE...VALIDATE

`RESTORE...VALIDATE` 命令用于验证将在恢复操作中使用的备份文件。此命令验证备份集、数据文件副本和归档日志文件：
```
RMAN> restore validate database;
```

使用 `RESTORE...VALIDATE` 时不会实际恢复任何文件。这意味着你可以在数据库联机且可用时运行该命令。

### 使用恢复目录

当你使用恢复目录时，可以在与目标数据库相同的数据库、同一台服务器上创建恢复目录用户。但是，不建议采用这种方法，因为你肯定不希望目标数据库或其所在服务器的可用性影响到恢复目录。因此，你应该在与目标数据库服务器不同的服务器上创建恢复目录数据库。根据规模大小，恢复目录可以用于整个数据库环境，但请记住它通常存储的是控制文件中保存的信息。

## 创建恢复目录

当我使用恢复目录时，我倾向于拥有一个专门仅用于恢复目录的数据库。这确保了恢复目录不会受到其他应用程序所需的维护或停机时间的影响（反之亦然）。



