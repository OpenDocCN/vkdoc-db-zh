# 8. 窗口函数

到目前为止，你已经了解了两大类计算：

*   大多数计算都是基于表的`列`：对于每一行，根据一个或多个列计算出一个值。
*   聚合查询用于汇总`行`：针对整个表，汇总部分或全部行。

`窗口函数`是一组将行数据作为列添加的函数。我们将研究三组窗口函数：

*   聚合函数：通常，你会将聚合值作为表数据的单独汇总获取，但聚合窗口函数允许你将聚合值与每一行一起包含。除了其他用途，你将看到如何利用它来生成累计总计。
*   排名函数：这将根据当前行在数据集中的位置生成一个值。使用排序函数，你可以获取行号、相对排名，甚至是像十分位这样的分组。
*   值函数：你可以获取当前行之前或之后的行中的数据。你还可以获取每组中的第一个和最后一个值。例如，这将帮助你获取当前行与另一行之间的值差。

在本章中，我们将探讨所有这些函数。

窗口函数对于 SQL 来说相对较新，但大多数现代数据库管理系统现在都支持它们。再次落后的是 MariaDB（在 10.2 版本引入）和 MySQL（在 8 版本引入）。

在开始之前，部分示例将使用`sales`表。该表包含一些下单日期/时间的`NULL`值。大概这些销售从未结账。到目前为止我们一直比较宽容，时不时将它们过滤掉，但现在是时候处理它们了。我们可以按如下方式删除所有`NULL`销售记录：

```
DELETE FROM sales WHERE ordered IS NULL;
```

你会注意到`saleitems`表有一个指向`sales`表的外键，这通常不允许在存在相关项目时删除销售记录。但是，如果你检查生成示例数据库的脚本，你会发现`ON DELETE CASCADE`子句，它会自动删除孤立的销售项目。

## 编写窗口函数

窗口函数在一组行上生成一个值。这组行被称为一个`窗口`。

窗口函数的通用语法是：

```
fn() OVER (PARTITION BY columns |
ORDER BY columns | frame clause)
```

重要部分是`OVER()`子句，它定义了要汇总的窗口。有三个主要的窗口子句：

*   `PARTITION BY`：为定义的组计算函数。它等同于`GROUP BY`。默认的分区是整个表。
*   `ORDER BY`：按定义的顺序累积计算函数。换句话说，它生成累计总计。这个顺序不需要与表的`ORDER BY`子句相同。
*   还有一个可选的框架子句。这会在分区内创建一个滑动窗口。框架子句需要一个`ORDER BY`窗口子句。默认情况下，框架是从开头到当前行的行，但当我们讲到那里时需要限定说明。

在以下示例中，`SELECT`语句末尾通常有一个`ORDER BY`子句，其内容与`OVER()`子句中的相同。这不是必需的，但它使结果更容易理解。

### 简单的聚合窗口

如你所知，你不能在非聚合查询中混合使用聚合函数。例如，这是一个无法工作的聚合查询：

```
SELECT
id, givenname, familyname,
count(*)
FROM customerdetails;
```

但是，这样可以工作：

```
SELECT
id, givenname, familyname,
count(*) OVER ()
FROM customerdetails;
```

这会给你类似这样的结果：

| **id** | **givenname** | **familyname** | **count** |
| --- | --- | --- | --- |
| 42 | May | Knott | 303 |
| 459 | Rick | Shaw | 303 |
| 597 | Ike | Andy | 303 |
| 186 | Pat | Downe | 303 |
| 352 | Basil | Isk | 303 |
| 576 | Pearl | Divers | 303 |
| ~ 303 行 ~ |

`OVER()`子句将聚合函数变成了一个`窗口`函数。现在将为每一列生成这个聚合函数。稍后你会看到，`OVER()`子句定义了任何分组（称为分区）、顺序以及在聚合中考虑的行数。

对于如此简单的情况，你可以使用子查询获得相同的结果：

```
SELECT
id, givenname, familyname,
(SELECT count(*) FROM customers)
FROM customerdetails;
```

当你应用一个窗口子句时，窗口函数会变得更有趣。例如：

```
SELECT
id, givenname, familyname,
count(*) OVER (ORDER BY id)
FROM customerdetails;
```

这将按`id`的顺序给出截至当前行（包括当前行）的运行计数。实际表结果可能按行顺序排列，也可能不按行顺序排列，特别是如果你包含其他表达式时，因此最好在末尾添加`ORDER BY`：

```
SELECT
id, givenname, familyname,
count(*) OVER (ORDER BY id) AS running_count
FROM customerdetails
ORDER BY id;
```

现在你会得到类似这样的结果：

| **id** | **Givenname** | **familyname** | **running_count** |
| --- | --- | --- | --- |
| 1 | Pierce | Dears | 1 |
| 2 | Arthur | Moore | 2 |
| 5 | Ray | King | 3 |
| 6 | Gene | Poole | 4 |
| 9 | Donna | Worry | 5 |
| 10 | Ned | Duwell | 6 |
| ~ 303 行 ~ |

`running_count`列看起来非常像一个简单的行号。我们稍后会看到，如果`ORDER BY`列不是唯一的，它不一定相同。


#### 聚合函数

通常，你无法在常规查询中使用聚合函数，除非将它们嵌入子查询中。然而，它们可以重新用作**窗口函数**。

之前你已经看到，可以使用表达式 `count(*) OVER ()` 在每一行上给出总计数。你也可以对 `sum()` 或 `avg()` 函数做类似的事情。

例如，假设你想比较销售总额与总体平均值：

```sql
SELECT
id, ordered, total,
total-avg(total) OVER () AS difference
FROM sales;
```

你会得到类似下面的结果：

| **id** | **Ordered** | **total** | **difference** |
| --- | --- | --- | --- |
| 39 | 2022-05-15 21:12:07.988741 | 28 | -33.783 |
| 40 | 2022-05-16 03:03:16.065969 | 34 | -27.783 |
| 42 | 2022-05-16 10:09:13.674823 | 58.5 | -3.283 |
| 43 | 2022-05-16 15:02:43.285565 | 50 | -11.783 |
| 45 | 2022-05-16 16:48:14.674202 | 17.5 | -44.283 |
| 518 | [NULL] | 13 | -48.783 |
| ~ 5549 rows ~ |

在一个更复杂的例子中，假设你想比较每天的销售额与该周其余时间的对比。首先，你可以从 `sales` 表中仅提取星期几和总额。为此，你可以使用星期名称或星期数字，但我们使用星期数字：

```sql
--  PostgreSQL: Sunday=0
SELECT
EXTRACT(dow FROM ordered) AS weekday_number,
total
FROM sales;

--  MSSQL: Sunday=1
SELECT
datepart(weekday,ordered) AS weekday_number,
total
FROM sales;

--  Oracle: Sunday=1
SELECT
to_char(ordered,'D')+0 AS weekday_number,
total
FROM sales;

--  MariaDB/MySQL: Sunday=1
SELECT
dayofweek(ordered) AS weekday_number,
total
FROM sales;

--  SQLite: Sunday=0
SELECT
strftime('%w',ordered) AS weekday_number,
total
FROM sales;
```

你会看到它们都有不同的实现方式，甚至对星期数字的定义都无法达成一致。幸运的是，它们对一周的第一天是什么达成了一致：

| **weekday_number** | **Total** |
| --- | --- |
| 0 | 28 |
| 1 | 34 |
| 1 | 58.5 |
| 1 | 50 |
| 1 | 17.5 |
| 0 | 13 |
| ~ 5549 rows ~ |

接下来，将其放入一个 CTE 中，以便可以进行聚合：

```sql
WITH
data AS (
SELECT
... AS weekday,
total
FROM sales
)
--  待续
;
```

然后，你可以在另一个 CTE 中汇总数据：

```sql
WITH
data AS (
SELECT
... AS weekday_number,
total
FROM sales
),
summary AS (
SELECT weekday_number, sum(total) AS total
FROM data
GROUP BY weekday_number
)
--  等等
```

最后，你可以使用窗口聚合将每日总计与总计进行比较：

```sql
WITH
data AS (...),
summary AS (...)
SELECT
weekday_number, total,
total/sum(total) OVER() AS proportion
FROM summary
ORDER BY weekday_number;
```

这将给你一个逐日的汇总：

| **weekday_number** | **total** | **?column?** |
| --- | --- | --- |
| 0 | 48182.22 | 0.147 |
| 1 | 49304 | 0.151 |
| 2 | 45156.5 | 0.138 |
| 3 | 45959.5 | 0.141 |
| 4 | 47528 | 0.145 |
| 5 | 42372.5 | 0.13 |
| 6 | 48415.5 | 0.148 |

注意，表达式 `total/sum(total) OVER()` 可能有点令人困惑，因为 `OVER()` 子句看起来似乎没什么作用。你可能更愿意将其写成 `total/(sum(total) OVER ())`，以更清楚地表明这实际上是一个单一的表达式。这取决于你的偏好，但通常不这样写。

你可以通过给计算起一个别名、将其显示为百分比并按星期排序来完成：

```sql
WITH
data AS (...),
summary AS (...)
SELECT
weekday_number, total,
100*total/sum(total) OVER() AS proportion
FROM summary
;
```

如果你想显示百分号，这取决于数据库管理系统（DBMS）。你可以尝试以下几种方法：

```sql
--  PostgreSQL
to_char(100*total/sum(total) OVER(),'99.9%')

--  MariaDB/MySQL
format(100*total/sum(total) OVER(),2) || '%'

--  MSSQL
format(100*total/sum(total) OVER(),'0.0%')

--  SQLite: 等同于 printf(...)
select format('%.1f%%',100*total/sum(total) OVER())

--  Oracle
to_char(100*total/sum(total) OVER(),'99.9') || '%'
```

这看起来更有说服力：

| **weekday_number** | **total** | **proportion** |
| --- | --- | --- |
| 0 | 48182.22 | 14.7% |
| 1 | 49304 | 15.1% |
| 2 | 45156.5 | 13.8% |
| 3 | 45959.5 | 14.1% |
| 4 | 47528 | 14.5% |
| 5 | 42372.5 | 13.0% |
| 6 | 48415.5 | 14.8% |

我们使用 `OVER()` 来计算表的总计。然而，我们也可以使用滑动窗口，这将在下一节中看到。

#### 聚合窗口函数和 ORDER BY

回顾我们的入门示例，其中在 `OVER()` 子句中包含了 `ORDER BY` 子句：

```sql
SELECT
id, givenname, familyname,
count(*) OVER (ORDER BY id) AS running_count
FROM customerdetails
ORDER BY id;
```

在这个例子中，`id` 作为主键是唯一的。这会给我们一个关于其工作原理的错误印象，所以让我们看看使用非唯一的 `height`。我们还将过滤掉 `NULL` 的身高值，以便更明显：

```sql
SELECT
id, givenname, familyname,
height,
count(*) OVER (ORDER BY height) AS running_count
FROM customerdetails
WHERE height IS NOT NULL
ORDER BY height;
```

你会看到一些重复的身高值，以及它们如何影响窗口函数：

| **id** | **givenname** | **familyname** | **Height** | **running_count** |
| --- | --- | --- | --- | --- |
| 597 | Ike | Andy | 153 | 2 |
| 283 | Ethel | Glycol | 153 | 2 |
| 451 | Fred | Knott | 153.8 | 3 |
| 194 | Rod | Fishing | 154.3 | 4 |
| 534 | Minnie | Bus | 156.4 | 6 |
| 352 | Basil | Isk | 156.4 | 6 |
| ~ 267 rows ~ |

在 `OVER` 子句中使用 `ORDER BY` 意味着计算到当前值为止的行数。这可能符合也可能不符合你的预期。

## 帧子句

在这个例子中，存在一个隐含的**帧**子句，它默认为此行为。如果你愿意，可以使其更具体：

```sql
count(*) OVER (ORDER BY height
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

这相当冗长，但这就是 SQL 语言的发展方式：如果能用二十个词说清楚的事，为什么要只用两个词呢？^(⁵) 这里，`RANGE` 一词指的是 `height` 的值。例如，在前面的第五行中，该值与下一行相同，因此 `count(*)` 包含了两者。

一个显而易见的替代方案是：

```sql
SELECT
id, givenname, familyname,
height,
count(*) OVER (ORDER BY height
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_count
FROM customerdetails
WHERE height IS NOT NULL
ORDER BY height;
```

一个细微的变化是从 `RANGE BETWEEN` 变为 `ROWS BETWEEN`。它现在计算的是到当前*行*为止的行数。

| **id** | **givenname** | **familyname** | **Height** | **running_count** |
| --- | --- | --- | --- | --- |
| 597 | Ike | Andy | 153 | 1 |
| 283 | Ethel | Glycol | 153 | 2 |
| 451 | Fred | Knott | 153.8 | 3 |
| 194 | Rod | Fishing | 154.3 | 4 |
| 534 | Minnie | Bus | 156.4 | 5 |
| 352 | Basil | Isk | 156.4 | 6 |
| ~ 267 rows ~ |

这有点不公平：两个身高相同的客户被任意地排在了彼此之前。我们稍后会看到更多这种不公平性。

帧子句可以采用以下形式：

```sql
[ROW|RANGE] BETWEEN start AND end
```

如我们所见，`ROWS` 和 `RANGE` 之间的区别在于，`RANGE` 包含所有与当前值匹配的行，而 `ROWS` 则不是。

### 帧边界

`start` 和 `end` 表达式，也称为**帧边界**，可以采用以下形式之一：

| Expression | Meaning |
| --- | --- |
| `UNBOUND PRECEDING` | 开始 |
| `n PRECEDING` | 当前行*之前*的行数 |
| `CURRENT ROW` | |
| `n FOLLOWING` | 当前行*之后*的行数 |
| `UNBOUND FOLLOWING` | 结束 |

还有一个简写形式：

```sql
ROWS|RANGE start
```

这表示介于开始和当前行之间。



#### 创建每日销售视图

在我们继续之前，我们后续的一些示例需要准备好的销售数据。虽然我们可以在公共表表达式中完成此操作，但准备一个视图会更有意义，可以省去我们之后的一些麻烦。

我们将需要每日销售额以及销售月份。视图看起来会像这样：

```sql
CREATE VIEW daily_sales AS
SELECT
ordered_date,
--  PostgreSQL, Oracle
to_char(ordered_date,'YYYY-MM') AS ordered_month,
--  MariaDB/MySQL
--  date_format(ordered_date,'%Y-%m')
AS ordered_month,
--  MSSQL
--  format(ordered_date,'yyyy-MM') AS ordered_month,
--  SQLite
--  strftime('%Y-%m',ordered_date) AS ordered_month,
sum(total) AS daily_total
FROM sales
WHERE ordered IS NOT NULL
GROUP BY ordered_date;
```

（对于 MSSQL，别忘了在语句前后用 `GO` 包裹。）

我们可以测试一下：

```sql
SELECT * FROM daily_sales ORDER BY ordered_date;
```

你应该会看到类似这样的结果：

| **订购日期** | **订购月份** | **日销售总额** |
| --- | --- | --- |
| 2022-05-04 | 2022-05 | 43 |
| 2022-05-05 | 2022-05 | 150.5 |
| 2022-05-06 | 2022-05 | 110.5 |
| 2022-05-07 | 2022-05 | 142 |
| 2022-05-08 | 2022-05 | 214.5 |
| 2022-05-09 | 2022-05 | 16.5 |
| ~ 约 389 行 ~ |

#### 滑动窗口

这是一个使用带框架子句的滑动窗口的示例。假设我们想为每一天以及截至该天的一周生成每日总额。我们可以使用

```sql
SELECT
ordered_date, daily_total,
sum(daily_total) OVER(ORDER BY ordered_date
ROWS 6 PRECEDING) AS week_total,
sum(daily_total) OVER(ORDER BY ordered_date
ROWS UNBOUNDED PRECEDING) AS running_total
FROM daily_sales
ORDER BY ordered_date;
```

对于两个框架子句，我们都使用了较短的形式，因为我们希望截至当前行。对于累计总额，我们本可以完全省略框架子句，但我们必须将默认的 `RANGE BETWEEN` 改掉，以防两个日销售总额相同。

你会得到类似下面的结果：

| **订购日期** | **日销售总额** | **周总额** | **累计总额** |
| --- | --- | --- | --- |
| 2022-05-04 | 43 | 43 | 43 |
| 2022-05-05 | 150.5 | 193.5 | 193.5 |
| 2022-05-06 | 110.5 | 304 | 304 |
| 2022-05-07 | 142 | 446 | 446 |
| 2022-05-08 | 214.5 | 660.5 | 660.5 |
| 2022-05-09 | 16.5 | 677 | 677 |
| 2022-05-10 | 160 | 837 | 837 |
| 2022-05-11 | 115 | 909 | 952 |
| 2022-05-12 | 205 | 963.5 | 1157 |
| 2022-05-13 | 164.5 | 1017.5 | 1321.5 |
| 2022-05-14 | 46.5 | 922 | 1368 |
| 2022-05-15 | 457.5 | 1165 | 1825.5 |
| ~ 约 389 行 ~ |

注意，对于前七天，周总额和累计总额是相同的，因为之前没有总额数据。然而，从那以后，累计总额持续累加，而周总额则被限制在当前的七天内。

如果你仔细观察，可能还会发现日期中有一些间隔。这意味着那些天没有销售，这也可能给理解数据带来麻烦，因为一行不一定代表一天。我们将在第 9 章解决这个问题。

记住，你不仅限于 `count()` 和 `sum()` 函数。例如，你也可以创建滑动平均值：

```sql
SELECT
ordered_date, daily_total,
sum(daily_total) OVER(ORDER BY ordered_date
ROWS 6 PRECEDING) AS week_total,
avg(daily_total) OVER(ORDER BY ordered_dat
ROWS 6 PRECEDING) AS week_average,
sum(daily_total) OVER(ORDER BY ordered_date
ROWS UNBOUNDED PRECEDING) AS running_total
FROM daily_sales
ORDER BY ordered_date;
```

周平均值是包含当天的七天内的平均值：

| **订购日期** | **日销售总额** | **周总额** | **周平均值** | **累计总额** |
| --- | --- | --- | --- | --- |
| 2022-05-04 | 43 | 43 | 43 | 43 |
| 2022-05-05 | 150.5 | 193.5 | 96.75 | 193.5 |
| 2022-05-06 | 110.5 | 304 | 101.333 | 304 |
| 2022-05-07 | 142 | 446 | 111.5 | 446 |
| 2022-05-08 | 214.5 | 660.5 | 132.1 | 660.5 |
| 2022-05-09 | 16.5 | 677 | 112.833 | 677 |
| 2022-05-10 | 160 | 837 | 119.571 | 837 |
| 2022-05-11 | 115 | 909 | 129.857 | 952 |
| 2022-05-12 | 205 | 963.5 | 137.643 | 1157 |
| 2022-05-13 | 164.5 | 1017.5 | 145.357 | 1321.5 |
| 2022-05-14 | 46.5 | 922 | 131.714 | 1368 |
| 2022-05-15 | 457.5 | 1165 | 166.429 | 1825.5 |
| ~ 约 389 行 ~ |

你还可以选择滑动最小值和最大值，或者截至当前的平均值。你需要根据自己的目的来决定其中哪些是有用的。



#### 窗口函数小计

之前，我们用类似 `sum(total) OVER()` 的表达式创建了总计。`OVER()` 表达式是求和整个表的简写。

也可以按组进行求和（或计数，或其他操作）。你可能会认为语法会是类似 `sum(total) OVER (GROUP BY ...)` 的形式，但这太直白了。实际上，我们使用表达式 `(PARTITION BY ...)` 来表示分组。

默认的分区是整个表。你可以按任何可以分组的字段进行分区。例如，假设你想用前面的例子获取每月总计，你可以使用

```sql
SELECT
ordered_date, daily_total,
sum(daily_total) OVER(ORDER BY ordered_date
ROWS 6 PRECEDING) AS week_total,
sum(daily_total) OVER(ORDER BY ordered_date
ROWS UNBOUNDED PRECEDING) AS running_total,
sum(daily_total) OVER(PARTITION BY ordered_month)
AS monthly_total
FROM daily_sales
ORDER BY ordered_date;
```

现在你会看到类似这样的结果：

| `ordered_date` | `daily_total` | `week_total` | `running_total` | `monthly_total` |
| --- | --- | --- | --- | --- |
| 2022-05-04 | 43 | 43 | 43 | 6966.5 |
| 2022-05-05 | 150.5 | 193.5 | 193.5 | 6966.5 |
| 2022-05-06 | 110.5 | 304 | 304 | 6966.5 |
| 2022-05-07 | 142 | 446 | 446 | 6966.5 |
| 2022-05-08 | 214.5 | 660.5 | 660.5 | 6966.5 |
| 2022-05-09 | 16.5 | 677 | 677 | 6966.5 |
| 2022-05-10 | 160 | 837 | 837 | 6966.5 |
| 2022-05-11 | 115 | 909 | 952 | 6966.5 |
| 2022-05-12 | 205 | 963.5 | 1157 | 6966.5 |
| 2022-05-13 | 164.5 | 1017.5 | 1321.5 | 6966.5 |
| 2022-05-14 | 46.5 | 922 | 1368 | 6966.5 |
| 2022-05-15 | 457.5 | 1165 | 1825.5 | 6966.5 |
| ~ 389 行 ~ |

当然，对于每个月，你都会得到一个新的总计。

现在，棘手的部分来了。你也可以将 `PARTITION BY` 和 `ORDER BY` 结合起来使用：

```sql
sum(daily_total) OVER(
PARTITION BY ordered_month
ORDER BY ordered_date ROWS UNBOUNDED PRECEDING
) AS month_running_total
```

以下是一些可能组合的示例：

```sql
SELECT
ordered_date, daily_total,
sum(daily_total) OVER(ORDER BY ordered_date
ROWS UNBOUNDED PRECEDING) AS running_total,
sum(daily_total) OVER(PARTITION BY ordered_month)
AS month_total,
sum(daily_total) OVER(ORDER BY ordered_month)
AS running_month_total,
sum(daily_total) OVER(PARTITION BY ordered_month
ORDER BY ordered_date ROWS UNBOUNDED PRECEDING)
AS month_running_total
FROM daily_sales
ORDER BY ordered_date;
```

你会看到类似这样的结果（列名已缩写以适配页面）：

| `ordered_date` | `daily_total` | `Rt` | `mt` | `rmt` | `mrt` |
| --- | --- | --- | --- | --- | --- |
| 2022-05-04 | 43 | 43 | 6966.5 | 6966.5 | 43 |
| 2022-05-05 | 150.5 | 193.5 | 6966.5 | 6966.5 | 193.5 |
| 2022-05-06 | 110.5 | 304 | 6966.5 | 6966.5 | 304 |
| 2022-05-07 | 142 | 446 | 6966.5 | 6966.5 | 446 |
| 2022-05-08 | 214.5 | 660.5 | 6966.5 | 6966.5 | 660.5 |
| 2022-05-09 | 16.5 | 677 | 6966.5 | 6966.5 | 677 |
| 2022-05-10 | 160 | 837 | 6966.5 | 6966.5 | 837 |
| 2022-05-11 | 115 | 952 | 6966.5 | 6966.5 | 952 |
| 2022-05-12 | 205 | 1157 | 6966.5 | 6966.5 | 1157 |
| 2022-05-13 | 164.5 | 1321.5 | 6966.5 | 6966.5 | 1321.5 |
| 2022-05-14 | 46.5 | 1368 | 6966.5 | 6966.5 | 1368 |
| 2022-05-15 | 457.5 | 1825.5 | 6966.5 | 6966.5 | 1825.5 |
| ~ 389 行 ~ |

这些名字可能有些令人困惑，所以下面的表格解释了具体发生了什么：

| 子句 | 名称 | 发生了什么 |
| --- | --- | --- |
| `ORDER BY date ...` | `running_total` | 从开头到当前行的累计总计 |
| `PARTITION BY month` | `month_total` | 当前组的总计 |
| `ORDER BY month` | `running_month_total` | 每个月的累计总计 |
| `PARTITION BY month ORDER BY date ...` | `month_running_total` | 每个月份内的累计总计 |

（同样，列名已缩写以便适配。）

注意我们是如何使用分组列 `ordered_month` 既进行分区，又进行累计总计的。因为它的默认框架是 `RANGE ...`，它会对到目前为止的所有值进行求和，这实际上就是整个月的总计。如果你按非唯一的列排序，就会出现这种情况。

整个过程中最难的部分是为结果想个好名字。

作为汇总，这些都是很好的保存为视图的候选对象。

但请注意，仅在 SQL Server 中，如果不使用额外的技巧，你无法在视图中包含 `ORDER BY` 子句。因此，你至少应该确保你的 `SELECT` 语句包含你想要排序的列，并在使用视图时再包含 `ORDER BY` 子句。

或者，你可以在 `ORDER BY` 子句末尾使用 `OFFSET 0 ROWS` 作为变通方法。



### 按多列进行 `PARTITION BY`

已知 `PARTITION BY` 会生成小计，那么 `PARTITION BY` 多列将会生成子子计——如果这个词真存在的话。

例如，假设你想生成一份按州、城镇和客户划分的销售报告。数据是存在的，但它分布在多个表中，你首先需要对其进行准备。

首先，你需要将包含州和城镇信息的 `customerdetails` 视图与销售数据连接起来。到时候，我们会将其放入一个名为 `customer_sales` 的 CTE 中：

```sql
--  customer_sales
SELECT c.id AS customerid, c.state, c.town, total
FROM customerdetails AS c JOIN sales AS s
ON c.id=s.customerid
```

然后，我们希望按州、城镇和客户 ID 进行分组汇总。同样，这会进入另一个 CTE：

```sql
--  totals
SELECT state, town, customerid, sum(total) AS total
FROM customer_sales
GROUP BY state, town, customerid
```

我们可以将它们整合在一起并检查结果：

```sql
WITH
customer_sales AS (
SELECT c.id AS customerid, c.state, c.town, total
FROM customerdetails AS c JOIN sales AS s
ON c.id=s.customerid
),
totals AS (
SELECT state, town, customerid, sum(total) AS total
FROM customer_sales
GROUP BY state, town, customerid
)
SELECT state, town, customerid, total AS customer_total
FROM totals
ORDER BY state, customerid;
```

你会得到类似这样的结果：

| `state` | `Town` | `customerid` | `customer_total` |
| --- | --- | --- | --- |
| ACT | Kingston | 85 | 2469 |
| ACT | Kingston | 112 | 1387 |
| ACT | Kingston | 147 | 2439.5 |
| ACT | Kingston | 355 | 689.5 |
| ACT | Gordon | 489 | 199 |
| NSW | Reedy Creek | 10 | 3089 |
| ~ 约 269 行 ~ |

现在来看窗口函数。首先，要获取按州分组的总计，我们可以使用

```sql
sum(total) OVER(PARTITION BY state) AS state_total
```

要获取每个城镇的组总计，请记住，一个城镇名可能出现在多个州中。使用 `PARTITION BY town` 将是错误的，因为城镇名会被混淆。相反，我们应该使用

```sql
sum(total) OVER(PARTITION BY state, town) AS town_total
```

将这两个表达式合并，并添加一个 `ORDER BY` 子句以便完整查看，我们得到：

```sql
WITH
customer_sales AS (
SELECT c.id AS customerid, c.state, c.town, total
FROM customerdetails AS c JOIN sales AS s
ON c.id=s.customerid
),
totals AS (
SELECT state, town, customerid, sum(total) AS total
FROM customer_sales
GROUP BY state, town, customerid
)
SELECT
state, town, customerid, total AS customer_total,
sum(total) OVER(PARTITION BY state) AS state_total,
sum(total) OVER(PARTITION BY state, town) AS town_total
FROM totals
ORDER BY state, customerid;
```

结果如下所示：

| `state` | `Town` | `customerid` | `customer_total` | `state_total` | `town_total` |
| --- | --- | --- | --- | --- | --- |
| ACT | Kingston | 85 | 2469 | 7184 | 6985 |
| ACT | Kingston | 112 | 1387 | 7184 | 6985 |
| ACT | Kingston | 147 | 2439.5 | 7184 | 6985 |
| ACT | Kingston | 355 | 689.5 | 7184 | 6985 |
| ACT | Gordon | 489 | 199 | 7184 | 199 |
| NSW | Reedy Creek | 10 | 3089 | 106389.22 | 12655 |
| ~ 约 269 行 ~ |

州和城镇之间存在隐含的层级关系：城镇是州的一部分（并且，目前来看，客户位于某个城镇中）。因此，`PARTITION BY` 子句必须遵循这个层级：`state,town`。你也可以使用不相关的列，例如州和出生年份，在这种情况下，列的顺序可以任意。

## 排名函数

到目前为止使用的窗口函数基本上是赋予了新上下文的聚合函数。另一组函数是窗口函数所特有的。通常，它们与当前行的位置相关。大体上，我们可以称它们为 **排名函数**。

有一个聚合窗口函数，我们已经见过，它同时也充当排名函数：

```sql
SELECT
id, givenname, familyname,
height,
count(*) OVER (ORDER BY height
ROWS UNBOUNDED PRECEDING) AS running_count
FROM customers
WHERE height IS NOT NULL
ORDER BY height;
```

只要使用框架子句 `ROWS UNBOUNDED PRECEDING`（`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` 的简写），`count(*)` 就会统计到当前行为止的行数，这基本上就是结果集中的行号。

有一个更简单的替代方法：

```sql
SELECT
id, givenname, familyname,
height,
row_number() OVER (ORDER BY height) AS running_count
FROM customers
WHERE height IS NOT NULL
ORDER BY height;
```

`row_number()` 函数基本上就是生成这个：为结果集中的每一行生成一个编号。



### 基本排名函数

主要有四个排名函数：

*   `row_number()`: 计算当前行在当前分区中指定顺序下的行号。

    如果`ORDER BY`子句中的两个值相同，它们仍会获得不同的行号；无法保证谁排在前面。

*   `rank()`: 在结果集中给出排名。

    如果`ORDER BY`子句中的两个值相同，它们将获得相同的排名。下一个不同的值不会获得紧接的下一个排名；它会追上一行的行号。

*   `count(*)`: 如果你省略窗口框架子句，让它默认为`RANGE`，它的行为将与`rank()`相似，但有一个区别。我们稍后会看这个区别。

*   `dense_rank()`: 这也会给出一个排名，类似于之前的`rank()`。然而，下一个不同的值会获得下一个排名，因此它会逐渐落后于行号。

如果未指定分区（即没有`PARTITION BY`子句），则上述函数将应用于整个表。否则，它们将给出组内的位置。

`rank()`和`dense_rank()`之间的区别在于，对于相等的值，`rank()`会从下一个`row_number()`开始计数，而`dense_rank()`不会。

如果`ORDER BY`值不是唯一的

*   `row_number()`是任意的。

*   `rank()`给出的是组*开始*处的排名。

*   `count(*)`给出的是组*结束*处的排名。

*   `dense_rank()`给出的是组的排名。

如果`ORDER BY`值是唯一的，这些函数都会给出相同的结果。

我们可以用客户身高来测试这一点，我们知道有些身高是重复的：

```sql
SELECT
id, givenname, familyname,
height,
row_number() OVER (ORDER BY height) AS row_number,
count(*) OVER (ORDER BY height) AS count,
rank() OVER (ORDER BY height) AS rank,
dense_rank() OVER (ORDER BY height) AS dense_rank
FROM customers
WHERE height IS NOT NULL
ORDER BY height;
```

你会得到类似这样的结果：

| `id` | `…` | `height` | `row_number` | `count` | `rank` | `dense_rank` |
| --- | --- | --- | --- | --- | --- | --- |
| 597 | … | 153 | 1 | 2 | 1 | 1 |
| 283 | … | 153 | 2 | 2 | 1 | 1 |
| 451 | … | 153.8 | 3 | 3 | 3 | 2 |
| 194 | … | 154.3 | 4 | 4 | 4 | 3 |
| 534 | … | 156.4 | 5 | 6 | 5 | 4 |
| 352 | … | 156.4 | 6 | 6 | 5 | 4 |
| ~ 267 行 ~ |

当然，你的实际结果可能不同。然而，在上面的例子中，我们可以看到

*   `row_number()`是唯一的，与实际值无关。
*   对于相等的值，`rank()`是相同的。*下一个*值与`row_number()`匹配。
*   对于相等的值，`count(*)`也是相同的。*下一个*值也与`row_number()`匹配。
*   对于相等的值，`rank()`与*第一个*`row_number()`相同；`count(*)`与*最后一个*`row_number()`相同。
*   对于相等的值，`dense_rank()`也是相同的。*下一个*值会获得下一个排名。等到结果集结束时，它会与行号相差很大。

对于大多数数据库管理系统，排名函数都需要一个`ORDER BY`窗口子句。这是合理的，因为没有顺序，排名就没有意义。

例外包括 PostgreSQL 和 SQLite，它们允许空的窗口子句：

```sql
--  PostgreSQL, SQLite
SELECT
id, givenname, familyname,
height,
row_number() OVER () AS row_number,
count(*) OVER () AS count,
rank() OVER () AS rank,
dense_rank() OVER () AS dense_rank
FROM customers
WHERE height IS NOT NULL
ORDER BY height;
```

然而，结果是没有意义的。`count(*)`、`rank()`和`dense_rank()`表达式都为整个结果集给出一个值，而`row_number()`则以任意顺序给出的行号。

### 使用 PARTITION BY 进行排名

默认情况下，像`row_number()`这样的排名函数会对整个结果集进行排名。你也可以使用`PARTITION BY`按组进行排名：

```sql
SELECT
id, ordered_date, total,
row_number() OVER (PARTITION BY ordered_date) AS row_number
FROM sales
ORDER BY ordered;
```

结果将是类似这样的：

| `id` | `ordered_date` | `total` | `row_number` |
| --- | --- | --- | --- |
| 1 | 2022-05-04 | 43 | 1 |
| 2 | 2022-05-05 | 54.5 | 1 |
| 3 | 2022-05-05 | 96 | 2 |
| 6 | 2022-05-06 | 18 | 2 |
| 7 | 2022-05-06 | 92.5 | 1 |
| 4 | 2022-05-07 | 17.5 | 1 |
| ~ 5295 行 ~ |

行号可能不是预期的顺序，因为没有指定排序。为了完成工作，我们还应该包含排序：

```sql
SELECT
id, ordered_date, total,
row_number() OVER (
PARTITION BY ordered_date ORDER BY ordered
) AS row_number
FROM sales
ORDER BY ordered;
```

现在的行号就是我们所预期的顺序了：

| `id` | `ordered_date` | `total` | `row_number` |
| --- | --- | --- | --- |
| 1 | 2022-05-04 | 43 | 1 |
| 2 | 2022-05-05 | 54.5 | 1 |
| 3 | 2022-05-05 | 96 | 2 |
| 6 | 2022-05-06 | 18 | 1 |
| 7 | 2022-05-06 | 92.5 | 2 |
| 4 | 2022-05-07 | 17.5 | 1 |
| ~ 5295 行 ~ |

你可以以一种创造性的方式使用分组行号。例如，你可能只想显示每天第一笔销售的日期。你可以使用`CASE ... END`表达式有选择地显示日期：

```sql
CASE
WHEN row_number() OVER
(PARTITION BY ordered_date ORDER BY ordered)=1
THEN CAST(ordered_date AS varchar(16))
ELSE ''
END AS ordered_date,
```

重新排列并重命名几个列，你会得到

```sql
SELECT
id,
CASE
WHEN row_number() OVER
(PARTITION BY ordered_date ORDER BY ordered)=1
THEN CAST(ordered_date AS varchar(16))
ELSE ''
END AS ordered_date,
row_number() OVER (PARTITION BY ordered_date) AS item,
total
FROM sales
ORDER BY ordered;
```

这会给你一个看起来更简洁的结果：

| `id` | `ordered_date` | `item` | `Total` |
| --- | --- | --- | --- |
| 1 | 2022-05-04 | 1 | 43 |
| 2 | 2022-05-05 | 1 | 54.5 |
| 3 | | 2 | 96 |
| 6 | 2022-05-06 | 1 | 18 |
| 7 | | 2 | 92.5 |
| 4 | 2022-05-07 | 1 | 17.5 |
| 5 | | 2 | 63 |
| 9 | | 3 | 61.5 |
| 10 | 2022-05-08 | 1 | 67.5 |
| 11 | | 2 | 18.5 |
| 8 | | 3 | 54 |
| 13 | | 4 | 74.5 |
| ~ 5295 行 ~ |

当然，你仍然可以包含你的累计总数。



#### 分页结果

你可能需要知道总行数的一个原因，是希望将结果分成多页。例如，假设你希望每页显示，比如说，二十条记录，而现在你想展示其中的第三页。

我们可以从之前的 `pricelist` 视图开始，并加入 `row_number()` 窗口函数：

```sql
SELECT
id, title, published, author,
price, tax, inc,
row_number() OVER(ORDER BY id) AS row_number
FROM aupricelist;
```

我们尚未包含 `ORDER BY` 子句，因为后面还有内容。某些数据库管理系统可能会按 `id` 顺序生成结果，但这当然不能保证。

现在，我们可以将其放入一个公用表表达式（CTE）中，并基于行号进行筛选：

```sql
WITH cte AS (
SELECT
id, title, published, author,
price, tax, inc,
row_number() OVER(ORDER BY id) AS row_number
FROM aupricelist
)
SELECT *
FROM cte
WHERE row_number BETWEEN 40 AND 59
ORDER BY id;
```

你会得到类似下面的结果：

| **id** | **title** | **…** | **price** | **tax** | **inc** | **row_number** |
| --- | --- | --- | --- | --- | --- | --- |
| 98 | 卡米拉 | … | 12 | 1.2 | 13.2 | 40 |
| 102 | 汉索姆之谜 … | … | 14.5 | 1.45 | 15.95 | 41 |
| 103 | 波斯信札 | … | 15.5 | 1.55 | 17.05 | 42 |
| 104 | 罪人在愤怒上帝手中 … | … | 19.5 | 1.95 | 21.45 | 43 |
| 106 | 特拉法尔加 | … | 16 | 1.6 | 17.6 | 44 |
| 109 | 红字与 … | … | 19.5 | 1.95 | 21.45 | 45 |
| ~ 约 20 行 ~ |

Oracle 有一个内置的值叫做 `rownum`。遗憾的是，你仍然需要通过 CTE 或子查询来使用它。

当然，你不一定要按 `id` 排序。你可以使用 `title` 或 `price`，只要在窗口函数和 `ORDER BY` 子句中都包含它就行。当然，你也可以使用 `DESC` 来降序排列。

还有一种替代方法。正式地说，你可以使用 `OFFSET ... FETCH ...` 子句：

```sql
--  PostgreSQL, MSSQL, Oracle
SELECT
id, title, published, author,
price, tax, inc,
row_number() OVER(ORDER BY id) AS row_number
ORDER BY id OFFSET 40 ROWS FETCH FIRST 20 ROWS ONLY;
```

这会跳过前 40 行，并获取之后的 20 行。

非正式地，一些数据库管理系统支持 `LIMIT ... OFFSET`：

```sql
--  PostgreSQL (再次), MariaDB/MySQL, SQLite
SELECT
id, title, published, author,
price, tax, inc,
row_number() OVER(ORDER BY id) AS row_number
ORDER BY id LIMIT 20 OFFSET 40;
```

这是一种更简单的语法，但不幸的是，它不是官方语法。

MSSQL 也支持简单的 `SELECT TOP` 语法，但它不够灵活。

当然，这两种替代方法比使用窗口函数技术简单得多，但使用窗口函数有一个优势。

假设你正在按某个非唯一字段排序，比如 `price`。普通的分页技术（包括前面的 `row_number()`）的问题在于，页面严格在行数处停止（如果没有更多数据，则可能更少）。

如果你想将相同价格的记录分在一组，可以改用类似下面的方法：

```sql
WITH cte AS (
SELECT
id, title, published, author,
price, tax, inc,
rank() OVER(ORDER BY price) AS rank
FROM aupricelist
)
SELECT *
FROM cte
WHERE rank BETWEEN 40 AND 59
ORDER BY price;
```

只要分组不是太大，它应该能给你几乎相同的结果，但同一价格的所有书籍都会被归在一起。

#### 使用 ntile

如果你想将排序后的结果集分成，比如，十个组，我们称这些组为 `十分位数`，源自拉丁语中表示“十”的词。如果你想要五个组，则称为 `五分位数`，一百个组就是 `百分位数`。如果你懂足够多的拉丁语，你还可以继续分成七组或十三组。

数学家对任何数字都有一个通用名称，称为 `n`，一旦你习惯了，这个名字相当吸引人。如果你将排序后的数据分成组，你就创建了 `ntiles`，对应的窗口函数是 `ntile(n)`，其中 `n` 是组的数量。

例如，要在客户表中按身高创建十分位数，你可以使用：

```sql
SELECT
id, givenname, familyname, height,
ntile(10) OVER (order by height) AS decile
FROM customers
WHERE height IS NOT NULL;
```

你会得到类似这样的结果：

| **id** | **givenname** | **familyname** | **height** | **decile** |
| --- | --- | --- | --- | --- |
| 597 | 艾克 | 安迪 | 153 | 1 |
| 283 | 埃塞尔 | 格利科尔 | 153 | 1 |
| 451 | 弗雷德 | 诺特 | 153.8 | 1 |
| 194 | 罗德 | 菲什 | 154.3 | 1 |
| 534 | 明妮 | 巴斯 | 156.4 | 1 |
| 352 | 巴兹尔 | 伊斯克 | 156.4 | 1 |
| ~ 约 267 行 ~ |

注意我们已经过滤掉了 `NULL` 身高值。如果我们没有过滤，那么根据你的数据库管理系统，第一个或最后一个十分位左右的组里会充斥着 `NULL` 身高值。这会创建一个本不应存在的组，但无论如何被包括进来了。

这只是 `ntile()` 的一个陷阱。实际上有两个陷阱，其中一个可能是决定性的障碍。

首先，请注意前面的结果有 267 行，这不能被 10 整除。这没关系，但 SQL 必须处理这种情况，你会发现前七个组会有 27 行，其余组有 26 行。当然，你自己的结果可能不同，但思路是一样的：余下的行会从前面开始填充。

第二个陷阱可能需要仔细寻找，并且在你自己的示例数据库中可能不明显。如果你仔细查找，可能会发现类似这样的情况：

| **id** | **givenname** | **familyname** | **height** | **decile** |
| --- | --- | --- | --- | --- |
| … |
| 388 | 罗恩 | 德莱 | 166.9 | 3 |
| 546 | 帕特 | 埃拉 | 167.1 | 3 |
| 106 | 杰伊 | 沃克 | 167.1 | 3 |
| 77 | 林恩 | 希德 | 167.1 | 4 |
| 403 | 威尔 | 诺特 | 167.3 | 4 |
| 314 | 杰克 | 波茨 | 167.4 | 4 |
| … |

在这个例子中，你会看到三位客户身高相同（`167.1`），但其中一位没能挤进前一个十分位数，所以被推到了下一个组。这更多是前面提到的不公平性，是由于 `ntile` 的计算纯粹基于行号和值。

例如，如果你要根据某些十分位数给客户颁奖或折扣，仅仅因为排序顺序不可预测而遗漏某人，这是不公平的。

如果你依赖 `ntile`，这可能是一个决定性的障碍。不过，有一个解决方法。



#### 针对 `ntile` 的一个变通方案

正如我们提到的，`ntile` 是基于行号的。然而，如果 `ntile` 能基于 `rank()`、`count()` 甚至是 `dense_rank()`，那么具有相同值的行最终就会被分到同一个十分位组中。

在这种情况下，我们将生成二十个组，称为二十分位组。为此，我们必须计算自己的分组。我们从计算每个组的大小开始：

```sql
SELECT count(*)/20.0 AS bin
FROM customers WHERE height IS NOT NULL
```

我们将这个值称为 `bin`，这是统计学中对分组的常见命名。

我们可以把它放到一个 CTE（公共表表达式）中，然后运行以下查询：

```sql
--  PostgreSQL, MariaDB/MySQL, MSSQL, Oracle
WITH data AS (
SELECT count(*)/20.0 AS bin
FROM customers WHERE height IS NOT NULL
)
SELECT
id, givenname, familyname, height,
row_number() OVER(ORDER BY height) AS row_number,
ntile(20) OVER(ORDER BY height) AS vigintile,
floor((row_number() OVER(ORDER BY height)-1)/bin)+1
AS row_vitintile,
floor((rank() OVER(ORDER BY height)-1)/bin)+1
AS rank_vigintile,
floor((count(*) OVER(ORDER BY height)-1)/bin)+1
AS count_vigintile,
bin
FROM customers, data
WHERE height IS NOT NULL
ORDER BY height;
```

SQLite 没有 `floor()` 函数，但你可以使用 `cast(... AS int)` 作为替代：

```sql
cast((row_number() OVER(ORDER BY height)-1)/bin AS int)+1
AS row_vigintile,
cast((rank() OVER(ORDER BY height)-1)/bin AS int)+1
AS rank_vigintile,
cast((count(*) OVER(ORDER BY height)-1)/bin AS int)+1
AS count_vigintile,
```

你将得到以下结果：

| id   | …    | height | rn   | vig  | row_vig | rank_vig | count_vig | bin    |
| :--- | :--- | :----- | :--- | :--- | :------ | :------- | :-------- | :----- |
| 597  | …    | 153    | 1    | 1    | 1       | 1        | 1         | 13.35  |
| 283  | …    | 153    | 2    | 1    | 1       | 1        | 1         | 13.35  |
| 451  | …    | 153.8  | 3    | 1    | 1       | 1        | 1         | 13.35  |
| 194  | …    | 154.3  | 4    | 1    | 1       | 1        | 1         | 13.35  |
| 534  | …    | 156.4  | 5    | 1    | 1       | 1        | 1         | 13.35  |
| 352  | …    | 156.4  | 6    | 1    | 1       | 1        | 1         | 13.35  |
| ~ 共 267 行 ~ |

注意，`vigintile` 和 `row_vigintile` 的值应该是相同的；`row_vigintile` 用于展示二十分位组是如何从行号计算出来的。

更重要的是，你会看到 `rank_vigintile` 和 `count_vigintile` 列是根据 `rank()` 和 `count(*)` 值计算的，它们总是将具有相同身高的行放在同一个组中。选择哪种方式取决于你。

## 处理前后行数据

在处理有序结果集的同时，我们也可以获取前一行和后一行的数据。这些结果分别称为 `lag`（滞后）和 `lead`（超前）。

该函数的通用语法是：

```sql
lead(column,number) OVER (...)
lag(column,number) OVER (...)
```

在这里，除了 `OVER` 子句，我们还需要提供两个值。`column` 值指的是你想要获取的另一行中的哪一列数据。`number` 值指的是要向后或向前获取多少行。如果需要，你可以省略它，这种情况下它将默认为 1。

例如，假设你想查看每天的销售额，以及前一天和后一天的销售额。你可以这样写：

```sql
SELECT
ordered_date, daily_total,
lag(daily_total) OVER (ORDER BY ordered_date)
AS previous,
lead(daily_total) OVER (ORDER BY ordered_date)
AS next
FROM daily_sales
ORDER BY ordered_date;
```

你会看到：

| ordered_date | daily_total | previous | next   |
| :----------- | :---------- | :------- | :----- |
| 2022-05-04   | 43          | [NULL]   | 150.5  |
| 2022-05-05   | 150.5       | 43       | 110.5  |
| 2022-05-06   | 110.5       | 150.5    | 142    |
| 2022-05-07   | 142         | 110.5    | 214.5  |
| 2022-05-08   | 214.5       | 142      | 16.5   |
| 2022-05-09   | 16.5        | 214.5    | 160    |
| ~ 共 388 行 ~ |

你会注意到，第一行的 `previous` 是 `NULL`；最后一行的 `next` 也是 `NULL`。

你可能会认为这有点没意义，因为你只需移动视线就能查看上一行或下一行。然而，你也可以将滞后或超前函数纳入计算中。例如，假设你想将每天的销售额与一周前进行比较。你可以使用：

```sql
SELECT
ordered_date, daily_total,
lag(daily_total,7) OVER (ORDER BY ordered_date)
AS last_week,
daily_total
- lag(daily_total,7) OVER (ORDER BY ordered_date)
AS difference
FROM daily_sales
ORDER BY ordered_date;
```

结果是：

| ordered_date | daily_total | last_week | difference |
| :----------- | :---------- | :-------- | :--------- |
| 2022-05-04   | 43          | [NULL]    | [NULL]     |
| 2022-05-05   | 150.5       | [NULL]    | [NULL]     |
| 2022-05-06   | 110.5       | [NULL]    | [NULL]     |
| 2022-05-07   | 142         | [NULL]    | [NULL]     |
| 2022-05-08   | 214.5       | [NULL]    | [NULL]     |
| 2022-05-09   | 16.5        | [NULL]    | [NULL]     |
| 2022-05-10   | 160         | [NULL]    | [NULL]     |
| 2022-05-11   | 115         | 43        | 72         |
| 2022-05-12   | 205         | 150.5     | 54.5       |
| 2022-05-13   | 164.5       | 110.5     | 54         |
| 2022-05-14   | 46.5        | 142       | -95.5      |
| 2022-05-15   | 457.5       | 214.5     | 243        |
| ~ 共 388 行 ~ |

在这里，表达式 `lag(total,7)` 获取的是前 7 行的值。正如你所料，前 7 行的值为 `NULL`。

要有效且有意义地使用 `lag` 或 `lead`，有两个重要条件：

*   对于每个你想测试的实例，必须只有一行数据。例如，你不能有两行具有相同的日期。
*   数据不能有间隔（缺失）。例如，不能有缺失的日期。

这是因为我们将每一行解释为一天。如果你只是处理一个序列或销售数据而不考虑日期，这就无关紧要了。

如果你仔细（且耐心地）浏览数据，会发现有几个缺失的日期。这意味着前一行并不总是“昨天”，前 7 行也不总是“上周”。我们将在第 9 章中看到如何填补这些间隔。

