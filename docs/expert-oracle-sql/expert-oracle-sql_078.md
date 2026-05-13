# 分区与子分区统计概述

在深入探讨这些主题之前，我需要先解释如何获取分区级别的统计信息。为特定分区或子分区设置、导出和导入统计信息，其方法与为整个表或索引执行这些操作的方法相同，只需指定分区或子分区的名称即可；因此我不再赘述这些步骤。然而，在分区或子分区级别收集统计信息存在一些复杂情况，我将在下面说明。

表统计信息与分区级别统计信息之间的关系，可以扩展到分区级别统计信息与子分区级别统计信息之间的关系。为了简洁起见，在本节的大部分内容中，我将忽略复合分区表和子分区级别的统计信息。

## 在分区表上收集统计信息

当我们调用如 `DBMS_STATS.GATHER_TABLE_STATS` 之类的例程并提供分区表的名称作为参数时，我们可以为 `GRANULARITY` 参数指定一个值。对于分区表，`GRANULARITY` 的默认值是 `ALL`，这意味着同时收集全局统计信息和分区级别统计信息。

你可能会认为，通过扫描分区表中每个分区的数据，可以同时构建全局统计信息和分区级别统计信息。但这并非默认行为。

如果你为 `DBMS_STATS.GATHER_TABLE_STATS` 的 `GRANULARITY` 参数指定 `ALL`，并为 `PARTNAME` 参数指定 `NULL`，默认情况下实际发生的是：所有分区会被扫描两次。一次扫描用于获取全局统计信息，另一次用于获取分区级别统计信息。因此，如果你的表有五个分区，总共将进行十次分区扫描。

如果你为 `DBMS_STATS.GATHER_TABLE_STATS` 的 `GRANULARITY` 参数指定 `ALL`，并为 `PARTNAME` 参数指定特定分区的名称，那么仅对指定的分区收集分区级别统计信息，但是*所有*分区都会被扫描以收集全局统计信息。因此，如果你的表有五个分区，总共将进行六次分区扫描。

许多用户喜欢在数据加载到先前为空的分区时收集分区级别统计信息，而扫描所有分区以更新全局统计信息的开销是不可接受的。因此，在收集单个分区的统计信息时，调用 `DBMS_STATS.GATHER_TABLE_STATS` 通常为 `GRANULARITY` 参数指定 `PARTITION` 值。此设置意味着仅在指定的分区上进行一次扫描，而不扫描任何其他分区。幸运的是，这种捷径的负面影响远小于你的想象。清单 9-14 展示了当未显式收集全局统计信息时会发生什么。

### 清单 9-14. 将 GRANULARITY 设置为 PARTITION 以收集统计信息

```sql
CREATE /*+ NO_GATHER_OPTIMIZER_STATISTICS */TABLE statement_part
PARTITION BY RANGE
   (transaction_date)
   (
      PARTITION p1 VALUES LESS THAN (DATE '2013-01-05')
     ,PARTITION p2 VALUES LESS THAN (maxvalue))
AS
   SELECT * FROM statement;

BEGIN
   DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,partname      => 'P1'
     ,GRANULARITY   => 'PARTITION'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1');

DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,partname      => 'P2'
     ,GRANULARITY   => 'PARTITION'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1');
END;
/

SELECT a.num_rows p1_rows
      ,a.global_stats p1_global_stats
      ,b.num_rows p2_rows
      ,b.global_stats p2_global_stats
      ,c.num_rows tab_rows
      ,c.global_stats tab_global_stats
  FROM all_tab_statistics a
       FULL JOIN all_tab_statistics b USING (owner, table_name)
       FULL JOIN all_tab_statistics c USING (owner, table_name)
 WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT_PART'
       AND a.partition_name = 'P1'
       AND b.partition_name = 'P2'
       AND c.partition_name IS NULL;

P1_ROWS   P1_GLOBAL_STATS   P2_ROWS    P2_GLOBAL_STATS    TAB_ROWS      TAB_GLOBAL_STATS
200       YES               300        YES                500           NO
```

清单 9-14 创建了一个表 `STATEMENT_PART`，其内容与 `STATEMENT` 相同，但所有虚拟列都变成了真实列，并且表通过 `TRANSACTION_DATE` 列按范围分区。创建了两个分区；它们被命名为 `P1` 和 `P2`。我指定了 `NO_GATHER_OPTIMIZER_STATISTICS` 提示，以便在 12cR1 及更高版本中不会自动收集全局统计信息。

一旦 `STATEMENT_PART` 被创建并填充，统计信息会分别收集在两个分区——`P1` 和 `P2` 上。清单 9-14 随后查看了两个分区以及整个表的 `NUM_ROWS` 和 `GLOBAL_STATS` 统计列。我们可以看到，分区 `P1` 中显然有 200 行，`P2` 中有 300 行。然而，与我上面关于 `GRANULARITY` 参数的解释明显矛盾的是，整个表的全局统计信息是存在的，因为 `ALL_TAB_STATISTICS` 中的 `NUM_ROWS` 被设置为 500！

理解所发生事情的关键是查看名为 `GLOBAL_STATS` 的列的值。多年来，我曾对这个列感到非常困惑。我曾以为设置为 `YES` 或 `NO` 的 `GLOBAL_STATS` 与统计信息是否是全局统计信息有关。我希望你能理解我的错误。如果我认真阅读了参考手册，我会看到以下内容：

GLOBAL_STATS VARCHAR2 (3) 指示统计信息是否在不合并底层分区的情况下计算（YES）还是（NO）

直到不久前，Doug Burns (`http://oracledoug.com/index.html`) 在一次会议演讲中恰当地解释了这个概念，才将我从困惑中解救出来。你看，一旦表或索引中所有分区的统计信息都可用，这些分区级别的统计信息就可以被合并以确定全局统计信息。在这个案例中，来自 `P1` 的 200 和来自 `P2` 的 300 被相加得到整个 `STATEMENT_PART` 的 500。

这种合并分区级别统计信息的方法似乎使得收集全局统计信息变得不必要。然而，这并不完全正确。清单 9-15 展示了与合并分区级别统计信息相关的一个问题。

### 清单 9-15. 合并分区级别统计信息时的 NUM_DISTINCT 计算

```sql
SELECT column_name
      ,a.num_distinct p1_distinct
      ,b.num_distinct p2_distinct
      ,c.num_distinct tab_distinct
  FROM all_part_col_statistics a
       FULL JOIN all_part_col_statistics b
          USING (owner, table_name, column_name)
       FULL JOIN all_tab_col_statistics c
          USING (owner, table_name, column_name)
 WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT_PART'
       AND a.partition_name = 'P1'
       AND b.partition_name = 'P2';
```


| 列名 | P1_ 唯一值数量 | P2_ 唯一值数量 | 表 _ 唯一值数量 |
| --- | --- | --- | --- |
| CUSTOMER_CATEGORY | 50 | 50 | 50 |
| PRODUCT_CATEGORY | 4 | 6 | 10 |
| AMOUNT_CATEGORY | 3 | 3 | 3 |
| TRANSACTION_AMOUNT | 112 | 166 | 166 |
| DESCRIPTION | 4 | 4 | 4 |
| POSTING_DELAY | 3 | 3 | 3 |
| POSTING_DATE | 6 | 8 | 8 |
| TRANSACTION_DATE | 4 | 6 | 10 |
| TRANSACTION_DATE_TIME | 4 | 6 | 10 |

清单 9-15 查询了分区 `P1`、`P2` 以及整个表的 `NUM_DISTINCT` 统计列。我们可以看到，在 `P1` 中 `TRANSACTION_DATE_TIME` 有四个唯一值，在 `P2` 中有六个唯一值。`DBMS_STATS` 包正确判断出整个表中总共有十个 `TRANSACTION_DATE_TIME` 的唯一值，这是因为 `P1` 中 `TRANSACTION_DATE_TIME` 的最大值低于 `P2` 中的最小值。另一方面，`DBMS_STATS` 看到 `P1` 中 `POSTING_DATE` 有六个唯一值，`P2` 中有八个，但它无法知道整个表中 `POSTING_DATE` 的唯一值数量是 8、14 还是介于两者之间。这种不确定性是由于 `P1` 中 `POSTING_DATE` 的最大值大于 `P2` 中的最小值造成的。`STATEMENT_PART` 表中 `POSTING_DATE` 的实际唯一值数量是 12。在这种情况下，`DBMS_STATS` 总是取最小的可能值。

Oracle 数据库 11gR1 引入了一个巧妙的新功能，允许我们通过合并分区列统计信息来获取准确或近乎准确的全局列统计信息。清单 9-16 向我们展示了如何做到这一点。

清单 9-16. 增量收集全局统计信息

```
BEGIN
   DBMS_STATS.set_table_prefs(
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT_PART'
     ,pname     => 'INCREMENTAL'
     ,pvalue    => 'TRUE');
END;
/

BEGIN
   DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,partname      => 'P1'
     ,GRANULARITY   => 'APPROX_GLOBAL AND PARTITION'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1');

DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,partname      => 'P2'
     ,granularity   => 'APPROX_GLOBAL AND PARTITION'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1');
END;
/

SELECT column_name
      ,a.num_distinct p1_distinct
      ,b.num_distinct p2_distinct
      ,c.num_distinct tab_distinct
  FROM all_part_col_statistics a
       FULL JOIN all_part_col_statistics b
          USING (owner, table_name, column_name)
       FULL JOIN all_tab_col_statistics c
          USING (owner, table_name, column_name)
 WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT_PART'
       AND a.partition_name = 'P1'
       AND b.partition_name = 'P2';
```

| 列名 | P1_ 唯一值数量 | P2_ 唯一值数量 | 表 _ 唯一值数量 |
| --- | --- | --- | --- |
| CUSTOMER_CATEGORY | 50 | 50 | 50 |
| PRODUCT_CATEGORY | 4 | 6 | 10 |
| AMOUNT_CATEGORY | 3 | 3 | 3 |
| TRANSACTION_AMOUNT | 112 | 166 | 262 |
| DESCRIPTION | 4 | 4 | 4 |
| POSTING_DELAY | 3 | 3 | 3 |
| POSTING_DATE | 6 | 8 | 12 |
| TRANSACTION_DATE | 4 | 6 | 10 |
| TRANSACTION_DATE_TIME | 4 | 6 | 10 |

清单 9-16 使用 `DBMS_STATS.SET_TABLE_PREFS` 过程为分区设置所谓的“概要”。然后使用 `GRANULARITY` 参数值为 `APPROX_GLOBAL AND PARTITION` 来收集分区 `P1` 和 `P2` 的统计信息，此时全局 `NUM_DISTINCT` 统计信息的计算就得到了修正。特别要注意，现在整个表 `POSTING_DATE` 的唯一值数量是 12。

![image](img/sq.jpg) **注意** 你还可以通过 `DBMS_STATS.SET_DATABASE_PREFS`、`DBMS_STATS.SET_SCHEMA_PREFS` 和 `DBMS_STATS.SET_TABLE_PREFS` 设置许多其他首选项。我稍后会解释为什么你应该保持大多数默认设置。

对于唯一值数量较少的列，概要本质上就是这些唯一值的列表，统计信息合并是 100% 准确的，如清单 9-16 所示。当唯一值数量很大时，可能会进行一些复杂的哈希处理，导致合并后的统计信息出现轻微的不准确。在大多数情况下，使用增量维护的全局统计信息将减少收集统计信息所需的总时间，并且产生的全局统计信息完全可以使用。另一方面，概要会占用空间并增加收集分区级统计信息的时间。许多用户仍然喜欢在工作日快速收集分区级统计信息（不使用概要），而在系统较空闲的周末收集全局统计信息。

关于收集分区级统计信息，总结几点：

*   当使用 `INCREMENTAL` 首选项并将 `GRANULARITY` 设置为 `APPROX_GLOBAL AND PARTITION` 时，表/索引的整体 `GLOBAL_STATS` 值将是 `YES`，尽管统计信息实际上是通过合并分区级统计信息获得的。
*   如果分区的 `GLOBAL_STATS=NO`，则意味着该分区的统计信息是从子分区级统计信息合并而来的。`GLOBAL_STATS` 对于子分区级统计信息没有意义。
*   `GLOBAL_STATS` 和 `USER_STATS` 都会被 `DBMS_STATS.SET_xxx_STATS` 过程设置为 `YES`。
*   通过合并分区统计信息来更新全局统计信息并不总是会发生。例如，当表/索引整体已存在统计信息，且这些统计信息的 `GLOBAL_STATS=YES` 时，合并就不会发生。无论这些未合并的统计信息有多旧，除非在收集分区级统计信息时使用了 `GRANULARITY` 的 `APPROX_GLOBAL AND PARTITION` 值，否则它们永远不会被从分区级统计信息合并而来的统计信息更新。显然，在其他一些未文档化的场景中，分区级统计信息合并到全局统计信息的过程也不会发生。
*   如果通过调用 `DBMS_STATS.SET_TABLE_PREFS` 将 `INCREMENTAL` 首选项设置为 `TRUE`，那么收集整个表的统计信息（指定 `GRANULARITY` 为 `GLOBAL`）实际上会收集分区级统计信息并将其合并。因此，在这种情况下，`GLOBAL` 和 `ALL` 似乎是 `GRANULARITY` 的等效值。

