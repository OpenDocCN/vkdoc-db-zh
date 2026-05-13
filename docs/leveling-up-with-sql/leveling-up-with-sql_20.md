# 9. 关于公共表表达式的更多内容

你已经利用 CTEs 为聚合和其他操作准备了数据。

在这里，我们将进一步探讨 CTEs 一些更强大的功能。



## 将 CTE 用作变量

在第 4 章中，我们使用一个测试值进行了一些计算：

```sql
WITH vars AS (
SELECT ' abcdefghijklmnop ' AS string
--  FROM dual   --  Oracle
)
SELECT
string,
--  示例字符串函数
FROM vars;
```

在本章后面，当我们探讨表字面量时，会看到这种技术更复杂的版本。现在，我们先看看如何使用它。

一些数据库管理系统以及所有编程语言都有`变量`的概念。变量是一个临时的、有名称的值。在数据库系统支持的情况下，你可以声明一个变量名并赋值，然后在后续步骤中使用。例如，在 MSSQL 中，你可以这样写：

```sql
--  MSSQL
DECLARE @taxrate decimal(4,2);
SET @taxrate = 12.5;
SELECT
id, title,
price, price/@taxrate/100 AS tax
FROM books;
```

要运行这段代码，你需要高亮选中所有语句并一次性执行。

本章不会重点讨论这些变量，关于变量的更多用法将在第 10 章中介绍。相反，我们将看看如何使用通用表表达式（CTE）来完成类似的工作。

严格来说，我们将要使用的不是变量，而是`常量`，这意味着我们只会设置一次它们的值。不过，我们可以使用更宽泛的术语“变量”，因为它更通用。

定义变量有两个主要好处：

*   你可以一次性指定一个任意值，并多次使用它。
*   你可以将任意值移到准备部分。

在前面的 CTE 示例中，由于我们处理的不是真实数据，我们只是简单地从 CTE 本身进行选择。在更实际的例子中，我们会将 CTE 与其他表进行交叉连接。

### 设置硬编码常量

CTE 变量的一个简单用途是设置一个任意值供主查询使用。例如，假设我们想使用一个任意税率生成一个简单的价格列表。

我们可以从一个包含税率的 CTE 开始：

```sql
WITH vars AS (
SELECT 0.1 AS taxrate
--  FROM dual   --  Oracle
)
```

现在，我们可以使用简单的交叉连接将 CTE 与`books`表结合起来：

```sql
WITH vars AS (
SELECT 0.1 AS taxrate
--  FROM dual   --  Oracle
)
SELECT * FROM books, vars;
```

结果如下所示：

| **id** | **authorid** | **Truncate** | **published** | **price** | **taxrate** |
| --- | --- | --- | --- | --- | --- |
| 2078 | 765 | The Duel | 1811 | 12.5 | 0.1 |
| 503 | 128 | Uncle Silas | 1864 | 17 | 0.1 |
| 2007 | 99 | North and South | 1854 | 17.5 | 0.1 |
| 702 | 547 | Jane Eyre | 1847 | 17.5 | 0.1 |
| 1530 | 28 | Robin Hood, The ... | 1862 | 12.5 | 0.1 |
| 1759 | 17 | La Curée | 1872 | 16 | 0.1 |
| ~ 1201 行 ~ |

交叉连接将一张表的每一行与另一张表的每一行组合在一起。由于`vars` CTE 只有一行，交叉连接的效果就是简单地给`books`表添加了另一列。

SQL 有一个更现代的交叉连接语法：`books CROSS JOIN vars`。这里，我们将使用较旧的语法，因为它更简单、更易读。

我们现在可以计算含税价格列表：

```sql
WITH vars AS (SELECT 0.1 AS taxrate)
SELECT
id, title,
price, price*taxrate AS tax, price*(1+taxrate) AS total
FROM books, vars;
```

结果如下：

| **Id** | **Title** | **price** | **tax** | **total** |
| --- | --- | --- | --- | --- |
| 2078 | The Duel | 12.5 | 1.25 | 13.75 |
| 503 | Uncle Silas | 17 | 1.7 | 18.7 |
| 2007 | North and South | 17.5 | 1.75 | 19.25 |
| 702 | Jane Eyre | 17.5 | 1.75 | 19.25 |
| 1530 | Robin Hood, The Prince of Thieves | 12.5 | 1.25 | 13.75 |
| 1759 | La Curée | 16 | 1.6 | 17.6 |
| ~ 1201 行 ~ |

当然，我们本可以直接使用`0.1`而不是`taxrate`，并省略 CTE 和交叉连接。然而，CTE 的好处在于，它允许我们在开头设置一次税率，这样便于维护，并且可以在后面多次使用。

### 派生常量

这些值不必是字面值。你也可以从另一个查询中派生这些值。例如，要获取最年长和最年轻的客户，首先在变量中设置最小和最大的出生日期：

```sql
--  vars CTE
SELECT min(dob) AS oldest, max(dob) AS youngest
FROM customers
```

然后你可以将该 CTE 与`customers`表进行交叉连接，以获取匹配的客户：

```sql
WITH vars AS (
SELECT min(dob) AS oldest, max(dob) AS youngest
FROM customers
)
SELECT *
FROM customers, vars
WHERE dob IN(oldest, youngest);
```

你应该会看到类似这样的结果：

| **id** | **givenname** | **familyname** | **...** | **dob** | **...** |
| --- | --- | --- | --- | --- | --- |
| 92 | Nan | Keen | ... | 1943-05-18 | ... |
| 228 | Cam | Payne | ... | 2003-01-27 | ... |
| 577 | Sybil | Service | ... | 2003-01-27 | ... |
| 392 | Daisy | Chain | ... | 1943-05-18 | ... |

要获取个子较矮的客户，你可以将平均身高设置在一个变量中：

```sql
WITH vars AS (SELECT avg(height) AS average FROM customers)
SELECT *
FROM customers, vars
WHERE height<average;
```

这是你用其他方法做不到的事情，因为平均值是一个聚合值。

### 在 CTE 中使用聚合

正如我们多次看到的，你不能将聚合与非聚合查询混合使用。解决方案总是单独计算任何你需要的聚合，然后将结果合并到下一个查询中。

#### 查找每个客户最近的销售记录

例如，假设你想获取每个客户最近一次销售的详细信息。要获取最近的销售记录，你首先需要一个简单的聚合查询：

```sql
SELECT customerid, max(ordered) AS last_order
FROM sales
GROUP BY customerid;
```

你会得到类似这样的结果：

| **Customerid** | **last_order** |
| --- | --- |
| 550 | 2023-04-18 09:18:51.933845 |
| 272 | 2023-04-28 09:15:17.85286 |
| 70 | 2023-04-19 14:00:44.880376 |
| 190 | 2023-04-09 10:12:53.416293 |
| 539 | 2023-04-22 16:14:16.173923 |
| 314 | 2023-04-11 03:33:57.825786 |
| ~ 269 行 ~ |

这里，我们有两个重要数据：客户 ID 以及最近订单的日期和时间。在子查询中使用这个结果，我们可以将其与`customers`和`sales`表连接起来以获取更多详细信息：

```sql
WITH cte(customerid, last_order) AS (
SELECT customerid, max(ordered) AS last_order
FROM sales
GROUP BY customerid
)
SELECT
customers.id AS customerid,
customers.givenname, customers.familyname,
sales.id AS saleid,
sales.ordered_date, sales.total
FROM
sales
JOIN cte ON sales.customerid=cte.customerid
AND sales.ordered=cte.last_order
JOIN customers ON customers.id=cte.customerid
;
```

我们会得到类似这样的结果：

| **customer** | **Givenname** | **familyname** | **sale** | **ordered_date** | **total** |
| --- | --- | --- | --- | --- | --- |
| 287 | Judy | Free | 4209 | 2023-02-04 | 50.5 |
| 26 | Bess | Twishes | 4542 | 2023-02-19 | 11 |
| 368 | Sharon | Sharalike | 4793 | 2023-03-01 | 56 |
| 282 | Howard | Youknow | 4939 | 2023-03-07 | 39 |
| 395 | Holly | Day | 4953 | 2023-03-07 | 75.5 |
| 474 | Alf | Abet | 5092 | 2023-03-13 | 94 |
| ~ 266 行 ~ |

请注意，CTE 被用来连接两个表并充当过滤器。我们实际上不需要它的结果出现在输出中。




#### 查找同名的客户

在第 2 章中，我们了解了如何使用聚合查询查找重复项。我们这样做是为了查找重复的姓名（确实存在）和重复的电话号码（不存在）。

如果我们对重复的客户姓名更加关注，就需要更多关于客户的详细信息。首先，让我们找出重复的姓名：

```
--  公用表表达式
SELECT familyname, givenname FROM customers
GROUP BY familyname, givenname HAVING count(*)>1
```

这里，`customers` 表同时按姓和名进行分组，并筛选出包含多条记录的分组。

将这个查询放入一个 CTE 中，我们就可以将其与 `customers` 表进行连接：

```
WITH names AS (
SELECT familyname, givenname FROM customers
GROUP BY familyname, givenname HAVING count(*)>1
)
SELECT
c.id, c.givenname, c.familyname,
c.email, c.phone
--  等等
FROM customers AS c
JOIN names  ON c.givenname=names.givenname
AND c.familyname=names.familyname
ORDER BY c.familyname, c.givenname;
```

你会得到类似这样的结果：

| `id` | `givenname` | `familyname` | `email` | `phone` |
| --- | --- | --- | --- | --- |
| 429 | Corey | Ander | corey.ander429@example.net | 0355503360 |
| 85 | Corey | Ander | corey.ander85@example.net | 0255501923 |
| 174 | Paul | Bearer | paul.bearer174@example.com | 0370109921 |
| 482 | Paul | Bearer | paul.bearer482@example.com | 0755502522 |
| 402 | Terry | Bell | terry.bell402@example.com | 0755504982 |
| 295 | Terry | Bell | terry.bell295@example.com | 0355509630 |
| ~ 约 16 行 ~ |

我们通过两列连接了 CTE 和 `customers` 表，并包含了他们的 `email` 地址和 `phone` 号码（如果有），以便后续联系。

## CTE 参数名称

默认情况下，列名源自 CTE，并且你应确保所有计算都有别名，如前所述。如果 CTE 中的列没有别名（例如当你计算了某些东西时），那么 (a) 你无法引用该数据，(b) 某些数据库管理系统 (DBMS) 不允许你继续操作。

你还可以在 CTE 名称后指定列名作为参数。例如，当我们查找最早和最晚的出生日期时，我们可以将别名放在 `cte` 表达式中：

```
WITH vars(oldest, youngest) AS (    --  参数名
SELECT min(dob), max(dob)       --  没有别名
FROM customers
)
SELECT *
FROM customers, vars
WHERE dob IN(oldest, youngest);
```

在大多数情况下，是采用这种方式还是在 CTE 内部添加别名，更多是个人偏好问题。如果你包含名称，它们将覆盖 CTE 中的任何别名。

你可能更喜欢 CTE 参数名称的一个原因是，如果你认为它更具可读性，因为你将所有名称都放在了一个地方。稍后，我们将编写涉及多个 CTE 和联合查询的更复杂的 CTE，使用参数名肯定会更容易理解，所以从这里开始，你会更多地看到这种风格。

## 使用多个公用表表达式

我们已经看到，在其最简单的形式中，CTE 可以写成一个子查询：

```
SELECT columns
FROM (
SELECT columns FROM table
) AS sq;
```

通过将这个子查询放在开头，CTE 可以使它更易于管理：

```
WITH cte AS (
SELECT columns FROM table
)
SELECT columns
FROM cte;
```

这已经是一个改进，但当子查询本身还包含子查询时，改进变得更加明显：

```
SELECT columns
FROM (
SELECT columns FROM (
SELECT columns FROM table
) AS sq1
) AS sq2;
```

这被称为嵌套子查询，如果变得过于复杂，它可能会成为一场噩梦。

幸运的是，CTE 的工作方式要简单得多：

```
WITH
sq1 AS (SELECT columns FROM table),
sq2 AS (SELECT columns FROM sql1)
SELECT columns FROM sq2;
```

你可以有多个以这种方式链接的 CTE，只要记得用逗号将它们分隔开。正如你在本例中看到的，每个子查询都可以引用链中的前一个。

我们稍后会进一步构建这个例子，并且会看到额外的 CTE 不一定必须引用前面的 CTE。

### 使用多个 CTE 汇总重复姓名

当我们生成重复姓名列表时，每个姓名实例都占一行。在第 7 章中，我们生成了一个更精简的列表，但没有使用 CTE 的优势。

在这里，我们将重新生成精简的列表，但使用 CTE 使其更易于操作。

我们从先前查询重复姓名的语句开始：

```
WITH names AS (
SELECT familyname, givenname FROM customers
GROUP BY familyname, givenname HAVING count(*)>1
)
SELECT
c.id, c.givenname, c.familyname,
c.email, c.phone
FROM customers AS c
JOIN names  ON c.givenname=names.givenname
AND c.familyname=names.familyname
ORDER BY c.familyname, c.givenname;
```

这次，我们将结果放入第二个 CTE 中：

```
WITH
names AS (
SELECT familyname, givenname FROM customers
GROUP BY familyname, givenname HAVING count(*)>1
),
duplicates(givenname, familyname, info) AS (
SELECT
c.givenname, c.familyname,
cast(c.id AS varchar(5)) || ': ' || c.email
--  MSSQL: 使用 +
FROM customers AS c     --  Oracle: 不用 AS
JOIN names ON c.givenname=names.givenname
AND c.familyname=names.familyname
)
SELECT * from duplicates
ORDER by familyname, giv1enname;
```

注意

*   布局已更改，以使多个 CTE 更易于理解。
*   `duplicates` CTE 使用了参数名以求简洁。对于 `names` CTE 则没有必要这样做，因为其中没有计算值；不过，为了保持一致性，你可能也想这样做。
*   我们没有单独列出 `id`，而是将其转换为字符串并与 `email` 地址拼接。这是为后续步骤做准备。
*   为简单起见，我们忽略了 `phone` 号码，因为它可能缺失。

你可以看到目前为止的结果：

| `givenname` | `familyname` | `info` |
| --- | --- | --- |
| Corey | Ander | 429: corey.ander429@example.net |
| Corey | Ander | 85: corey.ander85@example.net |
| Paul | Bearer | 174: paul.bearer174@example.com |
| Paul | Bearer | 482: paul.bearer482@example.com |
| Terry | Bell | 402: terry.bell402@example.com |
| Terry | Bell | 295: terry.bell295@example.com |
| ~ 约 16 行 ~ |

下一步是通过合并 `info` 列的值来汇总它们：

```
WITH
names AS ( ),
duplicates(givenname, familyname, info) AS ( )
SELECT
givenname, familyname, count(*),
--  PostgreSQL, MSSQL
string_agg(info,', ') AS info
--  MySQL/MariaDB
--  group_concat(info SEPARATOR ', ') AS info
--  SQLite
--  group_concat(info,', ') AS info
--  Oracle
--  listagg(info,', ') AS info
FROM duplicates
GROUP BY familyname, givenname
ORDER by familyname, givenname;
```

汇总后的列表如下所示：

| `givenname` | `familyname` | `count` | `info` |
| --- | --- | --- | --- |
| Corey | Ander | 2 | 429: corey.ander ..., 85: corey.ander8 ... |
| Paul | Bearer | 2 | 174: paul.bearer ..., 482: paul.bearer ... |
| Terry | Bell | 2 | 402: terry.bell4 ..., 295: terry.bell2 ... |
| Mary | Christmas | 2 | 465: mary.christ ..., 594: mary.christ ... |
| Ida | Dunnit | 2 | 504: ida.dunnit5 ..., 90: ida.dunnit90 ... |
| Judy | Free | 2 | 93: judy.free93@ ..., 287: judy.free28 ... |
| Annie | Mate | 2 | 505: annie.mate5 ..., 357: annie.mate3 ... |
| Ken | Tuckey | 2 | 98: ken.tuckey98 ..., 488: ken.tuckey4 ... |

我们将在接下来的章节中看到更多关于多 CTE 的例子。



### 递归 CTE

如你所见，使用 CTE 的一个特性是，一个 CTE 可以引用之前的 CTE。另一个特性是 CTE 可以引用自身。

任何引用自身的事物都被称为**递归**。如果你是程序员，递归函数就是调用自身的函数，如果不妥善处理，它们的风险很高。同样，一个递归 CTE 如果不小心，也可能非常危险。

一个递归 CTE 有两种形式，具体取决于你的 DBMS：

```
--  PostgreSQL, MariaDB/MySQL, SQLite
WITH RECURSIVE cte AS (
--  Anchor
SELECT ...
UNION
--  Recursive Member
SELECT ... FROM cte WHERE ...
)
--  MSSQL, Oracle
WITH cte AS (
--  Anchor
SELECT ...
UNION ALL
--  Recursive Member
SELECT ... FROM cte WHERE ...
)
```

正如你所看到的，PostgreSQL、MariaDB/MySQL 和 SQLite 使用 `RECURSIVE` 关键字。MSSQL 和 Oracle 不使用，但要求使用 `UNION ALL` 而不是简单的 `UNION`。

在两种情况下，你都会看到递归 CTE 有两个部分：

*   **锚点**定义了起点或第一个成员。
    在简单情况下，可能只有一个值，但在其他查询中，可能不止一个。
*   **递归成员**基于从前一个 CTE 迭代继承的数据定义数据。也就是说，它定义了下一个成员。
    同样，如果有多个锚点成员，那么也会有多个递归成员。

请注意，递归 CTE 必须定义何时结束，或者更准确地说，何时可以继续。通常，这是通过一个 `WHERE` 子句实现的，正如你之前看到的，但也可以使用任何其他方法，比如连接。

一个简单的递归 CTE 例子是生成一个简单的序列。例如：

```
--  PostgreSQL, MariaDB/MySQL, SQLite
WITH RECURSIVE cte(n) AS (
--  Anchor
SELECT 1
UNION
--  Recursive Member
SELECT n+1 FROM cte WHERE n<10
)
SELECT * FROM cte;
--  MSSQL, Oracle
WITH cte(n) AS (
--  Anchor
SELECT 1    --  Oracle: FROM dual
UNION ALL
--  Recursive Member
SELECT n+1 FROM cte WHERE n<10
)
SELECT * FROM cte;
```

这个 CTE 为了方便包含了一个参数 (`cte(n)`)。否则，你可以把别名放在 `SELECT` 语句中。

单个锚点值，在这个例子中是数字 1。递归（下一个）值是 `n+1`，只要 `n<10`。之后，它就会停止，最终你得到：

| **N** |
| 1 |
| 2 |
| 3 |
| ... |
| 8 |
| 9 |
| 10 |

——一个从一到十的数字序列。

递归 CTE 是标准 SQL 中最接近迭代或循环的东西。^(⁶)

递归 CTE 的两个常见用途是：

*   生成序列
*   遍历层次结构

我们还将使用递归 CTE 将字符串拆分成更小的部分，只是为了向你展示可以如何在查询中发挥一点创造力。

### 生成序列

我们已经看到了如何生成一个数字序列：

```
WITH cte AS (
--  Anchor
SELECT 0 AS n
UNION ALL
--  Recursive
SELECT n+1 FROM cte WHERE n<100
)
SELECT * FROM cte;
```

需要记住的是，递归成员有一个 `WHERE` 子句来限制序列。没有它，递归查询将试图永远运行下去，而你知道，没有什么是永恒的。

MSSQL 有一个内置的安全限制，即 100 次递归，我们稍后必须绕过它：

```
--  MSSQL
WITH cte (
)
SELECT ... FROM cte OPTION(MAXRECURSION ...);
```

其他数据库没有这个限制，但对于 PostgreSQL、MariaDB 和 MySQL，你可以轻松设置一个时间限制：

```
--  PostgreSQL
SET statement_timeout TO '5s';
--  MariaDB
SET MAX_STATEMENT_TIME=1;       --  秒
--  MySQL
SET MAX_EXECUTION_TIME=1000;    --  毫秒
```

如果你确信你的递归会正常终止，就不需要担心这个。然而，在 MSSQL 中，对于某些查询，你将需要增加或禁用递归限制。

不过，在接下来的内容中包含一个简单的数字序列作为安全保障总是好的。

序列有用的一种情况是获取一系列日期。这将简单地定义一个开始日期，并在递归成员中增加一天。

CTE 的开始足够简单：

```
--  PostgreSQL, MariaDB / MySQL
WITH RECURSIVE dates(d, n) AS (
SELECT date'2023-01-01', 1
)
SELECT * FROM dates;
--  MSSQL
WITH dates(d, n) AS (
SELECT cast('2023-01-01' as date), 1
)
SELECT * FROM dates;
--  Oracle
WITH dates(d, n) AS (
SELECT date '2023-01-01', 1 FROM dual
)
SELECT * FROM dates;
--  SQLite
WITH RECURSIVE dates(d, n) AS (
SELECT '2023-01-01', 1
)
SELECT * FROM dates;
```

请注意，第一个值 `d` 被转换为日期类型，SQLite 除外，它没有日期类型。设置为 `1` 的 `n` 被添加为序列号，但实际上没有必要。这里添加它是为了说明如何用它来防止 CTE 超出范围。

递归部分也足够简单，但增加一天的方式因 DBMS 而异：

```
--  PostgreSQL
WITH RECURSIVE dates(d, n) AS (
SELECT date'2023-01-01', 1
UNION
SELECT d+1, n+1 FROM dates
WHERE d<'2023-05-01' AND n<10000
)
SELECT * FROM dates;
--  MariaDB / MySQL
WITH RECURSIVE dates(d, n) AS (
SELECT date'2023-01-01', 1
UNION
SELECT date_add(d, interval 1 day), n+1 FROM dates
WHERE d<'2023-05-01' AND n<10000
)
SELECT * FROM dates;
--  MSSQL
WITH dates(d, n) AS (
SELECT cast('2023-01-01' as date), 1
UNION ALL
SELECT dateadd(day,1,d), n+1 FROM dates
WHERE d<'2023-05-01' AND n<10000
)
SELECT * FROM dates;
--  SQLite
WITH RECURSIVE dates(d, n) AS (
SELECT '2023-01-01', 1
UNION
SELECT strftime('%Y-%m-%d',d,'+1 day'), n+1 FROM dates
WHERE d<'2023-05-01' AND n<10000
)
SELECT * FROM dates;
--  Oracle
WITH dates(d, n) AS (
SELECT date '2023-01-01', 1 FROM dual
UNION ALL
SELECT d+1, n+1 FROM dates
WHERE d<date'2023-05-01' AND n<10000
)
SELECT * FROM dates;
```

你将看到一系列的日期（和数字）：

| **D** | **n** |
| --- | --- |
| 2023-01-01 | 1 |
| 2023-01-02 | 2 |
| 2023-01-03 | 3 |
| 2023-01-04 | 4 |
| 2023-01-05 | 5 |
| 2023-01-06 | 6 |
| ~ 121 行 ~ |

你会注意到，对于 MSSQL，我们添加了 `OPTION (MAXRECURSION 0)`，这基本上禁用了递归限制。

还要注意 `WHERE` 子句中的 `AND n<10000`。这个数字相当大，相当于超过 27 年，但它不是无限的。如果你在何时停止 CTE 上犯了错误，这个表达式应该能限制递归次数。

你可能想知道为什么你想要一个介于 `2023-01-01` 和 `2023-05-01` 之间的日期序列，答案会是“为什么不呢？”，这并没有说服力。然而，我们将使用这种技术来克服第 8 章中提到的问题：我们的一些日期会从摘要中缺失。

### 连接序列 CTE 以获取缺失值

你可以将生成序列的递归 CTE 与另一个存在序列缺口的表或 CTE 进行`JOIN`，以填充缺失的值。

例如，要获取每年出生的客户数量，某些年份的数据可能会缺失，但你仍然希望将这些缺失的年份也包含在内。

首先，获取年份序列：

```sql
--  PostgreSQL, MariaDB/MySQL, SQLite
WITH RECURSIVE
allyears(year) AS (
SELECT 1940
UNION
SELECT year+1 FROM allyears WHERE year<2010
)
--  MSSQL, Oracle
WITH
allyears(year) AS (
SELECT 1940
UNION ALL
SELECT year+1 FROM allyears WHERE year<2010
)
```

接下来，获取客户（`id`）及其出生年份：

```sql
--  PostgreSQL, MariaDB/MySQL, Oracle
yobs(yob) AS (
SELECT id, EXTRACT(year FROM dob)
FROM customers WHERE dob IS NOT NULL
)
--  MSSQL
yobs(yob) AS (
SELECT id, year(dob)
FROM customers WHERE dob IS NOT NULL
)
--  SQLite
yobs(yob) AS (
SELECT id, strftime('%Y',dob)
FROM customers WHERE dob IS NOT NULL
)
```

最后，将它们进行`JOIN`并获取聚合结果：

```sql
WITH RECURSIVE  --  MSSQL, Oracle: 无 RECURSIVE
allyears(year) AS ( ),
yobs AS ( )
SELECT allyears.year, count(*) AS nums
FROM allyears LEFT JOIN yobs ON allyears.year=yobs.yob
GROUP BY allyears.year
ORDER BY allyears.year;
```

你需要使用`LEFT JOIN`来包含所有的年份序列，即使某一年没有匹配到客户数据；毕竟，这正是使用它的原因。

| `year` | `nums` |
| --- | --- |
| 1940 | 1 |
| 1941 | 1 |
| 1942 | 1 |
| 1943 | 1 |
| 1944 | 1 |
| 1945 | 1 |
| ~ 共 71 行 ~ | |

我们将对销售数据进行同样的处理。

### 包含缺失日期的每日对比

同样的方法也适用于缺失的日期。在第 8 章中，我们生成了每日销售摘要。然后我们创建了一个包含可用每日销售数据的视图。我们可以从该视图中进行选择：

```sql
SELECT *
FROM daily_sales
ORDER BY ordered_date;
```

你会得到类似这样的结果：

| `ordered_date` | `ordered_month` | `daily_total` |
| --- | --- | --- |
| 2022-04-08 | 2022-04 | 97.5 |
| 2022-04-09 | 2022-04 | 96 |
| 2022-04-10 | 2022-04 | 191 |
| 2022-04-11 | 2022-04 | 201.5 |
| 2022-04-12 | 2022-04 | 91 |
| 2022-04-13 | 2022-04 | 160 |
| ~ 共 385 行 ~ |   |   |

然而，如果你仔细查看，会发现一些日期缺失了。我们正准备填补这些空缺。

为此，我们需要以下内容：
*   `daily_sales` 视图
*   一个包含每日销售首尾日期的 CTE
*   一个日期序列

你已经知道如何生成日期序列了。这次，我们不在任意日期开始和停止，而是根据`daily_sales`视图的最早和最晚日期来开始和停止。我们可以将这些值放入一个 CTE 中以便引用：

```sql
WITH
vars(first_date, last_date) AS (
SELECT min(ordered_date), max(ordered_date)
FROM daily_sales
)
```

现在我们可以使用这些值来生成日期序列：

```sql
--  PostgreSQL
WITH RECURSIVE
vars(first_date, last_date) AS ( ),
dates(d) AS (
SELECT first_date FROM vars
UNION
SELECT d+1 FROM vars, dates WHERE d<last_date
)
--  MariaDB / MySQL
WITH RECURSIVE
vars(first_date, last_date) AS ( ),
dates(d) AS (
SELECT first_date FROM vars
UNION
SELECT date_add(d, interval 1 day)
FROM vars, dates WHERE d<last_date
)
--  MSSQL
WITH
vars(first_date, last_date) AS ( ),
dates(d) AS (
SELECT first_date FROM vars
UNION ALL
SELECT dateadd(day,1,d)
FROM vars, dates WHERE d<last_date
)
--  SQLite
WITH RECURSIVE
vars(first_date, last_date) AS ( ),
dates(d) AS (
SELECT first_date FROM vars
UNION
SELECT strftime('%Y-%m-%d',d,'+1 day')
FROM vars, dates WHERE d<last_date
)
--  Oracle
WITH
vars(first_date, last_date) AS ( ),
dates(d) AS (
SELECT first_date FROM vars
UNION ALL
SELECT d+1 FROM vars, dates WHERE d<last_date
)
```

对于那些使用`RECURSIVE`关键字的 DBMS，即使某些 CTE 不是递归的，你也只需在开头使用一次该关键字。

注意，我们交叉连接了`vars`和`dates`，这是将变量应用到另一个表的常用技巧。我们本可以写`CROSS JOIN`，但没必要这么麻烦。

我们现在可以使用`LEFT JOIN`来完成查询，以获取所有的日期序列：

```sql
WITH RECURSIVE  --  MSSQL, Oracle: 无 RECURSIVE
vars(first_date, last_date) AS (
--  等等
),
dates(d) AS (
--  等等
)
SELECT d AS ordered_date, daily_sales.daily_total
FROM dates LEFT JOIN daily_sales ON dates.d=daily_sales.ordered_date
ORDER BY dates.d;
```

现在我们将看到以下结果：

| `ordered_date` | `daily_total` |
| --- | --- |
| 2022-04-08 | 97.5 |
| 2022-04-09 | 96 |
| 2022-04-10 | 191 |
| 2022-04-11 | 201.5 |
| 2022-04-12 | 91 |
| 2022-04-13 | 160 |
| ~ 共 387 行 ~ | |

注意，我们选择了`dates.d AS ordered_date`，而不是`daily_sales`视图中的`ordered_date`。这是因为后者存在一些缺失日期，而这正是我们最初要费这番功夫的原因。

当然，生成一个简单的序列并不是递归 CTE 的唯一用途。

### 遍历层次结构

递归 CTE 的另一个用例是遍历层次结构。我们要看的层次结构位于`employees`表中：

```sql
SELECT * FROM employees;
```

当然，在一个真实的员工表中，会有更多细节；这里我们只包含足以说明问题的内容。

具体来说，你会看到在`employees`表中有一个`supervisorid`列，它是引用同一张表的外键：

```sql
employees.supervisorid ➤ employees.id
```

一种更简单但错误的方法是：要么包含主管的姓名（这与在书籍表中包含作者姓名一样，原因相同，是错误的），要么引用另一个主管表（这出于一个更微妙的原因，也是错误的）。

对于书籍和作者，关键在于作者与书籍不是同一种事物。在一个设计良好的数据库中，每个表只包含一种类型的成员。但对于员工和主管来说，情况并非如此。简单来说，主管就是另一名员工。

我们将遍历`employees`表，以获取员工及其主管的列表。

#### 获取单层层次结构

不使用递归 CTE，你可以通过简单的`OUTER JOIN`到同一张表来获取员工的主管。这通常被称为`自连接`：

```sql
SELECT
e.id AS eid,
e.givenname, e.familyname,
s.id AS sid,
s.givenname||' '||s.familyname AS supervisor
--  s.givenname+' '+s.familyname AS supervisor  --  MSSQL
FROM employees AS e LEFT JOIN employees AS s
ON e.supervisorid=s.id                --  Oracle: 不用 AS
ORDER BY e.id;
```

你会得到类似这样的结果：

| `eid` | `givenname` | `familyname` | `sid` | `supervisor` |
| --- | --- | --- | --- | --- |
| 1 | Marmaduke | Mayhem | 10 | Beryl Bubbles |
| 2 | Osric | Pureheart | 12 | Mildred Thisenthat |
| 3 | Rubin | Croucher | [NULL] | [NULL] |
| 4 | Gladys | Raggs | 29 | Fred Nurke |
| 5 | Cynthia | Hyphen-Smythe | 12 | Mildred Thisenthat |
| 6 | Sebastian | Trefether | 5 | Cynthia Hyphen-Smythe |
| ~ 共 34 行 ~ | | | | |

关键在于，当将表与自身连接时，你需要为该表指定两个不同的别名来限定连接。


### 使用递归 CTE 实现多级层级结构

我们真正需要的不只是直接上级，而是每位员工的所有上级组成的层级列表。这当然需要使用递归 CTE。

锚定成员将是那些没有上级的员工，通常是位于层级结构顶端的人员：

```sql
WITH RECURSIVE      --  MSSQL, Oracle: 不用 RECURSIVE
cte(id, givenname, familyname, supervisorid,
supervisors, n) AS (
--  锚定成员
SELECT
    id, givenname, familyname, supervisorid, '', 1
    FROM employees WHERE supervisorid IS NULL
)
```

这些列包含部分原始详细信息，以及一个上级字符串。显然，对于锚定成员，上级 ID 将是`NULL`，字符串为空。你还会注意到一个从 1 开始的序列号。这是我们稍后会用到的一个技巧。

锚定成员可能有多行。这没关系，处理方式仍然相同。只是会启动多个序列。

递归成员将是那些有上级的员工（即其余员工），其上级列表会逐渐增长：

```sql
--  目前尚不支持 MSSQL 或 MariaDB/MySQL!
WITH RECURSIVE  --  MSSQL, Oracle: 不用 RECURSIVE
cte(id, givenname, familyname, supervisorid,
supervisors, n) AS (
--  锚定成员
    SELECT
        id, givenname, familyname, supervisorid, '', 1
        FROM employees WHERE supervisorid IS NULL
UNION ALL
--  递归成员：其他员工（supervisorid 不为 NULL）
    SELECT
        e.id, e.givenname, e.familyname, e.supervisorid,
        cte.givenname||' '||cte.familyname||' < '||
        cte.supervisors, n+1
        FROM cte JOIN employees AS e ON cte.id=e.supervisorid
        --  Oracle: 不用 AS
)
SELECT * FROM cte
ORDER BY id;
```

这个连接类似于之前的自连接。当前员工在`e`表别名中引用，这个别名表与 CTE（代表上级）进行连接。原始数据来自别名表，而上级的详细信息则被拼接为新的`supervisors`参数。

通常，你会希望用`WHERE`子句来限制递归。对于这个例子，连接本身就会完成这项工作，因为当没有更多可以连接的行时，它会自动停止。

关键在于`supervisors`字符串的表达式。在递归成员中，CTE 代表继承的值。

| **id** | **givenname** | **familyname** | **sid** | **supervisors** | **n** |
| --- | --- | --- | --- | --- | --- |
| 1 | Marmaduke | Mayhem | 10 | Beryl Bubbles < Mildred Thisenth ... < | 3 |
| 2 | Osric | Pureheart | 12 | Mildred Thisenth ... < | 2 |
| 3 | Rubin | Croucher | [NULL] | [NULL] | 1 |
| 4 | Gladys | Raggs | 29 | Fred Nurke < Murgatroyd Murdo ... < Rubin Croucher < | 4 |
| 5 | Cynthia | Hyphen-Smythe | 12 | Mildred Thisenth ... < | 2 |
| 6 | Sebastian | Trefether | 5 | Cynthia Hyphen-S ... < Mildred Thisenth ... < | 3 |
| ~ 共 34 行 ~ |

这在大多数数据库管理系统中都有效，但目前还不支持 MSSQL 或 MariaDB/MySQL。不过，它已经`接近`能用了。

对于 MariaDB/MySQL，锚定成员中的`''`会导致它推断该字符串长度为零，因此`supervisors`列将为空。

你需要在锚定成员中将空字符串转换（cast）为更长的字符串：

```sql
SELECT
    ..., cast('' AS char(255)), 1
    FROM employees WHERE supervisorid IS NULL
```

MySQL 的一个长期问题是，你不能转换为`varchar`，而是转换为`char`，而这个`char`与常规的`char`不同，它不是固定长度的，所以实际上还是`varchar`。没人知道为什么。在 MariaDB 中，他们允许使用`varchar`。长度`255`应该足够了。

MSSQL 的情况更糟。自然，锚定成员和递归成员中的列应该在数据类型上匹配，但在 MSSQL 中，这种匹配需要非常精确。两者都是字符串类型还不够。字符串需要是相同类型和大小。拼接字符串会产生更长的字符串，MSSQL 会判定这些更长的字符串与其它字符串在数据上不兼容。

在这种情况下，你需要将*两个*表达式都转换为相同类型：

```sql
SELECT
    ..., cast('' AS nvarchar(255)), ...
    FROM employees WHERE supervisorid IS NULL
UNION ALL
SELECT
    ...,
    cast(cte.givenname+' '+cte.familyname
         +' < '+cte.supervisors as nvarchar(255)), ...
    FROM cte JOIN employees AS e ON cte.id=e.supervisorid
```

进行这些更改后，查询应该就能工作了。

### 清理列表末尾的多余字符

你会注意到，除了空的`supervisors`字符串外，末尾还有一个`<`。这是因为在递归成员中总是添加了这个字符。

如果你查看`n`列，会发现它代表层级编号。这个字符应该只在已经存在另一个上级时才添加——也就是说，当我们正在添加第二个或后续的上级时。这意味着当`n`达到 2 或更高时。

我们可以通过使用`CASE ... END`表达式来修改这部分表达式：

```sql
--  其他数据库
cte.givenname||' '||cte.familyname
||  CASE WHEN n>1 THEN ' < ' ELSE '' END
    || cte.supervisors
```

这样会产生更清晰的结果：

| **id** | **givenname** | **familyname** | **Sid** | **supervisors** | **n** |
| --- | --- | --- | --- | --- | --- |
| 1 | Marmaduke | Mayhem | 10 | Beryl Bubbles < Mildred Thisenth ... | 3 |
| 2 | Osric | Pureheart | 12 | Mildred Thisenth ... | 2 |
| 3 | Rubin | Croucher | [NULL] | [NULL] | 1 |
| 4 | Gladys | Raggs | 29 | Fred Nurke < Murgatroyd Murdo ... < Rubin Croucher | 4 |
| 5 | Cynthia | Hyphen-Smythe | 12 | Mildred Thisenth ... | 2 |
| 6 | Sebastian | Trefether | 5 | Cynthia Hyphen-S ... < Mildred Thisenth ... | 3 |
| ~ 共 34 行 ~ |

当然，我们不再需要最后的`n`列了。


## 处理表字面量

在某些时候，您可能想要处理一组尚未保存到任何位置的值——例如一个虚拟表中的数据，该表会存在足够长的时间让您处理数据，然后在您完成后悄然消失。

从原理上讲，SQL 在将字面值插入表时一直都在这样做。例如，像下面这样的语句：

```sql
INSERT INTO table(columns)
VALUES ( ... ), ( ... ), ( ... );
```

就是从一个由 `VALUES` 子句生成的虚拟表中插入数据。这也意味着，从原理上讲，您应该能够将 `VALUES ...` 作为一个虚拟表来使用，而无需实际插入任何数据。不幸的是，事情并非如此简单。

一个 `表字面量` 是一个表达式，其结果是一组行和列——即一个虚拟表。如果一切按计划进行，它可能看起来像这样：

```sql
VALUES ('a','apple'), ('b','banana'), ('c','cherry')
```

并非所有的数据库管理系统（DBMS）都这样看待它。有些 DBMS 确实允许这样的表达式，但其他的则需要稍微复杂一点的写法。

稍后，我们将需要使用一个虚拟表来进行实验，所以第一步是将其放入一个 `CTE`（公用表表达式）中。使用标准语法，您可以写：

```sql
--  PostgreSQL, MariaDB (非 MySQL), SQLite
WITH cte(id,value) AS (
  VALUES ('a','apple'), ('b','banana'), ('c','cherry')
)
SELECT * FROM cte;
```

您将看到以下结果：

| id | value |
| --- | --- |
| a | apple |
| b | banana |
| c | cherry |

请注意，我们已经将列名包含在了 `CTE` 的名称中。

对于其他的 DBMS，有各种替代写法：

```sql
--  MSSQL
WITH cte(id,value) AS (
  SELECT * FROM
    (VALUES ('a','apple'), ('b','banana'),
      ('c','cherry')) AS sq(a,b)
)
SELECT * FROM cte;
```
```sql
--  MySQL (非 MariaDB)
WITH cte(id,value) AS (
  VALUES ROW('a','apple'), ROW('b','banana'),
    ROW('c','cherry')
)
SELECT * FROM cte;
```
```sql
--  Oracle
WITH cte(id,value) AS (
  SELECT 'a','apple' FROM dual
  UNION ALL SELECT 'b','banana' FROM dual
  UNION ALL SELECT 'c','cherry' FROM dual
)
SELECT * FROM cte;
```

如您所见，最笨拙版本的“奖项”属于 Oracle，它还不支持一个恰当的 `表字面量`。显然，这个功能即将推出。

`MSSQL` 确实支持 `表字面量`，但出于某种未知的原因，它必须放在一个子查询内，并且必须有一个虚构的子查询名称和虚构的列名。

`MySQL` 也支持 `表字面量`，但要求将每一行都放在一个 `ROW()` 构造器中，因为 `MySQL` 有一个非标准的 `values()` 函数，这与其简单地用作 `表字面量` 相冲突。这是 `MariaDB` 和 `MySQL` 不同的情况之一。

### 使用表字面量进行测试

您可能想要使用一次性值的一个原因是，当您在测试某些东西，而又没有精力将测试值放入一个真实或临时表中时。

例如，假设您想测试计算日期之间的差值，比如计算年龄。您可以这样做：

```sql
WITH dates(dob,today) AS (
  --  列表：出生日期和今日日期值
)
SELECT
  --  今日日期 - 出生日期 AS 年龄
FROM dates;
```

实际的代码被注释掉了，因为各个 DBMS 都有自己的方法。由于日期字面量的存在，这变得进一步复杂。

我们将尝试使用以下一系列日期：

| `dob` | `today` |
| --- | --- |
| 1940-07-07 | 2023-01-01 |
| 1943-02-25 | 2023-01-01 |
| 1942-06-18 | 2023-01-01 |
| 1940-10-09 | 2023-01-01 |
| 1940-07-07 | 2022-12-31 |
| 1943-02-25 | 2022-12-31 |
| 1942-06-18 | 2022-12-31 |
| 1940-10-09 | 2022-12-31 |
| 1940-07-07 | 2023-07-07 |
| 1943-02-25 | 2023-02-25 |
| 1942-06-18 | 2023-06-18 |
| 1940-10-09 | 2023-10-09 |

（如果您认出了这些出生日期，请不要声张。）

首先，您需要设置您的 `dates` `CTE`。这很复杂，因为在 SQL 中，日期字面量是用单引号括起来的。然而，如果没有上下文，SQL 会将单引号字面量视为字符串，这无法用于日期计算。例外是 `SQLite`，它本来就只处理日期字符串。

`dates` `CTE` 看起来会像这样：

```sql
--  PostgreSQL, MariaDB (非 MySQL)
WITH dates(dob, today) AS (
  VALUES
    (date'1940-07-07',date'2023-01-01'),
    ('1943-02-25','2023-01-01'),
    ('1942-06-18','2023-01-01')
    --  等等
)
```
```sql
--  MySQL (非 MariaDB)
WITH dates(dob, today) AS (
  VALUES
    row(date'1940-07-07',date'2023-01-01'),
    row('1943-02-25','2023-01-01'),
    row('1942-06-18','2023-01-01')
    --  等等
)
```
```sql
--  MSSQL
WITH dates(dob, today) AS (
  SELECT * FROM (VALUES
    (cast('1940-07-07' as date),
      cast('2023-01-01' as date)),
    ('1943-02-25','2023-01-01'),
    ('1942-06-18','2023-01-01')
    --  等等
  ) AS sq(a,b)
)
```
```sql
--  SQLite
WITH dates(dob, today) AS (
  VALUES
    ('1940-07-07','2023-01-01'),
    ('1943-02-25','2023-01-01'),
    ('1942-06-18','2023-01-01')
    --  等等
)
```
```sql
--  Oracle
WITH dates(dob, today) AS (
  SELECT date'1940-07-07',date'2023-01-01' FROM dual
  UNION ALL SELECT date'1943-02-25',date'2023-01-01'
    FROM dual
  UNION ALL SELECT date'1942-06-18',date'2023-01-01'
    FROM dual
    --  等等
)
```

请注意

*   对于 `PostgreSQL`、`MariaDB/MySQL` 和 `Oracle`，您可以使用简单的表达式 `date'...'` 将字面量解释为日期。
*   对于 `PostgreSQL`、`MariaDB/MySQL` 和 `MSSQL`，仅需对第一行的字面量进行转换；SQL 会从那里理解您的意图。而 `Oracle` 则需要所有行都进行转换。

现在您有了一个包含一组测试日期的虚拟表。您可以尝试您的年龄计算了：

```sql
--  PostgreSQL
WITH dates(dob, today) AS (
  --  等等
)
SELECT
  dob, today,
  extract(year from age(today,dob)) AS 年龄
FROM dates;
```
```sql
--  MariaDB/MySQL
WITH dates(dob, today) AS (
  --  等等
)
SELECT
  dob, today,
  timestampdiff(year,dob,current_timestamp) AS 年龄
FROM dates;
```
```sql
--  MSSQL
WITH dates(dob, today) AS (
  --  等等
)
SELECT
  dob, today,
  datediff(year,dob,today) AS 年龄
FROM dates;
```
```sql
--  SQLite
WITH dates(dob, today) AS (
  --  等等
)
SELECT
  dob, today,
  cast(
    strftime('%Y.%m%d', today) - strftime('%Y.%m%d', dob)
    as int) AS 年龄
FROM dates;
```
```sql
--  Oracle
WITH dates(dob, today) AS (
  --  等等
)
SELECT
  dob, today,
  trunc(months_between(today,dob)/12) AS 年龄
FROM dates;
```

我们已经在第 4 章中指出过 `MSSQL` 计算年龄时出错的问题，而这是您可以测试该问题的一种方法。

### 使用表字面量进行排序

在第 5 章中，我们提到了对字符串值排序的问题。简而言之，对于本应按顺序排列的项目，按字母顺序排列很少是最佳方式。

例如，如果你有星期几的名称、月份的名称、彩虹的颜色，甚至是数字的名称（`"One"`、`"Two"`、`"Three"`等），将字符串按字母顺序排序只会让事情变得混乱。

在第 5 章中，我们通过依赖字符串位置取了巧。另一个更具弹性的解决方案是使用一个（虚拟的）值表及其正确位置。

在第 8 章中，我们生成了一个按工作日统计的销售摘要：

```sql
WITH
data AS (...)
summary AS (...)
SELECT
weekday_number, total,
100*total/sum(total) OVER()
FROM weekday_number
ORDER BY weekday_number;
```

问题是我们必须获取工作日编号才能正确排序。如果能使用工作日名称会更好。然后，我们可以使用一个额外的虚拟表来对名称进行排序。

#### 使用工作日名称重新定义 data CTE

首先，让我们用日期名称重做 `data` CTE：

```sql
--  PostgreSQL, Oracle
WITH data AS (
SELECT to_char(ordered,'FMDay') AS weekday, total
FROM sales
)
--  MSSQL
WITH data AS (
SELECT datename(weekday,ordered) AS weekday, total
FROM sales
)
--  MariaDB/MySQL
WITH data AS (
SELECT date_format(ordered,'%W'), total
FROM sales
)
```

你会注意到列表中没有包含 SQLite。这是因为它没有获取工作日名称的方法。如果你需要，你会想用到下一节的反向技术。

#### 按工作日名称分组

`summary` CTE 现在将按工作日名称分组：

```sql
WITH
data AS (
SELECT
... AS weekday,
total
FROM sales
),
summary AS (
SELECT weekday, sum(total) AS total
FROM data
GROUP BY weekday
)
--  等等
```

#### 创建用于工作日的表字面量

我们现在将需要一个包含星期几以及序列号的表字面量。

| **序号** | **工作日** |
| --- | --- |
| 1 | 星期一 |
| 2 | 星期二 |
| 3 | 星期三 |
| 4 | 星期四 |
| 5 | 星期五 |
| 6 | 星期六 |
| 7 | 星期日 |

```sql
--  PostgreSQL, MariaDB (非 MySQL), SQLite
weekdays(sequence,weekday) AS (
VALUES (1,'星期一'),(2,'星期二')           --  等等
)
--  MySQL (非 MariaDB)
weekdays(sequence,weekday) AS (
VALUES row(1,'星期一'), row(2,'星期二')    --  等等
)
--  MSSQL
weekdays(sequence,weekday) AS (
SELECT * FROM (
VALUES (1,'星期一'),(2,'星期二')       --  等等
) AS sq(a,b)
)
--  Oracle
weekdays(sequence,weekday) AS (
SELECT 1,'星期一' FROM dual
UNION ALL SELECT 2,'星期二' FROM dual      --  等等
)
```

#### 连接并按序列号排序

最后，要进行排序，你可以将 `summary` CTE 与 `weekdays` CTE 连接起来，并按序列号排序：

```sql
WITH
data AS ( ),
summary AS ( ),
weekdays(dob, today) AS ( )
SELECT
summary.weekday, summary.total,
100*total/sum(summary.total) OVER()
FROM summary JOIN weekdays
ON summary.weekday=weekdays.weekday
ORDER BY weekdays.sequence;
```

你应该会看到类似这样的结果：

| **工作日** | **总销售额** | **占比** |
| --- | --- | --- |
| 星期一 | 49304 | 15.081 |
| 星期二 | 45156.5 | 13.813 |
| 星期三 | 45959.5 | 14.058 |
| 星期四 | 47528 | 14.538 |
| 星期五 | 42372.5 | 12.961 |
| 星期六 | 48415.5 | 14.81 |
| 星期日 | 48182.22 | 14.738 |

这项技术的一个优点是，你可以更改表字面量中的序列编号，例如，如果更适合你，可以从星期三开始。

顺便说一下，如果你需要经常按工作日或类似内容排序，最好将数据保存在永久的查找表中。

### 使用表字面量作为查找表

我们已经注意到 SQLite 没有获取星期几名称的函数——只有日期编号。无论如何，你可能遇到其他情况，你所拥有的数据并不像你希望的那样友好或易于理解。

一个解决方案是使用表字面量作为查找表。

例如，`vip` 表有一个状态级别，值为 1、2 或 3。你应该意识到它分别代表金卡、银卡和铜卡。在这里，我们将使用表字面量来实现这一点。

#### 创建包含状态名称的 CTE

首先，我们将开发一个包含状态名称的 CTE：

```sql
--  PostgreSQL, MariaDB (非 MySQL), SQLite
WITH statuses(status,name) AS (
VALUES (1,'金卡'),(2,'银卡'),(3,'铜卡')
)
--  MySQL (非 MariaDB)
WITH statuses(status,name) AS (
VALUES row(1,'金卡'),row(2,'银卡'),row(3,'铜卡')
)
--  MSSQL
WITH statuses(status,name) AS (
SELECT * FROM (
VALUES (1,'金卡'),(2,'银卡'),(3,'铜卡')
) AS sq(a,b)
)
--  Oracle
WITH statuses(status,name) AS (
SELECT 1,'金卡' FROM DUAL
UNION ALL SELECT 2,'银卡' FROM DUAL
UNION ALL SELECT 3,'铜卡' FROM DUAL
)
```

#### 将 CTE 连接到 customers 和 vip 表

我们现在可以将 CTE 连接到 `customers` 和 `vip` 表：

```sql
WITH statuses(status,name) AS (
--  等等
)
SELECT *
FROM
customers
LEFT JOIN vip ON customers.id=vip.id
LEFT JOIN statuses ON vip.status=statuses.status
;
```

同样，好处在于你可以动态地更改状态名称。

你也可以对作者和客户的性别做类似的事情。使用这项技术，你还可以做的一件事是从一组名称翻译到另一组名称。

你可能想知道为什么我们不在表本身中包含性别或贵宾状态的全名。记住，你应该只记录一次数据，并且它应该是尽可能简单的版本。将值存储为单个字符（如性别）或整数（如贵宾状态），可以减少数据错误或差异的可能性，并且你可以在需要时将其拼写出来。


## 字符串分割技术详解

### 分割字符串

如果你有勇气去查看生成数据库的脚本，会在末尾找到两个递归 CTE：

```
--  填充流派表
INSERT INTO genres(genre)
WITH split(bookid,genre,rest,genres) AS (
...
)
SELECT DISTINCT genre
FROM split
WHERE split.genre IS NOT NULL;
--  填充书籍流派表
INSERT INTO bookgenres(bookid,genreid)
WITH split(bookid,genre,rest,genres) AS (
...
)
SELECT split.bookid,genres.id
FROM split JOIN genres ON split.genre=genres.genre
WHERE split.genre IS NOT NULL;
```

原因纯粹是出于实用考虑。存在数千个书籍-流派组合，与其直接导出`bookgenres`表，不如将组合的流派编码到一个表中，然后使用递归 CTE 来拆分，这样更为方便。将流派保持组合状态是错误的，原因正如第 3 章所讨论的那样。

接下来，我们将通过拆分几个示例字符串来了解这个过程是如何工作的。

首先，我们取一个简单的字符串并将其放入表字面量中。目前，它将是一个包含逗号分隔值的简单字符串。稍后，它会变得更复杂。

```
--  PostgreSQL, SQLite, MariaDB (非 MySQL)
WITH
cte(fruit) AS (
VALUES ('Apple,Banana,Cherry,Date,
Elderberry,Fig')
),
--  MySQL (非 MariaDB)
WITH
cte(fruit) AS (
VALUES row('Apple,Banana,Cherry,Date,
Elderberry,Fig')
),
--  MSSQL
WITH
cte(fruit) AS (
SELECT *
FROM (VALUES ('Apple,Banana,Cherry,Date,
Elderberry,Fig')) AS sq(a)
),
--  Oracle
WITH
cte(fruit) AS (
SELECT 'Apple,Banana,Cherry,Date,
Elderberry,Fig' FROM dual
),
```

为了使代码可读，字符串被拆分在两行上。*在你的实际代码中不要这样做！*

有些 DBMS 不喜欢内部有换行的字符串字面量。对于那些接受换行的情况，它将成为数据的一部分，而我们并不希望如此。

务必在一行上书写字符串，即使它很长。

对于递归 CTE，我们将构建两个值：单个项目和一个包含原始字符串剩余部分的字符串。该 CTE 可以称为`split`：

```
WITH
cte(fruit) AS (),
split(fruit, rest) AS (
)
```

锚点成员将从字符串中获取第一个项目（直到逗号），以及逗号之后剩余的部分：

```
WITH
cte(fruit) AS (),
--  PostgreSQL
split(fruit, rest) AS (
SELECT
substring(fruit,0,position(',' in fruits)),
substring(fruit,position(',' in fruits)+1)||','
FROM cte
)
--  MariaDB, MySQL
split(fruit, rest) AS (
SELECT
substring(fruit,1,position(',' in fruits)-1),
substring(fruit,position(',' in fruits)+1)||','
FROM cte
)
--  MSSQL
split(fruit, rest) AS (
SELECT
cast(substring(fruit,0,charindex(',',fruits)) as varchar(255)),
cast(substring(fruit,charindex(',',fruits)+1,255)+',' as varchar(255))
FROM cte
)
--  SQLite
split(fruit, rest) AS (
SELECT
substring(fruit,0,instr(fruits,',')),
substring(fruit,instr(fruits,',')+1)||','
FROM cte
)
--  Oracle
split(fruit, rest) AS (
SELECT
substr(fruit,1,instr(fruits,',')-1),
substr(fruit,instr(fruits,',')+1)||','
FROM cte
)
```

注意，对于`MSSQL`，我们不得不将计算转换为`varchar(255)`，这是由于字符串兼容性方面的一个特性。

对于递归成员，我们使用`rest`值。首先，我们获取到第一个逗号之前的字符串，成为`fruit`值。然后，我们获取从逗号开始的剩余字符串，成为`rest`的新值：

```
WITH
cte(fruit) AS (),
--  PostgreSQL
split(fruit, rest) AS (
SELECT ...
UNION
SELECT
substring(rest,0,position(',' in rest)),
substring(rest,position(',' in rest)+1)
FROM cte WHERE rest''
)
--  MariaDB, MySQL
split(fruit, rest) AS (
SELECT ...
UNION
SELECT
substring(rest,1,position(',' in rest)-1),
substring(rest,position(',' in rest)+1)
FROM cte WHERE rest''
)
--  MSSQL
split(fruit, rest) AS (
SELECT ...
UNION ALL
SELECT
substring(rest,0,charindex(',', rest)),
substring(rest,charindex(',', rest)+1,255)
FROM cte WHERE rest''
)
--  SQLite
split(fruit, rest) AS (
SELECT ...
UNION
SELECT
substring(rest,0,instr(rest,',')),
substring(rest,instr(rest,',')+1)
FROM cte WHERE rest''
)
--  Oracle
split(fruit, rest) AS (
SELECT ...
UNION ALL
SELECT
substr(rest,1,instr(rest,',')-1),
substr(rest,instr(rest,',')+1)
FROM cte WHERE rest''
)
```

注意，这次我们没有向`rest`值添加逗号：那只是为了启动递归。

我们还在`FROM`子句中添加了`WHERE rest<>''`。这是因为当字符串没有更多内容可供搜索时，我们需要停止递归。

你现在可以尝试一下：

```
WITH
cte(fruit) AS (),
split(fruit,rest) AS ()
SELECT * FROM split;
```

你现在应该看到以下内容：

| **水果** | **剩余部分** |
| --- | --- |
| Apple | Banana,Cherry,Date,Elderberry,Fig, |
| Banana | Cherry,Date,Elderberry,Fig, |
| Cherry | Date,Elderberry,Fig, |
| Date | Elderberry,Fig, |
| Elderberry | Fig, |
| Fig | [NULL] |

当然，我们不需要在输出中看到`rest`值：它只是为了让你看到它的处理过程。



### 拆分更复杂的数据

到目前为止，我们只拆分了一个简单的字符串。我们可以对更复杂的数据集执行相同的操作。这里，我们将使用一个包含三行两列的 CTE：

| **名称 (name)** | **列表 (list)** |
| --- | --- |
| colours | Red,Orange,Yellow,Green,Blue,Indigo,Violet |
| elements | Hydrogen,Helium,Lithium,Beryllium,Boron,Carbon |
| numbers | One,Two,Three,Four,Five,Six,Seven,Eight,Nine |

好消息是，这个过程几乎相同。

首先，我们将使用表字面量创建一个 CTE：

```sql
--  PostgreSQL, SQLite, MariaDB (非 MySQL)
WITH
cte(name, items) AS (
VALUES
('colours','Red,Orange,...,Indigo,Violet'),
('elements','Hydrogen,Helium,...,Carbon'),
('numbers','One,Two,...,Eight,Nine')
),
--  MySQL (非 MariaDB)
WITH
cte(name, items) AS (
VALUES
row('colours','Red,Orange,...,Indigo,Violet'),
row('elements','Hydrogen,Helium,...,Carbon'),
row('numbers','One,Two,...,Eight,Nine')
),
--  MSSQL
WITH
cte(name, items) AS (
SELECT *
FROM (
VALUES
('colours','Red,Orange,...,Indigo,Violet'),
('elements','Hydrogen,Helium,...,Carbon'),
('numbers','One,Two,...,Eight,Nine')
) AS sq(a,b)
),
--  Oracle
WITH
cte(name, items) AS (
SELECT 'colours','Red,Orange,...,Indigo,Violet'
FROM dual
UNION ALL SELECT 'elements','Hydrogen,...,Carbon'
FROM dual
UNION ALL SELECT 'numbers','One,Two,...,Eight,Nine'
FROM dual
),
```

（我们显然为了页面美观而缩写了列表。）

对于锚点成员，我们以与之前相同的方式开始，但会包含列表的 `name`。它不会参与拆分，但对输出很有用。

```sql
WITH
cte(name, items) AS (),
--  PostgreSQL
split(name, item, rest) AS (
SELECT
name,
substring(items,0,position(',' in items)),
substring(items,position(',' in items)+1)||','
FROM cte
)
--  MariaDB, MySQL
split(name, list, rest) AS (
SELECT
name,
substring(items,1,position(',' in items)-1),
substring(items,position(',' in items)+1)||','
FROM cte
)
--  MSSQL
split(name, list, rest) AS (
SELECT
name,
cast(substring(items,0,charindex(',', items)) as varchar(255)),
substring(items,charindex(',', items)+1,255)+','
FROM cte
)
--  SQLite
split(name, list, rest) AS (
SELECT
name,
substring(items,0,instr(items,',')),
substring(items,instr(items,',')+1)||','
FROM cte
)
--  Oracle
split(name, list, rest) AS (
SELECT
name,
substr(items,1,instr(items,',')-1),
substr(items,instr(items,',')+1)||','
FROM cte
)
```

至于递归成员，同样遵循相同的思路，并包含了 `name` 值：

```sql
WITH
cte(name, items) AS (),
--  PostgreSQL
split(name, list, rest) AS (
SELECT ...
UNION
SELECT
name,
substring(rest,0,position(',' in rest)),
substring(rest,position(',' in rest)+1)
FROM cte WHERE rest''
)
--  MariaDB, MySQL
split(name, list, rest) AS (
SELECT ...
UNION
SELECT
name,
substring(rest,1,position(',' in rest)-1),
substring(rest,position(',' in rest)+1)
FROM cte WHERE rest''
)
--  MSSQL
split(name, list, rest) AS (
SELECT ...
UNION ALL
SELECT
name,
cast(substring(rest,0,charindex(',', rest)) as varchar(255)),
substring(rest,charindex(',', rest)+1,255)
FROM cte WHERE rest''
)
--  SQLite
split(name, list, rest) AS (
SELECT ...
UNION
SELECT
name,
substring(rest,0,instr(rest,',')),
substring(rest,instr(rest,',')+1)
FROM cte WHERE rest''
)
--  Oracle
split(name, list, rest) AS (
SELECT ...
UNION ALL
SELECT
name,
substr(rest,1,instr(rest,',')-1),
substr(rest,instr(rest,',')+1)
FROM cte WHERE rest''
)
```

我们现在可以对此进行测试：

```sql
WITH
cte(name, items) AS ()
split(name, item, rest) AS ()
SELECT *
FROM split
ORDER BY name, item;
```

当一切顺利运行时，您应该会看到类似以下的结果：

| **名称 (name)** | **项目 (item)** | **剩余 (rest)** |
| --- | --- | --- |
| colours | Blue | Indigo,Violet, |
| colours | Green | Blue,Indigo,Violet, |
| colours | Indigo | Violet, |
| colours | Orange | Yellow,Green,Blue,Indigo,Violet, |
| colours | Red | Orange,Yellow,Green,Blue,Indigo,Violet, |
| colours | Violet | [NULL] |
| colours | Yellow | Green,Blue,Indigo,Violet, |
| elements | Beryllium | Boron,Carbon, |
| elements | Boron | Carbon, |
| elements | Carbon | [NULL] |
| elements | Helium | Lithium,Beryllium,Boron,Carbon, |
| elements | Hydrogen | Helium,Lithium,Beryllium,Boron,Carbon, |
| elements | Lithium | Beryllium,Boron,Carbon, |
| numbers | Eight | Nine, |
| numbers | Five | Six,Seven,Eight,Nine, |
| numbers | Four | Five,Six,Seven,Eight,Nine, |
| numbers | Nine | [NULL] |
| numbers | One | Two,Three,Four,Five,Six,Seven,Eight,Nine, |
| numbers | Seven | Eight,Nine, |
| numbers | Six | Seven,Eight,Nine, |
| numbers | Three | Four,Five,Six,Seven,Eight,Nine, |
| numbers | Two | Three,Four,Five,Six,Seven,Eight,Nine, |

如您所见，递归 CTE 能够处理多行数据。

