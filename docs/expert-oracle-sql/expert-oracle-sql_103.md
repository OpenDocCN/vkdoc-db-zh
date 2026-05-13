# 子查询合并

子查询合并是 Oracle 11gR2 引入的一项转换技术，目前仍处于发展初期。代码清单 13-28 展示了少数几个能够应用此转换的案例之一。

## 代码清单 13-28. 子查询合并

```sql
SELECT *
  FROM sh.sales s1
 WHERE    EXISTS
             (SELECT /*+   coalsesce_sq */
                     /* no_coalesce_sq */
                    *
               FROM sh.sales s2
              WHERE     s1.time_id = s2.time_id
                    AND s2.amount_sold > s1.amount_sold + 100)
       OR EXISTS
             (SELECT /*+   coalsesce_sq */
                     /* no_coalesce_sq */
                    *
               FROM sh.sales s3
              WHERE     s1.time_id = s3.time_id
                    AND s3.amount_sold < s1.amount_sold - 100);

-- 未转换的执行计划 (NO_COALESCE_SQ)

| Id  | Operation                                   | Name           | Cost (%CPU)|

|   0 | SELECT STATEMENT                            |                |  1205K  (1)|
|*  1 |  FILTER                                     |                |            |
|   2 |   PARTITION RANGE ALL                       |                |   518   (2)|
|   3 |    TABLE ACCESS FULL                        | SALES          |   518   (2)|
|   4 |   PARTITION RANGE SINGLE                    |                |     1   (0)|
|*  5 |    TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |     1   (0)|
|   6 |     BITMAP CONVERSION TO ROWIDS             |                |            |
|*  7 |      BITMAP INDEX SINGLE VALUE              | SALES_TIME_BIX |            |
|   8 |   PARTITION RANGE SINGLE                    |                |     1   (0)|
|*  9 |    TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |     1   (0)|
|  10 |     BITMAP CONVERSION TO ROWIDS             |                |            |
|* 11 |      BITMAP INDEX SINGLE VALUE              | SALES_TIME_BIX |            |

Predicate Information (identified by operation id):

1 - filter( EXISTS (SELECT /*+ NO_COALESCE_SQ */ 0 FROM "SH"."SALES"
              "S2" WHERE "S2"."TIME_ID"=:B1 AND "S2"."AMOUNT_SOLD">:B2+100) OR  EXISTS
              (SELECT 0 FROM "SH"."SALES" "S3" WHERE "S3"."TIME_ID"=:B3 AND
              "S3"."AMOUNT_SOLD"<:B4-100))
   5 - filter("S2"."AMOUNT_SOLD">:B1+100)
   7 - access("S2"."TIME_ID"=:B1)
   9 - filter("S3"."AMOUNT_SOLD"<:B1-100)
  11 - access("S3"."TIME_ID"=:B1)

-- 转换后的查询

SELECT *
  FROM sh.sales s1
 WHERE EXISTS
          (SELECT *
             FROM sh.sales s2
            WHERE        s2.amount_sold > s1.amount_sold + 100
                     AND s2.time_id = s1.time_id
                  OR     s2.amount_sold < s1.amount_sold - 100
                     AND s2.time_id = s1.time_id);

-- 转换后的执行计划 (默认)

| Id  | Operation                                   | Name           | Cost (%CPU)|

|   0 | SELECT STATEMENT                            |                |   358K  (1)|
|*  1 |  FILTER                                     |                |            |
|   2 |   PARTITION RANGE ALL                       |                |   517   (2)|
|   3 |    TABLE ACCESS FULL                        | SALES          |   517   (2)|
|   4 |   PARTITION RANGE INLIST                    |                |     5   (0)|
|*  5 |    TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |     5   (0)|
|   6 |     BITMAP CONVERSION TO ROWIDS             |                |            |
|   7 |      BITMAP OR                              |                |            |
|*  8 |       BITMAP INDEX SINGLE VALUE             | SALES_TIME_BIX |            |
|*  9 |       BITMAP INDEX SINGLE VALUE             | SALES_TIME_BIX |            |

Predicate Information (identified by operation id):

1 - filter( EXISTS (SELECT 0 FROM "SH"."SALES" "S2" WHERE
              ("S2"."TIME_ID"=:B1 OR "S2"."TIME_ID"=:B2) AND ("S2"."TIME_ID"=:B3 AND
              "S2"."AMOUNT_SOLD">:B4+100 OR "S2"."TIME_ID"=:B5 AND
              "S2"."AMOUNT_SOLD"<:B6-100)))
   5 - filter("S2"."TIME_ID"=:B1 AND "S2"."AMOUNT_SOLD">:B2+100 OR
              "S2"."TIME_ID"=:B3 AND "S2"."AMOUNT_SOLD"<:B4-100)
   8 - access("S2"."TIME_ID"=:B1)
   9 - access("S2"."TIME_ID"=:B1)
```

代码清单 13-28 中的两个子查询通过 `OR` 组合在一起，这意味着它们无法被反嵌套。然而，这两个子查询可以被合并。请注意，未转换查询的执行计划包含一个具有三个子节点的 `FILTER` 操作，这意味着两个子查询过滤器是分别应用的。而转换后执行计划中的 `FILTER` 操作仅有两个子节点，这意味着只进行了一次过滤。

你会发现在转换后的执行计划中，我们将第 8 行和第 9 行两个相同的位图进行了不必要的合并。如果你自己重写这个查询，可以避免这个不必要的步骤。

子查询合并是我想讨论的最后一项与子查询相关的转换。不过，还有一些其他的转换有待介绍。

