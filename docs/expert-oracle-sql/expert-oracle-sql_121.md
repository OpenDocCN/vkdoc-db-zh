# 绑定变量

尽管 Oracle 在释放了绑定变量窥视带来的麻烦后，引入了自适应游标共享作为损害控制措施，但我仍然建议在涉及带有直方图的列的谓词中使用绑定变量时要谨慎。让我们把 `MOSTLY_BORING` 表恢复到 Listing 16-3 中的样子，看看当我们为 `SPECIAL_FLAG` 使用绑定变量时会发生什么。

Listing 16-6. 绑定变量的不恰当使用

```
/*  MOSTLY_BORING 表的设置参见 Listing 16-3 */

SELECT *
  FROM mostly_boring
 WHERE special_flag = :b1;

| Id  | Operation         | Name          | Rows  | Cost (%CPU)|
|   0 | SELECT STATEMENT  |               | 50000 |   384   (1)|
|*  1 |  TABLE ACCESS FULL| MOSTLY_BORING | 50000 |   384   (1)|
```


表中约有 100,000 行数据，且 `SPECIAL_FLAG` 有两个可能值，因此当使用 `EXPLAIN PLAN` 时，CBO 会假设清单 16-6 中的查询将返回 50,000 行。当然，实际上，如果我们运行该语句，基数估算和执行计划都将取决于所提供的绑定变量值。最终，无论每次执行时提供的绑定变量值如何，我们都将使用相同的计划来重复执行该语句。有几种方法可以解决此问题。你可以像我在清单 6-7 中所做的那样复制代码，或者可以使用动态 SQL 生成带有字面值的代码。这两种方法都会创建两个具有独立计划的不同 SQL 语句。由于只有两种不同的执行计划，你也可以只编写一个带有 `UNION ALL` 的语句。请暂时记住这一点，让我们先关注 `UNION ALL`。

## UNION, UNION ALL 与 OR

作为一本名为 *Expert Oracle SQL* 的书籍的读者，我相信你了解 `UNION`、`UNION ALL` 和 `OR` 在语义上的区别。无论幸运与否（这取决于你的视角），许多 SQL 是由对 SQL 性能问题了解相对较少的业务用户编写的，对他们来说，这些结构之间的区别并不清晰。清单 16-7 展示了三条返回相同结果但具有截然不同执行计划的语句。

清单 16-7. 三条结果相同但计划不同的语句

```sql
SELECT *
  FROM sh.customers
 WHERE cust_id = 3228
UNION ALL
SELECT *
  FROM sh.customers
 WHERE cust_id = 6783;
```

```
| Id  | Operation                    | Name         | Rows  | Cost (%CPU)|
|---  |---                           |---           |---    |---         |
|   0 | SELECT STATEMENT             |              |     2 |     4  (50)|
|   1 |  UNION-ALL                   |              |       |            |
|   2 |   TABLE ACCESS BY INDEX ROWID| CUSTOMERS    |     1 |     2   (0)|
|*  3 |    INDEX UNIQUE SCAN         | CUSTOMERS_PK |     1 |     1   (0)|
|   4 |   TABLE ACCESS BY INDEX ROWID| CUSTOMERS    |     1 |     2   (0)|
|*  5 |    INDEX UNIQUE SCAN         | CUSTOMERS_PK |     1 |     1   (0)|
```

```sql
SELECT *
  FROM sh.customers
 WHERE cust_id = 3228
UNION
SELECT *
  FROM sh.customers
 WHERE cust_id = 6783;
```

```
| Id  | Operation                     | Name         | Rows  | Cost (%CPU)|
|---  |---                            |---           |---    |---         |
|   0 | SELECT STATEMENT              |              |     2 |     4  (50)|
|   1 |  SORT UNIQUE                  |              |     2 |     4  (50)|
|   2 |   UNION-ALL                   |              |       |            |
|   3 |    TABLE ACCESS BY INDEX ROWID| CUSTOMERS    |     1 |     2   (0)|
|*  4 |     INDEX UNIQUE SCAN         | CUSTOMERS_PK |     1 |     1   (0)|
|   5 |    TABLE ACCESS BY INDEX ROWID| CUSTOMERS    |     1 |     2   (0)|
|*  6 |     INDEX UNIQUE SCAN         | CUSTOMERS_PK |     1 |     1   (0)|
```

```sql
SELECT *
  FROM sh.customers
 WHERE cust_id = 3228 OR cust_id = 6783;
```

```
| Id  | Operation                    | Name         | Rows  | Cost (%CPU)|
|---  |---                           |---           |---    |---         |
|   0 | SELECT STATEMENT             |              |     2 |     5   (0)|
|   1 |  INLIST ITERATOR             |              |       |            |
|   2 |   TABLE ACCESS BY INDEX ROWID| CUSTOMERS    |     2 |     5   (0)|
|*  3 |    INDEX UNIQUE SCAN         | CUSTOMERS_PK |     2 |     3   (0)|
```

清单 16-7 中的三条语句都返回两行——且是完全相同的两行。然而，这些语句的执行计划却各不相同。要理解其原因，请看清单 16-8 中的三个查询。

清单 16-8. UNION, UNION ALL 与 OR 产生不同结果

```sql
SELECT cust_first_name, cust_last_name
  FROM sh.customers
 WHERE cust_first_name = 'Abner'
UNION ALL
SELECT cust_first_name, cust_last_name
  FROM sh.customers
 WHERE cust_last_name = 'Everett';  -- 返回 144 行
```

```
| Id  | Operation          | Name      | Rows  | Cost (%CPU)|
|---  |---                 |---        |---    |---         |
|   0 | SELECT STATEMENT   |           |   173 |   845  (51)|
|   1 |  UNION-ALL         |           |       |            |
|*  2 |   TABLE ACCESS FULL| CUSTOMERS |    43 |   423   (1)|
|*  3 |   TABLE ACCESS FULL| CUSTOMERS |   130 |   423   (1)|

Predicate Information (identified by operation id):
   2 - filter("CUST_FIRST_NAME"='Abner')
   3 - filter("CUST_LAST_NAME"='Everett')
```

```sql
SELECT cust_first_name, cust_last_name
  FROM sh.customers
 WHERE cust_first_name = 'Abner'
UNION
SELECT cust_first_name, cust_last_name
  FROM sh.customers
 WHERE cust_last_name = 'Everett'; -- 返回 10 行
```

```
| Id  | Operation           | Name      | Rows  | Cost (%CPU)|
|---  |---                  |---        |---    |---         |
|   0 | SELECT STATEMENT    |           |   173 |   845  (51)|
|   1 |  SORT UNIQUE        |           |   173 |   845  (51)|
|   2 |   UNION-ALL         |           |       |            |
|*  3 |    TABLE ACCESS FULL| CUSTOMERS |    43 |   423   (1)|
|*  4 |    TABLE ACCESS FULL| CUSTOMERS |   130 |   423   (1)|

Predicate Information (identified by operation id):
   3 - filter("CUST_FIRST_NAME"='Abner')
   4 - filter("CUST_LAST_NAME"='Everett')
```

```sql
SELECT cust_first_name, cust_last_name
  FROM sh.customers
 WHERE cust_first_name = 'Abner' OR cust_last_name = 'Everett' -- 返回 128 行;
```

```
| Id  | Operation         | Name      | Rows  | Cost (%CPU)|
|---  |---                |---        |---    |---         |
|   0 | SELECT STATEMENT  |           |     1 |   423   (1)|
|*  1 |  TABLE ACCESS FULL| CUSTOMERS |     1 |   423   (1)|

Predicate Information (identified by operation id):
   1 - filter("CUST_FIRST_NAME"='Abner' AND "CUST_LAST_NAME"='Everett')
```

清单 16-8 中的所有三个查询都组合了来自 `SH.CUSTOMERS` 的两个行子集。第一个查询先返回 80 行匹配名字 `Abner` 的记录，然后返回 64 行匹配姓氏 `Everett` 的记录。第二个查询只返回 10 行。`UNION` 操作符会从选择列表中移除所有重复值，结果证明只有 10 个不同的名字和姓氏组合符合我们的谓词。第三个查询返回的行数又不同。`OR` 条件不会移除重复项，但它确保如果某一行同时匹配两个谓词，则只返回一次。由于 `SH.CUSTOMERS` 表中有 16 行数据的名字为 `Abner` 且姓氏为 `Everett`，这 16 行会被 `UNION ALL` 查询的两个部分返回，而被 `OR` 操作符只返回一次，因此 `UNION ALL` 查询的结果集比使用 `OR` 操作符的查询多出 16 行。

当已知 `UNION ALL` 能产生相同结果时，使用 `UNION` 绝非明智之举。`UNION` 只是先执行 `UNION ALL`，然后再执行一次 `SORT UNIQUE` 操作。如果不需要移除重复项，那么 `UNION` 所执行的排序就完全是不必要的开销，有时这种开销还相当显著。正如你在清单 16-8 中所看到的，`OR` 操作符只使用了一次全表扫描，而其他结构则需要两次。一般来说，当已知结果相同时，你应该使用 `OR` 操作符代替 `UNION` 或 `UNION ALL`。不过，让我给你看一个可能的例外情况。清单 16-9 回到了我们的 `MOSTLY_BORING` 表。

清单 16-9. 使用 UNION ALL 避免动态 SQL

```sql
SELECT *
  FROM mostly_boring
 WHERE special_flag = 'Y' AND :b1 = 'Y'
UNION ALL
SELECT *
  FROM mostly_boring
 WHERE special_flag = 'N' AND :b1 = 'N';
```

```
| Id  | Operation                             | Name          | Cost (%CPU)|
```



## 执行计划对比

```sql
|   0 | SELECT STATEMENT                      |               |   386 (100)|
|   1 |  UNION-ALL                            |               |            |
|   2 |   FILTER                              |               |            |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| MOSTLY_BORING |     2   (0)|
|   4 |     INDEX RANGE SCAN                  | SPECIAL_INDEX |     1   (0)|
|   5 |   FILTER                              |               |            |
|   6 |    TABLE ACCESS FULL                  | MOSTLY_BORING |   384   (1)|

Predicate Information (identified by operation id):

2 - filter(:B1='Y')
   4 - access("SPECIAL_FLAG"='Y')
   5 - filter(:B1='N')
   6 - filter("SPECIAL_FLAG"='N')

SELECT /*+ use_concat */
       *
  FROM mostly_boring
 WHERE    (special_flag = 'Y' AND :b1 = 'Y')
       OR (special_flag = 'N' AND :b1 = 'N');

| Id  | Operation                             | Name          | Cost (%CPU)|

|   0 | SELECT STATEMENT                      |               |   387   (1)|
|   1 |  CONCATENATION                        |               |            |
|*  2 |   FILTER                              |               |            |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| MOSTLY_BORING |     2   (0)|
|*  4 |     INDEX RANGE SCAN                  | SPECIAL_INDEX |     1   (0)|
|*  5 |   FILTER                              |               |            |
|*  6 |    TABLE ACCESS FULL                  | MOSTLY_BORING |   385   (1)|

Predicate Information(identified by operation id):

2 - filter(:B1='Y')
   4 - access("SPECIAL_FLAG"='Y')
   5 - filter(:B1='N')
   6 - filter("SPECIAL_FLAG"='N' AND (LNNVL(:B1='Y') OR
              LNNVL("SPECIAL_FLAG"='Y')))
```

清单 16-9 展示了如何在不使用动态 SQL 的情况下，将两种不同的访问路径合并到单个语句中。清单 16-9 中的第一个查询使用了 `UNION ALL` 集合运算符。该查询的前半部分仅在绑定变量的值为 `Y` 时才选择行。因此，选择了索引访问路径。第一个查询的后半部分仅在绑定变量的值为 `N` 时才返回行，因此全表扫描是合适的。第 2 行和第 5 行的 `FILTER` 操作确保只执行两个操作中的一个。

清单 16-9 中的第二个查询使用 `OR` 运算符实现了相同的效果，但需要一个 `USE_CONCAT` 提示；这是因为 CBO 没有意识到该语句的两个部分中只有一个会被执行，它将全表扫描的成本与索引访问路径的成本相加来得出语句的总成本。我们知道该语句只有一半会被运行，并且理解为什么 CBO 的成本计算是错误的。因此，我们可以放心地提供这个提示。

## 通用视图的问题

数据字典视图非常有用，定义越复杂，视图可能越有用。除了提高生产力和提升使用复杂视图的查询的可读性之外，当需求变更时，它还有可能避免在多处修改代码。

不幸的是，视图越复杂，就越有可能潜入低效问题。在近期的 Oracle 数据库版本中，由于引入了连接消除等转换，这种风险已大大降低，但问题仍然存在。清单 16-10 展示了一个典型示例。

### 清单 16-10. 复杂视图的低效问题

```sql
CREATE OR REPLACE VIEW sales_data
AS
   SELECT *
     FROM sh.sales
          JOIN sh.customers USING (cust_id)
          JOIN sh.products USING (prod_id);

SELECT prod_name, SUM (amount_sold)
    FROM sales_data
GROUP BY prod_name;

| Id  | Operation                 | Name         | Cost (%CPU)|

|   0 | SELECT STATEMENT          |              |   576   (6)|
|   1 |  HASH GROUP BY            |              |   576   (6)|
|*  2 |   HASH JOIN               |              |   576   (6)|
|   3 |    VIEW                   | VW_GBC_9     |   573   (6)|
|   4 |     HASH GROUP BY         |              |   573   (6)|
|*  5 |      HASH JOIN            |              |   552   (2)|
|   6 |       INDEX FAST FULL SCAN| CUSTOMERS_PK |    33   (0)|
|   7 |       PARTITION RANGE ALL |              |   517   (2)|
|   8 |        TABLE ACCESS FULL  | SALES        |   517   (2)|
|   9 |    TABLE ACCESS FULL      | PRODUCTS     |     3   (0)|

```

清单 16-10 首先定义了一个连接 `SH.SALES`、`SH.CUSTOMERS` 和 `SH.PRODUCTS` 的视图。你想要自己连接 `SH.SALES` 和 `SH.PRODUCTS`，并决定使用提供的视图。毕竟，你了解 CBO 具有的神奇的连接消除转换功能，并且确信使用该视图不会导致低效。然而，我们仍然可以看到执行计划中包含了与 `SH.CUSTOMERS` 表的连接。原因是确保 `SH.SALES` 中的每一行在 `SH.CUSTOMERS` 中都有对应行的引用完整性约束未被验证，因此连接消除是不合法的。要避免这个额外步骤，你需要自己连接 `SH.SALES` 和 `SH.PRODUCTS`。

## 如何使用临时表

我在清单 1-15 中展示了，你可以（并且通常应该）使用带因子的子查询来避免使用仅单个 SQL 语句需要的临时表。理论上，当可以使用临时表的索引时，临时表可能比带因子的子查询提供更好的性能。然而，这种理论上的可能性非常罕见，因为构建和维护索引的成本通常超过该索引可能为单个语句带来的任何好处。

但是，如果同一个会话中的多个独立 SQL 语句使用了类似的构造，那么使用临时表可能是个好主意，以避免重复工作。清单 16-11 展示了一个急需优化的存储过程。

### 清单 16-11. 多个 SQL 语句中的重复构造

```sql
CREATE TABLE key_electronics_customers
(
   cust_id                  NUMBER PRIMARY KEY
  ,latest_sale_month        DATE
  ,total_electronics_sold   NUMBER (10, 2)
);

CREATE TABLE electronics_promotion_summary
(
   sales_month              DATE
  ,promo_id                 NUMBER
  ,total_electronics_sold   NUMBER (10, 2)
  ,PRIMARY KEY (sales_month, promo_id)
);

CREATE OR REPLACE PROCEDURE get_electronics_stats_v1 (p_sales_month DATE)
IS
   v_sales_month        CONSTANT DATE := TRUNC (p_sales_month, 'MM'); -- 合理性检查
   v_next_sales_month   CONSTANT DATE := ADD_MONTHS (v_sales_month, 1);
BEGIN
   --
   -- 识别本月在电子产品上消费超过 1000 的关键电子产品客户
   --
   MERGE INTO key_electronics_customers c
        USING (  SELECT cust_id, SUM (amount_sold) amount_sold
                   FROM sh.sales s JOIN sh.products p USING (prod_id)
                  WHERE     time_id >=v_sales_month
                        AND time_id < v_next_sales_month
                        AND prod_category = 'Electronics'
               GROUP BY cust_id
                 HAVING SUM (amount_sold) > 1000) t
           ON (c.cust_id = t.cust_id)
   WHEN MATCHED
   THEN
      UPDATE SET
         c.latest_sale_month = v_sales_month
        ,c.total_electronics_sold = t.amount_sold
   WHEN NOT MATCHED
   THEN
      INSERT     (cust_id, latest_sale_month, total_electronics_sold)
          VALUES (t.cust_id, v_sales_month, t.amount_sold);

--
   -- 删除近期活动较少的客户
   --
   DELETE FROM key_electronics_customers
         WHERE latest_sale_month < ADD_MONTHS (v_sales_month, -3);
```



```
   -- 现在为电子产品销售生成促销统计
   --
   MERGE INTO `electronics_promotion_summary` `p`
        USING (  SELECT `promo_id`, SUM (`amount_sold`) `amount_sold`
                   FROM `sh.sales` `s` JOIN `sh.products` `p` USING (`prod_id`)
                  WHERE     `time_id` >=`v_sales_month`
                        AND `time_id` < `v_next_sales_month`
                        AND `prod_category` = 'Electronics'
               GROUP BY `promo_id`) `t`
           ON (`p`.`promo_id` = `t`.`promo_id` AND `p`.`sales_month` = `v_sales_month`)
   WHEN MATCHED
   THEN
      UPDATE SET `p`.`total_electronics_sold` = `t`.`amount_sold`
   WHEN NOT MATCHED
   THEN
      INSERT     (`sales_month`, `promo_id`, `total_electronics_sold`)
          VALUES (`v_sales_month`, `t`.`promo_id`, `t`.`amount_sold`);
END `get_electronics_stats_v1`;
/
```

`代码清单 16-11` 创建了两个表和一个存储过程。`KEY_ELECTRONICS_CUSTOMERS` 用于识别在过去三个月中，至少有一个月在电子产品上花费超过 1000 货币单位的客户。`ELECTRONICS_PROMOTION_SUMMARY` 则汇总了每个月每个促销活动的电子产品销售总额。这些表由一个名为 `GET_ELECTRONICS_STATS_V1` 的存储过程维护，该过程从 `SH.SALES` 和 `SH.PRODUCTS` 表中获取数据并合并到这两个表中。

我们可以看到，两个 `MERGE` 语句以极其相似的方式访问了 `SH.SALES` 和 `SH.PRODUCTS` 表：它们的连接方式相同，都只选择特定月份电子产品的行，并且都执行某种聚合操作。我们可以通过执行一次公共操作并将结果保存到临时表中来优化此代码。`代码清单 16-12` 展示了这种方法。

## 通过使用临时表避免重复工作 (`代码清单 16-12`)

```sql
CREATE GLOBAL TEMPORARY TABLE `electronics_analysis_gtt`
(
   `cust_id`       NUMBER NOT NULL
  ,`promo_id`      NUMBER NOT NULL
  ,`time_id`       DATE
  ,`amount_sold`   NUMBER
) ON COMMIT DELETE ROWS;

CREATE OR REPLACE PROCEDURE `get_electronics_stats_v2` (`p_sales_month` DATE)
IS
   `v_sales_month`        CONSTANT DATE := TRUNC (`p_sales_month`, 'MM'); -- 合理性检查
   `v_next_sales_month`   CONSTANT DATE := ADD_MONTHS (`v_sales_month`, 1);
BEGIN
   --
   -- 创建半聚合数据以供后续使用
   --

DELETE FROM `electronics_analysis_gtt`; -- 以防万一

INSERT INTO `electronics_analysis_gtt`(`cust_id`
                                        ,`promo_id`
                                        ,`time_id`
                                        ,`amount_sold`)
        SELECT `cust_id`
              ,`promo_id`
              ,MAX (`time_id`) `time_id`
              ,SUM (`amount_sold`) `amount_sold`
          FROM `sh.sales` JOIN `sh.products` `p` USING (`prod_id`)
         WHERE     `time_id` >=`v_sales_month`
               AND `time_id` < `v_next_sales_month`
               AND `prod_category` = 'Electronics'
      GROUP BY `cust_id`, `promo_id`;

   --
   -- 识别本月在电子产品上消费超过 1000 的关键电子客户
   --
   MERGE INTO `key_electronics_customers` `c`
        USING (  SELECT `cust_id`, SUM (`amount_sold`) `amount_sold`
                   FROM `electronics_analysis_gtt`
               GROUP BY `cust_id`
                 HAVING SUM (`amount_sold`) > 1000) `t`
           ON (`c`.`cust_id` = `t`.`cust_id`)
   WHEN MATCHED
   THEN
      UPDATE SET
         `c`.`latest_sale_month` = `v_sales_month`
        ,`c`.`total_electronics_sold` = `t`.`amount_sold`
   WHEN NOT MATCHED
   THEN
      INSERT     (`cust_id`, `latest_sale_month`, `total_electronics_sold`)
          VALUES (`t`.`cust_id`, `v_sales_month`, `t`.`amount_sold`);

   --
   -- 删除近期活动较少的客户
   --
   DELETE FROM `key_electronics_customers`
         WHERE `latest_sale_month` < ADD_MONTHS (`v_sales_month`, -3);

   --
   -- 现在为电子产品销售生成促销统计
   --
   MERGE INTO `electronics_promotion_summary` `p`
        USING (  SELECT `promo_id`, SUM (`amount_sold`) `amount_sold`
                   FROM `electronics_analysis_gtt`
               GROUP BY `promo_id`) `t`
           ON (`p`.`promo_id` = `t`.`promo_id` AND `p`.`sales_month` = `v_sales_month`)
   WHEN MATCHED
   THEN
      UPDATE SET `p`.`total_electronics_sold` = `t`.`amount_sold`
   WHEN NOT MATCHED
   THEN
      INSERT     (`sales_month`, `promo_id`, `total_electronics_sold`)
          VALUES (`v_sales_month`, `t`.`promo_id`, `t`.`amount_sold`);
END `get_electronics_stats_v2`;
/
```

`代码清单 16-12` 首先创建了一个全局临时表 `ELECTRONICS_ANALYSIS_GTT`，用于存储半聚合数据，供我们升级后的存储过程 `GET_ELECTRONICS_STATS_V2` 使用。升级后的过程首先生成指定月份电子产品按 `CUST_ID` 和 `PROMO_ID` 组合聚合的数据。然后两个 `MERGE` 语句进一步聚合临时表中的数据。

如果您指定 2001 年 12 月 1 日运行该过程，临时表将包含 1071 行。因为我们最初只从 `SH.SALES` 表中匹配到 4132 行，所以这种优化带来的性能提升不易衡量，但在实际应用中，半聚合实现的数据缩减量可能大得多，效益也相当可观。

## 避免多个相似的子查询

正如 `代码清单 16-11` 中多个相似的聚合一样，许多 SQL 语句也包含多个相似的子查询。`代码清单 16-13` 展示了一个典型例子。

## 多个相似的子查询 (`代码清单 16-13`)

```sql
SELECT `p`.`prod_id`
      ,`p`.`prod_name`
      ,`p`.`prod_category`
      , (SELECT SUM (`amount_sold`)
           FROM `sh.sales` `s`
          WHERE `s`.`prod_id` = `p`.`prod_id`)
          `sum_amount_sold`
      , (SELECT SUM (`quantity_sold`)
           FROM `sh.sales` `s`
          WHERE `s`.`prod_id` = `p`.`prod_id`)
          `sum_quantity_sold`
  FROM `sh.products` `p`;
```

```
| Id  | Operation               | Name     | Cost (%CPU)|
|---|---|---|---|
|   0 | SELECT STATEMENT        |          |  1078   (6)|
|   1 |  HASH JOIN OUTER        |          |  1078   (6)|
|   2 |   HASH JOIN OUTER       |          |   541   (6)|
|   3 |    TABLE ACCESS FULL    | PRODUCTS |     3   (0)|
|   4 |    VIEW                 | VW_SSQ_2 |   538   (6)|
|   5 |     HASH GROUP BY       |          |   538   (6)|
|   6 |      PARTITION RANGE ALL|          |   517   (2)|
|   7 |       TABLE ACCESS FULL | SALES    |   517   (2)|
|   8 |   VIEW                  | VW_SSQ_1 |   537   (6)|
|   9 |    HASH GROUP BY        |          |   537   (6)|
|  10 |     PARTITION RANGE ALL |          |   516   (2)|
|  11 |      TABLE ACCESS FULL  | SALES    |   516   (2)|
```

`代码清单 16-13` 中的查询在选择列表中使用了两个相似的子查询来计算每个产品的销售总额和总数量。我展示了来自 12cR1 数据库的执行计划，我们可以看到 CBO 已经成功地将这两个子查询去相关化。不幸的是，在撰写本文时，CBO 无法合并这些子查询（如果未来 CBO 能够做到这一点，我不会感到惊讶），因此我们仍然需要对 `SH.SALES` 表进行两次全表扫描。我们可以看到，估计成本的主要部分就来自这两次重复的全表扫描。

实际上有几种不同的方法可以解决重复子查询的问题，但在这种情况下，我们有一个简单的解决方案。`代码清单 16-14` 展示了如何使用连接来避免重复工作。

## 使用连接避免重复子查询 (`代码清单 16-14`)



