# 可延迟约束与级联更新

在 Oracle 中，我们还能够延迟约束检查，这对于各种操作来说可能相当有利。首先想到的就是需要将主键的 `UPDATE` 操作级联到子键的要求。许多人声称你永远不应该需要这样做——因为主键是不可变的（我就是其中之一），但许多人仍然坚持希望拥有级联 `UPDATE` 功能。可延迟约束使这成为可能。

### 注意

执行更新级联来修改主键被认为是**极其糟糕的做法**。它违反了主键的设计意图。如果你必须这样做一次来纠正错误信息，那是一回事，但如果你发现它作为应用程序的一部分在持续进行，你将需要回过头去重新思考那个过程——你选择了错误的属性作为键！

在 Oracle 的早期版本中，是有可能进行 `CASCADE UPDATE` 的，但这样做涉及大量工作并且存在某些限制。有了可延迟约束，这就变得几乎轻而易举了。代码可能如下所示：

```
$ sqlplus eoda/foo@PDB1
SQL> create table parent ( pk  int primary key );
Table created.
SQL> create table child
( fk  constraint child_fk_parent
references parent(pk)
deferrable
initially immediate
);
Table created.
SQL> insert into parent values ( 1 );
1 row created.
SQL> insert into child values ( 1 );
1 row created.
```

我们有一个父表 `PARENT` 和一个子表 `CHILD`。表 `CHILD` 引用了表 `PARENT`，用于强制实施该规则的约束名为 `CHILD_FK_PARENT`（子外键到父键）。这个约束被创建为 `DEFERRABLE`（可延迟），但它被设置为 `INITIALLY IMMEDIATE`（初始为立即）。这意味着我们可以将该约束延迟到 `COMMIT`（提交）或其他某个时间。然而，默认情况下，它将在语句级别进行验证。这是可延迟约束最常见的用法。大多数现有应用程序不会在 `COMMIT` 语句上检查约束违反，最好不要让它们感到意外。按照定义，表 `CHILD` 的行为方式与以往一样，但它给了我们显式改变其行为的能力。现在让我们尝试对表进行一些 DML 操作，看看会发生什么：

```
SQL> update parent set pk = 2;
update parent set pk = 2
*
ERROR at line 1:
ORA-02292: integrity constraint (EODA.CHILD_FK_PARENT) violated - child record
found
```

由于约束处于 `IMMEDIATE`（立即）模式，此 `UPDATE` 失败。我们将更改模式并重试：

```
SQL> set constraint child_fk_parent deferred;
Constraint set.
SQL> update parent set pk = 2;
1 row updated.
```

现在它成功了。为了说明目的，我将展示如何在提交前显式检查延迟约束，以查看我们所做的修改是否符合业务规则（换句话说，检查约束当前是否没有被违反）。在提交或将控制权交给程序的其他部分（可能不期望延迟约束）之前，这样做是个好主意：

```
SQL> set constraint child_fk_parent immediate;
set constraint child_fk_parent immediate
*
ERROR at line 1:
ORA-02291: integrity constraint (EODA.CHILD_FK_PARENT) violated - parent key
not found
```

它如预期那样立即失败并返回错误，因为我们知道约束已经被违反。对 `PARENT` 的 `UPDATE` 操作并未被回滚（那会违反语句级原子性）；它仍然未决。另外请注意，由于 `SET CONSTRAINT` 命令失败，我们的事务仍在使用延迟的 `CHILD_FK_PARENT` 约束。现在让我们继续，将 `UPDATE` 级联到 `CHILD`：

```
SQL> update child set fk = 2;
1 row updated.
SQL> set constraint child_fk_parent immediate;
Constraint set.
SQL> commit;
Commit complete.
```

这就是它的工作方式。请注意，要延迟约束，你必须以这种方式创建它——你必须删除并重新创建约束，才能将其从不可延迟更改为可延迟。这可能会让你认为，应该将所有约束都创建为“可延迟初始立即”，以防你将来某个时候想延迟它们。一般来说，事实并非如此。你只应在确实需要延迟约束时才允许这样做。通过创建延迟约束，你会在物理实现（数据结构）中引入可能不明显的变化。例如，如果你创建了一个可延迟的 `UNIQUE`（唯一）或 `PRIMARY KEY`（主键）约束，Oracle 为支持该约束强制实施而创建的索引将是一个非唯一索引。通常，你期望一个唯一索引来强制实施唯一约束，但由于你指定了该约束可以被暂时忽略，因此它不能使用该唯一索引。还会观察到其他细微的变化，例如，对于 `NOT NULL`（非空）约束。如果你允许你的 `NOT NULL` 约束可延迟，优化器将开始将该列视为支持 `NULL`（空值）——因为事实上在你的事务期间它确实支持 `NULL`。例如，假设你有一个包含以下列和数据的表：

```
SQL> create table t
( x int constraint x_not_null not null deferrable,
y int constraint y_not_null not null,
z varchar2(30)
);
Table created.
SQL> insert into t(x,y,z)
select rownum, rownum, rpad('x',30,'x')
from all_users;
61 rows created.
SQL> exec dbms_stats.gather_table_stats( user, 'T' );
PL/SQL procedure successfully completed.
```

在这个例子中，列 `X` 被创建为：当你 `COMMIT` 时，`X` 将不为空。然而，在你的事务期间，`X` 被允许为空，因为约束是可延迟的。另一方面，列 `Y` 始终是 `NOT NULL`。假设你为列 `Y` 创建索引：

```
SQL> create index t_idx on t(y);
Index created.
```

然后你运行一个可以利用 `Y` 上这个索引的查询——但仅当 `Y` 是 `NOT NULL` 时，如下列查询所示：

```
SQL> explain plan for select count(*) from t;
Explained.
SQL> select * from table(dbms_xplan.display(null,null,'BASIC'));

| Id  | Operation        | Name  |

|   0 | SELECT STATEMENT |       |
|   1 | SORT AGGREGATE   |       |
|   2 | INDEX FULL SCAN  | T_IDX |
```

你会很高兴看到优化器选择使用 `Y` 上的小索引来计算行数，而不是全扫描整个表 `T`。然而，假设你删除了该索引并改为索引列 `X`：

```
SQL> drop index t_idx;
Index dropped.
SQL> create index t_idx on t(x);
Index created.
```

然后你再次运行查询来计算行数；你会发现数据库没有使用你的索引，事实上也无法使用：

```
SQL> explain plan for select count(*) from t;
Explained.
SQL> select * from table(dbms_xplan.display(null,null,'BASIC'));

| Id  | Operation          | Name |

|   0 | SELECT STATEMENT   |      |
|   1 | SORT AGGREGATE     |      |
|   2 | TABLE ACCESS FULL  | T    |
```

它全扫描了表。为了计算行数，它不得不全扫描表。这是由于在 Oracle B*树索引中，完全为 null 的索引键条目不会被创建。也就是说，对于表 `T` 中索引列全为 null 的任何行，索引将不包含其条目。由于 `X` 被允许暂时为空，优化器必须假设 `X` 可能为空，因此不会出现在 `X` 的索引中。因此，从索引返回的计数可能与对表的计数不同（错误）。

我们可以看到，如果 `X` 上有一个不可延迟的约束，这个限制就被解除了；也就是说，如果 `NOT NULL` 约束不可延迟，列 `X` 实际上和列 `Y` 一样好：



