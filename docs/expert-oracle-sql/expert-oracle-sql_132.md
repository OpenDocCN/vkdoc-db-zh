# SQL 查询优化分析

## 查询语句
```sql
SELECT /*+ leading(v2) use_nl(s3) */
      s3.*, v2.country_sales_total
  FROM (  SELECT /*+ no_merge */
                v1.sales_rowid
                , (SELECT /*+ no_unnest */
                          SUM (amount_sold)
                     FROM sh.customers c1 JOIN sh.sales s1 USING (cust_id)
                    WHERE c1.country_id = v1.country_id)
                    country_sales_total
            FROM (  SELECT /*+ no_merge no_eliminate_oby */
                          c2.country_id, s2.ROWID sales_rowid
                      FROM sh.customers c2 JOIN sh.sales s2 USING (cust_id)
                     WHERE s2.time_id BETWEEN DATE '2000-01-01'
                                          AND DATE '2000-12-31'
                  ORDER BY c2.country_id, s2.ROWID) v1
        ORDER BY country_sales_total) v2
      ,sh.sales s3
 WHERE s3.ROWID = v2.sales_rowid;
```

## 查询意图解析
在查看 Listing 18-13 中查询的执行计划之前，让我们先不考虑其中嵌入的提示，理解这个查询的意图。

首先看标记为 `V1` 的内联视图。这个子查询从 `SH.SALES` 获取 `ROWID`，从 `SH.CUSTOMERS` 获取 `COUNTRY_ID`，然后对它们进行排序，使得 `SH.SALES` 中来自同一国家的所有行排列在一起。外部子查询 `V2` 获取这个排序后的数据，并在 `SELECT` 列表中使用关联子查询来获取相关国家的 `COUNTRY_SALES_TOTAL` 值。因为 `V1` 的数据是按 `COUNTRY_ID` 排序的，我们可以完全确信标量子查询缓存机制能确保特定国家的聚合操作不会被重复执行。

现在，我们在 `SELECT` 列表中用 `COUNTRY_SALES_TOTAL` 替换了 `COUNTRY_ID`，就可以使用 `COUNTRY_SALES_TOTAL` 作为第二次排序的键，这次排序仍然只有两列。最后，主查询通过 `ROWID` 从 `SH.SALES` 获取额外的列，来扩展结果集中的行。

## 执行计划对比
Listing 18-14 展示了 Listing 18-13 中查询的两个执行计划。第一个执行计划是在没有提示的情况下获得的，第二个是在有提示的情况下获得的。

Listing 18-14. Listing 18-13 的执行计划

```
-- 无提示的执行计划

| Id  | Operation                   | Name      | Bytes |TempSpc| Cost (%CPU)|

|   0 | SELECT STATEMENT            |           |    42M|       | 12922   (1)|
|   1 |  SORT ORDER BY              |           |    42M|    51M| 12922   (1)|
|*  2 |   HASH JOIN RIGHT OUTER     |           |    42M|       |  2024   (2)|
|   3 |    VIEW                     | VW_SSQ_1  |   342 |       |   961   (4)|
|   4 |     HASH GROUP BY           |           |   532 |       |   961   (4)|
|*  5 |      HASH JOIN              |           |   193K|       |   961   (4)|
|   6 |       VIEW                  | VW_GBC_7  |   124K|       |   538   (6)|
|   7 |        HASH GROUP BY        |           | 70590 |       |   538   (6)|
|   8 |         PARTITION RANGE ALL |           |  8973K|       |   517   (2)|
|   9 |          TABLE ACCESS FULL  | SALES     |  8973K|       |   517   (2)|
|  10 |       TABLE ACCESS FULL     | CUSTOMERS |   541K|       |   423   (1)|
|* 11 |    HASH JOIN                |           |  8796K|  1200K|  1061   (1)|
|  12 |     TABLE ACCESS FULL       | CUSTOMERS |   541K|       |   423   (1)|
|  13 |     PARTITION RANGE ITERATOR|           |  6541K|       |   131   (2)|
|* 14 |      TABLE ACCESS FULL      | SALES     |  6541K|       |   131   (2)|

谓词信息（通过操作 ID 识别）：
2 - access("ITEM_1"(+)="C2"."COUNTRY_ID")
   5 - access("C1"."CUST_ID"="ITEM_1")
  11 - access("C2"."CUST_ID"="S2"."CUST_ID")
  14 - filter("S2"."TIME_ID"<=TO_DATE(' 2000-12-31 00:00:00',
              'syyyy-mm-dd hh24:mi:ss'))

-- 有提示的执行计划

| Id  | Operation                      | Name      | Bytes |TempSpc| Cost (%CPU)|

|   0 | SELECT STATEMENT               |           |    11M|       |   253K  (1)|
|   1 |  SORT AGGREGATE                |           |    20 |       |            |
|*  2 |   HASH JOIN                    |           |  7426K|       |   942   (2)|
|*  3 |    TABLE ACCESS FULL           | CUSTOMERS | 29210 |       |   423   (1)|
|   4 |    PARTITION RANGE ALL         |           |  8973K|       |   517   (2)|
|   5 |     TABLE ACCESS FULL          | SALES     |  8973K|       |   517   (2)|
|   6 |  NESTED LOOPS                  |           |    11M|       |   253K  (1)|
|   7 |   VIEW                         |           |  5638K|       | 22723   (1)|
|   8 |    SORT ORDER BY               |           |  5638K|  8184K| 22723   (1)|
|   9 |     VIEW                       |           |  5638K|       |  3162   (1)|
|  10 |      SORT ORDER BY             |           |  7894K|    10M|  3162   (1)|
|* 11 |       HASH JOIN                |           |  7894K|  1200K|  1018   (1)|
|  12 |        TABLE ACCESS FULL       | CUSTOMERS |   541K|       |   423   (1)|
|  13 |        PARTITION RANGE ITERATOR|           |  5638K|       |   131   (2)|
|* 14 |         TABLE ACCESS FULL      | SALES     |  5638K|       |   131   (2)|
|  15 |   TABLE ACCESS BY USER ROWID   | SALES     |    29 |       |     1   (0)|

谓词信息（通过操作 ID 识别）：
2 - access("C1"."CUST_ID"="S1"."CUST_ID")
   3 - filter("C1"."COUNTRY_ID"=:B1)
  11 - access("C2"."CUST_ID"="S2"."CUST_ID")
  14 - filter("S2"."TIME_ID"<=TO_DATE(' 2000-12-31 00:00:00',
              'syyyy-mm-dd hh24:mi:ss'))
```

## 提示的必要性分析
没有提示时，我们为了让 SQL 语句按期望方式运行所付出的所有努力都白费了。我们可以看到子查询解嵌套、`GROUP BY` 位置调整、`ORDER BY` 消除等转换的证据。CBO 估计的最后对完整行进行排序所需的最大临时表空间消耗为 51M。

带有提示的执行计划看起来更符合我们的期望，尽管我不得不检查列投影数据来验证关联子查询具体在何时被评估。粗略来看，子查询似乎是在最后才评估，但实际上 `COUNTRY_SALES_TOTAL` 是第 8 行 `SORT ORDER BY` 的键，关联子查询正是在那里被评估的。

添加提示后，我们可以看到最大表空间消耗减少了五倍，降至 10M。这 10M 的消耗出现在执行计划中我们有两个活动工作区的点上：一个用于第 11 行的 `HASH JOIN`，另一个用于第 2 行的 `SORT ORDER BY`。

但我们为什么需要这么多提示？它们真的都是必需的吗？答案是肯定的，它们都是必要的。[¹] 事实上，至少有理由再添加一个提示。分析如下：

*   针对 `V1` 和 `V2` 的两个 `NO_MERGE` 提示是必需的，因为简单的视图合并是一种无条件应用的启发式转换。
*   针对关联子查询的 `UNNEST` 提示也是必需的，因为启发式的子查询解嵌套转换具有无条件性。
*   `V1` 中的 `NO_ELIMINATE_OBY` 提示是必需的，因为当后续需要排序时，`ORDER BY` 消除是一种总是被应用的启发式转换。事实上，即使 `V2` 后续的连接操作目前不会触发 `ORDER BY` 消除，我也会给 `V2` 添加一个 `NO_ELIMINATE_OBY` 提示；在未来版本中，`ORDER BY` 消除转换可能会扩展到涵盖连接操作。当然，在实际工作中，我会使用带别名的子查询，在我比较谨慎的日子里，我甚至会给它们也加上 `NO_ELIMINATE_OBY` 提示！
*   主查询中的 `LEADING` 和 `USE_NL` 提示是必需的，因为正如我们在前一章讨论的原因，无法在主查询中包含 `ORDER BY` 子句。

[¹]: 脚注：它们都是必要的。


