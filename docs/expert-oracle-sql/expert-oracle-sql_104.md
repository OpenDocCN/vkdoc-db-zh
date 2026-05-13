# 其他转换

在本章中，我试图将这些转换归类到逻辑组中，但有一些转换不太方便归入任何类别。让我们从“或”扩展开始。

## 或扩展

有时，将使用 `OR` 简洁编写的谓词改写并分开处理其各个部分是有意义的，如代码清单 13-29 所示。

### 代码清单 13-29. 或扩展

```sql
SELECT /*+ use_concat */
       /* no_expand */
      SUM (amount_sold) cnt
 FROM sh.sales JOIN sh.products USING (prod_id)
WHERE time_id = DATE '1998-03-31' OR prod_name = 'Y Box';

-- 未转换的执行计划 (NO_EXPAND)

| Id  | Operation             | Name     | Cost (%CPU)|

|   0 | SELECT STATEMENT      |          |   522   (2)|
|   1 |  SORT AGGREGATE       |          |            |
|*  2 |   HASH JOIN           |          |   522   (2)|
|   3 |    TABLE ACCESS FULL  | PRODUCTS |     3   (0)|
|   4 |    PARTITION RANGE ALL|          |   517   (2)|
|   5 |     TABLE ACCESS FULL | SALES    |   517   (2)|

Predicate Information (identified by operation id):

2 - access("SALES"."PROD_ID"="PRODUCTS"."PROD_ID")
       filter("SALES"."TIME_ID"=TO_DATE(' 1998-03-31 00:00:00',
              'syyyy-mm-dd hh24:mi:ss') OR "PRODUCTS"."PROD_NAME"='Y Box')

-- 转换后的查询 (近似)

WITH q1
     AS (SELECT amount_sold
           FROM sh.sales JOIN sh.products USING (prod_id)
          WHERE prod_name = 'Y Box'
         UNION ALL
         SELECT amount_sold
           FROM sh.sales JOIN sh.products USING (prod_id)
          WHERE time_id = DATE '1998-03-31' AND LNNVL (prod_name = 'Y Box'))
SELECT SUM (amount_sold) cnt
  FROM q1;

-- 转换后的执行计划 (默认)

| Id  | Operation                                     | Name           | Cost (%CPU)|
```



