# 空值感知反连接

*   `NOT IN ( ... )`
*   `NOT EXISTS ( ... )`
*   `!= ALL ( ... ), > ALL( ... ), < ALL ( ... ), <= ALL ( ... ), >=ALL ( ... )`

我们前一节展示的执行计划之所以可能，是因为 `CUSTOMERS` 表和 `COUNTRIES` 表中的 `COUNTRY_ID` 列都定义了 `NOT NULL` 约束。你能看出 Listing 11-15 中两个查询的区别吗？

**Listing 11-15. NOT EXISTS 与 NOT IN 对比**

```sql
SELECT COUNT (*)                             SELECT COUNT (*)
  FROM sh.customers c                         FROM sh.customers c
 WHERE NOT EXISTS                           WHERE c.cust_src_id NOT IN(SELECT prod_src_id
          (SELECT 1                                   FROM sh.products p);
             FROM sh.products p
            WHERE prod_src_id = cust_src_id);
-----------------------------------------        --------------------------------------------
| Id|  Operation            | Name      |        | Id| Operation                | Name      |
-----------------------------------------        --------------------------------------------
| 0 | SELECT STATEMENT      |           |        | 0 | SELECT STATEMENT         |           |
| 1 |  SORT AGGREGATE       |           |        | 1 |  SORT AGGREGATE          |           |
| 2 |   HASH JOIN RIGHT ANTI|           |        | 2 |   HASH JOIN RIGHT ANTI NA|           |
| 3 |    TABLE ACCESS FULL  | PRODUCTS  |        | 3 |    TABLE ACCESS FULL     | PRODUCTS  |
| 4 |    TABLE ACCESS FULL  | CUSTOMERS |        | 4 |    TABLE ACCESS FULL     | CUSTOMERS |
-----------------------------------------        --------------------------------------------
```

看不出来？如果你运行它们，会发现左边的查询返回 55,500，而右边的查询返回 0！这是因为有些产品的 `PROD_SRC_ID` 未指定（`NULL`）。如果一个值未指定，我们就无法断言任何值与它不相等。Listing 11-16 可能是一个更清晰的例子。

**Listing 11-16. 在 IN 列表中包含 NULL 的简单查询**

```sql
SELECT *
  FROM DUAL
 WHERE 1 NOT IN (NULL);
```

Listing 11-16 中的查询不返回任何行，因为如果我们不知道 `NULL` 的值，它可能是 1。

在 11gR1 之前，`CUST_SRC_ID` 可能为 `NULL` 或 `PROD_SRC_ID` 可能为 `NULL` 这种可能性，会阻止对 Listing 11-15 右侧查询的子查询解嵌套。

`HASH JOIN RIGHT ANTI NA` 是空值感知反连接的交换连接输入变体。所谓的单一空值感知反连接操作，例如 `HASH JOIN ANTI SNA` 和 `HASH JOIN RIGHT ANTI SNA` 操作，会在 `PROD_SRC_ID` 非空但 `CUST_SRC_ID` 可为空的情况下成为可能。有关空值感知反连接的更多细节，只需在你喜欢的搜索引擎中输入“Greg Rahn null-aware anti-join”。对空值感知反连接的讨论结束了与在串行执行计划中连接表相关的主题列表。但是，我们还有一些来自第 8 章关于并行执行计划的未竟事宜，现在我们可以处理了。让我们现在开始吧。

