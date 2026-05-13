# Oracle CBO 优化器转换与提示

CBO（基于成本的优化器）能够连续执行多次转换，这一事实使得理解其意图变得有些困难。好消息是，除了那些“理所当然”的转换外，与 CBO 所选转换相关联的提示，会出现在`DBMS_XPLAN`函数生成的执行计划的概述部分，正如我在第 8 章中所描述的那样。然而，你永远不会在显示的执行计划概述部分看到像`NO_MERGE`或`NO_EXPAND`这样的提示，因为`OUTLINE`和`OUTLINE_LEAF`提示的存在意味着，缺少像`MERGE`或`USE_CONCAT`这样的提示就表明相应的转换并未被应用。

本文在前文中不得不多次特别提及优化器的“理所当然”转换。现在让我们来认识它们。

## 理所当然的转换

我确信你不会惊讶地发现，Oracle 官方并未将任何 CBO 转换称为*理所当然的*（*no-brainers*）；这只是我自己创造的一个术语，用来指代一组具有特殊性质的转换。这些转换是如此基础，以至于它们没有相关联的提示，因此既不能被强制启用，也不能被禁用。事实上，甚至连`NO_QUERY_TRANSFORMATION`提示也无法禁用这些转换。这些转换是如此隐形，以至于只有两种方式可以知道它们的存在：

*   你可以仔细研读 10053 跟踪文件。Christian Antognini 就是这么做的。
*   你可以阅读 Christian 的著作*《Troubleshooting Oracle Performance》*。我就是这么做的！

讨论几个这样的“理所当然”转换是有价值的，但我不会试图穷尽它们。让我们从计数转换开始。

## 计数转换

看一下 Listing 13-1 中展示的两个查询及其相关的执行计划。

**Listing 13-1. 计数转换**

```sql
SELECT COUNT (cust_income_level) FROM sh.customers;

| Id  | Operation          | Name      |

|   0 | SELECT STATEMENT   |           |
|   1 |  SORT AGGREGATE    |           |
|   2 |   TABLE ACCESS FULL| CUSTOMERS |

SELECT COUNT (cust_id) FROM sh.customers;

| Id  | Operation                     | Name                 |

|   0 | SELECT STATEMENT              |                      |
|   1 |  SORT AGGREGATE               |                      |
|   2 |   BITMAP CONVERSION COUNT     |                      |
|   3 |    BITMAP INDEX FAST FULL SCAN| CUSTOMERS_GENDER_BIX |
```

两个查询的执行计划和结果都不同。事实证明，`SH.CUSTOMERS`表中的`CUST_INCOME_LEVEL`列没有`NOT NULL`约束，因此该列可能为`NULL`（某些行确实为`NULL`）。这些行在 Listing 13-1 的第一个查询中不会被计数。由于`CUST_INCOME_LEVEL`没有索引，执行`COUNT (CUST_INCOME_LEVEL)`的唯一方法就是进行全表扫描。然而，`CUST_ID`是`NOT NULL`的，因此`COUNT (CUST_ID)`可以被转换为`COUNT (*)`。我们可以通过全表扫描来实现`COUNT (*)`，但我们也可以通过位图索引或在`NOT NULL`列上的 B 树索引来实现`COUNT (*)`。在第二个查询的情况下，索引`CUSTOMERS_GENDER_BIX`是一个建立在非空列上的位图索引，因此它是全表扫描的一个双重合适、廉价的替代方案。

感到无聊了吗？下一个转换更有趣一些，但也仅此而已。

## 谓词移动

考虑 Listing 13-2 中的查询和执行计划。

**Listing 13-2. 谓词移动**

```sql
WITH cust_q
     AS (  SELECT cust_id, promo_id, SUM (amount_sold) cas
             FROM sh.sales
         GROUP BY cust_id, promo_id)
    ,prod_q
     AS (  SELECT prod_id, promo_id, SUM (amount_sold) pas
             FROM sh.sales
            WHERE promo_id = 999
         GROUP BY prod_id, promo_id)
SELECT promo_id
      ,prod_id
      ,pas
      ,cust_id
      ,cas
  FROM cust_q NATURAL JOIN prod_q;

| Id  | Operation              | Name  |

|   0 | SELECT STATEMENT       |       |
|*  1 |  HASH JOIN             |       |
|   2 |   VIEW                 |       |
|   3 |    HASH GROUP BY       |       |
|   4 |     PARTITION RANGE ALL|       |
|*  5 |      TABLE ACCESS FULL | SALES |
|   6 |   VIEW                 |       |
|   7 |    HASH GROUP BY       |       |
|   8 |     PARTITION RANGE ALL|       |
|*  9 |      TABLE ACCESS FULL | SALES |

Predicate Information (identified by operation id):

1 - access("CUST_Q"."PROMO_ID"="PROD_Q"."PROMO_ID")
   5 - filter("PROMO_ID"=999)
   9 - filter("PROMO_ID"=999)
```

Listing 13-2 中的查询没有做任何有意义的事情。对于产品的每种组合，它列出了客户针对`PROMO_ID` 999 的总销售额，以及产品针对`PROMO_ID` 999 的总销售额。这里有两个因子化子查询和一个`WHERE`子句，但自然连接意味着通过传递闭包，我们可以将谓词也应用到另一个子查询上。然而，Listing 13-3 使查询稍微复杂了一些，从而导致谓词移动转换不再起作用。

**Listing 13-3. 谓词移动未被识别**

```sql
WITH cust_q
     AS (  SELECT cust_id
                 ,promo_id
                 ,SUM (amount_sold) cas
                 ,MAX (SUM (amount_sold)) OVER (PARTITION BY promo_id) max_cust
             FROM sh.sales
         GROUP BY cust_id, promo_id)
    ,prod_q
     AS (  SELECT prod_id
                 ,promo_id
                 ,SUM (amount_sold) pas
                 ,MAX (SUM (amount_sold)) OVER (PARTITION BY promo_id) max_prod
             FROM sh.sales
            WHERE promo_id = 999
         GROUP BY prod_id, promo_id)
SELECT promo_id
      ,prod_id
      ,pas
      ,cust_id
      ,cas
  FROM cust_q NATURAL JOIN prod_q
 WHERE cust_q.cas = cust_q.max_cust AND prod_q.pas = prod_q.max_prod;

| Id  | Operation                | Name  |

|   0 | SELECT STATEMENT         |       |
|   1 |  MERGE JOIN              |       |
|*  2 |   VIEW                   |       |
|   3 |    WINDOW BUFFER         |       |
|   4 |     SORT GROUP BY        |       |
|   5 |      PARTITION RANGE ALL |       |
|   6 |       TABLE ACCESS FULL  | SALES |
|*  7 |   SORT JOIN              |       |
|*  8 |    VIEW                  |       |
|   9 |     WINDOW BUFFER        |       |
|  10 |      SORT GROUP BY       |       |
|  11 |       PARTITION RANGE ALL|       |
|* 12 |        TABLE ACCESS FULL | SALES |

Predicate Information (identified by operation id):

2 - filter("CUST_Q"."CAS"="CUST_Q"."MAX_CUST")
   7 - access("CUST_Q"."PROMO_ID"="PROD_Q"."PROMO_ID")
       filter("CUST_Q"."PROMO_ID"="PROD_Q"."PROMO_ID")
   8 - filter("PROD_Q"."PAS"="PROD_Q"."MAX_PROD")
  12 - filter("PROMO_ID"=999)
```

Listing 13-3 做的是一件更有意义的事：它识别出针对`PROMO_ID` 999 卖得最多的产品，以及针对`PROMO_ID` 999 买得最多的客户。但是，在子查询中添加分析函数导致 CBO 对谓词移动的正确性失去了信心。为了使查询高效，你必须自己将`WHERE`子句添加到`CUST_Q`因子化子查询中。

还有一两个其他的“理所当然”转换：


*   `过滤器下推`是谓词移动的一种更简单的变体。
*   `去重消除`会从查询中移除冗余的`DISTINCT`操作，当操作数涉及主键的所有列、所有`NOT NULL`的唯一键列或`ROWID`时。
*   `选择列表修剪`会从子查询的选择列表中移除未被引用的项。

这些转换的关键点在于，应用它们没有可能的负面影响。例如，通过从子查询的选择列表中移除未引用的项，不可能使查询变慢。所有这些都有一些价值，但不太大，所以让我们继续看第二组转换：集合和连接转换。

## 集合与连接转换

存在几种与集合和连接操作相关的转换，其中大部分都是最近才实现的，这令人惊讶，因为它们中的许多都非常有用。让我们从连接消除开始。

## 连接消除

连接消除是那种可以通过内联视图和因子化子查询来演示的转换之一，但它实际上是为通用数据字典视图设计的。我将在清单 13-4 中演示这种对数据字典视图的关注，但我将在未来的示例中假定这一点已被理解。

**清单 13-4. 对数据字典视图进行连接消除**

```sql
CREATE OR REPLACE VIEW cust_sales
AS
   WITH sales_q
        AS (  SELECT cust_id
                    ,SUM (amount_sold) amount_sold
                    ,AVG (amount_sold) avg_sold
                    ,COUNT (*) cnt
                FROM sh.sales s
            GROUP BY cust_id)
   SELECT c.*
         ,s.cust_id AS sales_cust_id
         ,s.amount_sold
         ,s.avg_sold
         ,cnt
     FROM sh.customers c JOIN sales_q s ON c.cust_id = s.cust_id;

SELECT sales_cust_id
      ,amount_sold
      ,avg_sold
      ,cnt
  FROM cust_sales;

| Id  | Operation              | Name         |
|---|---|---|
|   0 | SELECT STATEMENT       |              |
|   1 |  NESTED LOOPS          |              |
|   2 |   VIEW                 |              |
|   3 |    HASH GROUP BY       |              |
|   4 |     PARTITION RANGE ALL|              |
|   5 |      TABLE ACCESS FULL | SALES        |
|*  6 |   INDEX UNIQUE SCAN    | CUSTOMERS_PK |

CREATE OR REPLACE VIEW cust_sales
AS
   WITH sales_q
        AS (  SELECT cust_id
                    ,SUM (amount_sold) amount_sold
                    ,AVG (amount_sold) avg_sold
                    ,COUNT (*) cnt
                FROM sh.sales s
            GROUP BY cust_id)
   SELECT c.*
         ,s.cust_id AS sales_cust_id
         ,s.amount_sold
         ,s.avg_sold
         ,cnt
     FROM sh.customers c RIGHT JOIN sales_q s ON c.cust_id = s.cust_id;

SELECT /*+   eliminate_join(@SEL$13BD1B6A c@sel$2) */
       /* no_eliminate_join(@SEL$13BD1B6A c@sel$2) */sales_cust_id
      ,amount_sold
      ,avg_sold
      ,cnt
  FROM cust_sales;

-- 未转换的执行计划 (NO_ELIMINATE_JOIN)

| Id  | Operation              | Name         |
|---|---|---|
|   0 | SELECT STATEMENT       |              |
|   1 |  NESTED LOOPS OUTER    |              |
|   2 |   VIEW                 |              |
|   3 |    HASH GROUP BY       |              |
|   4 |     PARTITION RANGE ALL|              |
|   5 |      TABLE ACCESS FULL | SALES        |
|*  6 |   INDEX UNIQUE SCAN    | CUSTOMERS_PK |

-- 转换后的查询

SELECT cust_id AS sales_cust_id
        ,SUM (amount_sold) amount_sold
        ,AVG (amount_sold) avg_sold
        ,COUNT (*) cnt
    FROM sh.sales s
GROUP BY cust_id;

-- 转换后的执行计划 (默认)

| Id  | Operation            | Name  |
|---|---|---|
|   0 | SELECT STATEMENT     |       |
|   1 |  HASH GROUP BY       |       |
|   2 |   PARTITION RANGE ALL|       |
|   3 |    TABLE ACCESS FULL | SALES |
```

清单 13-4 首先创建了数据字典视图 `CUST_SALES`，该视图通过在 `SH.SALES` 表上执行聚合，向 `SH.CUSTOMERS` 示例模式表添加了一些列。然后查询使用了 `CUST_SALES` 视图，但只对来自 `SH.SALES` 表的列感兴趣。不幸的是，聚合操作导致 CBO 无法追踪确保 `SH.SALES` 中的所有行在 `SH.CUSTOMERS` 中恰好有一行对应行这一引用完整性约束的重要性。结果，CBO 没有应用连接消除优化。

清单 13-4 继续重新定义 `CUST_SALES` 视图以使用外连接。这向 CBO 确认了聚合的 `SH.SALES` 数据与 `SH.CUSTOMERS` 的连接不会导致任何行丢失，并且连接列 `CUST_ID` 是 `SH.CUSTOMERS` 的主键这一事实也意味着连接不会导致添加任何行。CBO 现在确信，消除从 `SH.SALES` 到 `SH.CUSTOMERS` 的连接不会影响结果，并继续进行转换。

清单 13-4 中的第二个查询包含两个注释。第一个是强制默认行为的提示。如果删除第一个注释并在第二个注释前加一个加号，你可以禁用该转换。不幸的是，因为连接发生在视图内部，你必须查看 `DBMS_XPLAN.DISPLAY` 的大纲部分来识别查询块，并生成一个全局提示，如果你真的想禁用该转换。清单 13-4 也显示了与未转换查询、转换后的查询以及转换后执行计划相关的执行计划。

请仔细查看清单 13-4，因为我将在介绍其他转换时使用这种方法。

连接消除还有另一个有趣的应用，适用于`嵌套表`。由于嵌套表不提供任何性能或管理优势，它们在 21 世纪设计的模式中很少见。但以防你需要处理 20 世纪的模式，请查看清单 13-5，它引用了 `PM.PRINT_MEDIA` 示例模式。

**清单 13-5. 对嵌套表进行连接消除**

```sql
 SELECT /*+   eliminate_join(parent_tab) */
         /* no_eliminate_join(parent_tab) */
         nested_tab.document_typ, COUNT (*) cnt
    FROM pm.print_media parent_tab
        ,TABLE (parent_tab.ad_textdocs_ntab)nested_tab
GROUP BY nested_tab.document_typ;

-- 未转换的执行计划 (NO_ELIMINATE_JOIN)

| Id  | Operation           | Name               |
|---|---|---|
|   0 | SELECT STATEMENT    |                    |
|   1 |  HASH GROUP BY      |                    |
|   2 |   NESTED LOOPS      |                    |
|   3 |    TABLE ACCESS FULL| TEXTDOCS_NESTEDTAB |
|*  4 |    INDEX UNIQUE SCAN| SYS_C0011792       |

-- 转换后的执行计划 (默认)

| Id  | Operation                            | Name                    |
|---|---|---|
|   0 | SELECT STATEMENT                     |                         |
|   1 |  HASH GROUP BY                       |                         |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED| TEXTDOCS_NESTEDTAB      |
|*  3 |    INDEX FULL SCAN                   | SYS_FK0000091775N00007$ |
```

清单 13-5 有趣的地方在于，在 10gR2 版本引入连接消除转换之前，引用嵌套表而不引用其父表的可能性是不存在的。没有有效的 SQL 语法来表示连接消除转换的结果！



你可能认为连接消除（join elimination）应该是 `no-brainer`（显而易见的事）：保留连接永远不会有任何性能益处。然而，我怀疑用于控制转换的提示（hints）的创建现在已成为一种标准，如果是这样，我表示赞同。在 `DBMS_XPLAN` 函数的概要部分看到提示，意味着你可以更容易地弄清楚发生了什么，并允许你出于教育或研究目的禁用该转换。

## 外连接转内连接

有时外连接无法被消除，但可以被转换为内连接。让我们再看一下 清单 13-4 中 `CUST_SALES` 视图的第二个定义。尽管对视图定义的更改帮助了那些只需要聚合 `SH.SALES` 数据的查询，但它使得那些确实需要 `SH.CUSTOMERS` 列的查询效率降低。如果我们更改这些查询，就可以消除那个开销。看看 清单 13-6。

**清单 13-6. 外连接转内连接转换**

```
SELECT /*+   outer_join_to_inner(@SEL$13BD1B6A c@sel$2) */
       /* no_outer_join_to_inner(@SEL$13BD1B6A c@sel$2) */
      *
 FROM cust_sales c
WHERE cust_id IS NOT NULL;

-- 未转换的执行计划 (NO_OUTER_JOIN_TO_INNER)

| Id  | Operation               | Name      |

|   0 | SELECT STATEMENT        |           |
|*  1 |  FILTER                 |           |
|*  2 |   HASH JOIN OUTER       |           |
|   3 |    VIEW                 |           |
|   4 |     HASH GROUP BY       |           |
|   5 |      PARTITION RANGE ALL|           |
|   6 |       TABLE ACCESS FULL | SALES     |
|   7 |    TABLE ACCESS FULL    | CUSTOMERS |

Predicate Information (identified by operation id):

1 - filter("C"."CUST_ID" IS NOT NULL)
   2 - access("C"."CUST_ID"(+)="S"."CUST_ID")

-- 转换后的查询

WITH sales_q
        AS (  SELECT cust_id
                    ,SUM (amount_sold) amount_sold
                    ,AVG (amount_sold) avg_sold
                    ,COUNT (*) cnt
                FROM sh.sales s
            GROUP BY cust_id)
   SELECT c.*
         ,s.cust_id AS sales_cust_id
         ,s.amount_sold
         ,s.avg_sold
         ,cnt
     FROM sh.customers c JOIN sales_q s ON c.cust_id = s.cust_id;

-- 转换后的执行计划 (默认)

| Id  | Operation              | Name      |

|   0 | SELECT STATEMENT       |           |
|*  1 |  HASH JOIN             |           |
|   2 |   VIEW                 |           |
|   3 |    HASH GROUP BY       |           |
|   4 |     PARTITION RANGE ALL|           |
|   5 |      TABLE ACCESS FULL | SALES     |
|   6 |   TABLE ACCESS FULL    | CUSTOMERS |

Predicate Information (identified by operation id):

1 - access("C"."CUST_ID"="S"."CUST_ID")
```

暂且不谈参照完整性约束，`SH.CUSTOMERS` 上的外连接与内连接之间的潜在区别在于，可能会添加 `C.CUST_ID` 为 `NULL` 的额外行。存在 `WHERE cust_id IS NOT NULL` 这个谓词意味着任何此类额外行都会被消除，因此 CBO 将应用从外连接到内连接的转换，除非被提示禁止这样做。

## 全外连接转外连接

外连接转内连接转换的一个逻辑扩展是全外连接转外连接转换。完全相同的原则适用。如果你理解了 清单 13-6，清单 13-7 应该不言自明。

**清单 13-7. 全外连接转外连接**

```
SELECT /*+    full_outer_join_to_outer(cust) */
       /*  no_full_outer_join_to_outer(cust) */
      *
 FROM sh.countries FULL OUTER JOIN sh.customers cust USING (country_id)
WHERE cust.cust_id IS NOT NULL;

-- 未转换的执行计划 (NO_FULL_OUTER_JOIN_TO_OUTER)

| Id  | Operation             | Name      |

|   0 | SELECT STATEMENT      |           |
|*  1 |  VIEW                 | VW_FOJ_0  |
|*  2 |   HASH JOIN FULL OUTER|           |
|   3 |    TABLE ACCESS FULL  | COUNTRIES |
|   4 |    TABLE ACCESS FULL  | CUSTOMERS |

Predicate Information (identified by operation id):

1 - filter("CUST"."CUST_ID" IS NOT NULL)
   2 - access("COUNTRIES"."COUNTRY_ID"="CUST"."COUNTRY_ID")

-- 转换后的查询

SELECT *
  FROM sh.countries RIGHT OUTER JOIN sh.customers cust USING (country_id);

-- 转换后的执行计划 (默认)

| Id  | Operation             | Name      |

|   0 | SELECT STATEMENT      |           |
|*  1 |  HASH JOIN RIGHT OUTER|           |
|   2 |   TABLE ACCESS FULL   | COUNTRIES |
|   3 |   TABLE ACCESS FULL   | CUSTOMERS |

Predicate Information (identified by operation id):

1 - access("COUNTRIES"."COUNTRY_ID"(+)="CUST"."COUNTRY_ID")
```

当然，如果我们在 清单 13-7 中添加一个诸如 `WHERE COUNTRIES.COUNTRY_NAME IS NOT NULL` 的谓词，我们也可以应用外连接转内连接转换，从而得到一个内连接。

## 半连接转内连接

虽然半连接转内连接的转换已经存在一段时间，但用于管理该转换的提示在 `11.2.0.3` 版本中是新的。清单 13-8 进行了演示。

**清单 13-8. 半连接转内连接**

```
SELECT *
  FROM sh.sales s
 WHERE EXISTS
          (SELECT /*+   semi_to_inner(c) */
                  /*  no_semi_to_inner(c) */

FROM sh.customers c
           WHERE     c.cust_id = s.cust_id
                 AND cust_first_name = 'Abner'
                 AND cust_last_name = 'Everett');

-- 未转换的执行计划 (NO_SEMI_TO_INNER)

| Id  | Operation            | Name      |

|   0 | SELECT STATEMENT     |           |
|*  1 |  HASH JOIN RIGHT SEMI|           |
|*  2 |   TABLE ACCESS FULL  | CUSTOMERS |
|   3 |   PARTITION RANGE ALL|           |
|   4 |    TABLE ACCESS FULL | SALES     |

Predicate Information (identified by operation id):

1 - access("C"."CUST_ID"="S"."CUST_ID")
   2 - filter("CUST_FIRST_NAME"='Abner' AND "CUST_LAST_NAME"='Everett')

-- 转换后的查询 (近似)

WITH q1
     AS (SELECT DISTINCT cust_id
           FROM sh.customers
          WHERE cust_first_name = 'Abner' AND cust_last_name = 'Everett')
SELECT s.*
  FROM sh.sales s JOIN q1 ON s.cust_id = q1.cust_id;

-- 转换后的执行计划 (默认)

| Id  | Operation                          | Name           |

|   0 | SELECT STATEMENT                   |                |
|   1 |  NESTED LOOPS                      |                |
|   2 |   NESTED LOOPS                     |                |
|   3 |    SORT UNIQUE                     |                |
|*  4 |     TABLE ACCESS FULL              | CUSTOMERS      |
|   5 |    PARTITION RANGE ALL             |                |
|   6 |     BITMAP CONVERSION TO ROWIDS    |                |
|*  7 |      BITMAP INDEX SINGLE VALUE     | SALES_CUST_BIX |
|   8 |   TABLE ACCESS BY LOCAL INDEX ROWID| SALES          |

Predicate Information (identified by operation id):

4 - filter("CUST_FIRST_NAME"='Abner' AND "CUST_LAST_NAME"='Everett')
   7 - access("C"."CUST_ID"="S"."CUST_ID")
```

清单 13-8 中显示的转换为嵌套循环和合并半连接提供了额外的连接顺序。在此转换创建之前，半连接要求未嵌套的子查询是嵌套循环或合并半连接的探测行源（对于哈希连接，连接输入可以交换）。现在我们可以将半连接转换为内连接，因此可以使用子查询作为驱动行源。



注意在转换后的执行计划中有一个 `SORT UNIQUE` 操作。此操作的目的是确保 `SH.SALES` 中的任何行在结果集中最多只出现一次。但是，如果你获取转换后的查询并对*那个*查询运行 `DBMS_XPLAN.DISPLAY`，你将不会看到这个 `SORT UNIQUE` 操作。这是因为如果直接执行或解释转换后的查询，就会执行无脑的重复消除转换。我推测，由该转换引入的冗余 `SORT UNIQUE` 操作将在未来的版本中消失。

## 子查询脱钩

我们已经在第 11 章的半连接和反连接上下文中看到了子查询脱钩转换。我们还将在第 14 章中探讨在选择列表中的子查询脱钩。然而，这里我还想讨论一两种其他变体的子查询脱钩。请看清单 13-9。

### 清单 13-9. 使用脱钩的子查询脱钩

```sql
SELECT c.cust_first_name, c.cust_last_name, s1.amount_sold
  FROM sh.customers c, sh.sales s1
 WHERE     s1.amount_sold = (SELECT /*     unnest */
                                    /*+ no_unnest */
                                    MAX (amount_sold)
                               FROM sh.sales s2
                              WHERE s1.cust_id= s2.cust_id)
       AND c.cust_id = s1.cust_id;
```

-- 未转换的执行计划 (NO_UNNEST)

| Id  | Operation                                    | Name           |

|   0 | SELECT STATEMENT                             |                |
|*  1 |  FILTER                                      |                |
|*  2 |   HASH JOIN                                  |                |
|   3 |    TABLE ACCESS FULL                         | CUSTOMERS      |
|   4 |    PARTITION RANGE ALL                       |                |
|   5 |     TABLE ACCESS FULL                        | SALES          |
|   6 |   SORT AGGREGATE                             |                |
|   7 |    PARTITION RANGE ALL                       |                |
|   8 |     TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |
|   9 |      BITMAP CONVERSION TO ROWIDS             |                |
|* 10 |       BITMAP INDEX SINGLE VALUE              | SALES_CUST_BIX |

谓词信息 (由操作 ID 标识):

1 - filter("S1"."AMOUNT_SOLD"= (SELECT /*+ NO_UNNEST */
              MAX("AMOUNT_SOLD") FROM "SH"."SALES" "S2" WHERE "S2"."CUST_ID"=:B1))
   2 - access("C"."CUST_ID"="S1"."CUST_ID")
  10 - access("S2"."CUST_ID"=:B1

-- 转换后的查询

```sql
WITH vw_sq_1
     AS (  SELECT cust_id AS sq_cust_id, MAX (amount_sold) max_amount_sold
             FROM sh.sales s2
         GROUP BY cust_id)
SELECT c.cust_first_name, c.cust_last_name, s1.amount_sold
  FROM sh.sales s1, sh.customers c, vw_sq_1
 WHERE     s1.amount_sold = vw_sq_1.max_amount_sold
       AND s1.cust_id = vw_sq_1.sq_cust_id
       AND s1.cust_id = c.cust_id;
```

-- 转换后的执行计划 (默认)

| Id  | Operation                    | Name         |

|   0 | SELECT STATEMENT             |              |
|   1 |  NESTED LOOPS                |              |
|   2 |   NESTED LOOPS               |              |
|*  3 |    HASH JOIN                 |              |
|   4 |     VIEW                     | VW_SQ_1      |
|   5 |      HASH GROUP BY           |              |
|   6 |       PARTITION RANGE ALL    |              |
|   7 |        TABLE ACCESS FULL     | SALES        |
|   8 |     PARTITION RANGE ALL      |              |
|   9 |      TABLE ACCESS FULL       | SALES        |
|* 10 |    INDEX UNIQUE SCAN         | CUSTOMERS_PK |
|  11 |   TABLE ACCESS BY INDEX ROWID| CUSTOMERS    |

谓词信息 (由操作 ID 标识):

3 - access("S1"."AMOUNT_SOLD"="MAX(AMOUNT_SOLD)" AND
              "S1"."CUST_ID"="ITEM_1")
  10 - access("C"."CUST_ID"="S1"."CUST_ID")
```

`WHERE` 子句中的原始子查询是一个*相关*子查询，这意味着在未转换的查询中，该子查询在逻辑上为连接返回的每一行执行一次。然而，在转换后的查询中，聚合只执行一次，为每个 `CUST_ID` 值进行聚合。

子查询脱钩相当不同寻常，因为它有时作为*基于成本的转换*应用，有时作为*启发式转换*应用。如果你使用诸如 `WHERE C1 = (<subquery>)` 的结构，子查询脱钩是基于成本的。然而，在本书的所有清单中，子查询脱钩都是作为启发式转换应用的，这意味着 CBO 不使用基于成本的框架来决定是否应用此转换。与我们迄今为止讨论的所有其他转换（半连接转内连接转换除外）一样，子查询脱钩*总是*执行（除非使用提示），无论预期性能是否会提高。在我们迄今为止讨论的所有优化器转换中，转换都不会损害性能。但在某些情况下，子查询脱钩*确实*可能损害性能。我们将在第 14 章和第 18 章回到启发式转换的潜在性能影响这个话题，但现在只需记下这一点。我们需要关注子查询脱钩的另一个案例，见清单 13-10。

### 清单 13-10. 使用窗口函数的子查询脱钩

```sql
SELECT c.cust_first_name, c.cust_last_name, s1.amount_sold
  FROM sh.customers c, sh.sales s1
 WHERE     amount_sold = (SELECT /*     unnest */
                                 /*+ no_unnest */
MAX (amount_sold)
                            FROM sh.sales s2
                           WHERE c.cust_id= s2.cust_id)
       AND c.cust_id = s1.cust_id;
```

-- 未转换的执行计划 (NO_UNNEST)

| Id  | Operation                                    | Name           |

|   0 | SELECT STATEMENT                             |                |
|*  1 |  FILTER                                      |                |
|*  2 |   HASH JOIN                                  |                |
|   3 |    TABLE ACCESS FULL                         | CUSTOMERS      |
|   4 |    PARTITION RANGE ALL                       |                |
|   5 |     TABLE ACCESS FULL                        | SALES          |
|   6 |   SORT AGGREGATE                             |                |
|   7 |    PARTITION RANGE ALL                       |                |
|   8 |     TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |
|   9 |      BITMAP CONVERSION TO ROWIDS             |                |
|* 10 |       BITMAP INDEX SINGLE VALUE              | SALES_CUST_BIX |

谓词信息 (由操作 ID 标识):

1 - filter("AMOUNT_SOLD"= (SELECT /*+ NO_UNNEST */
              MAX("AMOUNT_SOLD") FROM "SH"."SALES" "S2" WHERE "S2"."CUST_ID"=:B1))
   2 - access("C"."CUST_ID"="S1"."CUST_ID")
  10 - access("S2"."CUST_ID"=:B1)

-- 转换后的查询

```sql
WITH vw_wif_1
     AS (SELECT c.cust_first_name
               ,c.cust_last_name
               ,s.amount_sold
               ,MAX (amount_sold) OVER (PARTITION BY s.cust_id) AS item_4
           FROM sh.customers c, sh.sales s
          WHERE s.cust_id = c.cust_id)
SELECT cust_first_name, cust_last_name, amount_sold
  FROM vw_wif_1
 WHERE CASE WHEN item_4 = amount_sold THEN ROWID END IS NOT NULL;
```

-- 转换后的执行计划 (默认)

| Id  | Operation              | Name      |



|   0 | SELECT 语句             |           |
|---|---|---|
|*  1 |  视图                  | VW_WIF_1  |
|   2 |   窗口排序             |           |
|*  3 |    哈希连接            |           |
|   4 |     表全扫描           | CUSTOMERS |
|   5 |     分区范围全扫描     |           |
|   6 |      表全扫描          | SALES     |

谓词信息（由操作 ID 标识）：
1 - filter(`VW_COL_4` IS NOT NULL)
3 - access(`C`.`CUST_ID`=`S1`.`CUST_ID`)

清单 13-9 和 清单 13-10 中展示的查询之间的差异很难发现。实际上，清单 13-9 中的关联列是 `S1.CUST_ID`，而在 清单 13-10 中是 `C.CUST_ID`。鉴于查询中还有一个等值谓词连接 `S1.CUST_ID` 和 `C.CUST_ID`，清单 13-10 中生成的执行计划与 清单 13-9 中的计划差异如此之大，可能会令人惊讶。这是 *传递闭包* 问题的一个例子，我们将在 第 14 章 再次讨论这个主题。现在，让我们暂且接受，因为关联列 `(C.CUST_ID)` 来自一个表，而子查询又与另一个表 `(S1.AMOUNT_SOLD)` 的列进行了等值关联，所以 清单 13-10 使用了与 清单 13-9 不同的子查询展开变体。我们可以看到，在 清单 13-9 中，我们对 `SH.SALES` 进行了两次全表扫描（不好），但没有排序（好）。在 清单 13-10 中，我们只对 `SH.SALES` 进行了一次扫描（好），但有一次排序（不好）。如果你有类似最后两个清单中的查询，最好尝试修改代码，看看哪种执行计划性能最佳。

清单 13-10 中转换后的查询对连接结果中的所有行进行排序，然后仅选择排名最高的行。为什么使用涉及 `case` 表达式的复杂谓词？我不想猜测。

**部分连接**

部分连接是 12cR1 的一个新特性，但在我看来，它们的解释并不充分。让我看看能否从另一个角度来阐述。请看 清单 13-11。

清单 13-11. 部分连接转换

```
SELECT /*+    partial_join(iv) */
         /*  no_partial_join(iv) */
         product_id, MAX (it.quantity)
    FROM oe.order_items it JOIN oe.inventories iv USING (product_id)
GROUP BY product_id;
```

-- 未转换的执行计划 (NO_PARTIAL_JOIN)

| Id  | Operation              | Name         |
|---|---|---|
|   0 | SELECT STATEMENT       |              |
|   1 |  HASH GROUP BY         |              |
|*  2 |   HASH JOIN            |              |
|   3 |    TABLE ACCESS FULL   | ORDER_ITEMS  |
|   4 |    INDEX FAST FULL SCAN| INVENTORY_IX |

谓词信息（由操作 ID 标识）：
2 - access(`IT`.`PRODUCT_ID`=`IV`.`PRODUCT_ID`)

-- 转换后的查询

```
SELECT product_id, MAX (quantity)
    FROM oe.order_items it
   WHERE EXISTS
            (SELECT 1
               FROM oe.inventories iv
              WHERE it.product_id = iv.product_id)
GROUP BY it.product_id;
```

-- 转换后的执行计划 (默认)

| Id  | Operation              | Name         |
|---|---|---|
|   0 | SELECT STATEMENT       |              |
|   1 |  HASH GROUP BY         |              |
|*  2 |   HASH JOIN SEMI       |              |
|   3 |    TABLE ACCESS FULL   | ORDER_ITEMS  |
|   4 |    INDEX FAST FULL SCAN| INVENTORY_IX |

谓词信息（由操作 ID 标识）：
2 - access(`IT`.`PRODUCT_ID`=`IV`.`PRODUCT_ID`)

如你所见，部分连接将内连接转换为了半连接。但之前我们不是有从半连接到内连接的转换吗？你感到困惑了吗？我当时是。让我来解开这个谜团。

部分连接的好处最好用具有多对多关系的两个表的连接来解释，因此在 清单 13-11 中，我暂时放弃了 `SH` 示例模式，而使用了 `OE` 示例模式中具有多对多关系的两个表。

让我们以 `PRODUCT_ID` 1797 为例，首先看看未转换的查询是如何处理它的。该产品属于三个订单，在 `OE.ORDER_ITEMS` 的三行匹配记录中，`QUANTITY` 的值分别为 7、9 和 12。`PRODUCT_ID` 1797 也存放在三个仓库中，因此在 `OE.INVENTORIES` 中有三行与 `PRODUCT_ID` 1797 匹配。当我们使用 `PRODUCT_ID` 连接 `OE.ORDER_ITEMS` 和 `OE.INVENTORIES` 时，对于 `PRODUCT_ID` 1797，我们得到了九行记录，`QUANTITY` 值分别为 7, 9, 12, 7, 9, 12, 7, 9, 和 12（来自 `OE.ORDER_ITEMS` 的三行被复制了三次）。当我们将这九行输入到 `HASH GROUP BY` 操作时，`MAX (QUANTITY)` 的值是 12。如果我们在选择列表中包含了 `MIN (QUANTITY)`、`SUM (DISTINCT QUANTITY)`、`AVG (QUANTITY)` 和 `SUM (QUANTITY)`，我们得到的结果将分别是 7、28、9⅓ 和 84。

转换为半连接可以防止 `OE.INVENTORIES` 中 `PRODUCT_ID` 1797 的任何重复值导致 `OE.ORDER_ITEMS` 的行被重复，因此输入到 `HASH GROUP BY` 操作的值仅为 7、9 和 12。`MAX (PRODUCT_ID)` 的值仍然是 12，与处理所有九行时完全相同。基于这些行的 `MIN (QUANTITY)`、`SUM (DISTINCT QUANTITY)`、`AVG (QUANTITY)` 和 `SUM (QUANTITY)` 的值将分别为 7、28、9⅓ 和 28。在我所有的示例中，对于转换后查询的三行和未转换查询的九行，聚合值都是相同的，除了 `SUM (QUANTITY)` 产生了不同的结果。因此，如果你尝试在 清单 13-11 的查询的选择列表中添加 `SUM (QUANTITY)`，你会发现部分连接突然变得不合法了。在 12.1.0.1 中，`AVG (QUANTITY)` 函数也会禁用部分连接，我推测这是因为 `COUNT (QUANTITY)` 和 `SUM (QUANTITY)` 都确实不合法。这类问题可能会在后续版本中得到修复。

部分连接的主要好处是减少输入到 `GROUP BY` 或 `DISTINCT` 操作的行数。在转换后执行计划中的 `HASH JOIN SEMI` 操作中，这是通过在找到第一个匹配后立即从哈希集群中移除条目来实现的。从那时起，来自 `OE.INVENTORIES` 的具有相同 `PRODUCT_ID` 的后续行将不再匹配。

部分连接应用于嵌套循环连接时也有一些有趣的副作用。请看 清单 13-12，其中显示了编辑后的运行时统计信息以及执行计划。

清单 13-12. 带嵌套循环连接的部分连接转换

```
BEGIN
   FOR r IN (  SELECT/*+ TAGPJ gather_plan_statistics */
                     cust_id, MAX (s.amount_sold)
                 FROM sh.customers c JOIN sh.sales s USING (cust_id)
                WHERE s.amount_sold > 1782 AND s.prod_id = 18
             GROUP BY cust_id)
   LOOP
      NULL;
   END LOOP;
END;
```

SET LINES 200 PAGES 0

```
SELECT p.*
  FROM v$sql s
      ,TABLE (
          DBMS_XPLAN.display_cursor (s.sql_id
                                    ,s.child_number
                                    ,'BASIC IOSTATS LAST')) p
 WHERE sql_text LIKE 'SELECT /*+ TAGPJ%';
```

| Id  | Operation                    | Name           | Starts | E-Rows | A-Rows |



## 执行计划分析

| 0 | SELECT STATEMENT             |                | 1 |     | 83 |
|---|------------------------------|----------------|---|-----|----|
| 1 | HASH GROUP BY                |                | 1 | 3   | 83 |
| 2 | NESTED LOOPS SEMI            |                | 1 | 3   | 108 |
| 3 | PARTITION RANGE ALL          |                | 1 | 3   | 108 |
|* 4 | TABLE ACCESS BY LOCAL INDEX ROWID | SALES      | 28 | 3   | 108 |
| 5 | BITMAP CONVERSION TO ROWIDS  |                | 16 |    | 9591 |
|* 6 | BITMAP INDEX SINGLE VALUE    | SALES_PROD_BIX | 16 |    | 16 |
|* 7 | INDEX UNIQUE SCAN            | CUSTOMERS_PK   | 83 | 55500 | 83 |

谓词信息（按操作 ID 标识）：

4 - 过滤(`"SALES"."AMOUNT_SOLD"`>1782)
6 - 访问(`"SALES"."PROD_ID"`=18)
7 - 访问(`"CUSTOMERS"."CUST_ID"`=`"SALES"."CUST_ID"`)

乍一看，应用于清单 13-12 中嵌套循环连接的**部分连接**似乎毫无意义。`SH.CUSTOMERS`表的`CUST_ID`列上存在主键约束，因此第 7 行的`INDEX UNIQUE SCAN`只会获取一行，那么将`NESTED LOOPS`转换为`NESTED LOOPS SEMI`有什么好处呢？答案可以通过查看操作 7 的`STARTS`列得出。尽管有 108 行`SH.SALES`数据满足谓词`SH.PROD_ID=18`和`SH.AMOUNT_SOLD > 1782`，但`INDEX UNIQUE SCAN`只执行了 83 次！这是因为在这 108 行中，`CUST_ID`只有 83 个不同的值，而`NESTED LOOPS SEMI`和`NESTED LOOPS ANTI`连接操作可以利用**标量子查询缓存**。常规的`NESTED LOOPS`连接则无法利用这一点！

不过，也许我们有点跑题了。**部分连接转换**的主要目的是尽早消除在流程后期会被无谓地输入到`DISTINCT`或`GROUP BY`操作中的行。这需要将内连接转换为半连接。另一方面，**半连接到内连接转换**的目的是让子查询充当`NESTED LOOPS`或`MERGE`连接操作中的**驱动行源**。这两个目标是相互排斥的。幸运的是，部分连接和半连接到内连接的转换都是**基于成本**的，当 CBO 认为进行部分连接转换会产生不利的连接顺序时，它将不会执行该转换。

## 连接因式分解

**连接因式分解转换**的目的是从`UNION ALL`操作的操作数中移除公共组件，然后在操作之后统一应用它们一次。清单 13-13 提供了一个示例。

### 示例查询

清单 13-13. 连接因式分解

```sql
SELECT /*+   factorize_join(@set$1(s@u1 S@u2)) qb_name(u1) */
       /* no_factorize_join(@set$1) */
       *
  FROM sh.sales s, sh.customers c
 WHERE     c.cust_first_name = 'Abner'
       AND c.cust_last_name = 'Everett'
       AND s.cust_id = c.cust_id
/*AND prod_id = 13
AND time_id = DATE '2001-09-13' */

UNION ALL
SELECT /*+ qb_name(u2) */
       *
  FROM sh.sales s, sh.customers c
 WHERE     c.cust_first_name = 'Abigail'
       AND c.cust_last_name = 'Ruddy'
       AND s.cust_id = c.cust_id
/*AND prod_id = 13
AND time_id = DATE '2001-09-13' */;
```

### 未转换的执行计划 (NO_FACTORIZE_JOIN)

```sql
| Id  | Operation                           | Name           |
|---|---|---|
|   0 | SELECT STATEMENT                    |                |
|   1 |  UNION-ALL                          |                |
|   2 |   NESTED LOOPS                      |                |
|   3 |    NESTED LOOPS                     |                |
|*  4 |     TABLE ACCESS FULL               | CUSTOMERS      |
|   5 |     PARTITION RANGE ALL             |                |
|   6 |      BITMAP CONVERSION TO ROWIDS    |                |
|*  7 |       BITMAP INDEX SINGLE VALUE     | SALES_CUST_BIX |
|   8 |    TABLE ACCESS BY LOCAL INDEX ROWID| SALES          |
|   9 |   NESTED LOOPS                      |                |
|  10 |    NESTED LOOPS                     |                |
|* 11 |     TABLE ACCESS FULL               | CUSTOMERS      |
|  12 |     PARTITION RANGE ALL             |                |
|  13 |      BITMAP CONVERSION TO ROWIDS    |                |
|* 14 |       BITMAP INDEX SINGLE VALUE     | SALES_CUST_BIX |
|  15 |    TABLE ACCESS BY LOCAL INDEX ROWID| SALES          |
```

谓词信息（按操作 ID 标识）：

4 - 过滤(`"C"."CUST_FIRST_NAME"`='Abner' AND
              `"C"."CUST_LAST_NAME"`='Everett')
7 - 访问(`"S"."CUST_ID"`=`"C"."CUST_ID"`)
11 - 过滤(`"C"."CUST_FIRST_NAME"`='Abigail' AND
              `"C"."CUST_LAST_NAME"`='Ruddy')
14 - 访问(`"S"."CUST_ID"`=`"C"."CUST_ID"`)

### 转换后的查询

```sql
WITH vw_jf
     AS (SELECT *
           FROM sh.customers c
          WHERE c.cust_first_name = 'Abner' AND c.cust_last_name = 'Everett'
         UNION ALL
         SELECT *
           FROM sh.customers c
          WHERE c.cust_first_name = 'Abigail' AND c.cust_last_name = 'Ruddy')
SELECT *
  FROM sh.sales s, vw_jf
 WHERE s.cust_id = vw_jf.cust_id;
```

### 转换后的执行计划 (default)

```sql
| Id  | Operation            | Name               |
|---|---|---|
|   0 | SELECT STATEMENT     |                    |
|*  1 |  HASH JOIN           |                    |
|   2 |   VIEW               | VW_JF_SET$F472D255 |
|   3 |    UNION-ALL         |                    |
|*  4 |     TABLE ACCESS FULL| CUSTOMERS          |
|*  5 |     TABLE ACCESS FULL| CUSTOMERS          |
|   6 |   PARTITION RANGE ALL|                    |
|   7 |    TABLE ACCESS FULL | SALES              |
```

谓词信息（按操作 ID 标识）：

6 - 访问(`"TIME_ID"`=TO_DATE(' 2001-09-13 00:00:00', 'syyyy-mm-dd
              hh24:mi:ss'))
7 - 访问(`"PROD_ID"`=13)
10 - 过滤(`"C"."CUST_FIRST_NAME"`='Abner' AND
              `"C"."CUST_LAST_NAME"`='Everett')
11 - 访问(`"C"."CUST_ID"`=`"S"."CUST_ID"`)
12 - 过滤(`"C"."CUST_FIRST_NAME"`='Abigail' AND
              `"C"."CUST_LAST_NAME"`='Ruddy')
13 - 访问(`"C"."CUST_ID"`=`"S"."CUST_ID"`)

清单 13-13 中的原始查询，其两个分支都以相同的方式引用了`SH.SALES`表。转换后的查询提取了两个分支中的连接，并在`UNION ALL`操作之后仅执行一次连接。正如您能想象的，这种转换可能会带来巨大的节省，但在本例中，我们用一次全表扫描替换了两次索引访问，收益很小。实际上，如果您添加注释掉的谓词，执行连接因式分解反而会不利。幸运的是，连接因式分解是一种**基于成本**的转换，您会发现当包含额外谓词时，除非使用提示强制，否则不会应用该转换。

提示连接因式分解很棘手，因为提示适用于一个**集合查询块**。这会产生一系列连锁的复杂问题。



## 转换为连接（Join）

`INTERSECT` 和 `MINUS` 集合操作可以表达为连接操作。清单 13-14 展示了如何实现这一点。

清单 13-14. 集合到连接的转换

```sql
SELECT /*+ set_to_join(@set$1) */ /* no_set_to_join(@set$1) */
       prod_id
  FROM sh.sales s1
 WHERE time_id < DATE '2000-01-01' AND cust_id = 13
MINUS
SELECT prod_id
  FROM sh.sales s2
 WHERE time_id >=DATE '2000-01-01';
```

```
-- 未转换的执行计划（默认）

| Id  | Operation                                    | Name           | Cost (%CPU)|

|   0 | SELECT STATEMENT                             |                |  2396  (99)|
|   1 |  MINUS                                       |                |            |
|   2 |   SORT UNIQUE                                |                |    24   (0)|
|   3 |    PARTITION RANGE ITERATOR                  |                |    24   (0)|
|   4 |     TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |    24   (0)|
|   5 |      BITMAP CONVERSION TO ROWIDS             |                |            |
|*  6 |       BITMAP INDEX SINGLE VALUE              | SALES_CUST_BIX |            |
|   7 |   SORT UNIQUE                                |                |  2372   (1)|
|   8 |    PARTITION RANGE ITERATOR                  |                |   275   (2)|
|   9 |     TABLE ACCESS FULL                        | SALES          |   275   (2)|

Predicate Information (identified by operation id):

6 - access("CUST_ID"=13)
```

```sql
-- 转换后的查询

SELECT DISTINCT s1.prod_id
  FROM sh.sales s1
 WHERE     time_id < DATE '2000-01-01'
       AND cust_id = 13
       AND prod_id NOT IN (SELECT prod_id
                             FROM sh.sales s2
                            WHERE time_id >=DATE '2000-01-01');
```

```
-- 转换后的执行计划（SET_TO_JOIN）

| Id  | Operation                                    | Name           | Cost (%CPU)|

|   0 | SELECT STATEMENT                             |                |   301   (2)|
|   1 |  HASH UNIQUE                                 |                |   301   (2)|
|*  2 |   HASH JOIN ANTI                             |                |   301   (2)|
|   3 |    PARTITION RANGE ITERATOR                  |                |    24   (0)|
|   4 |     TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |    24   (0)|
|   5 |      BITMAP CONVERSION TO ROWIDS             |                |            |
|*  6 |       BITMAP INDEX SINGLE VALUE              | SALES_CUST_BIX |            |
|   7 |    PARTITION RANGE ITERATOR                  |                |   275   (2)|
|   8 |     TABLE ACCESS FULL                        | SALES          |   275   (2)|

Predicate Information (identified by operation id):

2 - access("PROD_ID"="PROD_ID")
   6 - access("CUST_ID"=13)
```

清单 13-14 中的查询用于找出 `CUST_ID` 为 13 的客户在 20 世纪购买、但在 21 世纪未被任何人购买过的产品。未转换的查询在识别结果集元素之前，会对 `UNION ALL` 操作的两个分支分别执行唯一排序；这正是 `MINUS` 和 `INTERSECT` 集合操作的一贯实现方式。应用此转换意味着可以避免其中一个排序操作。但是，看看转换后查询执行计划中的操作 2——`HASH JOIN ANTI` 操作的工作区包含了重复值，因此我们需要反思这个转换是否真的值得。在这个案例中，`CUST_ID` 13 在 20 世纪只进行了 19 次交易，涉及 6 种产品，所以从这个工作区中消除 13 行数据并非大事。然而，该转换避免了对 21 世纪交易相关的 492,064 行 `SH.SALES` 数据中的 72 种产品进行唯一排序。这次转换是个好主意。

尽管成本估算显示 CBO（正确地）认为集合到连接的转换能产生更优的执行计划，但如果没有提示（hint），此转换**不会**被应用到未加提示的查询上，通常情况下**永远不会**被应用！

要理解为什么我们需要在清单 13-14 中使用 `SET_TO_JOIN` 提示，我们需要进一步了解基于成本的转换与启发式转换之间的区别。当 CBO 考虑基于成本的转换时，它会尝试判断该转换是否会减少查询的执行时间；如果转换会减少估计的执行时间——即成本更低——则会应用该转换，否则不应用。启发式转换则通过应用某种规则来工作。到目前为止，我们在本章中遇到的所有启发式转换都是无条件应用的，除非被提示禁用。在这些情况下，其工作假设是：这个提示很可能（即使不是一定）会有帮助。

对于集合到连接的启发式转换，其启发式规则是：除非使用提示，否则**永不**应用该转换，其工作假设是：需要此转换的情况很少。考虑到 CBO 需要在毫秒级时间内生成执行计划，除非你使用提示，否则它不会浪费时间去考虑集合到连接的转换。

如果你喜欢摆弄隐藏的初始化参数，并且不在意这样做的支持影响，你可以修改 `"_convert_set_to_join"` 的值。如果你在会话级别将此参数的值从其默认值 `FALSE` 改为 `TRUE`，那么启发式规则将被更改，使得该转换**总是**被应用。这是唯一一种需要你使用 `NO_SET_TO_JOIN` 提示来禁用该转换的情况。

集合到连接转换为我们关于集合与连接相关转换的讨论画上了一个圆满的句号。现在，我们可以将注意力转向与聚合操作相关的一类新转换。

## 聚合转换

在 Oracle 数据库中，聚合是一个异常复杂的问题，并且在 SQL 调优专家的工作中占据重要地位。我们需要理解需要聚合什么信息、何时聚合以及如何聚合。在最近的版本中，Oracle 引入了许多与聚合相关的转换，它们不仅有助于查询性能，还充当了极佳的教学指南，可帮助你分析包含聚合的查询。让我们从了解如何优化**不同值聚合**开始吧。

### 不同值聚合



我不知道你怎么想，但我一直觉得包含 `DISTINCT` 关键字的聚合函数本质上效率低下，使用它只是有点偷懒。不过现在我不必担心了，因为如今的优化器（CBO）会把我的简洁查询转换成不那么简洁但更高效的形式。请看 清单 13-15。

## 去重聚合转换

```
SELECT /*+   transform_distinct_agg */
       /* no_transform_distinct_agg */
      COUNT (DISTINCT cust_id) FROM sh.sales;

-- 未转换的执行计划 (NO_TRANSFORM_DISTINCT_AGG)

| Id  | Operation                      | Name           | Cost (%CPU)|

|   0 | SELECT STATEMENT               |                |   407   (0)|
|   1 |  SORT GROUP BY                 |                |            |
|   2 |   PARTITION RANGE ALL          |                |   407   (0)|
|   3 |    BITMAP CONVERSION TO ROWIDS |                |   407   (0)|
|   4 |     BITMAP INDEX FAST FULL SCAN| SALES_CUST_BIX |            |

-- 转换后的查询

WITH vw_dag
     AS (  SELECT cust_id
             FROM sh.sales
         GROUP BY cust_id)
SELECT COUNT (cust_id)
  FROM vw_dag;

-- 转换后的执行计划 (默认)

| Id  | Operation                        | Name           | Cost (%CPU)|

|   0 | SELECT STATEMENT                 |                |   428   (5)|
|   1 |  SORT AGGREGATE                  |                |            |
|   2 |   VIEW                           | VW_DAG_0       |   428   (5)|
|   3 |    HASH GROUP BY                 |                |   428   (5)|
|   4 |     PARTITION RANGE ALL          |                |   407   (0)|
|   5 |      BITMAP CONVERSION TO ROWIDS |                |   407   (0)|
|   6 |       BITMAP INDEX FAST FULL SCAN| SALES_CUST_BIX |            |
```

去重聚合转换是在 `11gR2` 版本中引入的，在我看来，这是一种启发式转换，在合法的情况下会被无条件应用。转换后的估算成本比不转换要高，但通过将识别不同值的工作与计数工作分开，查询实际上运行得更快得多。

## 去重放置

去重放置是一种基于成本的转换，从 `11gR2` 版本开始使用，目的是尽早消除重复行。从概念上讲，去重放置与我们将在下一节讨论的分组放置非常相似。清单 13-16 展示了基本思想。

```
SELECT /*+no_partial_join(s1) no_partial_join(s2)     place_distinct(s1) */
       /*  no_partial_join(s1) no_partial_join(s2) no_place_distinct(s1) */
      DISTINCT cust_id, prod_id
 FROM sh.sales s1 JOIN sh.sales s2 USING (cust_id, prod_id)
WHERE s1.time_id < DATE '2000-01-01' AND s2.time_id >=DATE '2000-01-01';

-- 未转换的执行计划 (NO_PLACE_DISTINCT)

| Id  | Operation                  | Name  | Cost (%CPU)|

|   0 | SELECT STATEMENT           |       |  5302   (1)|
|   1 |  HASH UNIQUE               |       |  5302   (1)|
|*  2 |   HASH JOIN                |       |  1787   (1)|
|   3 |    PARTITION RANGE ITERATOR|       |   246   (2)|
|   4 |     TABLE ACCESS FULL      | SALES |   246   (2)|
|   5 |    PARTITION RANGE ITERATOR|       |   275   (2)|
|   6 |     TABLE ACCESS FULL      | SALES |   275   (2)|

谓词信息 (由操作 ID 标识):

2 - access("S1"."PROD_ID"="S2"."PROD_ID" AND
              "S1"."CUST_ID"="S2"."CUST_ID")
-- 转换后的查询

WITH vw_dtp
     AS (SELECT /*+ no_partial_join(s1) no_merge */
                DISTINCT s1.cust_id, s1.prod_id
           FROM sh.sales s1
          WHERE s1.time_id < DATE '2000-01-01')
SELECT /*+ no_partial_join(s2) */
       DISTINCT cust_id, prod_id
  FROM vw_dtp NATURAL JOIN sh.sales s2
 WHERE s2.time_id >=DATE '2000-01-01';

-- 转换后的执行计划 (默认)

| Id  | Operation                    | Name            | Cost (%CPU)|

|   0 | SELECT STATEMENT             |                 |  5595   (1)|
|   1 |  HASH UNIQUE                 |                 |  5595   (1)|
|*  2 |   HASH JOIN                  |                 |  3326   (1)|
|   3 |    VIEW                      | VW_DTP_6DE9D1A7 |  2138   (1)|
|   4 |     HASH UNIQUE              |                 |  2138   (1)|
|   5 |      PARTITION RANGE ITERATOR|                 |   246   (2)|
|   6 |       TABLE ACCESS FULL      | SALES           |   246   (2)|
|   7 |    PARTITION RANGE ITERATOR  |                 |   275   (2)|
|   8 |     TABLE ACCESS FULL        | SALES           |   275   (2)|

谓词信息 (由操作 ID 标识):

2 - access("ITEM_2"="S2"."PROD_ID" AND "ITEM_1"="S2"."CUST_ID")
```

清单 13-16 中的查询查找了在 20 世纪和 21 世纪都出现过的 `cust_id` 和 `prod_id` 的组合。为了简单起见，我禁用了部分连接转换（如果没有 `NO_PARTIAL_JOIN` 提示，该转换会发生）。额外的 `HASH UNIQUE` 操作意味着，在转换后的执行计划中，第 2 行 `HASH JOIN` 操作所使用的工作区仅包含 72 行，而在未转换查询的操作 2 所关联的工作区中，则放置了来自 `SH.SALES` 的 426,779 行 20 世纪的销售记录。成本估算表明这种转换不值得，但不知何故，CBO 无论如何都执行了转换，并且查询速度明显加快。

## 分组放置

分组放置的实现方式和原因与去重放置相同。然而，清单 13-17 突出了一些新的要点。

```
 SELECT /*+   place_group_by((s p)) */
         /* no_place_group_by */
         cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,p.prod_category
        ,SUM (s.amount_sold) total_amt_sold
    FROM sh.sales s
         JOIN sh.customers c USING (cust_id)
         JOIN sh.products p USING (prod_id)
GROUP BY cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,p.prod_category;

-- 未转换的执行计划 (NO_PLACE_GROUP_BY)

| Id  | Operation                | Name                 | Cost (%CPU)|

|   0 | SELECT STATEMENT         |                      | 19957   (1)|
|   1 |  HASH GROUP BY           |                      | 19957   (1)|
|*  2 |   HASH JOIN              |                      |  2236   (1)|
|   3 |    VIEW                  | index$_join$_004     |     2   (0)|
|*  4 |     HASH JOIN            |                      |            |
|   5 |      INDEX FAST FULL SCAN| PRODUCTS_PK          |     1   (0)|
|   6 |      INDEX FAST FULL SCAN| PRODUCTS_PROD_CAT_IX |     1   (0)|
|*  7 |    HASH JOIN             |                      |  2232   (1)|
|   8 |     TABLE ACCESS FULL    | CUSTOMERS            |   423   (1)|
|   9 |     PARTITION RANGE ALL  |                      |   517   (2)|
|  10 |      TABLE ACCESS FULL   | SALES                |   517   (2)|

谓词信息 (由操作 ID 标识):

2 - access("S"."PROD_ID"="P"."PROD_ID")
   4 - access(ROWID=ROWID)
   7 - access("S"."CUST_ID"="C"."CUST_ID")
```


WITH vw_gbc
     AS (  SELECT /*+ no_place_group_by */
                 s.cust_id
                 ,p.prod_category
                 ,SUM (s.amount_sold) total_amt_sold
             FROM sh.sales s JOIN sh.products p USING (prod_id)
         GROUP BY s.cust_id, p.prod_category, prod_id)
  SELECT /*+ no_place_group_by leading(vw_gbc)
            use_hash(c) no_swap_join_inputs(c) */
         cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,vw_gbc.prod_category
        ,SUM (vw_gbc.total_amt_sold) total_amt_sold
    FROM vw_gbc JOIN sh.customers c USING (cust_id)
GROUP BY cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,vw_gbc.prod_category;
```

```
-- 转换后的执行计划（默认）

| Id  | Operation                   | Name                 | Cost (%CPU)|
| --- | --------------------------- | -------------------- | ---------- |
|   0 | SELECT STATEMENT            |                      |  4802   (1)|
|   1 |  HASH GROUP BY              |                      |  4802   (1)|
|*  2 |   HASH JOIN                 |                      |  4319   (1)|
|   3 |    VIEW                     | VW_GBC_1             |  3682   (1)|
|   4 |     HASH GROUP BY           |                      |  3682   (1)|
|*  5 |      HASH JOIN              |                      |   521   (2)|
|   6 |       VIEW                  | index$_join$_004     |     2   (0)|
|*  7 |        HASH JOIN            |                      |            |
|   8 |         INDEX FAST FULL SCAN| PRODUCTS_PK          |     1   (0)|
|   9 |         INDEX FAST FULL SCAN| PRODUCTS_PROD_CAT_IX |     1   (0)|
|  10 |       PARTITION RANGE ALL   |                      |   517   (2)|
|  11 |        TABLE ACCESS FULL    | SALES                |   517   (2)|
|  12 |    TABLE ACCESS FULL        | CUSTOMERS            |   423   (1)|

谓词信息（按操作 ID 标识）：

2 - access("ITEM_1"="C"."CUST_ID")
5 - access("S"."PROD_ID"="P"."PROD_ID")
7 - access(ROWID=ROWID)
```

清单 13-17 中的查询识别了每个客户在每个产品类别上的总销售额。转换后的查询在 `SH.SALES` 与 `SH.PRODUCTS` 连接之后执行聚合，因此第 2 行 `HASH JOIN` 的工作区很小。

分组放置转换的应用似乎存在一些限制¹，其中有些我尚未能完全弄清楚。然而，目前看来，我们确实无法在不对 `SH.SALES` 的行至少进行一次连接的情况下对其进行分组。

在 清单 13-17 中，产生最终结果的连接的两个操作数是：
-   由 `SH.SALES` 和 `SH.PRODUCTS` 连接形成的中间结果集
-   表 `SH.CUSTOMERS`

可以对这两个操作数中的任何一个或两者都执行分组放置。你可以在 清单 13-17 中看到强制在第一个操作数上进行分组的语法。提示 `PLACE_GROUP_BY ((S P) (C))` 将用于对两个操作数都执行分组放置。

清单 13-18 展示了当我们从 清单 13-17 中的转换后查询中移除提示，然后运行 `EXPLAIN PLAN` 时会发生什么。

## 清单 13-18. 子查询中的分组放置

```sql
WITH vw_gbc
     AS (  SELECT /*+   place_group_by((s)) */
                  /* no_place_group_by */
                  s.cust_id, p.prod_category, SUM (s.amount_sold) total_amt_sold
             FROM sh.sales s JOIN sh.products p USING (prod_id)
         GROUP BY s.cust_id, p.prod_category, prod_id)
  SELECT /*+ place_group_by((vw_gbc)) */
         /* no_place_group_by leading(vw_gbc)use_hash(c) no_swap_join_inputs(c) */
         cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,vw_gbc.prod_category
        ,SUM (vw_gbc.total_amt_sold) total_amt_sold
    FROM vw_gbc JOIN sh.customers c USING (cust_id)
GROUP BY cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,vw_gbc.prod_category;
```

```
-- 未转换的执行计划（两个 NO_PLACE_GROUP_BY）

| Id  | Operation                   | Name                 | Cost (%CPU)|
| --- | --------------------------- | -------------------- | ---------- |
|   0 | SELECT STATEMENT            |                      | 29396   (1)|
|   1 |  HASH GROUP BY              |                      | 29396   (1)|
|*  2 |   HASH JOIN                 |                      | 11675   (1)|
|   3 |    VIEW                     |                      |  9046   (1)|
|   4 |     HASH GROUP BY           |                      |  9046   (1)|
|*  5 |      HASH JOIN              |                      |   521   (2)|
|   6 |       VIEW                  | index$_join$_002     |     2   (0)|
|*  7 |        HASH JOIN            |                      |            |
|   8 |         INDEX FAST FULL SCAN| PRODUCTS_PK          |     1   (0)|
|   9 |         INDEX FAST FULL SCAN| PRODUCTS_PROD_CAT_IX |     1   (0)|
|  10 |       PARTITION RANGE ALL   |                      |   517   (2)|
|  11 |        TABLE ACCESS FULL    | SALES                |   517   (2)|
|  12 |    TABLE ACCESS FULL        | CUSTOMERS            |   423   (1)|

谓词信息（按操作 ID 标识）：

2 - access("VW_GBC"."CUST_ID"="C"."CUST_ID")
5 - access("S"."PROD_ID"="P"."PROD_ID")
7 - access(ROWID=ROWID)
```

```
-- 转换后的查询

WITH vw_gbc_1
     AS (  SELECT s.cust_id, s.prod_id, SUM (s.amount_sold) AS total_amt_sold
             FROM sh.sales s
         GROUP BY s.cust_id, s.prod_id)
    ,vw_gbc_2
     AS (  SELECT vw_gbc_1.cust_id, vw_gbc_1.total_amt_sold, p.prod_category
             FROM vw_gbc_1 JOIN sh.products p USING (prod_id)
         GROUP BY vw_gbc_1.cust_id
                 ,vw_gbc_1.total_amt_sold
                 ,p.prod_category
                 ,prod_id)
    ,vw_gbc
     AS (  SELECT vw_gbc_2.cust_id
                 ,SUM (vw_gbc_2.total_amt_sold) AS total_amt_sold
                 ,vw_gbc_2.prod_category
             FROM vw_gbc_2
         GROUP BY vw_gbc_2.cust_id, vw_gbc_2.prod_category)
  SELECT cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,vw_gbc.prod_category
        ,SUM (vw_gbc.total_amt_sold) total_amt_sold
    FROM vw_gbc JOIN sh.customers c USING (cust_id)
GROUP BY cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,vw_gbc.prod_category;
```

```
-- 转换后的执行计划（默认）

| Id  | Operation                     | Name                 | Cost (%CPU)|
| --- | ----------------------------- | -------------------- | ---------- |
```

## 分组下推

分组下推对于并行聚合的作用，类似于去重和分组放置对连接聚合的作用。其核心思想相同：尽可能早地在流程中执行尽可能多的聚合操作。清单 13-19 展示了分组下推转换的实际应用。

**清单 13-19.** 分组下推

```sql
 SELECT /*+ parallel gby_pushdown */ /* parallel no_gby_pushdown */
        prod_id
        ,cust_id
        ,promo_id
        ,COUNT (*) cnt
    FROM sh.sales
   WHERE amount_sold > 100
GROUP BY prod_id, cust_id, promo_id;
-- 未转换的执行计划 (NO_GBY_PUSHDOWN)

| Id  | Operation               | Name     | Cost (%CPU)|

|   0 | SELECT STATEMENT        |          |   289   (3)|
|   1 |  PX COORDINATOR         |          |            |
|   2 |   PX SEND QC (RANDOM)   | :TQ10001 |   289   (3)|
|   3 |    HASH GROUP BY        |          |   289   (3)|
|   4 |     PX RECEIVE          |          |   288   (2)|
|   5 |      PX SEND HASH       | :TQ10000 |   288   (2)|
|   6 |       PX BLOCK ITERATOR |          |   288   (2)|
|*  7 |        TABLE ACCESS FULL| SALES    |   288   (2)|

谓词信息 (通过操作 ID 识别):

7 - filter("AMOUNT_SOLD">100)

-- 转换后查询的概念

WITH pq
     AS (  SELECT prod_id
                 ,cust_id
                 ,promo_id
                 ,COUNT (*) cnt
             FROM sh.sales
            WHERE amount_sold > 100 AND time_id < DATE '2000-01-01'
         GROUP BY prod_id, cust_id, promo_id
         UNION ALL
           SELECT prod_id
                 ,cust_id
                 ,promo_id
                 ,COUNT (*) cnt
             FROM sh.sales
            WHERE amount_sold > 100 AND time_id >=DATE '2000-01-01'
         GROUP BY prod_id, cust_id, promo_id)
  SELECT prod_id
        ,cust_id
        ,promo_id
        ,SUM (cnt) AS cnt
    FROM pq
GROUP BY prod_id, cust_id, promo_id;

-- 转换后的执行计划

| Id  | Operation                | Name     | Cost (%CPU)|

|   0 | SELECT STATEMENT         |          |   325   (1)|
|   1 |  PX COORDINATOR          |          |            |
|   2 |   PX SEND QC (RANDOM)    | :TQ10001 |   325   (1)|
|   3 |    HASH GROUP BY         |          |   325   (1)|
|   4 |     PX RECEIVE           |          |   325   (1)|
|   5 |      PX SEND HASH        | :TQ10000 |   325   (1)|
|   6 |       HASH GROUP BY      |          |   325   (1)|
|   7 |        PX BLOCK ITERATOR |          |   144   (2)|
|*  8 |         TABLE ACCESS FULL| SALES    |   144   (2)|

谓词信息 (通过操作 ID 识别):

8 - filter("AMOUNT_SOLD">100)
```

清单 13-19 中的查询是对 `SH.SALES` 执行的一个简单的 `COUNT` 聚合，并以并行方式进行。若没有此转换，所有满足 `AMOUNT_SOLD > 100` 的 `SH.SALES` 行都将从一个并行查询服务器发送到另一个服务器进行聚合。此转换的概念通过清单 13-19 中的一个 `UNION ALL` 因子化子查询来展示。`UNION ALL` 的每个分支都对应于一个并行查询服务器，该服务器读取 `SH.SALES` 表的一部分数据。概念性查询使用世纪来分割分支间的数据，而实际的并行查询将使用块范围粒度。

在概念性查询中，`UNION ALL` 的每个分支先执行自己的 `COUNT` 聚合，然后主查询再对结果执行 `SUM` 聚合以获得最终结果。在并行查询中，每个读取 `SH.SALES` 的并行查询从进程都会对其数据子集执行 `COUNT` 聚合，因此需要发送到 DFO:`TQ10001` 进行最终 `SUM` 聚合的行数大大减少。

我们再次看到，CBO 执行了一个实际上增加了估算成本的转换。越来越多的证据表明，基于成本的转换框架实际上并未对每个可能的转换执行最终状态优化，而是执行了某种近似的成本计算。

分组下推是一种基于成本的转换，它适用于 `DISTINCT` 操作以及 `GROUP BY` 操作。由于 `GROUP BY` 和 `DISTINCT` 操作几乎总是能显著降低基数，因此只要合法，分组下推几乎总是会被应用。

分组下推转换是我想要介绍的最后一个聚合转换。现在，是时候看看子查询了。

## 子查询转换

正如我在第 7 章中解释的，我在本书中非正式地使用术语 *子查询* 来指代一个集合查询块，或任何以关键字 `SELECT` 开头且不是查询主查询块的 SQL 语句部分。因此，例如，一个内联视图或数据字典视图在此处的讨论中被视为一个子查询。

我们已经在第 11 章和本章前面讨论过子查询非合并，因此我不再重访该主题。但我们还有很多其他转换要介绍。让我们通过简要回顾简单视图合并来开始吧。

### 简单视图合并

|   0 | SELECT STATEMENT              |                      |  7224   (1)|
|   1 |  HASH GROUP BY                |                      |  7224   (1)|
|*  2 |   HASH JOIN                   |                      |  6741   (1)|
|   3 |    VIEW                       | VW_GBC_3             |  6104   (1)|
|   4 |     HASH GROUP BY             |                      |  6104   (1)|
|   5 |      VIEW                     |                      |  6104   (1)|
|   6 |       HASH GROUP BY           |                      |  6104   (1)|
|*  7 |        HASH JOIN              |                      |  3022   (1)|
|   8 |         VIEW                  | index$_join$_002     |     2   (0)|
|*  9 |          HASH JOIN            |                      |            |
|  10 |           INDEX FAST FULL SCAN| PRODUCTS_PK          |     1   (0)|
|  11 |           INDEX FAST FULL SCAN| PRODUCTS_PROD_CAT_IX |     1   (0)|
|  12 |         VIEW                  | VW_GBC_2             |  3019   (1)|
|  13 |          HASH GROUP BY        |                      |  3019   (1)|
|  14 |           PARTITION RANGE ALL |                      |   517   (2)|
|  15 |            TABLE ACCESS FULL  | SALES                |   517   (2)|
|  16 |    TABLE ACCESS FULL          | CUSTOMERS            |   423   (1)|

谓词信息 (通过操作 ID 识别):

2 - access("ITEM_1"="C"."CUST_ID")
   7 - access("ITEM_1"="P"."PROD_ID")
   9 - access(ROWID=ROWID)
```

清单 13-18 表明，如果我们直接运行或解释清单 13-17 的转换后查询，它将经历进一步的转换。现在我们有了两个独立的查询块，CBO 可以在任何连接之前聚合来自 `SH.SALES` 的数据。在与 `SH.PRODUCTS` 连接之后、与 `SH.CUSTOMERS` 连接之前，数据被再次聚合。在此阶段引入了一个额外且不必要的聚合：第 4 行和第 6 行的操作本可以被合并。

这一分析引出了一个有趣的 SQL 调优技术：通过观察 CBO 执行了哪些转换，我们可以通过迭代运行转换过程来进一步改进。可能并非所有这些转换在实践中都有效，但至少你会有一些想法可以尝试！


我们已经在本书中看到了大量简单视图合并的示例，现在让我们简要地正式审视一下 Listing 13-20 中的转换。

### 简单视图合并

```
WITH q1
     AS (SELECT /*+   MERGE */
                /* NO_MERGE */
               CASE prod_category
                   WHEN 'Electronics' THEN amount_sold * 0.9
                   ELSE amount_sold
                END
                   AS adjusted_amount_sold
           FROM sh.sales JOIN sh.products USING (prod_id))
  SELECT adjusted_amount_sold, COUNT (*) cnt
    FROM q1
GROUP BY adjusted_amount_sold;
```

-- 未转换的执行计划 (NO_MERGE)
```
| Id  | Operation                 | Name                 | Cost (%CPU)|

|   0 | SELECT STATEMENT          |                      |   542   (6)|
|   1 |  HASH GROUP BY            |                      |   542   (6)|
|   2 |   VIEW                    |                      |   521   (2)|
|*  3 |    HASH JOIN              |                      |   521   (2)|
|   4 |     VIEW                  | index$_join$_002     |     2   (0)|
|*  5 |      HASH JOIN            |                      |            |
|   6 |       INDEX FAST FULL SCAN| PRODUCTS_PK          |     1   (0)|
|   7 |       INDEX FAST FULL SCAN| PRODUCTS_PROD_CAT_IX |     1   (0)|
|   8 |     PARTITION RANGE ALL   |                      |   517   (2)|
|   9 |      TABLE ACCESS FULL    | SALES                |   517   (2)|

Predicate Information (identified by operation id):

3 - access("SALES"."PROD_ID"="PRODUCTS"."PROD_ID")
   5 - access(ROWID=ROWID)
```

-- 转换后的查询
```
SELECT CASE prod_category
            WHEN 'Electronics' THEN amount_sold * 0.9
            ELSE amount_sold
         END
            AS adjusted_amount_sold
    FROM sh.sales JOIN sh.products USING (prod_id)
GROUP BY CASE prod_category
            WHEN 'Electronics' THEN amount_sold * 0.9
            ELSE amount_sold
         END;
```

-- 转换后的执行计划 (默认)
```
| Id  | Operation                | Name                 | Cost (%CPU)|

|   0 | SELECT STATEMENT         |                      |  3233   (1)|
|   1 |  HASH GROUP BY           |                      |  3233   (1)|
|*  2 |   HASH JOIN              |                      |   521   (2)|
|   3 |    VIEW                  | index$_join$_002     |     2   (0)|
|*  4 |     HASH JOIN            |                      |            |
|   5 |      INDEX FAST FULL SCAN| PRODUCTS_PK          |     1   (0)|
|   6 |      INDEX FAST FULL SCAN| PRODUCTS_PROD_CAT_IX |     1   (0)|
|   7 |    PARTITION RANGE ALL   |                      |   517   (2)|
|   8 |     TABLE ACCESS FULL    | SALES                |   517   (2)|

Predicate Information (identified by operation id):

2 - access("SALES"."PROD_ID"="PRODUCTS"."PROD_ID")
   4 - access(ROWID=ROWID)
```

子查询展开和视图合并的概念很容易混淆：

*   视图合并适用于出现在外部查询块 `FROM` 子句中作为行源的内联视图、因子化子查询和数据字典视图。视图合并由 `MERGE` 和 `NO_MERGE` 提示控制。
*   子查询展开与 `SELECT` 列表、`WHERE` 子句或 Oracle 未来可能支持的任何其他地方的子查询相关。子查询展开由 `UNNEST` 和 `NO_UNNEST` 提示控制。

Listing 13-20 展示了使用因子化子查询如何节省输入；我们不必将 `GROUP BY` 表达式复制到选择列表。我们只是使用了子查询中的列别名 `adjusted_amount_sold`。再一次，Oracle 将我们简洁的查询转换成了概念上更简单（即使不那么简洁）的形式。

简单视图合并是一种启发式转换，在合法的情况下会被无条件地应用。简单视图合并的主要好处是可能实现额外的连接顺序（尽管在 Listing 13-20 中并非如此）。

我不会深入探讨简单视图合并的限制。你只需记住，简单的行源可以被合并，而复杂的则不能，这样就没问题了。有一种特殊情况，复杂的行源可以被合并，但不是通过简单视图合并。现在让我们来看看这个。

### 复杂视图合并

如果一个子查询包含 `GROUP BY` 子句和/或 `DISTINCT` 关键字，并且这是*唯一*导致无法使用简单视图合并的原因，那么复杂视图合并是一个替代方案。Listing 13-21 给出了一个例子。

### 复杂视图合并

```
WITH agg_q
     AS (  SELECT /*+   merge */
                  /* no_merge */
                  s.cust_id, s.prod_id, SUM (s.amount_sold) total_amt_sold
             FROM sh.sales s
         GROUP BY s.cust_id, s.prod_id)
SELECT cust_id
      ,c.cust_first_name
      ,c.cust_last_name
      ,c.cust_email
      ,p.prod_name
      ,agg_q.total_amt_sold
  FROM agg_q
       JOIN sh.customers c USING (cust_id)
       JOIN sh.countries co USING (country_id)
       JOIN sh.products p USING (prod_id)
 WHERE     co.country_name = 'Japan'
       AND prod_category = 'Photo'
       AND total_amt_sold > 20000;
```

-- 未转换的执行计划 (NO_MERGE)
```
| Id  | Operation                       | Name         | Cost (%CPU)|

|   0 | SELECT STATEMENT                |              |   541   (6)|
|   1 |  NESTED LOOPS                   |              |            |
|   2 |   NESTED LOOPS                  |              |   541   (6)|
|   3 |    NESTED LOOPS                 |              |   540   (6)|
|   4 |     NESTED LOOPS                |              |   539   (6)|
|   5 |      VIEW                       |              |   538   (6)|
|*  6 |       FILTER                    |              |            |
|   7 |        HASH GROUP BY            |              |   538   (6)|
|   8 |         PARTITION RANGE ALL     |              |   517   (2)|
|   9 |          TABLE ACCESS FULL      | SALES        |   517   (2)|
|  10 |      TABLE ACCESS BY INDEX ROWID| CUSTOMERS    |     1   (0)|
|* 11 |       INDEX UNIQUE SCAN         | CUSTOMERS_PK |     0   (0)|
|* 12 |     TABLE ACCESS BY INDEX ROWID | COUNTRIES    |     1   (0)|
|* 13 |      INDEX UNIQUE SCAN          | COUNTRIES_PK |     0   (0)|
|* 14 |    INDEX UNIQUE SCAN            | PRODUCTS_PK  |     0   (0)|
|* 15 |   TABLE ACCESS BY INDEX ROWID   | PRODUCTS     |     1   (0)|

Predicate Information (identified by operation id):

6 - filter(SUM("S"."AMOUNT_SOLD")>20000)
  11 - access("AGG_Q"."CUST_ID"="C"."CUST_ID")
  12 - filter("CO"."COUNTRY_NAME"='Japan')
  13 - access("C"."COUNTRY_ID"="CO"."COUNTRY_ID")
  14 - access("AGG_Q"."PROD_ID"="P"."PROD_ID")
  15 - filter("P"."PROD_CATEGORY"='Photo')
```

-- 转换后的查询
```
SELECT cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,p.prod_name
        ,SUM (s.amount_sold) AS total_amt_sold
    FROM sh.sales s
         JOIN sh.customers c USING (cust_id)
         JOIN sh.countries co USING (country_id)
         JOIN sh.products p USING (prod_id)
   WHERE co.country_name = 'Japan' AND prod_category = 'Photo'
GROUP BY cust_id
        ,c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,p.prod_name
  HAVING SUM (s.amount_sold) > 20000;
```

-- 转换后的执行计划 (默认)
```
| Id  | Operation                              | Name                 | Cost (%CPU)|
```



|   0 | SELECT STATEMENT                       |                      |   948   (2)|
|*  1 |  FILTER                                |                      |            |
|   2 |   HASH GROUP BY                        |                      |   948   (2)|
|*  3 |    HASH JOIN                           |                      |   948   (2)|
|   4 |     TABLE ACCESS BY INDEX ROWID BATCHED| PRODUCTS             |     3   (0)|
|*  5 |      INDEX RANGE SCAN                  | PRODUCTS_PROD_CAT_IX |     1   (0)|
|*  6 |     HASH JOIN                          |                      |   945   (2)|
|*  7 |      HASH JOIN                         |                      |   426   (1)|
|*  8 |       TABLE ACCESS FULL                | COUNTRIES            |     3   (0)|
|   9 |       TABLE ACCESS FULL                | CUSTOMERS            |   423   (1)|
|  10 |      PARTITION RANGE ALL               |                      |   517   (2)|
|  11 |       TABLE ACCESS FULL                | SALES                |   517   (2)|

谓词信息（由操作 ID 标识）：
```
   1 - filter(SUM("S"."AMOUNT_SOLD")>20000)
   3 - access("S"."PROD_ID"="P"."PROD_ID")
   5 - access("P"."PROD_CATEGORY"='Photo')
   6 - access("S"."CUST_ID"="C"."CUST_ID")
   7 - access("C"."COUNTRY_ID"="CO"."COUNTRY_ID")
   8 - filter("CO"."COUNTRY_NAME"='Japan')
```

清单 13-21 展示了日本市场摄影产品对每个客户的总销售额。此处视图合并的好处在于，聚合操作被延迟执行，直到来自其他国家和其他产品类别的行被过滤掉之后才进行。

现在，你可能在疑惑为什么简单视图合并和复杂视图合并被分开讨论。毕竟，这两种转换做的事情是一样的，并且都由 `MERGE` 和 `NO_MERGE` 暗示来管理。关键的区别在于，简单视图合并是一种*启发式*转换，而复杂视图合并是一种*基于成本的*转换。

如果你不合并一个复杂视图，你就有机会提前聚合，减少输入到连接中的行数。如果你*确实*合并了，那么你的连接可以减少被聚合的行数。因此，是先执行连接还是先执行聚合，取决于哪种操作能最大程度地减小中间结果集。事实上，如果你从清单 13-21 的原始查询中移除 `WHERE` 子句，除非你包含了 `MERGE` 暗示，否则复杂视图合并将不会发生。

这听起来熟悉吗？应该是的。我们这里进行的讨论，与之前关于 `DISTINCT` 位置和 `GROUP BY` 位置转换的讨论完全相同，只是方向相反！可下载的资料表明，如果你取清单 13-16 中的转换后查询，并添加一个 `MERGE` 暗示，你就可以通过反转原始转换来重建原始查询。²

## 提取子查询物化

在第 1 章中，我解释了提取子查询的一个关键好处是它们可以在一个查询中被多次使用。事实上，当它们被多次使用时，会创建一个专属于该语句的临时表；这个表保存了提取子查询的结果。清单 13-22 演示了这一点。

清单 13-22. 提取子查询物化

```sql
WITH q1
     AS (  SELECT prod_name, prod_category, SUM (amount_sold) total_amt_sold
             FROM sh.sales JOIN sh.products USING (prod_id)
            WHERE prod_category = 'Electronics'
         GROUP BY prod_name, prod_category)
    ,q2
     AS (SELECT 1 AS order_col, prod_name, total_amt_sold FROM q1
         UNION ALL
         SELECT 2, 'Total', SUM (total_amt_sold) FROM q1)
  SELECT prod_name, total_amt_sold
    FROM q2
ORDER BY order_col, prod_name;
```

```
| Id  | Operation                               | Name                      | Cost (%CPU)|

|   0 | SELECT STATEMENT                        |                           |   529   (3)|
|   1 |  TEMP TABLE TRANSFORMATION              |                           |            |
|   2 |   LOAD AS SELECT                        | SYS_TEMP_0FD9D672C_4E5E5D |            |
|   3 |    HASH GROUP BY                        |                           |   525   (3)|
|*  4 |     HASH JOIN                           |                           |   522   (2)|
|   5 |      TABLE ACCESS BY INDEX ROWID BATCHED| PRODUCTS                  |     3   (0)|
|*  6 |       INDEX RANGE SCAN                  | PRODUCTS_PROD_CAT_IX      |     1   (0)|
|   7 |      PARTITION RANGE ALL                |                           |   517   (2)|
|   8 |       TABLE ACCESS FULL                 | SALES                     |   517   (2)|
|   9 |   SORT ORDER BY                         |                           |     4   (0)|
|  10 |    VIEW                                 |                           |     4   (0)|
|  11 |     UNION-ALL                           |                           |            |
|  12 |      VIEW                               |                           |     2   (0)|
|  13 |       TABLE ACCESS FULL                 | SYS_TEMP_0FD9D672C_4E5E5D |     2   (0)|
|  14 |      SORT AGGREGATE                     |                           |            |
|  15 |       VIEW                              |                           |     2   (0)|
|  16 |        TABLE ACCESS FULL                | SYS_TEMP_0FD9D672C_4E5E5D |     2   (0)|

谓词信息（由操作 ID 标识）：
```
   4 - access("SALES"."PROD_ID"="PRODUCTS"."PROD_ID")
   6 - access("PRODUCTS"."PROD_CATEGORY"='Electronics')
```

清单 13-22 与本章其他清单略有不同，因为我使用了一个无暗示的查询，并且只展示了转换后的执行计划。该查询显示了按产品统计的总销售额，并在末尾给出了总计。清单 13-22 展示的这类查询，可能是由不熟悉在 `GROUP BY` 操作中使用 `ROLLUP` 关键字的人编写的。

`TEMP TABLE TRANSFORMATION` 操作标志着创建一个或多个临时表，这些表仅在该操作期间存在。该操作的子节点总是一个或多个 `LOAD AS SELECT` 操作（它们将行插入临时表），外加一个用于主查询的额外子节点（该主查询利用创建的一个或多个临时表）。

清单 13-22 使用了提取子查询 `Q1` 两次，但该子查询只被求值一次；查询的结果存储在一个名称古怪的临时表中（本例中为 `SYS_TEMP_0FD9D672C_4E5E5D`），并且该临时表在 `UNION ALL` 操作的两个分支中被使用。

当选择的是对象类型或 `LOB` 时，提取子查询物化是不可能的，并且在分布式事务中该转换也是非法的。

在合法的情况下，提取子查询物化是一种启发式优化。如果一个子查询只被引用一次，则它不会被物化，但如果它被引用两次或更多次，通常就会被物化。³ 你可以在清单 13-22 中看到，提取子查询 `Q2` 只被引用了一次，因此它没有被物化。CBO 选择改为对提取子查询与主查询进行简单的视图合并。

所有这些听起来都很合理，但千万别以为提取子查询物化是无需动脑筋的选择。我们将在第 18 章讨论*为什么*你可能需要覆盖这些启发式规则，但现在我们需要专注于*如何*覆盖默认行为。

暗示的过程在几个方面有点奇怪：



*   要强制物化因子子查询，需使用 `MATERIALIZE` 提示符；要禁用物化，则使用 `INLINE` 提示符。
*   `MATERIALIZE` 和 `INLINE` 提示符均不支持全局提示符语法。这些提示符必须直接放置在因子子查询内部。
*   由于上述限制，`MATERIALIZE` 和 `INLINE` 提示符永远不会出现在大纲提示符、SQL 基线、SQL 配置文件等之中。将提示符放置在代码中是唯一的方法！

牢记以上要点，清单 13-23 展示了如何阻止因子子查询 `Q1` 的物化并物化 `Q2`（在某种意义上）。

清单 13-23. `MATERIALIZE` 和 `INLINE` 提示符的使用

```sql
WITH q1
     AS (  SELECT /*+ inline */
                 prod_name, prod_category, SUM (amount_sold) total_amt_sold
             FROM sh.sales JOIN sh.products USING (prod_id)
            WHERE prod_category = 'Electronics'
         GROUP BY prod_name, prod_category)
    ,q2
     AS (SELECT 1 AS order_col, prod_name, total_amt_sold FROM q1
         UNION ALL
         SELECT 2, 'Total', SUM (total_amt_sold) FROM q1)
    ,q3AS (SELECT /*+ materialize */
                  * FROM q2)
  SELECT prod_name, total_amt_sold
    FROM q3
ORDER BY order_col, prod_name;

| Id  | Operation                                   | Name                      | Cost (%CPU)|

|   0 | SELECT STATEMENT                            |                           |  1053   (3)|
|   1 |  TEMP TABLE TRANSFORMATION                  |                           |            |
|   2 |   LOAD AS SELECT                            | SYS_TEMP_0FD9D672D_4E5E5D |            |
|   3 |    VIEW                                     |                           |  1051   (3)|
|   4 |     UNION-ALL                               |                           |            |
|   5 |      HASH GROUP BY                          |                           |   525   (3)|
|*  6 |       HASH JOIN                             |                           |   522   (2)|
|   7 |        TABLE ACCESS BY INDEX ROWID BATCHED  | PRODUCTS                  |     3   (0)|
|*  8 |         INDEX RANGE SCAN                    | PRODUCTS_PROD_CAT_IX      |     1   (0)|
|   9 |        PARTITION RANGE ALL                  |                           |   517   (2)|
|  10 |         TABLE ACCESS FULL                   | SALES                     |   517   (2)|
|  11 |      SORT AGGREGATE                         |                           |            |
|  12 |       VIEW                                  |                           |   525   (3)|
|  13 |        HASH GROUP BY                        |                           |   525   (3)|
|* 14 |         HASH JOIN                           |                           |   522   (2)|
|  15 |          TABLE ACCESS BY INDEX ROWID BATCHED| PRODUCTS                  |     3   (0)|
|* 16 |           INDEX RANGE SCAN                  | PRODUCTS_PROD_CAT_IX      |     1   (0)|
|  17 |          PARTITION RANGE ALL                |                           |   517   (2)|
|  18 |           TABLE ACCESS FULL                 | SALES                     |   517   (2)|
|  19 |   SORT ORDER BY                             |                           |     2   (0)|
|  20 |    VIEW                                     |                           |     2   (0)|
|  21 |     TABLE ACCESS FULL                       | SYS_TEMP_0FD9D672D_4E5E5D |     2   (0)|

Predicate Information (identified by operation id):

6 - access("SALES"."PROD_ID"="PRODUCTS"."PROD_ID")
   8 - access("PRODUCTS"."PROD_CATEGORY"='Electronics')
  14 - access("SALES"."PROD_ID"="PRODUCTS"."PROD_ID")
  16 - access("PRODUCTS"."PROD_CATEGORY"='Electronics')
```

在清单 13-23 中使用 `MATERIALIZE` 和 `INLINE` 提示符，导致 `SH.PRODUCTS` 和 `SH.SALES` 的连接被执行了两次，并且 `UNION ALL` 的结果被放入了一个临时表中，即使该临时表仅被引用了一次。

我不得不在这里做一点取巧。你会注意到，我在清单 13-23 中悄悄地添加了第三个因子子查询 `Q3`。这是因为 `Q2` 的查询块是集合查询块 `SET$1`，只能使用全局提示符进行提示（在第一个 `SELECT` 之后应用的提示符将应用于 `UNION ALL` 的第一个分支）。由于 `MATERIALIZE` 只支持局部提示，唯一的办法就是创建一个新的查询块，将 `SET$1` 合并进去，然后物化该查询块。

我想强调的是，本次讨论侧重于因子子查询物化的机制。我不希望给你留下这样的印象：我在清单 13-23 中应用的提示符在性能方面有任何意义。我们将在第 18 章中回到性能问题。

**子查询下推**

当子查询出现在 `WHERE` 子句中时，CBO 如果能够进行子查询非嵌套，通常就会这样做。如果子查询由于某种原因无法非嵌套，那么 CBO 就会面临一个类似于处理聚合时的问题：应该尽早评估子查询，还是推迟到所有连接之后再评估？奇怪的是，这个决定是通过一个查询转换来做出的。

尽管子查询下推是作为查询转换实现的，但它在逻辑上并非一个转换。事实上，我曾一度认为，CBO 决定何时评估子查询是我们第 12 章讨论的最终状态转换逻辑的一部分。显然，无法提供任何显示所谓转换结果的已转换 SQL。

它的工作原理如下：CBO 首先假设 `WHERE` 子句中的子查询应在所有连接执行之后进行评估。只有当基于成本的转换框架决定应该改变此行为（或存在提示符）时，此行为才会改变。清单 13-24 或许有助于更好地解释这一点。

清单 13-24. 子查询下推

```sql
WITH q1
     AS (  SELECT prod_id
             FROM sh.sales
         GROUP BY prod_id
         ORDER BY SUM (amount_sold) DESC)
  SELECT c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,p.prod_name
    FROM sh.sales s
         JOIN sh.customers c USING (cust_id)
         JOIN sh.products p USING (prod_id)
   WHERE prod_id = (SELECT /*+   push_subq */
                           /* no_push_subq */
                           prod_id
                      FROM q1
                     WHERE ROWNUM = 1)
GROUP BY c.cust_first_name
        ,c.cust_last_name
        ,c.cust_email
        ,p.prod_name
  HAVING SUM (s.amount_sold) > 20000;

-- 未下推子查询 (NO_PUSH_SUBQ)

| Id  | Operation                  | Name      | Cost (%CPU)|
```



## 执行计划分析

| Id | Operation | Name | Cost (%CPU) |
|---|---|---|---|
| 0 | SELECT STATEMENT | | 10277 (1) |
| *1 | FILTER | | |
| 2 | HASH GROUP BY | | 10277 (1) |
| *3 | FILTER | | |
| *4 | HASH JOIN | | 2237 (1) |
| 5 | TABLE ACCESS FULL | PRODUCTS | 3 (0) |
| *6 | HASH JOIN | | 2232 (1) |
| 7 | TABLE ACCESS FULL | CUSTOMERS | 423 (1) |
| 8 | PARTITION RANGE ALL | | 517 (2) |
| 9 | TABLE ACCESS FULL | SALES | 517 (2) |
| *10 | COUNT STOPKEY | | |
| 11 | VIEW | | 559 (9) |
| *12 | SORT ORDER BY STOPKEY | | 559 (9) |
| 13 | SORT GROUP BY | | 559 (9) |
| 14 | PARTITION RANGE ALL | | 517 (2) |
| 15 | TABLE ACCESS FULL | SALES | 517 (2) |

**谓词信息（由操作 ID 识别）：**

1 - filter(`SUM("S"."AMOUNT_SOLD")>20000`)
3 - filter(`"P"."PROD_ID"= (SELECT /*+ NO_PUSH_SUBQ */ "PROD_ID" FROM (SELECT "PROD_ID" "PROD_ID" FROM "SH"."SALES" "SALES" GROUP BY "PROD_ID" ORDER BY SUM("AMOUNT_SOLD") DESC) "Q1" WHERE ROWNUM=1)`)
4 - access(`"S"."PROD_ID"="P"."PROD_ID"`)
6 - access(`"S"."CUST_ID"="C"."CUST_ID"`)
10 - filter(`ROWNUM=1`)
12 - filter(`ROWNUM=1`)

## 子查询下推（默认）

| Id | Operation | Name | Cost (%CPU) |
|---|---|---|---|
| 0 | SELECT STATEMENT | | 865 (1) |
| *1 | FILTER | | |
| 2 | HASH GROUP BY | | 865 (1) |
| *3 | HASH JOIN | | 864 (1) |
| 4 | NESTED LOOPS | | 441 (0) |
| 5 | TABLE ACCESS BY INDEX ROWID | PRODUCTS | 1 (0) |
| *6 | INDEX UNIQUE SCAN | PRODUCTS_PK | 0 (0) |
| *7 | COUNT STOPKEY | | |
| 8 | VIEW | | 559 (9) |
| *9 | SORT ORDER BY STOPKEY | | 559 (9) |
| 10 | SORT GROUP BY | | 559 (9) |
| 11 | PARTITION RANGE ALL | | 517 (2) |
| 12 | TABLE ACCESS FULL | SALES | 517 (2) |
| 13 | PARTITION RANGE ALL | | 441 (0) |
| 14 | TABLE ACCESS BY LOCAL INDEX ROWID BATCHED | SALES | 441 (0) |
| 15 | BITMAP CONVERSION TO ROWIDS | | |
| *16 | BITMAP INDEX SINGLE VALUE | SALES_PROD_BIX | |
| 17 | TABLE ACCESS FULL | CUSTOMERS | 423 (1) |

**谓词信息（由操作 ID 识别）：**

1 - filter(`SUM("S"."AMOUNT_SOLD")>20000`)
3 - access(`"S"."CUST_ID"="C"."CUST_ID"`)
6 - access(`"P"."PROD_ID"= (SELECT /*+ PUSH_SUBQ */ "PROD_ID" FROM (SELECT "PROD_ID" "PROD_ID" FROM "SH"."SALES" "SALES" GROUP BY "PROD_ID" ORDER BY SUM("AMOUNT_SOLD") DESC) "Q1" WHERE ROWNUM=1)`)
7 - filter(`ROWNUM=1`)
9 - filter(`ROWNUM=1`)
16 - access(`"S"."PROD_ID"="P"."PROD_ID"`)

在子查询中存在 `ROWNUM=1` 谓词（见 Listing 13-24）会阻止任何子查询解嵌套。如果 `NO_PUSH_SUBQ` 提示抑制了子查询下推转换，则执行计划中会出现 `FILTER` 操作。理论上，如果你看到一个带有多个操作数的 `FILTER` 操作，则第二个及后续的操作数（即子查询）会对第一个操作数（主查询）返回的每一行进行评估。事实上，在 Listing 13-24 中，子查询是不相关的，标量子查询缓存将确保子查询只被评估一次。

但在 Listing 13-24 的情况下，抑制转换会使查询变慢。关键在于，如果没有这个转换，我们将把 `SH.SALES` 中的所有行与 `SH.PRODUCTS` 和 `SH.CUSTOMERS` 表进行连接，最后丢弃大部分连接后的行。我们想要做的是尽早评估子查询，并将过滤条件应用于 `SH.PRODUCTS` 的行，从而得到我们想要的那一行。然后我们可以使用嵌套循环连接和索引访问，仅从 `SH.SALES` 中获取匹配该 `PROD_ID` 值的行。

当应用子查询下推时，子查询会作为某个通常不会有子节点的操作的子节点被应用。在此例中，操作 6 的 `INDEX UNIQUE SCAN` 通过评估其子节点来获取所需的 `PROD_ID` 值。

![image](img/sq.jpg) **注意** `PUSH_SUBQ` 提示的 SQL 语言参考手册文档声称，当应用于使用合并连接（merge join）连接的表时，该提示将无效。实际上，该限制仅当子查询与第二个表相关时才适用，在这种情况下，第二个相关表必须在连接顺序中先于第一个表，并且第一个表必须使用嵌套循环连接（nested loops join）访问。

Listing 13-25 展示了一个由于连接方法导致子查询下推非法的示例。

### Listing 13-25. 子查询下推：非法和不佳的示例

```
SELECT /* leading(p s1) use_nl(s1) */
       -- Add plus sign for hint to be recognised
      *
 FROM sh.sales s1
      JOIN sh.customers c USING (cust_id)
      JOIN sh.products p ON s1.prod_id = p.prod_id
WHERE     s1.amount_sold > (SELECT /*+ push_subq */
                                   -- Hint inapplicable to hash joins
                                  AVG (s2.amount_sold)
                             FROM sh.sales s2
                            WHERE s2.prod_id = p.prod_id)
      AND p.prod_category = 'Electronics'
      AND c.cust_year_of_birth = 1919;

-- Execution plan with hash join (push_subq hint ignored)

| Id  | Operation                              | Name                 | Cost (%CPU)|

|   0 | SELECT STATEMENT                       |                      |   942   (2)|
|*  1 |  FILTER                                |                      |            |
|*  2 |   HASH JOIN                            |                      |   610   (2)|
|*  3 |    TABLE ACCESS FULL                   | CUSTOMERS            |   271   (1)|
|*  4 |    HASH JOIN                           |                      |   339   (3)|
|   5 |     TABLE ACCESS BY INDEX ROWID BATCHED| PRODUCTS             |     3   (0)|
|*  6 |      INDEX RANGE SCAN                  | PRODUCTS_PROD_CAT_IX |     1   (0)|
|   7 |     PARTITION RANGE ALL                |                      |   334   (3)|
|   8 |      TABLE ACCESS FULL                 | SALES                |   334   (3)|
|   9 |   SORT AGGREGATE                       |                      |            |
|  10 |    PARTITION RANGE ALL                 |                      |   332   (2)|
|* 11 |     TABLE ACCESS FULL                  | SALES                |   332   (2)|
```



