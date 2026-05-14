# 物化视图日志的创建与管理

快速刷新的物化视图需要在主（基）表上创建物化视图（MV）日志。使用 `CREATE MATERIALIZED VIEW LOG` 命令来创建 MV 日志。此示例在 `SALES` 表上创建 MV 日志，并指定使用主键来标识日志中的行：

```sql
SQL> create materialized view log on sales with primary key;
```

您也可以指定存储信息，例如表空间名称：

```sql
SQL> create materialized view log on sales
pctfree 5
tablespace users
with primary key;
```

当您在表上创建 MV 日志时，Oracle 会创建一个表来存储自上次刷新以来对主表的更改。MV 日志表的名称遵循此格式：`MLOG$_<master_table_name>`。您可以使用 SQL*Plus 的 `DESCRIBE` 语句查看 MV 日志的列：

```sql
SQL> desc mlog$_sales;
Name                              Null?    Type
-------------------------------- -------- ----------------------------
SALES_ID                                  NUMBER
SNAPTIME$$                                DATE
DMLTYPE$$                                 VARCHAR2(1)
OLD_NEW$$                                 VARCHAR2(1)
CHANGE_VECTOR$$                           RAW(255)
XID$$                                     NUMBER
```

您可以查询这个底层的 `MLOG$` 表，以确定自上次刷新以来的事务数量。每次刷新后，MV 日志表都会被清除。如果多个物化视图使用同一个 MV 日志，则直到所有相关的物化视图都刷新后，日志表才会被清除。

如果您在具有主键的表上创建 MV 日志，则还会创建一个 `RUPD$_<master_table_name>` 表。此表用于可更新的物化视图。如果您不使用可更新物化视图功能，则此表永远不会被使用，可以忽略它。

创建 MV 日志时，您可以指定使用以下子句之一来唯一标识 MV 日志表中的行：

*   `WITH PRIMARY KEY`
*   `WITH ROWID`
*   `WITH OBJECT ID`

如果主表有主键，请在创建 MV 日志时使用 `WITH PRIMARY KEY`。如果主表没有主键，则必须使用 `WITH ROWID` 来指定使用 `ROWID` 值唯一标识 MV 日志记录。在对象表上创建 MV 日志时，可以使用 `WITH OBJECT ID`。

Oracle 使用 `SNAPTIME$$` 列来确定哪些记录需要刷新或清除，或两者都需要。您可以选择创建基于 `COMMIT SCN` 的 MV 日志（而非基于时间戳）。这种类型的 MV 日志使用事务的 SCN 来确定哪些记录需要应用于任何相关的物化视图。基于 `COMMIT SCN` 的 MV 日志比基于时间戳的 MV 日志更高效。使用 `WITH COMMIT SCN` 子句来实现：

```sql
SQL> create materialized view log on sales with commit scn;
```

您可以通过查询 `USER_MVIEW_LOGS` 来查看 MV 日志是否基于 SCN：

```sql
SQL> select log_table, commit_scn_based from user_mview_logs;
```

> **注意**
> 使用 COMMIT SCN 创建的 MV 日志没有 `SNAPTIME$$` 列。

## 索引 MV 日志列

有时，您可能需要您的快速刷新物化视图具有更好的性能。一种方法是对 MV 日志表的列创建索引。特别是，Oracle 在刷新或清除时会使用 `SNAPTIME$$` 列或主键列，或两者。因此，对这些列创建索引可以提高性能：

```sql
SQL> create index mlog$_sales_idx1 on mlog$_sales(snaptime$$);
SQL> create index mlog$_sales_idx2 on mlog$_sales(sales_id);
```

您不应该仅仅因为认为可能有益就添加索引。只有在已知快速刷新存在性能问题时，才在 MV 日志表上添加索引。请记住，添加索引会消耗数据库中的资源。Oracle 必须为表上的 DML 操作维护索引，并且索引会占用磁盘空间。

## 查看 MV 日志使用的空间

您应考虑定期检查 MV 日志占用的空间。如果使用的空间在增长（并且从未缩小），您可能遇到了物化视图刷新不成功的问题，从而导致 MV 日志从未被清除。以下是检查 MV 日志空间的查询：

```sql
SQL> select segment_name, tablespace_name
,bytes/1024/1024 meg_bytes, extents
from dba_segments
where segment_name like 'MLOG$%'
order by meg_bytes;
```

以下是一些示例输出：

```sql
SEGMENT_NAME         TABLESPACE_NAME                 MEG_BYTES    EXTENTS
-------------------- ------------------------------ ---------- ----------
MLOG$_USERS          MV_DATA                              1609       3218
MLOG$_ASSET_ATTRS    MV_DATA                            3675.5       7351
```

此输出表明几个 MV 日志很可能存在清除问题。在这种情况下，可能有多个物化视图正在使用该 MV 日志，并且其中一个没有按日志清除。

您可能会遇到 MV 日志很长时间未被清除的情况。这可能是因为您有多个物化视图使用同一个 MV 日志，而其中一个物化视图不再成功刷新。当 DBA 构建开发环境并将开发物化视图连接到生产环境时（这不应该发生，但确实会发生），就可能发生这种情况。在之后的某个时间点，DBA 删除了开发数据库。生产环境仍然有关于远程开发物化视图的信息，并且不会清除 MV 日志记录，因为它需要为快速刷新物化视图保留日志数据，而实际上并非如此。

在这些场景中，您应确定哪些物化视图正在使用该日志（请参阅本章后面的“[确定有多少物化视图引用中央 MV 日志](https://example.org/referencing_mviews)”一节），并解决任何问题。问题解决后，检查日志使用的空间，并查看是否可以收缩（请参阅下一节“收缩 MV 日志中的空间”）。在这种情况下，监控物化视图和 MV 日志的空间非常重要，可以提供对这些问题的洞察。

## 收缩 MV 日志中的空间

如果 MV 日志未能成功删除记录，它会变得很大。在您解决问题并从 MV 日志中删除记录后，您可以将 MV 日志表的高水位线设置为一个较高的值。但是，这样做可能会导致性能问题，并且会不必要地消耗磁盘空间。在这种情况下，请考虑收缩 MV 日志使用的空间。

在此示例中，由于关联的物化视图未能成功刷新，`MLOG$_SALES` 在清除记录方面存在问题。此 MV 日志随后变得很大。问题已被识别并解决，现在需要减少日志的空间。要收缩 MV 日志中的空间，首先对相应的 MV 日志 `MLOG$` 表启用行移动：

```sql
SQL> alter table mlog$_sales enable row movement;
```

接下来，发出 `ALTER MATERIALIZED VIEW LOG ON...SHRINK` 语句。请注意，关键字 `ON` 后面的表名是主表的名称：

```sql
SQL> alter materialized view log on sales shrink space;
```

根据收缩的空间量，此语句可能需要很长时间。语句完成后，您可以禁用行移动：

```sql
SQL> alter table mlog$_sales disable row movement;
```

您可以通过运行上一节的查询（从 `DBA_SEGMENTS` 中选择）来验证空间是否已减少。

## 检查 MV 日志的行数

如前所述，有时物化视图的刷新会出现问题，这会导致相应的 MV 日志表中积累大量行。当多个物化视图使用一个 MV 日志，并且其中一个物化视图无法执行快速刷新时，就可能发生这种情况。在这种情况下，MV 日志会持续增长，直到问题得到解决。


