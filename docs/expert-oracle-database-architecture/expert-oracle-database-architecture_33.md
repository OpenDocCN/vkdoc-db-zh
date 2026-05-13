# SQL> alter table t drop constraint x_not_null;
表已变更。
SQL> alter table t modify x constraint x_not_null not null;
表已变更。
SQL> explain plan for select count(*) from t;
已解释。
SQL> select * from table(dbms_xplan.display(null,null,'BASIC'));

| 标识 | 操作            | 名称  |
|------|----------------|-------|
| 0    | SELECT STATEMENT |       |
| 1    | SORT AGGREGATE   |       |
| 2    | INDEX FULL SCAN  | T_IDX |

所以，底线是，仅在你有明确需求的地方使用可延迟约束。它们会引入微妙的副作用，可能导致物理实现（非唯一索引与唯一索引）或查询计划上的差异——正如刚刚演示的！

## 不良的事务习惯

许多开发人员在处理事务时有一些不良习惯。我经常在那些曾使用过“支持”但不“提倡”使用事务的数据库的开发人员身上看到这一点。例如，在 Informix（默认情况下）、Sybase 和 SQL Server 中，你必须显式地 `BEGIN` 一个事务；否则，每条单独的语句本身就是一个事务。与 Oracle 在离散语句周围包装 `SAVEPOINT` 的方式类似，这些数据库会在每条语句周围包装一个 `BEGIN WORK`/`COMMIT` 或 `ROLLBACK`。这是因为，在这些数据库中，锁是宝贵的资源，并且读者会阻塞写者，反之亦然。为了尝试提高并发性，这些数据库希望你将事务尽可能缩短——有时甚至以牺牲数据完整性为代价。

Oracle 采取了相反的方法。事务总是隐式的，除非应用程序自己实现，否则没有办法实现“自动提交”（有关更多详细信息，请参阅本章后面的“使用自动提交”部分）。在 Oracle 中，每个事务都应在必须提交时提交，且绝不应提前提交。事务应该根据需要尽可能长。诸如锁、阻塞等问题不应真正被视为决定事务大小的驱动力——数据完整性才是你事务大小的*驱动力*。锁不是稀缺资源，并发的数据读者和写者之间也不存在争用问题。这使你能够在数据库中拥有健壮的事务。这些事务的持续时间不必很短——它们应该正好是需要的长度（但不再更长）。事务不是为了方便计算机及其软件；它们是为了保护你的数据。

### 在循环中提交

面对更新多行的任务，大多数程序员会尝试找出某种过程化方式在循环中执行，以便每更新若干行就提交一次。我听过两个*（错误的！）*这样做的理由：

*   频繁提交许多小事务比处理和提交一个大事务更快、更高效。
*   我们没有足够的回滚空间。

这两个理由都是被误导的。此外，过于频繁地提交会使你在更新中途失败时，面临数据库处于“未知”状态的危险。要编写一个在失败时能够顺利重启的进程，需要复杂的逻辑。迄今为止，最好的选择是仅根据业务流程的需要来提交，并相应地调整回滚段的大小。

让我们更详细地看看这些问题。

#### 性能影响

频繁提交通常不会更快——几乎总是单条 SQL 语句执行更快。举个小例子，假设我们有一个包含大量行的表 `T`，我们要更新该表中每一行的一个列值。我们将用以下语句来设置这样一个表（在以下三种情况之前运行这四个设置步骤）：

```bash
$ sqlplus eoda/foo@PDB1
```
```sql
SQL> drop table t;
表已删除。
SQL> create table t as select * from all_objects;
表已创建。
SQL> exec dbms_stats.gather_table_stats( user, 'T' );
PL/SQL 过程已成功完成。
SQL> variable n number
```

那么，当我们去更新时，我们可以简单地在一个 `UPDATE` 语句中完成，像这样：

```sql
SQL> set serverout on
SQL> exec :n := dbms_utility.get_cpu_time;
PL/SQL 过程已成功完成。
SQL> update t set object_name = lower(object_name);
72614 行已更新。
SQL> set serverout on;
SQL> exec dbms_output.put_line((dbms_utility.get_cpu_time-:n)|| ' cpu hsecs...' );
117 cpu hsecs...
```

许多人，无论出于什么原因，都觉得必须像这样做——慢速/逐行处理——以便每 N 条记录提交一次：

```sql
SQL> exec :n := dbms_utility.get_cpu_time;
PL/SQL 过程已成功完成。
SQL> begin
for x in ( select rowid rid, object_name, rownum r
from t )
loop
update t
set object_name = lower(x.object_name)
where rowid = x.rid;
if ( mod(x.r,100) = 0 ) then
commit;
end if;
end loop;
commit;
end;
/
PL/SQL 过程已成功完成。
SQL> exec dbms_output.put_line((dbms_utility.get_cpu_time-:n)||' cpu hsecs...' );
323 cpu hsecs...
```

在这个简单的例子中，为了频繁提交而在循环中处理要慢*很多*倍。如果你能用*单条* SQL 语句完成，就这样做，因为它几乎肯定更快。即使我们“优化”过程化代码，对更新使用批量处理（如下所示），它确实快得多，但仍然比它本可以达到的速度慢得多：

```sql
SQL> exec :n := dbms_utility.get_cpu_time;
PL/SQL 过程已成功完成。
SQL> declare
type ridArray is table of rowid;
type vcArray is table of t.object_name%type;
l_rids  ridArray;
l_names vcArray;
cursor c is select rowid, object_name from t;
begin
open c;
loop
fetch c bulk collect into l_rids, l_names LIMIT 100;
forall i in 1 .. l_rids.count
update t
set object_name = lower(l_names(i))
where rowid = l_rids(i);
commit;
exit when c%notfound;
end loop;
close c;
end;
/
PL/SQL 过程已成功完成。
SQL> exec dbms_output.put_line((dbms_utility.get_cpu_time-:n)||' cpu hsecs...' );
67 cpu hsecs...
PL/SQL 过程已成功完成。
```

不仅如此，你还应该注意到代码变得越来越复杂。从单条 `UPDATE` 语句的极致简单，到过程化代码，再到更复杂的过程化代码——我们正朝着错误的方向前进！此外（是的，还有更多可抱怨的），前面的过程化代码还没完。它没有处理“当我们失败时会发生什么”（不是*如果我们*，而是*当我们*）。如果这段代码执行到一半然后系统失败了怎么办？你如何带着一个已提交的点来重启这个过程化代码？你将不得不添加更多的代码，以便知道从哪里继续处理。而使用单条 `UPDATE` 语句，我们只需重新发出 `UPDATE`。我们知道它要么完全成功，要么完全失败；不会有需要担心的部分工作。我们在“可重启的进程需要复杂逻辑”一节中会更详细地讨论这一点。



