# 备份数据库

你可以使用 Management Studio 界面、T-SQL、PowerShell 以及第三方工具来备份和还原数据库。本章我们将重点讨论 T-SQL 的实现方式。

### 清单 31-1：执行完整数据库备份

执行完整数据库备份的 T-SQL 语句使用 `BACKUP DATABASE` 命令，并以磁盘作为目标。

```sql
backup database OrderEntryDb
to disk = N'e:\backups\OrderEntry.bak'
with format, init,
    name = N'OrderEntryDb-Full Database Backup',
    stats = 5, checksum, compression;
```

## 第 31 章：备份与还原

SQL Server 允许你在单个文件中存储备份集。然而，使用这种方法时必须非常小心。虽然它减少了磁盘上的文件数量并简化了管理，但可能会覆盖现有备份并导致备份链失效。

你的备份位置设计应考虑在发生灾难时，需要通过网络复制的数据量应尽可能减少。不要将来自不同日志链的备份存储在同一个文件中。此外，不要将差异备份与其他冗余的差异备份和/或日志备份存储在一起。这既能减小备份文件的大小，也能在灾难发生时减少通过网络复制文件所需的时间。

`FORMAT` 和 `INIT` 选项告知 SQL Server 覆盖备份文件中的所有现有备份。

`CHECKSUM` 选项强制 SQL Server 验证数据页上的校验和，并为备份文件生成校验和。这有助于验证数据页在保存到磁盘后未被 I/O 子系统损坏。但是，此选项不应替代使用 `DBCC CHECKDB` 命令进行的常规数据库一致性检查。`BACKUP WITH CHECKSUM` 不测试数据库对象和分配映射页的完整性，也不测试那些没有生成 `CHECKSUM` 的页。

最后，`COMPRESSION` 选项强制 SQL Server 压缩备份。备份压缩可以显著减小备份文件的大小，尽管在备份和还原过程中会使用更多的 CPU 资源。建议使用备份压缩，除非系统 CPU 负载极高，或者数据库已加密。在后一种情况下，备份压缩不会带来任何空间节省。

备份压缩在 SQL Server 2008R2 及以上版本的企业版和标准版，以及 SQL Server 2008 的企业版中可用。值得一提的是，每个版本的 SQL Server 都可以还原压缩的备份。

> **注意**：你可以在 [`technet.microsoft.com/en-us/library/ms186865.aspx`](http://technet.microsoft.com/en-us/library/ms186865.aspx) 查看所有可用的 `BACKUP` 命令选项。

### 清单 31-2：执行差异数据库备份

你可以使用 `DIFFERENTIAL` 选项来执行差异备份。

```sql
backup database OrderEntryDb
to disk = N'e:\backups\OrderEntry.bak'
with differential, noformat, noinit,
    name = N'OrderEntryDb-Differential Database Backup',
    stats = 5, checksum, compression;
```

现在，我们的备份文件 `OrderEntry.bak` 包含两个备份：一个 `FULL`（完整）备份和一个 `DIFFERENTIAL`（差异）备份。最后，清单 31-3 展示了如何通过将其放入另一个文件来执行事务日志备份。

### 清单 31-3：执行事务日志备份

```sql
backup log OrderEntryDb
to disk = N'e:\backups\OrderEntry.trn'
with format, init,
    name = N'OrderEntryDb-Transaction Log Backup',
    stats = 5, checksum, compression;
```

你必须被授予 `BACKUP DATABASE` 和 `BACKUP LOG` 权限才能执行备份操作。默认情况下，这些权限授予 `sysadmin` 服务器角色、`db_owner` 和 `db_backupoperator` 数据库角色的成员。此外，SQL Server 启动账户应该具有足够的



向指定位置写入备份文件的权限。

