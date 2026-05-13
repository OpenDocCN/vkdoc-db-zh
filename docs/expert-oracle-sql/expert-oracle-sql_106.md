# 查询转换与优化示例

```sql
3 - access("S"."TIME_ID"="T"."TIME_ID")

-- 转换后的查询

WITH vw_gbc
     AS (  SELECT s.time_id, SUM (s.amount_sold) AS dollars
             FROM sh.sales s
         GROUP BY s.time_id)
    ,gset_union
     AS (  SELECT TO_CHAR (vw_gbc.time_id) AS time_id
                 ,t.calendar_month_desc
                 ,SUM (vw_gbc.dollars) AS dollars
             FROM vw_gbc, sh.times t
            WHERE vw_gbc.time_id = t.time_id
         GROUP BY vw_gbc.time_id, t.calendar_month_desc
         UNION ALL
         SELECT 'Month Total' AS time_id, mv.calendar_month_desc, mv.dollars
           FROM sh.cal_month_sales_mv mv)
  SELECT *
    FROM gset_union u
ORDER BY u.calendar_month_desc, u.time_id;
```

```sql
-- 转换后的执行计划（默认）

| Id  | Operation                       | Name               | Cost (%CPU)|

|   0 | SELECT STATEMENT                |                    |   369   (8)|
|   1 |  SORT ORDER BY                  |                    |   369   (8)|
|   2 |   VIEW                          |                    |   369   (8)|
|   3 |    UNION-ALL                    |                    |            |
|   4 |     HASH GROUP BY               |                    |   367   (8)|
|*  5 |      HASH JOIN                  |                    |   367   (8)|
|   6 |       VIEW                      | VW_GBC_6           |   355   (8)|
|   7 |        HASH GROUP BY            |                    |   355   (8)|
|   8 |         PARTITION RANGE ALL     |                    |   334   (3)|
|   9 |          TABLE ACCESS FULL      | SALES              |   334   (3)|
|  10 |       TABLE ACCESS FULL         | TIMES              |    12   (0)|
|  11 |     MAT_VIEW REWRITE ACCESS FULL| CAL_MONTH_SALES_MV |     2   (0)|

谓词信息（通过操作 ID 标识）：

5 - access("ITEM_1"="T"."TIME_ID")
```

Listing 13-31 中的查询提供了每日销售总价值以及按月汇总的总计。尽管这是一个串行查询，但未转换的查询包含某种变体的布隆过滤器修剪，但我们不要因此分心。

转换后的查询执行了一个简单的`GROUP BY`操作，没有使用`ROLLUP`，然后从物化视图中添加了月度总计的行。有趣的是，除了将`SORT GROUP BY ROLLUP`操作替换为更直接的`HASH GROUP BY`操作外，该转换还使得分组放置成为可能！既然新的分组放置转换已经创建，那么旧的扩展分组集到联合（expand-grouping-sets-to-union）转换可能会被重新审视，以便在没有任何物化视图的情况下也能操作。但就目前而言，此转换似乎仅在物化视图可用时才起作用。

## 排序消除

在子查询中出现`ORDER BY`表达式是完全合法且非常常见的，而且你有权依赖此类子查询结果的排序。例如，在 Listing 13-24 中，我在包含`ORDER BY`子句的子查询结果上使用了`ROWNUM=1`谓词。

然而，有时你可能无法依赖子查询的`ORDER BY`子句，因为结果查询是另一个连接、聚合或集合操作的输入。在这种情况下，CBO 认为有权删除你看似多余的`ORDER BY`子句。

问题是：为什么你添加了`ORDER BY`子句却不使用它？几乎总是，多余的`ORDER BY`子句出现在数据字典视图中，而这正是 order-by-elimination 转换旨在解决的情况。另一方面，也可以使用内联视图来演示 order-by-elimination 转换，如 Listing 13-32 所示。

### Listing 13-32. 排序消除

```sql
SELECT COUNT (*)
  FROM (  SELECT /*+   eliminate_oby */
                 /* no_eliminate_oby */
                 o1.*
            FROM oe.order_items o1
           WHERE product_id = (SELECT MAX (o2.product_id)
                                 FROM oe.order_items o2
                                WHERE o2.order_id = o1.order_id)
        ORDER BY order_id) v;
```

```sql
-- 未转换的执行计划（NO_ELIMINATE_OBY）

| Id  | Operation                  | Name           | Cost (%CPU)|

|   0 | SELECT STATEMENT           |                |     5   (0)|
|   1 |  SORT AGGREGATE            |                |            |
|   2 |   VIEW                     |                |     5   (0)|
|   3 |    SORT ORDER BY           |                |     5   (0)|
|*  4 |     HASH JOIN              |                |     5   (0)|
|   5 |      VIEW                  | VW_SQ_1        |     2   (0)|
|   6 |       HASH GROUP BY        |                |     2   (0)|
|   7 |        INDEX FAST FULL SCAN| ORDER_ITEMS_UK |     2   (0)|
|   8 |      TABLE ACCESS FULL     | ORDER_ITEMS    |     3   (0)|

谓词信息（通过操作 ID 标识）：

4 - access("PRODUCT_ID"="MAX(O2.PRODUCT_ID)" AND
              "ITEM_1"="O1"."ORDER_ID")
```

```sql
-- 转换后的查询

SELECT COUNT (*)
  FROM (SELECT o1.*
          FROM oe.order_items o1
         WHERE product_id = (SELECT MAX (o2.product_id)
                               FROM oe.order_items o2
                              WHERE o2.order_id = o1.order_id)) v;
```

```sql
-- 转换后的执行计划（默认）

| Id  | Operation                | Name           | Cost (%CPU)|

|   0 | SELECT STATEMENT         |                |     2   (0)|
|   1 |  SORT AGGREGATE          |                |            |
|   2 |   NESTED LOOPS           |                |     2   (0)|
|   3 |    VIEW                  | VW_SQ_1        |     2   (0)|
|   4 |     HASH GROUP BY        |                |     2   (0)|
|   5 |      INDEX FAST FULL SCAN| ORDER_ITEMS_UK |     2   (0)|
|*  6 |    INDEX UNIQUE SCAN     | ORDER_ITEMS_UK |     0   (0)|

谓词信息（通过操作 ID 标识）：

6 - access("ITEM_1"="O1"."ORDER_ID" AND
              "PRODUCT_ID"="MAX(O2.PRODUCT_ID)")
```

Listing 13-32 中的查询包含一个隐藏在内联视图中的未使用`ORDER BY`子句。未转换的查询包含一个`SORT ORDER BY`操作（执行排序）和一个`SORT AGGREGATE`操作（不执行排序）。转换后的查询删除了冗余的`ORDER BY`子句，最终执行计划只包含评估`COUNT (*)`的不排序的`SORT AGGREGATE`操作。

你可能想知道为什么我突然在 Listing 13-32 中放弃了我喜欢的带因子子查询。原因是 order-by-elimination 不会发生在带因子子查询上！我实际上认为这是合理的事情。如果你在带因子子查询中放置`ORDER BY`子句，那是有原因的。真正需要 order-by-elimination 转换的其实只有数据字典视图。

## 表扩展

表扩展（Table expansion）是 11gR2 中引入的一个转换，特定于分区表。在引入表扩展转换之前，一个分区上本地索引的不可用性可能会妨碍使用其他分区上可用的索引。

为了演示这个转换，我必须更改索引分区的可用性，因此我将创建一个新表，而不是更改示例模式数据的状态。Listing 13-33 创建了一个具有不可用本地索引的分区表，然后仅重建一个分区。

### Listing 13-33. 为表扩展演示创建表


