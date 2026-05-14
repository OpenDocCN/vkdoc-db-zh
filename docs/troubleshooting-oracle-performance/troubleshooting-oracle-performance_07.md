# 绑定变量

绑定变量通过两个方面影响应用程序。首先，从开发角度来看，它们使编程变得更简单或更困难（更准确地说，需要编写更多或更少的代码）。在这种情况下，其影响取决于用于执行 SQL 语句的应用程序编程接口。例如，如果你在编写 PL/SQL 代码，使用绑定变量执行它们会更容易。另一方面，如果你使用 Java 和 JDBC 进行编程，不使用绑定变量执行 SQL 语句则更为简便。其次，从性能角度来看，绑定变量既带来优势也带来劣势。

***

**注意** 在以下章节中，你将看到一些执行计划。如何获取和解释执行计划将在第 6 章中说明。如果有什么不清楚的地方，你可以考虑稍后再回到本章。

***

## 优势

绑定变量的优势在于它们允许在库缓存中共享游标，从而避免硬解析及其相关的开销。下面的示例摘自脚本 `bind_variables.sql` 的输出，展示了三条 `INSERT` 语句，由于使用了绑定变量，它们在库缓存中共享同一个游标：

```
SQL> variable n NUMBER
SQL> variable v VARCHAR2(32)
SQL> execute :n := 1; :v := 'Helicon';
SQL> INSERT INTO t (n, v) VALUES (:n, :v);
SQL> execute :n := 2; :v := 'Trantor';
SQL> INSERT INTO t (n, v) VALUES (:n, :v);
SQL> execute :n := 3; :v := 'Kalgan';
SQL> INSERT INTO t (n, v) VALUES (:n, :v);
```

```
SQL> SELECT sql_id, child_number, executions
  2  FROM v$sql
  3  WHERE sql_text = 'INSERT INTO t (n, v) VALUES (:n, :v)';

SQL_ID        CHILD_NUMBER EXECUTIONS
------------- ------------ ----------
6cvmu7dwnvxwj            0          3
```

然而，存在一些即使使用绑定变量也会创建多个子游标的情况。下面的示例展示了这种情况。请注意，`INSERT` 语句与之前的示例相同。只有 `VARCHAR2` 变量的最大大小发生了变化（从 32 变为 33）。

```
SQL> variable v VARCHAR2(33)
SQL> execute :n := 4; :v := 'Terminus';
SQL> INSERT INTO t (n, v) VALUES (:n, :v);
```

```
SQL> SELECT sql_id, child_number, executions
  2  FROM v$sql
  3  WHERE sql_text = 'INSERT INTO t (n, v) VALUES (:n, :v)';

SQL_ID        CHILD_NUMBER EXECUTIONS
------------- ------------ ----------
6cvmu7dwnvxwj            0          3
6cvmu7dwnvxwj            1          1
```

创建新的子游标 (1) 是因为前三条 `INSERT` 语句与第四条之间的执行环境发生了变化。通过查询视图 `v$sql_shared_cursor` 可以确认，这种不匹配是由绑定变量引起的。

```
SQL> SELECT child_number, bind_mismatch
  2  FROM v$sql_shared_cursor
  3  WHERE sql_id = '6cvmu7dwnvxwj';

CHILD_NUMBER BIND_MISMATCH
------------ -------------
           0 N
           1 Y
```

发生的情况是数据库引擎应用 *绑定变量分级* 功能。此功能旨在通过根据大小将绑定变量（其大小各不相同）分为四组来最小化子游标的数量。第一组包含最多 32 字节的绑定变量，第二组包含 33 到 128 字节之间的绑定变量，第三组包含 129 到 2,000 字节之间的绑定变量，最后一组包含超过 2,000 字节的绑定变量。数据类型为 `NUMBER` 的绑定变量会被分级为其最大长度，即 22 字节。如下面的示例所示，视图 `v$sql_bind_metadata` 显示一个组的最大大小。请注意即使子游标 1 的变量定义为 33，这里仍然使用了值 128。

```
SQL> SELECT s.child_number, m.position, m.max_length,
  2         decode(m.datatype,1,'VARCHAR2',2,'NUMBER',m.datatype) AS datatype
  3  FROM v$sql s, v$sql_bind_metadata m
  4  WHERE s.sql_id = '6cvmu7dwnvxwj'
  5  AND s.child_address = m.address
  6  ORDER BY 1, 2;

CHILD_NUMBER   POSITION MAX_LENGTH DATATYPE
------------ ---------- ---------- ----------------------------------------
           0          1         22 NUMBER
           0          2         32 VARCHAR2
           1          1         22 NUMBER
           1          2        128 VARCHAR2
```

不言而喻，每当创建一个新的子游标时，都会生成一个执行计划。这个新的执行计划是否与另一个子游标使用的执行计划相同，还取决于绑定变量的值。这将在下一节中描述。

## 劣势

在 `WHERE` 子句中使用绑定变量的劣势在于，关键信息对查询优化器是隐藏的。实际上，对于查询优化器来说，使用字面值比使用绑定变量要好得多。使用字面值，它能够改进其估算。当它检查一个值是否在可用值范围之外（即小于最小值或大于列中存储的最大值）以及当它利用直方图时，这一点尤其重要。为了说明，让我们以表 `t` 为例，该表有 1,000 行，在 `id` 列中存储了介于 1（最小值）和 1,000（最大值）之间的值：

```
SQL> SELECT count(id), count(DISTINCT id), min(id), max(id) FROM t;

COUNT(ID) COUNT(DISTINCTID)    MIN(ID)    MAX(ID)
---------- ----------------- ---------- ----------
      1000              1000          1       1000
```

当用户选择所有 `id` 小于 990 的行时，查询优化器知道（得益于对象统计信息）选择了大约 99% 的表数据。因此，它选择了一个包含全表扫描的执行计划。同时请注意估算的基数（执行计划中的 `Rows` 列）与查询返回的行数相符。

```
SQL> SELECT count(pad) FROM t WHERE id < 990;

COUNT(PAD)
----------
       989
```

```
-------------------------------------------
| Id | Operation            | Name | Rows |
-------------------------------------------
|  0 | SELECT STATEMENT     |      |      |
|  1 |  SORT AGGREGATE      |      |    1 |
|  2 |   TABLE ACCESS FULL  | T    |  990 |
-------------------------------------------
```

当另一个用户选择所有 `id` 小于 10 的行时，查询优化器知道只选择了大约 1% 的表数据。因此，它选择了一个包含索引扫描的执行计划。同样在这里，请注意良好的估算。

```
SQL> SELECT count(pad) FROM t WHERE id < 10;

COUNT(PAD)
----------
         9
```

```
-----------------------------------------------------
| Id | Operation                      | Name | Rows |
-----------------------------------------------------
|  0 | SELECT STATEMENT               |      |      |
|  1 |  SORT AGGREGATE                |      |    1 |
|  2 |   TABLE ACCESS BY INDEX ROWID  | T    |    9 |
|  3 |    INDEX RANGE SCAN            | T_PK |    9 |
-----------------------------------------------------
```

每当处理绑定变量时，查询优化器过去常常忽略它们的值。因此，像前面例子中那样的良好估算是不可能的。为了解决这个问题，Oracle9*i* 中引入了一项名为 *绑定变量窥探* 的功能。

***

**注意** Oracle9*i* 附带的 JDBC thin 驱动程序不支持绑定变量窥探。此限制记录在 MetaLink 注释 273635.1 中。

***



`绑定变量窥探`的概念很简单。在物理优化阶段，查询优化器会窥探绑定变量的值，并将它们当作字面量使用。这种方法的问题在于，生成的执行计划依赖于第一次执行时提供的值。下面的示例基于脚本`bind_variables_peeking.sql`说明了这种行为。请注意，第一次优化是使用值`990`执行的。因此，查询优化器选择了全表扫描。正是这个选择，由于游标是共享的（`sql_id`和子游标号相同），影响了第二次使用`10`进行选择的查询。

```sql
SQL> variable id NUMBER
SQL> execute :id := 990;
SELECT count(pad) FROM t WHERE id < :id;

COUNT(PAD)
----------
989

SQL_ID asth1mx10aygn, child number 0
-------------------------------------------
| Id | Operation            | Name | Rows |
-------------------------------------------
|  0 | SELECT STATEMENT     |      |      |
|  1 |  SORT AGGREGATE      |      |    1 |
|  2 |   TABLE ACCESS FULL  | T    |  990 |
-------------------------------------------

SQL> execute :id := 10;
SQL> SELECT count(pad) FROM t WHERE id < :id;

COUNT(PAD)
----------
9

SQL_ID asth1mx10aygn, child number 0
-------------------------------------------
| Id | Operation            | Name | Rows |
-------------------------------------------
|  0 | SELECT STATEMENT     |      |      |
|  1 |  SORT AGGREGATE      |      |    1 |
|  2 |   TABLE ACCESS FULL  | T    |  990 |
-------------------------------------------
```

当然，如下例所示，如果第一次执行使用值`10`，查询优化器会选择带有索引扫描的执行计划——而且，这同样会影响两次查询。请注意，为了避免共享上一个示例中使用的游标，这些查询是用小写字母编写的。

```sql
SQL> execute :id := 10;
SQL> select count(pad) from t where id < :id;

COUNT(PAD)
----------
9

SQL_ID 7h6n1xkn8trkd, child number 0
-----------------------------------------------------
| Id | Operation                      | Name | Rows |
-----------------------------------------------------
|  0 | SELECT STATEMENT               |      |      |
|  1 |  SORT AGGREGATE                |      |    1 |
|  2 |   TABLE ACCESS BY INDEX ROWID  | T    |    9 |
|  3 |    INDEX RANGE SCAN            | T_PK |    9 |
-----------------------------------------------------

SQL> execute :id := 990;
SQL> select count(pad) from t where id < :id;

COUNT(PAD)
----------
989

SQL_ID 7h6n1xkn8trkd, child number 0
-----------------------------------------------------
| Id | Operation                      | Name | Rows |
-----------------------------------------------------
|  0 | SELECT STATEMENT               |      |      |
|  1 |  SORT AGGREGATE                |      |    1 |
|  2 |   TABLE ACCESS BY INDEX ROWID  |    T |    9 |
|* 3 |    INDEX RANGE SCAN            | T_PK |    9 |
-----------------------------------------------------
```

必须理解的是，只要游标保留在库缓存中并且可以共享，它就会被重用。无论与之相关的执行计划效率如何，都会发生这种情况。

为了解决这个问题，从 Oracle Database 11g 开始，提供了一项名为`扩展游标共享`（也称为`自适应游标共享`）的新功能。其目的是自动识别重用已有游标何时会导致低效的执行。要理解此功能的工作原理，让我们从查看之前示例中所用 SQL 语句的`v$sql`视图内容开始。从 Oracle Database 11g 开始，提供了以下新列：

`is_bind_sensitive`
不仅指示是否使用绑定变量窥探来生成执行计划，还指示执行计划是否依赖于窥探到的值。如果是这种情况，则该列设置为`Y`；否则，设置为`N`。

`is_bind_aware`
指示游标是否正在使用扩展游标共享。如果是，则该列设置为`Y`；如果不是，则设置为`N`。如果设置为`N`，则该游标已过时，将不再使用。

`is_shareable`
指示游标是否可以共享。如果可以，则该列设置为`Y`；否则，设置为`N`。如果设置为`N`，则该游标已过时，将不再使用。

在以下示例中，游标是可共享的，并且对绑定变量敏感，但它并未使用扩展游标共享：

```sql
SQL> SELECT child_number, is_bind_sensitive, is_bind_aware, is_shareable
  2  FROM v$sql
  3  WHERE sql_id = '7h6n1xkn8trkd'
  4  ORDER BY child_number;

CHILD_NUMBER IS_BIND_SENSITIVE IS_BIND_AWARE IS_SHAREABLE
------------ ----------------- ------------- ------------
           0 Y                 N             Y
```

当使用不同的绑定变量值多次执行该游标时，会发生一些有趣的事情。在使用`10`和`990`这两个值执行几次之后，`v$sql`视图提供的信息变得不同。请注意，子游标号`0`不再可共享，而两个新的子游标开始使用扩展游标共享。

```sql
SQL> SELECT child_number, is_bind_sensitive, is_bind_aware, is_shareable
  2  FROM v$sql
  3  WHERE sql_id = '7h6n1xkn8trkd'
  4  ORDER BY child_number;

CHILD_NUMBER IS_BIND_SENSITIVE IS_BIND_AWARE IS_SHAREABLE
------------ ----------------- ------------- ------------
           0 Y                 N             N
           1 Y                 Y             Y
           2 Y                 Y             Y
```

查看与游标相关的执行计划，正如您所料，您会看到其中一个新子游标的执行计划基于全表扫描，而另一个则基于索引扫描：

```sql
SQL_ID 7h6n1xkn8trkd, child number 0
---------------------------------------------
| Id | Operation                      | Name |
---------------------------------------------
|  0 | SELECT STATEMENT               |      |
|  1 |  SORT AGGREGATE                |      |
|  2 |   TABLE ACCESS BY INDEX ROWID  |    T |
|  3 |    INDEX RANGE SCAN            | T_PK |
---------------------------------------------

SQL_ID 7h6n1xkn8trkd, child number 1
-----------------------------------
| Id | Operation           | Name |
-----------------------------------
|  0 | SELECT STATEMENT    |      |
|  1 |  SORT AGGREGATE     |      |
|  2 |   TABLE ACCESS FULL | T    |
-----------------------------------

SQL_ID 7h6n1xkn8trkd, child number 2
---------------------------------------------
| Id | Operation                      | Name |
---------------------------------------------
|  0 | SELECT STATEMENT               |      |
|  1 |  SORT AGGREGATE                |      |
|  2 |   TABLE ACCESS BY INDEX ROWID  | T    |
|  3 |    INDEX RANGE SCAN            | T_PK |
---------------------------------------------
```

为了进一步分析生成两个子游标的原因，可以使用新的动态性能视图：`v$sql_cs_statistics`、`v$sql_cs_selectivity`和`v$sql_cs_histogram`。第一个视图显示是否使用了窥探，以及每个子游标的相关执行统计信息。在以下输出中，可以确认对于其中一次执行，子游标`1`处理的行数高于子游标`2`。因此，在一种情况下查询优化器选择了全表扫描，而在另一种情况下选择了索引扫描。

```sql
SQL> SELECT child_number, peeked, executions, rows_processed, buffer_gets
  2  FROM v$sql_cs_statistics
  3  WHERE sql_id = '7h6n1xkn8trkd'
  4  ORDER BY child_number;
```



