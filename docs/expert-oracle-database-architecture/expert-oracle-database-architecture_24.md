# TX（事务）锁

`TX` 锁（事务锁）是在事务发起其首次更改时获取的。此时事务会自动启动（在 Oracle 中你不需要显式启动事务）。该锁会一直保持，直到事务执行 `COMMIT` 或 `ROLLBACK`。它被用作一种排队机制，以便其他会话可以等待事务完成。在一个事务中，你修改或通过 `SELECT FOR UPDATE` 选中的每一行，都会指向与该事务关联的 `TX` 锁。虽然这听起来开销很大，但其实不然。要理解原因，你需要从概念上理解锁的存储位置和管理方式。在 Oracle 中，锁被存储为数据的一个属性（有关 Oracle 块格式的概述，请参见第 10 章）。Oracle 没有一个传统的锁管理器来维护系统中每一行被锁定的长列表。许多其他数据库采用那种方式，因为对他们来说，锁是稀缺资源，其使用需要被监控。使用的锁越多，这些系统需要管理的就越多，因此如果使用了太多锁，在这些系统中就是一个问题。

在一个具有基于内存的传统锁管理器的数据库中，锁定一行的过程类似于以下步骤：

1.  找到你想要锁定的行的地址。
2.  在锁管理器处排队（这必须被序列化，因为它是一个公共的内存结构）。
3.  锁定列表。
4.  搜索列表，查看是否有人已经锁定了这行。
5.  在列表中创建一个新条目，以确立你已经锁定了该行的事实。
6.  解锁列表。

现在你已经锁定了该行，可以对其进行修改。之后，当你提交更改时，你必须继续执行以下过程：

1.  再次排队。
2.  锁定锁列表。
3.  搜索列表并释放你所有的锁。
4.  解锁列表。

如你所见，获取的锁越多，在修改数据前后花费在此操作上的时间就越多。Oracle 不是这样做的。Oracle 的流程如下：

1.  找到你想要锁定的行的地址。
2.  前往该行。
3.  就在那里，就在那时锁定该行——在行所在的位置，而不是在某个大列表里（如果该行已被锁定，则等待持有该锁的事务结束，除非你使用 `NOWAIT` 选项）。

就这样。由于锁是作为数据的属性存储的，Oracle 不需要传统的锁管理器。事务只需直接前往数据并锁定它（如果尚未锁定）。有趣的是，即使数据实际上未被锁定，当你到达时它也可能看起来是锁定的。当你在 Oracle 中锁定数据行时，该行会指向一个存储在包含该数据的块中的事务 ID 副本，当锁被释放时，该事务 ID 会被留下。这个事务 ID 对你的事务是唯一的，代表撤销段号、槽和序列号。你将它留在包含你的行的块上，以告知其他会话你拥有此数据（不是块上的所有数据，只是你正在修改的那一行）。当另一个会话到来时，它会看到该事务 ID，并利用它代表一个事务这一事实，可以快速查看持有锁的事务是否仍然活动。如果锁不活动，则允许会话访问数据。如果锁仍然活动，该会话将请求在锁释放时立即收到通知。因此，你就有了一个排队机制：请求锁的会话将排队等待该事务完成，然后它将获得数据。

下面是一个使用三个 `V$` 表的小例子，展示了这是如何发生的：

*   `V$TRANSACTION`，其中包含每个活动事务的条目。
*   `V$SESSION`，显示已登录的会话。
*   `V$LOCK`，其中包含所有正在持有的入队锁以及正在等待锁的会话的条目。你不会在这个视图中看到会话锁定的每一行对应的一行。如前所述，行级别的主锁列表并不存在。如果一个会话锁定了 `EMP` 表中的一行，则此视图中将有一行指示该事实。如果一个会话锁定了 `EMP` 表中的数百万行，此视图中仍然只有一行。此视图显示各个会话持有的入队锁。

首先，让我们获取 `EMP` 和 `DEPT` 表的副本。如果你的模式中已有它们，请使用以下定义替换它们：

```
$ sqlplus eoda/foo@PDB1
SQL> create table dept as select * from scott.dept;
Table created.
SQL> create table emp as select * from scott.emp;
Table created.
SQL> alter table dept
add constraint dept_pk
primary key(deptno);
Table altered.
SQL> alter table emp
add constraint emp_pk
primary key(empno);
Table altered.
SQL> alter table emp
add constraint emp_fk_dept
foreign key (deptno)
references dept(deptno);
Table altered.
SQL> create index emp_deptno_idx on emp(deptno);
Index created.
```

现在让我们启动一个事务：

```
SQL> update dept set dname = initcap(dname);
4 rows updated.
```

现在，让我们看看此时系统的状态。此示例假设是单用户系统；否则，你可能会在 `V$TRANSACTION` 中看到多行。即使在单用户系统中，如果在 `V$TRANSACTION` 中看到多行也不要惊讶，因为许多后台 Oracle 进程可能也在执行事务。

```
SQL> select username,
v$lock.sid,
trunc(id1/power(2,16)) rbs,
bitand(id1,to_number('ffff','xxxx'))+0 slot,
id2 seq,
lmode,
request
from v$lock, v$session
where v$lock.type = 'TX'
and v$lock.sid = v$session.sid
and v$session.username = USER;
USERNAME             SID       RBS      SLOT       SEQ     LMODE    REQUEST
--------------- -------- --------- --------- --------- --------- ----------
EODA                  22         2        27     21201         6          0
SQL> select XIDUSN, XIDSLOT, XIDSQN from v$transaction;
XIDUSN    XIDSLOT     XIDSQN
---------- ---------- ----------
2         27      21201
```

这里需要注意的要点如下：

*   `V$LOCK` 表中的 `LMODE` 是 6，`REQUEST` 是 0。如果你参考 *Oracle Database Reference* 手册中对 `V$LOCK` 表的定义，你会发现 `LMODE=6` 是一个排他锁。请求列中的值 `0` 表示你没有提出请求；你已经持有了锁。
*   此表中只有一行。这个 `V$LOCK` 表更像是一个排队表而不是锁表。许多人期望在 `V$LOCK` 中有四行，因为我们锁定了四行。但请记住，Oracle 没有在任何地方存储所有被锁定行的主列表。要确定一行是否被锁定，我们必须前往该行。
*   我对 `ID1` 和 `ID2` 列进行了一些操作。Oracle 需要保存三个 16 位数字，但只有两列来做到这一点。因此，第一列 `ID1` 保存了其中两个数字。通过使用 `trunc(id1/power(2,16)) rbs` 除以 2¹⁶，并使用 `bitand(id1,to_number('ffff','xxxx'))+0 slot` 屏蔽高位，我能够取回隐藏在那个数字中的两个数字。
*   `RBS`、`SLOT` 和 `SEQ` 值与 `V$TRANSACTION` 信息匹配。这就是我的事务 ID。

现在，我们将使用相同的用户名启动另一个会话，更新 `EMP` 中的一些行，然后尝试更新 `DEPT`：

```
SQL> update emp set ename = upper(ename);
14 rows updated.
SQL> update dept set deptno = deptno-10;
```

我们现在在这个会话中被阻塞了。如果我们再次运行 `V$` 查询，我们会看到以下内容：



## 锁查询示例

```
SQL> select username,
v$lock.sid,
trunc(id1/power(2,16)) rbs,
bitand(id1,to_number('ffff','xxxx'))+0 slot,
id2 seq,
lmode,
request
from v$lock, v$session
where v$lock.type = 'TX'
and v$lock.sid = v$session.sid
and v$session.username = USER;
USERNAME            SID      RBS     SLOT        SEQ      LMODE    REQUEST
------------ ---------- -------- -------- ---------- ---------- ----------
EODA                 17        2       27      21201          0          6
EODA                 22        2       27      21201          6          0
EODA                 17        8       17      21403          6          0
SQL> select XIDUSN, XIDSLOT, XIDSQN from v$transaction;
XIDUSN    XIDSLOT     XIDSQN
---------- ---------- ----------
2         27      21201
8         17      21403
```

我们在这里看到，一个新事务开始了，其事务 ID 为 `(8,17,21403)`。我们的新会话 `SID=17` 这次在 `V$LOCK` 中有两行。一行表示它拥有的锁（`LMODE=6`）。另一行显示了一个值为 6 的 `REQUEST`。这是对一个排他锁的请求。需要注意的有趣之处是，这个请求行的 `RBS/SLOT/SEQ` 值是锁持有者的事务 ID。`SID=22` 的事务正在阻塞 `SID=17` 的事务。我们可以通过对 `V$LOCK` 进行自连接来更明确地看到这一点：

```
SQL> select
(select username from v$session where sid=a.sid) blocker,
a.sid,
' is blocking ',
(select username from v$session where sid=b.sid) blockee,
b.sid
from v$lock a, v$lock b
where a.block = 1
and b.request > 0
and a.id1 = b.id1
and a.id2 = b.id2;
BLOCKER                SID 'ISBLOCKING'  BLOCKEE                SID
--------------- ---------- ------------- --------------- ----------
EODA                    22  is blocking  EODA                    17
```

现在，如果我们提交原始事务 `SID=22`，并重新运行锁查询，我们会发现请求行消失了：

```
SQL> select username,
v$lock.sid,
trunc(id1/power(2,16)) rbs,
bitand(id1,to_number('ffff','xxxx'))+0 slot,
id2 seq,
lmode,
request
from v$lock, v$session
where v$lock.type = 'TX'
and v$lock.sid = v$session.sid
and v$session.username = USER;
USERNAME             SID      RBS       SLOT        SEQ     LMODE   REQUEST
------------- ---------- -------- ---------- ---------- --------- ---------
EODA                  17        8         17      21403         6         0
SQL> select XIDUSN, XIDSLOT, XIDSQN from v$transaction;
XIDUSN    XIDSLOT     XIDSQN
---------- ---------- ----------
8         17      21403
```

请求行在其他会话释放锁的瞬间就消失了。那个请求行就是排队机制。数据库能够在事务完成的瞬间唤醒被阻塞的会话。各种 GUI 工具可以提供更美观的显示，但紧急情况下，了解你需要查看哪些表是非常有用的。

## 块头事务表

然而，在我们说已经很好地理解了 Oracle 中行锁如何工作之前，我们必须看最后一个主题：锁和事务信息如何与数据本身一起管理。它是块头开销的一部分。在第 10 章中，我们将详细介绍块格式，但可以简单地说，在数据库块的顶部有一些前置开销空间，用于存储该块的事务表。这个事务表包含每个在该块中锁定了数据的真实事务的一个条目。这个结构的大小由对象的 `CREATE` 语句上的两个物理属性参数控制：

*   `INITRANS`：此结构的初始预分配大小。索引和表默认为 2。
*   `MAXTRANS`：此结构可以增长到的最大尺寸。在 Oracle 的近期版本中，`MAXTRANS` 总是设置为 255。

每个块在诞生时默认有两个事务槽。一个块能同时拥有的活动事务数量受 `MAXTRANS` 的值和块上空间可用性的限制。如果没有足够的空间来增长这个结构，你可能无法在一个块上实现 255 个并发事务。

我们可以通过创建一个表来人为地演示这是如何工作的，将许多行打包到一个块中，使得块从一开始就很满；在我们初始加载数据后，块上将只剩下很少的空间。这些行的存在将限制事务表可以增长的大小，因为缺乏空间。我当时使用了 8KB 的块大小（所以，如果你有 8KB 的块大小，你应该能够重现这个）。我们将从创建打包表开始。我尝试了不同的数据长度，最终得到了这个非常特殊的大小：

```
$ sqlplus eoda/foo@PDB1
SQL> set serverout on
SQL> create table t
( x int primary key,
y varchar2(4000)
);
Table created.
SQL> insert into t (x,y)
select rownum, rpad('*',148,'*')
from dual
connect by level < 47;
46 rows created.
SQL> commit;
Commit complete.
SQL> select length(y),
dbms_rowid.rowid_block_number(rowid) blk,
count(*), min(x), max(x)
from t
group by length(y), dbms_rowid.rowid_block_number(rowid);
LENGTH(Y)         BLK   COUNT(*)     MIN(X)     MAX(X)
---------- ---------- ---------- ---------- ----------
148      23470         46          1         46
```

所以，我们的表有 46 行，都在同一个块上。我选择了 148 个字符，因为如果再多一个字符，就需要两个块来保存这 46 条记录。现在，我们需要一种方法来观察当许多事务试图同时在该单个块上锁定数据时会发生什么。为此，我们将再次使用 `AUTONOMOUS_TRANSACTION`，这样我们就可以使用单个会话，而不必运行许多并发的 SQL*Plus 会话。我们的存储过程将通过主键锁定表中的一行，从主键值 1 开始（插入的第一条记录）。如果我们的过程无需等待（不会被阻塞）就获得了该行的锁，它将简单地将主键值增加 1，并使用递归重复整个过程。因此，第二次调用将尝试锁定记录 2，第三次调用锁定记录 3，依此类推。如果该过程被强制等待，它将引发一个 ORA-54 资源忙错误，我们将打印出“locked out trying to select row <主键值>”。这将表明我们在锁定所有行之前就已经用完了该块上的事务槽。另一方面，如果我们没有找到要锁定的行，那就意味着我们已经锁定了该块上的每一行，我们将打印出成功（意味着块头中的事务表能够增长以容纳所有事务）。以下是该存储过程：

```
SQL> create or replace procedure do_update( p_n in number )
as
pragma autonomous_transaction;
l_rec t%rowtype;
resource_busy exception;
pragma exception_init( resource_busy, -54 );
begin
select *
into l_rec
from t
where x = p_n
for update NOWAIT;
do_update( p_n+1 );
commit;
exception
when resource_busy
then
dbms_output.put_line( 'locked out trying to select row ' || p_n );
commit;
when no_data_found
then
dbms_output.put_line( 'we finished - no problems' );
commit;
end;
/
Procedure created.
```

关键在第 14 行，我们在那里递归地调用自身，用新的主键值一遍又一遍地尝试锁定。如果你在用 148 个字符的字符串填充表后运行该过程，你应该观察到：

```
SQL> exec do_update(1);
locked out trying to select row 38
PL/SQL procedure successfully completed.
```

这个输出表明我们能够锁定 37 行，但在尝试锁定第 38 行时用完了事务槽。对于这个给定的块，最多可以有 37 个事务并发访问它。如果我们用稍小一点的字符串重做这个例子，我们会看到它顺利完成：

```
SQL> exec do_update(1);
we finished - no problems
PL/SQL procedure successfully completed.
```

在这个例子中，使用 146 个字符的字符串，我们能够在块上存储 47 行，并且事务表可以增长到足够大以支持 47 个并发事务。



