# 排序优化与避免排序

## 单遍排序与多遍排序
图 17-1 右侧的九个方框代表九个排序归档。方框中的数字代表已排序的行，尽管一个排序归档中通常远多于四行。一旦所有排序归档创建完毕，每个排序归档的前几个块会被读回内存，合并并输出。随着合并行的输出，内存被释放，可以从磁盘读入更多数据。这种排序方式被称为**单遍排序**，因为每一行只被写入磁盘并读回一次。

然而，如果我们第一遍产生了 9000 个排序归档，而不是 9 个，会发生什么？此时，你可能无法将来自所有 9000 个排序归档的哪怕一个块都装入工作区。在这种情况下，你只将一部分排序归档的前段读入内存并进行合并。合并后的行不会像单遍排序那样输出，而是再次写入磁盘，创建另一个更大的排序归档。图 17-1 展示了从第一遍产生的九个排序归档被分成三个子集，每个子集包含三个。想象一下，这三个子集各自包含来自第一遍的 3000 个排序归档，而不是图中显示的三个。每个子集被合并，产生三个大的排序归档。这三个更大的排序归档现在可以合并以创建最终输出。这种排序变体被称为**两遍排序**，因为每一行被写入磁盘并读回两次。在第一遍中，每一行作为小排序归档的一部分写入磁盘；在第二遍中，该行被写入更大的、合并后的排序归档。

我们可以将这个思路推演到逻辑上的终点。如果第一遍结束时我们没有 9000 个排序归档，而是 9,000,000 个，会发生什么？现在在第二遍结束时，我们拥有的不是三个大的合并排序归档，而是 3000 个。此时我们遇到了大麻烦，因为可能需要通过创建特大型排序归档来进一步减少大排序归档的数量，才能产生最终的排序结果。这被称为**三遍排序**，我相信读者已经明白这个思路了。只要我们的临时表空间有足够的空间（以及无限的耐心），我们就可以用这种方式对任意大的数据集进行排序。

判断一次排序是单遍最优、单遍还是多遍的最简单方法是查看 `V$SQL_PLAN_STATISTICS_ALL` 中的 `LAST_EXECUTION` 列。与文档相反，该列会告诉你多遍排序中确切的遍数。不过你可能不太关心这个：如果你无法用单遍完成排序，你可能会耗尽时间和耐心。

如果你遇到了多遍排序，你第一个想法可能是尝试为你的工作区分配更多内存。`V$SQL_PLAN_STATISTICS_ALL` 中的 `ESTIMATED_OPTIMAL_SIZE` 和 `ESTIMATED_ONEPASS_SIZE` 列大致给出了你的工作区需要多大才能让排序作为最优或单遍排序执行。报告的单位是字节，如果你看到 1TB（并不罕见）或更大的数字，你需要想办法减少排序量。我稍后会讨论这一点，但偶尔你会发现稍微大一点的工作区就能解决问题。如果是这种情况，你可能会在你的 PL/SQL 代码中加入 代码清单 17-1 中的动态 SQL 行：

**代码清单 17-1. 手动增加重要排序的内存量**

```
BEGIN
   EXECUTE IMMEDIATE 'ALTER SESSION SET workarea_size_policy=manual';
   EXECUTE IMMEDIATE 'ALTER SESSION SET sort_area_size=2147483647';
END;
/
```

你应该仅对那些既重要又不可避免的大排序使用 代码清单 17-1 中的代码；你不能通过允许每个进程随意占用内存来神奇地创造更多内存。但一旦你决定打开闸门，半途而废就没有意义：如果你不需要完整的 2GB，它不会被分配。一旦你的重要排序完成，你的代码应该发出一条 `ALTER SESSION SET WORKAREA_SIZE_POLICY=AUTO` 语句来关闭内存分配的闸门。

## 避免排序
我希望所有关于多遍排序低效性的讨论能激励你避免不必要的排序。我已经在 代码清单 7-17 中向你展示了一种消除排序的方法。在本节中，我将再展示几个如何消除排序的例子。

### 非排序聚合函数
假设你想从 `SH.SALES` 表中提取每个 `PROD_ID` 的最大 `AMOUNT_SOLD` 值，并且想知道与该销售关联的 `CUST_ID`。代码清单 17-2 展示了一种你可能从其他书籍中学到的方法。

**代码清单 17-2. 使用 FIRST_VALUE 分析函数排序行**

```
WITH q1
     AS (SELECT prod_id
               ,amount_sold
               ,MAX (amount_sold) OVER (PARTITION BY prod_id) largest_sale
               ,FIRST_VALUE (
                   cust_id)
                OVER (PARTITION BY prod_id
                      ORDER BY amount_sold DESC, cust_id ASC)
                   largest_sale_customer
           FROM sh.sales)
SELECT DISTINCT prod_id, largest_sale, largest_sale_customer
  FROM q1
 WHERE amount_sold = largest_sale;
```

```
| Id  | Operation              | Name  | Rows  | Cost (%CPU)|
|---|---|---|---|---|
|   0 | SELECT STATEMENT       |       |    72 |  5071   (1)|
|   1 |  HASH UNIQUE           |       |    72 |  5071   (1)|
|   2 |   VIEW                 |       |   918K|  5050   (1)|
|   3 |    WINDOW SORT         |       |   918K|  5050   (1)|
|   4 |     PARTITION RANGE ALL|       |   918K|   517   (2)|
|   5 |      TABLE ACCESS FULL | SALES |   918K|   517   (2)|
```

在这个例子中，`FIRST_VALUE` 分析函数用于识别与每个产品最大 `AMOUNT_SOLD` 值关联的 `CUST_ID` 值（当行按 `AMOUNT_SOLD` 降序排序时的第一个值）。`FIRST_VALUE` 函数的一个问题是，当有多个 `CUST_ID` 值都与最大 `AMOUNT_SOLD` 值关联时，结果是不确定的。为了避免歧义，我们也按 `CUST_ID` 排序，以保证从合格的 `CUST_ID` 值集合中获得最低值。

从执行计划中可以看出，我们在因子化子查询中排序了所有 918,843 行，然后在主查询中丢弃了其中除 72 行外的所有行。这是一种非常低效的做法，我建议在大多数情况下不要使用 `FIRST_VALUE` 分析函数。我遇到的许多专业 SQL 程序员使用 代码清单 17-3 中的技巧试图改进。

**代码清单 17-3. 使用 ROW_NUMBER**

```
WITH q1
     AS (SELECT prod_id
               ,amount_sold
               ,cust_id
               ,ROW_NUMBER ()
                OVER (PARTITION BY prod_id
                      ORDER BY amount_sold DESC, cust_id ASC)
                   rn
           FROM sh.sales)
SELECT prod_id, amount_sold largest_sale, cust_id largest_sale_customer
  FROM q1
 WHERE rn = 1;
```

```
| Id  | Operation                | Name  | Rows  | Cost (%CPU)|
|---|---|---|---|---|
```



## 优化排序操作

|   0 | SELECT STATEMENT         |       |   918K|  5050   (1)|
|   1 |  VIEW                    |       |   918K|  5050   (1)|
|   2 |   WINDOW SORT PUSHED RANK|       |   918K|  5050   (1)|
|   3 |    PARTITION RANGE ALL   |       |   918K|   517   (2)|
|   4 |     TABLE ACCESS FULL    | SALES |   918K|   517   (2)|

`WINDOW SORT PUSHED RANK`操作听起来可能比`WINDOW SORT`更高效，但实际上并非如此。如果你做一个 10032 跟踪，你会看到`ROW_NUMBER`分析函数使用的是所谓的*version 1 sort*，这是一种内存密集型的、基于插入的排序，而`FIRST_VALUE`函数使用的是*version 2 sort*，这显然是基数排序和快速排序的巧妙组合¹，它需要的内存更少，比较次数也更少。

我们可以做得比这更好得多。看一下下面的示例。

示例 17-4. 使用 FIRST 聚合函数优化排序

```sql
 SELECT prod_id
        ,MAX (amount_sold) largest_sale
        ,MIN (cust_id) KEEP (DENSE_RANK FIRST ORDER BY amount_sold DESC)
            largest_sale_customer
    FROM sh.sales
GROUP BY prod_id;

| Id  | Operation            | Name  | Rows  | Cost (%CPU)|

|   0 | SELECT STATEMENT     |       |    72 |   538   (6)|
|   1 |  SORT GROUP BY       |       |    72 |   538   (6)|
|   2 |   PARTITION RANGE ALL|       |   918K|   517   (2)|
|   3 |    TABLE ACCESS FULL | SALES |   918K|   517   (2)|
```

`FIRST`聚合函数的名字与`FIRST_VALUE`非常相似，但相似之处仅此而已。`FIRST`只为 72 个产品中的每一个保留一行，因此只消耗极少的内存。性能的改进虽然不如估计成本所显示的那么显著，但仍然是可测量的，即使在示例模式的小数据量情况下也是如此。

在某些情况下，`FIRST`和`LAST`函数可以更高效，甚至完全避免排序！如果我们只想知道整个表中`AMOUNT_SOLD`的最大值，而不是每个产品的最大值，会怎样？下面的示例展示了结果。

示例 17-5. 使用 FIRST 聚合函数避免排序

```sql
SELECT MAX (amount_sold) largest_sale
      ,MIN (cust_id) KEEP (DENSE_RANK FIRST ORDER BY amount_sold DESC)
          largest_sale_customer
  FROM sh.sales;

| Id  | Operation            | Name  | Rows  | Cost (%CPU)|

|   0 | SELECT STATEMENT     |       |     1 |   517   (2)|
|   1 |  SORT AGGREGATE      |       |     1 |            |
|   2 |   PARTITION RANGE ALL|       |   918K|   517   (2)|
|   3 |    TABLE ACCESS FULL | SALES |   918K|   517   (2)|
```

当我们消除了`GROUP BY`子句，`SORT GROUP BY`操作就被替换成了`SORT AGGREGATE`操作。我相信你还记得，`SORT AGGREGATE`操作从不排序。

### 索引范围扫描与索引全扫描

索引基本上只是一个或多个列的值以及表 ROWID 的有序列表，有时我们可以利用这个预先排序的列表来避免查询中的排序。正如我在第 10 章中解释的，通过索引访问可能比全表扫描成本更高，所以这并不总是一个好主意。但要排序的数据量越大，通过消除排序来获得性能提升的潜力就越大。下面的示例展示了一个虚构的测试案例。

示例 17-6. 使用索引避免排序

```sql
 SELECT /*+ cardinality(s 1e9)*/
         s.*
    FROM sh.sales s
ORDER BY s.time_id;

| Id  | Operation                          | Name           | Rows  | Cost (%CPU)|

|   0 | SELECT STATEMENT                   |                |  1000M|  3049   (1)|
|   1 |  PARTITION RANGE ALL               |                |  1000M|  3049   (1)|
|   2 |   TABLE ACCESS BY LOCAL INDEX ROWID| SALES          |  1000M|  3049   (1)|
|   3 |    BITMAP CONVERSION TO ROWIDS     |                |       |            |
|   4 |     BITMAP INDEX FULL SCAN         | SALES_TIME_BIX |       |            |
```

示例使用了`cardinality`提示来观察如果`SH.SALES`表实际包含 1,000,000,000 行，并且我们想要按`TIME_ID`对所有行进行排序，CBO 会怎么做。我们可以使用`TIME_ID`上的索引来避免任何形式的排序，因为数据在索引中已经排序了。注意`PARTITION RANGE ALL`操作在执行计划中出现在其他操作*之上*。这是因为我们按分区键排序，而分区已经对我们的数据进行了部分排序，使得我们可以依次处理每个分区。

![image](img/sq.jpg) **提示**  按`LIST`分区表很常见，并且每个分区只有一个值。例如，我们可能按`LIST`重新分区`SH.SALES`，为每个`TIME_ID`设置一个分区。在这种情况下，只要分区顺序正确且*没有默认分区*，按分区列对多个分区进行排序仍然可以通过逐个处理分区来实现！

如果我们想按不同的列排序数据会怎样？下面的示例说明了为什么事情会变得有点复杂。

示例 17-7. 使用本地索引排序数据的失败尝试

```sql
 SELECT /*+ cardinality(s 1e9) index(s (cust_id))*/
         s.*
    FROM sh.sales s
ORDER BY s.cust_id;

| Id  | Operation                                   | Name           | Rows  | Cost (%CPU)|

|   0 | SELECT STATEMENT                            |                |  1000M|  8723K  (1)|
|   1 |  SORT ORDER BY                              |                |  1000M|  8723K  (1)|
|   2 |   PARTITION RANGE ALL                       |                |  1000M|   3472   (1)|
|   3 |    TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |  1000M|   3472   (1)|
|   4 |     BITMAP CONVERSION TO ROWIDS             |                |       |            |
|   5 |      BITMAP INDEX FULL SCAN                 | SALES_CUST_BIX |       |            |
```

示例使用了一个提示来尝试强制生成类似于示例的执行计划，但这当然行不通。本地索引`SALES_CUST_BIX`按顺序返回每个分区中的行，但这与对整个表的数据进行排序不是一回事——除了增加开销外，我们通过添加提示一无所获。当然，全局 B 树索引可以避免这个问题，但示例展示了另一种方法。

示例 17-8. 使用维度表避免全局索引

```sql
 SELECT /*+ cardinality(s 1e9) */
         s.*
    FROM sh.sales s , sh.customers c
   WHERE s.cust_id = c.cust_id
ORDER BY s.cust_id;

| Id  | Operation                          | Name           | Rows  | Cost (%CPU)|
```



