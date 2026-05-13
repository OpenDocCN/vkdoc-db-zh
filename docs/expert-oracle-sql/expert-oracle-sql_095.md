# 并行连接的分布机制

第 8 章介绍了并行执行的主题，特别是并行查询服务器集（PQSS）通过表队列（TQ）进行通信的一些不同方式。我们不得不将关于并行连接相关分布机制的讨论推迟到我们涵盖串行连接之后，但现在我们可以重新审视这个主题了。

当 PQSS 协作执行两个表的连接时，有相当多的通信机制，其中一些仅适用于分区表，而另一些适用于任何表。正确选择连接分布机制对性能的影响程度并未得到广泛认识，因此我将详细介绍每种分布机制。但在那之前，让我先讨论一下 `PQ_DISTRIBUTE` 提示在并行连接中的使用方式。

## PQ_DISTRIBUTE 提示与并行连接

当 `PQ_DISTRIBUTE` 用作控制负载分布的本地提示时，只提供两个参数，正如我在 Listings 8-23 和 8-24 中演示的那样。当用作控制连接分布的本地提示时，则有三个参数。第一个参数标识该提示应用的连接，为此我们使用探测源的名称，就像我们为 `USE_HASH` 等提示所做的那样。第二和第三个参数组合使用以指定分布机制。是时候举个例子了。让我们从第一个连接分布机制开始：*全分区智能连接*。

## 全分区智能连接

全分区智能连接仅在同时满足以下两个条件时才会发生：

*   连接的两个表都是分区表，并且以相同的方式分区。¹
*   在分区（或子分区）列上存在等值连接谓词。换句话说，一个表中的分区列与另一个表中对应的分区列相等。Listing 11-17 展示了全分区智能连接的两个实际示例。

**Listing 11-17. 全分区智能连接**

```sql
CREATE TABLE t_part1
PARTITION BY HASH (c1)
   PARTITIONS 8
AS
   SELECT c1, ROWNUM AS c3 FROM t1;

CREATE TABLE t_part2
PARTITION BY HASH (c2)
   PARTITIONS 8
AS
   SELECT c1 AS c2, ROWNUM AS c4 FROM t1;

SELECT *
  FROM t_part1, t_part2
 WHERE t_part1.c1 = t_part2.c2;

| Id  | Operation           | Name    |

|   0 | SELECT STATEMENT    |         |
|   1 |  PARTITION HASH ALL |         |
|*  2 |   HASH JOIN         |         |
|   3 |    TABLE ACCESS FULL| T_PART1 |
|   4 |    TABLE ACCESS FULL| T_PART2 |

Predicate Information (identified by operation id):

2 - access("T_PART1"."C1"="T_PART2"."C2")

SELECT /*+ parallel(t_part1 8) parallel(t_part2 8) leading(t_part1)
           pq_distribute(t_part2 NONE NONE)*/
       *
  FROM t_part1, t_part2
 WHERE t_part1.c1 = t_part2.c2;

| Id  | Operation               | Name     |    TQ  |IN-OUT| PQ Distrib |

|   0 | SELECT STATEMENT        |          |        |      |            |
|   1 |  PX COORDINATOR         |          |        |      |            |
|   2 |   PX SEND QC (RANDOM)   | :TQ10000 |  Q1,00 | P->S | QC (RAND)  |
|   3 |    PX PARTITION HASH ALL|          |  Q1,00 | PCWC |            |
|*  4 |     HASH JOIN           |          |  Q1,00 | PCWP |            |
|   5 |      TABLE ACCESS FULL  | T_PART1  |  Q1,00 | PCWP |            |
|   6 |      TABLE ACCESS FULL  | T_PART2  |  Q1,00 | PCWP |            |

Predicate Information (identified by operation id):

4 - access("T_PART1"."C1"="T_PART2"."C2")
```



