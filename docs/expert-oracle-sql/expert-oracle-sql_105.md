# 数据库查询优化与转换

## OR 扩展转换

```
|   0 | SELECT STATEMENT                              |                |   472   (0)|
|   1 |  SORT AGGREGATE                               |                |            |
|   2 |   CONCATENATION                               |                |            |
|   3 |    NESTED LOOPS                               |                |            |
|   4 |     NESTED LOOPS                              |                |   445   (0)|
|*  5 |      TABLE ACCESS FULL                        | PRODUCTS       |     3   (0)|
|   6 |      PARTITION RANGE ALL                      |                |            |
|   7 |       BITMAP CONVERSION TO ROWIDS             |                |            |
|*  8 |        BITMAP INDEX SINGLE VALUE              | SALES_PROD_BIX |            |
|   9 |     TABLE ACCESS BY LOCAL INDEX ROWID         | SALES          |   445   (0)|
|* 10 |    HASH JOIN                                  |                |    27   (0)|
|* 11 |     TABLE ACCESS FULL                         | PRODUCTS       |     3   (0)|
|  12 |     PARTITION RANGE SINGLE                    |                |    24   (0)|
|  13 |      TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |    24   (0)|
|  14 |       BITMAP CONVERSION TO ROWIDS             |                |            |
|* 15 |        BITMAP INDEX SINGLE VALUE              | SALES_TIME_BIX |            |

Predicate Information (identified by operation id):

5 - filter("PRODUCTS"."PROD_NAME"='Y Box')
   8 - access("SALES"."PROD_ID"="PRODUCTS"."PROD_ID")
  10 - access("SALES"."PROD_ID"="PRODUCTS"."PROD_ID")
  11 - filter(LNNVL("PRODUCTS"."PROD_NAME"='Y Box'))
  15 - access("SALES"."TIME_ID"=TO_DATE(' 1998-03-31 00:00:00', 'syyyy-mm-dd
              hh24:mi:ss'))
```

如果没有基于成本的 OR 扩展转换可用，列表 13-29 中两个独立谓词的存在使得我们别无选择，只能对`SH.SALES`进行全表扫描。通过添加一个`UNION ALL`集合操作符，我们可以使用两个索引分别获取匹配各个谓词的行。

然而，`OR`表达式和`UNION ALL`集合操作符之间存在细微的语义差异。事实证明，在 1998 年 3 月 31 日有一个 Y Box 的销售记录。`UNION ALL`集合操作符通常会列出此行两次，而`OR`条件只会列出一次。为了避免这种重复，通过`LNNVL`函数将同时匹配两个条件的行从`UNION ALL`的第二个分支中排除。如果您不熟悉此函数，`LNNVL(PRODUCTS.PROD_NAME='Y Box')` 的含义是 `(PROD_NAME != 'Y Box' OR PROD_NAME IS NULL)`。

以下是关于 OR 扩展转换的几个关键点：

*   如前所述，抑制 OR 扩展的方法是使用`NO_EXPAND`提示。没有`NO_USE_CONCAT`提示。
*   当 OR 扩展转换创建`UNION ALL`操作时，它显示为`CONCATENATION`。但是，`UNION ALL`和`CONCATENATION`操作的功能完全相同。
*   当您解读包含`CONCATENATION`操作的执行计划时，请记住，其子节点的列出顺序通常与原始查询中出现的顺序相反。
*   有文档记载的`USE_CONCAT`变体是一种全有或全无的方法。一方面，如果`WHERE`子句中有多个`OR`条件，您不能决定只扩展其中一个。另一方面，如果您查看显示执行计划的概要部分，您会发现`USE_CONCAT`有一些未文档记录的参数，可以像替代的`OR_EXPAND`提示一样提供更精细的控制。不过，我尚未弄清楚它们具体如何工作。CBO 有时会自主地扩展一部分`OR`条件。

## 物化视图重写

物化视图是一个很大的话题，完全可以写一整章来讨论。事实上，《数据仓库指南》就有两章！在此，我只想简要演示一下如何将针对某些基础表编写的查询转换为针对物化视图的查询。列表 13-30 提供了一个简单示例。

列表 13-30. 物化视图重写

```
EXEC dbms_mview.refresh('SH.CAL_MONTH_SALES_MV');

ALTER SESSION SET query_rewrite_integrity=trusted;

SELECT /*+   rewrite(sh.cal_month_sales_mv) */
         /* no_rewrite */
         t.calendar_month_desc, SUM (s.amount_sold) AS dollars
    FROM sh.sales s, sh.times t
   WHERE s.time_id = t.time_id
GROUP BY t.calendar_month_desc;

-- 未转换的执行计划 (NO_REWRITE)

| Id  | Operation               | Name     | Cost (%CPU)|

|   0 | SELECT STATEMENT        |          |   367   (8)|
|   1 |  HASH GROUP BY          |          |   367   (8)|
|*  2 |   HASH JOIN             |          |   367   (8)|
|   3 |    VIEW                 | VW_GBC_5 |   355   (8)|
|   4 |     HASH GROUP BY       |          |   355   (8)|
|   5 |      PARTITION RANGE ALL|          |   334   (3)|
|   6 |       TABLE ACCESS FULL | SALES    |   334   (3)|
|   7 |    TABLE ACCESS FULL    | TIMES    |    12   (0)|

Predicate Information (identified by operation id):

2 - access("ITEM_1"="T"."TIME_ID")

-- 转换后的查询

SELECT *
  FROM sh.cal_month_sales_mv mv;

-- 转换后的执行计划 (默认)

| Id  | Operation                    | Name               | Cost (%CPU)|

|   0 | SELECT STATEMENT             |                    |     3   (0)|
|   1 |  MAT_VIEW REWRITE ACCESS FULL| CAL_MONTH_SALES_MV |     3   (0)|

```

尽管我们在开始前刷新了`SH.CAL_MONTH_SALES_MV`物化视图，但查询重写前仍必须设置`query_rewrite_integrity=trusted`。问题在于`SH.SALES`和`SH.TIMES`之间的引用完整性约束未经验证。虽然确定物化视图重写在何时何地是合法且可取的很棘手，但基本概念是直截了当的。在此示例中，汇总月度销售数据的前期工作已经完成，无需重复。

我还想讨论另一种物化视图转换。这种转换相当晦涩。

## 分组集到 UNION 扩展

分组集到 UNION 扩展转换在 9iR2 中引入，但相关著述甚少。该转换仅适用于涉及分组集的查询，且这些分组集的汇总数据可以从物化视图中获得。列表 13-31 进行了演示。

列表 13-31. 分组集到 UNION 扩展

```
ALTER SESSION SET query_rewrite_integrity=trusted;

SELECT /*+   expand_gset_to_union */
         /* no_expand_gset_to_union */
         DECODE (GROUPING (t.time_id), 1, 'Month total', t.time_id) AS time_id
        ,t.calendar_month_desc
        ,SUM (s.amount_sold) AS dollars
    FROM sh.sales s, sh.times t
   WHERE s.time_id = t.time_id
GROUP BY ROLLUP (t.time_id), t.calendar_month_desc
ORDER BY calendar_month_desc, time_id;

-- 未转换的执行计划 (NOEXPAND_GSET_TO_UNION)

| Id  | Operation                      | Name    | Cost (%CPU)|

|   0 | SELECT STATEMENT               |         |  6340   (1)|
|   1 |  SORT ORDER BY                 |         |  6340   (1)|
|   2 |   SORT GROUP BY ROLLUP         |         |  6340   (1)|
|*  3 |    HASH JOIN                   |         |   537   (2)|
|   4 |     PART JOIN FILTER CREATE    | :BF0000 |            |
|   5 |      TABLE ACCESS FULL         | TIMES   |    18   (0)|
|   6 |     PARTITION RANGE JOIN-FILTER|         |   517   (2)|
|   7 |      TABLE ACCESS FULL         | SALES   |   517   (2)|

Predicate Information (identified by operation id):
```



