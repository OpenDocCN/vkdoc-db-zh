# 表扩展与星型转换

```
CREATE TABLE t
PARTITION BY RANGE
   (d)
   (
      PARTITION
         t_q1_2013 VALUES LESS THAN (TO_DATE ('2013-04-01', 'yyyy-mm-dd'))
     ,PARTITION
         t_q2_2013 VALUES LESS THAN (TO_DATE ('2013-07-01', 'yyyy-mm-dd'))
     ,PARTITION
         t_q3_2013 VALUES LESS THAN (TO_DATE ('2013-10-01', 'yyyy-mm-dd'))
     ,PARTITION
         t_q4_2013 VALUES LESS THAN (TO_DATE ('2014-01-01', 'yyyy-mm-dd')))
PCTFREE 99
PCTUSED 1
AS
       SELECT DATE '2013-01-01' + ROWNUM - 1 d, ROWNUM n
         FROM DUAL
   CONNECT BY LEVEL <= 365;

CREATE INDEX i
   ON t (n)
   LOCAL
   UNUSABLE;

ALTER INDEX i
   REBUILD PARTITION t_q3_2013;
```

现在我们已经有了分区表，清单 13-34 可以展示表扩展转换如何工作。

清单 13-34. 表扩展转换

```
SELECT /*+     expand_table(t) */
       /*  no_expand_table(t) */
      COUNT (*)
 FROM t
WHERE n = 8;

-- 未转换的执行计划 (NO_EXPAND_TABLE)

| Id  | Operation            | Name | Cost (%CPU)|

|   0 | SELECT STATEMENT     |      |    45   (0)|
|   1 |  SORT AGGREGATE      |      |            |
|   2 |   PARTITION RANGE ALL|      |    45   (0)|
|*  3 |    TABLE ACCESS FULL | T    |    45   (0)|

Predicate Information (identified by operation id):

3 - filter("N"=8)

-- 转换后的查询

WITH vw_te
     AS (SELECT *
           FROM t PARTITION (t_q3_2013)
          WHERE n = 8
         UNION ALL
         SELECT *
           FROM t
          WHERE     n = 8
                AND (   d < TO_DATE ('2013-07-01', 'yyyy-mm-dd')
                     OR d >=TO_DATE ('2013-10-01', 'yyyy-mm-dd')))
SELECT COUNT (*)
  FROM vw_te;

-- 转换后的执行计划 (默认)

| Id  | Operation                 | Name    | Cost (%CPU)|

|   0 | SELECT STATEMENT          |         |    34   (0)|
|   1 |  SORT AGGREGATE           |         |            |
|   2 |   VIEW                    | VW_TE_1 |    34   (0)|
|   3 |    UNION-ALL              |         |            |
|   4 |     PARTITION RANGE SINGLE|         |     1   (0)|
|*  5 |      INDEX RANGE SCAN     | I       |     1   (0)|
|   6 |     PARTITION RANGE OR    |         |    33   (0)|
|*  7 |      TABLE ACCESS FULL    | T       |    33   (0)|

Predicate Information (identified by operation id):

5 - access("N"=8)
   7 - filter("N"=8 AND ("T"."D"<TO_DATE(' 2013-07-01 00:00:00',
              'syyyy-mm-dd hh24:mi:ss') OR "T"."D">=TO_DATE(' 2013-10-01 00:00:00',
              'syyyy-mm-dd hh24:mi:ss') AND "T"."D"<TO_DATE(' 2014-01-01 00:00:00',
              'syyyy-mm-dd hh24:mi:ss')))
```

如果没有表扩展转换，清单 13-34 中的查询将无法利用本地索引，因为它的某些分区是不可用的。应用表扩展转换将查询拆分为 `UNION-ALL` 操作的两个分支。第一个分支针对具有可用本地索引的分区，第二个分支则针对只能使用全表扫描的分区。

这种转换很可能有益于许多定期重建本地位图索引分区的数据仓库应用。

表扩展转换是我们介绍的最后一个杂项转换，但我把最好的留到了最后。著名的星型转换可以说是自成一类。让我们以解释这个最著名的优化器转换的作用及其如何帮助我们作为本章的结尾。

## 星型转换

星型转换是最难理解的优化器转换，因此我们将仔细梳理。我们将从考虑 CBO 面临的问题开始，然后探讨如何解决它。

### 分布式连接过滤问题

让我们从查看清单 13-35 开始，并思考如果我们是 CBO 会怎么做。

清单 13-35. 用于星型转换的候选查询

```
SELECT prod_id
      ,cust_id
      ,p.prod_name
      ,c.cust_first_name
      ,s.time_id
      ,s.amount_sold
  FROM sh.sales s
       JOIN sh.customers c USING (cust_id)
       JOIN sh.products p USING (prod_id)
 WHERE c.cust_last_name = 'Everett' AND p.prod_category = 'Electronics';
```

让我们看一些统计数据：
*   `SH.SALES` 表中有 918,843 行。
*   这 918,843 行中有 116,267 行匹配产品过滤器。
*   这 918,843 行中有 740 行匹配客户过滤器。
*   有 115 行同时匹配客户和产品过滤器。

那么，我们选择什么连接顺序呢？
*   如果我们从 `SH.SALES` 表与 `SH.PRODUCTS` 或 `SH.CUSTOMERS` 开始连接，最终会从 `SH.SALES` 读取过多的行；从 `SH.SALES` 读取的行中超过 80% 随后会被丢弃。
*   如果我们首先连接 `SH.PRODUCTS` 和 `SH.CUSTOMERS`，最终会得到 5,760 行，即使使用多列索引开销也很高。

这是一个 *分布式过滤器* 的示例，其中两个或多个过滤条件组合起来的选择性远高于任何单个过滤器。

这个问题与我们在第 10 章讨论的 `INDEX_COMBINE` 和 `AND_EQUAL` 提示所解决的问题非常相似；我们希望识别同时匹配我们所有过滤器的行，然后只读取那些匹配所有条件的行。不同之处在于，我们的过滤条件源自其他表。

这就是我们的问题陈述。现在让我们将注意力转向如何解决它。

### 解决分布式连接过滤问题

清单 13-36 是对清单 13-35 的一次重写尝试。

清单 13-36. 解决分布式连接过滤的首次尝试

```
ALTER SESSION SET star_transformation_enabled=temp_disable;

SELECT /* no_table_lookup_by_nl(s) */
      s.time_id, s.amount_sold
  FROM sh.sales s
 WHERE     s.cust_id IN (SELECT c.cust_id
                           FROM sh.customers c
                          WHERE cust_last_name = 'Everett')
       AND s.prod_id IN (SELECT p.prod_id
                           FROM sh.products p
                          WHERE prod_category = 'Electronics');

| Id  | Operation                      | Name                 | Cost (%CPU)|
```


|   0 | SELECT 语句                       |                      |   923   (1)|
|   1 |  视图                            | VW_ST_8230C7ED       |   923   (1)|
|   2 |   嵌套循环                       |                      |   498   (1)|
|   3 |    分区范围-全部                 |                      |   440   (1)|
|   4 |     位图转换为 ROWID              |                      |   440   (1)|
|   5 |      位图与操作                  |                      |            |
|   6 |       位图合并                   |                      |            |
|   7 |        位图键迭代                |                      |            |
|   8 |         缓冲区排序               |                      |            |
|*  9 |          全表扫描               | CUSTOMERS            |   423   (1)|
|* 10 |         位图索引范围扫描         | SALES_CUST_BIX       |            |
|  11 |       位图合并                   |                      |            |
|  12 |        位图键迭代                |                      |            |
|  13 |         缓冲区排序               |                      |            |
|* 14 |          视图                    | index$_join$_051     |     2   (0)|
|* 15 |           哈希连接               |                      |            |
|* 16 |            索引范围扫描          | PRODUCTS_PROD_CAT_IX |     1   (0)|
|  17 |            索引快速全扫描        | PRODUCTS_PK          |     1   (0)|
|* 18 |         位图索引范围扫描         | SALES_PROD_BIX       |            |
|  19 |    通过用户 ROWID 访问表           | SALES                |   483   (1)|

谓词信息（通过操作 ID 识别）：
```
  9 - 过滤("CUST_LAST_NAME"='Everett')
 10 - 访问("S"."CUST_ID"="C"."CUST_ID")
 14 - 过滤("PROD_CATEGORY"='Electronics')
 15 - 访问(ROWID=ROWID)
 16 - 访问("PROD_CATEGORY"='Electronics')
 18 - 访问("S"."PROD_ID"="P"."PROD_ID")
```

请注意，由于`CUST_ID`在`SH.CUSTOMERS`表中是唯一的，且`PROD_ID`在`SH.PRODUCTS`表中是唯一的，因此清单 13-35 和清单 13-36 返回的行数应该相同；如果`SH.CUSTOMERS`表中相同的`CUST_ID`对应多行，那么清单 13-36 返回的行数将少于清单 13-35。

这个重写后的查询无法使用索引合并，因为正如我在第 10 章中提到的，该技术不支持`IN`列表。清单 13-36 中的执行计划展示了星形转换，这种替代方法提供了一个高效的执行计划。

执行计划显示，已使用`SH.PRODUCTS`表上的索引连接来获取`PROD_ID`的值列表，并且这些值已被用于在`SH.SALES`表的`SALES_PROD_BIX`索引中进行多次查找，以获取多个位图。这些位图是通过第 12 行的`BITMAP KEY ITERATION`操作获得的。`BITMAP KEY ITERATION`的使用是星形转换的一个标志：每次你看到`BITMAP KEY ITERATION`，就意味着发生了星形转换，反之亦然。⁴ 随着这些位图的获取，它们在第 11 行被合并，生成一个单一的位图，标识出所有来自电子产品类别的产品的`SH.SALES`表中的行。

类似的过程应用于`CUST_ID`的值，使用的是`SALES_CUST_BIX`索引。第 6 行的位图标识了对 14 位名叫 Abner Everett 的客户的销售记录。第 6 行和第 11 行生成的位图被输入到第 5 行的`BITMAP AND`操作中，生成一个单一的位图，标识出同时满足两个谓词的所有行。既然我们有了最终的 ROWID 列表，我们就可以访问`SH.SALES`表，仅获取我们需要的行。

![image](img/sq.jpg)  **注意**  `SH.SALES`表是通过嵌套循环连接和一个`TABLE ACCESS BY USER ROWID`操作访问的。这个方法实际上是 11gR2 引入的另一个转换的结果，可以通过在粗体显示的注释中添加一个加号来禁用它。如果禁用了通过嵌套循环进行表查找的转换，第 4 行的`BITMAP CONVERSION TO ROWIDS`操作将成为`TABLE ACCESS BY LOCAL INDEX ROWID BATCHED`操作的子操作，嵌套循环连接就会消失。我不知道通过嵌套循环进行表查找的转换能提供什么好处。

请注意，要应用星形转换，你必须使用企业版，并且对于未加提示的查询，初始化参数`STAR_TRANSFORMATION_ENABLED`必须设置为`TRUE`或`TEMP_DISABLE`（我稍后会解释两者的区别）。

但是清单 13-36 中的查询与清单 13-35 中的查询并不相同：我们失去了选择维度列的能力。我们可以通过将维度表重新添加回我们的表列表中来解决这个问题，如清单 13-37 所示。

清单 13-37. 通过星形转换添加维度列

```
ALTER SESSION SET star_transformation_enabled=temp_disable;

SELECT prod_id
      ,cust_id
      ,prod_name
      ,cust_first_name
      ,time_id
      ,amount_sold
  FROM sh.sales s
       JOIN sh.products USING (prod_id)
       JOIN sh.customers USING (cust_id)
 WHERE     cust_id IN (SELECT c.cust_id
                         FROM sh.customers c
                        WHERE cust_last_name = 'Everett')
       AND prod_id IN (SELECT p.prod_id
                         FROM sh.products p
                        WHERE prod_category = 'Electronics');

| Id  | 操作                            | 名称                 | 成本 (%CPU)|
```


| 标识 | 操作 | 名称 | 成本 (%CPU) |
|---|---|---|---|
| 0 | SELECT STATEMENT | | 1348 (1) |
|* 1 | HASH JOIN | | 1348 (1) |
| 2 | TABLE ACCESS BY INDEX ROWID BATCHED | PRODUCTS | 3 (0) |
|* 3 | INDEX RANGE SCAN | PRODUCTS_PROD_CAT_IX | 1 (0) |
|* 4 | HASH JOIN | | 1345 (1) |
|* 5 | TABLE ACCESS FULL | CUSTOMERS | 423 (1) |
| 6 | VIEW | VW_ST_C72FA945 | 923 (1) |
| 7 | NESTED LOOPS | | 498 (1) |
| 8 | PARTITION RANGE ALL | | 440 (1) |
| 9 | BITMAP CONVERSION TO ROWIDS | | 440 (1) |
| 10 | BITMAP AND | | |
| 11 | BITMAP MERGE | | |
| 12 | BITMAP KEY ITERATION | | |
| 13 | BUFFER SORT | | |
|* 14 | TABLE ACCESS FULL | CUSTOMERS | 423 (1) |
|* 15 | BITMAP INDEX RANGE SCAN | SALES_CUST_BIX | |
| 16 | BITMAP MERGE | | |
| 17 | BITMAP KEY ITERATION | | |
| 18 | BUFFER SORT | | |
|* 19 | VIEW | index$_join$_055 | 2 (0) |
|* 20 | HASH JOIN | | |
|* 21 | INDEX RANGE SCAN | PRODUCTS_PROD_CAT_IX | 1 (0) |
| 22 | INDEX FAST FULL SCAN | PRODUCTS_PK | 1 (0) |
|* 23 | BITMAP INDEX RANGE SCAN | SALES_PROD_BIX | |
| 24 | TABLE ACCESS BY USER ROWID | SALES | 483 (1) |

谓词信息（通过操作标识识别）：
1 - 访问(`"ITEM_1"="P"."PROD_ID"`)
3 - 访问(`"PROD_CATEGORY"='Electronics'`)
4 - 访问(`"ITEM_2"="C"."CUST_ID"`)
5 - 过滤(`"CUST_LAST_NAME"='Everett'`)
14 - 过滤(`"CUST_LAST_NAME"='Everett'`)
15 - 访问(`"S"."CUST_ID"="C"."CUST_ID"`)
19 - 过滤(`"PROD_CATEGORY"='Electronics'`)
20 - 访问(`ROWID=ROWID`)
21 - 访问(`"PROD_CATEGORY"='Electronics'`)
23 - 访问(`"S"."PROD_ID"="P"."PROD_ID"`)

当 `STAR_TRANSFORMATION_ENABLED` 设置为 `TEMP_DISABLE` 时，代码清单 13-37 中显示的执行计划由此产生。

你会注意到代码清单 13-37 中的第 6 到 24 行与代码清单 13-36 中的第 1 到 19 行相同，但这次我们再次访问了 `SH.PRODUCTS` 和 `SH.CUSTOMERS` 表各一次，以获取维度列。尽管我们访问了维度表各两次，但我们避免了从 `SH.SALES` 表中进行的大部分昂贵的表提取操作。

虽然这个查询开始看起来有点难以阅读，但它仍然不是最优的：为什么我们必须访问维度表两次？难道我们不能在第一次读取 `SH.PRODUCTS` 和 `SH.CUSTOMERS` 表时就缓存我们 SELECT 列表中的列吗？

假设我们运行了代码清单 13-38 所示的脚本。

代码清单 13-38. 手动创建临时表的星型转换

```sql
CREATE GLOBAL TEMPORARY TABLE cust_cache ON COMMIT PRESERVE ROWS
AS
   SELECT cust_id, cust_first_name
     FROM sh.customers c
    WHERE cust_last_name = 'Everett';

CREATE GLOBAL TEMPORARY TABLE prod_cache ON COMMIT PRESERVE ROWS
AS
   SELECT prod_id, prod_name
     FROM sh.products p
    WHERE prod_category = 'Electronics';

SELECT s.prod_id
      ,s.cust_id
      ,p.prod_name
      ,c.cust_first_name
      ,s.time_id
      ,s.amount_sold
  FROM sh.sales s
       JOIN cust_cache c ON s.cust_id = c.cust_id
       JOIN prod_cache p ON s.prod_id = p.prod_id
 WHERE     s.prod_id IN (SELECT prod_id FROM prod_cache)
       AND s.cust_id IN (SELECT cust_id FROM cust_cache);
```

| 标识 | 操作 | 名称 | 成本 (%CPU) |
|---|---|---|---|
| 0 | SELECT STATEMENT | | 358 (0) |
|* 1 | HASH JOIN | | 358 (0) |
| 2 | TABLE ACCESS FULL | CUST_CACHE | 2 (0) |
|* 3 | HASH JOIN | | 356 (0) |
| 4 | TABLE ACCESS FULL | PROD_CACHE | 2 (0) |
| 5 | VIEW | VW_ST_456F1C80 | 354 (0) |
| 6 | NESTED LOOPS | | 350 (0) |
| 7 | PARTITION RANGE ALL | | 58 (0) |
| 8 | BITMAP CONVERSION TO ROWIDS | | 58 (0) |
| 9 | BITMAP AND | | |
| 10 | BITMAP MERGE | | |
| 11 | BITMAP KEY ITERATION | | |
| 12 | BUFFER SORT | | |
| 13 | TABLE ACCESS FULL | PROD_CACHE | 2 (0) |
|* 14 | BITMAP INDEX RANGE SCAN | SALES_PROD_BIX | |
| 15 | BITMAP MERGE | | |
| 16 | BITMAP KEY ITERATION | | |
| 17 | BUFFER SORT | | |
| 18 | TABLE ACCESS FULL | CUST_CACHE | 2 (0) |
|* 19 | BITMAP INDEX RANGE SCAN | SALES_CUST_BIX | |
| 20 | TABLE ACCESS BY USER ROWID | SALES | 296 (0) |

谓词信息（通过操作标识识别）：
1 - 访问(`"ITEM_1"="C"."CUST_ID"`)
3 - 访问(`"ITEM_2"="P"."PROD_ID"`)
14 - 访问(`"S"."PROD_ID"="P"."PROD_ID"`)
19 - 访问(`"S"."CUST_ID"="C"."CUST_ID"`)

你可能会好奇为什么我创建了真实的临时表，而不是使用因子化子查询。嗯，我使用的逻辑流程实际上与 CBO 真正使用的逻辑有很大不同，而且截至 12cR1，CBO 似乎还无法处理这样的因子化子查询；在这个例子中，CBO 甚至被 ANSI 连接的 `USING` 子句搞糊涂了。

注意第 12 和 17 行的 `BUFFER SORT` 操作。看起来 `BITMAP KEY ITERATION` 操作要求输入是排序的。当输入是索引时（到目前为止的情况就是这样），这会自动发生；我们本可以通过将 `CUST_CACHE` 和 `PROD_CACHE` 表组织为索引组织表来避免排序，但我有点跑题了。

好消息是，在实际应用中，你不需要创建临时表或做其他任何花哨的事情。如果你将 `STAR_TRANSFORMATION_ENABLED` 设置为 `TRUE`（而不是 `TEMP_DISABLE`），CBO 会为你解决所有这些问题。代码清单 13-39 显示了当 `STAR_TRANSFORMATION_ENABLED` 设置为 `TRUE` 时，代码清单 13-35 的执行计划。

代码清单 13-39. star_transformation_enabled=true 时的最终星型转换

```
| 标识 | 操作 | 名称 | 成本 (%CPU) |
```



## 执行计划分析

```
|   0 | SELECT STATEMENT                       |                           |   509   (1)|
|   1 |  TEMP TABLE TRANSFORMATION             |                           |            |
|   2 |   LOAD AS SELECT                       | SYS_TEMP_0FD9D6747_4E5E5D |            |
|*  3 |    TABLE ACCESS FULL                   | CUSTOMERS                 |   423   (1)|
|*  4 |   HASH JOIN                            |                           |    87   (2)|
|   5 |    TABLE ACCESS FULL                   | SYS_TEMP_0FD9D6747_4E5E5D |     2   (0)|
|*  6 |    HASH JOIN                           |                           |    85   (2)|
|   7 |     TABLE ACCESS BY INDEX ROWID BATCHED| PRODUCTS                  |     3   (0)|
|*  8 |      INDEX RANGE SCAN                  | PRODUCTS_PROD_CAT_IX      |     1   (0)|
|   9 |     VIEW                               | VW_ST_B49D23E2            |    82   (2)|
|  10 |      NESTED LOOPS                      |                           |    78   (2)|
|  11 |       PARTITION RANGE ALL              |                           |    19   (0)|
|  12 |        BITMAP CONVERSION TO ROWIDS     |                           |    19   (0)|
|  13 |         BITMAP AND                     |                           |            |
|  14 |          BITMAP MERGE                  |                           |            |
|  15 |           BITMAP KEY ITERATION         |                           |            |
|  16 |            BUFFER SORT                 |                           |            |
|  17 |             TABLE ACCESS FULL          | SYS_TEMP_0FD9D6747_4E5E5D |     2   (0)|
|* 18 |            BITMAP INDEX RANGE SCAN     | SALES_CUST_BIX            |            |
|  19 |          BITMAP MERGE                  |                           |            |
|  20 |           BITMAP KEY ITERATION         |                           |            |
|  21 |            BUFFER SORT                 |                           |            |
|* 22 |             VIEW                       | index$_join$_052          |     2   (0)|
|* 23 |              HASH JOIN                 |                           |            |
|* 24 |               INDEX RANGE SCAN         | PRODUCTS_PROD_CAT_IX      |     1   (0)|
|  25 |               INDEX FAST FULL SCAN     | PRODUCTS_PK               |     1   (0)|
|* 26 |            BITMAP INDEX RANGE SCAN     | SALES_PROD_BIX            |            |
|  27 |       TABLE ACCESS BY USER ROWID       | SALES                     |    62   (0)|
```

谓词信息（由操作 ID 标识）：

```
3 - filter("C"."CUST_LAST_NAME"='Everett')
   4 - access("ITEM_1"="C0")
   6 - access("ITEM_2"="P"."PROD_ID")
   8 - access("P"."PROD_CATEGORY"='Electronics')
  18 - access("S"."CUST_ID"="C0")
  22 - filter("P"."PROD_CATEGORY"='Electronics')
  23 - access(ROWID=ROWID)
  24 - access("P"."PROD_CATEGORY"='Electronics')
  26 - access("S"."PROD_ID"="P"."PROD_ID")
```

Listing 13-39 表明，即使 CBO 无法处理你编写的因子化子查询，它也能自行生成并妥善使用它们！

让我们逐步分析。在第 2 行和第 3 行，我们扫描`customers`表，找到匹配的行，并将`CUST_FIRST_NAME`和`CUST_ID`列存储在一个临时表中。第 9 行到第 27 行与其他我们见过的执行计划非常相似，唯一的例外是在第 17 行我们使用了临时表来获取`CUST_ID`。同样，我们在第 5 行使用了同一个临时表来为选择列表获取`CUST_FIRST_NAME`。

请注意，没有为`SH.PRODUCTS`生成临时表；CBO 判断使用索引连接来获取`PROD_ID`（第 22 至 25 行），然后重复索引查找以获取`PROD_NAME`（第 7 行和第 8 行）是更好的方案。这很可能是一次在成本方面非常接近的权衡。

关于星型转换有很多奇怪之处。《数据仓库指南》中记录了各种限制，例如在使用绑定变量时它们不起作用。然而，有一项记录在案的限制已被证明不实。事实证明，在没有位图索引的情况下也可以使用星型转换：Oracle 可以将 B 树索引转换为位图索引并加以利用。

星型转换是本章要讨论的最后一个优化器转换，但在结束之前，我想让你快速瞥眼可能即将到来的新技术。

## 未来展望

本章迄今为止是本书中最长的一章，涵盖了超过两打优化器转换。然而，你可以期待更多内容的到来。2009 年，VLDB 基金会发表了一篇由 CBO 开发团队撰写的学术论文，题为“Oracle 中的增强型子查询优化”，该文展示了未来可能发展的方向。请看一下 Listing 13-40 中的两个查询及相关执行计划。

Listing 13-40. 子查询合并的未来

```sql
-- 原始查询

SELECT prod_id
      ,p1.prod_name
      ,p1.prod_desc
      ,p1.prod_category
      ,p1.prod_list_price
  FROM sh.products p1
 WHERE     EXISTS
              (SELECT 1
                 FROM sh.products p2
                WHERE     p1.prod_category = p2.prod_category
                      AND p1.prod_id <> p2.prod_id)
       AND NOT EXISTS
                  (SELECT 1
                     FROM sh.products p3
                    WHERE     p1.prod_category = p3.prod_category
                          AND p1.prod_id <> p3.prod_id
                          AND p3.prod_list_price > p1.prod_list_price);
```

```
| Id  | Operation               | Name                 | Cost (%CPU)|
|   0 | SELECT STATEMENT        |                      |     8   (0)|
|*  1 |  HASH JOIN RIGHT SEMI   |                      |     8   (0)|
|   2 |   VIEW                  | index$_join$_002     |     2   (0)|
|*  3 |    HASH JOIN            |                      |            |
|   4 |     INDEX FAST FULL SCAN| PRODUCTS_PK          |     1   (0)|
|   5 |     INDEX FAST FULL SCAN| PRODUCTS_PROD_CAT_IX |     1   (0)|
|*  6 |   HASH JOIN ANTI        |                      |     6   (0)|
|   7 |    TABLE ACCESS FULL    | PRODUCTS             |     3   (0)|
|   8 |    TABLE ACCESS FULL    | PRODUCTS             |     3   (0)|
```

谓词信息（由操作 ID 标识）：

```
1 - access("P1"."PROD_CATEGORY"="P2"."PROD_CATEGORY")
       filter("P1"."PROD_ID"<>"P2"."PROD_ID")
   3 - access(ROWID=ROWID)
   6 - access("P1"."PROD_CATEGORY"="P3"."PROD_CATEGORY")
       filter("P3"."PROD_LIST_PRICE">"P1"."PROD_LIST_PRICE" AND
              "P1"."PROD_ID"<>"P3"."PROD_ID")
```

```sql
-- 未来的一种转换形式

SELECT p1.prod_id
        ,p1.prod_name
        ,p1.prod_desc
        ,prod_category
        ,p1.prod_list_price
    FROM sh.products p1 JOIN sh.products p2 USING (prod_category)
   WHERE p1.prod_id <> p2.prod_id
GROUP BY p1.prod_id
        ,p1.prod_name
        ,p1.prod_desc
        ,prod_category
        ,p1.prod_list_price
  HAVING SUM (
            CASE
               WHEN p2.prod_list_price > p1.prod_list_price THEN 1
               ELSE 0
            END) = 0;
```

```
| Id  | Operation            | Name     | Cost (%CPU)|
|   0 | SELECT STATEMENT     |          |     6   (0)|
|*  1 |  FILTER              |          |            |
|   2 |   HASH GROUP BY      |          |     6   (0)|
|*  3 |    HASH JOIN         |          |     6   (0)|
|   4 |     TABLE ACCESS FULL| PRODUCTS |     3   (0)|
|   5 |     TABLE ACCESS FULL| PRODUCTS |     3   (0)|
```

谓词信息（由操作 ID 标识）：

```
1 - filter(SUM(CASE  WHEN "P2"."PROD_LIST_PRICE">"P1"."PROD_LIST_PRIC
              E" THEN 1 ELSE 0 END )=0)
   3 - access("P1"."PROD_CATEGORY"="P2"."PROD_CATEGORY")
       filter("P1"."PROD_ID"<>"P2"."PROD_ID"
```



代码清单 13-40 中的两个查询版本都识别出了在其所属类别中具有最高标价、但又不自成一类的产品。代码清单 13-40 中的后一个查询，既合并（coalesced）了两个子查询，又将它们解除了嵌套（unnested）。我必须得说，我盯着已发布的查询看了好几分钟⁵，才确信它们在语义上是等价的⁶，并且这样的转换确实能带来实际益处。

## 总结

基于成本的优化器（CBO）拥有大量的优化器转换手段。本章涵盖了其中超过两 dozen 种转换，尽管在写作时这份列表可能（虽然不太可能）是全面的，但等到本书付印时它很可能就不再全面了。然而，学习如何识别这些优化器转换，并在必要时影响其选择，对于调优复杂查询至关重要。

一项优化器转换可能是基于成本框架的一部分，并会基于计算出的成本收益被选择应用。然而，有几种优化器转换是启发式的，意味着它们是否被应用取决于某些固定规则。这意味着可能会应用一项导致预估成本增加的转换。当然，如果查询的实际执行时间减少了，增加的预估成本可能就无关紧要了。

本章应该清楚说明的一点是，人类要找出构建 SQL 语句的正确方式，以便仅通过最终状态优化就能推导出最优执行计划，有时几乎需要超凡的 SQL 技能。请记住，尽管我们都应该为所有这些优化器转换的存在而感到高兴，但我们不能固步自封，祈求 CBO 能将我们编写的任何糟糕代码转换为好代码；真正糟糕的代码仍然会迷惑 CBO。

尽管如此，既然我们已经理解了第 12 章中介绍的最终状态优化过程的威力与灵活性，以及 CBO 所具备的转换的广度和复杂性，你可能会疑惑为什么市场上还有那么多 SQL 调优专家。事实是，在现实世界中，事情总会出错——而且一直在出错。让我们在第 14 章中接下来看看这个问题。

__________________

¹我有理由相信在 `12.1.0.1` 版本中可能确实存在一个 bug。
²对复杂视图合并的限制阻止了在代码清单 13-17 中进行逆向转换。
³偶尔，除非使用提示，否则即使被引用多次，`INDEX FAST FULL SCAN` 也可能不会被物化。
⁴实际上，Jonathan Lewis 给我看了一个使用了 `SEMIJOIN_DRIVER` 提示的反例，但如果没有这种恶意提示，CBO 永远不会生成这样的计划。
⁵那篇学术论文没有使用 `SH` 示例模式，但我已经制作了一个等效的例子。
⁶在纠正了看起来像印刷错误的问题后，它们是等价的。

