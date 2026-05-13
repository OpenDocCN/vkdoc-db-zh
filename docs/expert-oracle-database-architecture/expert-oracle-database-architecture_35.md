# 快照过旧错误

现在让我们看看开发者倾向于在过程循环中提交更新的第二个原因，这源于他们（被误导的）试图节俭地使用“有限资源”（回滚段）。这是一个配置问题；你*需要*确保你有足够的回滚空间来正确设定你的事务大小。在循环中提交，除了通常更慢之外，也是导致令人畏惧的 `ORA-01555` 错误的最常见原因。让我们更详细地看看这个问题。

正如你在阅读前一章后将理解的，Oracle 的多版本模型使用回滚段数据来重构块在语句或事务开始时（取决于隔离模式）的样子。如果必要的回滚信息不再存在，你将收到 `ORA-01555: snapshot too old` 错误消息，并且你的查询将无法完成。所以，如果你正在修改你正在读取的表（如前面的例子），你正在生成你的查询所需的回滚信息。你的 `UPDATE` 生成了回滚信息，你的查询很可能会利用这些信息来获取它需要更新的数据的读一致视图。如果你提交了，你就允许系统重用你刚刚填满的回滚段空间。如果它确实重用了回滚空间，擦除了你的查询随后需要的旧回滚数据，那你就麻烦大了。你的 `SELECT` 会失败，而你的 `UPDATE` 会中途停止。你有一个部分完成的逻辑事务，并且很可能没有好的方法来重启它（稍后会详细讨论）。

让我们通过一个小演示来看看这个概念是如何发生的。在一个小型测试数据库中，我们建立了一个表：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t as select * from all_objects;
Table created.
SQL> create index t_idx on t(object_name);
Index created.
SQL> exec dbms_stats.gather_table_stats( user, 'T', cascade=>true );
PL/SQL procedure successfully completed.
```

然后我创建了一个非常小的回滚表空间，并更改系统以使用它。注意，通过设置 `AUTOEXTEND` 为关闭，我将此系统中所有 `UNDO` 的大小限制为 10MB 或更小：

```
SQL> create undo tablespace undo_small
datafile '/tmp/undo_small.dbf'
size 10m reuse
autoextend off;
Tablespace created.
SQL> alter system set undo_tablespace = undo_small;
System altered.
```

现在，只使用这个小的回滚表空间，我运行了这个代码块来执行 `UPDATE`：

```
SQL> begin
for x in ( select /*+ INDEX(t t_idx) */ rowid rid, object_name, rownum r
from t
where object_name > ' ' )
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
begin
*
ERROR at line 1:
ORA-01555: snapshot too old: rollback segment number  with name "" too small
ORA-06512: at line 2
```

我得到了这个错误。我应该指出，我在查询中添加了一个索引提示和一个 `WHERE` 子句，以确保我是在随机读取表（它们共同导致基于成本的优化器按索引键“排序”读取表）。当我们通过索引处理表时，我们倾向于读取一个块来获取一行，然后我们想要的下一行会在另一个块上。最终，我们将处理块 1 上的所有行，只是不是同时处理。块 1 可能包含，比如说，所有 `OBJECT_NAME` 以字母 A、M、N、Q 和 Z 开头的行的数据。因此，我们会多次访问这个块，因为我们是按 `OBJECT_NAME` 排序读取数据，并且可能有许多 `OBJECT_NAME` 以 A 和 M 之间的字母开头。由于我们频繁提交并重用回滚空间，我们最终会重访一个我们已经无法回滚到查询开始时状态的块，此时我们就得到了错误。

这是一个非常人为的例子，只是为了可靠地展示它是如何发生的。我的 `UPDATE` 语句正在生成回滚。我有一个非常小的回滚表空间（10MB）可以操作。我的回滚段多次回绕，因为它们是循环使用的。每次我提交，我都允许 Oracle 覆盖我生成的回滚数据。最终，我需要一些我生成的数据片段，但它已不复存在，于是我收到了 `ORA-01555` 错误。

你可能会正确地指出，在这个例子中，如果我没有在第 10 行提交，我会收到以下错误：

```
begin
*
ERROR at line 1:
ORA-30036: unable to extend segment by 8 in undo tablespace 'UNDO_SMALL'
ORA-06512: at line 6
```

这两个错误的主要区别如下：

*   `ORA-01555` 例子*将我的更新置于一个完全未知的状态*。部分工作已完成；部分未完成。
*   鉴于我在游标 `FOR` 循环中提交了，*我绝对无法避免 `ORA-01555` 错误*。
*   `ORA-30036` *错误可以通过在系统中分配适当的资源来避免*。这个错误可以通过正确的大小调整来避免；第一个错误则不行。此外，即使我没有避免这个错误，至少更新会被回滚，数据库保持在一个已知的、一致的状态——而不是半途停滞在某个大更新中。

这里的底线是，你不能通过频繁提交来“节省”回滚空间——你需要那些回滚数据。我收到 `ORA-01555` 错误时是在一个单用户系统中。只需要一个会话就能导致那个错误，并且很多时候即使在真实环境中，也是一个会话导致了其自身的 `ORA-01555` 错误。开发人员和 DBA 需要共同努力，为需要完成的工作充分设定这些段的大小。这里不能有任何短缺。你必须通过分析你的系统来发现你最大的事务是什么，并相应地为它们设定大小。动态性能视图 `V$UNDOSTAT` 对于监控你生成的回滚量和最长运行查询的持续时间非常有用。许多人认为像临时表空间、回滚表空间和重做日志这样的东西是开销——要分配尽可能少的存储。这让人想起计算机行业在 2000 年 1 月 1 日遇到的一个问题，那完全是由于试图在日期字段中节省 2 个字节造成的。这些数据库组件不是开销，而是系统的关键组成部分。它们必须被适当地设定大小（不能太大也不能太小）。

说到 `UNDO` 段太小，在运行这些测试后，务必将你的回滚表空间设置回常规的那个，例如：

```
SQL> alter system set undo_tablespace=undotbs2;
SQL> drop tablespace undo_small including contents and datafiles;
```

如果你不把回滚表空间改回一个正常大小的，本书的其余部分你都会遇到 `ORA-30036` 错误！



#### 可重启的进程需要复杂的逻辑

“在逻辑事务结束前提交”这种方法最严重的问题在于，如果 `UPDATE` 操作中途失败，它常常会使你的数据库处于一个未知状态。除非你预先为此做了规划，否则很难重启失败的进程并让其从中断处继续执行。例如，假设我们不是像前面的例子那样对列应用 `LOWER()` 函数，而是应用其他函数，比如这样：

```
last_ddl_time = last_ddl_time + 1;
```

如果我们中途停止了 `UPDATE` 循环，该如何重启它？我们不能简单地重新运行它，因为那样会导致某些日期的值加了 2，而另一些只加了 1。如果我们再次失败，就会出现一些加了 3，一些加了 2，其余的加了 1，如此往复。我们需要更复杂的逻辑——某种“分区”数据的方法。例如，我们可以先处理所有以 `A` 开头的 `OBJECT_NAME`，然后是 `B`，依此类推：

```
$ sqlplus eoda/foo@PDB1
SQL> create table to_do
as
select distinct substr( object_name, 1,1 ) first_char
from T
/
Table created.
SQL> set serverout on
SQL> begin
for x in ( select * from to_do )
loop
update t set last_ddl_time = last_ddl_time+1
where object_name like x.first_char || '%';
dbms_output.put_line( sql%rowcount || ' rows updated' );
delete from to_do where first_char = x.first_char;
commit;
end loop;
end;
/
238 rows updated
5730 rows updated
1428 rows updated
...
262 rows updated
1687 rows updated
PL/SQL procedure successfully completed.
```

现在，如果这个进程失败了，我们可以重启它，因为我们不会处理任何已经成功处理过的对象名。然而，这种方法的问题在于，除非我们有某个能均匀分区数据的属性，否则最终会导致行的分布非常不均。第二次 `UPDATE` 处理的工作量比其他所有次加起来还要多。此外，如果有其他会话正在访问这张表并修改数据，他们也可能更新 `OBJECT_NAME` 字段。假设在我们处理完所有以 `A` 开头的对象*之后*，另一个会话将某个名为 `Z` 的对象更新为 `A`。我们就会错过那条记录。而且，与 `UPDATE T SET LAST_DDL_TIME = LAST_DDL_TIME+1` 相比，这是一个非常低效的过程。我们很可能在使用索引来读取表中的每一行，或者对表进行了 *n* 次全表扫描，这两种情况都不可取。这种方法有很多缺点。

最好的方法是力求简单。如果能在 SQL 中完成，就在 SQL 中完成。SQL 无法完成的，再在 PL/SQL 中完成。使用尽可能少的代码来实现。分配足够的资源。始终考虑发生错误时的情况。我曾多次看到人们编写的更新循环在测试数据上运行良好，但在应用于真实数据时中途失败。然后他们就真的陷入困境了，因为他们不知道循环处理到哪里停止了。正确地估算 undo 空间大小，比重写一个可重启的程序要容易得多。如果你确实有需要更新的大型表，你应该使用分区，这样你可以单独更新每个分区。你甚至可以使用并行 DML 来执行更新，或者在 Oracle 11*g* Release 2 及以上版本中，使用 `DBMS_PARALLEL_EXECUTE` 包。

### 使用自动提交

关于不良事务习惯，我最后要讨论的是使用流行的编程 API ODBC 和 JDBC 时产生的问题。这些 API 默认是“自动提交”的。考虑以下语句，它们将 1000 美元从支票账户转到储蓄账户：

```
update accounts set balance = balance - 1000 where account_id = 123;
update accounts set balance = balance + 1000 where account_id = 456;
```

如果你的程序在使用 ODBC 或 JDBC 提交这些语句时，它们会在*每条* `UPDATE` 语句之后（静默地）注入一个提交。想想看，如果系统在第一个 `UPDATE` 之后、第二个 `UPDATE` 之前发生故障，会有什么影响。你刚刚损失了 1000 美元！

我大概能理解为什么 ODBC 这样做。SQL Server 的开发者设计了 ODBC，而这个数据库由于其并发模型（写阻塞读，读阻塞写，并且锁是稀缺资源）要求你使用非常短的事务。我无法理解的是，这怎么会延续到 JDBC 中，一个本应支持“企业级”应用的 API。我认为，在 JDBC 中打开连接后的下一行代码，应该总是这样的：

```
Connection conn = DriverManager.getConnection
("jdbc:oracle:oci:@database","scott","tiger");
conn.setAutoCommit (false);
```

这样就把事务的控制权交还给你——开发者，而这正是它应有的归属。然后你就可以安全地编写账户转账事务，并在两条语句都成功执行后再提交。在这种情况下，不了解你的 API 可能是致命的。我见过不止一个开发者没有意识到这个自动提交“特性”，当错误发生时，他们的应用程序陷入了大麻烦。


