# 使用格式化的日期按工作日分组

你也可以使用相同的技术来提取工作日名称。例如：

```sql
--  PostgreSQL, Oracle
SELECT
    id, ordered,
    to_char(ordered,'Dy') AS weekday    --    或 'Day'
FROM sales;
--  MariaDB / MySQL
SELECT
    id, ordered,
    date_format(ordered,'%a') AS weekday    --  或 '%W'
FROM sales;
--  MSSQL
SELECT
    id, ordered,
    format(ordered,'ddd') AS weekday        --  或 'dddd'
FROM sales;
```

这会给出如下结果：

| id  | 订购时间                     | weekday |
| --- | ---------------------------- | ------- |
| 52  | 2022-03-07 16:10:45.739071   | Mon     |
| 54  | 2022-03-08 00:23:39.53316    | Tue     |
| 55  | 2022-03-08 06:23:28.387395   | Tue     |
| 57  | 2022-03-09 00:02:29.974004   | Wed     |
| 59  | 2022-03-09 06:26:24.808237   | Wed     |
| 60  | 2022-03-09 15:01:05.592177   | Wed     |
|  ~ 2509 行 |                              |         |

SQLite 没有简单的方法来生成星期名称。无论如何，你总是可以按工作日数字进行分组。

请记住，如果你需要按工作日排序，你将需要使用第 4 章中提到的特殊非字母排序技术。

