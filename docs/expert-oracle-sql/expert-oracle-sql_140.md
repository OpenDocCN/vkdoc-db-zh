# 复制分区统计信息：机制与陷阱

## 问题：为新分区提供统计信息

当基于日期等范围进行分区时，新分区最初可能缺少有效的统计信息。通过查询 `all_part_col_statistics` 视图可以确认这一点。例如，分区 `P1`、`P2`、`P3` 的 `DESCRIPTION` 列都只有 4 个不同值。

```sql
SELECT partition_name, num_distinct
  FROM all_part_col_statistics
 WHERE owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT_PART'
       AND column_name = 'DESCRIPTION'
UNION ALL
SELECT 'TABLE' partition_name, num_distinct
  FROM all_tab_col_statistics
 WHERE owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
       AND table_name = 'STATEMENT_PART'
       AND column_name = 'DESCRIPTION';
```

```
PARTITION_NAME   NUM_DISTINCT
---------------- ------------
P1                          4
P2                          4
P3                          4
TABLE                       4
```

因此，针对新分区（如 `P2`）的查询现在可以获得合理的基数估计。

```sql
SELECT COUNT (*)
  FROM statement_part PARTITION (p2)
 WHERE description = 'Flight';
```

```
| Id  | Operation               | Name           | Rows  | Pstart| Pstop |
------------------------------------------------------------------------
|   0 | SELECT STATEMENT        |                |     1 |       |       |
|   1 |  SORT AGGREGATE         |                |     1 |       |       |
|   2 |   PARTITION RANGE SINGLE|                |    50 |     2 |     2 |
|*  3 |    TABLE ACCESS FULL    | STATEMENT_PART |    50 |     2 |     2 |
```

## 解决方案：使用 `DBMS_STATS.COPY_TABLE_STATS`

**Listing 20-6** 重新创建了我们在 **Chapter 9** 中创建的表 `STATEMENT_PART`，这次包含三个分区。首先，我删除任何现有统计信息，然后仅收集分区 `P1` 的统计信息。接着，我使用 `DBMS_STATS.COPY_TABLE_STATS` 将统计信息从分区 `P1` 复制到分区 `P2` 和 `P3`。结果表明，我们不仅为 `P2` 获得了一套可用的统计信息，而且 `STATEMENT_PART` 的全局统计信息也已生成。这看起来很不错。然而，事情并不像乍看起来那样简单直接。

## `DBMS_STATS.COPY_TABLE_STATS` 的缺陷

**Listing 20-7** 突显了使用 `DBMS_STATS.COPY_TABLE_STATS` 可能出现的主要问题。

**Listing 20-7. `DBMS_STATS.COPY_TABLE_STATS` 的问题**

```sql
DELETE FROM statement_part
      WHERE transaction_date NOT IN (DATE '2013-01-01', DATE '2013-01-06');

BEGIN
   DBMS_STATS.delete_table_stats (
      ownname           => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname           => 'STATEMENT_PART'
     ,cascade_parts     => TRUE
     ,cascade_indexes   => TRUE
     ,cascade_columns   => TRUE);

   DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,partname      => 'P1'
     ,granularity   => 'PARTITION'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1');

   DBMS_STATS.copy_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,srcpartname   => 'P1'
     ,dstpartname   => 'P2');

   DBMS_STATS.copy_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,srcpartname   => 'P1'
     ,dstpartname   => 'P3');
END;
/

CREATE OR REPLACE FUNCTION convert_date_stat (raw_value RAW)
   RETURN DATE
IS
   date_value   DATE;
BEGIN
   DBMS_STATS.convert_raw_value (rawval => raw_value, resval => date_value);
   RETURN date_value;
END convert_date_stat;
/

SELECT column_name
        ,partition_name
        ,convert_date_stat (low_value) low_value
        ,convert_date_stat (high_value) high_value
        ,num_distinct
    FROM all_part_col_statistics
   WHERE owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
         AND table_name = 'STATEMENT_PART'
         AND column_name IN ('TRANSACTION_DATE', 'POSTING_DATE')
ORDER BY column_name DESC, partition_name;
```

```
COLUMN_NAME          PARTITION_NAME       LOW_VALUE HIGH_VALU NUM_DISTINCT
-------------------- -------------------- --------- --------- ------------
TRANSACTION_DATE     P1                   01-JAN-13 01-JAN-13            1
TRANSACTION_DATE     P2                   05-JAN-13 05-JAN-13            1
TRANSACTION_DATE     P3                   11-JAN-13 11-JAN-13            1
POSTING_DATE         P1                   01-JAN-13 03-JAN-13            3
POSTING_DATE         P2                   01-JAN-13 03-JAN-13            3
POSTING_DATE         P3                   01-JAN-13 03-JAN-13            3
```

```sql
SELECT COUNT (*)
  FROM statement_part
 WHERE transaction_date = DATE '2013-01-06';
```

```
| Id  | Operation               | Name               | Rows  |
----------------------------------------------------------------
|   0 | SELECT STATEMENT        |                    |     1 |
|   1 |  SORT AGGREGATE         |                    |     1 |
|   2 |   PARTITION RANGE SINGLE|                    |     1 |
|   3 |    INDEX RANGE SCAN     | STATEMENT_PART_IX1 |     1 |
```

**Listing 20-7** 首先从表 `STATEMENT_PART` 中删除行，使每个分区仅包含一个 `TRANSACTION_DATE`。虽然在可以保证分区列的值在每个分区中唯一时，使用列表分区更有意义，但在这种情况下使用范围分区也并不少见。对于周末没有数据的业务来说，这些范围覆盖超过 24 小时也很常见。

重新收集分区 `P1` 的统计信息并将其复制到 `P2` 和 `P3` 后，我们看到了第一个问题：`DBMS_STATS.COPY_TABLE_STATS` 过程并非盲目地将统计信息从 `P2` 复制到 `P3`。它进行了它认为合理的调整：分区 `P2` 中 `TRANSACTION_DATE` 列的 `HIGH_VALUE` 和 `LOW_VALUE` 列统计信息都被设置为 1 月 5 日。在 11.2.0.2 和 11.2.0.3 版本中，`HIGH_VALUE` 会被设置为 1 月 11 日，以反映分区的范围，但这导致了一个问题（错误 14607573），因为分区 `P2` 中 `TRANSACTION_DATE` 的 `NUM_DISTINCT` 列统计值为 1，而 `HIGH_VALUE` 和 `LOW_VALUE` 却不同。无论如何，由于分区 `P2` 中行的 `TRANSACTION_DATE` 值实际上是 1 月 6 日，即使在 12cR1 中，`HIGH_VALUE` 和 `LOW_VALUE` 仍然具有误导性。

尽管 `DBMS_STATS.COPY_TABLE_STATS` 试图调整分区列的列统计信息，但它没有尝试调整其他列的统计信息，这导致了第二个问题。`POSTING_DATE` 列与 `TRANSACTION_DATE` 相关，每当调整其中一个列的统计信息时，从逻辑上讲也应该调整另一个；如果不使用 TSTATS，我建议在对按日期分区的表使用 `DBMS_STATS.COPY_TABLE_STATS` 时，手动调整所有基于时间的列的 `HIGH_VALUE` 和 `LOW_VALUE`。

## `DBMS_STATS.COPY_TABLE_STATS` 的逻辑矛盾

我第一次实现 T 是在 2009 年。第一个应用程序，我称之为 AJAX，包含了按一个我称之为 `BUSINESS_DATE` 的列进行分区的表。AJAX 的程序员遵循一个特定的编码标准：所有 SQL 语句*必须*对访问的每个分区表包含一个关于 `BUSINESS_DATE` 的等式谓词。这个标准的意图是避免任何对全局统计信息的依赖。分区级统计信息被独家使用，随着新分区的创建，使用 `DBMS_STATS.COPY_TABLE_STATS` 过程为这些新分区生成统计信息。`BUSINESS_DATE` 的列统计信息被手动调整。

TSTATS 实现是成功的，并为其他应用程序转换到 TSTATS 铺平了道路。我当时非常高兴，并感谢幸运之星带来了美妙的 `DBMS_STATS.COPY_TABLE_STATS` 过程。


几年后，某个我无法想起缘由的清晨，我在床上醒来，脑海中闪过一个惊人的想法：使用分区级统计信息的唯一合理原因，应该是分区之间存在显著差异。但 `DBMS_STATS.COPY_TABLE_STATS` 仅在所有分区都相似时才有效。因此，`DBMS_STATS.COPY_TABLE_STATS` 仅在**不需要**它的时候才可用！

我醒来时常常有这种想法：仿佛整个世界都忽略了某些显而易见的事情并为之疯狂。通常，在我揉掉睡意、喝杯茶、吃完早餐麦片后，我那昏昏沉沉想法中的致命缺陷就会变得清晰起来。然而，这次我却想不出任何缺陷。我依然相信，尽管它被广泛使用，但 `DBMS_STATS.COPY_TABLE_STATS` 的有效用例寥寥无几。

无论你是否采用 TSTATS 方法，你在分区表全局统计信息方面面临的问题，与未分区表的统计信息问题并无太大不同：基于时间的列会导致执行计划变更，相关列和表达式会导致基数估计错误，列中倾斜的值可能要求同一语句有多个执行计划。分区级统计信息并不能妥善解决所有这些问题，正如我在第 9 章的列表 9-17 至 9-21 中所展示的那样。

不幸的是，我们不能总是像对待未分区表那样对待分区表。我们不能总是简单地删除分区和子分区统计信息然后继续生活。现在让我停止讨论 `DBMS_STATS.COPY_TABLE_STATS`，更详细地审视分区表上的全局统计信息，以便我们理解全局统计信息**会**和**不会**带来哪些问题。

## 使用全局统计信息的基数估计

让我们尝试另一种方法来管理 `STATEMENT_PART` 上的对象统计信息。列表 20-8 完全舍弃了分区级统计信息，仅设置了全局统计信息。

### 列表 20-8. STATEMENT_PART 上的全局统计信息

```sql
BEGIN
   DBMS_STATS.delete_table_stats (
      ownname           => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname           => 'STATEMENT_PART'
     ,cascade_parts     => TRUE
     ,cascade_indexes   => TRUE
     ,cascade_columns   => TRUE);

   DBMS_STATS.gather_table_stats (
      ownname       => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname       => 'STATEMENT_PART'
     ,granularity   => 'GLOBAL'
     ,method_opt    => 'FOR ALL COLUMNS SIZE 1');

   tstats.adjust_column_stats_v2 (
      p_owner        => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,p_table_name   => 'STATEMENT_PART');
END;
/
```

```sql
SELECT COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-06';
```

| Id  | 操作                | 名称               | 行数  | 代价 (%CPU)| Pstart| Pstop |
| --- | ----------------------- | ------------------ | ----- | ---------- | ----- | ----- |
|   0 | SELECT STATEMENT        |                    |     1 |    29   (0)|       |       |
|   1 |  SORT AGGREGATE         |                    |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                    |    50 |    29   (0)|     2 |     2 |
|*  3 |    INDEX FAST FULL SCAN | STATEMENT_PART_IX1 |    50 |    29   (0)|     2 |     2 |

```sql
SELECT /*+ full(t) */
       COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-06';
```

| Id  | 操作                | 名称           | 行数  | 代价 (%CPU)| Pstart| Pstop |
| --- | ----------------------- | -------------- | ----- | ---------- | ----- | ----- |
|   0 | SELECT STATEMENT        |                |     1 |    47   (0)|       |       |
|   1 |  SORT AGGREGATE         |                |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                |    50 |    47   (0)|     2 |     2 |
|*  3 |    TABLE ACCESS FULL    | STATEMENT_PART |    50 |    47   (0)|     2 |     2 |

`STATEMENT_PART` 的全局统计信息已使用列表 20-3 介绍的 `TSTATS.ADJUST_COLUMN_STATS_V2` 过程进行了调整。针对 `STATEMENT_PART` 查询产生的基数估计非常合理，就像针对未分区表一样。请注意，我们的查询使用了我们创建的索引，而基于提示的全表扫描的代价更高。但如果我们增加几个分区会怎样呢？列表 20-9 将展示这一点。

### 列表 20-9. 向 STATEMENT_PART 添加分区

```sql
ALTER TABLE statement_part
   SPLIT PARTITION p3
      AT (DATE '2013-01-12')
      INTO (PARTITION p3, PARTITION p4);

ALTER TABLE statement_part
   SPLIT PARTITION p4
      AT (DATE '2013-01-13')
      INTO (PARTITION p4, PARTITION p5);

ALTER TABLE statement_part
   SPLIT PARTITION p5
      AT (DATE '2013-01-14')
      INTO (PARTITION p5, PARTITION p6);
```

```sql
SELECT COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-06';
```

| Id  | 操作                | 名称           | 行数  | 代价 (%CPU)| Pstart| Pstop |
| --- | ----------------------- | -------------- | ----- | ---------- | ----- | ----- |
|   0 | SELECT STATEMENT        |                |     1 |    25   (0)|       |       |
|   1 |  SORT AGGREGATE         |                |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                |    50 |    25   (0)|     2 |     2 |
|*  3 |    TABLE ACCESS FULL    | STATEMENT_PART |    50 |    25   (0)|     2 |     2 |

```sql
SELECT COUNT (*)
  FROM statement_part t
 WHERE transaction_date = DATE '2013-01-13';
```

| Id  | 操作                | 名称           | 行数  | 代价 (%CPU)| Pstart| Pstop |
| --- | ----------------------- | -------------- | ----- | ---------- | ----- | ----- |
|   0 | SELECT STATEMENT        |                |     1 |    25   (0)|       |       |
|   1 |  SORT AGGREGATE         |                |     1 |            |       |       |
|   2 |   PARTITION RANGE SINGLE|                |    50 |    25   (0)|     5 |     5 |
|*  3 |    TABLE ACCESS FULL    | STATEMENT_PART |    50 |    25   (0)|     5 |     5 |

在我们向 `STATEMENT_PART` 添加分区后，我们可以看到我们的示例查询在分区维护前后的基数估计都是相同的 50 行，而针对新创建分区的类似查询的基数估计也是 50 行。请注意，`TRANSACTION_DATE` 的 `NUM_DISTINCT` 值仅为 2，即使存在对应更多 `TRANSACTION_DATE` 值的分区。我强调这一点是因为许多与我交谈过的人都曾对此类差异表示担忧。只要列 `TRANSACTION_DATE` 的 `NUM_DISTINCT` 值与表的 `NUM_ROWS` 值保持一致，每个 `TRANSACTION_DATE` 的行数就会是合理的。

如果分区维护没有改变基数估计，为什么列表 20-8 和 20-9 中的执行计划不同？答案与基数无关，而完全与代价有关。

## 计算表分区的全表扫描代价

当分区表的总分区数量发生变化而其对象统计信息未改变时，会产生一个显著的复杂情况。再看一下列表 20-8 和 20-9 中的执行计划。列表 20-8 中未加提示的查询使用了索引。使用 `FULL` 提示的变体所关联的代价更高——到目前为止没有问题。



