# 使用 T-SQL 的典型方法

使用 T-SQL 的典型方法是利用表的自连接，如下所示：

```sql
SELECT
    T1.x,
    SUM(T2.x) AS running_x
FROM
    T AS T1 INNER JOIN T AS T2
    ON T1.x >= T2.x
GROUP BY
    T1.x;
```

