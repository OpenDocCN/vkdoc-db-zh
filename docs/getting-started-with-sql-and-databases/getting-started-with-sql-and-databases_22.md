# 使用格式化的日期按月分组

有时，你需要获取一个细粒度的日期/时间，并按更大的组进行分析。一个*非常*有用的日期格式是类似 `yyyy-mm` 这样的截断形式。

例如：

```sql
--  PostgreSQL, Oracle
SELECT id, ordered, to_char(ordered,'YYYY-MM') AS month
FROM sales;
--  MariaDB / MySQL
SELECT id, ordered, date_format(ordered,'%Y-%m') AS month
FROM sales;
--  MSSQL
SELECT id, ordered, format(ordered,'yyyy-MM') AS month
FROM sales;
--  SQLite
SELECT id, ordered, strftime('%Y-%m',ordered) AS month
FROM sales;
```

结果看起来像这样：

| id  | 订购时间                     | month   |
| --- | ---------------------------- | ------- |
| 52  | 2022-03-07 16:10:45.739071   | 2022-03 |
| 54  | 2022-03-08 00:23:39.53316    | 2022-03 |
| 55  | 2022-03-08 06:23:28.387395   | 2022-03 |
| 57  | 2022-03-09 00:02:29.974004   | 2022-03 |
| 59  | 2022-03-09 06:26:24.808237   | 2022-03 |
| 60  | 2022-03-09 15:01:05.592177   | 2022-03 |
|  ~ 2509 行 |                              |         |

这种格式的日期字符串将能够正确排序，因为结果长度相同且以较大的部分开头。

你将在后面学习如何对数据进行分组。

