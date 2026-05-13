# Oracle CBO 的优化器转换与数据访问策略

我们第一个查询中的谓词是 `T3.C1=1` 和 `T3.C1=T5.C1`。如果查看此第一个查询的执行计划中显示的过滤谓词，您会看到 `T3.C1=T5.C1` 谓词已被替换为 `T5.C1=1`。当存在字面值或绑定变量时，传递闭合在简单情况下通常能被识别，但任何更复杂的情况都会使 CBO 感到困惑。第二和第三个查询在语义上是等价的。两个谓词 `T3.C1=T4.c1 AND T4.C1=T5.C1` 显然等价于另外两个谓词 `T3.C1=T5.c1 AND T4.C1=T5.c1`。这两对谓词都意味着所有三个列是相同的。然而，我们可以看到最后两条语句选择的执行计划是不同的：CBO 没有“传递闭合”转换。除非两个执行计划的性能完全相同，否则它们不可能都是最优的！但是，“传递闭合”只是 CBO 未来可能实现、但目前尚不存在的众多潜在转换之一。让我们更深入地思考一下其含义。

## 不支持的转换

考虑 清单 14-7 中的两条 SQL 语句。看看并判断您认为哪一条语句写得更好。

清单 14-7. 不支持的转换

```sql
CREATE TABLE t6
AS
 SELECT ROWNUM c1,MOD(ROWNUM, 2) c2, RPAD ('X', 10) filler
         FROM DUAL
   CONNECT BY LEVEL <= 1000;
CREATE TABLE t7 AS
    SELECT ROWNUM c1,ROWNUM c2 FROM DUAL CONNECT BY LEVEL <= 10;

CREATE INDEX t6_i1 ON t6(c2,c1);

WITH subq AS
     (SELECT   t6.c2, MAX (t6.c1) max_c1
          FROM t6
      GROUP BY t6.c2)
SELECT c2, subq.max_c1
  FROM t7 LEFT JOIN subq USING (c2);
```

```
| Id  | Operation            | Name | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT     |      |    10 |   290 |     4  (23)| 00:00:01 |
|*  1 |  HASH JOIN OUTER     |      |    10 |   290 |     4  (23)| 00:00:01 |
|   2 |   TABLE ACCESS FULL  | T7   |    10 |    30 |     2   (0)| 00:00:01 |
|   3 |   VIEW               |      |     2 |    52 |     2  (20)| 00:00:01 |
|   4 |    HASH GROUP BY     |      |     2 |    14 |     2  (20)| 00:00:01 |
|   5 |     TABLE ACCESS FULL| T6   |  1000 |  7000 |     2   (0)| 00:00:01 |

Predicate Information (identified by operation id):

1 - access("T7"."C2"="SUBQ"."C2"(+))
```

```sql
SELECT c2, (SELECT /*+ no_unnest */ MAX (c1)
              FROM t6
             WHERE t6.c2 = t7.c2) max_c1
  FROM t7;
```

```
| Id  | Operation                    | Name  | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT             |       |    10 |    30 |     3   (0)| 00:00:01 |
|   1 |  SORT AGGREGATE              |       |     1 |     8 |            |          |
|   2 |   FIRST ROW                  |       |     1 |     8 |     2   (0)| 00:00:01 |
|*  3 |    INDEX RANGE SCAN (MIN/MAX)| T6_I1 |     1 |     8 |     2   (0)| 00:00:01 |
|   4 |  TABLE ACCESS FULL           | T7    |    10 |    30 |     3   (0)| 00:00:01 |

Predicate Information (identified by operation id):

3 - access("T6"."C2"=:B1)
```

如果您仔细查看这两条语句，会发现它们在语义上是等价的。两个查询都列出了来自 `T7` 的所有行，以及 `T6` 中匹配 `C2` 值对应的最大值（如果有）。那么，哪一种构造更优呢？

清单 14-7 中的上半部分查询将获取 `T6` 中每个 `C2` 值对应的 `C1` 最大值。这个计算保证对 `T6` 中的每个 `C2` 值只执行一次。

下半部分查询的构造方式使得仅对 `T7` 中存在的 `C2` 值计算 `C1` 的最大值。然而，对于 `T7` 中存在的某个 `C2` 值，其对应的 `C1` 最大值可能会被计算多次。子查询缓存可能会缓解这种情况。此外，我们还需考虑在 清单 10-8 中介绍的 `MAX/MIN` 优化。

是否觉得熟悉？我们应该把所有事情做一次（包括我们可能永远不需要做的事情），还是应该避免做我们永远不需要做的事情，即使冒着把需要做的事情做多次的风险？这与围绕使用索引访问和嵌套循环与哈希连接及全表扫描的争论是完全相同的思路。请让您的思维朝这个方向思考。这正是问题的关键所在。

因此，数据的性质和是否有合适的索引似乎决定了哪种构造更优。如果 `T7` 是一个只有几行的小表，而 `T6` 有数百万行、数千个不同的 `C2` 值，那么后者构造似乎更优。另一方面，如果 `T7` 有数百万行，而 `T6` 只有几千行（虽小但足以危及标量子查询缓存），那么可能最好使用前一种构造。

但还有一个最后的复杂情况：Oracle 12cR1 通常会反嵌套下半部分查询，将其转换为前一种形式。要防止这种启发式转换发生，您需要包含一个 `NO_UNNEST` 提示。

## 缺失信息

CBO 可能无法识别合适执行计划的另一个原因是，它不理解业务规则。如果您不以某种方式将业务规则嵌入到 SQL 中，CBO 最终可能会做出错误的决定。请看 清单 14-8，它显示了对示例 `OE` 模式中 `ORDERS` 表的两个查询。

清单 14-8. 缺失的信息

```sql
select sales_rep_id,sum(order_total) from oe.orders where order_status=7 group by sales_rep_id order by sales_rep_id;
```

```
| Id  | Operation          | Name   | Rows  | Bytes | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT   |        |     9 |   144 |     4  (25)| 00:00:01 |
|   1 |  SORT GROUP BY     |        |     9 |   144 |     4  (25)| 00:00:01 |
|*  2 |   TABLE ACCESS FULL| ORDERS |    73 |  1168 |     3   (0)| 00:00:01 |
```

```sql
select sales_rep_id,sum(order_total) from oe.orders
where order_status=7 and sales_rep_id is not null group by sales_rep_id order by sales_rep_id;
```

```
| Id  | Operation                    | Name             | Rows  | Cost (%CPU)| Time     |

|   0 | SELECT STATEMENT             |                  |     9 |     2   (0)| 00:00:01 |
|   1 |  SORT GROUP BY NOSORT        |                  |     9 |     2   (0)| 00:00:01 |
|*  2 |   TABLE ACCESS BY INDEX ROWID| ORDERS           |    49 |     2   (0)| 00:00:01 |
|*  3 |    INDEX FULL SCAN           | ORD_SALES_REP_IX |    70 |     1   (0)| 00:00:01 |
```

第一个查询列出了 `ORDER_STATUS` 为 7 的每个 `SALES_REP_ID` 的总订单价值。执行计划使用了全表扫描，并使用排序对数据进行分组，因为输出数据需要排序。

第二个查询所做的完全相同，只是我们添加了一个额外的谓词 `SALES_REP_ID is not null`。如您所见，这个额外的谓词导致了一个完全不同的执行计划，除其他外，该计划不需要排序。关键在于，`SALES_REP_ID` 上没有 `NOT NULL` 约束，因为一些交易，包括所有在线销售，没有销售员。然而，您可能知道 `ORDER_STATUS=7` 意味着使用了 `salesman`。¹ 除非您提供这个额外的谓词，否则 CBO 无法知道使用该索引是安全的，因此可能会产生一个较差的执行计划。

## 糟糕的物理设计


