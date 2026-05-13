# 完成价格列表

我们已经开始使用连接表来创建价格列表：

```sql
SELECT
p.id,
p.title,
a.givenname||' '||a.familyname AS artist,
a.nationality,
p.price, p.price*0.1 AS tax, p.price*1.1 AS total
FROM paintings AS p JOIN artists AS a ON p.artistid=a.id;
```

与之前使用子查询的版本相比，这个版本既有我们想要的东西不足，又有我们不需要的东西过多。

首先，是缺失画作的问题。记住，有些画作没有 `artistid`，而之前的内连接会遗漏它们。一个合适的外连接应该可以解决这个问题。

其次，有些画作没有价格。我们无需决定*为什么*没有价格：也许它们是尚未定价的新画，或者是不再出售的旧画。关键是，把它们包含在价格列表中毫无意义。为此，我们可以过滤掉 `NULL` 值。

我们现在可以做的是删除旧版的价格列表，并用连接版本替换它：

```sql
DROP VIEW IF EXISTS pricelist;
CREATE VIEW pricelist AS
SELECT
p.id,
p.title,
a.givenname||' '||a.familyname AS artist,
a.nationality,
p.price, p.price*0.1 AS tax, p.price*1.1 AS total
FROM paintings AS p LEFT JOIN artists AS a ON p.artistid=a.id
WHERE p.price IS NULL;
```

对于各种数据库管理系统（DBMS），请记住：

*   Oracle 不支持 `IF EXISTS`。
*   Microsoft 要求在 `CREATE VIEW` 语句块前后加上 `GO`。
*   MySQL/MariaDB 需要设置为 ANSI 模式才能使用字符串连接；否则，你需要使用 `concat()` 函数。Microsoft 需要使用 `+` 进行连接。

