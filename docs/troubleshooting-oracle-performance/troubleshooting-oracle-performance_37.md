# 函数 display_cursor

函数 `display_cursor()` 返回存储在库缓存中的执行计划。它从 Oracle Database 10*g* 开始可用。与函数 `display()` 一样，其返回值是集合 `dbms_xplan_type_table` 的一个实例。该函数具有以下输入参数：

*   `sql_id` 指定要返回其执行计划的父游标。默认值为 `NULL`。如果使用默认值，则返回当前会话最后执行的 SQL 语句的执行计划。
*   `cursor_child_no` 指定子游标号，它与 `sql_id` 共同标识要返回其执行计划的子游标。默认值为 0。如果指定为 `NULL`，则返回由参数 `sql_id` 标识的父游标的所有子游标。
*   `format` 指定显示哪些信息。支持与函数 `display()` 的 `format` 参数相同的值。此外，如果执行统计信息可用（换句话说，如果初始化参数 `statistics_level` 设置为 `all` 或在 SQL 语句中指定了提示 `gather_plan_statistics`），则也支持 表 6-4 中描述的修饰符。默认值为 `typical`。

要使用 `display_cursor()` 函数，调用者需要在以下动态性能视图上具有 `SELECT` 权限：`v$session`、`v$sql`、`v$sql_plan` 和 `v$sql_plan_statistics_all`。角色 `select_catalog_role` 和系统权限 `select any dictionary` 等提供这些权限。

## 表 6-4. 参数 `format` 接受的修饰符

| **值** | **描述** |
| --- | --- |
| `allstats`* | 这是 `iostats memstats` 的简写。 |
| `iostats`* | 控制 I/O 统计信息的显示。 |
| `last`* | 默认情况下，显示所有执行的累计统计信息。如果指定了此值，则仅显示最后一次执行的统计信息。 |
| `memstats`* | 控制与 PGA 相关的统计信息的显示。 |
| `runstats_last` | 与 `iostats last` 相同。仅在 Oracle Database 10*g* Release 1 中可用。 |
| `runstats_tot` | 与 `iostats` 相同。仅在 Oracle Database 10*g* Release 1 中可用。 |
| \* 此值仅在 Oracle Database 10*g* Release 2 及以上版本中可用*。 |

以下示例展示了一个使用提示 `gather_plan_statistics` 来启用生成执行统计信息的查询。然后，函数 `display_cursor()` 被指示显示最后一次执行的 I/O 统计信息。请注意，由于没有物理读写操作，因此只显示了逻辑读操作（`Buffers`）。以下是脚本 `display_cursor.sql` 生成的输出摘录：

```sql
SQL> SELECT /*+ gather_plan_statistics */ count(*)
  2  FROM t
  3  WHERE mod(n,19) = 0;
```

```text
  COUNT(*)
----------
        52
```

```sql
SQL> SELECT * FROM table(dbms_xplan.display_cursor(NULL,NULL, 'iostats last'));
```

```text
PLAN_TABLE_OUTPUT
------------------------------------------------------------------------------------

EXPLAINED SQL STATEMENT:
------------------------
SELECT /*+ gather_plan_statistics */ count(*) FROM t WHERE mod(n,19) = 0

Plan hash value: 2966233522

------------------------------------------------------------------------------------
| Id | Operation         | Name | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
------------------------------------------------------------------------------------
|  1 |  SORT AGGREGATE   |      |      1 |      1 |      1 |00:00:00.01 |     147 |
|* 2 |   TABLE ACCESS FULL| T    |      1 |     10 |     52 |00:00:00.01 |     147 |
------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter(MOD("N",19)=0)
```

