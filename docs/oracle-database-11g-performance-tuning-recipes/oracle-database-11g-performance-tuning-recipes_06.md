# 工作原理

直接路径插入相比常规插入语句具有两个性能优势：

*   如果指定了 `NOLOGGING`，则生成的重做量极少。
*   缓冲区缓存被绕过，数据直接加载到数据文件中。这可以显著提升加载性能。

`NOLOGGING` 功能仅针对直接路径操作最小化重做日志的生成。对于直接路径插入，`NOLOGGING` 选项能显著提高加载速度。有一种误解认为 `NOLOGGING` 会消除表上所有 DML 操作的重做生成。这是不正确的。`NOLOGGING` 功能从不影响常规 `INSERT`、`UPDATE`、`MERGE` 和 `DELETE` 语句的重做生成。

减少重做生成的一个缺点是，如果在数据加载后（且在备份该表之前）发生故障，你将无法恢复通过 `NOLOGGING` 创建的数据。如果你可以容忍一定的数据丢失风险，那么可以使用 `NOLOGGING`，但要在数据加载后尽快备份该表。如果你的数据至关重要，则不要使用 `NOLOGGING`。如果你的数据可以轻松重新创建，那么在尝试提高大批量数据加载的性能时，`NOLOGGING` 是可取的。

如果你在以 `NOLOGGING` 模式填充表之后（且在备份该表之前）发生了介质故障，会怎样？在执行还原和恢复操作后，表似乎已经还原：

```sql
SQL> desc f_regs;

Name                                      Null?    Type
----------------------------------------- -------- ----------------------------
REG_ID                                             NUMBER
REG_NAME                                           VARCHAR2(2000)
```

然而，当执行扫描表中每个数据块的查询时，会抛出错误。

```sql
SQL> select * from f_regs;
```

这表明数据文件中存在逻辑损坏：

```
ORA-01578: ORACLE data block corrupted (file # 10, block # 198)
ORA-01110: data file 10: '/ora01/dbfile/O11R2/users201.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
```

如上述输出所示，表中的数据是不可恢复的。仅在数据不重要或可以在数据创建后立即备份的场景下使用 `NOLOGGING`。

![Tip icon](img/square.jpg) **提示** 如果你使用 RMAN 备份数据库，可以通过 `REPORT UNRECOVERABLE` 命令报告不可恢复的数据文件。

`NOLOGGING` 有一些需要解释的特殊性。你可以在数据库、表空间和对象级别指定日志记录特性。如果你的数据库已启用强制日志记录模式，那么这将覆盖为表指定的任何 `NOLOGGING` 设置。如果你在表空间级别指定了日志记录子句，它会为任何未显式使用日志记录子句的 `CREATE TABLE` 语句设置默认的日志记录模式。

你可以通过以下方式验证数据库的日志记录模式：

```sql
SQL> select name, log_mode, force_logging from v$database;
```

下一条语句验证表空间的日志记录模式：

```sql
SQL> select tablespace_name, logging from dba_tablespaces;
```

而这个示例则验证表的日志记录模式：

```sql
SQL> select owner, table_name, logging from dba_tables where logging = 'NO';
```

如何判断 Oracle 是否为某个操作记录了重做日志？一种方法是测量在启用日志记录和在 `NOLOGGING` 模式下操作所产生的重做量。如果你有一个用于测试的开发环境，可以在事务进行时监控重做日志切换的频率。另一个简单的测试是测量有日志记录和无日志记录时操作所需的时间。在 `NOLOGGING` 模式下执行的操作应该更快，因为加载期间生成的重做量极少。

