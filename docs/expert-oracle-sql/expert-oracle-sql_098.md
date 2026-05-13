# 第 12 章

## 最终状态优化

当并行执行与连接操作结合使用时，有多种不同的分布方法可供选择，但在无法进行分区级连接的情况下，对探查行源的数据缓冲往往是性能的隐性威胁。

第 10 章 涵盖了访问行源的方法，本章则讨论了这些行源如何被连接。一个不幸的事实是，成本基优化器在选择访问和连接行源的方式时常常做出不恰当的决定，理解这些概念背后的计算机科学原理对于 SQL 调优至关重要。但仅仅理解访问和连接行源的机制是不够的。我们还需要理解 CBO 所遵循流程的其他方面。第 12 章 重点介绍优化过程，随后第 13 章 将探讨查询转换。

__________________

¹当涉及复合分区时，可以容忍一些差异。

我在第 2 章介绍 CBO 时解释过，CBO 通过两个过程为运行时引擎生成执行计划：查询转换和最终状态优化。本章我将介绍最终状态优化，而转换将是第 13 章的重点。换句话说，本章探讨的是在执行了任何查询转换之后，CBO 如何评估不同的候选执行计划。

在第 10 章和第 11 章中，我介绍了访问和连接查询中指定的行源的机制。在本章中，我将提供 CBO 用于确定最佳连接顺序、连接方法和访问方法的高级概述。我还将探讨 `inlist iteration`。严格来说，并行化是最终状态转换的一部分，但我在第 2 章和第 8 章中已将并行化作为一个单独的主题进行了讨论，因此这里无需赘述。

阅读本章时请记住，所涉及的主题都不是相互独立的。在确定连接顺序之前，你可能无法总是知道访问表的最佳索引（如果有的话）。但最佳的连接顺序可能又取决于被连接表的访问方法。这实际上意味着 CBO 必须进行多次评估。事实上，本章涵盖的整个最终状态优化过程，需要针对每个候选的查询转换重复进行！

幸运的是，正如我在第 2 章中所说，我们不需要了解 CBO 如何解决所有这些依赖关系的精细细节，也不需要了解它为了在合理时间内完成任务而采用的任何捷径。我们只需要对 CBO 的操作方式有足够的了解，以便当出现问题时，我们能够找出原因并想出解决办法。

最终状态优化的目标是找出成本最低的执行计划；换句话说，就是运行时引擎最有可能在最短时间内执行的执行计划。实现这一目标的主要方法是：

-   避免检索行后又将其丢弃
-   避免访问数据块后又丢弃块中的所有行
-   避免多次读取同一个数据块
-   使用多块读取而非单块读取来读取多个连续的数据块

避免浪费这一主题是本章的关键。让我们从 CBO 如何选择最优连接顺序开始。

### 连接顺序

确定 SQL 查询的最佳连接顺序变化多端，完全可以写一整本书来讨论。事实上，在 2003 年，Dan Tow 就这么做了！他由 O'Reilly 出版的《SQL Tuning》一书，几乎全部致力于描述一种系统化的方法来确定查询的最佳连接顺序。该书出版时，Oracle 尚未支持哈希连接输入交换或星型转换（我们将在第 13 章中讨论这些）。如今，连接顺序优化的主题比 2003 年时更加复杂。

幸运的是，如今连接顺序的问题在 Oracle SQL 性能专家需要担心的问题清单上排名已经相当靠后。这是因为，*如果*（这是一个很大的前提）CBO 能够对每次对象访问和每次连接做出准确的基数估计，那么 CBO 在确定连接顺序方面做得相当不错。CBO 所使用的连接顺序算法的可靠性，是 Wolfgang Breitling 著名的“通过基数反馈进行调优”论文的核心（我在第 6 章的不同上下文中提到过）。Wolfgang 的调优方法是纠正 CBO 做出的基数估计，然后祈祷 CBO 做出明智的决定。而如今，CBO 通常*确实*会做出明智的决定！

那么，是什么让一种连接顺序比另一种更好呢？请看清单 12-1，它提供了一个非常简单的例子。

**清单 12-1. 简单的连接顺序优化**

```sql
SELECT /*+ gather_plan_statistics */
       COUNT (*)
  FROM oe.order_items i JOIN oe.product_descriptions p USING (product_id)
 WHERE i.order_id = 2367 AND p.language_id = 'US';

SET LINES 200 PAGES 0

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC ROWS IOSTATS LAST'));

| Id  | Operation           | Name           | Starts | E-Rows | A-Rows |

|   0 | SELECT STATEMENT    |                |      1 |        |      1 |
|   1 |  SORT AGGREGATE     |                |      1 |      1 |      1 |
|   2 |   NESTED LOOPS      |                |      1 |     13 |      8 |
|*  3 |    INDEX RANGE SCAN | ORDER_ITEMS_UK |      1 |      8 |      8 |
|*  4 |    INDEX UNIQUE SCAN| PRD_DESC_PK    |      8 |      2 |      8 |

Predicate Information (identified by operation id):

3 - access("I"."ORDER_ID"=2367)
4 - access("I"."PRODUCT_ID"="P"."PRODUCT_ID" AND "P"."LANGUAGE_ID"='US')
```

清单 12-1 连接了示例模式`OE`中的`ORDER_ITEMS`和`PRODUCT_DESCRIPTIONS`表。查询对两个表都有选择谓词，并且在`PRODUCT_ID`上有一个等值连接谓词。查询中唯一的提示是`GATHER_PLAN_STATISTICS`，它用于收集运行时统计信息。

根据`DBMS_XPLAN.DISPLAY_CURSOR`显示的执行计划，CBO 决定采用以下连接顺序：`ORDER_ITEMS` `PRODUCT_DESCRIPTIONS`。CBO 估计从`ORDER_ITEMS`中会有 8 行匹配，并且由于操作 3 的`E-ROWS`和`A-ROWS`列相匹配，我们知道这是一个完全准确的猜测。尽管操作 4 是对`PRODUCT_DESCRIPTIONS`表索引的`INDEX UNIQUE SCAN`，但 CBO 假设可能会返回多于一行；预计有 13 行输入到第 1 行的`SORT AGGREGATE`操作，而不是实际返回的 8 行。这无关紧要。CBO 已经选择了正确的连接顺序。清单 12-2 让我们看看如果我们试图覆盖选定的连接顺序会发生什么。

**清单 12-2. 不恰当的连接顺序**

```sql
SELECT /*+ gather_plan_statistics leading(p)*/
       COUNT (*)
  FROM oe.order_items i JOIN oe.product_descriptions p USING (product_id)
 WHERE i.order_id = 2367 AND p.language_id = 'US';

SET LINES 200 PAGES 0
```


SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC ROWS IOSTATS LAST'));

| 标识 | 操作                     | 名称           | 开始次数 | 预估行数 | 实际行数 |
|---|---|---|---|---|---|
|   0 | SELECT STATEMENT       |                |      1 |        |      1 |
|   1 |  SORT AGGREGATE        |                |      1 |      1 |      1 |
|   2 |   NESTED LOOPS         |                |      1 |     13 |      8 |
|*  3 |    INDEX FAST FULL SCAN| PRD_DESC_PK    |      1 |    288 |    288 |
|*  4 |    INDEX UNIQUE SCAN   | ORDER_ITEMS_UK |    288 |      1 |      8 |

谓词信息（由操作标识识别）：
3 - filter("P"."LANGUAGE_ID"='US')
4 - access("I"."ORDER_ID"=2367 AND "I"."PRODUCT_ID"="P"."PRODUCT_ID")

清单 12-2 中的 `LEADING` 提示让我们看到连接顺序 `PRODUCT_DESCRIPTIONS` `ORDER_ITEMS` 是如何执行的。结果证明，有 288 行符合 `LANGUAGE_ID` 上的谓词，这再次被 CBO（基于成本的优化器）完美预测到。然而，在这 288 个产品描述中，有 280 个最终在第 2 行的 `NESTED LOOPS` 操作中被丢弃了。真是浪费。现在我们明白了为什么 `PRODUCT_DESCRIPTIONS` `ORDER_ITEMS` 是一个糟糕的连接顺序。

![image](img/sq.jpg) **提示** 如果你期望 CBO 选择计划 A，但它却选择了计划 B，而你又无法立即理解原因，可以尝试添加提示来强制使用计划 B。然后查看计划 B 的基数和成本估算。你很可能会发现其背后的道理。接着，你应该能看出“误解”发生在哪里。是你遗漏了什么，还是 CBO 犯了错？

我最近处理过一个查询，它从第一个行源返回了大约 1,000,000 行数据，与第二个行源连接后返回了 400,000,000 行，而在与第三个行源进行第二次连接后，基数又回落到 1,000,000 行。如果你看到这种效果（基数急剧上升然后下降），你应该强烈怀疑连接顺序不是最优的。

尽管已经过时，但仍然可以从 Tow 的书中学到一些东西。让我告诉你我从这本书中学到的最重要的两点。

*   确保第一个行源正确。在大多数情况下，这足以获得一个不错的执行计划。
*   作为引导行源的最佳选择通常是选择性最强的那个行源。这不一定是指行数最少的行源。

## 连接方法

第 11 章 解释了四种连接机制各自的优缺点，因此这里不再重复。不过，如果我之前在书中给读者留下了人类专家总能比 CBO 更聪明的印象，那么请看看清单 12-3。

**清单 12-3. 一个反直觉的连接方法**
```sql
SELECT /*+ gather_plan_statistics */ *
  FROM hr.employees e JOIN hr.departments d USING (department_id)
 WHERE e.job_id = 'SH_CLERK';

SET LINES 200 PAGES 0

SELECT *
  FROM TABLE (
          DBMS_XPLAN.display_cursor (format => 'BASIC ROWS IOSTATS LAST'));
```
| 标识 | 操作                             | 名称        | 开始次数 | 预估行数 | 实际行数 | 缓冲区 |
|---|---|---|---|---|---|---|
|   0 | SELECT STATEMENT                      |             |      1 |        |     20 |    5 |
|   1 |  MERGE JOIN                           |             |      1 |     20 |     20 |    5 |
|   2 |   TABLE ACCESS BY INDEX ROWID         | DEPARTMENTS |      1 |     27 |      6 |    2 |
|   3 |    INDEX FULL SCAN                    | DEPT_ID_PK  |      1 |     27 |      6 |    1 |
|*  4 |   SORT JOIN                           |             |      6 |     20 |     20 |    3 |
|   5 |    TABLE ACCESS BY INDEX ROWID BATCHED| EMPLOYEES   |      1 |     20 |     20 |    3 |
|*  6 |     INDEX RANGE SCAN                  | EMP_JOB_IX  |      1 |     20 |     20 |    1 |

谓词信息（由操作标识识别）：
4 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
   filter("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
6 - access("E"."JOB_ID"='SH_CLERK')

这个例子找出所有 `JOB_ID` 为 `SH_CLERK` 的员工，并列出他们所属的部门信息。我必须承认，就算再给我一个月的时间，我也绝不可能想出 CBO 制定的这个执行计划。然而，仔细检查后，它看起来相当不错！

为什么使用 `MERGE JOIN`？这些表都很小，而且两张表上都有 `DEPARTMENT_ID` 的索引，所以显然 `NESTED LOOPS` 连接更合适？我在第 11 章中宣扬的那些简单规则，似乎没有一条暗示 `MERGE JOIN` 是合适的。这是怎么回事？让我们深入看看。

操作 2 和 3 似乎只是加深了困惑。执行计划执行了一个没有过滤谓词的 `INDEX FULL SCAN`，然后访问表。这意味着操作 2 和 3 将访问表中的每一行。为什么不用 `TABLE FULL SCAN`？在继续阅读之前，请试着想一想。

操作 4、5 和 6 通过索引访问 `EMPLOYEES` 表中符合 `JOB_ID` 谓词的 20 行数据，并按照 `MERGE JOIN` 的要求对它们进行排序。这没有什么神秘的。但接下来发生了什么？嗯，`MERGE JOIN` 会为 `DEPARTMENTS` 表中的每一行，按顺序探测（probe）这 20 行 `EMPLOYEES` 数据。这就是通过 `INDEX FULL SCAN` 访问 `DEPARTMENTS` 表的关键：结果已经是按 `DEPARTMENT_ID` 排序的，不需要再排序了！不仅如此，事实证明，`JOB_ID` 值 `SH_CLERK` 的意思是“发货文员”，而所有发货文员都属于 `DEPARTMENT_ID` 为 50 的“Shipping”部门。因此，当运行时引擎查看 `DEPARTMENT_ID` 60（第六个部门）时，它发现 `EMPLOYEES` 表中已没有更多行需要探测，于是停止了对 `DEPARTMENTS` 表的进一步访问。这就是为什么 `A-ROWS` 列显示的值为 6；只访问了部门 10、20、30、40、50 和 60。

如果你感兴趣，我原本猜测会更好的执行计划是使用 `NESTED LOOPS` 连接。通过强制执行我认为更好的执行计划的技术，我证明了我的计划并不更好。请看清单 12-4。

**清单 12-4. 我设计的较差执行计划**
```sql
SELECT /*+ gather_plan_statistics leading(e) use_nl(d)*/
       *
  FROM hr.employees e JOIN hr.departments d USING (department_id)
 WHERE e.job_id = 'SH_CLERK';

SET LINES 200 PAGES 0

SELECT *
  FROM TABLE (
          DBMS_XPLAN.display_cursor (format => 'BASIC ROWS IOSTATS LAST'));
```
| 标识 | 操作                             | 名称        | 开始次数 | 预估行数 | 实际行数 | 缓冲区 |
|---|---|---|---|---|---|---|
|   0 | SELECT STATEMENT                      |             |      1 |        |     20 |    27 |
|   1 |  NESTED LOOPS                         |             |      1 |        |     20 |    27 |
|   2 |   NESTED LOOPS                        |             |      1 |     20 |     20 |     7 |
|   3 |    TABLE ACCESS BY INDEX ROWID BATCHED| EMPLOYEES   |      1 |     20 |     20 |     3 |
|*  4 |     INDEX RANGE SCAN                  | EMP_JOB_IX  |      1 |     20 |     20 |     1 |
|*  5 |    INDEX UNIQUE SCAN                  | DEPT_ID_PK  |     20 |      1 |     20 |     4 |
|   6 |   TABLE ACCESS BY INDEX ROWID         | DEPARTMENTS |     20 |      1 |     20 |    20 |

谓词信息（由操作标识识别）：
4 - access("E"."JOB_ID"='SH_CLERK')
   5 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")

你可以看到，`NESTED LOOPS` 连接对 `DEPT_ID_PK` 的查找重复了 20 次，导致总共产生了 27 次缓冲区访问，而不是清单 12-3 中的 5 次。



这一切都相当酷。是 CBO 算出来的吗？不是。你可以看到 CBO 完全没料到运行时引擎会在 `DEPARTMENT_ID` 60 处停止。我们之所以知道这点，是因为 `DEPARTMENTS` 表中有 27 行，而操作 2 的 `E-ROWS` 值是 27。所以 CBO 只是在猜测，而且猜得挺走运。但即便如此，CBO 的猜测也远比我凭借多年 SQL 调优经验做出的猜测要好得多。

![image](img/sq.jpg) **提示**  本书花了很多时间告诉你为什么 CBO 经常出错。当 CBO 犯下严重错误时，通常是因为它做决策所依据的信息要么完全缺失，要么大错特错。听听 Wolfgang 的建议吧。纠正 CBO 的误解，它通常会比你我联手工作做得更好。唯一的问题是，要纠正 CBO 所有的错误假设有时相当不切实际，正如我们将在第 14 章中看到的那样。

## 访问方法

正如你在第 10 章中所读到的，访问表的方式多种多样，但可能对查询性能产生最大影响（无论好坏）的访问方法选择，是那些与索引范围扫描相关的：如果有索引，应该选择哪一个？

仅仅因为连接方法是 `NESTED LOOPS`，并不意味着必须通过连接谓词上的索引来访问；同样，仅仅因为使用了 `HASH JOIN`，也不意味着探测表上必然进行全表扫描。看看清单 12-5，它使用了哈希连接但没有全表扫描。

### 清单 12-5. 无全表扫描的哈希连接

```sql
SELECT *
  FROM sh.sales s, sh.customers c
 WHERE     c.cust_marital_status = 'married'
       AND c.cust_gender = 'M'
       AND c.cust_year_of_birth = 1976
       AND c.cust_id = s.cust_id
       AND s.prod_id = 136;

| Id  | Operation                                   | Name                  | Rows  |

|   0 | SELECT STATEMENT                            |                       |    40 |
|*  1 |  HASH JOIN                                  |                       |    40 |
|   2 |   TABLE ACCESS BY INDEX ROWID BATCHED       | CUSTOMERS             |    38 |
|   3 |    BITMAP CONVERSION TO ROWIDS              |                       |       |
|   4 |     BITMAP AND                              |                       |       |
|*  5 |      BITMAP INDEX SINGLE VALUE              | CUSTOMERS_YOB_BIX     |       |
|*  6 |      BITMAP INDEX SINGLE VALUE              | CUSTOMERS_MARITAL_BIX |       |
|*  7 |      BITMAP INDEX SINGLE VALUE              | CUSTOMERS_GENDER_BIX  |       |
|   8 |   PARTITION RANGE ALL                       |                       |   710 |
|   9 |    TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES                 |   710 |
|  10 |     BITMAP CONVERSION TO ROWIDS             |                       |       |
|* 11 |      BITMAP INDEX SINGLE VALUE              | SALES_PROD_BIX        |       |

Predicate Information (identified by operation id):

1 - access("C"."CUST_ID"="S"."CUST_ID")
   5 - access("C"."CUST_YEAR_OF_BIRTH"=1976)
   6 - access("C"."CUST_MARITAL_STATUS"='married')
   7 - access("C"."CUST_GENDER"='M')
  11 - access("S"."PROD_ID"=136)
```

清单 12-5 中的查询在 `SH` 示例模式中对 `CUSTOMERS` 和 `SALES` 表执行连接。CBO 估计有 23 位已婚、出生于 1976 年的男性客户，并且在这 23 位客户中，针对产品 136 的销售记录有 24 条。基于这些假设，CBO 认为 `NESTED LOOPS` 连接需要探测 `SALES` 表 23 次。在具有 23 次探测的嵌套循环和需要全表扫描的哈希连接之间做选择可能难分伯仲，但在此例中，哈希连接看起来快得多。原因是 `CUSTOMERS` 表上存在**独立于连接**的选择性谓词，并且 `SALES` 表上有一个（`s.prod_id = 136`）**独立于连接**的谓词，因此即使使用哈希连接，`SALES` 表也可以通过索引访问。注意第 8 行的 `PARTITION RANGE ALL` 操作。无论使用何种连接方法，由于分区键 `TIME_ID` 上没有谓词，所以必须访问所有分区。

CBO 对合适连接方法和访问方法的分析无可挑剔，但碰巧实际有 284 位已婚、出生于 1976 年的男性客户，并且连接输入应该交换顺序，不过这次影响不大。

### 清单 12-6. 未索引连接列的嵌套循环

```sql
SELECT *
  FROM oe.order_items i1, oe.order_items i2
 WHERE     i1.quantity = i2.quantity
       AND i1.order_id = 2392
       AND i1.line_item_id = 4
       AND i2.product_id = 2462;

| Id  | Operation                            | Name            | Rows  |

|   0 | SELECT STATEMENT                     |                 |     1 |
|   1 |  NESTED LOOPS                        |                 |     1 |
|   2 |   TABLE ACCESS BY INDEX ROWID        | ORDER_ITEMS     |     1 |
|*  3 |    INDEX UNIQUE SCAN                 | ORDER_ITEMS_PK  |     1 |
|*  4 |   TABLE ACCESS BY INDEX ROWID BATCHED| ORDER_ITEMS     |     1 |
|*  5 |    INDEX RANGE SCAN                  | ITEM_PRODUCT_IX |     2 |

Predicate Information (identified by operation id):

3 - access("I1"."ORDER_ID"=2392 AND "I1"."LINE_ITEM_ID"=4)
   4 - filter("I1"."QUANTITY"="I2"."QUANTITY")
   5 - access("I2"."PRODUCT_ID"=2462)
```

清单 12-6 在 `OE` 示例模式中的 `ORDER_ITEMS` 表上执行了一个自连接。该查询查找与指定行项订单数量相匹配的订单项。连接列是 `QUANTITY`，它通常不出现在 `WHERE` 子句中，也没有被索引。这个查询可能有点不寻常，但或许可用于调查疑似数据录入问题。

与清单 12-5 类似，清单 12-6 中的两个行源都具有**独立于连接**的高度选择性谓词。这次，其中一个行源 `I1` 通过唯一索引访问，因此嵌套循环是合适的。

到目前为止，本章已经介绍了连接顺序、连接方法和访问方法。然而，在最终状态优化过程中，CBO 可能还需要考虑另外一两个因素。让我们从讨论带有 `IN` 列表的查询开始。

## IN 列表迭代

当查询的列（或由索引支持的多个列）上指定了 `IN` 列表时，CBO 需要选择使用多少列。看看清单 12-7 中的查询。

### 清单 12-7. IN 列表执行计划选项——无迭代

```sql
SELECT *
    FROM hr.employees e
   WHERE last_name = 'Grant' AND first_name IN ('Kimberely', 'Douglas')
ORDER BY last_name, first_name;

| Id  | Operation                   | Name        | Cost (%CPU)|
```



## SQL 执行计划分析

|   0 | SELECT STATEMENT            |             |     2   (0)|
|---|---|---|---|
|   1 |  TABLE ACCESS BY INDEX ROWID| `EMPLOYEES`   |     2   (0)|
|*  2 |   INDEX RANGE SCAN          | `EMP_NAME_IX` |     1   (0)|

谓词信息（由操作 ID 标识）：

```
2 - access("LAST_NAME"='Grant')
       filter("FIRST_NAME"='Douglas' OR "FIRST_NAME"='Kimberely')
```

`HR` 示例模式中 `EMPLOYEES` 表上的 `EMP_NAME_IX` 索引包含了 `LAST_NAME` 和 `FIRST_NAME` 两列。对于 `LAST_NAME = 'Grant'`，`FIRST_NAME` 的可能选择似乎不多，因此 CBO 选择在 `LAST_NAME` 上进行一次 `INDEX RANGE SCAN`，并过滤掉任何与为 `FIRST_NAME` 提供的两个值都不匹配的索引条目。假设存在大量不同的 `FIRST_NAME`——那么另一种执行计划可能更合适。请查看 清单 12-8 中带有提示的查询版本。

清单 12-8. 带迭代的 IN 列表执行计划选项

```sql
 SELECT /*+ num_index_keys(e emp_name_ix 2)*/
         *
    FROM hr.employees e
   WHERE last_name = 'Grant' AND first_name IN ('Kimberely','Douglas')
ORDER BY last_name,first_name ;
```

```
| Id  | Operation                    | Name        | Cost (%CPU)|
|---|---|---|---|
|   0 | SELECT STATEMENT             |             |     2   (0)|
|   1 |  INLIST ITERATOR             |             |            |
|   2 |   TABLE ACCESS BY INDEX ROWID| `EMPLOYEES`   |     2   (0)|
|*  3 |    INDEX RANGE SCAN          | `EMP_NAME_IX` |     1   (0)|
```

谓词信息（由操作 ID 标识）：

```
3 - access("LAST_NAME"='Grant' AND ("FIRST_NAME"='Douglas' OR
              "FIRST_NAME"='Kimberely'))
```

提示 `NUM_INDEX_KEYS` 可用于指示在存在 `IN` 列表时执行 `INDEX RANGE SCAN` 应使用多少列。提供的提示指定了使用两列。这意味着我们需要运行两次 `INDEX RANGE SCAN` 操作，由 `INLIST ITERATOR` 操作驱动。第一次 `INDEX RANGE SCAN` 使用 `LAST_NAME = 'Grant'` 和 `FIRST_NAME = 'Douglas'` 作为访问谓词，第二次 `INDEX RANGE SCAN` 使用 `LAST_NAME = 'Grant'` 和 `FIRST_NAME = 'Kimberly'` 作为访问谓词。就我个人而言，在这种情况下，我觉得 `DBMS_XPLAN.display` 输出中对访问谓词的描述不是特别有用，因此希望我的解释对您有所帮助。

请注意，在 清单 12-7 或 清单 12-8 的执行计划中都没有出现排序操作。在前一种情况下，索引条目已经按适当顺序排列。在后一种情况下，`IN` 列表的成员已被重新排序以避免排序的需要。在这个例子中，不同备选计划的估算成本几乎相同，但在更大的表上，迭代可能带来显著的效益。

## 总结

最终状态优化是我赋予 CBO 最为人熟知部分的名称：即处理连接顺序、连接方法和访问方法的部分。然而，最终状态优化也处理列表迭代。

然而，CBO 的能力远不止最终状态优化。CBO 还能够将您编写的查询转换成看起来与您提供的原始查询完全不同的形式。这个相当庞大的主题是 第 13 章 的重点。

