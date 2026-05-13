# 使用 ROWID 和 FIRST/LAST 函数优化排序性能

```
WITH q1
     AS (  SELECT /*+ cardinality(s1 1e9) */
                      ROWID rid
             FROM sh.sales s1
         ORDER BY cust_id)
SELECT /*+ cardinality(s2 1e9) leading(q1 s2 c p)
       use_nl(s2)
       rowid(s2)
       use_hash(c)
       use_hash(p)
       swap_join_inputs(c)
       swap_join_inputs(p)
       */
       *
  FROM q1
       JOIN sh.sales s2 ON s2.ROWID = q1.rid
       JOIN sh.customers c USING (cust_id)
       JOIN sh.products p USING (prod_id);
```

```
| Id  | Operation                          | Name           | Rows  | Bytes |TempSpc| Cost |
|---|---|---|---|---|---|---|
|   0 | SELECT STATEMENT                   |                |  1088G|   411T|       |  4432|
|*  1 |  HASH JOIN                         |                |  1088G|   411T|       |  4432|
|   2 |   TABLE ACCESS FULL                | PRODUCTS       |    72 | 12456 |       |     3|
|*  3 |   HASH JOIN                        |                |  1088G|   240T|    10M|  4423|
|   4 |    TABLE ACCESS FULL               | CUSTOMERS      | 55500 |    10M|       |   415|
|   5 |    NESTED LOOPS                    |                |  1088G|    53T|       |  1006|
|   6 |     VIEW                           |                |  1000M|    23G|       |  5602|
|   7 |      SORT ORDER BY                 |                |  1000M|    15G|    26G|  5602|
|   8 |       PARTITION RANGE ALL          |                |  1000M|    15G|       |   406|
|   9 |        BITMAP CONVERSION TO ROWIDS |                |  1000M|    15G|       |   406|
|  10 |         BITMAP INDEX FAST FULL SCAN| SALES_CUST_BIX |       |       |       |      |
|  11 |     TABLE ACCESS BY USER ROWID     | SALES          |  1088 | 31552 |       |     1|
```

深呼吸，保持冷静。这并不像看起来那么复杂。让我们看一下清单 17-12 中的子查询`Q1`。我们将`SH.SALES`中的所有`ROWID`按`CUST_ID`排序。我们只需要访问`SALES_CUST_BIX`索引即可完成此操作。我们没有使用`INDEX FULL SCAN`，因为本地索引不提供我们需要的全局排序，因此`INDEX FAST FULL SCAN`更合适。请注意，由于我们只有两列：`CUST_ID`和`ROWID`，排序估计仅占用 26GB。一旦我们按正确顺序获得了`SH.SALES`中的`ROWID`列表，就可以通过使用`ROWID`来获取`SH.SALES`的其余列。获得`SH.SALES`的所有列后，我们可以使用嵌套循环连接访问维度表中的列。但是，使用哈希连接的右深连接树效率更高，因为逻辑读取次数更少。

为什么需要这种复杂的重写和所有这些提示？因为这是一个非常复杂的查询转换，并且正如我在第 14 章中解释的那样，CBO 无法执行专家 SQL 程序员能做的所有查询转换。实际上，如果我们添加`ORDER BY`子句，CBO 不会识别出行将按排序顺序产生，并将执行冗余排序。

另一种使用`ROWID`来提高排序性能的方法涉及`FIRST`和`LAST`函数。清单 17-13 展示了清单 17-3 中所示的`ROW_NUMBER`技术的简洁高效的替代方案。

清单 17-13. 在`LAST`函数中使用`ROWID`函数

```
WITH q1
     AS (SELECT c.*
               ,SUM (amount_sold) OVER (PARTITION BY cust_state_province)
                   total_province_sales
               ,ROW_NUMBER ()
                OVER (PARTITION BY cust_state_province
                       ORDER BY amount_sold DESC, c.cust_id DESC) rn
           FROM sh.sales s JOIN sh.customers c ON s.cust_id = c.cust_id)
SELECT *
  FROM q1
 WHERE rn = 1;
```

```
| Id  | Operation              | Name      | Rows  | Cost (%CPU)|
|---|---|---|---|---|
|   0 | SELECT STATEMENT       |           |   918K| 41847   (1)|
|*  1 |  VIEW                  |           |   918K| 41847   (1)|
|   2 |   WINDOW SORT          |           |   918K| 41847   (1)|
|*  3 |    HASH JOIN           |           |   918K|  2093   (1)|
|   4 |     TABLE ACCESS FULL  | CUSTOMERS | 55500 |   271   (1)|
|   5 |     PARTITION RANGE ALL|           |   918K|   334   (3)|
|   6 |      TABLE ACCESS FULL | SALES     |   918K|   334   (3)|
```

谓词信息（通过操作 ID 标识）：
```
1 - filter("RN"=1)
3 - access("S"."CUST_ID"="C"."CUST_ID")
```

```
WITH q1
     AS (  SELECT MIN (c.ROWID) KEEP (DENSE_RANK LAST ORDER BY amount_sold, cust_id)
                     cust_rowid
                 ,SUM (amount_sold) total_province_sales
             FROM sh.sales s JOIN sh.customers c USING (cust_id)
         GROUP BY cust_state_province)
SELECT c.*, q1.total_province_sales
  FROM q1 JOIN sh.customers c ON q1.cust_rowid = c.ROWID;
```

```
| Id  | Operation                   | Name      | Rows  | Cost (%CPU)|
|---|---|---|---|---|
|   0 | SELECT STATEMENT            |           | 80475 |  1837   (2)|
|   1 |  NESTED LOOPS               |           | 80475 |  1837   (2)|
|   2 |   VIEW                      |           |   145 |  1692   (3)|
|   3 |    SORT GROUP BY            |           |   145 |  1692   (3)|
|*  4 |     HASH JOIN               |           |   918K|  1671   (1)|
|   5 |      TABLE ACCESS FULL      | CUSTOMERS | 55500 |   271   (1)|
|   6 |      PARTITION RANGE ALL    |           |   918K|   334   (3)|
|   7 |       TABLE ACCESS FULL     | SALES     |   918K|   334   (3)|
|   8 |   TABLE ACCESS BY USER ROWID| CUSTOMERS |   555 |     1   (0)|
```

谓词信息（通过操作 ID 标识）：
```
4 - access("S"."CUST_ID"="C"."CUST_ID")
```

清单 17-13 中的第一个查询使用`ROW_NUMBER`分析函数来识别每个省份中单笔销售额最大的客户，并使用`SUM`分析函数计算该省份的总`AMOUNT_SOLD`。该查询效率低下，因为 918,843 行被排序，这些行包括来自`SH.CUSTOMERS`的所有 23 列。与清单 17-4 类似，使用`FIRST`或`LAST`函数将提高性能，因为聚合排序的工作区将只包含 141 个省份（估计值 145 略有偏差）的每一行，而不是分析排序的 918,843 行。

尽管有提高性能的潜力，但重新编写此查询的前景可能看起来令人生畏。看起来你需要编写 23 次`LAST`聚合函数的调用，对应`SH.CUSTOMERS`表中的每一列。然而，清单 17-13 中的第二个查询仅需要一次对`ROWID`的`LAST`调用。请注意，与`LAST`一起提供的`MIN (C.ROWID)`聚合本来可以是`MAX (C.ROWID)`甚至是`AVG (C.ROWID)`，结果不会改变，因为`ORDER BY`子句已将聚合的行数限制为 1。这种`ROWID`的用法节省了输入，并避免了因 23 次单独的`LAST`调用而产生的冗长难读的 SQL。清单 17-13 中第二个查询所展示的技术不仅简洁，还可能进一步提高性能，因为现在不仅排序的行数减少到 141 行，而且工作区中的列数也减少到只有 5 列：`CUST_ROWID`、`CUST_ID`、`AMOUNT_SOLD`、`CUST_STATE_PROVINCE`和`TOTAL_PROVINCE_SALES`。

## 解决分页问题

