# Oracle SQL 执行计划分析

## 执行计划与谓词信息

谓词信息（由操作 ID 标识）：

```
1 - filter("S1"."AMOUNT_SOLD"> (SELECT /*+ PUSH_SUBQ */
              AVG("S2"."AMOUNT_SOLD") FROM "SH"."SALES" "S2" WHERE "S2"."PROD_ID"=:B1))
2 - access("S1"."CUST_ID"="C"."CUST_ID")
3 - filter("C"."CUST_YEAR_OF_BIRTH"=1919)
4 - access("S1"."PROD_ID"="P"."PROD_ID")
6 - access("P"."PROD_CATEGORY"='Electronics')
11 - filter("S2"."PROD_ID"=:B1)
```

### 使用嵌套循环时的转换查询（`push_subq`提示生效）

```
| Id  | Operation                                       | Name                 | Cost (%CPU)|
|   0 | SELECT STATEMENT                                |                      |  6166   (1)|
|*  1 |  HASH JOIN                                      |                      |  5726   (1)|
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED           | CUSTOMERS            |    10   (0)|
|   3 |    BITMAP CONVERSION TO ROWIDS                  |                      |            |
|*  4 |     BITMAP INDEX SINGLE VALUE                   | CUSTOMERS_YOB_BIX    |            |
|   5 |   NESTED LOOPS                                  |                      |  5716   (1)|
|   6 |    TABLE ACCESS BY INDEX ROWID BATCHED          | PRODUCTS             |     3   (0)|
|*  7 |     INDEX RANGE SCAN                            | PRODUCTS_PROD_CAT_IX |     1   (0)|
|   8 |    PARTITION RANGE ALL                          |                      |  5716   (1)|
|*  9 |     TABLE ACCESS BY LOCAL INDEX ROWID BATCHED   | SALES                |  5716   (1)|
|  10 |      BITMAP CONVERSION TO ROWIDS                |                      |            |
|* 11 |       BITMAP INDEX SINGLE VALUE                 | SALES_PROD_BIX       |            |
|  12 |      SORT AGGREGATE                             |                      |            |
|  13 |       PARTITION RANGE ALL                       |                      |   440   (0)|
|  14 |        TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES                |   440   (0)|
|  15 |         BITMAP CONVERSION TO ROWIDS             |                      |            |
|* 16 |          BITMAP INDEX SINGLE VALUE              | SALES_PROD_BIX       |            |
```

谓词信息（由操作 ID 标识）：

```
1 - access("S1"."CUST_ID"="C"."CUST_ID")
4 - access("C"."CUST_YEAR_OF_BIRTH"=1919)
7 - access("P"."PROD_CATEGORY"='Electronics')
9 - filter("S1"."AMOUNT_SOLD"> (SELECT /*+ PUSH_SUBQ */ AVG("S2"."AMOUNT_SOLD")
              FROM "SH"."SALES" "S2" WHERE "S2"."PROD_ID"=:B1))
11 - access("S1"."PROD_ID"="P"."PROD_ID")
16 - access("S2"."PROD_ID"=:B1)
```

## 子查询推入分析

`Listing 13-25`无法在与`SH.PRODUCTS`的连接中使用`USING`子句，因为需要在子查询中使用表别名来限定引用。`Listing 13-25`中查询的执行计划使用哈希连接来访问`SH.SALES`表，因此无法提前评估子查询。非法提示所做的只是防止子查询解嵌套。然而，如果我们应用`LEADING`和`USE_NL`提示来强制使用嵌套循环连接，那么将子查询推入到与`SH.CUSTOMERS`连接之前进行评估现在是完全合法的（尽管不建议这样做）。第 9 行的表访问现在有两个子节点。第一个子节点为位图索引操作返回的每一行获取`ROWID`。理论上，操作 9 的第二个子节点会对每个`ROWID`进行评估，但子查询缓存确保子查询只被评估了 13 次，电子类别中的 13 种产品各一次。如果子查询评估被推迟到与`SH.CUSTOMERS`连接之后，子查询将只被评估 4 次，因为 1919 年出生的客户只购买了 13 种电子产品中的 4 种。

## 连接谓词下推

当视图合并不可取或不可能时，连接谓词下推（`JPPD`）转换是一个备选方案。请看`Listing 13-26`。

### Listing 13-26. 连接谓词下推

```
WITH agg_q
     AS (  SELECT /*+   push_pred */
                  /* no_push_pred */
                  s.cust_id
                 ,prod_id
                 ,p.prod_name
                 ,SUM (s.amount_sold) total_amt_sold
             FROM sh.sales s JOIN sh.products p USING (prod_id)
         GROUP BY s.cust_id, prod_id)
SELECT cust_id
      ,c.cust_first_name
      ,c.cust_last_name
      ,c.cust_email
      ,agg_q.total_amt_sold
  FROM agg_q RIGHT JOIN sh.customers c USING (cust_id)
 WHERE cust_first_name = 'Abner' AND cust_last_name = 'Everett';
```

### 未转换的执行计划 (`NO_PUSH_PRED`)

```
| Id  | Operation                 | Name        | Cost (%CPU)|
|   0 | SELECT STATEMENT          |             |  5519   (1)|
|*  1 |  HASH JOIN OUTER          |             |  5519   (1)|
|*  2 |   TABLE ACCESS FULL       | CUSTOMERS   |   423   (1)|
|   3 |   VIEW                    |             |  5095   (1)|
|   4 |    HASH GROUP BY          |             |  5095   (1)|
|*  5 |     HASH JOIN             |             |  3021   (1)|
|   6 |      INDEX FULL SCAN      | PRODUCTS_PK |     1   (0)|
|   7 |      VIEW                 | VW_GBC_6    |  3019   (1)|
|   8 |       HASH GROUP BY       |             |  3019   (1)|
|   9 |        PARTITION RANGE ALL|             |   517   (2)|
|  10 |         TABLE ACCESS FULL | SALES       |   517   (2)|
```

谓词信息（由操作 ID 标识）：

```
1 - access("C"."CUST_ID"="AGG_Q"."CUST_ID"(+))
2 - filter("C"."CUST_FIRST_NAME"='Abner' AND
              "C"."CUST_LAST_NAME"='Everett')
5 - access("ITEM_1"="P"."PROD_ID")
```

### 转换后的查询

```
SELECT c.cust_id
      ,agg_q.prod_id
      ,agg_q.prod_name
      ,c.cust_first_name
      ,c.cust_last_name
      ,c.cust_email
      ,agg_q.total_amt_sold
  FROM sh.customers c
       OUTER APPLY
       (  SELECT prod_id, p.prod_name, SUM (s.amount_sold) total_amt_sold
            FROM sh.sales s JOIN sh.products p USING (prod_id)
           WHERE s.cust_id = c.cust_id
        GROUP BY prod_id, p.prod_name) agg_q
 WHERE cust_first_name = 'Abner' AND cust_last_name = 'Everett';
```

### 转换后的执行计划（默认）

```
| Id  | Operation                                        | Name           | Cost (%CPU)|
|   0 | SELECT STATEMENT                                 |                |   480   (0)|
|   1 |  NESTED LOOPS OUTER                              |                |   480   (0)|
|*  2 |   TABLE ACCESS FULL                              | CUSTOMERS      |   423   (1)|
|   3 |   VIEW PUSHED PREDICATE                          |                |    58   (0)|
|   4 |    SORT GROUP BY                                 |                |    58   (0)|
|*  5 |     HASH JOIN                                    |                |    58   (0)|
|   6 |      VIEW                                        | VW_GBC_6       |    55   (0)|
|   7 |       SORT GROUP BY                              |                |    55   (0)|
|   8 |        PARTITION RANGE ALL                       |                |    55   (0)|
|   9 |         TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |    55   (0)|
|  10 |          BITMAP CONVERSION TO ROWIDS             |                |            |
|* 11 |           BITMAP INDEX SINGLE VALUE              | SALES_CUST_BIX |            |
|  12 |      TABLE ACCESS FULL                           | PRODUCTS       |     3   (0)|
```

谓词信息（由操作 ID 标识）：

```
2 - filter("C"."CUST_FIRST_NAME"='Abner' AND "C"."CUST_LAST_NAME"='Everett')
5 - access("ITEM_1"="P"."PROD_ID")
11 - access("S"."CUST_ID"="C"."CUST_ID")
```



清单 13-26 中的查询展示了向名为 Abner Everett 的 14 位客户销售的每种产品的总销售额。外连接的使用排除了简单视图合并和复杂视图合并的可能性，因此，如果没有 JPPD，`GROUP BY` 操作将为每个客户和产品的组合生成 328,395 行数据。JPPD 转换要求从 `SH.CUSTOMERS` 到视图进行嵌套循环连接，以便在每次评估子查询时都能应用 `CUST_ID` 上的谓词。请注意，清单 13-26 中的两个执行计划变体都显示了应用了分组下推转换；`SH.SALES` 中的行在与产品表连接之前，先按 `CUST_ID` 进行了分组。

根据优化器团队的博客文章，[`blogs.oracle.com/optimizer/entry/basics_of_join_predicate_pushdown_in_oracle`](https://blogs.oracle.com/optimizer/entry/basics_of_join_predicate_pushdown_in_oracle)，JPPD 仅支持以下类型的视图：

*   `UNION ALL/UNION` 视图
*   外连接视图
*   反连接视图
*   半连接视图
*   `DISTINCT` 视图
*   `GROUP-BY` 视图

为什么 JPPD 要以这种方式受限呢？嗯，如果这些条件都不满足，那么启发式的简单视图合并转换几乎肯定会应用，从而使 JPPD 的限制变得无关紧要。不过，我将在第 19 章中介绍一个晦涩的案例，在该案例中既不支持 JPPD，也不支持简单视图合并。

JPPD 是一种基于成本的转换，可以通过执行计划中出现的 `VIEW PUSHED PREDICATE` 或 `UNION ALL PUSHED PREDICATE` 操作轻松识别。在早期阶段进行过滤的好处，是以必须多次执行子查询为代价的。如果多次评估子查询的成本过高，CBO 可能会选择不应用该转换，然后使用哈希连接，这样子查询只需评估一次。

尽管 JPPD 已存在很长时间，但直到 12c SQL 语法出现，我们才有能力展示转换的结果。清单 13-26 中的转换查询使用了 ANSI 语法变体进行外部 LATERAL 连接，其效果应与连接谓词下推转换完全相同。然而，在 12.1.0.1 中，似乎存在一个 bug，即使提供了提示，也会阻止 CBO 为 LATERAL 连接生成最优的执行计划。

## 子查询去关联

JPPD 和子查询去关联是彼此互逆的转换，就像复杂视图合并和分组下推是彼此互逆的转换一样。子查询去关联在清单 13-27 中展示。

清单 13-27. 子查询去关联

```
SELECT o.order_id
      ,o.order_date
      ,o.order_mode
      ,o.customer_id
      ,o.order_status
      ,o.order_total
      ,o.sales_rep_id
      ,o.promotion_id
      ,agg_q.max_quantity
  FROM oe.orders o
       CROSS APPLY(SELECT /*+   decorrelate */
                           /* no_decorrelate */
                          MAX (oi.quantity) max_quantity
                     FROM oe.order_items oi
                    WHERE oi.order_id = o.order_id) agg_q
 WHERE o.order_id IN (2458, 2397);

-- 未转换的执行计划 (NO_DECORRELATE)

| Id  | Operation                              | Name            | Cost (%CPU)|

|   0 | SELECT STATEMENT                       |                 |     8   (0)|
|   1 |  NESTED LOOPS                          |                 |     8   (0)|
|   2 |   INLIST ITERATOR                      |                 |            |
|   3 |    TABLE ACCESS BY INDEX ROWID         | ORDERS          |     2   (0)|
|*  4 |     INDEX UNIQUE SCAN                  | ORDER_PK        |     1   (0)|
|   5 |   VIEW                                 | VW_LAT_535DE542 |     3   (0)|
|   6 |    SORT AGGREGATE                      |                 |            |
|   7 |     TABLE ACCESS BY INDEX ROWID BATCHED| ORDER_ITEMS     |     3   (0)|
|*  8 |      INDEX RANGE SCAN                  | ITEM_ORDER_IX   |     1   (0)|

Predicate Information (identified by operation id):

4 - access("O"."ORDER_ID"=2397 OR "O"."ORDER_ID"=2458)
   8 - access("OI"."ORDER_ID"="O"."ORDER_ID")

-- 转换后的查询

SELECT o.order_id order_id
        ,o.order_date order_date
        ,o.order_mode order_mode
        ,o.customer_id customer_id
        ,o.order_status order_status
        ,o.order_total order_total
        ,o.sales_rep_id sales_rep_id
        ,o.promotion_id promotion_id
        ,MAX (oi.quantity) max_quantity
    FROM oe.orders o
         LEFT JOIN oe.order_items oi
            ON     oi.order_id = o.order_id
               AND (oi.order_id = 2397 OR oi.order_id = 2458)
   WHERE (o.order_id = 2397 OR o.order_id = 2458)
GROUP BY o.order_id
        ,o.ROWID
        ,o.promotion_id
        ,o.sales_rep_id
        ,o.order_total
        ,o.order_status
        ,o.customer_id
        ,o.order_mode
        ,o.order_date
        ,o.order_id;

-- 转换后的执行计划（默认）

| Id  | Operation                      | Name        | Cost (%CPU)|

|   0 | SELECT STATEMENT               |             |     5   (0)|
|   1 |  HASH GROUP BY                 |             |     5   (0)|
|*  2 |   HASH JOIN OUTER              |             |     5   (0)|
|   3 |    INLIST ITERATOR             |             |            |
|   4 |     TABLE ACCESS BY INDEX ROWID| ORDERS      |     2   (0)|
|*  5 |      INDEX UNIQUE SCAN         | ORDER_PK    |     1   (0)|
|*  6 |    TABLE ACCESS FULL           | ORDER_ITEMS |     3   (0)|

Predicate Information (identified by operation id):

2 - access("OI"."ORDER_ID"(+)="O"."ORDER_ID")
   5 - access("O"."ORDER_ID"=2397 OR "O"."ORDER_ID"=2458)
   6 - filter("OI"."ORDER_ID"(+)=2397 OR "OI"."ORDER_ID"(+)=2458)
```

我本可以使用清单 13-26 中的转换查询来演示子查询去关联，但需要相当复杂的提示来防止 JPPD 应用于去关联后的结果！我没有那样做，而是使用了 `OE.ORDERS` 和 `OE.ORDER_ITEMS` 表的 LATERAL 连接来演示一些额外的要点。

清单 13-27 列出了 `OE.ORDERS` 中来自订单 2397 和 2458 的两行，以及匹配订单在 `OE.ORDER_ITEMS` 中的 `QUANTITY` 最大值。

转换后的查询现在可以使用哈希连接，这是普通人不太可能想到的。以下是一些值得注意的有趣点：

*   尽管这是一个内 LATERAL 连接 `(CROSS APPLY)` 而不是外 LATERAL 连接 `(OUTER APPLY)`，但转换后的查询仍然使用了外连接。这是因为即使订单没有任何行项目，子查询也保证返回恰好一行。
*   `GROUP BY` 子句包含了 `O.ROWID`。在这种情况下没有必要这样做，因为 `GROUP BY` 列表中已经包含了主键，但通常需要这样做以确保不会对连接左侧的行进行聚合；聚合只适用于右侧，这里是 `OE.ORDER_ITEMS`。
*   该转换成功地应用了传递闭包，从而允许对 `OE.ORDERS` 和 `OE.ORDER_ITEMS` 分别进行过滤。

