# 7. 聚合数据

没人想看一百万行数据。你真正想要的是：

*   *部分*数据
*   数据的*摘要*

获取*部分*数据通常是使用 `WHERE` 子句进行筛选的问题，正如你之前所见。而*汇总*数据则是通过一个或多个`聚合`函数来运行数据，这些函数会对整个数据集或更小的分组进行汇总。

最简单的聚合函数非常直观：

```sql
SELECT count(*)
FROM customers;
```

`count(*)` 函数统计所有行。当然，功能远不止于此。除此之外，你还可以选择具体统计什么。

在本章中，你将了解一些更常见的聚合函数，例如用于汇总数字的 `sum()` 和 `avg()`，或用于查找任意范围端点的 `max()` 和 `min()`。

不过，首先我们将看看聚合在 SQL 中如何工作，以及你能用它们做什么、不能做什么。然后，我们将探讨如何对数据的不同部分进行聚合。你还将看到聚合如何用于筛选数据。

聚合可以应用于全部数据，也可以按分组应用（如小计）。我们将了解如何生成分组摘要以及如何筛选分组摘要。

我们还将探讨如何跨多个表进行汇总，以及如何使用虚拟表来生成更合适的摘要。

## 数据计数

前面的 `count()` 函数是一个特例：`count(*)` 统计数据集中的所有*行*。数据集也可以被筛选：

```sql
SELECT count(*) AS countrows
FROM customers
WHERE height<160.5;
```

在这种情况下，你将统计符合该条件的行数。

| countrows |
| --- |
| 20 |

与 `SELECT *` 表达式不同，这里的星号意味着*所有行*，而不是*所有列*。

### 统计值

除了统计行数，你还可以统计某列中的值的数量：

```sql
SELECT
count(*) as countrows,
count(id) as ids,                   -- 相同
count(email) as emails,             -- 再次相同
count(familyname) as familynames,   -- 再次相同
count(phone) as phones,
count(state) as states
FROM customers;
```

这将给你以下摘要：

| rows | ids | emails | familynames | phones | states |
| --- | --- | --- | --- | --- | --- |
| 304 | 304 | 304 | 304 | 268 | 269 |

请注意，行数将与 `id` 数、`email` 数和 `familyname` 数相同。然而，对于 `phone` 和 `state` 则*不*相同。在统计值时，请注意：

*   每行*必须*有一个主键（此处是 `id`），所以主键列中的值数量与行数相同。
*   在这组数据中，电子邮件地址和姓氏是必填的（在 SQL 中设置为 `NOT NULL`），所以 `email` 列中的值数量也与行数相同。
*   并非所有客户都有（记录的）电话号码，所以这个数字小于行数。
*   `state` 的情况类似，但在解释上会有进一步的复杂性，我们稍后会讨论。

除了统计行数，`count(…)` 表示统计列中*值*的数量。由于 `NULL` 不是值，`count()` 会跳过它们，只统计存在的值。这就是为什么 `phone` 和 `state` 以及其他一些列的计数小于行数。

事实上，*所有*聚合函数都会跳过 `NULL`，你将会看到。

## 聚合如何工作

根据前面的例子，你会注意到结果与所有 `SELECT` 语句一样，出现在一个虚拟表中。这个虚拟表始终只有一行。

要理解聚合过程，想象有*两个*表：原始的（可能是虚拟的）表和一个摘要表。每当 SQL 看到聚合查询时，就会生成摘要表。

你可以让汇总过程更加明确：

```sql
SELECT
count(*) as countrows,
count(phone) as phones,
count(dob) as dobs
FROM customers
GROUP BY ()     -- 仅限 PostgreSQL, MSSQL, Oracle
--  SELECT
;
```

`GROUP BY ()` 子句意味着原始表（`customers`）将被汇总。这实际上并没有改变什么，因为每当 SQL 遇到聚合函数（如 `count()`）时，它都隐含此意。事实上，一些数据库管理系统甚至不允许你添加它；这没什么大不了的，因为无论如何汇总都会发生。

注意子句的顺序。`SELECT` 是在 `ORDER BY` 之前最后处理的，这里将其作为注释包含在内以提醒你。

在这个摘要表中，你可以访问任何可想象的*摘要*；到目前为止只有 `count()`，但我们很快会看到更多。你*无法*访问未汇总的数据。

例如，这个查询注定会失败：

```sql
SELECT
id, givenname, familyname,      -- 等等，这无关紧要
count(*) as countrows,
count(phone) as phones,
count(dob) as dobs
FROM customers
--  GROUP BY ()
;
```

即使 `GROUP BY ()` 被注释掉了，表也正在被汇总，`id`、`givenname`、`familyname` 和其他单个值不再可访问。脚注:[⁷]

这里的道理是，你不能在同一个查询中既选择摘要又选择未汇总的值。

稍后，当更严肃地使用 `GROUP BY` 子句时，你会看到更多相关内容。


## 有选择地计数

大多数画作都有`price`（价格）值。你可以用以下方式计数：

```sql
SELECT count(price) FROM paintings;
```

这会返回类似以下结果：

| count |
| --- |
| 1137 |

有时，你可能只想计数其中的一部分，例如较便宜或较贵的画作。

你可以尝试：

```sql
SELECT count(price)
FROM paintings
WHERE price<130;
```

你现在得到了筛选后的汇总表：

| count |
| --- |
| 345 |

这样做可以，但你无法同时包含其他汇总；你需要单独的查询。

你想要的是类似这样的：

```sql
SELECT count(price<130) --  不能按预期工作
FROM paintings;
```

但它要么不工作，要么得不到预期结果，具体取决于你的数据库管理系统。

现代 SQL 允许使用以下形式的聚合筛选器：

```sql
--  仅限 PostgreSQL 和 SQLite
SELECT count(price) FILTER (WHERE price<130)
FROM paintings;
```

这会给出一个真正的筛选汇总：

| count |
| --- |
| 345 |

然而，它（目前）并未被广泛支持。

或者，你可以利用`CASE … END`表达式。记住，你可以使用`CASE … END`来生成类别。例如：

```sql
SELECT
id, title, price,
CASE
WHEN price<130 THEN 'inexpensive'
WHEN price<170 THEN 'reasonable'
WHEN price>=170 THEN 'prestige'
--  否则为 NULL
END as pricegroup
FROM paintings;
```

结果如下：

| id | title | price | pricegroup |
| --- | --- | --- | --- |
| 1222 | Haymakers Resting | 125 | inexpensive |
| 251 | Death in the Sickroom | 105 | inexpensive |
| 2190 | Cache-cache (Hide-and-Seek) | 185 | prestige |
| 1560 | Indefinite Divisibility | 125 | inexpensive |
| 172 | Girl with Racket and Shuttlecock | 195 | prestige |
| 2460 | The Procession to Calvary | 165 | reasonable |
| ~ 1273 行 ~ |

特别地，你可以使用一个简单版本来仅高亮便宜的画作：

```sql
SELECT
id, title, price,
CASE WHEN price<130 THEN 'cheap' END AS status
FROM paintings;
```

这仅高亮便宜的画作：

| id | title | price | status |
| --- | --- | --- | --- |
| 1222 | Haymakers Resting | 125 | cheap |
| 251 | Death in the Sickroom | 105 | cheap |
| 2190 | Cache-cache (Hide-and-Seek) | 185 |   |
| 1560 | Indefinite Divisibility | 125 | cheap |
| 172 | Girl with Racket and Shuttlecock | 195 |   |
| 2460 | The Procession to Calvary | 165 |   |
| ~ 1273 行 ~ |

记住，默认值总是`NULL`，除非你包含一个`ELSE`子句来使其为其他值。

`cheap`这个词并不重要。你也可以用`not so terribly expensive`（不太贵）。如果你只想计数它们，你甚至可以使用`orang utan`（猩猩）或`23`，因为计数不关心实际值是什么。传统上使用值`1`。

现在，如果我们想计数便宜的画作，我们使用`CASE … END`表达式将值设置为`1`（或任何我们想要的任意值）并计数结果：

```sql
SELECT
count(CASE WHEN price<130 THEN 1 END) AS cheap
FROM paintings;
```

这应该适用于所有数据库管理系统：

| cheap |
| --- |
| 345 |

记住，不匹配的将是`NULL`，而聚合函数总是跳过`NULL`，因此它们不会被计数。

同时，你也可以包含更昂贵的画作：

```sql
SELECT
count(CASE WHEN price<130 THEN 1 END) AS cheap,
count(CASE WHEN price>=130 AND price<170 THEN 1 END) AS reasonable,
count(CASE WHEN price>=170 THEN 1 END) AS expensive
FROM paintings;
```

你现在得到了所有类别的汇总：

| cheap | reasonable | expensive |
| --- | --- | --- |
| 345 | 489 | 359 |

如果你只想要便宜的，原来的`WHERE price<130`子句就可以。然而，当你还需要其他汇总时，你需要筛选聚合，而不是筛选表。这就是这里发生的情况。

类似地，你可以计数`spam`列中的结果。`spam`列表示客户是否同意接收通讯。在某些情况下，该列有`NULL`值，有些人会认为这意味着同意，而另一些人则认为相反。^(⁸)

`spam`列包含`true`或`false`值（如果数据库管理系统支持），或者包含`1`或`0`（如果不支持）。`true`或`false`值在数学家乔治·布尔之后被称为**布尔**值。

要计数不同的值，你可以使用：

```sql
--  PostgreSQL, MySQL/MariaDB, SQLite
SELECT
count(*) AS total,
count(spam) AS known,
count(CASE spam WHEN true THEN 1 END) AS yes,
count(CASE spam WHEN false THEN 1 END) AS no
FROM customers;
--  MSSQL, Oracle
SELECT
count(*) AS total,
count(spam) AS known,
count(CASE spam WHEN 1 THEN 1 END) AS yes,
count(CASE spam WHEN 0 THEN 1 END) AS no
FROM customers;
```

你现在得到以下结果：

| total | known | yes | no |
| --- | --- | --- | --- |
| 304 | 279 | 106 | 173 |

`count(spam)`的结果表示有多少行具有值，但不表示是真还是假。注意，`count(spam)`的值将是`yes`和`no`的总和。

如果你特别想计数某列的`NULL`值，你可以这样做：

```sql
SELECT count(*)-count(spam) AS nulls
FROM customers;
```

这会给出类似结果：

| nulls |
| --- |
| 25 |

即，总行数减去该列中的非空值数量，就得到了`NULL`的数量。

或者，你可以通过使用`CASE … END`使`NULL`变成某个值，而其他所有值变成`NULL`来计数`NULL`：

```sql
SELECT
count(*) AS total,
--  等等,
count(CASE WHEN spam IS NULL THEN 1 END) AS unknown
FROM customers;
```

你会得到相同的结果：

| total | unknown |
| --- | --- |
| 304 | 25 |

这里，我们实际上是在反转`NULL`：如果是`NULL`，就使其为某个值；如果不是`NULL`，就使其为`NULL`。

## 不同值

`customers`表中有多少个州？你可以尝试：

```sql
SELECT count(state) AS states
FROM customers;
```

对于州的数量，该语句正确地指出了数据集中州值的数量：

| states |
| --- |
| 269 |

然而，如果你想回答“customers 表中有多少个州？”这个问题，它可能实际上意味着“有多少个*不同的*州……？”而没有明确说出来。如果这看起来很有可能，你可以使用`DISTINCT`关键字：

```sql
--  列出不同的州：
SELECT DISTINCT state
FROM customers;
--  计数不同的州：
SELECT
count(state) AS addresses,
count(DISTINCT state) AS states
FROM customers;
```

你现在得到以下结果：

| addresses | states |
| --- | --- |
| 269 | 7 |

这是许多例子中的一个，我们在英语中说的并不总是代码中我们想要的意思。

正如你之前看到的，计数州值仍然有一些意义。鉴于每个地址都包含一个州，计数（非唯一的）州意味着计数记录的地址数量。



## 数字汇总

在处理数字时，还有一些额外的聚合函数非常有用：

```
SELECT
count(height) as heights,
sum(height) AS total,
avg(height) AS average,
sum(height)/count(height) AS computed_average
FROM customers;
```

这会给出一些统计结果：

| heights | total | average | computed_average |
| --- | --- | --- | --- |
| 248 | 42242.3 | 170.332 | 170.332 |

`sum()` 函数将列中的所有值相加，而 `avg()` 函数计算这些值的平均值。在上面的例子中，平均值也通过 `sum/count` 公式计算，你可以看到结果是一样的；原则上，`avg()` 函数只是为了方便。

请注意，前面提到的所有聚合函数都只使用实际值，并且，`NULL` 值总是会被跳过。对于 `sum()` 你可能不会看到任何差异，但需要特别注意的是，平均值仅根据值的数量计算，而不是行数。如果你曾愚蠢地为缺失的身高输入了 `0`，这些 `0` 会被包含在内，并大幅拉低平均值。

你可以通过除以总行数或者将 `NULL` 值合并为 `0` 来看到这个错误导致的结果：

```
SELECT
count(height) as heights,
sum(height) AS total,
avg(height) AS average,
sum(height)/count(height) AS computed_average,
sum(height)/count(*) AS not_ca,
avg(coalesce(height,0)) AS not_ca_again
FROM customers;
```

现在我们会得到一些具有误导性的值：

| heights | total | average | computed_average | not_ca | not_ca_again |
| --- | --- | --- | --- | --- | --- |
| 248 | 42242.3 | 170.332 | 170.332 | 138.955 | 138.955 |

在上面的查询中：

*   表达式 `sum(height)/count(*)` 用总身高除以行数，而不是身高值的数量，因此结果会低得多。
*   表达式 `coalesce(height,0)` 将 `NULL` 值替换为 `0`；这些零会被加进去，本身没有影响，但同样，你除以的数又变多了。

如果你对统计感兴趣，还可以计算标准差：

```
--  非 SQLite 语法
SELECT
count(height) as heights,
sum(height) AS total,
avg(height) AS average,
stddev(height) AS sd    --  MSSQL 语法: stdev(height)
FROM customers;
```

现在我们有了更全面的统计数据。

| heights | total | average | sd |
| --- | --- | --- | --- |
| 248 | 42242.3 | 170.332 | 6.926 |

请注意，如果你要聚合的数据不是数字，这些函数都将无法工作。

### 错误示例

有些事情，即使 SQL 允许，你也*绝不*应该用聚合函数去做。

下面这条语句是可以运行的：

```
--  不要这样做：
SELECT
sum(id) AS total_id,
sum(year) AS total_year,
sum(price) AS total_price
FROM paintings;
```

你会得到一个结果，但你可能会后悔：

| total_id | total_year | total_price |
| --- | --- | --- |
| 1619593 | 1906235 | 168995 |

SQL 在处理数字方面有些天真。前面相加的三个列都是数字，SQL 会按照指令愉快地将它们相加。然而，有些数字绝不应该以这种方式处理。

首先，有些数字并不是真正作为数字来使用的。真正数字最重要的特性是它能*计数*。客户的 `height` 以厘米为单位计数，销售总额以美元计数。

`id` 是一个数字，但它不是用来计数的。相反，它用于*排序*，任何其他序列，比如字母表，都可能起到同样的作用。显然，将 `id` 相加没有意义，原因是你不应该对序列进行相加。

`year` 也被用作一个序列。它确实计算了公元（BC）以来的年数，但在这里并非如此使用，因此相加年份同样没有意义。

`price` 则更为微妙。它确实用于计数美元的数量，但它实际上并不代表一个固定值。相反，它是*每份*的美元数量。实际上，它是一个平均值，你需要将它乘以份数，才能得到一个可以相加的固定值。

`price` 是一个名称可能产生误导的例子。也许为了更清晰地表达，它应该被命名为 `price_per_copy`。另一方面，`price` 是一个更简单的名称。关键在于，你经常会遇到需要你自行理解或变通处理的表或列名示例。

### 测量尺度

统计学家有时会谈论数字的不同用途。从技术上讲，它们都是数字，但它们的用途会影响你能对它们做什么操作。

以下类型通常被称为 `测量尺度`：

*   名义尺度：名义数字仅用作代码，没有任何数字意义。例如：
    *   电话“号码”只是代码，它们实际上并不测量任何东西。
    *   你可能为一个类别分配一个数字，比如艺术品类型，但无论是数值还是排序顺序都没有意义。

*   顺序尺度：顺序数字可以排序，但数值的实际大小没有意义。例如：
    *   排名，其中位置很重要，但数值之间的距离不重要
    *   量表，比如从好到坏的意见评级

在示例数据库中用作主键的 `id`，将属于名义尺度或顺序尺度，这取决于你是否认为顺序有意义。

*   区间尺度：数字之间的差异有意义，但没有固定的零点。例如：
    *   距离一个固定点的距离（该点是任意选定的）
    *   事件发生的年份（该年份是从不存在的公元 0 年开始任意测量的）
    注意，*相加*这些数字没有意义，但你可以*相减*它们来得到一个区间。

*   比率尺度：对于这类数字，有一个真正的零点，因此大小有意义。例如：
    *   一次销售的总成本
    *   一个人的身高

只有最后一种类型的数字可以相加。



## 聚合计算数据

`saleitems` 表中包含一些值得汇总的数据：

```sql
SELECT * FROM saleitems;
```

以下是原始数据：

| id | saleid | paintingid | quantity | price |
| --- | --- | --- | --- | --- |
| 2621 | 1066 | 1065 | 3 | 100 |
| 5169 | 2067 | 870 | 1 | 155 |
| 667 | 271 | 2061 | 1 | 165 |
| 6905 | 2749 | 1796 | 3 | 115 |
| 886 | 361 | 1874 | 1 | 140 |
| 6729 | 2681 | 1516 | 2 | 160 |
| ~ 6315 rows ~ |

这包含了一个 `quantity` 列，表示每件商品的销售数量，以及一个 `price` 列，表示每件商品的 *单价*。

你可以对这些列进行求和，但需要确保这种操作是有意义的：

*   `quantity` 列代表了实际售出的副本数量，因此将其相加是有意义的：它表示需要打印、包装或进行其他处理的印刷品总数。
*   `price` 列是单价，因此仅仅将其相加是 *没有* 意义的，原因在前面提到的错误示例中已经讨论过。
*   然而，你可以将价格乘以数量，然后再将它们相加。

考虑到这一点，以下查询应该是有效的：

```sql
SELECT
sum(quantity) AS total_copies,
sum(quantity*price) AS total_value
FROM saleitems;
```

汇总结果如下：

| total_copies | total_value |
| --- | --- |
| 9722 | 1450390 |

它应该有效，但这里有个问题。回想一下，有些数量是缺失的，因此你需要用 `coalesce()` 函数将 `NULL` 值处理为假定的含义 `1`。在汇总时你也必须做同样的处理。否则，结果会偏低：

```sql
SELECT
sum(coalesce(quantity,1)) AS total_copies,
sum(coalesce(quantity,1)*price) AS total_value
FROM saleitems;
```

这会给你一个更真实的结果：

| total_copies | total_value |
| --- | --- |
| 10237 | 1527050 |

顺便说一下，前面的 `total_value` 应该与你从下面查询得到的结果相同：

```sql
SELECT sum(total) FROM sales;
```

你应该得到相同的总价值：

| sum |
| --- |
| 1527050 |

显然，`coalesce()` 函数被用于计算销售总额。

## 其他聚合函数

到目前为止，你已经进行了计数，并且对于数字运行了一些统计函数。还有其他可用的函数，但这里我们将只关注两个：`max()` 和 `min()` 函数。

与 `count()` 函数一样，你可以在所有主要数据类型上使用 `max()` 和 `min()`。例如：

```sql
SELECT
min(height) as shortest, max(height) as tallest,
min(dob) as oldest, max(dob) as youngest,
min(familyname) as first, max(familyname) as last
FROM customers;
```

你将得到这些极值：

| shortest | tallest | oldest | youngest | first | last |
| --- | --- | --- | --- | --- | --- |
| 150.3 | 186.3 | 1962-09-24 | 2002-08-05 | Abet | Yourbusiness |

`max()` 和 `min()` 函数为你提供类似于 `ORDER BY` 子句排序后处于两端的值，不过像往常一样，它们会忽略可能出现的 `NULL`。

与排序类似，数据类型将影响哪些值位于开头或结尾。例如：

```sql
SELECT
count(*) AS countrows,
max(numbervalue) AS most, min(numbervalue) AS least,
max(datevalue) AS latest, min(datevalue) AS earliest,
max(stringvalue) AS last, min(stringvalue) AS first
FROM sorting;
```

值是否为字符串将影响结果：

| rows | most | least | latest | earliest | last | first |
| --- | --- | --- | --- | --- | --- | --- |
| 8 | 1024 | -8 | 1917-06-30 | 1775-12-16 | Date | apple |

在得到最小值和最大值之后，你可能想问哪些行实际匹配这些值。例如，谁是最年长的客户，或者哪幅画作最受欢迎？

## 使用聚合结果作为过滤器

你已经看到，不能将未汇总的数据与汇总的数据混合使用。但是，你可以在子查询中使用汇总结果来过滤未汇总的数据。这基本上是将查询分两步进行。

例如，谁是最年长的客户？要得到答案：

1.  找到最小的出生日期：

    ```sql
    SELECT min(dob) FROM customers
    ```

    这将作为下一步的子查询。（注意这里还没有分号，因为我们还没有完成；你仍然可以单独运行这个查询。）

2.  找到出生日期与该结果匹配的客户：

    ```sql
    SELECT * FROM customers
    WHERE dob=(SELECT min(dob) FROM customers);
    ```

    这将给你类似这样的结果：

    | id | email | … | dob | … |
    | --- | --- | --- | --- | --- |
    | 545 | jack.knife545@example.com | … | 1962-09-24 | … |
    | 344 | rose.boat344@example.net | … | 1962-09-24 | … |

    你会注意到有不止一个匹配项。这就是为什么期望将未汇总的数据与汇总的数据混合使用是无意义的原因之一。

你可以使用相同的技术来找到最年轻、最高或最矮的客户。

有时，你需要一个涉及关联表的查询。例如，假设你想找到单笔销售额最大的客户。

首先，从 `sales` 表中获取 `customerid`：

```sql
SELECT customerid FROM sales WHERE total=(
SELECT max(total) FROM sales
);
```

这里，我们使用 `max(total)` 并找到匹配的 `customerid`。当然，可能不止一个。

接下来，我们从 `customers` 表中找到匹配的客户：

```sql
--  Biggest Spender
SELECT *
FROM customers
WHERE id IN (
SELECT customerid FROM sales WHERE total=(
SELECT max(total) FROM sales
)
);
```

你现在得到了客户详细信息：

| id | email | familyname | givenname | … |
| --- | --- | --- | --- | --- |
| 19 | millie.pede19@example.net | Pede | Millie | … |

对于这个例子，你本可以使用：

```sql
WHERE id=(SELECT …)
```

然而，这样做是有风险的，因为如果有多个客户的 `max(total)` 相同，可能会出现多个结果。在那种情况下，你会得到一个错误。

你也可以使用其他统计函数作为过滤器。例如，如果你想找到身高低于平均值的客户：

```sql
SELECT * FROM customers
WHERE height<(SELECT avg(height) FROM customers);
```

你会得到较矮的客户：

| id | email | familyname | … | height |
| --- | --- | --- | --- | --- |
| 186 | ray.gunn186@example.net | Gunn | … | 163.8 |
| 179 | ivan.inkling179@example.com | Inkling | … | 170.3 |
| 523 | seymour.sights523@example.net | Sights | … | 167.3 |
| 351 | dick.tate351@example.com | Tate | … | 167.8 |
| 422 | wanda.why422@example.com | Why | … | 163.2 |
| 121 | lil.ting121@example.com | Ting | … | 162.8 |
| ~ 128 rows ~ |

如果你想要那些 *显著* 矮于平均的人，你也可以使用标准差：

```sql
SELECT * FROM customers
WHERE height<(SELECT avg(height)-stddev(height) FROM customers);
--  MSSQL: Use stdev(height)
```

你现在会得到矮得多的客户：

| id | email | familyname | … | height |
| --- | --- | --- | --- | --- |
| 422 | wanda.why422@example.com | Why | … | 163.2 |
| 121 | lil.ting121@example.com | Ting | … | 162.8 |
| 429 | tom.morrow429@example.com | Morrow | … | 156.9 |
| 138 | al.fresco138@example.net | Fresco | … | 162.6 |
| 468 | connie.fer468@example.com | Fer | … | 161.3 |
| 330 | clara.fied330@example.com | Fied | … | 156.6 |
| ~ 40 rows ~ |

像往常一样，你不能在单个 `SELECT` 语句中混合使用聚合值和非聚合值。但是，你可以在子查询中使用聚合，并在非聚合查询中使用其结果。



## 分组

正如你已经看到的，你可以使用 `WHERE` 子句来过滤结果：

```sql
SELECT count(*) AS countrows
FROM customers
WHERE state='VIC';
```

你将得到以下结果：

| countrows |
| --- |
| 52 |

如果需要，你可以对另一个过滤值执行相同的操作：

```sql
SELECT count(*) AS countrows FROM customers WHERE state='VIC';
SELECT count(*) AS countrows FROM customers WHERE state='NSW';
SELECT count(*) AS countrows FROM customers WHERE state='QLD';
```

你刚才手动完成的操作，实际上是在多个组中运行了 `count()` 函数。

你可以使用 `UNION` 操作符将结果合并为一个结果集：

```sql
SELECT count(*) AS countrows FROM customers WHERE state='VIC'
UNION ALL
SELECT count(*) AS countrows FROM customers WHERE state='NSW'
UNION ALL
SELECT count(*) AS countrows FROM customers WHERE state='QLD';
```

`UNION ALL` 子句可用于合并多个 `SELECT` 语句的结果。请注意，语句之间没有分号，因为它们都属于同一个语句。稍后你会看到更多关于 `UNION` 子句的内容。

这个结果的信息量稍微少了一些，因为没有任何标识来区分不同的行。你可以硬编码一个值来做到这一点：

```sql
SELECT 'vic' AS state, count(*) AS countrows
FROM customers WHERE state='VIC'
UNION ALL
SELECT 'nsw' AS state, count(*) AS countrows
FROM customers WHERE state='NSW'
UNION ALL
SELECT 'qld' AS state, count(*) AS countrows
FROM customers WHERE state='QLD';
```

这样可读性更高：

| state | countrows |
| --- | --- |
| vic | 52 |
| nsw | 67 |
| qld | 52 |

现在你也有了州名。然而，这相当繁琐，一定有更好的方法来做到这一点，当然，确实有。

### 使用 GROUP BY 子句

为不同组分别运行聚合函数的更好方法是让 SQL 通过 `GROUP BY` 子句来完成这项繁重的工作：

```sql
SELECT count(*) AS countrows
FROM customers
GROUP BY state;
```

这更方便，但正如你所见，还不完全正确：

| countrows |
| --- |
| 47 |
| 35 |
| 26 |
| 52 |
| 67 |
| 3 |
| 52 |
| 22 |

这次，你将得到多行结果：每个不同的州对应一行。或者我们这样认为。问题是，上次你知道自己正在过滤哪一个州，但现在你有多个结果，却无法识别哪个是哪个。

当你只使用聚合函数时，实际上隐含使用了之前提到的 `GROUP BY ()` 子句。它为我们提供 `总计`：整个（过滤后）表的汇总，并且只有一行。并非所有的数据库管理系统都支持它，而且它`从不`是必需的。

另一方面，`GROUP BY something` 则不同。它生成 `小计`：多行分组汇总以及组名。它受到所有数据库管理系统的支持。

如果你想要识别结果，就需要在 `SELECT` 子句中包含组名：

```sql
SELECT state, count(*) AS countrows
FROM customers
GROUP BY state;
```

这会得到类似这样的结果：

| state | countrows |
| --- | --- |
| WA | 47 |
|   | 35 |
| TAS | 26 |
| VIC | 52 |
| NSW | 67 |
| NT | 3 |
| QLD | 52 |
| SA | 22 |

你也可以按多个列进行分组：

```sql
SELECT state, town, count(*) AS countrows
FROM customers
GROUP BY state, town;
```

你现在得到了更细粒度的分组：

| state | town | countrows |
| --- | --- | --- |
| SA | Windsor | 1 |
|   |   | 35 |
| VIC | Belmont | 2 |
| SA | Alberton | 3 |
| NSW | Hamilton | 5 |
| WA | Wattle Grove | 4 |
| ~ 79 行 ~ |   |   |

按多个列分组会在组内创建子组。在这个例子中，你的主要分组是州，而子分组是城镇。

对于某些数据库管理系统，你可能会看到结果按某种顺序排列，而在其他系统中，尤其是 PostgreSQL 和 Oracle，则不会。在 MySQL/MariaDB 和 SQLite 中，结果将按 `state,town` 排序，而在 MSSQL 中，结果只会按 `town` 排序。

一如既往，除非你使用 `ORDER BY` 子句，否则永远不要依赖结果的顺序！SQL 并不要求 `GROUP BY` 进行任何排序，你看到的任何排序可能是数据库管理系统内部如何进行分组的副产品。如果你执行更复杂的操作，顺序很可能又会改变。

因此，要完成这项工作，请添加一个 `ORDER BY` 子句：

```sql
SELECT state, town, count(*) AS countrows
FROM customers
GROUP BY state, town
ORDER BY state, town;
```

结果可读性更强：

| state | town | countrows |
| --- | --- | --- |
| NSW | Bald Hills | 6 |
| NSW | Belmont | 4 |
| NSW | Broadwater | 5 |
| NSW | Buchanan | 3 |
| NSW | Darlington | 1 |
| NSW | Glenroy | 2 |
| ~ 79 行 ~ |   |   |

请记住，结果中可能有一些 `NULL` 值，并且根据数据库管理系统的不同，你可能会在开头或结尾看到一个 `NULL` 组。

你可能认为 SQL 应该能推断出你希望按这种方式排序，但这要求 SQL 开始猜测你的意图，而 `ORDER BY` 对于并非它实际被要求做的事情来说，工作量太大了。

然而，在绝大多数情况下（9.5 次中的 9.5 次），你可能都会希望这样做：

*   在 `SELECT` 子句中包含分组列
*   `ORDER BY` 的列与 `GROUP BY` 相同，或类似

你不一定需要完全按照 `GROUP BY` 的列来排序。在后面关于冗余分组的部分，你会看到如何按与分组类似的东西进行排序。不过，这个原则是成立的。

从 SQL 的角度来看，你可以按任意顺序进行 `GROUP BY`：

```sql
SELECT state, town, count(*) AS countrows
FROM customers
GROUP BY town, state
ORDER BY state, town;
```

你会得到相同的行，并且 `ORDER BY` 子句会负责排序。

这是因为这些列本应是独立的，SQL 当然不知道它们之间的任何关系。然而，在这种情况下，我们理解一个州是包含多个城镇的容器，因此先按州分组再按城镇分组在逻辑上更合理。

对于 `ORDER BY` 同样如此，我们可以按任何我们喜欢的列排序。然而，按与分组相同的列排序是最合理的。

有一个细微的变化。当我们写一个地址时，通常从小到大写，所以你可以更改 `SELECT` 子句中的列顺序：

```sql
SELECT town, state, count(*) AS countrows
FROM customers
GROUP BY state, town
ORDER BY state, town;
```

这会得到相同的结果，但列的顺序不同：

| town | state | countrows |
| --- | --- | --- |
| Bald Hills | NSW | 6 |
| Belmont | NSW | 4 |
| Broadwater | NSW | 5 |
| Buchanan | NSW | 3 |
| Darlington | NSW | 1 |
| Glenroy | NSW | 2 |
| ~ 79 行 ~ |   |   |

请记住，`SELECT` 子句中的列顺序无关紧要。

你可以使用相同的技术来获取每个客户的销售数量和总金额：

```sql
SELECT customerid, sum(total) as total, count(*) AS countrows
FROM sales
GROUP BY customerid
ORDER BY customerid;
```

你将得到类似这样的客户销售总额：

| customerid | total | countrows |
| --- | --- | --- |
| 2 | 14920 | 24 |
| 8 | 7885 | 16 |
| 9 | 10645 | 19 |
| 10 | 9010 | 19 |
| 11 | 16460 | 29 |
| 15 | 6005 | 13 |
| ~ 256 行 ~ |   |   |

在总计聚合中，你只有一行汇总信息。你无法获得非聚合值。在组汇总中，你会为每个分组额外获得一列，正如你所看到的那样。

在聚合查询中，你只能选择汇总值或分组值。不能选择其他任何内容。



