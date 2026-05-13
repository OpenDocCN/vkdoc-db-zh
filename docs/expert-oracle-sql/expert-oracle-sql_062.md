# 语句 `ALTER SESSION FORCE PARALLEL QUERY`

语句 `ALTER SESSION FORCE PARALLEL QUERY` 实质上是为会话中的每条 SQL 语句添加了 `PARALLEL` 和 `PARALLEL_INDEX` 语句级提示；清单 8-26 演示了该语句并不能保证并行执行。

## 清单 8-26. 理解 FORCE PARALLEL QUERY

```
ALTER SESSION FORCE PARALLEL QUERY PARALLEL 2;

SELECT *
  FROM sh.customers c
 WHERE cust_id < 100;

| Id  | Operation                           | Name         | Cost (%CPU)|

|   0 | SELECT STATEMENT                    |              |    54   (0)|
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| CUSTOMERS    |    54   (0)|
|*  2 |   INDEX RANGE SCAN                  | CUSTOMERS_PK |     2   (0)|

Predicate Information (identified by operation id):

2 - access("CUST_ID"<100)

SELECT /*+ full(c)*/
       *
  FROM sh.customers c
 WHERE cust_id < 100;

| Id  | Operation            | Name      | Cost (%CPU)|

|   0 | SELECT STATEMENT     |           |   235   (1)|
|   1 |  PX COORDINATOR      |           |            |
|   2 |   PX SEND QC (RANDOM)| :TQ10000  |   235   (1)|
|   3 |    PX BLOCK ITERATOR |           |   235   (1)|
|*  4 |     TABLE ACCESS FULL| CUSTOMERS |   235   (1)|

Predicate Information (identified by operation id):

4 - filter("CUST_ID"<100)

ALTER SESSION FORCE PARALLEL QUERY PARALLEL 20;

SELECT *
  FROM sh.customers c
 WHERE cust_id < 100;

| Id  | Operation            | Name      | Cost (%CPU)|

|   0 | SELECT STATEMENT     |           |    23   (0)|
|   1 |  PX COORDINATOR      |           |            |
|   2 |   PX SEND QC (RANDOM)| :TQ10000  |    23   (0)|
|   3 |    PX BLOCK ITERATOR |           |    23   (0)|
|*  4 |     TABLE ACCESS FULL| CUSTOMERS |    23   (0)|

Predicate Information (identified by operation id):

4 - filter("CUST_ID"<100)

ALTER SESSION ENABLE PARALLEL QUERY;
```

清单 8-26 首先“强制”设置并行度（DOP）为 2，然后查询 `SH.CUSTOMERS` 表。得到的执行计划使用了一个串行运行的索引范围扫描，成本为 54。当我使用提示强制进行全表扫描时，可以看到 CBO 选择并行运行，但成本 235 远高于替代的串行索引范围扫描。然而，清单 8-26 中的最终查询以 DOP 20 运行。现在，并行全表扫描的成本是 23，因为每个并行查询从属进程现在需要读取的数据块更少了；CBO 计算得出，并行查询操作现在将比等效的索引范围扫描更快完成。

在第 10 章中，我们将更详细地研究索引范围扫描，并将清楚地阐明为什么不能使用数据块范围粒度来并行运行索引范围扫描。

## 延伸阅读

关于并行执行这个主题，比我本章涵盖的内容要多得多。有几个初始化参数控制着诸如实例可以支持的最小和最大并行执行从属进程数之类的事情。还有一些功能，如`语句排队`和`DOP 降级`，它们决定了当所需的并行查询服务器不可用时会发生什么。您还可以使用`资源管理器`来为并行执行设置优先级。有关所有这些功能的详细信息以及可用于监视并行执行的视图信息，我再次建议您查阅《VLDB 和分区指南》。

关于并行执行的讨论到此结束。现在是时候进入本章第三个也是最后一个主要主题：全局提示。

## 理解全局提示

SQL 语句中绝大多数嵌入的提示都是*局部*提示，这意味着它们应用于其所在的查询块。在概要中，所有查询块的所有提示都集中放置在一个地方，因此必须找到某种方法来指示每个提示所适用的查询块。这是通过在提示的第一个参数的开头使用‘@’符号来实现的，将其标记为*全局*提示。看一下清单 8-7 中的第 4、8 和 12 行。这些都是 `LEADING` 提示，但第 4 行应用于查询块 `SEL$F1D6E378`（正如我之前解释的，这是由查询块 `SEL$1` 合并到 `SEL$3` 中形成的查询块），而第 8 行和第 12 行的提示则分别应用于查询块 `SEL$2` 和 `SEL$4`。

全局提示在《SQL 语言参考手册》中有文档说明，在嵌入 SQL 语句时得到完全支持，并且通常非常宝贵。至少有三种情况值得使用全局提示，如下所示：

*   您想提示源自数据字典视图的块。在这种情况下使用局部提示需要修改视图的定义，并会影响该视图的其他用户。
*   您想提示作为查询转换结果的查询块。在这些情况下，局部提示可能会让读者感到困惑。
*   当提示数量过多以至于损害了原始语句的可读性时，全局提示允许将它们全部集中放置在一个地方。

我将提供前两种场景的简单示例，但要为复杂的提示提供一个简单示例，这本身是有些矛盾的。在演示了这些场景之后，我将重点介绍 `NO_MERGE` 提示，因为这个非常常见提示的多个变体有些令人望而生畏。如果您能掌握 `NO_MERGE` 提示，您就可以认为自己是全局提示的大师了！

### 提示数据字典视图

清单 8-27 创建了一个数据字典视图并在查询中使用该视图。为了简单起见，我还删除了 `T1` 上的索引并禁用了并行查询。

清单 8-27. 使用数据字典视图的简单查询

```
ALTER SESSION DISABLE PARALLEL QUERY;

DROP INDEX t1_i1

CREATE OR REPLACE VIEW v1
AS
     SELECT c1, MEDIAN (c2) med_c2
       FROM t1, t2
      WHERE t1.c1 = t2.c2
   GROUP BY t1.c1;

SELECT *
  FROM v1, t3
 WHERE v1.c1 = t3.c3;

Plan hash value: 808098385

| Id  | Operation             | Name |

|   0 | SELECT STATEMENT      |      |
|   1 |  HASH JOIN            |      |
|   2 |   VIEW                | V1   |
|   3 |    HASH GROUP BY      |      |
|   4 |     HASH JOIN         |      |
|   5 |      TABLE ACCESS FULL| T1   |
|   6 |      TABLE ACCESS FULL| T2   |
|   7 |   TABLE ACCESS FULL   | T3   |

Query Block Name / Object Alias (identified by operation id):

1 - SEL$1
   2 - SEL$2 / V1@SEL$1
   3 - SEL$2
   5 - SEL$2 / T1@SEL$2
   6 - SEL$2 / T2@SEL$2
   7 - SEL$1 / T3@SEL$1

Outline Data

/*+
      BEGIN_OUTLINE_DATA
      USE_HASH_AGGREGATION(@"SEL$2")
      USE_HASH(@"SEL$2" "T2"@"SEL$2")
      LEADING(@"SEL$2" "T1"@"SEL$2" "T2"@"SEL$2")
      FULL(@"SEL$2" "T2"@"SEL$2")
      FULL(@"SEL$2" "T1"@"SEL$2")
      USE_HASH(@"SEL$1" "T3"@"SEL$1")
      LEADING(@"SEL$1" "V1"@"SEL$1" "T3"@"SEL$1")
      FULL(@"SEL$1" "T3"@"SEL$1")
      NO_ACCESS(@"SEL$1" "V1"@"SEL$1")
      OUTLINE_LEAF(@"SEL$1")
      OUTLINE_LEAF(@"SEL$2")
      ALL_ROWS
      DB_VERSION('12.1.0.1')
      OPTIMIZER_FEATURES_ENABLE('12.1.0.1')
      IGNORE_OPTIM_EMBEDDED_HINTS
      END_OUTLINE_DATA
  */
```

现在，假设您希望将 `T1` 和 `T2` 的 `HASH JOIN` 更改为由 `T1` 驱动的嵌套循环连接。当然，从性能角度来看这没有意义，但我此时只是试图用一个简单的例子来解释原理。清单 8-28 展示了如何在不编辑视图定义的情况下做到这一点。



```markdown
清单 8-28. 将提示应用于数据字典视图中嵌入的查询块

```sql
SELECT
/*+
    LEADING(@"SEL$2" "T1"@"SEL$2" "T2"@"SEL$2")
    USE_NL(@"SEL$2" "T2"@"SEL$2")
*/
*
  FROM v1, t3
 WHERE v1.c1 = t3.c3;

Plan hash value: 1369664419

| Id  | Operation             | Name |

|   0 | SELECT STATEMENT      |      |
|   1 |  HASH JOIN            |      |
|   2 |   VIEW                | V1   |
|   3 |    HASH GROUP BY      |      |
|   4 |     NESTED LOOPS      |      |
|   5 |      TABLE ACCESS FULL| T1   |
|   6 |      TABLE ACCESS FULL| T2   |
|   7 |   TABLE ACCESS FULL   | T3   |

Query Block Name / Object Alias (identified by operation id):

1 - SEL$1
   2 - SEL$2 / V1@SEL$1
   3 - SEL$2
   5 - SEL$2 / T1@SEL$2
   6 - SEL$2 / T2@SEL$2
   7 - SEL$1 / T3@SEL$1

Outline Data

/*+
      BEGIN_OUTLINE_DATA
      USE_HASH_AGGREGATION(@"SEL$2")
      USE_NL(@"SEL$2" "T2"@"SEL$2")
      LEADING(@"SEL$2" "T1"@"SEL$2" "T2"@"SEL$2")
      FULL(@"SEL$2" "T2"@"SEL$2")
      FULL(@"SEL$2" "T1"@"SEL$2")
      USE_HASH(@"SEL$1" "T3"@"SEL$1")
      LEADING(@"SEL$1" "V1"@"SEL$1" "T3"@"SEL$1")
      FULL(@"SEL$1" "T3"@"SEL$1")
      NO_ACCESS(@"SEL$1" "V1"@"SEL$1")
      OUTLINE_LEAF(@"SEL$1")
      OUTLINE_LEAF(@"SEL$2")
      ALL_ROWS
      DB_VERSION('12.1.0.1')
      OPTIMIZER_FEATURES_ENABLE('12.1.0.1')
      IGNORE_OPTIM_EMBEDDED_HINTS
      END_OUTLINE_DATA
  */
```

我在这里所做的是查看清单 8-27 中原始语句执行计划的`ALIAS`和`OUTLINE`部分的高亮行，以确定要使用的正确查询块名称和对象别名。由于操作 4 没有列出查询块，我可以通过查看操作 3 的查询块来确定正确的查询块。然后，我利用这些信息构建了合适的全局提示，并将其嵌入到我的 SQL 语句中。清单 8-28 中语句的执行计划表明，这些提示达到了预期的效果。

![image](img/sq.jpg) **注意** 重要的是要认识到，与本地提示不同，全局提示出现的位置无关紧要；它们可以放置在 SQL 语句中任何合法的位置，效果都是相同的。

将提示应用于转换后的查询

一般来说，本地提示比全局提示更可取，因为它们更容易被读者理解，但情况并非总是如此。在本书的第 13 章中，我将讨论`星型转换`和`子查询非嵌套转换`。这些转换以及其他一些转换，可能导致生成的执行计划与原始 SQL 语句看起来几乎没有相似之处，使得本地提示显得毫无意义。即使在没有或很少有转换的情况下，当大量本地提示分散在一个复杂的语句中时，也会成为原始代码可读性的障碍。在这些希望罕见的情况下，在我看来，将一组全局提示集中放置可以使理解语句及其相关提示变得更容易。提供一个混淆的简单例子有点挑战，但清单 8-29 可能有助于说明我的观点。

清单 8-29. 应用于非嵌套子查询的本地提示

```sql
SELECT *
  FROM t1
 WHERE NOT EXISTS
          (SELECT /*+ unnest use_nl(t2)*/

FROM t2
            WHERE t2.c2 = t1.c1);
```

我故意隐瞒了清单 8-29 的执行计划，以模拟程序员阅读源代码时的体验。`USE_NL`提示控制连接机制，但因为它出现在只包含一个行源的块中，所以看起来毫无意义。然而，这个提示是有效的，因为它应用于一个包含多个行源的转换后的块。由于这些提示只有在查看转换后的执行计划时才有意义，因此最好将所有这些提示集中放在一个地方，这样业务逻辑仍然清晰明了。

NO_MERGE 提示

我想通过关于`MERGE`和`NO_MERGE`提示的说明来结束对全局提示的讨论。当用作本地提示时，`MERGE`和`NO_MERGE`提示有两种变体。这两种本地变体都有等效的全局形式，因此总共有四种变体。很容易混淆这四种变体，所以让我来澄清一下。为了简单起见，我将坚持使用`NO_MERGE`提示，因为它显然是更常见的嵌入式提示。

*   当用作不带参数的本地提示时，`NO_MERGE`指示 CBO 不要合并提示所在的*查询块*。
*   当用作带有一个参数（前面没有‘@’符号）的本地提示时，该参数被假定为一个数据字典视图、一个因子化子查询或一个内联视图。`NO_MERGE`提示指示 CBO 不要合并参数中命名的*行源*。
*   当用作带有一个参数（前面有‘@’符号）的全局提示时，`NO_MERGE`提示指示 CBO 不要合并参数中命名的*查询块*。
*   当用作带有两个参数的全局提示（第一个参数前有‘@’符号）时，第二个参数被假定为一个数据字典视图、一个因子化子查询或一个出现在第一个参数指定的查询块中的内联视图。`NO_MERGE`提示指示 CBO 不要合并出现在第一个参数指定的*查询块*中、由第二个参数命名的*行源*。

困惑了吗？也许清单 8-30 能说明一些问题。

清单 8-30. `NO_MERGE`提示的四种变体

```sql
WITH fs AS (SELECT /*+ qb_name(qb1) no_merge*/
                  * FROM t1)
SELECT /*+ qb_name(qb2) */
       *
  FROM fs myalias, t2
 WHERE myalias.c1 = t2.c2;

WITH fs AS (SELECT /*+ qb_name(qb1) */
                  * FROM t1)
SELECT /*+ qb_name(qb2) no_merge(myalias)*/
       *
  FROM fs myalias, t2
 WHERE myalias.c1 = t2.c2;

WITH fs AS (SELECT /*+ qb_name(qb1) */
                  * FROM t1)
SELECT /*+ qb_name(qb2) no_merge(@qb1)*/
       *
  FROM fs myalias, t2
 WHERE myalias.c1 = t2.c2;

WITH fs AS (SELECT /*+ qb_name(qb1) no_merge(@qb2 myalias)*/
                  * FROM t1)
SELECT /*+ qb_name(qb2)  */
       *
  FROM fs myalias, t2
 WHERE myalias.c1 = t2.c2;

Plan hash value: 2191810965

| Id  | Operation           | Name |

|   0 | SELECT STATEMENT    |      |
|   1 |  HASH JOIN          |      |
|   2 |   VIEW              |      |
|   3 |    TABLE ACCESS FULL| T1   |
|   4 |   TABLE ACCESS FULL | T2   |
```

除了提示之外，清单 8-30 中的四个语句是相同的。它们都包含两个查询块，除非被提示阻止，否则它们将会被合并。这四个语句中的每一个都包含一个`NO_MERGE`提示，该提示成功地阻止了合并，但这四个提示之间有着细微的差别。

理解这些提示如何运作的关键在于理解为第一个查询块命名的三种不同方式。
```



