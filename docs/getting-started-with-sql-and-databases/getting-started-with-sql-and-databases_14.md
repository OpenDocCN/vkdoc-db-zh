# 按计算列排序

`ORDER BY`子句后跟一个或多个要排序的列。然而，这些列不一定是原始表列。你可以按任何计算值排序。例如：

```sql
SELECT id, givenname, familyname
FROM artists
ORDER BY died - born;
```

这会给出一个结果，但排序顺序不明确：

| id | givenname | familyname |
| --- | --- | --- |
| 348 | Honoré | Daumier |
| 370 | Johannes | Vermeer |
| 233 | Egon | Schiele |
| 10 | Frédéric | Bazille |
| 112 | Dirck van | Baburen |
| 296 | William | Winstanley |
| ~ 187 行 ~ |

通常，你可能希望在`SELECT`子句中列出计算项，这样你可以在那里计算它并按结果排序：

```sql
SELECT id, givenname, familyname, died - born
FROM artists
ORDER BY died - born;
```

现在排序顺序更清晰了。

| id | givenname | familyname | ?column? |
| --- | --- | --- | --- |
| 348 | Honoré | Daumier | 0 |
| 370 | Johannes | Vermeer | 0 |
| 233 | Egon | Schiele | 28 |
| 10 | Frédéric | Bazille | 29 |
| 112 | Dirck van | Baburen | 29 |
| 296 | William | Winstanley | 31 |
| ~ 187 行 ~ |

请记住，如果你的数据库管理系统(DBMS)将`NULL`排在前面，那么你会在实际值之前看到它们。另外，请记住我们发现了一些`born`和`died`值相同的艺术家，这解释了零值的来源。

当然，每个计算列都应该有一个别名：

```sql
SELECT id, givenname, familyname, died - born AS age
FROM artists
ORDER BY died - born;
```

这使结果更清晰：

| id | givenname | familyname | age |
| --- | --- | --- | --- |
| 348 | Honoré | Daumier | 0 |
| 370 | Johannes | Vermeer | 0 |
| 233 | Egon | Schiele | 28 |
| 10 | Frédéric | Bazille | 29 |
| 112 | Dirck van | Baburen | 29 |
| 296 | William | Winstanley | 31 |
| ~ 187 行 ~ |

由于`ORDER BY`子句是唯一在`SELECT`子句之后处理的子句，你实际上可以使用别名：

```sql
SELECT id, givenname, familyname, died - born as age
FROM artists
ORDER BY age;
```

其寓意在于，你可以按计算列排序，但你可能最终会在`SELECT`子句中计算它，然后按结果排序。

请注意，你*不能*在`WHERE`子句中这样做：

```sql
SELECT *, died-born AS age
FROM artists
WHERE died-born<50  --  不是 age<50 ∵ age（尚）不可用
--  SELECT
ORDER BY age;
```

我们包含了一个注释掉的`SELECT`子句来提醒你关于求值顺序：只有`ORDER BY`子句在`SELECT`之后求值，所以只有这个子句可以使用列别名。

你也可以对字符串函数做同样的事情：

```sql
--  PostgreSQL, MySQL/MariaDB, SQLite, Oracle
SELECT *, length(familyname) AS ln
FROM customers
ORDER BY ln;
-- MSSQL
SELECT *, len(familyname) AS ln
FROM customers
ORDER BY ln;
```

这会给你按名字长度排序的结果：

| id | givenname | familyname | … | ln |
| --- | --- | --- | --- | --- |
| 452 | Sue | Me | … | 2 |
| 383 | Rose | Up | … | 2 |
| 226 | Carrie | On | … | 2 |
| 467 | Luke | Up | … | 2 |
| 99 | Minnie | Bus | … | 3 |
| 312 | Frank | Lee | … | 3 |
| ~ 304 行 ~ |

你将在本书后面看到更多关于计算的内容。

### 限制结果数量

你已经看到了如何将结果限制为某些条件，例如某个州的客户或价格低于某个金额的画作。这里，我们看看简单地限制结果的`数量`。

最初，SQL 没有执行此操作的标准方式，大概是因为没有人看到这个需求。自那时起，`OFFSET … FETCH …`子句变得可用。然而，这仅适用于 PostgreSQL、Microsoft SQL Server 和 Oracle。对于其他 DBMS，下一节中介绍了一种非官方的替代方法。

例如，将结果限制为前五个：

```sql
--  PostgreSQL, MSSQL, Oracle
SELECT *
FROM customers
WHERE dob IS NOT NULL   --  排除缺失的出生日期
ORDER BY dob OFFSET 0 ROWS FETCH FIRST 5 ROWS ONLY;
```

添加`WHERE dob IS NOT NULL`子句是为了过滤掉缺失的出生日期。否则，你将只会在第一个或最后一个组中看到`NULL`出生日期：

| id | givenname | familyname | dob | … |
| --- | --- | --- | --- | --- |
| 344 | Rose | Boat | 1962-09-24 | … |
| 545 | Jack | Knife | 1962-09-24 | … |
| 416 | Pam | Pered | 1962-09-28 | … |
| 440 | Percy | Monn | 1962-12-12 | … |
| 261 | Vic | Tory | 1962-12-12 | … |
| ~ 5 行 ~ |   |   |   |   |

这个非常冗长的`OFFSET … FETCH`子句有两个重要部分：`OFFSET`部分实际上意味着跳过前若干行，而`FETCH FIRST`部分是你想要的最大行数。如果没有那么多行，你将获得尽可能多的可用行。

请注意以下几点：

*   `OFFSET … FETCH …`是`ORDER BY`子句的扩展。
*   `FIRST`可以替换为`NEXT`：效果完全相同。
*   `ROWS`可以写成`ROW`：效果也相同。

某些 DBMS 可能允许更多的灵活性。

另外，请注意`OFFSET … FETCH …`在结果数量上有些过于严格。例如，如果你使用`FETCH FIRST 5`，即使接下来的几行具有相同的值，你也永远不会得到超过 5 个。

## 分页

你可能想使用此功能的一个原因是分页结果，例如查看每页 20 个项目的目录：

```sql
--  第一页
SELECT * FROM paintings
ORDER BY title OFFSET 0 ROWS FETCH FIRST 20 ROWS ONLY;
--  第 4 页（跳过 3 页）
SELECT * FROM paintings
ORDER BY title OFFSET 3*20 ROWS FETCH FIRST 20 ROWS ONLY;
SELECT * FROM paintings
ORDER BY title OFFSET 60 ROWS FETCH FIRST 20 ROWS ONLY;
--  倒序：最后一页优先
SELECT * FROM paintings
ORDER BY title DESC
OFFSET 0 ROWS FETCH FIRST 20 ROWS ONLY;
```

如你所见，你也可以使用`DESC`来反转顺序。

### 使用 LIMIT … OFFSET … (MySQL/MariaDB, SQLite, 和 PostgreSQL)

MySQL/MariaDB 和 SQLite 目前不支持`OFFSET … FETCH …`子句。但是，它们支持一个简单得多的子句：

```sql
SELECT *
FROM customers
WHERE dob IS NOT NULL
ORDER BY dob LIMIT 5 OFFSET 0;
```

PostgreSQL 也支持此子句，因此 PostgreSQL 兼具灵活性和简单性的优势。

### 使用 TOP (MSSQL)

旧版本的 MSSQL 也不支持`LIMIT … OFFSET …`，但你可以使用类似这样的方法：

```sql
SELECT top 5 *
FROM customers
WHERE dob IS NOT NULL
ORDER BY dob;
```

这更简单，但不如`LIMIT … OFFSET …`灵活，因为你无法指定起点。要获取*最后的*行，你需要反转排序顺序：

```sql
SELECT top 5 *
FROM customers
WHERE dob IS NOT NULL
ORDER BY dob DESC;
```

MSSQL 实际上并不强制要求`TOP`子句与`ORDER BY`子句一起使用，但如果没有它，结果将毫无意义，因为否则你无法控制哪些行排在前面。

