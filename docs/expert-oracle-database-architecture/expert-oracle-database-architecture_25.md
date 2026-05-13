# Oracle 事务与锁机制详解

SQL> truncate table t;
Table truncated.
SQL> insert into t (x,y)
select rownum, rpad('*',147,'*')
from dual
connect by level  select length(y),
dbms_rowid.rowid_block_number(rowid) blk,
count(*), min(x), max(x)
from t
group by length(y), dbms_rowid.rowid_block_number(rowid);
LENGTH(Y)         BLK   COUNT(*)     MIN(X)     MAX(X)
---------- ---------- ---------- ---------- ----------
147      23470         46          1         46
SQL> exec do_update(1);
we finished - no problems
PL/SQL procedure successfully completed.

这次我们成功完成了——这就是一个字节所带来的差异！在这个案例中，块上多出的 46 字节可用空间（因为 46 个字符串每个都小了 1 字节）允许我们在该块上多执行至少九个并发事务。

这个例子演示了当多个事务尝试同时访问同一个块时会发生的情况——如果并发事务数量极高，就可能发生在事务表上的等待。如果 `INITRANS` 设置得太低，并且块上没有足够的空间来动态扩展事务，就可能发生阻塞。在大多数情况下，默认值 2 是足够的，因为空闲事务表插槽会动态增长（前提是空间允许），但在某些环境中，你可能需要增加这个设置（以预留更多插槽空间）来提高并发性并减少等待。

你可能需要增加此设置的一个例子是，对于一张表，或者更常见地，对于一个索引（因为索引块通常能比表块容纳更多的行），如果它被频繁修改且平均每块行数很多。你可能需要增加 `PCTFREE`（在第 10 章讨论）或 `INITRANS`，以便预先在块上为预期的并发事务数量预留足够的空间。如果你预计这些块一开始就会接近填满，意味着没有空间用于块上事务结构的动态扩展，那么这一点尤其重要。

最后关于 `INITRANS` 的一点说明。我有几次提到该属性的默认值是 2。然而，如果你在创建表后检查数据字典，会发现 `INITRANS` 显示的值是 1：

```
SQL> create table t ( x int );
SQL> select ini_trans from user_tables where table_name = 'T';
INI_TRANS
----------
```

那么，默认的事务插槽数量到底是 1 还是 2？尽管数据字典显示的值是 1，我们可以证明它实际上是 2。考虑这个实验。首先，为表 `T` 生成一个事务，插入一条记录：

```
SQL>  insert into t values ( 1 );
```

现在验证表 `T` 消耗了一个块：

```
SQL>  select dbms_rowid.ROWID_BLOCK_NUMBER(rowid)  from t;
DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
------------------------------------
```

接下来，将表 `T` 所使用块的块号和数据文件号放入变量 `B` 和 `F` 中：

```
SQL>  column b new_val B
SQL>  column f new_val F
SQL>  select dbms_rowid.ROWID_BLOCK_NUMBER(rowid) B,
dbms_rowid.ROWID_TO_ABSOLUTE_FNO( rowid, user, 'T' ) F
from t;
```

现在转储表 `T` 正在使用的块：

```
SQL>  alter system dump datafile &F block &B;
```

接着，将包含该块转储信息的跟踪文件的位置和名称放入名为 `TRACE` 的变量中：

```
SQL> column trace new_val TRACE
SQL> select c.value || '/' || d.instance_name || '_ora_' || a.spid || '.trc' trace
from v$process a, v$session b, v$diag_info c, v$instance d
where a.addr = b.paddr
and b.audsid = userenv('sessionid')
and c.name = 'Diag Trace';
```

你应该会看到类似以下的输出：

```
TRACE
-------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/cdb/CDB/trace/CDB_ora_18604.trc
```

现在终止会话并编辑跟踪文件：

```
SQL>  disconnect
SQL>  edit &TRACE
```

在跟踪文件中搜索 `Itl` 的值，我们看到有两个事务插槽已被初始化（尽管只为这个表发起过一个事务）：

```
Itl      Xid                  Uba                 Flag  Lck        Scn/Fsc
0x01   0x0013.00e.000024be  0x00c000bf.039e.2d  --U-    1  fsc 0x0000.01cfa56a
0x02   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
```

数据字典中报告的 `INITRANS` 值为 1 很可能是一个遗留值，在更新版本的 Oracle 中，它确实应该显示为 2。

## TM (DML 排队) 锁

TM 锁用于确保在你修改表的内容时，表的结构不被更改。例如，如果你更新了一张表，你将在该表上获得一个 TM 锁。这将阻止另一个用户对该表执行 `DROP` 或 `ALTER` 命令。如果另一个用户在你持有 TM 锁期间尝试对该表执行 DDL，他们将收到以下错误消息：

```
drop table dept
*
ERROR at line 1:
ORA-00054: resource busy and acquire with NOWAIT specified
```

> 注意
>
> 你可以设置 `DDL_LOCK_TIMEOUT` 来让 DDL 语句等待。这通常通过 `ALTER SESSION` 命令实现。例如，你可以在执行 `DROP TABLE` 命令之前，先执行 `ALTER SESSION SET DDL_LOCK_TIMEOUT=60;`。这样，发出的 `DROP TABLE` 命令将等待 60 秒，然后才返回错误（当然，它也可能在这期间内成功执行）。

ORA-00054 错误消息起初令人困惑，因为根本没有直接的方法在 `DROP TABLE` 上指定 `NOWAIT` 或 `WAIT`。它只是一个通用消息，当你尝试执行一个会被阻塞、但该操作本身又不允许被阻塞的操作时就会收到。正如你之前看到的，这与你对锁定的行执行 `SELECT FOR UPDATE NOWAIT` 时得到的错误消息是相同的。

以下展示了这些锁在 `V$LOCK` 表中的显示方式：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t1 ( x int );
Table created.
SQL> create table t2 ( x int );
Table created.
SQL> insert into t1 values ( 1 );
1 row created.
SQL> insert into t2 values ( 1 );
1 row created.
SQL> select (select username
from v$session
where sid = v$lock.sid) username,
sid,
id1,
id2,
lmode,
request, block, v$lock.type
from v$lock
where sid = sys_context('userenv','sid');
USERNAME      SID        ID1        ID2      LMODE    REQUEST      BLOCK TY
-------- -------- ---------- ---------- ---------- ---------- ---------- --
EODA           22        133          0          4          0          0 AE
EODA           22     244271          0          3          0          0 TM
EODA           22     244270          0          3          0          0 TM
EODA           22    1966095        152          6          0          0 TX
SQL> select object_name, object_id  from user_objects where object_id in (244271,244270);
OBJECT_NAM  OBJECT_ID
---------- ----------
T2             244271
T1             244270
```

> 注意
>
> `AE` 锁是一个版本锁，在 Oracle 11*g* 及以上版本可用。它是基于版本的重定义功能的一部分（本书不涉及）。`ID1` 是 `SID` 当前所使用版本的对象 ID。这个版本锁以类似于 `TM` 锁保护它们所指向的表免受结构修改的方式，来保护被引用的版本免受修改（例如，删除版本）。

我们每个事务只能获得一个 `TX` 锁，但我们可以获得与我们修改的对象数量一样多的 `TM` 锁。这里有趣的是，TM 锁的 `ID1` 列是 DML 锁定对象的对象 ID，因此很容易找到持有锁的对象。


