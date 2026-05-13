# 第 14 章 基数估算问题

## 列相关性

我在第 6 章介绍了列相关性的概念，在第 9 章的清单 9-11 中展示了两个名为`TRANSACTION_DATE`和`POSTING_DATE`的列之间的相关性。这两个日期列之间的相关性很容易看出，但在其他情况下可能就不那么明显了。考虑`SH`模式中`SALES`表的列：

*   `PROD_ID`
*   `CUST_ID`
*   `TIME_ID`
*   `PROMO_ID`
*   `QUANTITY_SOLD`
*   `AMOUNT_SOLD`

对于一个数据仓库表来说，这些列是相当真实的，尽管示例数据是人为构造的。请想象这个表属于一个酒商，它具有以下特征：

*   100 种葡萄酒
*   10,000 名客户
*   10 年的交易历史（假设`TIME_ID`有 10 个值）
*   每月有一次葡萄酒促销活动
*   售出了 10,000,000 瓶葡萄酒
*   每笔订单平均售出 10 瓶葡萄酒

这意味着表中有 1,000,000 行。

让我们看看表中哪些列是相互关联的。从`PROD_ID`和`CUST_ID`开始。乍一看，这些列似乎毫无关系。但是等等；假设我们想估算某一款特定葡萄酒的销售数量。我们知道有 100 种葡萄酒，所以选择率将是 1%，而基数估计（该葡萄酒的平均销售交易数）将是 1,000,000 的 1%，即 10,000。使用同样的逻辑，我们看到有 10,000 名客户，因此每位客户下了 0.01%的订单，这意味着平均每位客户在十年期间下了 100 笔订单。

那么现在我们来问这个问题：某位特定客户购买了多少瓶某款特定的葡萄酒？嗯，CBO 会假设每位客户为 100 种葡萄酒中的每一种都下了一笔订单。这不太可能是准确的，因为通常客户不会品尝每一种酒——他们会大量购买自己喜欢的酒。因此`CUST_ID`和`PROD_ID`是相关的。

![image](img/sq.jpg) **提示** 如果你意识到不会找到两个列的每一种可能组合对应的行，那么这两个列就是相关的，并且当同时存在对这两个列的谓词且没有扩展统计信息时，CBO 会低估基数。

每位客户每年都买酒吗？可能不是。客户来来去去，所以`CUST_ID`和`TIME_ID`是相关的。每位客户都会利用每一次促销活动吗？不太可能。大多数客户每次下单的金额可能相同或相似，并且几乎可以肯定，表中不会出现`CUST_ID`和`AMOUNT_SOLD`的每一种组合。所以`CUST_ID`与表中的每一个其他列都相关！那么`PROD_ID`呢？每种葡萄酒每年都会售出吗？不会：2012 年的酒不会在 2011 年或更早售出，而且售罄后就无法再获得。促销活动适用于每个产品吗？不适用。如果你继续思考下去，你会发现几乎每一列都与另一列相关。

我曾与一些应用程序合作过，其中几乎每一个关键执行计划中的基数估计都显著低于实际值，原因就是列相关性的影响。

## 统计信息反馈与`DBMS_STATS.SEED_COL_USAGE`特性

对于大多数复杂的应用程序来说，确定要创建的正确扩展统计信息集合几乎是不可能的，这一点优化器团队也意识到了，他们提供了两种解决这个问题的方法。

让我们从了解*统计信息反馈*开始，这是我在第 6 章提到过的基数反馈特性的扩展和重命名版本。如果你在 12cR1 或更高版本中遇到显著的基数错误，并且你没有通过将`OPTIMIZER_ADAPTIVE_FEATURES`设置为`FALSE`来禁用统计信息反馈，那么很可能会创建一个*SQL 计划指令*。SQL 计划指令的存在可能会、也可能不会引发以下一种或两种后果：

*   对该语句或将来执行的类似语句（无论是否由同一会话发出）进行动态采样。
*   下一次收集统计信息时，自动为某些列组创建扩展统计信息。

在我看来，这种动态自适应特性在生产环境中是不受欢迎的。正如我在第 6 章解释的那样，我们通常不希望未经测试的执行计划首次在生产环境中运行。但尽管这个概述很简短，我认为 SQL 计划指令在项目生命周期的早期阶段，用于在测试系统上识别扩展统计信息的需求是一个潜在有用的功能。在早期阶段使用动态采样对于识别扩展统计信息目前无法解决的问题（例如多表中的相关列）也很有用。你可以在 SQL 调优指南中阅读更多关于 SQL 计划指令的内容。

一个可能更容易管理的独立特性在 11gR2 中引入并支持，但直到 12cR1 才首次被记录。`DBMS_STATS.SEED_COL_USAGE`过程允许你在运行特定工作负载时收集关于列组统计信息需求的信息。清单 14-1 展示了基本方法。

**清单 14-1. 通过 DBMS_STATS.SEED_COL_USAGE 获取扩展统计信息**

```sql
BEGIN
   DBMS_STATS.reset_col_usage (ownname => 'SH', tabname => NULL); -- 可选

   DBMS_STATS.seed_col_usage (NULL, NULL, 3600);
END;
/

-- 等待一个多小时

DECLARE
   dummy   CLOB;
BEGIN
   FOR r
      IN (SELECT DISTINCT object_name
            FROM dba_objects, sys.col_group_usage$
           WHERE obj# = object_id AND owner = 'SH' AND object_type = 'TABLE')
   LOOP
      SELECT DBMS_STATS.create_extended_stats ('SH', r.object_name)
        INTO dummy
        FROM DUAL;
   END LOOP;
END;
/
```

清单 14-1 首先调用了未记录的`DBMS_STATS.RESET_COL_USAGE`过程来清除先前测试运行中遗留的关于`SH`模式的不具有代表性的数据，但这是可选的。下一个调用是`DBMS_STATS.SEED_COL_USAGE`，它开始监控 SQL 语句 3,600 秒。一小时过后，你就可以根据获得的列使用统计信息创建扩展统计信息了。请注意，对`DBMS_STATS.CREATE_EXTENDED_STATS`的调用省略了任何扩展的指定，因为它们是从数据字典中自动获取的。

你可以通过查看数据字典表`SYS.COL_GROUP_USAGE$`（如清单 14-1 所示）来找出需要扩展统计信息的表。作为替代方案，如果你传递`NULL`作为表名（与省略表名相对），你可以在一次调用中为整个`SH`模式创建必要的扩展。

现在让我离开相关列，转而讨论另一个主要的基数错误来源：函数。

## 函数

当你在谓词中包含函数调用时，CBO 通常无法知道该谓词的选择率。考虑清单 14-2：

**清单 14-2. 谓词中的函数调用**

```sql
SELECT *
  FROM some_table t1, another_table t2
 WHERE some_function (t1.some_column) = 1 AND t1.c1 = t2.c2;
```



假设 `SOME_TABLE` 的对象统计信息显示该表有 1,000,000 行。在应用过滤谓词 `some_function (t1.some_column) = 1` 后，`SOME_TABLE` 中会剩下多少行？你无法判断，对吧？CBO（基于成本的优化器）同样无法判断。如果从 `SOME_TABLE` 中只选择了一行或两行，那么使用与 `ANOTHER_TABLE` 的 `NESTED LOOPS`（嵌套循环）连接可能是合适的。但也有可能所有 1,000,000 行都被选中，这种情况下，`HASH JOIN`（哈希连接）将是避免灾难性性能的唯一方法。我在上面的例子中特意使用了无意义的表名和函数名。假设 清单 14-2 中的函数调用实际上是 `MOD (t1.some_column, 2)?` 那会怎样？既然 `MOD` 是 SQL 引擎提供的函数，CBO 当然应该意识到这个函数调用可能只会过滤掉大约一半的行？嗯，其实不然。SQL 语言提供了数十种内置函数，让 CBO 为每个函数都保存一套识别基数估计值的规则是不切实际的。CBO 唯一切实可行的做法是，像对待用户编写的函数一样对待它们。

当 CBO 无法确定基数时，它会使用一组固定的内置值之一进行任意猜测。对于函数调用，通常使用 1% 的选择性估计值。

## 过时的统计信息

这里我将简要说明。我已在第 6 章详细讨论过过时统计信息的主题，并将在第 20 章再次讨论。只需将过时的统计信息（在非 `TSTATS` 环境中）添加到导致基数错误的原因列表中即可。

## 愚蠢的数据类型

“愚蠢的数据类型”是《*Cost Based Oracle Fundamentals*》一书第 6 章中一个章节的标题。该章节完全致力于讨论基数错误的主题，我毫不掩饰地复制了该章节标题。清单 14-3 也几乎直接复制自 Jonathan Lewis 书中同一章节的内容。

清单 14-3. 愚蠢的数据类型

```sql
CREATE TABLE t2
AS
   SELECT d1
         ,TO_NUMBER (TO_CHAR (d1, 'yyyymmdd')) n1
         ,TO_CHAR (d1, 'yyyymmdd') c1
     FROM (SELECT TO_DATE ('31-Dec-1999') + ROWNUM d1
             FROM all_objects
            WHERE ROWNUM <= 1827);

SELECT *
  FROM t2
 WHERE d1 BETWEEN TO_DATE ('30-Dec-2002', 'dd-mon-yyyy')
              AND TO_DATE ('05-Jan-2003', 'dd-mon-yyyy');

| Id  | Operation         | Name | Rows  |

|   0 | SELECT STATEMENT  |      |     8 |
|*  1 |  TABLE ACCESS FULL| T2   |     8 |

SELECT *
  FROM t2
 WHERE n1 BETWEEN 20021230 AND 20030105;

| Id  | Operation         | Name | Rows  |

|   0 | SELECT STATEMENT  |      |   396 |
|*  1 |  TABLE ACCESS FULL| T2   |   396 |
```

清单 14-3 生成 2000 年 1 月 1 日到 2004 年 12 月 31 日之间的每一天一行数据。第一个查询使用日期列选取 2002 年 12 月 30 日到 2003 年 1 月 5 日（含首尾）之间的七行数据，其基数估计几乎是正确的。第二个查询使用了一个谓词，该谓词作用于一个以 `YYYYMMDD` 格式存储日期的数值列。此时的基数估计就偏差很大了，因为当被视为数字时，范围 20,021,230 到 20,030,105 似乎占了总范围 20,000,101 到 20,041,231 的相当大一部分。

我可以复制并粘贴 Jonathan 书中更多的内容，但没必要。关键不是试图记住一堆 CBO 出错的原因，而是培养一种思维方式，让你在需要时能够推断出问题所在。

我想关于基数错误我已经说完了我想说的。但基数错误并不是 CBO 出错的唯一原因。让我们继续讲述 CBO 问题的故事，将注意力转向缓存效应。

## 缓存效应

许多 Oracle 专家养成了使用术语 “cache” 作为 “buffer cache”（缓冲区缓存）缩写的习惯。我自己也经常这样做。然而，在 Oracle 数据库中有许多不同类型的缓存。我在第 4 章介绍了 OCI、结果和函数缓存，并在第 13 章相当详细地讨论了标量子查询缓存。所有这些缓存都会影响 SQL 语句的耗时和资源消耗。CBO 无法量化任何这些缓存带来的好处，因此在其计算中不会以任何方式考虑缓存效应。

我想讨论一个特定的案例，在这个案例中，CBO 有朝一日可能能够比目前考虑得更周全一点。它涉及到我在第 9 章解释过的索引聚簇因子统计信息。清单 14-4 展示了一个有趣的测试用例。

清单 14-4. 单列索引的聚簇缓存效应

```sql
ALTER SESSION SET statistics_level=all;

SET PAGES 900 LINES 200 SERVEROUTPUT OFF
COMMIT;
ALTER SESSION ENABLE PARALLEL DML;

CREATE TABLE t1
(
   n1       INT
  ,n2       INT
  ,filler   CHAR (10)
)
NOLOGGING;

INSERT /*+ parallel(t1 10) */
      INTO  t1
   WITH generator
        AS (    SELECT ROWNUM rn
                  FROM DUAL
            CONNECT BY LEVEL <= 4500)
   SELECT TRUNC (ROWNUM / 80000)
         ,ROWNUM + 5000 * (MOD (ROWNUM, 2))
         ,RPAD ('X', 10)
     FROM generator, generator;

COMMIT;

CREATE INDEX t1_n1
   ON t1 (n1)
   NOLOGGING
   PARALLEL 10;

COLUMN index_name FORMAT a10

SELECT index_name, clustering_factor
  FROM all_indexes
 WHERE index_name = 'T1_N1';
INDEX_NAME CLUSTERING_FACTOR
---------- -----------------
T1_N1                  74102

ALTER SYSTEM FLUSH BUFFER_CACHE;

SELECT MAX (filler)
  FROM t1
 WHERE n1 = 2;

SELECT *
  FROM TABLE (
          DBMS_XPLAN.display_cursor (NULL
                                    ,NULL
                                    ,'basic cost iostats -predicate'));

| Id  | Operation                 | Name  | Cost (%CPU)| A-Rows |A-Time   | Buffers | Reads  |

|   0 | SELECT STATEMENT          |       |   458 (100)|      1 | 00:00.58|     440 |    440 |
|   1 |  SORT AGGREGATE           |       |            |      1 |00:00.58 |     440 |    440 |
|   2 |   TABLE ACCESS BY INDEX RO| T1    |   458   (1)|  80000 |00:00.56 |     440 |    440 |
|   3 |    INDEX RANGE SCAN       | T1_N1 |   165   (0)|  80000 |00:00.21 |     159 |    159 |

ALTER SYSTEM FLUSH BUFFER_CACHE;

SELECT /*+ full(t1) */
       MAX (filler)
  FROM t1
 WHERE n1 = 2;

SELECT *
  FROM TABLE (
          DBMS_XPLAN.display_cursor (NULL
                                    ,NULL
                                    ,'basic cost iostats -predicate'));

| Id  | Operation          | Name | Cost (%CPU)| A-Rows |A-Time   | Buffers | Reads  |

|   0 | SELECT STATEMENT   |      | 19507 (100)|      1 | 00:08.67|   71596 |  71592 |
|   1 |  SORT AGGREGATE    |      |            |      1 |00:08.67 |   71596 |  71592 |
|   2 |   TABLE ACCESS FULL| T1   | 19507   (1)|  80000 |00:08.67 |   71596 |  71592 |
```

清单 14-4 创建了一个包含 20,025,000 行数据的表 `T1`。这些行是按照列 `N1` 排序添加的。然后在列 `N1` 上创建了一个单列索引 `T1_N1` 并收集了统计信息。该索引的聚簇因子相当低，为 74,102。为了公平比较，我在每个查询之前都刷新了缓冲区缓存。


当我们查询表以寻找具有特定 `N1` 值的行时，CBO 选择使用索引 `T1_N1`。当我们使用 hint 强制进行全表扫描时，我们看到成本要高得多，实际经过的时间也长得多。CBO 正确地得出结论：全表扫描会花费更长的时间。清单 14-5 展示了当我们向索引中添加第二个列时会发生什么。

清单 14-5. 使用多列索引的索引聚类缓存效应

```
DROP INDEX t1_n1;

CREATE  INDEX t1_n1_n2 ON t1(n1,n2);

select index_name,clustering_factor from all_indexes where index_name='T1_N1_N2';

INDEX_NAME CLUSTERING_FACTOR
---------- -----------------
T1_N1_N2            18991249

ALTER SYSTEM FLUSH BUFFER_CACHE;

SELECT MAX (filler)
  FROM t1
 WHERE n1 =2;

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (NULL,
                                         NULL,
                                         'basic cost iostats -predicate'));

| Id  | Operation          | Name | Cost (%CPU)| A-Rows |A-Time   | Buffers | Reads  |

|   0 | SELECT STATEMENT   |      | 19507 (100)|      1 | 00:09.78|   71596 |  71592 |
|   1 |  SORT AGGREGATE    |      |            |      1 |00:09.78 |   71596 |  71592 |
|   2 |   TABLE ACCESS FULL| T1   | 19507   (1)|  80000 |00:09.77 |   71596 |  71592 |

ALTER SYSTEM FLUSH BUFFER_CACHE;

SELECT /*+ index(t1 t1_n1_n2) */ MAX (filler)
  FROM t1
 WHERE n1 =2;

SELECT *
  FROM TABLE (DBMS_XPLAN.display_cursor (NULL,
                                         NULL,
                                         'basic cost iostats -predicate'));

| Id  | Operation                | Name     | Cost (%CPU)| A-Rows |A-Time   | Buffers | Reads  |

|   0 | SELECT STATEMENT         |          | 75014 (100)|      1 | 00:00.67|   74497 |    494 |
|   1 |  SORT AGGREGATE          |          |            |      1 |00:00.67 |   74497 |    494 |
|   2 |   TABLE ACCESS BY INDEX R| T1       | 75014   (1)|  80000 |00:00.65 |   74497 |    494 |
|   3 |    INDEX RANGE SCAN      | T1_N1_N2 |   231   (0)|  80000 |00:00.23 |     214 |    214 |
```

清单 14-5 删除了索引 `T1_N1`，并在列 `N1` 和 `N2` 上创建了索引 `T1_N1_N2`。当我们现在运行查询时，CBO 选择了全表扫描。当我们使用 hint 强制使用新索引时，我们看到 CBO 给多列索引操作赋予的成本比考虑使用单列索引 `T1_N1` 访问时高得多。为什么？`T1_N1_N2` 比 `T1_N1` 稍大，但这并不能解释成本的大幅增加。CBO 错了吗？

嗯，既对又不对。如果你查看索引操作的运行时统计信息，你会看到，事实上，一致读的数量大幅增加了（从 清单 14-4 中的 440 增加到 清单 14-5 中的 74,497），这正是 CBO 所怀疑的，因为索引中加入了 `N2`。逻辑 I/O 大量增加的原因是 `T1_N1_N2` 中的索引条目排序方式与 `T1_N1` 中的不同。我构造数据的方式使得 `T1_N1_N2` 中连续的索引条目几乎保证引用不同的表块，而 `T1_N1` 中大多数连续的索引条目引用相同的表块，并且可以作为同一个逻辑 I/O 操作的一部分进行处理。这种差异反映在 `T1_N1_N2` 的索引聚类因子统计信息中，该值为 18,991,249（几乎等于表中的行数，符合预期），远高于 `T1_N1` 的聚类因子。这解释了 CBO 如何得出对使用 `T1_N1_N2` 的查询的相应更高的成本估算。

尽管 CBO 正确地意识到通过 `T1_N1_N2` 进行索引访问涉及大量的逻辑 I/O 操作，但它没有意识到这些操作是针对同一小组表块的。因此，索引访问只有 494 次物理 I/O，经过的时间同样很低。全表扫描的物理 I/O 次数高达 71,592 次，因为表并未完全保存在缓冲区缓存中。

### 传递闭包

如果你在互联网上搜索传递闭包的定义，你可能会找到很多令人困惑的数学术语。让我给你一个外行人的解释：如果 `A=B` 且 `B=C`，那么 `A=C`。真的很简单，但你可能会惊讶于 Oracle 对传递闭包的理解是多么有限。清单 14-6 展示了三个有趣的查询及其相关的执行计划。

清单 14-6. 传递闭包示例

```
CREATE TABLE t3 AS SELECT ROWNUM c1 FROM DUAL CONNECT BY LEVEL <=10;
CREATE TABLE t4 AS SELECT MOD(ROWNUM,10)+100 c1 FROM DUAL CONNECT BY LEVEL <= 100;
CREATE TABLE t5 AS SELECT MOD(ROWNUM,10) c1,RPAD('X',30) filler FROM DUAL
                             CONNECT BY LEVEL <= 10000;

CREATE INDEX t5_i1 ON t5(c1);

SELECT COUNT (*)
   FROM t3, t5
  WHERE t3.c1 = t5.c1 AND t3.c1 = 1;

| Id  | Operation             | Name  | Rows  |

|   0 | SELECT STATEMENT      |       |     1 |
|   1 |  SORT AGGREGATE       |       |     1 |
|   2 |   MERGE JOIN CARTESIAN|       |  1000 |
|*  3 |    TABLE ACCESS FULL  | T3    |     1 |
|   4 |    BUFFER SORT        |       |  1000 |
|*  5 |     INDEX RANGE SCAN  | T5_I1 |  1000 |

Predicate Information(identified by operation id):

3 - filter("T3"."C1"=1)
   5 - access("T5"."C1"=1)

SELECT *
  FROM t3, t4, t5
 WHERE t3.c1 = t4.c1 AND t4.c1 = t5.c1;

| Id  | Operation                    | Name  | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT             |       |     1 |    39 |     8  (13)| 00:00:01 |
|   1 |  NESTED LOOPS                |       |       |       |            |          |
|   2 |   NESTED LOOPS               |       |     1 |    39 |     8  (13)| 00:00:01 |
|*  3 |    HASH JOIN                 |       |     1 |     6 |     7  (15)| 00:00:01 |
|   4 |     TABLE ACCESS FULL        | T3    |    10 |    30 |     3   (0)| 00:00:01 |
|   5 |     TABLE ACCESS FULL        | T4    |   100 |   300 |     3   (0)| 00:00:01 |
|*  6 |    INDEX RANGE SCAN          | T5_I1 |  1000 |       |     1   (0)| 00:00:01 |
|   7 |   TABLE ACCESS BY INDEX ROWID| T5    |     1 |    33 |     1   (0)| 00:00:01 |

Predicate Information (identified by operation id):

3 - access("T3"."C1"="T4"."C1")
   6 - access("T4"."C1"="T5"."C1")

SELECT *
  FROM t3, t4, t5
 WHERE t3.c1 = t5.c1 AND t4.c1 = t5.c1;

| Id  | Operation           | Name | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT    |      |     1 |    39 |    25   (4)| 00:00:01 |
|*  1 |  HASH JOIN          |      |     1 |    39 |    25   (4)| 00:00:01 |
|*  2 |   HASH JOIN         |      |     1 |    36 |    22   (5)| 00:00:01 |
|   3 |    TABLE ACCESS FULL| T4   |   100 |   300 |     3   (0)| 00:00:01 |
|   4 |    TABLE ACCESS FULL| T5   | 10000 |   322K|    18   (0)| 00:00:01 |
|   5 |   TABLE ACCESS FULL | T3   |    10 |    30 |     3   (0)| 00:00:01 |

Predicate Information(identified by operation id):

1 - access("T3"."C1"="T5"."C1")
   2 - access("T4"."C1"="T5"."C1")
```

