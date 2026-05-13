# 使用 NULL 表示一个常见值

|   0 | SELECT STATEMENT                    |               |    10 |     2   (0)|
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| MOSTLY_BORING |    10 |     2   (0)|
|   2 |   INDEX RANGE SCAN                  | SPECIAL_INDEX |    10 |     1   (0)|

```
SELECT *
  FROM mostly_boring
 WHERE special_flag = 'N';
```

| Id  | Operation         | Name          | Rows  | Cost (%CPU)|
|   0 | SELECT STATEMENT  |               | 99990 |   384   (1)|
|   1 |  TABLE ACCESS FULL| MOSTLY_BORING | 99990 |   384   (1)|

我创建了一个名为 `MOSTLY_BORING` 的表，其中包含 100,000 行数据，正如你猜的那样，大部分是乏味的。然而，有十行的 `SPECIAL_FLAG` 被设置为 `Y`；其余的 99,990 行的 `SPECIAL_FLAG` 值为 `N`。在收集统计信息时，我为 `SPECIAL_FLAG` 指定了一个直方图，以便记录这两个值的偏斜情况。当我们使用谓词 `SPECIAL_FLAG != N` 查询该表时，我们使用了全表扫描。我们无法利用我们建立的索引。当我们使用谓词 `SPECIAL_FLAG = 'Y'` 时，可以看到使用了索引，并且预估成本节省显著。
当我们使用谓词 `SPECIAL_FLAG = 'N'` 请求所有乏味的行时，CBO 正确地选择了全表扫描。通过索引来读取表中几乎所有的行是没有意义的。这意味着乏味行 `(SPECIAL_FLAG = 'N')` 在索引中的条目将永远不会被使用。我们可以避免存储这些行，并使我们的索引更小。我们可以通过一种被一些数据库理论家视为异端的技术来实现这一点：我们可以使用值的缺失 `(NULL)` 来表示一个乏味的行。让我们以这种方式重新加载表数据。

Listing 16-4 重新加载数据，使用 `NULL` 作为“值”来表示 99,990 个乏味行，而不是 `N`。我们现在不再需要 `SPECIAL_FLAG` 上的直方图，因为现在只有一个非空值。选择乏味行的修改后查询的预估成本降低了。这是预期的，因为乏味行中移除了 `N` 值意味着表稍微变小了，全表扫描的成本也相应降低了。但是，对于选择特殊行的查询，我们看到了什么？预估成本从 2 增加到了 11！为什么？事实上，增加的预估成本是正确的，而 Listing 16-3 中的预估成本 2 太低了。为什么？嗯，`SPECIAL_INDEX` 原始聚簇因子的值表明大多数时候连续的索引条目会在同一个块中。虽然这对于大多数索引条目（那些 `SPECIAL_FLAG = 'N'` 的）是真的，但对于我们的十个特殊行来说并非如此，没有两个特殊行在同一个块中！事实上，由于数据变化，`SPECIAL_INDEX` 的 `BLEVEL` 已从 1 减少到 0，因此无论 CBO 如何估算，我们确实从这一更改中获得了少量的性能改进。

Listing 16-4. 使用 NULL 表示一个常见值

```
TRUNCATE TABLE mostly_boring;

INSERT INTO mostly_boring (primary_key_id, special_flag)
       SELECT ROWNUM, DECODE (MOD (ROWNUM, 10000), 0, 'Y', NULL)
         FROM DUAL
   CONNECT BY LEVEL <= 100000;

BEGIN
   DBMS_STATS.gather_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'MOSTLY_BORING');
END;
/

SELECT *
  FROM mostly_boring
 WHERE special_flag = 'Y';

| Id  | Operation                           | Name          | Cost (%CPU)|
|   0 | SELECT STATEMENT                    |               |    11   (0)|
|   1 |  TABLE ACCESS BY INDEX ROWID BATCHED| MOSTLY_BORING |    11   (0)|
|   2 |   INDEX RANGE SCAN                  | SPECIAL_INDEX |     1   (0)|

SELECT *
  FROM mostly_boring
 WHERE special_flag IS NULL;

| Id  | Operation         | Name          | Rows  | Cost (%CPU)|
|   0 | SELECT STATEMENT  |               | 99990 |   379   (1)|
|   1 |  TABLE ACCESS FULL| MOSTLY_BORING | 99990 |   379   (1)|
```

