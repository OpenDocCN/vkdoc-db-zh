# PDB 管理操作

## 重命名 PDB

偶尔，你可能需要重命名一个 PDB。例如，数据库最初可能被错误命名，或者你不再使用该数据库并希望在其名称后附加一个`_OLD`。要重命名一个可插拔数据库，首先以具有`SYSDBA`权限的账户连接到它：

```sql
$ sqlplus pdbsys/foo@invpdb as sysdba
```

接下来，停止 PDB 并以受限模式重新启动它：

```sql
SQL> shutdown immediate;
SQL> startup restrict;
```

现在，可以重命名该可插拔数据库：

```sql
SQL> alter pluggable database INVPDB rename global_name to INVPDB_OLD;
```

## 限制 PDB 的空间消耗

你可以对一个 PDB 可以消耗的磁盘空间总量设置限制。虽然 PDB 数据文件的最大尺寸可以做到这一点，但这涉及到多个表空间和数据文件的情况。数据库的尺寸应可通过 ASM 磁盘组或文件系统容量得知。

在此示例中，对一个可插拔数据库设置了 100GB 的总体限制。首先，以`SYS`身份连接到该可插拔数据库：

```sql
$ sqlplus pdbsys/foo@speed2:1521/salespdb as sysdba
```

然后，修改可插拔数据库的最大尺寸限制。此命令将可插拔数据库的大小限制为最大 100GB：

```sql
SQL> alter pluggable database salespdb storage(maxsize 100G);
```

空间并不是唯一可以被 PDB 限制的资源。CPU 和内存可以通过在 PDB 中使用参数或使用资源管理器计划来限制。作为 PDB 中的特权用户，你可以修改系统设置，使`CPU_COUNT`等于小于 CDB 总`CPU_COUNT`的值。这将不允许该 PDB 使用超过那些 CPU 资源。对于内存也是如此，在 PDB 中设置内存限制参数，同样小于 CDB 的设置。连接到 PDB 并使用以下修改语句：

```sql
SQL> alter system set CPU_COUNT = 2 scope = both;
SQL> alter system set SGA_TARGET = 16GB scope = both;
```

## 限制在 PDB 级别对 SYSTEM 的更改

在管理 PDB 时，可以在 PDB 级别更改参数，如上所述。你也可以在 CDB 级别为 PDB 更改这些参数。可以限制这些更改，使得只有 CDB 管理员才能修改 PDB 的这些设置。这将允许 CDB DBA 了解 CDB 中有多少个 PDB 并进行管理。即使每个 PDB 中的`sysdba`用户在 PDB 中拥有 DBA 权限并可以管理 PDB，这些更改也将被保留。

这是通过`PDB_LOCKDOWN`完成的。创建锁定配置文件，为其设置 PDB，并将命令添加到配置文件中以限制 PDB DBA 更改诸如 CPU、内存等设置的配置。

在 CDB 中，以下是创建`PDB_LOCKDOWN`的快速概览：

```sql
SQL> CREATE LOCKDOWN PROFILE pdbprofile1;
SQL> ALTER SYSTEM SET PDB_LOCKDOWN=pdbprofile1;
```

并且你可以查看`PDB_LOCKDOWN`中的参数：

```sql
SQL> SHOW PARAMETER PDB_LOCKDOWN
```

Oracle 18c 具有动态锁定配置文件，允许额外的参数设置、资源管理器计划和选项。它们是动态的，因为不需要重启 PDB 并立即生效。

## 查看 PDB 历史记录

如果你需要查看 PDB 的创建时间，可以查询`CDB_PDB_HISTORY`视图，如下所示：

```sql
SQL> COL db_name FORM A10
SQL> COL con_id FORM 999
SQL> COL pdb_name FORM A15
SQL> COL operation FORM A16
SQL> COL op_timestamp FORM A10
SQL> COL cloned_from_pdb_name FORMAT A15
--
SQL> SELECT db_name, con_id, pdb_name, operation,
    op_timestamp, cloned_from_pdb_name
    FROM cdb_pdb_history
    WHERE con_id > 2
    ORDER BY con_id;
```

以下是一些示例输出：

```
DB_NAME    CON_ID PDB_NAME       OPERATION      OP_TIMESTA  CLONED_FROM_PDB
---------- ------ -------------- -------------- ----------- ---------------
CDB             3 SALESPDB       CREATE         04-DEC-12   PDB$SEED
CDB             4 HRPDB          CREATE         10-FEB-13   PDB$SEED
```

通过这种方式，你可以确定 PDB 是何时创建的以及来自什么源。

## 删除 PDB

偶尔，你可能需要删除一个 PDB。你可能这样做是因为不再需要该 PDB，或者因为你正在将其传输（拔出/插入）到不同的 CDB，并希望从原始 CDB 中删除该 PDB。如果需要移除 PDB，可以通过两种方式完成：

*   删除 PDB 及其数据文件。
*   删除 PDB 但将其数据文件保留在原地。

如果你计划不再使用该 PDB，则可以将其删除并指定同时删除数据文件。如果你计划将该 PDB 插入不同的 CDB，那么当然不要删除数据文件，因为这样做会将其从磁盘上移除。

要删除 PDB，首先以特权账户连接到根容器，并关闭 PDB：

```sql
SQL> alter pluggable database dkpdb close immediate;
```

此示例删除 PDB 及其数据文件：

```sql
SQL> drop pluggable database dkpdb including datafiles;
```

如果成功，你将看到此消息：

```
Pluggable database dropped.
```

下一个示例删除 PDB 但不移除数据文件。如果你要将可插拔数据库移动到不同的 CDB，可能需要这样做：

```sql
SQL> drop pluggable database dkpdb;
```

通过这种方式，PDB 与 CDB 解除关联，但其数据文件在磁盘上保持完整。

## 可刷新克隆 PDB

在 Oracle 18c 中，PDB 可以定期从源 PDB 刷新。这有助于克隆那些需要很长时间创建的情况，并为故障转移、负载均衡以及只需管理两个 CDB 而更新另一个 CDB 中的 PDB。角色也可以互换，使源 PDB 成为克隆，克隆成为源 PDB。这基本上取代了 PDB 的 Data Guard 故障转移。

PDB 以刷新模式创建并配置克隆。这允许自动刷新克隆 PDB 并为其设置发生间隔。也可以手动触发刷新。要创建克隆：

```sql
SQL> CREATE PLUGGABLE DATABASE pdbclone1 REFRESH MODE EVERY 5 MINUTES
```

将刷新模式更改为手动：

```sql
SQL> ALTER PLUGGABLE DATABASE pdbclone1 REFRESH MODE MANUAL;
```

可以在克隆 PDB 容器中使用以下命令切换克隆：

```sql
SQL> ALTER PLUGGABLE DATABASE REFRESH MODE MANUAL FROM pdb1@CDB1_dblink SWITCHOVER ;
```

## 云中的数据库

Oracle 数据库可以在不同的云平台上创建。有模板、容器和其他方式来安装 Oracle 软件，作为基础设施即服务(IaaS)或平台即服务(PaaS)的一部分，以创建数据库和管理环境。

Oracle 自治数据库云是 Oracle 在其公有云中提供的自治数据库产品。它由 Oracle 18c 驱动，最初是自治数据仓库云(ADWC)，现在包括自治事务处理数据库(ATP)。这个自驾驶数据库通过修补的形式提供了维护、性能和安全方面的若干自动化流程。使用 Oracle Cloud 的优势在于，Oracle 软件和选项可用并已实施，随时可用。快速版为你提供了本章前面讨论的 PDB。建议在云实现中创建 CDB 服务和多个 PDB。云中的访问可能不包括服务器访问，这就是为什么通过网络使用`sysdba`用户连接对于 CDB 和 PDB 很重要。在云中或本地管理用户、对象和其他容器的其他任务将是相同的。

自治数据库实际上可以进行试驾，理解 CDB 和 PDB 概念以及用户和管理将简化云中的测试，并支持任何迁移和未来的云环境。

## 总结

Oracle Database 12c 新引入的 PDB 是存在于 CDB 内部的一组数据文件和元数据。PDB 具有几个有趣的架构特性：


