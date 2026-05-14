# 自动刷新调度

如果你想自动化按需刷新，可以在 `CREATE MATERIALIZED VIEW` 和 `ALTER MATERIALIZED VIEW` 中指定首次刷新的时间（`START WITH` 子句）以及计算后续刷新时间的表达式（`NEXT` 子句）。例如，在以下 SQL 语句中，从 SQL 语句执行之时起，每十分钟安排一次刷新：

```sql
ALTER MATERIALIZED VIEW sales_mv REFRESH COMPLETE ON DEMAND
START WITH sysdate NEXT sysdate+to_dsinterval('0 00:10:00')
```

为了调度刷新，会提交一个基于包 `dbms_job` 的作业。请注意，这里使用了包 `dbms_refresh` 而不是包 `dbms_mview`。由于包 `dbms_refresh` 也受到错误 3168840 的影响，此作业在 Oracle9*i* 中可能无效，你可能被迫调度自己的作业来绕过此错误。

```sql
SELECT what, interval
  FROM user_jobs;
```

```
WHAT                                     INTERVAL
---------------------------------------- -----------------------------------
dbms_refresh.refresh('"SH"."SALES_MV"'); sysdate+to_dsinterval('0 00:10:00')
```

