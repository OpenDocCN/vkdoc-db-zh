# UPDATE 操作

当执行 SQL 语句 `UPDATE` 时会使用此操作。其特定特点是它支持可变数量的子操作。大多数情况下，它只有一个子操作，因此被视为独立操作。仅当在 `SET` 子句中使用了子查询时，才会有两个或更多子操作可用。如果它有多个子操作，其行为与 `FILTER` 操作相同。换句话说，第一个子操作驱动其他子操作的执行。

以下是一个示例 SQL 语句及其执行计划（其父子关系的图形化表示见 图 6-7）：

```sql
UPDATE emp e1
SET sal = (SELECT avg(sal) FROM emp e2 WHERE e2.deptno = e1.deptno),
    comm = (SELECT avg(comm) FROM emp e3)
```



