# 为什么我的索引没有被使用？

这有很多种可能的原因。在本节中，我们将看看一些最常见的原因。

## 情况 1

我们正在使用一个 B*树索引，而我们的谓词没有使用索引的前导边缘。在这种情况下，我们可能有一个表 `T`，在 `T(x,y)` 上有一个索引。我们查询 `SELECT * FROM T WHERE Y = 5`。优化器倾向于不使用索引，因为我们的谓词没有涉及列 `X`——在这种情况下，它可能不得不检查每一个索引条目（我们很快会讨论索引跳跃扫描，那里情况不同）。它通常会选择对 `T` 进行全表扫描。

这并不妨碍索引被使用。如果查询是 `SELECT X,Y FROM T WHERE Y = 5`，优化器会注意到它不必访问表来获取 `X` 或 `Y`（它们在索引中），并且很可能选择对索引本身进行快速全扫描，因为索引通常比底层表小得多。还要注意，这种访问路径仅在 CBO（基于成本的优化器）下可用。

在 CBO 下，`T(x,y)` 上的索引可以被使用的另一种情况是在索引跳跃扫描期间。跳跃扫描工作良好——但前提是——索引的前导边缘（上一个例子中的 `X`）具有非常少的唯一值，并且优化器理解这一点。例如，考虑一个在 `(GENDER, EMPNO)` 上的索引，其中 `GENDER` 有 `M` 和 `F` 两个值，而 `EMPNO` 是唯一的。像下面这样的查询：

```
SQL> select * from t where empno = 5;
```

可能会考虑使用 `T` 上的那个索引来以 `skip scan` 方法满足查询，这意味着查询在概念上将像这样被处理：

```
SQL> select * from t where GENDER='M' and empno = 5
UNION ALL
select * from t where GENDER='F' and empno = 5;
```

它将在整个索引中跳跃，假装它是两个索引：一个是 `M`，一个是 `F`。我们可以在查询计划中轻松地看到这一点。我们将建立一个包含双值列的表并为其建立索引：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t as
select decode(mod(rownum,2), 0, 'M', 'F' ) gender, all_objects.*
from all_objects;
Table created.
SQL> create index t_idx on t(gender,object_id);
Index created.
SQL> exec dbms_stats.gather_table_stats( user, 'T' );
PL/SQL procedure successfully completed.
```

现在，当我们查询这个时，我们应该看到：

```
SQL> set autotrace traceonly explain
SQL> select * from t t1 where object_id = 42;
Execution Plan
----------------------------------------------------------
Plan hash value: 2985246541

--------------------------------------------------------------------------------------
| Id  | Operation                           | Name  |  Rows | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                    |       |     1 |    91 |     4   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| T     |     1 |    91 |     4   (0)| 00:00:01 |
|*  2 |   INDEX SKIP SCAN                   | T_IDX |     1 |       |     3   (0)| 00:00:01 |
--------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("OBJECT_ID"=42)
       filter("OBJECT_ID"=42)
```

`INDEX SKIP SCAN` 步骤告诉我们 Oracle 将在整个索引中跳跃，寻找 `GENDER` 值变化的点，并从那里向下读取树，在每个被考虑的虚拟索引中寻找 `OBJECT_ID=42`。

## 情况 2

我们正在使用一个 `SELECT COUNT(*) FROM T` 查询（或类似的东西），并且我们在表 `T` 上有一个 B*树索引。然而，优化器正在对表进行全表扫描，而不是对（小得多的）索引条目进行计数。在这种情况下，索引可能位于一组可以包含空值的列上。由于完全为空的索引条目永远不会被创建，索引中的行数将不是表中的行数。在这里，优化器正在做正确的事情——如果它使用索引来计数行，将会得到错误的答案。

## 情况 3

对于一个被索引的列，我们使用以下方式查询，并发现 `INDEXED_COLUMN` 上的索引未被使用：

```
select * from t where f(indexed_column) = value
```

这是由于在列上使用了函数。我们索引的是 `INDEXED_COLUMN` 的值，而不是 `F(INDEXED_COLUMN)` 的值。使用索引的能力在这里受到了限制。如果我们选择这样做，可以对函数进行索引。


#### 案例 4

我们为一个字符列创建了索引。该列只包含数字数据。我们使用以下语法进行查询：

```
SQL> select * from t where indexed_column = 5
```

请注意，查询中的数字 5 是常量*数字* 5（而不是字符串）。`INDEXED_COLUMN`上的索引未被使用。这是因为前面的查询等同于以下查询：

```
SQL> select * from t where to_number(indexed_column) = 5
```

我们隐式地对列应用了函数，正如案例 3 中指出的，这将妨碍索引的使用。通过一个简单的例子可以很容易地看到这一点。在这个例子中，我们将使用内置包`DBMS_XPLAN`：

```
$ sqlplus eoda/foo@PDB1
SQL > create table t ( x char(1) constraint t_pk primary key, y date );
Table created.
SQL> insert into t values ( '5', sysdate );
1 row created.
SQL> explain plan for select * from t where x = 5;
Explained.
SQL> select * from table(dbms_xplan.display);
PLAN_TABLE_OUTPUT

Plan hash value: 1601196873

| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT  |      |    1  |    12 |    3    (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| T    |    1  |    12 |    3    (0)| 00:00:01 |

Predicate Information (identified by operation id):

1 - filter(TO_NUMBER("X")=5)
```

如你所见，它进行了全表扫描。即使我们对以下查询使用提示，它使用了索引，但并非如我们可能预期的那样进行`UNIQUE SCAN`——它是在`FULL SCANNING`这个索引：

```
SQL> explain plan for select /*+ INDEX(t t_pk) */ * from t  where x = 5;
Explained.
SQL> select * from table(dbms_xplan.display);
PLAN_TABLE_OUTPUT

Plan hash value: 180604526

| Id  | Operation                           | Name |  Rows | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT                    |      |     1 |    12 |    3    (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| T    |     1 |    12 |    3    (0)| 00:00:01 |
|*  2 |   INDEX FULL SCAN                   | T_PK |     1 |       |    2    (0)| 00:00:01 |

Predicate Information (identified by operation id):

2 - filter(TO_NUMBER("X")=5)
```

原因在于输出的最后一行：`filter(TO_NUMBER("X")=5)`。有一个隐式的函数被应用到了数据库列上。存储在`X`中的字符串在比较值 5 之前必须被转换为数字。我们不能将 5 转换为字符串，因为我们的 NLS 设置控制着 5 在字符串中可能的样子（这是不确定的），因此我们将字符串转换为数字，这妨碍了使用索引来快速找到这一行。如果我们简单地比较字符串与字符串：

```
SQL> explain plan for select * from t where x = '5';
Explained.
SQL> select * from table(dbms_xplan.display);
PLAN_TABLE_OUTPUT

Plan hash value: 1303508680

| Id  | Operation                   | Name |  Rows | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT            |      |     1 |    12 |    2    (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| T    |     1 |    12 |    2    (0)| 00:00:01 |
|*  2 |   INDEX UNIQUE SCAN         | T_PK |     1 |       |    1    (0)| 00:00:01 |

Predicate Information (identified by operation id):

2 - access("X"='5')
```

我们得到了预期的`INDEX UNIQUE SCAN`，并且可以看到函数没有被应用。无论如何，你应该*总是*避免隐式转换。始终进行类型一致的比较。另一个经常出现这种情况的例子是日期。我们尝试查询：

```
-- find all records for today
select * from t where trunc(date_col) = trunc(sysdate);
```

然后发现`DATE_COL`上的索引不会被使用。我们可以选择索引`TRUNC(DATE_COL)`，或者，也许更简单地，使用范围比较运算符进行查询。以下演示了在日期上使用大于和小于运算符。一旦我们意识到条件：

```
TRUNC(DATE_COL) = TRUNC(SYSDATE)
```

等同于条件：

```
SQL> select *
from t
where date_col >=trunc(sysdate)
and date_col < trunc(sysdate+1)
```

这会将所有函数移动到等式的右侧，允许我们使用`DATE_COL`上的索引（并且提供与`WHERE TRUNC(DATE_COL) = TRUNC(SYSDATE)`相同的结果）。

*如果可能，当函数出现在谓词中的数据库列上时，你应该总是移除它们。* 这样做不仅可以考虑使用更多的索引，还可以减少数据库需要处理的数据量。在前面的例子中，当我们使用：

```
where date_col >=trunc(sysdate)
and date_col < trunc(sysdate+1)
```

`TRUNC`的值在查询时被计算一次，然后可以使用索引来查找符合条件的值。而当我们使用`TRUNC(DATE_COL) = TRUNC(SYSDATE)`时，`TRUNC(DATE_COL)`必须对整个表中的每一行评估一次*（没有使用索引）*。

#### 案例 5

使用索引实际上会更慢。我经常看到这种情况——人们理所当然地认为索引总会让查询变得更快。于是，他们建立一个小表，分析它，然后发现优化器没有使用索引。在这种情况下，优化器的做法完全正确。Oracle（在 CBO 下）只会在有意义时才使用索引。考虑以下例子：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t(x int);
Table created.
SQL> insert into t select rownum from dual connect by level  create index ti on t(x);
Index created.
SQL> exec dbms_stats.gather_table_stats(user,'T');
PL/SQL procedure successfully completed.
```

如果我们运行一个需要获取表中相对较小百分比的查询，如下所示：

```
SQL> set autotrace on explain
SQL> select count(*) from t where x < 50;
COUNT(*)

Execution Plan
...

| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT  |      |     1 |     5 |    3    (0)| 00:00:01 |
|   1 |  SORT AGGREGATE   |      |     1 |     5 |            |          |
|*  2 |   INDEX RANGE SCAN| TI   |    49 |   245 |    3    (0)| 00:00:01 |
```

它会很乐意使用索引；然而，我们会发现，当通过索引估计要检索的行数超过某个阈值时（该阈值根据各种优化器设置、物理统计信息、版本等而变化），我们会开始观察到全表扫描：

```
SQL> select count(*) from t where x < 1000000;
COUNT(*)

Execution Plan

...

| Id  | Operation          | Name | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT   |      |    1  |     5 |  620    (1)| 00:00:01 |
|   1 |  SORT AGGREGATE    |      |    1  |     5 |            |          |
|*  2 |   TABLE ACCESS FULL| T    |  999K |  4882K|  620    (1)| 00:00:01 |
```

这个例子表明优化器不会*总是*使用索引，事实上，在跳过索引时它做出了正确的选择。在优化查询时，如果你发现一个索引在你认为*应该*被使用时却没有被使用，不要仅仅强制使用它——在推翻 CBO 之前，请先测试并证明使用索引确实更快（通过耗时和 I/O 计数）。要合理分析。

#### 案例 6

表的统计信息不新鲜了。这些表曾经很小，但现在我们查看时，它们已经变得相当大。现在使用索引是有意义的，而最初则不是。如果我们为表生成统计信息，它就会使用索引。

没有正确的统计信息，CBO *无法*做出正确的决策。

#### 索引案例总结

根据我的经验，我发现索引未被使用的*主要*原因就是这六个案例。这通常归结为“它们不能被使用——使用它们会返回错误的结果”，或者“它们不应该被使用——如果使用了它们，性能会非常糟糕”。



