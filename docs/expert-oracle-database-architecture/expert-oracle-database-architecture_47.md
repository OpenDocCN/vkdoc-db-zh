# 实验重现由延迟块清理引起的 ORA-01555 错误

为了说明这一点，我们将创建一个包含许多需要清理的数据块的表。然后，我们将在该表上打开一个游标，并允许许多小事务对另一个表（不是我们刚刚更新并打开了游标的那个表）执行操作。最后，我们将尝试获取游标的数据。现在，我们*知道*游标所需的数据将是“OK”的——我们应该能够看到所有数据，因为对表的修改*在*我们打开游标之前就已经发生并提交了。当我们这次遇到 `ORA-01555` 错误时，那将是由先前描述的延迟块清理问题导致的。为此示例做准备，我们将使用：

*   4MB 的 `UNDO_SMALL` UNDO 表空间。
*   300MB 的 SGA，这样我们可以将一些脏块刷写到磁盘以观察此现象。

在此之前，我们将创建 UNDO 表空间和我们将要查询的“大”表：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create undo tablespace undo_small
datafile '/tmp/undo.dbf' size 4m
autoextend off;
Tablespace created.
SQL> create table big  as
select a.*, rpad('*',1000,'*') data
from all_objects a;
Table Created.
SQL> alter table big add constraint big_pk  primary key(object_id);
Table altered.
SQL> exec dbms_stats.gather_table_stats( user, 'BIG' );
PL/SQL procedure successfully completed.
```

注意

你可能想知道为什么在收集统计信息的调用中，我没有使用 `CASCADE=>TRUE` 来收集主键约束默认创建的索引的统计信息。这是因为每当被索引的表不为空时，`CREATE INDEX` 或 `ALTER INDEX REBUILD` 已经隐含地添加了计算统计信息。因此，创建索引这一行为本身就会产生收集自身统计信息的副作用。没有必要重新收集我们已经拥有的统计信息。

由于我们使用那个大的数据字段，每个块大约能得到六到七行，并且我的 `ALL_OBJECTS` 表有超过 70,000 行，所以之前的表会有很多块。接下来，我们将创建许多小事务将要修改的小表：

```sql
SQL> create table small ( x int, y char(500) );
Table created.
SQL> insert into small select rownum, 'x' from all_users;
25 rows created.
SQL> commit;
Commit complete.
SQL> exec dbms_stats.gather_table_stats( user, 'SMALL' );
PL/SQL procedure successfully completed.
```

现在，我们要弄脏那个大表。我们有一个非常小的 UNDO 表空间，因此我们希望尽可能多地更新这个大表的数据块，同时生成尽可能少的 UNDO。我们将使用一个巧妙的 `UPDATE` 语句来实现这一点。基本上，下面的子查询正在查找每个块上某一行的“第一个”rowid。该子查询将为每个数据库块返回一个 rowid，标识其上的一行。我们将更新该行，设置一个 `VARCHAR2(1)` 字段。这将让我们更新表中的所有块（示例中大约 8000 多个），用将被写出的脏块淹没缓冲区缓存。我们也将确保使用那个小的 UNDO 表空间。为了实现这一点且不超过 UNDO 表空间的容量，我们将设计一个 `UPDATE` 语句，只更新每个块上的“第一行”。内置分析函数 `ROW_NUMBER()` 在此操作中至关重要；它按数据库块为表中的“第一行”分配编号 1，这将是我们要更新的单行：

```sql
SQL> alter system set undo_tablespace = undo_small;
System altered.
SQL> update big
set temporary = temporary
where rowid in
(
select r
from (
select rowid r, row_number() over
(partition by dbms_rowid.rowid_block_number(rowid) order by rowid) rn
from big
)
where rn = 1
);
3064 rows updated.
SQL> commit;
Commit complete.
```

好的，现在我们知道磁盘上有很多脏块。我们肯定写出了一些，因为我们根本没有空间容纳所有这些块。接下来，我们将打开一个游标，但暂时不会获取任何一行。请记住，当我们打开游标时，结果集是预先确定的，因此即使 Oracle 没有实际处理任何数据行，打开该结果集这一行为就固定了结果必须“截至”的时间点。现在，由于我们将要获取的是我们刚刚更新并提交的数据，并且我们知道没有其他人修改这些数据，我们应该能够检索这些行而根本不需要任何 UNDO。但这就是延迟块清理问题显露头角的地方。修改这些块的事务太新了，Oracle 在我们开始之前必须验证它是否已提交，如果我们覆盖了这些信息（也存储在 UNDO 表空间中），查询就会失败。所以，下面是打开游标的操作：

```sql
SQL> variable x refcursor
SQL> exec open :x for select * from big where object_id  !./run.sh
```

`run.sh` 文件是一个 shell 脚本；它只是使用一个命令启动了几个 SQL*Plus 会话：

```bash
$ORACLE_HOME/bin/sqlplus eoda/foo@localhost:1521/PDB1 @test2 1  &
$ORACLE_HOME/bin/sqlplus eoda/foo@localhost:1521/PDB1 @test2 2  &
$ORACLE_HOME/bin/sqlplus eoda/foo@localhost:1521/PDB1 @test2 3  &
...
```

在前面的代码中，每个 SQL*Plus 会话被传递了一个不同的数字（那是数字 1；还有 2、3 等等）。在前面的脚本中，请确保用你环境中的用户名和密码替换用户名和密码。它们每个运行的脚本 `test2.sql` 如下：

```sql
begin
for i in 1 .. 5000
loop
update small set y = i where x= &1;
commit;
end loop;
end;
/
exit
```

所以，我们有几个会话在紧密循环中发起了许多事务。`run.sh` 脚本等待这几个 SQL*Plus 会话完成工作，然后我们返回到我们的会话，即那个带有打开游标的会话。在尝试打印它时，我们观察到以下内容：

```sql
SQL> print x
ORA-01555: snapshot too old: rollback segment number 44 with name
"_SYSSMU44_3913812538$" too small
no rows selected
```

正如我所说，上述情况是罕见的。它需要满足很多条件，所有这些条件必须同时存在才会发生。我们需要存在需要清理的块，而这些块是稀缺的。`DBMS_STATS` 调用收集统计信息会清除它们，因此最常见的原因——大规模批量更新和批量加载——应该不是问题，因为无论如何，在此类操作后都需要分析表。大多数事务倾向于触及缓冲区缓存中不到百分之十的块；因此，它们不会产生需要清理的块。如果你认为你遇到了这个问题，即对一个没有其他 DML 操作应用的表执行 `SELECT` 时引发了 `ORA-01555 error` ，请尝试以下解决方案：

*   首先确保你使用的是“大小合适”的事务。确保你没有比应该的更频繁地提交。
*   使用 `DBMS_STATS` 扫描相关对象，在加载后清理它们。由于块清理是非常大的批量 `UPDATE` 或 `INSERT` 的结果，无论如何都需要这样做。
*   通过给予 UNDO 表空间扩展的空间并增加 `undo_retention` ，允许其增长。这降低了在你的长时间运行查询过程中 UNDO 段事务表槽被覆盖的可能性。这与 `ORA-01555` 错误的另一个原因（两者密切相关；你在查询处理过程中经历了 UNDO 段重用）的解决方案相同。事实上，我重新运行了前面的示例，将 UNDO 表空间设置为每次自动扩展 1MB，`undo_retention` 设置为 900 秒。针对表 `BIG` 的查询成功完成。
*   减少查询的运行时间——对其进行优化。如果可能，这总是好的，所以它可能是你首先尝试的事情。

最后一点建议：在你运行先前实验的数据库中，别忘了将 undo 表空间设回原来的那个。例如：

```sql
SQL> alter system set undo_tablespace=undotbs2;
SQL> drop tablespace undo_small including contents and datafiles;
```

这样能确保你的数据库不是在一个极小的 undo 表空间下运行的。

## 总结

在本章中，我们探讨了如何度量重做（redo）。我们也查看了 `NOLOGGING` 对重做生成的影响。当与直接路径操作（例如，直接路径插入）结合使用时，重做的生成可以显著减少。然而，对于常规的 DML 语句，`NOLOGGING` 子句没有效果。

我们探究了日志切换可能延迟的原因。如果 `DBWn` 还未完成对重做日志所保护数据的检查点操作，或者 `ARCn` 还未完成将重做日志文件复制到归档目的地，Oracle 就不允许覆盖重做日志。这主要是 DBA 需要检测（检查 `alert.log`）和管理的问题。

我们讨论了在临时表中发生的事务如何处理重做。在 12c 及以上版本中，重做量可以减少到几乎为零。对于使用临时表的应用程序，这可能对性能产生积极影响。

在本章中，我们还调查了哪些语句生成最少和最多的 undo。通常，`INSERT` 生成的 undo 量最少，`UPDATE` 生成的比 `INSERT` 多，而 `DELETE` 生成的 undo 最多。

最后，我们探讨了臭名昭著的 `ORA-01555` 错误（快照过旧）的成因。该错误可能因为 undo 表空间大小设置过小而发生。DBA 必须确保 undo 表空间足够大，主要是为了消除这是错误的一个原因。我们还查看了延迟的块清除如何导致问题。如果你正确设置了你的事务大小和 undo 表空间大小，你可能很少会遇到这个错误。调整引发 `ORA-01555` 错误的查询应始终是解决该问题的首选方法之一。

