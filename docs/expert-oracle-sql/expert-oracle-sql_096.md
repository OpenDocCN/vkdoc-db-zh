# 分区连接与分发机制

## 全分区连接

Listing 11-17 首先重新创建了来自 Listing 8-23 的分区表 `T_PART1` 和 `T_PART2`，然后以串行和并行方式连接它们。由于两张表以相同方式分区，且连接条件使两个分区列 `T_PART1.C1` 和 `T_PART2.C2` 相等，因此可以进行全分区连接。从并行变体中可以看出，只有一个 `TQ` 和一个 DFO。每个并行查询服务器从 `T_PART1` 的一个分区读取数据，然后将其与来自 `T_PART2` 对应分区的行进行连接。如果分区数多于并行查询服务器数，则可能需要一个并行查询服务器为多个分区重复连接操作。

全分区连接同时支持哈希连接和归并连接，但对我来说原因不特别清楚的是，在 12.1.0.1 版本中，嵌套循环似乎是非法的。

由于全分区连接一次处理一对分区，因此内存使用最小化，并且在并行运行时只有一个 DFO，不需要并行查询服务器之间的任何通信。并行全分区连接的潜在缺点是使用了分区粒度，因此如果一张或两张表的某些分区比其他分区大，负载均衡可能无效。在解决这个顾虑之前，让我们考虑当只有一张表按等值连接列分区时会发生什么。

## 部分分区连接

Listing 11-18 展示了一个涉及分区表和非分区表的连接。

**Listing 11-18. 部分分区连接**

```sql
SELECT                                   SELECT
  /*+ parallel(t_part1 8)                 /*+ parallel(t_part1 8)
      parallel(t1 8) full(t1)                 parallel(t1 8) full(t1)
      leading(t_part1)                        leading(t1)
      no_swap_join_inputs(t1)                 swap_join_inputs(t_part1)
      pq_distribute(t1 NONE PARTITION) */     pq_distribute(t_part1 PARTITION NONE) */
       *                                       *
FROM t_part1 JOIN t1 USING (c1);          FROM t_part1 JOIN t1 USING (c1);
```

```
| Id  | Operation                   | Name     |    TQ  |IN-OUT| PQ Distrib |
|---  |---                          |---       |---     |---   |---         |
|   0 | SELECT STATEMENT            |          |        |      |            |
|   1 |  PX COORDINATOR             |          |        |      |            |
|   2 |   PX SEND QC (RANDOM)       | :TQ10001 |  Q1,01 | P->S | QC (RAND)  |
|*  3 |    HASH JOIN BUFFERED       |          |  Q1,01 | PCWP |            |
|   4 |     PX PARTITION HASH ALL   |          |  Q1,01 | PCWC |            |
|   5 |      TABLE ACCESS FULL      | T_PART1  |  Q1,01 | PCWP |            |
|   6 |     PX RECEIVE              |          |  Q1,01 | PCWP |            |
|   7 |      PX SEND PARTITION (KEY)| :TQ10000 |  Q1,00 | P->P | PART (KEY) |
|   8 |       PX BLOCK ITERATOR     |          |  Q1,00 | PCWC |            |
|   9 |        TABLE ACCESS FULL    | T1       |  Q1,00 | PCWP |            |

Predicate Information (identified by operation id):
3 - access("T_PART1"."C1"="T1"."C1")
```

与全分区连接不同，像 Listing 11-18 中所示的部分分区连接仅在并行运行语句的执行计划中看到。

Listing 11-18 连接了两张表，但只有哈希连接中驱动行源按连接谓词中使用的列进行了分区。我们在这里使用了两个 PQSS。`T_PART1` 使用了分区粒度，而 `T1` 使用了块范围粒度。从 `T1` 读取的行只能与 `T_PART1` 中一个分区中的对应行匹配，因此 PQSS1 读取的每一行仅被发送到 PQSS2 中的一个服务器。

请注意，尽管我们消除了被探测行源需要分区的需求，并因此消除了该行源中因数据倾斜导致负载不平衡的风险，但现在我们有了两个 DFO 和一个 TQ 的开销：一个 DFO 从驱动行源获取行并执行连接，第二个 DFO 访问被探测行源。需要明确的是，这种部分分区连接的变体对于驱动行源中分区大小的变化没有任何帮助。

Listing 11-18 展示了我们如何使用提示获得相同执行计划的两种不同方式。如果我们不交换连接输入，则使用关键字 `NONE PARTITION` 来强制此分发机制；如果我们交换了连接输入，我们也交换关键字并指定 `PARTITION NONE`。

当连接中只有被探测的表被适当分区时，我们也可以使用部分分区连接，我将在接下来关于布隆过滤的讨论中进行演示。

### 广播分发

Listing 11-19 连接了两张非分区表。

**Listing 11-19. BROADCAST 分发机制**

```sql
CREATE INDEX t1_i1 ON t1 (c1);

SELECT /*+ index(t1) parallel(t2 8)       SELECT /*+ index(t1) parallel(t2 8)
         leading(t1)                                 leading(t2)
         use_hash(t2)                                use_hash(t1)
         no_swap_join_inputs(t2)                     swap_join_inputs(t1)
         pq_distribute(t2 BROADCAST NONE)            pq_distribute(t1 NONE BROADCAST)
         no_pq_replicate(t2)                         no_pq_replicate(t1)
       */                                         */
       *                                          *
  FROM t1 JOIN t2 ON t1.c1 = t2.c2;          FROM t1 JOIN t2 ON t1.c1 = t2.c2;
```

```
| Id  | Operation             | Name     |    TQ  |IN-OUT| PQ Distrib |
|---  |---                    |---       |---     |---   |---         |
|   0 | SELECT STATEMENT      |          |        |      |            |
|   1 |  PX COORDINATOR       |          |        |      |            |
|   2 |   PX SEND QC (RANDOM) | :TQ10001 |  Q1,01 | P->S | QC (RAND)  |
|*  3 |    HASH JOIN          |          |  Q1,01 | PCWP |            |
|   4 |     PX RECEIVE        |          |  Q1,01 | PCWP |            |
|   5 |      PX SEND BROADCAST| :TQ10000 |  Q1,00 | S->P | BROADCAST  |
|   6 |       PX SELECTOR     |          |  Q1,00 | SCWC |            |
|   7 |        INDEX FULL SCAN| T1_I1    |  Q1,00 | SCWP |            |
|   8 |     PX BLOCK ITERATOR |          |  Q1,01 | PCWC |            |
|   9 |      TABLE ACCESS FULL| T2       |  Q1,01 | PCWP |            |

Predicate Information (identified by operation id):
3 - access("T1"."C1"="T2"."C2")
```

Listing 11-19 首先在 `T1.C1` 上创建索引，然后运行两个查询，并排显示。当查询运行时，PQSS1 中的一个从属进程使用索引从 `T1` 读取行，并将读取的每一行发送到 *每个* PQSS2 成员，这些行被放入第 3 行 `HASH JOIN` 的工作区中。一旦 `T1.T1_I1` 索引读取完成，PQSS2 的每个成员使用块范围粒度读取 `T2` 的一部分。这些行在第 3 行进行连接，然后发送给 QC。

现在，这可能看起来效率低下。为什么我们要将 `T1` 的行发送到 PQSS2 的 *每个* 成员呢？嗯，如果 `T1` 有大量宽行，我们不会这样做。但如果 `T1` 只有少量窄行，广播这些行的开销可能并不那么大。但仍然存在一些开销，那么为什么还要这样做呢？答案在于，我们可以避免在连接之前将 `T2` 的行发送到任何地方！如果 `T2` 比 `T1` 大得多，通过 TQ 多次发送 `T1` 的行可能比通过 TQ 一次发送 `T2` 的行更好。



广播分发方法支持所有连接方式，而清单 11-19 展示了两种用于提示哈希连接的变体，类似于部分分区连接。

![image](img/sq.jpg) **提示** 你无法对外连接的保留行源进行广播。这是因为对可选行源的存在性检查必须在其所有行上执行。类似的限制也适用于半连接和反连接：你无法广播主查询，因为存在性检查需要应用于子查询中的所有行。

与我在清单 11-18 中展示的部分分区连接类似，清单 11-19 中的广播分发机制使用了两个 DFO，但这次连接是由访问探测行源而非驱动行源的 DFO 执行的。

从技术上讲，广播探测行源而非驱动行源是可行的，但我想不到这有什么用。一种实现方法是用 `SWAP_JOIN_INPUTS` 替换清单 11-19 左侧查询中的 `NO_SWAP_JOIN_INPUTS`。一方面，广播 `T2` 意味着 `T2` 小于 `T1`。另一方面，选择 `T1` 作为哈希连接的驱动行源这一事实表明 `T1` 小于 `T2`。这种矛盾意味着 CBO 不太可能选择这种并行连接方法，你或许也不应该这样做。

### 行源复制

这是一种在 12cR1 中引入的并行查询分发机制，我尚未看到它的官方名称，因此暂称之为行源复制。出于某种原因，该分发机制不是通过 `PQ_DISTRIBUTE` 的新变体来提示，而是通过一个新的提示 `PQ_REPLICATE` 来提示，该提示修改了广播复制的行为。

细心的读者会注意到清单 11-19 中的查询包含了 `NO_PQ_REPLICATE` 提示。清单 11-20 将这些提示改为了 `PQ_REPLICATE`。

清单 11-20. 行源复制

```sql
SELECT /*+ parallel(t2 8)                   SELECT /*+ parallel(t2 8)
         leading(t1)                                 leading(t2)
         index_ffs(t1) parallel_index(t1)            index_ffs(t1) parallel_index(t1)
         use_hash(t2)                                use_hash(t1)
         no_swap_join_inputs(t2)                     swap_join_inputs(t1)
         pq_distribute(t2 BROADCAST NONE)            pq_distribute(t1 NONE BROADCAST)
         pq_replicate(t2)                            pq_replicate(t1)
       */                                         */
       *                                          *
  FROM t1 JOIN t2 ON t1.c1 = t2.c2;          FROM t1 JOIN t2 ON t1.c1 = t2.c2;

| Id  | Operation               | Name     |

|   0 | SELECT STATEMENT        |          |
|   1 |  PX COORDINATOR         |          |
|   2 |   PX SEND QC (RANDOM)   | :TQ10000 |
|*  3 |    HASH JOIN            |          |
|   4 |     INDEX FAST FULL SCAN| T1_I1    |
|   5 |     PX BLOCK ITERATOR   |          |
|   6 |      TABLE ACCESS FULL  | T2       |

Predicate Information (identified by operation id):

3 - access("T1"."C1"="T2"."C2")
```

现在，这里发生的事情相当酷。由于 `T1` 非常小，CBO 推断让连接 DFO 中的每个并行查询服务器完整读取 `T1`，比使用单独的 DFO 和单独的 TQ 来读取 `T1` 并广播结果更划算。在完整读取 `T1` 后，每个并行查询服务器读取 `T2` 的一部分。

这种分发机制仅在 `T1` 小而 `T2` 大时才有意义。因此，如果你从 TB 级的表 `T1` 中选择一行，除非有提示，否则 CBO 将使用旧的广播分发机制而非复制机制。

你可能已经注意到，我将清单 11-19 中的 `INDEX (T1)` 提示替换为了清单 11-20 中的两个提示 `INDEX_FFS (T1)` 和 `PARALLEL_INDEX (T1)`。`T1` 被视为并行读取，尽管它实际上是串行但多次读取。由于这是复制而非并行执行，因此没有理由禁止索引范围扫描或全扫描。然而，截至 `12.1.0.1`，此功能尚未支持，因此我不得不使用索引快速全扫描作为替代。全表扫描也可以工作。

还有一种分发机制我们尚未讨论。这种连接机制解决了与分区大小相关的所有负载不平衡问题，但有代价。

## 哈希分布

想象一下，你需要将一个行源的 `3GB` 数据与另一个行源的 `3GB` 数据连接。这个连接发生在你夜间批量处理的关键节点。在你的连接完成之前，没有其他任务在运行，也不会有其他任务运行，因此时间至关重要。你有 `15GB` 的 PGA、`32` 个处理器和 `20` 个并行查询服务器。串行执行不在考虑之列。这不仅会浪费你可用的所有 CPU 和磁盘带宽，而且你的连接操作会溢出到磁盘：工作区的最大大小是 `2GB`，而你有 `3GB` 的数据。

你需要使用并行执行，并选择 `10` 的并行度（DOP），但你认为应该使用哪种连接分发机制？你的第一个想法应该是某种分区连接。但让我们假设这不是一个选项。也许你的行源不是分区表。也许它们是分区表，但你的全部 `3GB` 数据来自一个分区。也许 `3GB` 来自多个分区，但连接条件与分区列无关。无论出于何种原因，你都需要找到分区连接的替代方案。

行源复制和广播分发似乎都不吸引人。当你将 `3GB` 复制或广播到十个并行查询从属进程时，你总共使用了 `30GB` 的 PGA。这超过了可用的 `15GB`。清单 11-21 展示了我们如何解决此类问题。

清单 11-21. 哈希分布

```sql
ALTER SESSION SET optimizer_adaptive_features=FALSE;
SELECT /*+ full(t1) parallel(t1)
        parallel(t_part2 8)
       leading(t1)
       use_hash(t_part2)
       no_swap_join_inputs(t_part2)
       pq_distribute(t_part2 HASH HASH)*/
       *
  FROM t1 JOIN t_part2 ON t1.c1 = t_part2.c4
 WHERE t_part2.c2 IN (1, 3, 5);

| Id  | Operation               | Name     | Pstart| Pstop |    TQ  |IN-OUT| PQ Distrib |

|   0 | SELECT STATEMENT        |          |       |       |        |      |            |
|   1 |  PX COORDINATOR         |          |       |       |        |      |            |
|   2 |   PX SEND QC (RANDOM)   | :TQ10002 |       |       |  Q1,02 | P->S | QC (RAND)  |
|*  3 |    HASH JOIN BUFFERED   |          |       |       |  Q1,02 | PCWP |            |
|   4 |     PX RECEIVE          |          |       |       |  Q1,02 | PCWP |            |
|   5 |      PX SEND HASH       | :TQ10000 |       |       |  Q1,00 | P->P | HASH       |
|   6 |       PX BLOCK ITERATOR |          |       |       |  Q1,00 | PCWC |            |
|   7 |        TABLE ACCESS FULL| T1       |       |       |  Q1,00 | PCWP |            |
|   8 |     PX RECEIVE          |          |       |       |  Q1,02 | PCWP |            |
|   9 |      PX SEND HASH       | :TQ10001 |       |       |  Q1,01 | P->P | HASH       |
|  10 |       PX BLOCK ITERATOR |          |KEY(I) |KEY(I) |  Q1,01 | PCWC |            |
|* 11 |        TABLE ACCESS FULL| T_PART2  |KEY(I) |KEY(I) |  Q1,01 | PCWP |            |
```



