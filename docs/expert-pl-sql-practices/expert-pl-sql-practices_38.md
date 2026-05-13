# 基于函数的索引

应用于索引列的函数会完全禁用该索引在执行计划中的使用。为了解决这个问题，自 Oracle 8*i* 起就提供了基于函数的索引，允许对此类表达式进行索引。在下面的清单中，我演示了基于函数的索引在一个使用 PL/SQL 函数谓词的查询中的用法。清单 9-44 从一个应用于高选择性索引列的简单 PL/SQL 函数开始。

## 清单 9-44. 应用于索引列的 PL/SQL 函数

```sql
SQL> CREATE FUNCTION promo_function(
  2                  p_promo_category IN VARCHAR2
  3                  ) RETURN VARCHAR2 DETERMINISTIC IS
  4  BEGIN
  5     RETURN UPPER(p_promo_category);
  6  END promo_function;
  7  /
```

`Function created.`

```sql
SQL> SELECT *
  2  FROM   sales       s
  3  ,      promotions  p
  4  ,      times       t
  5  WHERE  s.promo_id = p.promo_id
  6  AND    s.time_id  = t.time_id
  7  AND    t.time_id BETWEEN DATE '2000-01-01' AND DATE '2000-03-31'
  8  AND    `promo_function(p.promo_category)` = 'AD NEWS';
```

```
Execution Plan
----------------------------------------------------------

-------------------------------------------------------------
| Id  | Operation                       | Name       | Rows  |
-------------------------------------------------------------
|   0 | SELECT STATEMENT                |            | 51609 |
|*  1 |  HASH JOIN                      |            | 51609 |
|   2 |   PART JOIN FILTER CREATE       | :BF0000    |    92 |
|   3 |    TABLE ACCESS BY INDEX ROWID| TIMES      |    92 |
|*  4 |     INDEX RANGE SCAN          | TIMES_PK   |    92 |
|*  5 |   HASH JOIN                   |            | 52142 |
|*  6 |    TABLE ACCESS FULL          | PROMOTIONS |     5 |
|   7 |    PARTITION RANGE SINGLE     |            | 62197 |
|*  8 |     TABLE ACCESS FULL         | SALES      | 62197 |
-------------------------------------------------------------
```

你可以看到，尽管在 `PROMO_CATEGORY` 列上有一个索引，但对该列应用 PL/SQL 函数迫使 CBO 选择对 `PROMOTIONS` 表进行 *全表扫描*（它别无选择）。幸运的是，我可以使用基于函数的索引来解决这个问题（*仅因为* `PROMO_FUNCTION` 是确定性的）。在清单 9-45 中，我创建了基于函数的索引并包含了新的执行计划。

## 清单 9-45. 使用基于函数的索引

```sql
SQL> CREATE INDEX promotions_fbi
  2     ON promotions (`promo_function(promo_category)`)
  3     COMPUTE STATISTICS;
```

`Index created.`

```
Execution Plan
----------------------------------------------------------
Plan hash value: 3568243509

-----------------------------------------------------------------
| Id  | Operation                       | Name           | Rows  |
-----------------------------------------------------------------
|   0 | SELECT STATEMENT                |                | 51609 |
|*  1 |  HASH JOIN                      |                | 51609 |
|   2 |   PART JOIN FILTER CREATE       | :BF0000        |    92 |
|   3 |    TABLE ACCESS BY INDEX ROWID| TIMES          |    92 |
|*  4 |     INDEX RANGE SCAN          | TIMES_PK       |    92 |
|*  5 |   HASH JOIN                   |                | 52142 |
|   6 |    TABLE ACCESS BY INDEX ROWID| PROMOTIONS     |     5 |
|*  7 |     INDEX RANGE SCAN          | PROMOTIONS_FBI |     2 |
|   8 |    PARTITION RANGE SINGLE     |                | 62197 |
|*  9 |     TABLE ACCESS FULL         | SALES          | 62197 |
-----------------------------------------------------------------
```

```
Predicate Information (identified by operation id):
---------------------------------------------------

   1 - access("S"."TIME_ID"="T"."TIME_ID")
   4 - access("T"."TIME_ID">=TO_DATE(' 2000-01-01 00:00:00', 'syyyy-mm-dd hh24:mi:ss') AND
              "T"."TIME_ID"<=TO_DATE(' 2000-03-31 00:00:00', 'syyyy-mm-dd hh24:mi:ss'))
   5 - access("S"."PROMO_ID"="P"."PROMO_ID")
   7 - access(`"SH"."PROMO_FUNCTION"("PROMO_CATEGORY")`='AD NEWS')
   9 - filter("S"."TIME_ID"<=TO_DATE(' 2000-03-31 00:00:00', 'syyyy-mm-dd hh24:mi:ss'))
```

这次 CBO 选择使用基于函数的索引。不仅因为索引存在，而且 CBO 还有一些统计数据可用于选择最优计划。

如果你想研究基于函数的索引，可以在诸如 `USER_INDEXES`、`USER_IND_EXPRESSIONS` 和 `USER_TAB_COLS`/`USER_IND_COLUMNS`（索引表达式作为被索引表中的隐藏虚拟列存储）等视图中找到关于其实现的元数据。



