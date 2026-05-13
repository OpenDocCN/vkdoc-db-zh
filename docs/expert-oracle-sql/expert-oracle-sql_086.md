# Oracle 执行计划与索引技术

## 索引连接

```
|   0 | SELECT STATEMENT             |                   |
|   1 |  NESTED LOOPS                |                   |
|   2 |   NESTED LOOPS               |                   |
|*  3 |    VIEW                      | index$_join$_001  |
|*  4 |     HASH JOIN                |                   |
|*  5 |      HASH JOIN               |                   |
|*  6 |       INDEX RANGE SCAN       | EMP_DEPARTMENT_IX |
|*  7 |       INDEX RANGE SCAN       | EMP_NAME_IX       |
|   8 |      INDEX FAST FULL SCAN    | EMP_MANAGER_IX    |
|*  9 |    INDEX UNIQUE SCAN         | EMP_EMP_ID_PK     |
|* 10 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEES         |
```

由于姓氏为 Grant 的员工只有两位，并且其中只有一位的经理是 Mourgos，因此对于这个特定查询来说，出于性能考虑，索引连接并不合适，但我们可以使用提示来演示。

请注意，对`M.LAST_NAME`的引用并不妨碍索引连接，因为它并非来自表别名为`E`的`EMPLOYEES`表副本。

*   索引连接要求被连接的索引中引用的列是唯一被引用的。你甚至不能引用 ROWID！
*   索引连接只可能在一组连接表中的第一个表上进行（因此使用了上面的`LEADING`提示）。
*   当合法时，可以使用`INDEX_JOIN`提示强制使用索引连接，如上面的清单 10-16 所示。

如果你确实看到索引连接频繁发生，你可能需要考虑创建一个多列索引，因为这将比索引连接稍微高效一些，但根据我的经验，索引连接通常不是严重性能问题的根源。

你可能认为，即使你想引用主表中的行，索引连接有时也可能是合适的。嗯，确实有办法做到这一点。

## AND_EQUAL

清单 10-17 展示了`AND_EQUAL`访问方法，但优化器除非被提示，否则不再使用它，并且该提示已弃用。

清单 10-17. 已弃用的 AND_EQUAL 访问方法

```
SELECT /*+ and_equal(e (manager_id) (job_id)) */
      employee_id
      ,first_name
      ,last_name
      ,email
  FROM hr.employees e
 WHERE e.manager_id = 124 AND e.job_id = 'SH_CLERK';

| Id  | Operation                   | Name           |

|   0 | SELECT STATEMENT            |                |
|*  1 |  TABLE ACCESS BY INDEX ROWID| EMPLOYEES      |
|   2 |   AND-EQUAL                 |                |
|*  3 |    INDEX RANGE SCAN         | EMP_MANAGER_IX |
|*  4 |    INDEX RANGE SCAN         | EMP_JOB_IX     |
```

清单 10-17 展示了如前所述合并两组 ROWID 的过程（但可能使用了不同的机制），然后我们使用 ROWID 来访问表并获取额外的列。

*   这是一个已弃用的特性。
*   存在诸多限制，其中之一是索引必须是单列的、非唯一索引。
*   它需要`AND_EQUAL`提示。除非被提示，否则优化器永远不会生成此计划。
*   这种机制很少能带来性能提升，但如果确实有提升，通常也需要一个多列索引。
*   无论是标准版用户还是企业版用户，都有替代方法。我们将在第 19 章讨论一个适用于标准版用户的选项。下一节关于位图索引的内容将展示几种供企业版用户组合索引的方法。

## 位图索引访问

位图索引的物理结构实际上与 B 树索引非常相似，其根块和分支块结构完全相同。

区别在于叶块。假设我们在`T1`表的`C1`列上有一个 B 树索引，并且`T1`中有 100 行的`C1`值为'X'。B 树索引将包含 100 个索引条目，每个条目都有表中特定匹配行的 ROWID。



## 位图索引结构与压缩

位图索引会为值'X'生成一个位图，该位图为表或分区中的每一行对应一个比特位；对于表中`C1`列值为'X'的 100 行，对应的比特位会被置位，其余行的比特位则保持清零状态。

叶块中的这些位图会经过压缩以清除长段的清零位，因此表中数据越聚集，索引体积就越小。这种压缩通常意味着，即使每个值平均只对应两行数据，位图索引也会比对应的 B 树索引更小。

## 位图索引的限制与访问

需要始终记住，位图是受限制 ROWID 集合的表示。因此，不可能：
*   创建全局位图索引；
*   将全局索引中的 ROWID 转换为位图；或
*   将类型为 ROWID 或 UROWID 的变量或表达式转换为位图。

访问位图索引的操作与 B 树索引的操作相似（参见表 10-3）。

表 10-3。位图索引访问操作

| 执行计划操作 | 提示 | 替代提示 |
| --- | --- | --- |
| `BITMAP INDEX SINGLE VALUE` | `INDEX` | `BITMAP_TREE` / `INDEX_RS_ASC` |
| `BITMAP INDEX RANGE SCAN` | `INDEX` | `BITMAP_TREE` /`INDEX_RS_ASC` |
| `BITMAP INDEX FULL SCAN` | `INDEX` | `BITMAP_TREE` |
| `BITMAP INDEX FAST FULL SCAN` | `INDEX_FFS` |  |
| `BITMAP CONSTRUCTION` | N/A |  |
| `BITMAP COMPACTION` | N/A |  |

查看表 10-3 可以注意到以下几点：
*   没有降序或跳跃扫描的变体。
*   没有可用的`MIN`/`MAX`优化。
*   没有`UNIQUE`选项——这并不奇怪，因为唯一位图索引这种愚蠢的想法甚至不被支持。

`BITMAP INDEX SINGLE VALUE`操作用于为索引中的列提供了相等谓词的情况，意味着处理单个位图。

提供`BITMAP CONSTRUCTION`和`BITMAP COMPACTION`是为了完整性。如果你对`CREATE BITMAP INDEX`或`ALTER INDEX...REBUILD`命令运行`EXPLAIN PLAN`，你会看到这些操作出现。它们是 DDL 特定的操作。

## 位图操作概览

除了在索引中访问位图，我们还可以对位图做更多事情。表 10-4 展示了完整的操作集。

表 10-4。位图操作

| 执行计划操作 | 提示 | 替代提示 |
| --- | --- | --- |
| `BITMAP CONVERSION FROM ROWIDS` |  |  |
| `BITMAP CONVERSION TO ROWIDS` |  |  |
| `BITMAP AND` | `INDEX_COMBINE` | `BITMAP_TREE` |
| `BITMAP OR` | `INDEX_COMBINE` | `BITMAP_TREE` |
| `BITMAP MINUS` | `INDEX_COMBINE` | `BITMAP_TREE` |
| `BITMAP MERGE` | `INDEX_COMBINE` | `BITMAP_TREE` |
| `BITMAP CONVERSION COUNT` |  |  |
| `BITMAP KEY ITERATION` | `STAR_TRANSFORMATION` |  |

操作`BITMAP CONVERSION FROM ROWIDS`用于将来自 B 树索引的受限 ROWID 转换为位图，而操作`BITMAP CONVERSION TO ROWIDS`用于将位图转换回受限 ROWID。

`BITMAP AND`、`BITMAP OR`和`BITMAP MINUS`操作按照预期的方式执行布尔代数运算；可以形成复杂的位图操作树。`BITMAP MERGE`操作类似于`BITMAP OR`，但用于 ROWID 集合已知不相交的某些上下文。

通常，在完成所有这些位图操作后，最终结果会传递给`BITMAP CONVERSION TO ROWIDS`操作。然而，有时我们只想要行数的计数。这可以通过`BITMAP CONVERSION COUNT`操作完成。

`BITMAP KEY ITERATION`操作用于星型转换，我们将在第 13 章中讨论。

在一个查询中看到大量位图操作并不少见，如清单 10-18 所示。

清单 10-18。位图操作示例

```
CREATE INDEX t1_i3
   ON t1 (c5);

CREATE BITMAP INDEX t1_bix1
   ON t1 (c3);

SELECT /*+ index_combine(t)no_expand */
         *
        FROM t1 t
       WHERE (c1 > 0 AND c5 = 1 AND c3 > 0) OR c4 > 0;

| Id  | Operation                         | Name    |

|   0 | SELECT STATEMENT                  |         |
|   1 |  TABLE ACCESS BY INDEX ROWID      | T1      |
|   2 |   BITMAP CONVERSION TO ROWIDS     |         |
|   3 |    BITMAP OR                      |         |
|   4 |     BITMAP MERGE                  |         |
|   5 |      BITMAP INDEX RANGE SCAN      | T1_BIX2 |
|   6 |     BITMAP AND                    |         |
|   7 |      BITMAP CONVERSION FROM ROWIDS|         |
|   8 |       INDEX RANGE SCAN            | T1_I3   |
|   9 |      BITMAP MERGE                 |         |
|  10 |       BITMAP INDEX RANGE SCAN     | T1_BIX1 |
|  11 |      BITMAP CONVERSION FROM ROWIDS|         |
|  12 |       SORT ORDER BY               |         |
|  13 |        INDEX RANGE SCAN           | T1_I1   |

```

清单 10-18 开始时创建了另外两个索引。该查询的执行计划看起来有点吓人，但阅读此类执行计划时首先要明白的是不要惊慌。如果你一步一步地看，就能轻松理解它们。

让我们从操作 5 开始，这是第一个没有子节点的操作。它在`T1_BIX2`上执行`BITMAP INDEX RANGE SCAN`，该索引基于列`C4`。此范围扫描将可能处理与`C4`不同值关联的多个位图，这些位图由操作 4 合并。这个合并后的位图代表了第 3 行`BITMAP OR`的第一个操作数。然后我们继续查看`BITMAP OR`的第二个操作数，它由第 6 行的操作及其子节点定义。

现在我们看操作 8，这是第二个没有子节点的操作。这是在`T1_I3`上的`INDEX RANGE SCAN`，`T1_I3`是基于`C5`的 B 树索引。这里我们有一个相等谓词`C5 = 1`，这意味着返回的所有行具有相同的索引键值。一个鲜为人知的事实是，具有相同键值的非唯一索引条目会按 ROWID 排序。这意味着`INDEX RANGE SCAN`返回的所有行都将按 ROWID 顺序排列，并且可以通过操作 7 直接转换为位图。这个`BITMAP MERGE`构成了第 6 行`BITMAP AND`操作的第一个操作数。

我们继续看第 10 行，在`T1_BIX1`上的`INDEX RANGE SCAN`，`T1_BIX1`是基于`C3`的位图索引。同样，`C3`可能有多个值，因此各个位图需要由第 9 行合并，最终形成操作 6（即`BITMAP AND`）的第二个操作数。

但是操作 6 还有第三个操作数。我们通过查看第 13 行开始评估这第三个操作数，这是在`T1_I1`上的`INDEX RANGE SCAN`，`T1_I1`是基于`C1`和`C2`的 B 树索引。由于范围扫描可能覆盖多个键值，索引条目可能不会按 ROWID 顺序返回；我们需要在第 12 行对 ROWID 进行排序，然后在第 11 行将它们转换为位图。

现在我们有了第 6 行`BITMAP AND`操作的三个操作数和第 3 行`BITMAP OR`操作的两个操作数。在完成所有位图操作后，最终的位图在第 2 行转换为一组 ROWID，这些 ROWID 最终在第 1 行用于访问表。

清单 10-19 对清单 10-18 稍作修改，返回行数计数而不是内容；我们可以看到，将位图转换为 ROWID 的操作可以被消除。

清单 10-19。`BITMAP CONVERSION COUNT`示例

```
SELECT /*+ index_combine(t) */
            COUNT (*)
        FROM t1 t
       WHERE (c1 > 0 AND c5 = 1 AND c3 > 0) OR c4 > 0;

| Id  | Operation                         | Name    |
```



