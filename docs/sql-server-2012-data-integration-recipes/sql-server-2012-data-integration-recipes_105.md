# 约束

要查看不允许进入 Oracle 数据库的数据，您可以使用以下 SQL 查看一个或多个表上的所有约束：

```sql
SELECT    ACC.TABLE_NAME, ACC.COLUMN_NAME, AC.SEARCH_CONDITION
FROM      ALL_CONS_COLUMNS ACC INNER JOIN ALL_CONSTRAINTS AC
          ON ACC.CONSTRAINT_NAME = AC.CONSTRAINT_NAME
WHERE     CONSTRAINT_TYPE = 'C'
```

当然，您可以扩展 `WHERE` 子句，将输出限制到您希望分析的表。

