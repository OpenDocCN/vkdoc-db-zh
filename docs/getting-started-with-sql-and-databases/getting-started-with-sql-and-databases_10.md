# 字符串

最基本的字符串筛选是查找精确匹配：

```sql
SELECT *
FROM customers
WHERE state='VIC';
```

这给出了位于 VIC 的客户：

| id | familyname | givenname | … | state | … | registered |
| --- | --- | --- | --- | --- | --- | --- |
| 186 | Gunn | Ray | … | VIC | … | 2021-11-15 |
| 523 | Sights | Seymour | … | VIC | … | 2022-07-11 |
| 545 | Knife | Jack | … | VIC | … | 2022-07-24 |
| 505 | Singers | Carol | … | VIC | … | 2022-06-29 |
| 492 | Long | Miles | … | VIC | … | 2022-06-21 |
| 374 | Sharalike | Sharon | … | VIC | … | 2022-03-29 |
| ~ 52 行 ~ |

要获取居住在其他地方的客户：

```sql
SELECT *
FROM customers
WHERE state<>'VIC'; --  WHERE NOT state='VIC'
```

这应该会给你所有其他州的客户：

| id | familyname | givenname | … | state | … | registered |
| --- | --- | --- | --- | --- | --- | --- |
| 474 | Free | Judy | … | NSW | … | 2022-06-12 |
| 144 | King | Ray | … | NSW | … | 2021-10-18 |
| 179 | Inkling | Ivan | … | WA | … | 2021-11-08 |
| 475 | Blood | Drew | … | QLD | … | 2022-06-13 |
| 341 | Idate | Val | … | NSW | … | 2022-03-03 |
| 351 | Tate | Dick | … | NSW | … | 2022-03-13 |
| ~ 217 行 ~ |

与所有筛选器一样，筛选不存在的值并不是错误：

```sql
SELECT *
FROM customers
WHERE state='XYZ';
```

但是，如果数据类型不匹配，这*可能*是错误：

```sql
SELECT *
FROM customers
WHERE state=23;
```

在某些数据库管理系统（如 PostgreSQL 和 Oracle）中，这将导致错误，因为你无法将字符串与数字匹配。要使比较生效，你需要将数字表示为字符串：`WHERE state='23'`。在其他一些系统（如 SQLite、MySQL/MariaDB 和 MSSQL）中，为了比较目的，数字会被隐式转换为字符串。


#### 引号

在 SQL 中，字符串使用*单引号*括起来。双引号具有完全不同的含义。

对此有一些罕见的例外：

*   Microsoft Access 允许你使用双引号作为单引号的替代；实际上，它似乎更偏好双引号，但你不应该这样做。
*   MySQL/MariaDB 也允许你使用双引号，但这取决于其运行模式；在 ANSI 模式下，双引号不能用于字符串。

无论如何，单引号总是适用的。

如果你使用双引号而不是单引号会发生什么？

```sql
SELECT *
FROM customers
WHERE state="VIC";      --  在 MySQL/MariaDB 中可能没问题
```

对于大多数数据库管理系统，你会收到错误信息，指出列 `VIC` 未知。也就是说，双引号被解释为包围列名，而不是字符串。SQL 认为你试图将 `state` 列与未知的 `VIC` 列进行匹配。

有时这可能会导致混淆：

```sql
SELECT * FROM customers WHERE familyname='Town';
SELECT * FROM customers WHERE familyname="Town";
```

前面的第一个示例查找 `familyname` 匹配 `Town` 的客户，而第二个示例则查找姓氏恰好与他们居住城镇名相同的客户。

如果你不在 ANSI 模式下运行 MySQL/MariaDB，双引号将被解释为字符串，并且你会得到一个成功的结果。请参阅下面关于在 MySQL/MariaDB 中运行 ANSI 模式的部分。

你可以在任何列周围使用双引号：

```sql
SELECT *
FROM customers
WHERE "state"='VIC';
```

这里的双引号根本不会产生任何影响，因为 SQL 已经知道 `state` 是列名。

关于计算列值的第 5 章有更多关于使用双引号的信息。

#### 关于 MySQL/MariaDB 模式的更多信息

如果你使用 MySQL 或 MariaDB，我们建议你始终将会话设置为 ANSI 模式。你可以一开始就这样做：

```sql
SET SESSION sql_mode = 'ANSI';
```

此语句只需要在会话开始时运行一次。就我们的目的而言，最重要的差异将体现在：

*   双引号的使用
*   字符串连接（拼接字符串），你稍后在处理计算时会看到。

如果你不使用 ANSI 模式，你仍然可以完成大多数操作，但某些语法可能需要调整。

本书将假定你已将会话设置为 ANSI 模式。

#### 关于双引号和单引号的更多内容

有一个名为 `badtable` 的表，其中可以看到各种有问题的名称：

```sql
SELECT * FROM badtable;
```

你将看到以下结果：

| customer code | customer | order | 1st | 42 | last-date |
| --- | --- | --- | --- | --- | --- |
| 23 | Fred | 42 | 2020-01-01 | Life, … | 2020-01-31 |
| 37 | Wilma | 54 | 2020-02-01 | I think …. | 2020-02-29 |

到目前为止，一切正常。但是，如果你尝试单独选择这些列，你会得到各种错误：

```sql
SELECT
customer code,  --  customer AS code
customer,
order,          --  ORDER BY
1st,            --  number 1 AS st
42,             --  number 42
last-date       --  last -  date
FROM badtable;
```

只有 `customer` 列是正确的。有些会导致错误，有些会被误解。

引用这些有问题列名的唯一方法是使用双引号将其括起来：

| code | customer | order | st | ?column? | last-date |
| --- | --- | --- | --- | --- | --- |
| Fred | Fred | 42 | 1 | 42 | 2020-01-31 |
| Wilma | Wilma | 54 | 1 | 42 | 2020-02-29 |

```sql
SELECT
customer code,  --  customer AS code
customer,
"order",
1st,            --  number 1 AS st
42,             --  number 42
"last-date"
FROM badtable;
```

两个被误解的列涉及别名。你稍后会看到更多关于别名的内容，但你会看到歧义是由于 `AS` 这个词是可选的这一事实造成的。

另一个被误解的列是 `42` 被解释为一个值，这是合法的，而不是列名。为了完成这项工作，你也需要给这些名称加上引号：

```sql
SELECT
"customer code",
customer,
"order",
"1st",
"42",
"last-date"
FROM badtable
```

关于列名有一些简单的规则，而我们似乎费了很大力气去违反它们：

*   名称不应包含空格或其他特殊字符，如连字符。如果你需要分隔符，通常应使用下划线 (`_`)。
*   名称不应以数字开头，并且当然不应 `be` 一个数字。
*   名称应避免使用 SQL 关键字，如 `order`。某些数据库管理系统可能会让你感到惊讶。例如，PostgreSQL 将“name”视为关键字。

在正常情况下，你的查询不应该需要双引号，因为一个好的 SQL 开发人员应该知道如何避免这些问题。然而，你不能总是为另一位开发人员所做的工作负责，所以有时你可能需要双引号。

#### 大小写敏感性

字符串的比较方式并不总是一样的。例如，`customers` 表中的所有州名都是大写的。如果你尝试匹配小写：

```sql
SELECT *
FROM customers
WHERE state='vic';
```

你的结果会有所不同。

在 PostgreSQL、Oracle 和 SQLite 中，默认情况下，你不会得到任何匹配项。然而，对于 MySQL/MariaDB 和 MSSQL，你会像之前一样得到匹配项。

字符串变体如何比较被称为**排序规则**。在其他语言中，可能存在许多变体，但在英语中，主要的变体是大写/小写。

PostgreSQL、Oracle 和 SQLite 的默认排序规则是大小写敏感的；也就是说，大写和小写被视为不同。在 MySQL/MariaDB 和 MSSQL 中，默认排序规则是大小写不敏感的。*但是，个别数据库可能已使用替代排序规则设置。*

表 `sorting` 在其 `stringvalue` 列中使用了不一致的大小写，因此你可以尝试：

```sql
SELECT *
FROM sorting
WHERE stringvalue='APPLE';
```

同样，结果的数量将取决于排序规则。

在任何情况下，你的表或数据库的排序规则可能不是默认的。如果你不确定排序规则，可以运行这个简单的查询：

```sql
SELECT *
FROM customers
WHERE 'a'='A';
```

如果你的排序规则是大小写敏感的，那么断言 `'a'='A'` 为假，因此你将不会得到任何行。如果是大小写不敏感的，那么断言为真，因此你将得到所有行。

如果你的排序规则是大小写敏感的，但你仍然希望进行大小写不敏感的匹配，则有两种解决方案。

首先，你可以强制将数据转换为大写或小写，然后测试结果。例如：

```sql
SELECT *
FROM customers
WHERE lower(state)='vic';
```

这是一个稍微昂贵的解决方案，因为数据库管理系统需要在每一行上执行一个操作才能进行比较。

第二个解决方案是要求数据库管理系统为查询使用替代排序规则。但是，这可能非常复杂。

如果你的数据库排序规则是大小写敏感的，并且你发现需要进行许多大小写不敏感的搜索，你也许可以使用**索引**来减少工作量。数据库索引就像书中的索引，可以帮助数据库管理系统更快地查找内容。索引将在后面讨论。



