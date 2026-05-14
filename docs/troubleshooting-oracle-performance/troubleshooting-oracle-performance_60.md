# 刷新

当表被修改时，其依赖的物化视图会变为过时状态。因此，需要执行刷新以使物化视图数据保持最新。创建物化视图时，你可以指定刷新的方式和时机。

## 刷新方法

要指定数据库引擎执行刷新的方式，你可以从以下方法中选择：

*   `REFRESH COMPLETE`：容器表的全部内容被删除，所有数据从基表重新加载。显然，此方法始终受支持。仅当基表有相当一部分数据被修改时，才应使用此方法。
*   `REFRESH FAST`：重用容器表的内容，仅将修改传播到容器表。如果基表中修改的数据很少，则应使用此方法。此方法仅在满足若干要求时才可用。如果其中一项要求未满足，则 `REFRESH FAST` 会被拒绝作为物化视图的有效参数，或者会引发错误。快速刷新将在后续章节中详细介绍。
*   `REFRESH FORCE`：首先尝试快速刷新。如果无法工作，则执行完全刷新。这是默认方法。
*   `NEVER REFRESH`：物化视图从不刷新。如果尝试刷新，则会以错误 `ORA-23538: cannot explicitly refresh a NEVER REFRESH materialized view` 终止。你应使用此方法来确保永远不会执行刷新。

## 刷新时机

你可以通过两种不同的方式选择物化视图刷新发生的时间点：

*   `ON DEMAND`：当明确请求时（手动或通过定期运行作业）刷新物化视图。这意味着从基表修改到物化视图刷新之间的这段时间内，物化视图可能包含过时数据。
*   `ON COMMIT`：在修改基表的同一事务中自动刷新物化视图。换句话说，就其他会话而言，物化视图始终包含最新数据。

你可以组合这些选项来指定物化视图的刷新方式和时机，并可在 `CREATE MATERIALIZED VIEW` 和 `ALTER MATERIALIZED VIEW` 语句中使用它们。以下是一个示例：

```sql
ALTER MATERIALIZED VIEW sales_mv REFRESH FORCE ON DEMAND
```

甚至可以使用 `REFRESH COMPLETE ON COMMIT` 选项创建物化视图。然而，这种配置在实践中不太可能有用。

要显示与物化视图关联的参数、它是否最新，以及最后一次刷新的方式和时间，可以查询视图 `user_mviews`：

```sql
SELECT refresh_method, refresh_mode, staleness,
       last_refresh_type, last_refresh_date
  FROM user_mviews
 WHERE mview_name = 'SALES_MV';
```

```
REFRESH_METHOD REFRESH_MODE STALENESS LAST_REFRESH_TYPE LAST_REFRESH_DATE
-------------- ------------ --------- ----------------- -------------------
FORCE          DEMAND       FRESH     COMPLETE          02.04.2008 20:38:31
```

