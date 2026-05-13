# MariaDB/MySQL 中的日期格式化

对于 MariaDB/MySQL，有 `date_format()` 函数：

```sql
WITH vars AS (
SELECT timestamp '1969-07-20 20:17:40' AS moonshot
)
SELECT
moonshot,
date_format(moonshot,'%W, %D %M %Y') AS fulldate,
date_format(moonshot,'%a %d %b %Y') AS shortdate
FROM vars;
```

你可以通过以下链接了解更多关于格式代码的信息：

*   MariaDB： [`mariadb.com/kb/en/date_format/`](https://mariadb.com/kb/en/date_format/)
*   MySQL： [`dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html`](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html)

