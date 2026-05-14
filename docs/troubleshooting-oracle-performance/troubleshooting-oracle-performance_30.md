# 使用 `dbms_xplan.display` 函数获取执行计划

你当然可以通过直接查询计划表来获取执行计划。然而，自 Oracle9*i* Release 2 起，有一种更简单、更好的方法——使用 `dbms_xplan` 包中的 `display` 函数。如下面的示例所示，它的用法非常简单。实际上，只需调用该函数就能显示由 `EXPLAIN PLAN` SQL 语句生成的执行计划。请注意，该函数的返回值（是一个集合）是如何通过 `table` 函数进行转换的。

`SQL> EXPLAIN PLAN FOR SELECT * FROM emp WHERE deptno = 10 ORDER BY ename;`

`SQL> SELECT * FROM table(dbms_xplan.display);`

```
PLAN_TABLE_OUTPUT
---------------------------------------------------------------------------

Plan hash value: 150391907

---------------------------------------------------------------------------
| Id  | Operation          | Name | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |     5 |   185 |     4  (25)| 00:00:01 |
|   1 |  SORT ORDER BY     |      |     5 |   185 |     4  (25)| 00:00:01 |
|*  2 |   TABLE ACCESS FULL| EMP  |     5 |   185 |     3   (0)| 00:00:01 |
---------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter("DEPTNO"=10)
```

`display` 函数并不仅限于无参数使用。因此，在本章后面，我将详细介绍 `dbms_xplan` 包，探索其所有功能，包括对生成输出的描述。

