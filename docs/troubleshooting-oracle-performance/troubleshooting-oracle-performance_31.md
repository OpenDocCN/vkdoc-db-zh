# 绑定变量陷阱

在使用 `EXPLAIN PLAN` SQL 语句时，我遇到的最常见错误是指定的 SQL 语句与实际要分析的语句不同。这自然可能导致错误的执行计划。由于格式本身对执行计划没有影响，这种差异通常是由替换绑定变量引起的。让我们检查以下 PL/SQL 过程中查询所使用的执行计划：

```sql
CREATE OR REPLACE PROCEDURE p (p_value IN NUMBER) IS
BEGIN
  FOR i IN (SELECT * FROM emp WHERE empno = p_value)
  LOOP
    -- do something
  END LOOP;
END;
```

一个常用的技术是复制/粘贴查询，并将 PL/SQL 变量替换为一个具体的值。然后你以如下方式执行 SQL 语句：

`EXPLAIN PLAN FOR SELECT * FROM emp WHERE empno = 7788`

问题在于，通过用常量替换绑定变量，你向查询优化器提交了一个不同的 SQL 语句。这种改变——可能是由于 SQL profiles、stored outlines、SQL plan baselines 或查询优化器用于估算 `WHERE` 子句中谓词选择性的方法——可能会影响查询优化器所做的决策。

正确的方法是使用相同的 SQL 语句。这是可行的，因为绑定变量可以与 `EXPLAIN PLAN` SQL 语句一起使用。例如，你应该执行类似以下内容的 SQL 语句。请注意，在 PL/SQL 变量前添加了一个冒号，以将其转换为 `EXPLAIN PLAN` SQL 语句的变量。

`EXPLAIN PLAN FOR SELECT * FROM emp WHERE empno = :p_value`

尽管如此，将绑定变量与 `EXPLAIN PLAN` SQL 语句一起使用有两个问题。第一个问题是，默认情况下，绑定变量被声明为 `VARCHAR2` 类型。因此，为了避免隐式转换，数据库引擎会自动添加显式转换。你可以通过包 `dbms_xplan` 中 `display` 函数生成的输出末尾显示的谓词信息来检查这一点。在下面的输出示例中，函数 `to_number` 就用于此目的：

`SQL> SELECT * FROM table(dbms_xplan.display);`

```
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------
Plan hash value: 4024650034

------------------------------------------------------------------------------------
| Id | Operation                   | Name   | Rows | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------
|  0 | SELECT STATEMENT            |        |    1 |    37 |     1   (0)| 00:00:01 |
|  1 |  TABLE ACCESS BY INDEX ROWID| EMP    |    1 |    37 |     1   (0)| 00:00:01 |
|* 2 |   INDEX UNIQUE SCAN         | EMP_PK |    1 |       |     0   (0)| 00:00:01 |
------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("EMPNO"=TO_NUMBER(:P_VALUE))
```

通常良好的做法是检查数据类型是否得到正确处理，例如，对所有非 `VARCHAR2` 类型的绑定变量使用显式转换。

将绑定变量与 `EXPLAIN PLAN` SQL 语句一起使用的第二个问题是，不会使用绑定变量窥探（bind variable peeking）。由于该问题没有解决方案，因此不能保证 `EXPLAIN PLAN` SQL 语句生成的执行计划就是运行时选择的计划。换句话说，每当涉及绑定变量时，`EXPLAIN PLAN` SQL 语句生成的输出是不可靠的。


#### 动态性能视图

四个动态性能视图显示库缓存中存在的游标信息：

*   `v$sql_plan` 提供的信息基本上与计划表相同。换句话说，它提供由查询优化器生成的执行计划及其他相关信息。该视图与计划表之间唯一的显著差异在于，该视图中有一些列用于标识库缓存中与执行计划相关的游标。
*   `v$sql_plan_statistics` 为 `v$sql_plan` 视图中的每个操作（*行源操作*）提供执行统计信息，例如耗时和产生的行数。本质上，它提供了执行计划的运行时行为。这是一个重要的信息片段，因为 `v$sql_plan` 视图仅显示查询优化器在解析时所做的估算和决策。由于收集执行统计信息会产生不可忽略的开销，默认情况下它们不会被收集。要激活收集，必须将初始化参数 `statistics_level` 设置为 `all`，或者在 SQL 语句中指定提示 `gather_plan_statistics`。
*   `v$sql_workarea` 提供有关执行游标所需的内存工作区的信息。它提供运行时内存使用情况，以及对高效执行操作所需内存量的估算。
*   `v$sql_plan_statistics_all` 在单个视图中显示由视图 `v$sql_plan`、`v$sql_plan_statistics` 和 `v$sql_workarea` 提供的所有信息。使用它，你可以轻松避免手动连接多个视图。

库缓存中的游标（因此也存在于这些视图中）由三个列标识：`address`、`hash_value` 和 `child_number`。使用 `address` 和 `hash_value` 列，你可以标识父游标。使用所有三个列，你可以标识子游标。此外，从 Oracle Database 10*g* 开始，也可以并且更常见的是使用 `sql_id` 列来代替 `address` 和 `hash_value` 列来标识游标。使用 `sql_id` 列的优势在于，其值仅取决于 SQL 语句本身。换句话说，对于给定的 SQL 语句，它永远不会改变。另一方面，`address` 列是指向内存中 SQL 语句句柄的指针，可能会随时间变化。要标识一个游标，基本上你面临两种搜索方法：要么你知道正在执行 SQL 语句的会话，要么你知道 SQL 语句的文本。在这两种情况下，一旦游标被标识，你就可以显示有关它的信息。

### 标识子游标

你面临的第一个常见情况是尝试获取与当前连接到实例的会话相关的 SQL 语句的信息。在这种情况下，你在视图 `v$session` 上执行搜索。当前执行的 SQL 语句由 `sql_id`（或 `sql_address` 和 `sql_hash_value`）以及 `sql_child_number` 列标识。最后执行的 SQL 语句由 `prev_sql_id`（或 `prev_sql_addr` 和 `prev_hash_value`）以及 `prev_child_number` 列标识。请注意，`sql_id`、`sql_child_number` 和 `prev_child_number` 列仅在 Oracle Database 10*g* 及以上版本可用。显然，在 Oracle9*i* 中，由于缺少与子游标的关系，如果存在多个子游标，则无法实现会话与游标之间的直接映射。为了说明此方法的使用，假设用户 Curtis 打电话给你，抱怨他正在等待几分钟前用应用程序提交的一个请求。对于此问题，直接查询视图 `v$session` 非常有用，如下例所示。根据该输出，你知道他当前正在运行一个 SQL 语句（否则状态不会是 `ACTIVE`），并且知道哪个游标与他的会话相关。

```sql
SQL> SELECT status, sql_id, sql_child_number
  2  FROM v$session
  3  WHERE username = 'CURTIS';

STATUS   SQL_ID        SQL_CHILD_NUMBER
-------- ------------- ----------------
ACTIVE   1hqjydsjbvmwq                0
```

第二种常见情况是，你确实知道要查找更多信息的 SQL 语句的文本。在这种情况下，你在视图 `v$sql` 上执行搜索。与游标关联的文本可在 `sql_text` 和 `sql_fulltext` 列中获得。第一列仅通过 `VARCHAR2(1000)` 显示文本的第一部分，第二列通过 `CLOB` 显示完整文本。例如，如果你知道要查找的 SQL 语句包含文本为 "online discount" 的字面量，可以使用以下查询来找出游标的标识符：

```sql
SQL> SELECT sql_id, child_number, sql_text
  2  FROM v$sql
  3  WHERE sql_text LIKE '%online discount%' AND sql_text NOT LIKE '%v$sql%';

SQL_ID        CHILD_NUMBER SQL_TEXT
------------- ------------ ------------------------------------------------------
1hqjydsjbvmwq            0 SELECT SUM(AMOUNT_SOLD) FROM SALES S, PROMOTIONS P
                           WHERE S.PROMO_ID = P.PROMO_ID AND PROMO_SUBCATEGORY =
                           'online discount'
```

### 查询动态性能视图

要获取执行计划，你可以直接对动态性能视图 `v$sql_plan` 和 `v$sql_plan_statistics_all` 运行查询。然而，从 Oracle Database 10*g* 开始，有一种更简单且更好的方法——你可以使用包 `dbms_xplan` 中的函数 `display_cursor`。如下例所示，其用法与之前讨论的函数 `display` 类似。唯一的区别是，传递给函数的两个参数用于标识要显示的子游标。

```sql
SQL> SELECT * FROM table(dbms_xplan.display_cursor('1hqjydsjbvmwq',0));

PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------
SQL_ID 1hqjydsjbvmwq, child number 0
-------------------------------------
SELECT SUM(AMOUNT_SOLD) FROM SALES S, PROMOTIONS P WHERE S.PROMO_ID =
P.PROMO_ID AND PROMO_SUBCATEGORY = 'online discount'

Plan hash value: 265338492

------------------------------------------------------------------------------------
| Id  | Operation             | Name       | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT      |            |       |       |   517 (100)|          |
|   1 |  SORT AGGREGATE       |            |     1 |    30 |            |          |
|*  2 |   HASH JOIN           |            |   913K|    26M|   517   (4)| 00:00:07 |
|*  3 |    TABLE ACCESS FULL  | PROMOTIONS |    23 |   483 |    17   (0)| 00:00:01 |
|   4 |    PARTITION RANGE ALL|            |   918K|  8075K|   494   (3)| 00:00:06 |
|   5 |     TABLE ACCESS FULL | SALES      |   918K|  8075K|   494   (3)| 00:00:06 |
------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("S"."PROMO_ID"="P"."PROMO_ID")
   3 - filter("PROMO_SUBCATEGORY"='online discount')
```

函数 `display_cursor` 并非仅限于无参数使用。因此，在本章后面，我将介绍包 `dbms_xplan`，探索所有可能性，包括对生成输出的描述。

