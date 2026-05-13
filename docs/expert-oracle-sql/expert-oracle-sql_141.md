# 分区维护后的查询优化与 CBO 成本估算异常

然而，一旦我们添加了几个分区，我们突然发现未使用提示的查询会使用全表扫描！我们可以看到，由于分区维护操作，全表扫描的成本已经降低。现在全表扫描的成本低于索引扫描的成本，未使用提示查询的执行计划也因此发生了变化。如果新添加的分区被删除，索引访问将再次变得更具吸引力。

这里发生的情况非常不寻常。CBO 估算分区全表扫描成本的方式是：将 `NUM_BLOCKS` 统计值除以从数据字典中获取的当前分区数量。通常，CBO 估算过程的输入是对象统计信息系统统计信息和初始化参数。这条规则的一个例外是数据字典中为表或索引指定的并行度。据我所知，CBO 使用数据字典信息（统计信息除外）的唯一另一种情况是在为分区计算全表扫描成本时。

## 分区消除异常

CBO 意识到对一个分区执行全表扫描将比扫描表中的所有分区花费更少的时间。然而，对于任何类型的索引扫描，应用的逻辑并非如此：在本地索引的一个分区上进行索引扫描的成本与扫描所有分区的成本是相同的；在所有数据库版本（直至并包括 12.1.0.1）中，这一点都是成立的。

此外，如果您使用复合分区，您会发现消除一个分区内的子分区也不会降低全表扫描的成本。当然，这些异常可能在未来得到修正。

在我看来，CBO 开发团队在这里错过了一个改进点。我认为，如果有一个对象统计信息（也许可以称为 `NUM_PARTITIONS`），能反映收集该表对象统计信息时存在的分区数量，那将会更好。但这只是一厢情愿的想法。我们需要面对现实。我们只需将表的 `NUM_BLOCKS` 统计值按分区数量增加的相同比例增加即可。最简单的方法是在收集统计信息时捕获每个分区的平均块数，然后为分区维护操作添加一个 TSTATS 钩子。Listing 20-10 展示了一种实现方法。

Listing 20-10. 通过分区维护操作管理 NUM_BLOCKS

```sql
CREATE OR REPLACE PACKAGE tstats
AS
   -- 其他过程已省略

PROCEDURE adjust_global_stats (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE
     ,p_mode VARCHAR2 DEFAULT 'PMOP');

PROCEDURE gather_table_stats (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE);

-- 其他过程已省略
END tstats;
/

CREATE OR REPLACE PACKAGE BODY tstats
AS
-- 其他过程已省略

PROCEDURE adjust_global_stats (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE
     ,p_mode VARCHAR2 DEFAULT 'PMOP')
   IS
      -- 这个辅助函数更新表中的块数统计信息，以保持分区的平均大小不变。
      -- 我们将这个值保存在未使用的 CACHEDBLK 统计信息中。
      --
      numblks     NUMBER;
      numrows     NUMBER;
      avgrlen     NUMBER;
      cachedblk   NUMBER;
      cachehit    NUMBER;
   BEGIN
      DBMS_STATS.get_table_stats (ownname     => p_owner
                                 ,tabname     => p_table_name
                                 ,numrows     => numrows
                                 ,avgrlen     => avgrlen
                                 ,numblks     => numblks
                                 ,cachedblk   => cachedblk
                                 ,cachehit    => cachehit);

IF p_mode = 'PMOP'
      THEN
         --
         -- 根据 CACHEDBLK（平均段大小）和当前分区数量重置 NUMBLKS。
         --
         IF cachedblk IS NULL
         THEN
            RETURN; -- 没有保存的值
         END IF;

--
         -- 根据当前分区数量和保存的平均段大小重新计算块数。
         -- 避免引用 DBA_SEGMENTS，以防没有权限。
         --
         SELECT cachedblk * COUNT (*)
           INTO numblks
           FROM all_objects
          WHERE owner = p_owner
                AND object_name = p_table_name
                AND object_type = 'TABLE PARTITION';
      ELSIF p_mode = 'GATHER'
      THEN
         --
         -- 根据 NUMBLKS 和当前分区数量，将平均段大小保存在 CACHEDBLK 中。
         --
         SELECT numblks / COUNT (*), TRUNC (numblks / COUNT (*)) * COUNT (*)
           INTO cachedblk, numblks
           FROM all_objects
          WHERE owner = p_owner
                AND object_name = p_table_name
                AND object_type = 'TABLE PARTITION';
      ELSE
         RAISE PROGRAM_ERROR;
      -- 只有在 p_mode 未设置为 PMOP 或 GATHER 时才会到达这里
      END IF;

DBMS_STATS.set_table_stats (ownname     => p_owner
                                 ,tabname     => p_table_name
                                 ,numblks     => numblks
                                 ,cachedblk   => cachedblk
                                 ,force       => TRUE);
   END adjust_global_stats;

PROCEDURE gather_table_stats (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE)
   IS
   BEGIN
      DBMS_STATS.unlock_table_stats (ownname   => p_owner
                                    ,tabname   => p_table_name);

FOR r IN (SELECT *
                  FROM all_tables
                 WHERE owner = p_owner AND table_name = p_table_name)
      LOOP
         DBMS_STATS.gather_table_stats (
            ownname       => p_owner
           ,tabname       => p_table_name
           ,granularity   => CASE r.partitioned
                               WHEN 'YES' THEN 'GLOBAL'
                               ELSE 'ALL'
                            END
           ,method_opt    => 'FOR ALL COLUMNS SIZE 1');

adjust_column_stats_v2 (p_owner        => p_owner
                                ,p_table_name   => p_table_name);
```


```sql
IF r.partitioned = 'YES'
         THEN
            adjust_global_stats (p_owner        => p_owner
                                ,p_table_name   => p_table_name
                                ,p_mode         => 'GATHER');
         END IF;
      END LOOP;

DBMS_STATS.lock_table_stats (ownname => p_owner, tabname => p_table_name);
   END gather_table_stats;

-- 其他过程被省略
END tstats;
/
```

清单 20-10 展示了 `TSTATS` 包中的更多过程。`TSTATS.GATHER_TABLE_STATS` 是一个封装例程，用于收集表的统计信息并调整列统计信息。对于分区表，仅收集全局统计信息，并且每个分区的平均块数通过 `TSTATS.ADJUST_TABLE_STATS` 过程保存在未使用的 `CACHEDBLK` 统计信息中。

![image](img/sq.jpg) **注意** 使用 Oracle 保留供未来使用的统计信息是非常非常“淘气”的做法。你们中的一些人可能会对在书中看到这种实践感到震惊。严格来说，我应该将我的私有数据保存在合法的地方，例如数据字典之外的常规表中。然而，在实际工作中，我发现使用这个未使用的统计信息的诱惑是难以抗拒的。该统计信息在调用 `DBMS_STATS.EXPORT_TABLE_STATS` 和 `DBMS_STATS.IMPORT_TABLE_STATS` 时会被保留下来，这一事实似乎使得该过程在未来某个时间点中断的风险值得承担。该统计信息列在所有版本（包括 `12.1.0.1`）中都是未使用的。

`TSTATS.ADJUST_TABLE_STATS` 例程以两种模式运行。当作为统计信息收集过程的一部分被调用时，它会计算并保存每个分区的平均块数。当执行分区维护操作时，会使用默认模式 `"PMOP"` 调用 `TSTATS.ADJUST_TABLE_STATS` 过程，这会导致基于当前分区数和保存的每个分区块数重新计算 `NUM_BLOCKS` 统计信息。

清单 20-11 展示了如何将 `TSTATS.ADJUST_TABLE_STATS` 集成到我们的分区维护过程中，以避免这些分区维护操作破坏我们的执行计划。

清单 20-11. 在分区维护过程中调整 `NUM_BLOCKS`

```sql
--
-- 首先删除空分区
--
ALTER TABLE statement_part DROP PARTITION p3;
ALTER TABLE statement_part DROP PARTITION p4;
ALTER TABLE statement_part DROP PARTITION p5;
ALTER TABLE statement_part DROP PARTITION p6;
--
-- 现在收集完整分区的统计信息
--

BEGIN
   tstats.gather_table_stats (
      p_owner        => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,p_table_name   => 'STATEMENT_PART');
END;
/

--
-- 在分区维护前检查执行计划和成本
--
SELECT COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-06';

| Id  | Operation               | Name               | Rows  | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT        |                    |     1 |    29   (0)|       |       |
|   1 |  SORT AGGREGATE         |                    |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                    |    50 |    29   (0)|     2 |     2 |
|*  3 |    INDEX FAST FULL SCAN | STATEMENT_PART_IX1 |    50 |    29   (0)|     2 |     2 |

SELECT /*+ full(t) */
       COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-06';

| Id  | Operation               | Name           | Rows  | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT        |                |     1 |    69   (0)|       |       |
|   1 |  SORT AGGREGATE         |                |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                |    50 |    69   (0)|     2 |     2 |
|*  3 |    TABLE ACCESS FULL    | STATEMENT_PART |    50 |    69   (0)|     2 |     2 |

--
-- 现在重新创建空分区

ALTER TABLE statement_part
   ADD PARTITION p3 VALUES LESS THAN (DATE '2013-01-12');

ALTER TABLE statement_part
   ADD PARTITION p4 VALUES LESS THAN (DATE '2013-01-13');

ALTER TABLE statement_part
   ADD PARTITION p5 VALUES LESS THAN (DATE '2013-01-14');

ALTER TABLE statement_part
   ADD PARTITION p6 VALUES LESS THAN (maxvalue);

--
-- 最后调用 TSTATS 钩子来调整 NUM_BLOCKS 统计信息
--

BEGIN
   tstats.adjust_global_stats (
      p_owner        => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,p_table_name   => 'STATEMENT_PART');
END;
/

--
-- 现在重新检查执行计划和成本
--

SELECT COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-06';

| Id  | Operation               | Name               | Rows  | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT        |                    |     1 |    29   (0)|       |       |
|   1 |  SORT AGGREGATE         |                    |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                    |    50 |    29   (0)|     2 |     2 |
|*  3 |    INDEX FAST FULL SCAN | STATEMENT_PART_IX1 |    50 |    29   (0)|     2 |     2 |

SELECT COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-13';

| Id  | Operation               | Name               | Rows  | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT        |                    |     1 |    29   (0)|       |       |
|   1 |  SORT AGGREGATE         |                    |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                    |    50 |    29   (0)|     5 |     5 |
|*  3 |    INDEX FAST FULL SCAN | STATEMENT_PART_IX1 |    50 |    29   (0)|     5 |     5 |

SELECT /*+ full(t) */ COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-06';

| Id  | Operation               | Name           | Rows  | Cost (%CPU)| Pstart| Pstop |

|   0 | SELECT STATEMENT        |                |     1 |    69   (0)|       |       |
|   1 |  SORT AGGREGATE         |                |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                |    50 |    69   (0)|     2 |     2 |
|*  3 |    TABLE ACCESS FULL    | STATEMENT_PART |    50 |    69   (0)|     2 |     2 |

```

在收集和调整统计信息后，全表扫描的原始成本由于舍入问题发生了一些变化，但我们可以看到，通过将 `TSTATS.ADJUST_TABLE_STATS` 集成到我们的分区维护过程中，全表扫描的成本保持不变，并且未加提示的查询继续使用索引。

## 临时表

老实说：对于 TSTATS 方法而言，临时表是一个具有挑战性的问题，原因有二：
*   你无法通过独立作业轻松地为临时表收集统计信息，因为临时表当时会是空的。
*   永久表的大小很少会按数量级变化，而临时表则很常见，可能一次容纳 1,000,000 行，不久后又只容纳 3 行。

当临时表在抽取-转换-加载（ETL）过程中用作暂存表时，这些困难往往显而易见。实际上，永久表有时也在 ETL 过程中用作暂存表，本节讨论的问题同样适用于这类表。

### 动态采样的利弊

尽管这完全违背了 TSTATS 的理念，但处理剧烈变化基数的一个务实方法是依赖动态采样。除非你明确禁用它，否则动态采样会在任何没有对象统计信息的表上自动发生，这包括临时表。

动态采样的好处在于它相当智能，完全理解列相关性等概念，因此您无需担心甚至无需了解**扩展统计**是什么！事实上，当您的语句被解析时，您很有可能获得一个相当不错的执行计划。而如果下周您使用临时表中完全不同的内容重新解析该语句，您也很可能会获得一个迥然不同、但非常适用于您修改后工作负载的计划。

![image](img/sq.jpg) **提示**：如果您运行的是 `12cR1` 或更高版本，并且有多个针对具有相同内容的同一临时表的查询，您可以通过收集特定于会话的临时表统计信息来避免重复的动态采样。详情请参阅《PL/SQL 包和类型参考手册》中 `DBMS_STATS.SET_TABLE_PREFS` 过程的说明，并查找 `GLOBAL_TEMP_TABLE_STATS` 首选项。然而，如果您使用会话特定统计信息作为动态采样的替代方案，则可能需要设置**扩展统计**！

这一切都很好，但我相信您能看出动态采样的潜在缺点。我们仍然面临着最初促使我们开发 TSTATS 的性能不佳的风险，具体如下：

*   动态采样产生的执行计划及其最终带来的性能都无法确定地预测。
*   一个会话创建的执行计划，可能被相同或不同的会话在临时表内容与语句解析时截然不同的情况下重复使用。

我必须承认，当我最初为 AJAX 应用程序实现 TSTATS 时，我对这些担忧**视而不见**，并寄希望于通过稳定所有不涉及临时表的语句的执行计划，并对剩余语句依赖动态采样，就足以让 AJAX 提供稳定的服务。不幸的是，我错了，最终不得不寻找一种方法来稳定涉及临时表的查询的计划。我最终放弃了动态采样，改为为临时表伪造统计信息。

### 为临时表伪造统计信息

思路很简单：使用 `DBMS_STATS.SET_TABLE_STATS`、`DBMS_STATS.SET_COLUMN_STATS`，并在适当时使用 `DBMS_STATS.SET_INDEX_STATS` 来为全局临时表及其关联的列和索引设置统计信息。这样，您就可以确保涉及全局临时表的执行计划像您所有其他查询一样是固定且可测试的。当然，问题在于如何确定该伪造哪些统计信息。

为了说明问题，让我们看看 清单 20-12，它执行了一个临时表与永久表的合并操作。

清单 20-12. 将临时表合并到永久表中

```sql
CREATE GLOBAL TEMPORARY TABLE payments_temp
AS
   SELECT *
     FROM payments
    WHERE 1 = 0;

MERGE /*+ cardinality(t 3 ) */
     INTO  payments p
     USING payments_temp t
        ON (p.payment_id = t.payment_id)
WHEN MATCHED
THEN
   UPDATE SET p.employee_id = t.employee_id
             ,p.special_flag = t.special_flag
             ,p.paygrade = t.paygrade
             ,p.payment_date = t.payment_date
             ,p.job_description = t.job_description
WHEN NOT MATCHED
THEN
   INSERT     (payment_id
              ,special_flag
              ,paygrade
              ,payment_date
              ,job_description)
       VALUES (t.payment_id
              ,t.special_flag
              ,t.paygrade
              ,t.payment_date
              ,t.job_description);
```

```
| Id  | Operation                      | Name          | Rows  | Cost (%CPU)|

|   0 | MERGE STATEMENT                |               |     3 |     5   (0)|
|   1 |  MERGE                         | PAYMENTS      |       |            |
|   2 |   VIEW                         |               |       |            |
|   3 |    NESTED LOOPS OUTER          |               |     3 |     5   (0)|
|   4 |     TABLE ACCESS FULL          | PAYMENTS_TEMP |     3 |     2   (0)|
|   5 |     TABLE ACCESS BY INDEX ROWID| PAYMENTS      |     1 |     1   (0)|
|   6 |      INDEX UNIQUE SCAN         | PAYMENTS_PK   |     1 |     0   (0)|
```

```sql
MERGE /*+ cardinality(t 10000 ) */
     INTO  payments p
     USING payments_temp t
        ON (p.payment_id = t.payment_id)
WHEN MATCHED
THEN
   UPDATE SET p.employee_id = t.employee_id
             ,p.special_flag = t.special_flag
             ,p.paygrade = t.paygrade
             ,p.payment_date = t.payment_date
             ,p.job_description = t.job_description
WHEN NOT MATCHED
THEN
   INSERT     (payment_id
              ,special_flag
              ,paygrade
              ,payment_date
              ,job_description)
       VALUES (t.payment_id
              ,t.special_flag
              ,t.paygrade
              ,t.payment_date
              ,t.job_description);
```

```
| Id  | Operation            | Name          | Rows  | Cost (%CPU)|

|   0 | MERGE STATEMENT      |               | 10000 |   176   (1)|
|   1 |  MERGE               | PAYMENTS      |       |            |
|   2 |   VIEW               |               |       |            |
|   3 |    HASH JOIN OUTER   |               | 10000 |   176   (1)|
|   4 |     TABLE ACCESS FULL| PAYMENTS_TEMP | 10000 |     2   (0)|
|   5 |     TABLE ACCESS FULL| PAYMENTS      |   120K|   174   (1)|
```

清单 20-12 展示了两个 `MERGE` 语句及其相关的执行计划。两个语句的唯一区别在于 `CARDINALITY` 提示的参数，一个案例中是 `3`，另一个案例中是 `10,000`。我们可以看到，在这两种情况下，CBO 都选择将临时表作为连接中的驱动表。在基数为 `3` 的案例中，CBO 倾向于使用**嵌套循环连接**，而在基数估计为 `10,000` 的案例中，则更偏好**哈希连接**。

我们希望伪造我们的统计信息，使得 CBO 能在无需我们使用提示的情况下选择一个合理的执行计划。但最优执行计划似乎取决于临时表中的行数，那么我们该怎么办呢？

回答这个问题的关键在于回答另外两个问题：

*   当临时表中有 `10,000` 行时，如果我们使用了嵌套循环连接，性能会有多糟糕？
*   当表中只有 `3` 行时，如果我们使用了哈希连接，性能会有多糟糕？

清单 20-13 展示了我们如何开始思考这个问题。

清单 20-13. 评估错误基数估计的影响

```sql
MERGE /*+ cardinality(t 3) leading(t) use_hash(p) no_swap_join_inputs(p) */
     INTO  payments p
     USING payments_temp t
        ON (p.payment_id = t.payment_id)
WHEN MATCHED
THEN
   UPDATE SET p.employee_id = t.employee_id
             ,p.special_flag = t.special_flag
             ,p.paygrade = t.paygrade
             ,p.payment_date = t.payment_date
             ,p.job_description = t.job_description
WHEN NOT MATCHED
THEN
   INSERT     (payment_id
              ,special_flag
              ,paygrade
              ,payment_date
              ,job_description)
       VALUES (t.payment_id
              ,t.special_flag
              ,t.paygrade
              ,t.payment_date
              ,t.job_description);
```

```
| Id  | Operation            | Name          | Rows  | Cost (%CPU)|
```

（注：此处原文似乎不完整，因此译文也相应结束。如果需要翻译完整的段落，请提供完整的原文内容。）

---
**请注意**：为确保没有遗漏任何内容，并方便您对照核查，我已将处理过程与原始文本结构完全对应。如果您需要核对任何特定部分的翻译，可以随时指出。


