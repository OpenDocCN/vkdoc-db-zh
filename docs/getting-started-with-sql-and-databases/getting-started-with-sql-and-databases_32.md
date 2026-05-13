# GROUP BY 与 DISTINCT

当你初次接触 `DISTINCT` 子句时，它被描述为列出分组。你可以看到，实际上并未使用任何聚合函数。例如：

```sql
SELECT state, town
FROM customers
GROUP BY state, town
ORDER BY state, town;

SELECT DISTINCT state, town -- 结果相同
FROM customers
ORDER BY state, town;
```

显然，要获得分组，如果你 `don't` 想要任何额外的聚合，使用 `DISTINCT` 更简单。但是，如果你想要额外的聚合，那么就使用 `GROUP BY` 子句。永远没有理由同时使用两者。

## 多表分组

通常，你想要汇总的数据分布在多个表中。例如，我们可以获取按客户统计的总销售额，这首先需要汇总销售表：

```sql
SELECT
    customerid,
    count(*) AS number_of_sales,
    sum(total) AS total
FROM sales
GROUP BY customerid
ORDER BY total, customerid;
```

这将给你客户 ID 的销售数据：

| customerid | number_of_sales | total |
| --- | --- | --- |
| 440 | 1 | 115 |
| 444 | 1 | 200 |
| 461 | 1 | 240 |
| 526 | 1 | 285 |
| 567 | 1 | 310 |
| 575 | 1 | 310 |
| ~ 256 行 ~ | | |

注意，`total` 这个名字有双重含义。在 `sales` 表中，它指的是单笔销售总额。在上面的查询中，它被用作 `sum(total)` 的别名，其含义是一致的。由于 `ORDER BY` 是在 `SELECT` 之后处理的，因此排序的是汇总后的总金额。如果你觉得这有点混乱或误导，你可以使用另一个名字，比如 `customer_total`。

前面的查询会给我们 `customerid`，这可以，但如果我们有客户姓名和其他详细信息会更好。

一种方法是使用子查询来获取客户姓名，并将其用于聚合中：

```sql
SELECT
    customerid,
    (SELECT givenname||' '||familyname  -- MSSQL: 使用 +
     FROM customers
     WHERE customers.id = sales.customerid
    ) AS customer,
    count(*) AS number_of_sales,
    sum(total) AS total
FROM sales
GROUP BY customerid
ORDER BY total, customerid;
```

现在你有了更多细节：

| customerid | customer | number_of_sales | total |
| --- | --- | --- | --- |
| 440 | Percy Monn | 1 | 115 |
| 444 | Jo King | 1 | 200 |
| 461 | Carol Singer | 1 | 240 |
| 526 | Cliff Face | 1 | 285 |
| 567 | Perry Patetic | 1 | 310 |
| 575 | Gene Poole | 1 | 310 |
| ~ 256 行 ~ | | | |

你可能会预期，选择甚至不在同一个表中的列，更不用说在 `GROUP BY` 子句中，会导致错误。但是，请记住子查询与主聚合查询是分开的。

你也可以使用连接来获得相同的结果：

```sql
SELECT
    c.id, c.givenname||' '||c.familyname AS customer,
    count(*) AS number_of_sales, sum(s.total) AS total
FROM sales AS s JOIN customers AS c ON s.customerid=c.id
GROUP BY c.id, c.givenname||' '||c.familyname
ORDER BY total, customerid;
```

（记住 MSSQL 使用 `+` 进行连接，而 Oracle 不喜欢表 `AS`，后面的示例也是如此。）

这是一个更清晰的版本。然而，`GROUP BY` 有点笨拙，因为你不能使用 `customer` 别名（记住 `SELECT` 在之后才处理）。也存在一个微小的风险，即你可能尝试按略有不同的计算进行分组（这不会成功）。

如果你愿意，可以将连接包装在通用表表达式中，这将简化聚合查询：

```sql
WITH cte AS (
    SELECT
        c.id, c.givenname||' '||c.familyname AS customer,
        s.total
    FROM customers AS c JOIN sales AS s ON c.id=s.customerid
)
SELECT
    id, customer, count(*) AS number_of_sales,
    sum(total) AS total
FROM cte
GROUP BY id, customer
ORDER BY total, id;
```

这将给出相同的结果。总的来说，这比之前的版本稍长；但是，它确实通过在 CTE 中准备好数据来简化了主要的聚合查询。

## 冗余分组

正如我们之前提到的，聚合只能选择摘要或分组。当你试图选择比你分组依据更具信息性的内容时，这会导致一个小问题。有时，你需要进行比你预期更多的分组。

这是一个简单的例子：

```sql
SELECT state, count(*) AS countrows
FROM customers
GROUP BY state, state;
```

是的，我们按 `state` 分组了两次，是的，这有点浪费时间。对于每个州，我们想要一个州内的子分组，当然，只有一个。

然而，如果分组实际上是同一事物的不同变体，那就没那么傻了。例如，如果你想按星期几对销售额进行分组，你会希望有两个版本的日期：一个用于显示，一个用于排序。

具体做法如下：

```sql
-- PostgreSQL, Oracle
SELECT to_char(ordered,'FMDay') AS dayname,
    sum(total) AS total
FROM sales
GROUP BY to_char(ordered,'FMDay'), to_char(ordered,'D')
ORDER BY to_char(ordered,'D');

-- MySQL / MariaDB
SELECT date_format(ordered,'%W') AS dayname,
    sum(total) AS total
FROM sales
GROUP BY date_format(ordered,'%W'),
    date_format(ordered,'%w')
ORDER BY date_format(ordered,'%w');

-- MSSQL
SELECT datename(weekday,ordered) AS dayname,
    sum(total) AS total
FROM sales
GROUP BY datename(weekday,ordered),
    datepart(weekday,ordered)
ORDER BY datepart(weekday,ordered);

-- SQLite 不可用
```

记住，SQL 只允许你选择或排序 `GROUP BY` 子句中的内容。注意，`GROUP BY` 先按星期几的名称分组，然后按星期几的数字分组。从技术上讲，这是冗余的，因为每天只对应一个日期。但是，在 `GROUP BY` 中同时包含两者使你能够选择其一并按另一个排序。

| dayname | total |
| --- | --- |
| Sunday | 214285 |
| Monday | 214950 |
| Tuesday | 214090 |
| Wednesday | 211050 |
| Thursday | 224720 |
| Friday | 223640 |
| Saturday | 224315 |

如果你觉得这看起来像一个法律漏洞，嗯，你可能是对的。

你在上一节中看到了同样的情况，你按客户 ID 分组，然后按客户姓名分组。同样，这是冗余的（每个客户只能有一个姓名），但它允许你在 `SELECT` 子句中包含两者，尽管只有 `id` 是分组所必需的。

如果你对不必要的分组感到内疚，你可以使用另一种解决方法。这是前面按客户统计销售查询的替代版本：

```sql
WITH cte AS (
    SELECT c.id, c.givenname||' '||c.familyname AS customer,
        s.total
    FROM customers AS c JOIN sales AS s ON c.id=s.customerid
)
SELECT id, min(customer), count(*) AS number_of_sales, sum(total) AS total
FROM cte
GROUP BY id
ORDER BY total, id;
```

这次，我们只按 `id` 分组，但使用 `min()` 函数（或者如果你喜欢，也可以用 `max()`）；从技术上讲这是一个摘要，因此你可以安全地使用它。


## 准备用于聚合的数据

分组聚合仅当存在可按列分组的数据（即多行在某一列中具有完全相同的值）时才有效。有时，数据并不完全符合此要求，但您仍需对其进行聚合。

例如，假设您要计算每日销售总额。如果您查看 `sales` 表：

```sql
SELECT * FROM sales;
```

原始数据如下：

| id | customerid | total | ordered | shipped |
| --- | --- | --- | --- | --- |
| 52 | 52 | 940 | 2022-03-07 16:10:45.739071 | 2022-03-19 |
| 54 | 37 | 1005 | 2022-03-08 00:23:39.53316 | 2022-03-22 |
| 55 | 19 | 795 | 2022-03-08 06:23:28.387395 | 2022-03-19 |
| 57 | 42 | 505 | 2022-03-09 00:02:29.974004 | 2022-03-14 |
| 59 | 53 | 360 | 2022-03-09 06:26:24.808237 | 2022-03-17 |
| 60 | 10 | 340 | 2022-03-09 15:01:05.592177 | 2022-03-23 |
| ~ 2509 行 ~ |

您会发现每笔销售都有一个 `ordered` 值，其类型为 `datetime`。两笔交易在同一时刻发生几乎是不可能的（即使不是完全不可能），因此您尝试对此进行分组将一无所获。

但是，如果仅提取日期部分，您可以简化 `ordered` 列。最直接的方法是将数据转换为 `date` 类型，这将丢弃时间部分。对于汇总，您只需要日期和每笔销售的总额：

```sql
SELECT cast(ordered as date) as ordered, total FROM sales;
```

Oracle 的工作方式不太一样 —— 它不会移除时间部分，而这正是此操作的目的。

相反，您应该使用 `trunc(date)`。

简化后的数据如下：

| ordered | total |
| --- | --- |
| 2022-03-07 | 940 |
| 2022-03-08 | 1005 |
| 2022-03-08 | 795 |
| 2022-03-09 | 505 |
| 2022-03-09 | 360 |
| 2022-03-09 | 340 |
| ~ 2509 行 ~ |

对于 Oracle，您需要使用 `trunc(ordered)` 来实现相同的效果，尽管它会包含一个零时间。这里使用原名作为别名，因为它仍然起着标明销售下单时间的作用。

您可以将前面的语句视为第一步；然后可以在汇总中使用简化后的数据：

```sql
WITH data AS (-- Oracle: 使用 trunc(ordered)
SELECT cast(ordered as date) as ordered, total FROM sales
)
SELECT ordered, sum(total) AS total
FROM data
GROUP BY ordered;
```

现在您有了每日总额：

| ordered | total |
| --- | --- |
| 2022-10-10 | 3630 |
| 2022-07-14 | 5890 |
| 2022-09-22 | 6730 |
| 2022-05-19 | 1785 |
| 2023-02-25 | 8495 |
| 2022-05-23 | 6835 |
| ~ 385 行 ~ |

当然，结果可能无序，因此您可以在末尾添加

```sql
ORDER BY ordered
```

如果您有已排序的结果，可能会注意到日期中存在间隔。这没有问题：它仅表示那些日子没有已完成的销售。这种情况会发生。

这是另一个公共表表达式。您可以使用表子查询来完成相同的操作：

```sql
SELECT ordered, sum(total) AS total
FROM (
SELECT cast(ordered as date) as ordered, total FROM sales
) AS data
GROUP BY ordered;
```

这将实现相同的功能，但使用 CTE 会更容易处理：始终最好先准备数据，然后再使用它。CTE 也更灵活，如果需要，可以进行更复杂的编码。

### 在 CTE 中使用 CASE

CTE 在复杂计算中非常方便。例如，我们已经看到如何在 `CASE … END` 表达式中生成价格分组：

```sql
SELECT
id, title,
CASE
WHEN price < 130 THEN '便宜'
WHEN price <= 170 THEN '合理'
WHEN price IS NOT NULL THEN '昂贵'
ELSE '未定价'
END AS price_category
FROM paintings;
```

这再次给出了我们的价格类别：

| id | title | price_category |
| --- | --- | --- |
| 1222 | Haymakers Resting | 便宜 |
| 251 | Death in the Sickroom | 便宜 |
| 2190 | Cache-cache (Hide-and-Seek) | 昂贵 |
| 1560 | Indefinite Divisibility | 便宜 |
| 172 | Girl with Racket and Shuttlecock | 昂贵 |
| 2460 | The Procession to Calvary | 合理 |
| ~ 1273 行 ~ |

如果您想在 `GROUP BY` 表达式中使用此结果，您必须完全重复该表达式。这很繁琐、不可读、不灵活且容易出错。

相反，您可以使用 CTE 来汇总结果：

```sql
WITH cte AS (
SELECT
id, title,  -- 您实际上不需要此列
CASE
WHEN price < 130 THEN '便宜'
WHEN price <= 170 THEN '合理'
WHEN price IS NOT NULL THEN '昂贵'
ELSE '未定价'
END AS price_category
FROM paintings
)
SELECT price_category, count(*) AS countrows
FROM cte
GROUP BY price_category;
```

现在您有了价格类别的汇总：

| price_category | countrows |
| --- | --- |
| 合理 | 489 |
| 便宜 | 345 |
| 未定价 | 136 |
| 昂贵 | 303 |

如您所见，您实际上不需要 id 和 title，因为您只使用分组并进行计数。

### 在 CTE 中使用连接

您还可以在 CTE 中使用连接，这简化了多表汇总。例如，如果您想获取各州的销售额，您将需要来自 `sales` 表的总额和来自 `customers` 表的州信息。

要仅获取州和总额，我们可以使用

```sql
SELECT c.state, s.total
FROM customers AS c JOIN sales AS s ON c.id=s.customerid;
```

简化后的结果如下：

| state | total |
| --- | --- |
| QLD | 940 |
| SA | 1005 |
| WA | 795 |
| VIC | 505 |
| TAS | 360 |
| NSW | 340 |
| ~ 2509 行 ~ |   |

将其用作 CTE，我们可以按州分组：

```sql
WITH cte AS (
SELECT c.state, s.total
FROM customers AS c JOIN sales AS s ON c.id=s.customerid
)
SELECT state, sum(total) AS total
FROM cte
GROUP BY state;
```

我们现在有了各州总额：

| state | total |
| --- | --- |
| TAS | 156305 |
| VIC | 272695 |
| NSW | 374660 |
| NT | 18855 |
| QLD | 333575 |
| SA | 115700 |
| WA | 255260 |

再次说明，CTE 的目的是在汇总数据之前对其进行准备。

