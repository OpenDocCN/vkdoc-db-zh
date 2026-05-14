# 修改和管理物化视图

## 使用 `ON PREBUILT TABLE` 子句修改物化视图

如果您最初使用 `ON PREBUILT TABLE` 子句创建了物化视图（MV），那么在保留基表时，可以执行与上一节类似的操作。以下是修改使用 `ON PREBUILT TABLE` 子句创建的物化视图的步骤：

1.  更改基表。
2.  删除物化视图。对于建立在预建表上的物化视图，此操作不会删除底层表。
3.  更改预建表。
4.  在预建表上重新创建物化视图。

下面是一个简单的示例来说明此过程。首先，更改基表：
```sql
SQL> alter table sales add(sales_loc varchar2(30));
```

然后，删除物化视图：
```sql
SQL> drop materialized view sales_mv;
```

对于在预建表上创建的物化视图，此操作不会删除底层表——只会删除物化视图对象。接下来，向预建表添加列：
```sql
SQL> alter table sales_mv add(sales_loc varchar2(30));
```

现在，您可以重新构建物化视图，使用添加了新列的预建表：
```sql
SQL> create materialized view sales_mv
on prebuilt table
refresh with primary key
complete on demand as
select sales_id, sales_amt, sales_dtt, sales_loc
from sales;
```

此过程的优点是允许您修改物化视图定义而无需删除底层表。您需要删除物化视图、更改底层表，然后使用新定义重新创建物化视图。如果底层表包含大量数据，此方法可以避免不必要的停机时间。

正如上一节提到的，您需要注意，如果在重新构建物化视图操作期间有任何针对基表的 DML 活动，当您尝试刷新物化视图时，这些事务不会反映在物化视图中。

## 切换物化视图的重做日志记录

回想一下，物化视图有一个底层数据库表。当您刷新物化视图时，这会启动底层表中的事务，从而生成重做（就像常规数据库表一样）。如果发生数据库故障，您可以恢复与物化视图关联的所有事务。

默认情况下，创建物化视图时启用重做日志记录。您可以指定在刷新物化视图时不记录重做。要启用 `nologging`，请使用 `NOLOGGING` 选项创建物化视图：
```sql
SQL> create materialized view sales_mv
nologging
refresh with primary key
fast on demand as
select sales_id ,sales_amt, sales_dtt
from sales;
```

您也可以将现有物化视图更改为 `nologging` 模式：
```sql
SQL> alter materialized view sales_mv nologging;
```

如果要重新启用记录，请执行以下操作：
```sql
SQL> alter materialized view sales_mv logging;
```

要验证物化视图是否已切换到 `NOLOGGING`，可以查询 `USER_TABLES` 视图：
```sql
SQL> select a.table_name, a.logging
from user_tables a
,user_mviews b
where a.table_name = b.mview_name;
```

启用 `nologging` 的优点是刷新操作执行得更快。刷新机制使用直接路径插入，当与 `NOLOGGING` 结合使用时，可以消除大部分重做生成。最大的缺点是，如果在物化视图刷新后不久发生介质故障，您将无法恢复物化视图中的数据。在这种情况下，首次尝试访问物化视图时，您会收到类似以下的错误：
```sql
ORA-01578: ORACLE data block corrupted (file # 5, block # 899)
ORA-01110: data file 5: '/u01/dbfile/o12c/users02.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
```

如果收到上述错误，那么您很可能必须重新构建物化视图才能再次访问数据。在许多环境中，这可能是可以接受的。您通过不为物化视图生成重做来节省数据库资源，但缺点是发生故障时恢复过程更长，需要您重新构建物化视图。

> **注意**
> 如果您的数据库处于 `force logging` 模式，则 `NOLOGGING` 子句无效。`force logging` 模式在使用 Data Guard 的环境中是必需的。

## 调整并行度

有时，创建物化视图时设置了较高的并行度以提高创建过程的性能：
```sql
SQL> create materialized view sales_mv
parallel 4
refresh with primary key
fast on demand as
select sales_id ,sales_amt, sales_dtt
from sales;
```

创建物化视图后，您可能不需要与底层表关联的相同并行度。这很重要，因为针对物化视图的查询将启动并行执行线程。换句话说，您可能需要并行度来快速构建物化视图，但不希望后续查询物化视图时使用并行度。您可以按如下方式更改物化视图的并行度：
```sql
SQL> alter materialized view sales_mv parallel 1;
```

您可以通过查询 `USER_TABLES` 来检查并行度：
```sql
SQL> select table_name, degree from user_tables where table_name= upper('&mv_name');
```

## 移动物化视图

随着操作系统环境条件的变化，您可能需要将物化视图从一个表空间移动到另一个表空间。在这些情况下，请使用 `ALTER MATERIALIZED VIEW...MOVE TABLESPACE` 语句。此示例将与物化视图关联的表移动到不同的表空间：
```sql
SQL> alter materialized view sales_mv move tablespace users;
```

如果与物化视图表关联有任何索引，移动操作会使这些索引不可用。您可以按如下方式检查索引状态：
```sql
SQL> select a.table_name, a.index_name, a.status
from user_indexes a
,user_mviews  b
where a.table_name = b.mview_name;
```

移动表后，必须重新构建任何关联的索引；例如：
```sql
SQL> alter index sales_pk2 rebuild;
```

## 管理物化视图日志

快速刷新物化视图需要物化视图日志。物化视图日志是一个表，用于存储主（基）表的 DML 信息。物化视图日志与主表在同一个数据库中创建，由拥有主表的同一用户创建。您需要 `CREATE TABLE` 权限才能创建物化视图日志。

物化视图日志由 Oracle 内部触发器填充（您无法控制此触发器）。在对主表执行 `INSERT`、`UPDATE` 或 `DELETE` 后，此内部触发器会向物化视图日志插入一行。您可以通过查询 `DBA/ALL/USER_INTERNAL_TRIGGERS` 来查看正在使用的内部触发器。

一个物化视图日志仅与一个表关联，并且每个主表只能为其定义一个物化视图日志。您可以在表或其他物化视图上创建物化视图日志。多个快速刷新物化视图可以使用一个物化视图日志。

在物化视图执行快速刷新后，物化视图日志中不再需要的任何记录都将被删除。如果多个物化视图使用一个物化视图日志，则只有当所有快速刷新物化视图都不再需要记录时，才会从物化视图日志中清除记录。

下表定义了与物化视图日志相关的术语。本章后续与物化视图日志相关的章节中会引用这些术语。

**表 15-3 物化视图日志术语和功能**

| 术语 | 含义 |
| :--- | :--- |
| 物化视图 (MV) 日志 | 跟踪 MV 基表 DML 更改的数据库对象；快速刷新所必需 |
| 主键 MV 日志 | 使用基表主键跟踪 DML 更改的 MV 日志 |
| `ROWID` MV 日志 | 使用基表 `ROWID` 跟踪 DML 更改的 MV 日志 |
| `Commit SCN MV log` | 基于提交 SCN 而不是时间戳的 MV 日志 |
| `Object ID` | 用于跟踪 DML 更改的对象标识符 |
| `Filter column` | 由 MV 子查询引用的非主键列；某些快速刷新场景所必需 |
| `Join column` | 在子查询 `WHERE` 子句中定义连接的非主键列；某些快速刷新场景所必需 |
| `Sequence` | 某些快速刷新场景所需的序列值 |
| `New values` | 指定在 MV 日志中记录旧值和新值；单表聚合视图符合快速刷新条件所必需 |

## 创建物化视图日志

