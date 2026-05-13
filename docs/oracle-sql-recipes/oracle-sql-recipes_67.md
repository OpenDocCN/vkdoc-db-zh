# 表 19-1. AUTOTRACE 选项（续）

## 选项(s) 结果

`SET AUTOTRACE TRACE EXPLAIN`

仅解释执行计划，不生成结果集。

`SET AUTOTRACE TRACE STAT`

仅显示统计信息，结果集已生成但不显示。

##### 19-8. 使用 DBMS_XPLAN 生成执行计划

### 问题
你需要一种方便的方法来查看执行计划。

### 解决方案
查看统计信息的一个有用方法是使用 `DBMS_XPLAN` 包。在使用此技术之前，请确保 `PLAN_TABLE` 可用：

```sql
SQL> desc plan_table
```

如果 `PLAN_TABLE` 不存在，你需要创建一个。运行此查询以在你的模式中创建 `PLAN_TABLE`：

```sql
SQL> @?/rdbms/admin/utlxplan
```

现在，为你要解释的 SQL 语句在 `PLAN_TABLE` 中创建一条记录。使用 `EXPLAIN PLAN FOR` 语句来完成此操作：

```sql
explain plan for select emp_id from emp;
```

接下来，通过 `SELECT` 语句调用 `DBMS_XPLAN` 包来显示解释计划：

```sql
select * from table(dbms_xplan.display);
```

以下是输出的部分列表：

```
Plan hash value: 610802841

| Id | Operation | Name | Rows | Bytes | Cost (%CPU)| Time |
| 0 | SELECT STATEMENT | | 3 | 27 | 3 (0)| 00:00:01|
| 1 | PARTITION RANGE ALL| | 3 | 27 | 3 (0)| 00:00:01|
| 2 | TABLE ACCESS FULL | EMP | 3 | 27 | 3 (0)| 00:00:01|
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 19 章 ■ SQL 监控与调优

为便于使用，你可以创建一个视图来查询 `DBMS_XPLAN.DISPLAY` 函数：

```sql
create view pt as select * from table(dbms_xplan.display);
```

现在你可以从 `PT` 视图中进行选择：

```sql
select * from pt;
```

### 工作原理
`DBMS_XPLAN` 是一个有用的工具，可以以几种不同的格式显示解释计划。`DBMS_XPLAN` 包中包含许多有用的函数（详见表 19-2）。例如，如果某个特定查询出现在你的 AWR 报告的高 CPU 消耗 SQL 语句部分，你可以通过 `DISPLAY_AWR` 函数为该查询运行解释计划：

```sql
select * from table(dbms_xplan.display_awr('413xuwws268a3'));
```

