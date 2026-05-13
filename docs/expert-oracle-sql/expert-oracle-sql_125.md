# 避免重复排序

有时排序是不可避免的，但请尽量不要进行不必要的排序。对相同的数据排序两次是没有意义的，但有时这种重复会发生。尽可能避免它。代码清单 17-9 展示了一个导致双重排序的查询。该查询看起来像是一个编写良好的 SQL 语句，直到我们查看其执行计划。

代码清单 17-9. 冗余排序

```sql
 SELECT cust_id
        ,time_id
        ,SUM (amount_sold) daily_amount
        ,SUM (
            SUM (amount_sold))
         OVER (PARTITION BY cust_id
               ORDER BY time_id
               RANGE BETWEEN INTERVAL '6' DAY PRECEDING AND CURRENT ROW)
            weekly_amount
    FROM sh.sales
GROUP BY cust_id, time_id
ORDER BY cust_id, time_id DESC;

| 标识  | 操作             | 名称  | 行数  | 成本 (%CPU)|
|-----|-----------------------|-------|-------|------------|
|   0 | SELECT STATEMENT      |       |   143K|  5275   (1)|
|   1 |  SORT ORDER BY        |       |   143K|  5275   (1)|
|   2 |   WINDOW SORT         |       |   143K|  5275   (1)|
|   3 |    PARTITION RANGE ALL|       |   143K|  5275   (1)|
|   4 |     SORT GROUP BY     |       |   143K|  5275   (1)|
|   5 |      TABLE ACCESS FULL| SALES |   918K|   517   (2)|
```

代码清单 17-9 聚合数据，以便将同一客户在同一天的多次销售相加。这本可以使用哈希聚合完成，但 CBO 决定使用排序。那就这样吧。然后我们在分析函数中使用窗口子句来计算截至当前日期当周内当前客户的总销售额。这需要第二次排序。但是，我们希望数据按客户和*降序*日期顺序呈现。这似乎需要第三次排序。通过对代码进行微小的改动，我们可以避免其中一次排序。代码清单 17-10 展示了方法。

代码清单 17-10. 避免冗余排序

```sql
 SELECT cust_id
        ,time_id
        ,SUM (amount_sold) daily_amount
        ,SUM (
            SUM (amount_sold))
         OVER (PARTITION BY cust_id
               ORDER BY time_id DESC
               RANGE BETWEEN CURRENT ROW AND INTERVAL '6' DAY FOLLOWING)
            weekly_amount
    FROM sh.sales
GROUP BY cust_id, time_id
ORDER BY cust_id, time_id DESC;

| 标识  | 操作            | 名称  | 行数  | 成本 (%CPU)|
|-----|----------------------|-------|-------|------------|
|   0 | SELECT STATEMENT     |       |   918K|  5745   (1)|
|   1 |  WINDOW SORT         |       |   918K|  5745   (1)|
|   2 |   PARTITION RANGE ALL|       |   918K|  5745   (1)|
|   3 |    SORT GROUP BY     |       |   918K|  5745   (1)|
|   4 |     TABLE ACCESS FULL| SALES |   918K|   517   (2)|
```

我们的分析函数现在以与 `ORDER BY` 子句相同的方式对数据进行排序，这样就消除了一次排序。请注意，在 代码清单 17-9 和 代码清单 17-10 的执行计划中，CBO 的成本估算都没有意义。然而，代码清单 17-10 效率更高。

