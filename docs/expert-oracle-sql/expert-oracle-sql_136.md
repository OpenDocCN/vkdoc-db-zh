# 优化基于时间列的统计信息

正如我们在第 6 章中所讨论的，基于时间的列（如日期或序列号），如果统计信息未定期收集，可能会导致执行计划发生不期望的变化。您可以通过移除`HIGH_VALUE`和`LOW_VALUE`这两个统计信息列来防止这些不期望的执行计划变更。然而，除了少数我们稍后将讨论的例外情况外，即使该列并非基于时间的，移除`HIGH_VALUE`和`LOW_VALUE`统计信息列通常也没有坏处。

![image](img/sq.jpg) **注意** 在本书定稿之际，我发现版本 12.1.0.1 中引入了一个错误：当两个表的连接列都缺少`HIGH_VALUE`和`LOW_VALUE`时，会导致连接基数计算错误。此问题的一个解决方法是设置一个极高的`HIGH_VALUE`和一个极低的`LOW_VALUE`。这个解决方法包含在可下载的资料中，但为了避免不必要的复杂性，本书默认假设将`HIGH_VALUE`和`LOW_VALUE`设置为`NULL`不会引发问题。

我通常的建议是，在设置 TSTATS 时，首先为您模式中的所有对象列移除`HIGH_VALUE`和`LOW_VALUE`统计信息列，除非您事先已知存在我稍后将介绍的例外情况。如果您不了解某些例外情况，也不必担心。您会在测试过程中发现它们。

代码清单 20-1 展示了一个名为`TSTATS`的包中的过程`ADJUST_COLUMN_STATS_V1()`，用于从指定表中移除`HIGH_VALUE`和`LOW_VALUE`统计信息列。

代码清单 20-1. 移除`HIGH_VALUE`和`LOW_VALUE`统计信息列

```sql
CREATE OR REPLACE PACKAGE tstats
AS
-- 其他未显示的过程

PROCEDURE adjust_column_stats_v1 (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE);

-- 其他未显示的过程
END tstats;

CREATE OR REPLACE PACKAGE BODY tstats
AS
-- 其他未显示的过程

PROCEDURE adjust_column_stats_v1 (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE)
   AS
      CURSOR c1
      IS
         SELECT *
           FROM all_tab_col_statistics
          WHERE     owner = p_owner
                AND table_name = p_table_name
                AND last_analyzed IS NOT NULL;
   BEGIN
      FOR r IN c1
      LOOP
         DBMS_STATS.delete_column_stats (ownname         => r.owner
                                        ,tabname         => r.table_name
                                        ,colname         => r.column_name
                                        ,cascade_parts   => TRUE
                                        ,no_invalidate   => TRUE
                                        ,force           => TRUE);
         DBMS_STATS.set_column_stats (ownname         => r.owner
                                     ,tabname         => r.table_name
                                     ,colname         => r.column_name
                                     ,distcnt         => r.num_distinct
                                     ,density         => r.density
                                     ,nullcnt         => r.num_nulls
                                     ,srec            => NULL -- 无 HIGH_VALUE/LOW_VALUE
                                     ,avgclen         => r.avg_col_len
                                     ,no_invalidate   => FALSE
                                     ,force           => TRUE);
      END LOOP;
   END adjust_column_stats_v1;
-- 其他未显示的过程
END tstats;
```

代码清单 20-1 中展示的过程会遍历指定表中所有具有列统计信息的列，然后删除并重新添加列统计信息，但不含`HIGH_VALUE`和`LOW_VALUE`部分。

## `NUM_DISTINCT=1`的列

我发现了一个相当奇特的 CBO 行为，它与那些根据对象统计信息，每一行都具有相同值（且该列值不为`NULL`）的列有关。让我举个例子。代码清单 16-4 展示了如何通过使用`NULL`来表示列中最常见的值，以最小化索引大小。该示例使用了一个名为`SPECIAL_FLAG`的列。如果行是特殊的，则`SPECIAL_FLAG`设置为`Y`；否则`SPECIAL_FLAG`为`NULL`。在这种情况下，`SPECIAL_FLAG`的不同值数量为 1，因为在 CBO（或数据库理论家）看来，`NULL`不算作一个值。

在收集统计信息后，`SPECIAL_VALUE`的`HIGH_VALUE`和`LOW_VALUE`都将为`Y`，而`NUM_DISTINCT`和`DENSITY`统计信息列都将为 1。这不无道理，当`NUM_DISTINCT`为 1 而`HIGH_VALUE`和`LOW_VALUE`不同时，CBO 会感到有些困惑！在这种情况下，CBO 会认为没有行匹配您可能提供的任何等式谓词。在 12cR1 之前，当`HIGH_VALUE`和`LOW_VALUE`统计信息被移除时，也会看到相同的行为，如代码清单 20-2 所示。

代码清单 20-2. CBO 在`NUM_DISTINCT=1`时的特殊处理

```sql
CREATE TABLE payments
(
   payment_id        INTEGER
  ,employee_id       INTEGER
  ,special_flag      CHAR (1)
  ,paygrade          INTEGER
  ,payment_date      DATE
  ,job_description   VARCHAR2 (50)
)
PCTFREE 0;
```



