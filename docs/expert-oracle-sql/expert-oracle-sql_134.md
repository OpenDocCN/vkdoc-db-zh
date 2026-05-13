# 第 19 章 高级调优技术

![image](img/frontdot.jpg)

当我最初规划这本书时，我想象这一章会比现在的篇幅长得多。事实上，我在 SQL 调优中使用的大多数高级技术已经在前面的章节中讨论过了。我只是有几个教育性（即使比较晦涩）的案例需要讲一下。让我们从一些使用 ROWID 伪列的不同寻常的方法开始。

ROWID 伪列的可用性是 Oracle 数据库产品的一个显著特征。这个伪列可以用来绕过产品的许多特性。我使用*特性*这个词而不是一个更具贬义的名词，因为无意批评。事实上，CBO 必须在尽可能短的时间内做出执行计划的决定，考虑那些只有在极少数情况下才有意义的选项是没有意义的。

## 利用 `INDEX FAST FULL SCAN`

在第 10 章中，我解释了只有当从表中选择的所有列都在索引中时，才能使用 `INDEX FAST FULL SCAN` 访问方法。然而，有时这可能会成为一个麻烦，清单 19-1 展示了在极端情况下如何解决这个问题。

清单 19-1. 使用自连接来利用 INDEX FAST FULL SCAN

```
SELECT /*+ no_eliminate_join(t1) no_eliminate_join(t2) leading(t1) use_nl(t2) rowid(t2) */
       t2.*
  FROM sh.sales t1, sh.sales t2
 WHERE MOD (t1.cust_id, 10000) = 1 AND t2.ROWID = t1.ROWID;

| Id  | Operation                      | Name           | Rows  | Cost (%CPU)|
|---  |---                             |---             |---    |---         |
|   0 | SELECT STATEMENT               |                |  9188 |  9597   (1)|
|   1 |  NESTED LOOPS                  |                |  9188 |  97     (1)|
|   2 |   PARTITION RANGE ALL          |                |  9188 |   407   (0)|
|   3 |    BITMAP CONVERSION TO ROWIDS |                |  9188 |   407   (0)|
|   4 |     BITMAP INDEX FAST FULL SCAN| SALES_CUST_BIX |       |            |
|   5 |   TABLE ACCESS BY USER ROWID   | SALES          |     1 |     1   (0)|
```

清单 19-1 选择了 `SH.SALES` 的所有列，但设法使用了 `INDEX FAST FULL SCAN`。这里的要点是，我们使用了一个自连接，并且从别名为 `T1` 的 `SH.SALES` 副本中引用的唯一列是 `CUST_ID` 和 `ROWID`。由于不需要从 `T1` 引用表数据，因此可以使用 `INDEX FAST FULL SCAN`。然后，我们使用从 `SH.SALES` 的 `T1` 副本获得的 `ROWID` 来访问 `T2` 副本，并从中获取其余的列。


需要采用此类方法的情况极为罕见，这几乎肯定是 CBO（基于成本的优化器）不会考虑它的原因，除非显式编码了自连接。以清单 19-1 为例，我们只读取了三行（而非 CBO 假设的任意 1%选择性），但这三行的索引条目分散在索引中。`INDEX RANGE SCAN`（索引范围扫描）不可行，因此在这种特殊情况下，除了`INDEX FAST FULL SCAN`（索引快速全扫描）外，我们别无良策。

我们需要使用所有这些提示¹来获得我们想要的执行计划，原因有二：首先，对匹配`MOD`函数调用的行数的估计过高；其次，如我在第 13 章中所解释的，连接消除是一种启发式优化器转换。请记住，即使启发式转换会导致估计成本更高的执行计划，它们也会被应用。

由于`SALES_CUST_BIX`索引较小，您可能会发现，使用传统的`INDEX FULL SCAN`（索引全扫描）而不进行自连接，其性能与清单 19-1 相当。但当索引很大且未缓存时，使用`INDEX FAST FULL SCAN`和自连接应该能显著提升性能。

`INDEX FAST FULL SCAN`并非唯一一种我们能够通过自连接来规避其限制的访问方法。现在，让我兑现我在第 13 章中做出的承诺，向您展示在极端情况下，Oracle 数据库标准版用户如何模拟星型转换。

## 模拟星型转换

Oracle 数据库标准版用户无法使用位图索引或分区表。清单 19-2 首先将`SH.SALES`表复制到当前模式中的一个未分区表，然后重写了清单 13-35 中的查询。

**清单 19-2.** 面向标准版用户的星型转换

```
CREATE TABLE t1
AS
     SELECT *
       FROM sh.sales s
   ORDER BY time_id, cust_id, prod_id;

CREATE INDEX t1_cust_ix
   ON t1 (cust_id)
   COMPRESS 1;

CREATE INDEX t1_prod_ix
   ON t1 (prod_id)
   COMPRESS 1;

WITH q1
     AS (SELECT /*+ no_merge */
               c.cust_first_name, s.ROWID rid
           FROM sh.customers c JOIN t1 s USING (cust_id)
          WHERE c.cust_last_name = 'Everett')
    ,q2
     AS (SELECT /*+ no_merge */
               p.prod_name, s.ROWID rid
           FROM sh.products p JOIN t1 s USING (prod_id)
          WHERE p.prod_category = 'Electronics')
SELECT /*+ no_eliminate_join(s) leading(q1 q2 s) use_nl(s) */
      q2.prod_name
      ,q1.cust_first_name
      ,s.time_id
      ,s.amount_sold
  FROM q1 NATURAL JOIN q2 JOIN t1 s ON rid = s.ROWID;
```

```
| Id  | Operation                               | Name                 | Rows  | Cost (%CPU)|

|   0 | SELECT STATEMENT                        |                      |  1591 |  2420   (1)|
|   1 |  NESTED LOOPS                           |                      |  1591 |  2420   (1)|
|   2 |   HASH JOIN                             |                      |  1591 |   829   (1)|
|   3 |    VIEW                                 |                      |  7956 |   545   (1)|
|   4 |     NESTED LOOPS                        |                      |  7956 |   545   (1)|
|   5 |      TABLE ACCESS FULL                  | CUSTOMERS            |    61 |   423   (1)|
|   6 |      INDEX RANGE SCAN                   | T1_CUST_IX           |   130 |     2   (0)|
|   7 |    VIEW                                 |                      |   183K|   284   (1)|
|   8 |     NESTED LOOPS                        |                      |   183K|   284   (1)|
|   9 |      TABLE ACCESS BY INDEX ROWID BATCHED| PRODUCTS             |    14 |     3   (0)|
|  10 |       INDEX RANGE SCAN                  | PRODUCTS_PROD_CAT_IX |    14 |     1   (0)|
|  11 |      INDEX RANGE SCAN                   | T1_PROD_IX           | 12762 |    20   (0)|
|  12 |   TABLE ACCESS BY USER ROWID            | T1                   |     1 |     1   (0)|
```

清单 19-2 中的查询使用了两个具名子查询`Q1`和`Q2`。`Q1`将维度表`SH.CUSTOMERS`连接到事实表`T1`，以识别`T1`中匹配指定客户的`ROWID`。只需要用到`T1.CUST_ID`上的索引；无需访问`T1`本身。具名子查询`Q2`对维度表`SH.PRODUCTS`执行类似操作。主查询连接`Q1`和`Q2`，利用来自`T1`的`ROWID`来识别仅满足两组条件的`T1`行，然后再从`T1`中读取这些行。

和往常一样，我们必须在代码中加入大量提示来强制执行这个计划，而该计划在这个样本数据上效率相当低。需要注意的主要点是，该计划没有使用企业版特性，如分区或位图转换。

我必须承认，我已有多年没有与使用标准版的客户合作了，所以我从未在实际生产环境中使用过这种技术这一事实并不能说明什么。然而，从理论角度来看，我怀疑这种查询的合法用途极为罕见。为了解释原因，我将在下一节提供一个此类技术的更简单示例。

## 模拟索引连接

正如我在第 10 章中提到的，索引连接访问方法仅在查询引用的所有列都位于要连接的索引中时才有效。但正如我们在清单 19-2 中看到的，我们可以编写自己的基于`ROWID`的哈希连接。考虑到企业版用户可用的`INDEX_COMBINE`提示，清单 19-3 中使用的技术可能最吸引标准版用户。

**清单 19-3.** 模拟索引连接

```
WITH q1
     AS (SELECT /*+ no_merge */
                ROWID rid
           FROM t1
          WHERE cust_id = 462)
    ,q2
     AS (SELECT /*+ no_merge */
                ROWID rid
           FROM t1
          WHERE prod_id = 19)
SELECT /*+ leading(q1 q2) use_nl(t1) */
       t1.*
  FROM q1, q2, t1
 WHERE q1.rid = q2.rid AND q2.rid = t1.ROWID;
```

```
| Id  | Operation                   | Name       | Rows  | Cost (%CPU)|
```


```
| 0 | SELECT 语句                |            | 130 | 155 (0)|
| 1 | 嵌套循环                  |            | 130 | 155 (0)|
| 2 | 哈希连接                  |            | 130 | 25 (0)|
| 3 | 视图                      |            | 130 | 3 (0)|
| 4 | 索引范围扫描              | T1_CUST_IX | 130 | 3 (0)|
| 5 | 视图                      |            | 12762 | 22 (0)|
| 6 | 索引范围扫描              | T1_PROD_IX | 12762 | 22 (0)|
| 7 | 通过用户 ROWID 访问表       | T1         | 1 | 1 (0)|
```

代码清单 19-3 使用的技术本质上与代码清单 19-2 相同，只是我们消除了对维度表的任何引用。要使代码清单 19-2 和代码清单 19-3 中显示的执行计划达到最优，必须满足以下所有条件：

1.  `INDEX_COMBINE`提示的选项不可用。
2.  两个索引的选择性都不能很高，否则我们就不需要连接第二个索引了。
3.  两个索引的组合选择性必须很强，否则我们会使用全表扫描。
4.  访问更少表行所带来的成本节约，必须超过额外索引访问和哈希连接的开销。

理论上，有可能找到满足所有这些条件的情况。最难满足的是上面列表中的第四个也是最后一个条件，因为哈希连接的开销远大于与位图索引结合使用的`BITMAP AND`操作；选择性差的索引会产生一个巨大的哈希表，而且哈希连接甚至可能溢出到磁盘。

那么，为什么我要花时间讨论这些我可能从未使用过、你大概也不会用到的技术呢？有两个原因。首先，现存的大量旧代码使用了`AND_EQUAL`提示（这基本是同类的东西），你应该意识到这些提示很可能会适得其反。第二个原因是，还有另一种场景下，基于 ROWID 的自连接实际上可能很有用。现在让我们来看一下。

### 连接多列索引

截至版本 12.1.0.1，当引用的索引是具有共同前缀的多列索引时，`INDEX_JOIN`和`INDEX_COMBINE`提示都不起作用。代码清单 19-4 用两个都以`TIME_ID`列为前缀的多列索引，替换了我们在代码清单 19-2 中创建在`T1`上的两个索引。

**代码清单 19-4. 尝试组合多列索引**

```
DROP INDEX t1_cust_ix;
DROP INDEX t1_prod_ix;

CREATE INDEX t1_cust_ix2
   ON t1 (time_id, cust_id);

CREATE INDEX t1_prod_ix2
   ON t1 (time_id, prod_id);

SELECT /*+ index_join(t1) index_combine(t1) */
       COUNT (*)
  FROM t1
 WHERE time_id = DATE '2001-12-28' AND cust_id = 1673 AND prod_id = 44;

| Id  | Operation                            | Name        | Rows  | Cost (%CPU)|

|   0 | SELECT 语句                          |             |     1 |     4   (0)|
|   1 |  排序聚合                            |             |     1 |            |
|   2 |   通过索引 ROWID 批量访问表            | T1          |     1 |     4   (0)|
|   3 |    索引范围扫描                      | T1_CUST_IX2 |     6 |     3   (0)|
```

从代码清单 19-4 可以看到，尽管存在`INDEX_JOIN`和`INDEX_COMBINE`提示，我们最终还是访问了`T1`表本身。这两个提示都被认为是无效的。当我最近遇到这种情况时，我很庆幸自己理解如何手动组合索引。代码清单 19-5 展示了一种合法的实现自连接以模拟索引连接的方法。

**代码清单 19-5. 模拟索引连接以避免表访问**

```
WITH q1
     AS (SELECT /*+ no_merge */
                ROWID rid
           FROM t1
          WHERE time_id = DATE '2001-12-28' AND cust_id = 1673)
    ,q2
     AS (SELECT /*+ no_merge */
                ROWID rid
           FROM t1
          WHERE time_id = DATE '2001-12-28' AND prod_id = 44)
SELECT /*+ leading(q1 q2) use_nl(t1) */
       COUNT (*)
  FROM q1 NATURAL JOIN q2;

| Id  | Operation           | Name        | Rows  | Cost (%CPU)|

|   0 | SELECT 语句         |             |     1 |     6   (0)|
|   1 |  排序聚合           |             |     1 |            |
|   2 |   哈希连接          |             |     6 |     6   (0)|
|   3 |    视图             |             |     6 |     3   (0)|
|   4 |     索引范围扫描    | T1_CUST_IX2 |     6 |     3   (0)|
|   5 |    视图             |             |    25 |     3   (0)|
|   6 |     索引范围扫描    | T1_PROD_IX2 |    25 |     3   (0)|
```

由于`T1_CUST_IX2`索引的强选择性和强聚簇因子，代码清单 19-4 和代码清单 19-5 中的查询在性能上没有太大区别，但在实际应用中，收益可能相当可观，特别是当索引被缓存而表本身没有被缓存时。

关于`ROWID`伪列，我还有最后一个用例，那就是支持应用程序编码的并行执行。现在让我们来看看这个案例。

### 使用 ROWID 范围进行应用程序编码的并行执行

理解问题是本书的核心主题之一，当面对如代码清单 19-4 中的查询时，值得尝试弄清楚为什么首先需要这个查询。应用程序可能想知道满足特定条件的行数的一个可能原因，是为了确定执行某种并行处理所需的应用程序线程数量。

这种由应用程序驱动的并行处理非常普遍，也是`DBMS_PARALLEL_EXECUTE`包存在的理由。该包允许你识别`ROWID`的范围，称为块。其思路是每个块将由不同的应用程序线程处理。上述包的潜在问题是它考虑了表中的所有行，并且每个块处理的行数可能会差异巨大。

当我在代码清单 19-2 中创建表`T1`时，我按`TIME_ID`、`CUST_ID`和`PROD_ID`对行进行了排序，这反映了现实中很可能出现的数据聚簇。现在假设代码清单 19-5 中的查询返回了 3,000,000 行，而不是实际返回的 3 行，并且我们希望将这些行分成三批，每批 1,000,000 行，用于某种进一步处理。代码清单 19-6 应该能让你大致了解如何做到这一点。

**代码清单 19-6. 针对聚簇数据的应用程序驱动并行执行**

```
DECLARE
   CURSOR c1
   IS
      WITH q1
           AS (SELECT /*+ no_merge */
                      ROWID rid
                 FROM t1
                WHERE time_id = DATE '2001-12-28' AND cust_id > 1)
          ,q2
           AS (SELECT /*+ no_merge */
                      ROWID rid
                 FROM t1
                WHERE time_id = DATE '2001-12-28' AND prod_id > 1)
          ,q3
           AS (SELECT /*+ leading(q1 q2) use_nl(t1) */
                     rid, ROW_NUMBER () OVER (ORDER BY rid) rn
                 FROM q1 NATURAL JOIN q2)
        SELECT TRUNC ( (rn - 1) / 100) + 1 chunk
              ,MIN (rid) min_rowid
              ,MAX (rid) max_rowid
              ,COUNT (*) chunk_size
          FROM q3
      GROUP BY TRUNC ( (rn - 1) / 100);
```



游标 `c2` (
      `min_rowid`    `ROWID`
     ,`max_rowid`    `ROWID`)
   IS
      SELECT /*+ rowid(t1) */
             *
        FROM `t1`
       WHERE     `ROWID` BETWEEN `min_rowid` AND `max_rowid`
             AND `time_id` = DATE '2001-12-28'
             AND `prod_id` > 1
             AND `cust_id` > 1;
BEGIN
   FOR `r1` IN `c1`
   LOOP
      FOR `r2` IN `c2` (`r1`.`min_rowid`, `r1`.`max_rowid`)
      LOOP
         `NULL`;
      END LOOP;
   END LOOP;
END;
/

-- 游标 `C1` 的执行计划如下

| `Id`  | `Operation`             | `Name`        | `Rows`  | `Cost` (`%CPU`)|
| --- | --- | --- | --- | --- |
|   0 | `SELECT STATEMENT`      |             |   628 |     9   (0)|
|   1 |  `HASH GROUP BY`        |             |   628 |     9   (0)|
|   2 |   `VIEW`                |             |   628 |     9   (0)|
|   3 |    `WINDOW SORT`        |             |   628 |     9   (0)|
|   4 |     `HASH JOIN`         |             |   628 |     9   (0)|
|   5 |      `VIEW`             |             |   629 |     5   (0)|
|   6 |       `INDEX RANGE SCAN`| `T1_CUST_IX2` |   629 |     5   (0)|
|   7 |      `VIEW`             |             |   629 |     4   (0)|
|   8 |       `INDEX RANGE SCAN`| `T1_PROD_IX2` |   629 |     4   (0)|

谓词信息（由操作 ID 标识）：

4 - access(`"Q1"."RID"`=`"Q2"."RID"`)
   6 - access(`"TIME_ID"`=`TO_DATE`(' 2001-12-28 00:00:00', 'syyyy-mm-dd hh24:mi:ss') AND `"CUST_ID"`>1)
   8 - access(`"TIME_ID"`=`TO_DATE`(' 2001-12-28 00:00:00', 'syyyy-mm-dd hh24:mi:ss') AND `"PROD_ID"`>1)

-- 游标 `C2` 的执行计划如下

| `Id`  | `Operation`                    | `Name` | `Rows`  | `Cost` (`%CPU`)|
| --- | --- | --- | --- | --- |
|   0 | `SELECT STATEMENT`             |      |     2 |  1206   (1)|
|   1 |  `FILTER`                      |      |       |            |
|   2 |   `TABLE ACCESS BY ROWID RANGE`| `T1`   |     2 |  1206   (1)|

谓词信息（由操作 ID 标识）：

1 - filter(`CHARTOROWID`(:B2)>=`CHARTOROWID`(:B1))
   2 - access(`ROWID`>=`CHARTOROWID`(:B1) AND `ROWID`<=`CHARTOROWID`(:B2))
       filter(`"TIME_ID"`=`TO_DATE`(' 2001-12-28 00:00:00', 'syyyy-mm-dd hh24:mi:ss') AND `"PROD_ID"`>1 AND `"CUST_ID"`>1)

`清单 19-6` 展示了一个包含两个游标的 PL/SQL 块。游标 `C1` 中的查询标识了 `T1` 表中符合我们选择谓词的行（我略微修改了谓词，使得返回 1,196 行而不是 3 行），然后将它们分组为最多 100 行一批的块。查询结果中的 12 个块中有 11 个与 `T1` 中的 100 行相关，最后一个块与 96 行相关。每个块的最小和最大 ROWID 被标识出来。
PL/SQL 块依次执行游标 `C2` 共 12 次，对应于游标 `C1` 标识出的 12 个块各一次。在现实生活中，这些执行将由独立的应用线程并行完成。这样只扫描表中包含我们聚集数据的那一小部分，对于更大数量的行，线程之间不会在 `T1` 的块上竞争，并且可以使用多块读取来高效地读取聚集的行。
请注意，游标 `C2` 必须包含原始的谓词。并非 ROWID 范围内的所有行都保证来自我们的结果集。
关于 ROWID 伪列我想说的就这么多。现在我想换个话题，谈谈另一个听起来太过离奇而不像真实的 SQL 调优经历。

## 将内部连接转换为外部连接

你可能认为本节标题有个印刷错误。其实没有！这是一个真实经历的故事，涉及约翰——我在前一章介绍过的、从事风险管理的客户。
约翰（你可能猜到了，这是一个复合角色）又带着一个性能不佳的查询来找我。这个查询涉及一个在数据字典中定义的视图。`清单 19-7` 展示了其基本思路。

`清单 19-7`. 涉及数据字典视图的次优执行计划

```
CREATE OR REPLACE VIEW `sales_simple`
AS
   SELECT `cust_id`
         ,`time_id`
         ,`promo_id`
         ,`amount_sold`
         ,`p`.`prod_id`
         ,`prod_name`
     FROM `sh`.`sales` `s`, `sh`.`products` `p`
    WHERE `s`.`prod_id` = `p`.`prod_id`;

SELECT *
  FROM `sales_simple` `v` JOIN `sh`.`customers` `c` ON `c`.`cust_id` = `v`.`cust_id`
 WHERE     `v`.`time_id` = DATE '1998-03-31'
       AND `c`.`cust_first_name` = 'Madison'
       AND `cust_last_name` = 'Roy'
       AND `cust_gender` = 'M';
```

| `Id`  | `Operation`                                     | `Name`           | `Rows`  | `Cost` (`%CPU`)|
| --- | --- | --- | --- | --- |
|   0 | `SELECT STATEMENT`                              |                |     1 |   448   (1)|
|   1 |  `NESTED LOOPS`                                 |                |       |            |
|   2 |   `NESTED LOOPS`                                |                |     1 |   448   (1)|
|   3 |    `HASH JOIN`                                  |                |     1 |   447   (1)|
|   4 |     `TABLE ACCESS FULL`                         | `CUSTOMERS`      |     1 |   423   (1)|
|   5 |     `PARTITION RANGE SINGLE`                    |                |  1188 |    24   (0)|
|   6 |      `TABLE ACCESS BY LOCAL INDEX ROWID BATCHED`| `SALES`          |  1188 |    24   (0)|
|   7 |       `BITMAP CONVERSION TO ROWIDS`             |                |       |            |
|   8 |        `BITMAP INDEX SINGLE VALUE`              | `SALES_TIME_BIX` |       |            |
|   9 |    `INDEX UNIQUE SCAN`                          | `PRODUCTS_PK`    |     1 |     0   (0)|
|  10 |   `TABLE ACCESS BY INDEX ROWID`                 | `PRODUCTS`       |     1 |     1   (0)|

`清单 19-7` 中的查询是次优的，因为 `SH.CUSTOMERS` 表中只有一行符合提供的谓词条件。CBO 本应选择一种连接顺序，使其能够通过 `SALES_CUST_BIX` 索引访问 `SH.SALES` 表，但出于一些并非显而易见的原因，它没有这样做。别担心——几个提示（hint）应该就能解决问题，如 `清单 19-8` 所示。

`清单 19-8`. 在调用代码中使用提示的数据字典查询

```
CREATE OR REPLACE VIEW `sales_simple`
AS
   SELECT /*+ qb_name(q1)  */
         `cust_id`
         ,`time_id`
         ,`promo_id`
         ,`amount_sold`
         ,`p`.`prod_id`
         ,`prod_name`
     FROM `sh`.`sales` `s`, `sh`.`products` `p`
    WHERE `s`.`prod_id` = `p`.`prod_id`;

SELECT /*+ merge(q1) leading(c s@q1 p@q1) use_nl(s@q1) use_nl(p@q1) index(s@q1 (cust_id))*/
       *
  FROM `sales_simple` `v` JOIN `sh`.`customers` `c` ON `c`.`cust_id` = `v`.`cust_id`
 WHERE     `v`.`time_id` = DATE '1998-03-31'
       AND `c`.`cust_first_name` = 'Madison'
       AND `cust_last_name` = 'Roy'
       AND `cust_gender` = 'M';
```

| `Id`  | `Operation`                                     | `Name`           | `Rows`  | `Cost` (`%CPU`)|
| --- | --- | --- | --- | --- |
|   0 | `SELECT STATEMENT`                              |                |     1 |   854   (1)|
|   1 |  `NESTED LOOPS`                                 |                |       |            |
|   2 |   `NESTED LOOPS`                                |                |     1 |   854   (1)|
|   3 |    `NESTED LOOPS`                               |                |     1 |   853   (1)|
|   4 |     `TABLE ACCESS FULL`                         | `CUSTOMERS`      |     1 |   423   (1)|
|   5 |     `PARTITION RANGE SINGLE`                    |                |     1 |   853   (1)|
|   6 |      `TABLE ACCESS BY LOCAL INDEX ROWID BATCHED`| `SALES`          |     1 |   853   (1)|
|   7 |       `BITMAP CONVERSION TO ROWIDS`             |                |       |            |
|   8 |        `BITMAP INDEX SINGLE VALUE`              | `SALES_CUST_BIX` |       |            |
|   9 |    `INDEX UNIQUE SCAN`                          | `PRODUCTS_PK`    |     1 |     0   (0)|
|  10 |   `TABLE ACCESS BY INDEX ROWID`                 | `PRODUCTS`       |     1 |     1   (0)|



清单 19-8 中的提示有点难以阅读，但它们确实有效：我们通过`SALES_CUST_BIX`索引访问`SH.SALES`表。在这个阶段，John 提醒我，他正在使用他的数据可视化工具，因此所有提示都必须放入视图中。我本可以使用全局提示，或者定义一个包含与`SH.CUSTOMERS`表连接的“包装器”视图。但由于 John 希望有一定的灵活性来更改他的查询，这两种选项都不太吸引人。还有别的办法吗？

我想到，如果阻止视图合并，我或许能实现我的目标。清单 19-9 是我的第一次尝试。

清单 19-9. 尝试将谓词推入简单视图

```
CREATE OR REPLACE VIEW sales_simple
AS
   SELECT /*+ no_merge leading(s p) push_pred index(s (cust_id)) */
         cust_id
         ,time_id
         ,promo_id
         ,amount_sold
         ,p.prod_id
         ,prod_name
     FROM sh.sales s, sh.products p
    WHERE s.prod_id = p.prod_id;

SELECT *
  FROM sales_simple v JOIN sh.customers c ON c.cust_id = v.cust_id
 WHERE     v.time_id = DATE '1998-03-31'
       AND c.cust_first_name = 'Madison'
       AND cust_last_name = 'Roy'
       AND cust_gender = 'M';

| Id  | Operation                                     | Name           | Rows  | Cost (%CPU)|

|   0 | SELECT STATEMENT                              |                |     1 |   450   (1)|
|*  1 |  HASH JOIN                                    |                |     1 |   450   (1)|
|*  2 |   TABLE ACCESS FULL                           | CUSTOMERS      |     1 |   423   (1)|
|   3 |   VIEW                                        | SALES_SIMPLE   |  1188 |    27   (0)|
|*  4 |    HASH JOIN                                  |                |  1188 |    27   (0)|
|   5 |     PARTITION RANGE SINGLE                    |                |  1188 |    24   (0)|
|   6 |      TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |  1188 |    24   (0)|
|   7 |       BITMAP CONVERSION TO ROWIDS             |                |       |            |
|*  8 |        BITMAP INDEX SINGLE VALUE              | SALES_TIME_BIX |       |            |
|   9 |     TABLE ACCESS FULL                         | PRODUCTS       |    72 |     3   (0)|

Predicate Information (identified by operation id):

1 - access("C"."CUST_ID"="V"."CUST_ID")
   2 - filter("C"."CUST_FIRST_NAME"='Madison' AND "C"."CUST_LAST_NAME"='Roy' AND
              "C"."CUST_GENDER"='M')
   4 - access("S"."PROD_ID"="P"."PROD_ID")
   8 - access("TIME_ID"=TO_DATE(' 1998-03-31 00:00:00', 'syyyy-mm-dd hh24:mi:ss'))
```

清单 19-9 中失败的优化尝试背后的想法如下：如果我们阻止视图合并，就可以强制查询使用`SALES_CUST_BIX`索引，方法是利用`PUSH_PRED`提示将谓词显式推入视图。嗯，我所做的大部分提示都实现了：视图没有被合并，并且选择了期望的连接顺序。但是，`SH.CUSTOMERS`上的谓词没有被推入视图中！这到底是为什么？

经过大约半小时的谷歌搜索，我找到了优化器团队的以下博客：

```
https://blogs.oracle.com/optimizer/entry/basics_of_join_predicate_pushdown_in_oracle
```

这篇博文解释了连接谓词下推（JPPD）转换将运行的情况，并列出了我在第 13 章中列出的条件。为了帮助你回忆，这是该列表：

*   UNION ALL/UNION 视图
*   外连接视图
*   反连接视图
*   半连接视图
*   DISTINCT 视图
*   GROUP-BY 视图

正如我在第 13 章中所解释的，这个条件列表与阻止简单视图合并的条件列表`几乎`完全相同。其思路似乎是：要么视图被合并，要么 JPPD 是合法的。但在我刚刚展示的列表中明显缺席的，是`NO_MERGE`提示的存在！现在我们遇到了我理解中一种未预料到的可能性：简单视图合并被抑制了，但 JPPD 却是非法的。

理解了这一点后，我重新定义了视图，然后请 John 在他的数据可视化工具中点击几个单选按钮，将查询更改为外连接。John 有些怀疑，指出在他的情况下查询结果不会改变，并暗示外连接不可能比内连接性能更好。但是，不入虎穴，焉得虎子。修改后的视图定义以及数据可视化工具生成的修改后的查询如清单 19-10 所示。

清单 19-10. 将谓词推入外连接

```
CREATE OR REPLACE VIEW sales_simple
AS
   SELECT /*+ no_merge no_index(s (time_id)) */
         cust_id
         ,time_id
         ,promo_id
         ,amount_sold
         ,p.prod_id
         ,prod_name
     FROM sh.sales s, sh.products p
    WHERE s.prod_id = p.prod_id;

SELECT *
  FROM sales_simple v
       RIGHT JOIN sh.customers c
          ON c.cust_id = v.cust_id AND v.time_id = DATE '1998-03-31'
 WHERE     c.cust_first_name = 'Madison'
       AND cust_last_name = 'Roy'
       AND cust_gender = 'M';

| Id  | Operation                                      | Name           | Rows  | Cost (%CPU)|

|   0 | SELECT STATEMENT                               |                |     1 |   427   (0)|
|   1 |  NESTED LOOPS OUTER                            |                |     1 |   427   (0)|
|*  2 |   TABLE ACCESS FULL                            | CUSTOMERS      |     1 |   423   (1)|
|   3 |   VIEW PUSHED PREDICATE                        | SALES_SIMPLE   |     1 |     5   (0)|
|   4 |    NESTED LOOPS                                |                |       |            |
|   5 |     NESTED LOOPS                               |                |     1 |     5   (0)|
|   6 |      PARTITION RANGE SINGLE                    |                |     1 |     4   (0)|
|*  7 |       TABLE ACCESS BY LOCAL INDEX ROWID BATCHED| SALES          |     1 |     4   (0)|
|   8 |        BITMAP CONVERSION TO ROWIDS             |                |       |            |
|*  9 |         BITMAP INDEX SINGLE VALUE              | SALES_CUST_BIX |       |            |
|* 10 |      INDEX UNIQUE SCAN                         | PRODUCTS_PK    |     1 |     0   (0)|
|  11 |     TABLE ACCESS BY INDEX ROWID                | PRODUCTS       |     1 |     1   (0)|

Predicate Information (identified by operation id):

2 - filter("C"."CUST_FIRST_NAME"='Madison' AND "C"."CUST_LAST_NAME"='Roy' AND
              "C"."CUST_GENDER"='M')
   7 - filter("TIME_ID"=TO_DATE(' 1998-03-31 00:00:00', 'syyyy-mm-dd hh24:mi:ss'))
   9 - access("CUST_ID"="C"."CUST_ID")
  10 - access("S"."PROD_ID"="P"."PROD_ID")
```

清单 19-10 中的查询已被修改为使用外连接。注意，我不得不将`TIME_ID`上的谓词包含在外连接条件中，以防止启发式的外连接转内连接转换。

现在我们有了外连接，JPPD 的条件已经满足，我只需要在视图中定义两个提示：`NO_MERGE`提示用于阻止视图合并，`NO_INDEX`提示用于阻止`SALES_TIME_BIX`索引不必要地与`SALES_CUST_BIX`索引结合使用！

我必须说，将内连接转换为外连接以提升性能，是我 SQL 调优职业生涯中最奇怪的经历之一，我认为以此结束本章关于高级技术的讨论是恰如其分的。

## 总结



