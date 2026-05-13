# 日期计算

日期计算在不同的 DBMS 之间差异是出了名的大。除此之外，日期还可能包含时间部分；通常这被称为“日期时间”或“时间戳”。在这里，我们简单地称其为“日期”。

对于大多数操作，DBMS 倾向于依赖函数。

通常，你想对日期做的三件事是：

-   简单计算：计算日期之间的差异，并将日期偏移一定量。
-   提取日期的部分，如月份或年份。
-   格式化日期。

以下是你可以对日期执行的主要计算的概述。

SQLite 处理日期的方法完全不同。这部分是因为它实际上不支持日期。因此，接下来的大部分讨论中都不会涉及 SQLite。附录 3（补充说明）包含一些关于在 SQLite 中处理日期的信息。

## 简单计算

一个你需要获取的重要值是当前日期和时间。在大多数 DBMS 中，你可以使用 `current_timestamp`：

```sql
SELECT current_timestamp;   --  Oracle: FROM dual;
```

一些 DBMS 还有各种函数来获取相同的结果，例如 MariaDB 的 `now()` 或 MSSQL 的 `getdate()`。但是，它们给出的结果相同。SQLite 没有直接的版本，因为它不原生支持日期；不过，需要时，字符串 `'now'` 可以完成这项工作。

从这里开始，情况变得复杂。以下是一些添加 4 个月的例子：

```sql
--  PostgreSQL
SELECT
date '2015-10-31' + interval '4 months',
current_timestamp + interval '4 months',
current_timestamp + interval '4' month  --  相同
;
--  MariaDB/MySQL
SELECT
date_add('2015-10-31',interval 4 month),
date_add(current_timestamp,interval 4 month),
current_timestamp + interval '4' month  --  相同
;
--  MSSQL
SELECT
dateadd(month,4,'2015-10-31'),
dateadd(month,4,current_timestamp)
;
--  Oracle
SELECT
add_months('31 Oct 2015',4),
current_timestamp + interval '4' month,
add_months(current_timestamp,4) --  同样有效
FROM dual;
--  SQLite
SELECT
strftime('%Y-%m-%d','2015-10-31','+4 month'),
strftime('%Y-%m-%d','now','+4 month')
;
```

根据运行代码的时间，你会得到类似这样的结果：

| 指定日期 | 当前时间戳 |
| --- | --- |
| 2016-02-29 00:00:00 | 2023-07-28 13:15:29.066381+10 |

这里重要的是，前面所有的例子都足够智能，可以处理不同长度的月份；添加 4 个月可能意味着从 120 天到 123 天不等，但前面的计算会为此进行调整。但是，如果结果超出了当月最后一天，除了 SQLite，其他都会限制在当月最后一天；SQLite 会进入下一个月。



### 年龄计算

一个重要的计算是找出两个日期之间的差值，例如计算年龄。这里的问题在于，真实结果通常不是整年（或整月，或你所测量的单位）。例如，PostgreSQL 提供了 `age()` 函数：

```sql
--  PostgreSQL
SELECT id, givenname, familyname, dob, age(dob)
FROM customers;
```

这将给出年龄：

| id | givenname | familyname | dob | age |
| --- | --- | --- | --- | --- |
| 474 | Judy | Free | 1978-04-01 | 44 years 11 mons 2 days |
| 186 | Ray | Gunn |   |   |
| 144 | Ray | King |   |   |
| 179 | Ivan | Inkling |   |   |
| 475 | Drew | Blood | 1989-12-06 | 33 years 2 mons 28 days |
| 523 | Seymour | Sights | 1965-01-06 | 58 years 1 mon 28 days |
| ~ 304 rows ~ |

然而，结果类似于 `38 years 6 mons 7 days`，如果你真的需要那样的细节，这已经相当不错了。得到的表达式被称为 `interval`（时间间隔）。时间间隔是 PostgreSQL 中日期计算的重要组成部分。

但大多数时候，你可能只需要年数（或月数等），因此以下计算可能更合适：

```sql
--  PostgreSQL
SELECT
id, givenname, familyname, dob,
age(dob) AS interval,
extract(year from age(dob)) AS samething
FROM customers;

--  MariaDB/MySQL
SELECT
id, givenname, familyname, dob,
timestampdiff(year,dob,current_timestamp) AS age
FROM customers;

--  MSSQL
SELECT
id, givenname, familyname, dob,
datediff(year,dob,current_timestamp)
AS age --  但不完全准确！
FROM customers;

--  Oracle
SELECT
id, givenname, familyname, dob,
trunc(months_between(current_timestamp,dob)/12)
AS age
FROM customers;

--  SQLite
SELECT
id, givenname, familyname, dob,
cast(
strftime('%Y.%m%d', 'now')
- strftime('%Y.%m%d', dob)
as int) AS age
FROM customers;
```

你会得到类似这样的结果。

| id | givenname | familyname | dob | years |
| --- | --- | --- | --- | --- |
| 474 | Judy | Free | 1978-04-01 | 44 |
| 186 | Ray | Gunn |   |   |
| 144 | Ray | King |   |   |
| 179 | Ivan | Inkling |   |   |
| 475 | Drew | Blood | 1989-12-06 | 33 |
| 523 | Seymour | Sights | 1965-01-06 | 58 |
| ~ 304 rows ~ |   |   |   |   |

请注意，只有 PostgreSQL 有内置函数来计算年龄。Oracle 有 `months_between()` 函数，几乎能完成这项工作；这个数字除以 12，然后 `trunc()` 函数会去掉余数。

在上述计算中，MSSQL 有一个简单的函数，但过于简单。它只是计算年份之间的差异，如果出生日期在年末而查询日期在年初，结果就会偏差很大。要得到更正确的结果需要更多的工作。

### 提取日期的部分内容

日期的另一项技术是提取其组成部分，例如日或年。这里再次体现，不同的 DBMS 差异很大。

SQLite 处理日期部分的方式完全不同，此处不予讨论。

#### 在 PostgreSQL、MariaDB/MySQL 和 Oracle 中提取日期

提取日期部分的标准方法是使用 `extract()` 函数。该函数的形式为

```
extract(part from datetime)
```

你可以看到 `extract()` 函数的实际应用：

```sql
WITH moonshot AS (
SELECT
timestamp '1969-07-20 20:17:40' AS datetime
--  FROM dual   --  (Oracle)
)
SELECT
datetime,
EXTRACT(year FROM datetime) AS year,        --  1969
EXTRACT(month FROM datetime) AS month,      --  7
EXTRACT(day FROM datetime) AS day,          --  20
--  非 Oracle 或 MariaDB/MySQL:
EXTRACT(dow FROM datetime) AS weekday,  --  0
EXTRACT(hour FROM datetime) AS hour,        --  20
EXTRACT(minute FROM datetime) AS minute,    --  17
EXTRACT(second FROM datetime) AS second     --  40
FROM moonshot;
```

请注意，Oracle 和 MariaDB/MySQL 没有直接提取星期几的方法，如果你希望将其用于分组，这可能是个问题。然而，正如你将在后面看到的，你可以使用格式化函数来获取星期几以及前面的值。

PostgreSQL 还提供了一个名为 `date_part('part',datetime)` 的函数，作为上述函数的替代。

日期/时间 `1969-07-20 20:17:40` 是人类首次踏上月球的时间。

#### 在 Microsoft SQL 中提取日期

Microsoft SQL 有两个主要函数来提取日期的部分：

*   `datepart(part,datetime)` 以数字形式提取日期/时间的组成部分。
*   `datename(part,datetime)` 以字符串形式提取日期/时间的组成部分。对于大多数组成部分，如年份，它只是 `datepart` 数字的字符串版本。然而，对于星期几和月份，它实际上是人类可读的名称。

你可以看到这两个函数的实际应用：

```sql
WITH moonshot AS (
SELECT cast('1969-07-20 20:17:40' as datetime) AS datetime
)
SELECT
datepart(year, datetime) AS year,       --  又名 year()
datename(year, datetime) AS yearstring,
datepart(month, datetime) AS month,     --  又名 month()
datename(month, datetime) AS monthname,
datepart(day, datetime) AS day,         --  又名 day()
datepart(weekday, datetime) AS weekday, --  Sunday=1
datename(weekday, datetime) AS weekdayname,
datepart(hour, datetime) AS hour,
datepart(minute, datetime) AS minute,
datepart(second, datetime) AS second
FROM moonshot;
```

注意

*   `datename(date,year)` 只给出 `2013` 的字符串版本。
*   有三个简短函数 `day()`、`month()` 和 `year()`，它们是 `datepart()` 的同义词。

#### 从日期时间中提取日期

一个明显缺失的功能是从日期时间中提取日期。最直接的方法是进行 `cast`（类型转换）：

```sql
WITH moonshot AS (
--  …
)
SELECT cast(thetime AS DATE) AS thedate --  1969-07-20 (非 Oracle)
FROM moonshot;
```

对于 Oracle，你需要一个稍微不同的方法。转换为日期类型实际上不会移除时间部分。相反，你应该使用 `trunc(thetime)`；它看起来不完全一样，但能起作用。

在我们后面尝试分析包含日期和时间的数据时，你将需要这种技术。

### 格式化日期

格式化日期就是以一种有用或友好的方式呈现它。通常，这涉及生成一个字符串，因为字符串是唯一能控制输出哪些字符的方式。

与数字类似，格式化的日期不再是日期。这意味着如果你需要做任何进一步的计算或排序，可能会遇到问题。

#### 在 PostgreSQL 和 Oracle 中格式化日期

对于 PostgreSQL 和 Oracle，你都可以使用 `to_char` 函数。这里有两个有用的格式：

```sql
--  PostgreSQL
WITH vars AS (
SELECT timestamp '1969-07-20 20:17:40' AS moonshot
)
SELECT
moonshot,
to_char(moonshot,'FMDay, DDth FMMonth YYYY') AS full,
to_char(moonshot,'Dy DD Mon YYYY') AS short
FROM vars;

--  Oracle
WITH vars AS (
SELECT timestamp '1969-07-20 20:17:40' AS moonshot
FROM dual
)
SELECT
moonshot,
to_char(moonshot,'FMDay, ddth Month YYYY')
AS fulldate,
to_char(moonshot,'Dy DD Mon YYYY') AS shortdate
FROM vars;
```

这应该给你以下结果：

| moonshot | fulldate | shortdate |
| --- | --- | --- |
| 1969-07-20 20:17:40 | Sunday, 20th July 1969 | Sun 20 Jul 1969 |

你会注意到 PostgreSQL 和 Oracle 之间的格式代码略有不同。

你可以在以下链接了解更多关于格式代码的信息：

*   PostgreSQL: [`www.postgresql.org/docs/current/functions-formatting.html#FUNCTIONS-FORMATTING-DATETIME-TABLE`](http://www.postgresql.org/docs/current/functions-formatting.html%23FUNCTIONS-FORMATTING-DATETIME-TABLE)
*   Oracle: [`https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/Format-Models.html`](https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/Format-Models.html%5D)



