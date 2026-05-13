# SQL 中的字符串与日期处理

## 尾部空格

如果您在搜索字符串的**末尾**添加一个空格：

```sql
SELECT *
FROM customers
WHERE state='VIC '; -- 末尾有额外空格
```

根据数据库管理系统（DBMS）的不同，您**可能**会得到一些结果：

*   MySQL/MariaDB 以及 MSSQL 会修剪尾部空格，因此您将获得所有匹配 `VIC` 的结果。
*   PostgreSQL、SQLite 和 Oracle **不会**修剪空格，因此将没有匹配项。

相反，如果您在开头添加额外的空格：

```sql
SELECT *
FROM customers
WHERE state=' VIC'; -- 开头有额外空格
```

您将得不到匹配项。

尽管 SQL 通常只接受字符串的精确匹配，但 SQL 标准要求在比较较短的字符串与较长的字符串之前，用空格对较短的字符串进行右填充。因此，如果您尝试匹配末尾带有额外空格的字符串，数据也会被右填充，从而得到一个匹配项。这不适用于左填充或任何其他字符。

在流行的 DBMS 中，只有 MySQL/MariaDB 和 MSSQL 似乎遵循此标准。

## 使用字符串函数过滤

您已经看到，根据 DBMS 的不同，您可能需要在 `WHERE` 子句中使用 `lower()` 函数。

您可以在 `WHERE` 子句中使用任何您喜欢的函数。例如，要选择较短的名字，您可以使用 `length()` 函数：

```sql
-- PostgreSQL, MySQL/MariaDB, SQLite, Oracle
SELECT *
FROM customers
WHERE length(familyname)<5;
-- MSSQL
SELECT *
FROM customers
WHERE len(familyname)<5;
```

这将给出较短的姓名：

| id | familyname | givenname | … | registered |
| --- | --- | --- | --- | --- |
| 474 | Free | Judy | … | 2022-06-12 |
| 186 | Gunn | Ray | … | 2021-11-15 |
| 144 | King | Ray | … | 2021-10-18 |
| 351 | Tate | Dick | … | 2022-03-13 |
| 422 | Why | Wanda | … | 2022-05-05 |
| 191 | Moss | Pete | … | 2021-11-19 |
| ~ 135 rows ~ |   |   |   |   |

也有用于提取字符串部分的字符串函数，但如果您是为了比较值而这样做，稍后使用通配符可能会获得更多收益。

如果字符串的长度重要到需要过滤，那么它可能也重要到需要选择：

| id | email | familyname | givenname |
| --- | --- | --- | --- |
| 474 | judy.free474@example.net | Free | Judy |
| 186 | ray.gunn186@example.net | Gunn | Ray |
| 144 | ray.king144@example.net | King | Ray |
| 351 | dick.tate351@example.com | Tate | Dick |
| 422 | wanda.why422@example.com | Why | Wanda |
| 191 | pete.moss191@example.com | Moss | Pete |
| ~ 135 rows ~ |

```sql
-- PostgreSQL, MySQL/MariaDB, SQLite, Oracle
SELECT *, length(familyname) AS size
FROM customers
WHERE length(familyname)<5;
-- MSSQL
SELECT *, len(familyname)<5 AS size
FROM customers
WHERE len(familyname)<5;
```

请记住，`SELECT` 子句是在 `WHERE` 子句**之后**计算的，这意味着您不能在 `WHERE` 子句中使用计算出的别名：

```sql
-- PostgreSQL, MySQL/MariaDB, SQLite, Oracle
SELECT *, length(familyname) AS size
FROM customers
WHERE size<5;   -- 错误
-- MSSQL
SELECT *, len(familyname)<5 AS size
FROM customers
WHERE size<5;   -- 错误
```

您将在后面的章节中看到更多关于计算和函数的内容。

## 处理引号与撇号

您的字符串数据可能包含单引号，尤其是用作撇号时。例如，您的姓氏可能是 `O'Shea`，或者您的家乡可能是 `'s-Gravenhage`（海牙的正式名称）。

如果您尝试在正常的单引号字符串中输入它们，将会遇到问题：

```sql
-- 这是错误的：
SELECT *
FROM customers
WHERE familyname = 'O'Shea'
OR town=''s-Gravenhage';
```

字符串中的单引号会提前结束字符串，从而导致语句的其余部分变得混乱。

如果需要包含单引号，您需要输入单引号**两次**（这**不**等同于双引号）：

```sql
-- 字符串内的 '' 被解释为 '
-- '' 不等同于 "
SELECT *
FROM customers
WHERE familyname = 'O''Shea' OR town='''s-Gravenhage';
```

更好的做法是，您的数据应该使用印刷体撇号：

```sql
-- 使用'印刷体'引号：
SELECT *
FROM customers
WHERE familyname = 'O’Shea' OR town='’s-Gravenhage';
```

您可以通过以下方式输入印刷体撇号：

*   在 Macintosh 上按 Shift+Option+]
*   在 Windows 上按 Alt+0146

当然，这只有在数据最初是以这种方式输入的情况下才有效，不幸的是，这种情况并不常见。

## 字符串的前后比较

其他比较运算符也适用于字符串，但您应该从它们在字母顺序中的位置来考虑，如表 3-2 所示。

表 3-2
字符串比较运算符

| 运算符 | 含义 |
| --- | --- |
| a < b | a 在 b **之前** |
| a <= b | a **直到** b |
| a > b | a 在 b **之后** |
| a >= b | a **自** b 起 |

例如：

| id | familyname | givenname | email | … | registered |
| --- | --- | --- | --- | --- | --- |
| 474 | Free | Judy | judy.free474@example.net | … | 2022-06-12 |
| 186 | Gunn | Ray | ray.gunn186@example.net | … | 2021-11-15 |
| 179 | Inkling | Ivan | ivan.inkling179@example.com | … | 2021-11-08 |
| 475 | Blood | Drew | drew.blood475@example.net | … | 2022-06-13 |
| 341 | Idate | Val | val.idate341@example.com | … | 2022-03-03 |
| 234 | Ering | Nat | nat.ering234@example.net | … | 2021-12-15 |
| ~ 137 rows ~ |

```sql
-- K 之前的姓名
SELECT *
FROM customers
WHERE familyname<'K';
```

您不会经常看到这种比较。稍后当我们查看通配符时，会看到一种更灵活的过滤这些字符串的方法。

### 日期

日期看起来足够简单，但可能导致混淆和困难。首先，术语“日期”可能包含也可能不包含时间。

广义上讲，时间是历史中的一个点，从过去的某个任意起点开始测量。为了方便，时间被分组为秒、分钟、小时和天。之后发生的事情就变得更复杂了。

您可以使用如下语句找到具有特定出生日期的所有客户：

```sql
SELECT *
FROM customers
WHERE dob='1989-11-09';
```

这将给出与出生日期匹配的客户：

| id | familyname | givenname | … | dob | … | registered |
| --- | --- | --- | --- | --- | --- | --- |
| 320 | Branch | Olive | … | 1989-11-09 | … | 2022-02-10 |
| 568 | Peace | Warren | … | 1989-11-09 | … | 2022-08-08 |
| ~ 2 rows ~ |

您可能会找到两到三个匹配项。

Oracle 默认使用不同的日期格式，可能无法自动解释前面的格式。您可能需要使用表达式 `date '1989-11-09'`。

如您所知，在 SQL 中，您用单引号将字符串括起来。您**也**用单引号将日期括起来。然而，日期**不是**字符串。^(⁵)

如果您在 Oracle 中尝试此操作，可能会发现 Oracle 不喜欢这种日期格式。默认情况下，Oracle 更喜欢 `09-NOV-89` 这样的默认格式。

但是，您可以使用 `date` 前缀强制 Oracle 识别前面的格式：

```sql
-- Oracle
SELECT *
FROM customers
WHERE dob = date '1989-11-09';
```

稍后您将看到日期字面量的其他一些变体。

### 日期不是字符串

尽管日期字面量用单引号书写，但它们不是字符串。当您尝试添加额外的空格时，您可以立即看到这一点：

```sql
SELECT *
FROM customers
WHERE dob=' 1989-11-09 ';
```

字符串匹配在这里会失败，但日期仍然匹配。您也可以在下一节关于日期格式的内容中看到这一点。


### 替代日期格式

推荐的格式是 `ISO 8601` 格式，这是一个描述日期、时间和其他相关数据的标准。对于日期，格式为 `yyyy-mm-dd`，正如你在之前的示例中所见。

ISO 8601 格式也允许你省略连字符：

```sql
SELECT *
FROM customers
WHERE dob='19891109';       --  等同于 '1989-11-09'
```

然而，这样更难阅读，因此很难让人接受。这种格式在 Oracle 或 SQLite 中无效。

某些数据库管理系统允许你使用替代格式：

```sql
--  仅限 PostgreSQL, MSSQL, MySQL/MariaDB:
SELECT *
FROM customers
WHERE dob='9 Nov 1989';
SELECT *
FROM customers
WHERE dob='November 9, 1989';
```

但是，即使可用，也不要使用斜杠格式 `??/??/yyyy`。这是因为不同国家对前两部分的解读方式不同，有些国家解读为 `日/月`，有些则解读为 `月/日`。

```sql
--  仅限 PostgreSQL, MSSQL, MySQL/MariaDB:
SELECT *
FROM customers
WHERE dob='9/11/1989';  --  d/m 还是 m/d？
```

对于 SQL：

*   数据库管理系统可能不认同你的解读方式。
*   其他用户可能不确定你如何解读这两个部分。

我们建议你在可能的情况下始终使用 ISO 8601 格式来书写日期。另一方面，在显示日期时，使用更人性化的格式可能更好。稍后你将看到如何做到这一点。

### 日期比较

除了两个日期相同之外，你还可以进行与数字类似的比较。不过，最好像表 3-3 那样重新表述它们的含义。

**表 3-3** 重新表述的比较运算符

| 运算符 | 含义 |
| --- | --- |
| `a = b` | 值相等 |
| `a < b` | a **在** b **之前** |
| `a <= b` | a **截至** b |
| `a > b` | a **在** b **之后** |
| `a >= b` | a **自** b **起** |

当然，实际使用的词并不重要；它们只是更具意义。

例如：

```sql
--  出生日期在 1980 年 1 月 1 日 之前
SELECT *
FROM customers
WHERE dob<'1980-01-01';
```

你将获得较年长的客户：

| id | familyname | givenname | … | dob | … | registered |
| --- | --- | --- | --- | --- | --- | --- |
| 474 | Free | Judy | … | 1978-04-01 | … | 2022-06-12 |
| 523 | Sights | Seymour | … | 1965-01-06 | … | 2022-07-11 |
| 341 | Idate | Val | … | 1976-06-04 | … | 2022-03-03 |
| 351 | Tate | Dick | … | 1969-08-03 | … | 2022-03-13 |
| 121 | Ting | Lil | … | 1964-09-17 | … | 2021-10-06 |
| 545 | Knife | Jack | … | 1962-09-24 | … | 2022-07-24 |
| ~ 96 行 ~ |

```sql
--  出生日期在 1980 年 1 月 1 日 起
SELECT *
FROM customers
WHERE dob>='1980-01-01';
```

那些不那么年长的：

| id | familyname | givenname | … | dob | … | registered |
| --- | --- | --- | --- | --- | --- | --- |
| 475 | Blood | Drew | … | 1989-12-06 | … | 2022-06-13 |
| 588 | Skies | Grace | … | 1999-06-28 | … | 2022-08-13 |
| 422 | Why | Wanda | … | 1999-07-15 | … | 2022-05-05 |
| 326 | Todeath | Boris | … | 1992-06-16 | … | 2022-02-15 |
| 191 | Moss | Pete | … | 1995-09-27 | … | 2021-11-19 |
| 234 | Ering | Nat | … | 1996-02-05 | … | 2021-12-15 |
| ~ 139 行 ~ |

和往常一样，`NULL` 日期将从结果中省略：如果你不知道出生日期，就无法声称他们出生在某个特定日期之前或之后。

你也可以使用 `BETWEEN`：

```sql
--  出生在 1980 年代
SELECT *
FROM customers
WHERE dob BETWEEN '1980-01-01' AND '1989-12-31';
```

1980 年代出生的孩子们：

| id | familyname | givenname | … | dob | … | registered |
| --- | --- | --- | --- | --- | --- | --- |
| 475 | Blood | Drew | … | 1989-12-06 | … | 2022-06-13 |
| 492 | Long | Miles | … | 1989-11-18 | … | 2022-06-21 |
| 468 | Fer | Connie | … | 1985-09-22 | … | 2022-06-09 |
| 86 | Byrd | Dicky | … | 1980-06-02 | … | 2021-09-09 |
| 75 | Tone | Barry | … | 1989-07-18 | … | 2021-09-01 |
| 306 | Noir | Bette | … | 1987-08-27 | … | 2022-01-28 |
| ~ 59 行 ~`

如果你想获取其他时间段的：

```sql
--  出生在其他时间
SELECT *
FROM customers
WHERE dob NOT BETWEEN '1980-01-01' AND '1989-12-31';
```

你会得到：

| id | familyname | givenname | … | dob | … | registered |
| --- | --- | --- | --- | --- | --- | --- |
| 474 | Free | Judy | … | 1978-04-01 | … | 2022-06-12 |
| 523 | Sights | Seymour | … | 1965-01-06 | … | 2022-07-11 |
| 341 | Idate | Val | … | 1976-06-04 | … | 2022-03-03 |
| 351 | Tate | Dick | … | 1969-08-03 | … | 2022-03-13 |
| 588 | Skies | Grace | … | 1999-06-28 | … | 2022-08-13 |
| 422 | Why | Wanda | … | 1999-07-15 | … | 2022-05-05 |
| ~ 176 行 ~ |

注意 `BETWEEN` 是包含性的：范围的起始和结束日期也匹配。

还要注意，在所有情况下，`NULL` 出生日期都会被省略。

### 使用日期计算进行过滤

与字符串类似，你可以使用日期计算来过滤结果。例如，要查找年龄超过 40 岁的客户（其出生日期在 40 年前之前）：

```sql
--  PostgreSQL, Oracle, MySQL/MariaDB
SELECT * FROM customers
WHERE dob<current_timestamp -   INTERVAL '40' YEAR;
--  MSSQL
SELECT * FROM customers
WHERE dob<dateadd(year,-40,current_timestamp);
--  SQLite
SELECT * FROM customers
WHERE dob<date('now','-40 year');
```

这会给出年龄超过 40 岁的客户：

| id | familyname | givenname | … | dob | … | registered |
| --- | --- | --- | --- | --- | --- | --- |
| 474 | Free | Judy | … | 1978-04-01 | … | 2022-06-12 |
| 523 | Sights | Seymour | … | 1965-01-06 | … | 2022-07-11 |
| 341 | Idate | Val | … | 1976-06-04 | … | 2022-03-03 |
| 351 | Tate | Dick | … | 1969-08-03 | … | 2022-03-13 |
| 121 | Ting | Lil | … | 1964-09-17 | … | 2021-10-06 |
| 545 | Knife | Jack | … | 1962-09-24 | … | 2022-07-24 |
| ~ 111 行 ~`

稍后你会了解更多关于日期计算的内容。

## 多重断言

之前的 `BETWEEN` 操作也可以写成：

```sql
--  出生在 1980 年代
SELECT *
FROM customers
WHERE dob>='1980-01-01' AND dob<='1989-12-31';
```

现在有两个断言：`dob >= '1980-01-01'` 和 `dob <= '1989-12-31'`，两者都必须为真。

这将给出相同的结果，并且很可能在内部，SQL 执行了相同的操作。有时，SQL 会理解你的意思并找到自己的执行方式。


### AND 与 OR

`AND` 操作符通常被称为逻辑操作符，并遵循数学逻辑的规则。

使用 `AND` 操作符，你还可以实现 `BETWEEN` 的变体：

```sql
--  包含性范围（与 BETWEEN 相同）
SELECT *
FROM artists
WHERE born>=1700 AND born<=1799;
```

所有这些查询应该给出相同的结果：

| id | 姓氏 | 名字 | 别名 | 出生年份 | 去世年份 | 国籍 |
| --- | --- | --- | --- | --- | --- | --- |
| 133 | Constable | John | constable | 1776 | 1837 | English |
| 158 | Shaw | Joshua | shaw | 1776 | 1860 | American |
| 78 | Turner | Joseph Mallord William | turner | 1775 | 1851 | English |
| 298 | Hiroshige | Ando | hiroshige | 1797 | 1858 | Japanese |
| 356 | Gros | Antoine-Jean | gros | 1771 | 1835 | French |
| 208 | Feke | Robert | feke | 1705 | 1752 | American |
| ~ 25 行 ~ |

注意，由于出生年份是离散值，你可以选择如何表达范围。

`AND` 操作符可以用来组合两个以上的断言。例如：

```sql
SELECT *
FROM customers
WHERE state='VIC' AND height>170 AND dob<'1980-01-01';
```

这会得到一个非常有限的组：

| id | 姓氏 | 名字 | 州 | 身高 | 出生日期 |
| --- | --- | --- | --- | --- | --- |
| 505 | Singers | Carol | VIC | 170.1 | 1969-07-24 |
| 406 | Shoes | Jim | VIC | 173.5 | 1970-12-10 |
| 59 | Field | Lily | VIC | 172.1 | 1972-01-17 |
| 537 | Rise | Theo | VIC | 176.9 | 1977-10-27 |
| 300 | Bee | Bill | VIC | 171.3 | 1975-12-21 |
| 380 | Downe | Bob | VIC | 178.1 | 1976-02-24 |
| ~ 8 行 ~ |

在这种情况下，*所有*断言都必须为真。请注意，这里的三个断言彼此独立，不像前面测试同一个值的例子。

你也可以用 `OR` 操作符组合断言：

```sql
SELECT *
FROM customers
WHERE state='VIC' OR state='QLD';
```

这里，我们得到了来自这两个合并州的客户：

| id | 姓氏 | 名字 | 州 |
| --- | --- | --- | --- |
| 186 | Gunn | Ray | VIC |
| 475 | Blood | Drew | QLD |
| 523 | Sights | Seymour | VIC |
| 588 | Skies | Grace | QLD |
| 305 | Net | Clara | QLD |
| 121 | Ting | Lil | QLD |
| ~ 104 行 ~ |

`OR` 操作符要求*至少*有一个断言为真。

与英语不同，`OR` 有效地组合了集合。在英语中，你*可能*说客户来自 `VIC` *和* `QLD`，但我们通常理解为不是同时来自两地。在逻辑中，我们需要说 `OR`。

同样，与英语不同，逻辑 `OR` 总是**包含性**的：一个或多个断言必须为真。`OR` 操作失败的唯一情况是所有断言都为假。在英语中，“or”有时是**排他性**的（如“茶或咖啡”）；在逻辑中，情况并非如此。

当两个断言独立时，你会更清楚地看到这一点。例如：

```sql
--  所有条件都必须为真：
SELECT *
FROM customers
WHERE state='QLD' AND dob<'1980-01-01';
```

这会得到一个有限的组：

| id | 姓氏 | 名字 | 州 | 出生日期 |
| --- | --- | --- | --- | --- |
| 121 | Ting | Lil | QLD | 1964-09-17 |
| 377 | Money | Xavier | QLD | 1969-07-14 |
| 266 | Blind | Rob | QLD | 1965-12-23 |
| 524 | Syrup | Mabel | QLD | 1978-03-09 |
| 28 | Aphone | Meg | QLD | 1963-01-20 |
| 201 | Soar | Dinah | QLD | 1971-06-09 |
| ~ 16 行 ~ |

将 `AND` 改为 `OR`：

```sql
--  任一（或全部）条件必须为真：
SELECT *
FROM customers
WHERE state='QLD' OR dob<'1980-01-01';
```

这会得到一个更混合的组：

| id | 姓氏 | 名字 | 州 | 出生日期 |
| --- | --- | --- | --- | --- |
| 474 | Free | Judy | NSW | 1978-04-01 |
| 475 | Blood | Drew | QLD | 1989-12-06 |
| 523 | Sights | Seymour | VIC | 1965-01-06 |
| 341 | Idate | Val | NSW | 1976-06-04 |
| 351 | Tate | Dick | NSW | 1969-08-03 |
| 588 | Skies | Grace | QLD | 1999-06-28 |
| ~ 132 行 ~ |

注意
*   `OR` 比 `AND` 更宽松。
*   `OR` 包含了 `AND` 的所有结果。

你可以将 `AND` 视为更多的过滤，将 `OR` 视为组合结果。如果你熟悉数学集合，`AND` 是多个集合的**交集**（共同部分），而 `OR` 是多个集合的**并集**（全部组合）。

如果将 `AND` 与 `OR` 混合，事情会变得有点复杂：

```sql
--  操作符优先级
SELECT *
FROM customers
WHERE state='QLD' OR state='VIC' AND dob<'1980-01-01';
```

结果可能不符合你的预期，无论你预期什么：

| id | 姓氏 | 名字 | 州 | 出生日期 |
| --- | --- | --- | --- | --- |
| 475 | Blood | Drew | QLD | 1989-12-06 |
| 523 | Sights | Seymour | VIC | 1965-01-06 |
| 588 | Skies | Grace | QLD | 1999-06-28 |
| 305 | Net | Clara | QLD |   |
| 121 | Ting | Lil | QLD | 1964-09-17 |
| 545 | Knife | Jack | VIC | 1962-09-24 |
| ~ 70 行 ~ |

在算术表达式 `1 + 2 * 3` 中，你知道乘法在加法之前执行。也就是说，乘法优先于加法。

类似地，在逻辑表达式 `AssertionA OR AssertionB AND AssertionC` 中，`AND` 优先于 `OR`。这意味着前一条语句的结果等同于

```sql
--  等同于：state='QLD' OR state='VIC' AND dob<'1980-01-01'
SELECT *
FROM customers
WHERE state='QLD' OR (state='VIC' AND dob<'1980-01-01');
```

用英语说，意思是组合*一个州的全部*和*另一个州的年长者*。

如果你本意是让出生日期断言同时应用于两个州，你需要使用括号来改变优先级：

```sql
--  改变优先级
SELECT *
FROM customers
WHERE (state='QLD' OR state='VIC') AND dob<'1980-01-01';
```

现在你将得到来自这两个州的年长客户。

| id | 姓氏 | 名字 | 电子邮箱 | … | 注册日期 |
| --- | --- | --- | --- | --- | --- |
| 523 | Sights | Seymour | seymour.sights523@example.net | … | 2022-07-11 |
| 121 | Ting | Lil | lil.ting121@example.com | … | 2021-10-06 |
| 545 | Knife | Jack | jack.knife545@example.com | … | 2022-07-24 |
| 505 | Singers | Carol | carol.singers505@example.net | … | 2022-06-29 |
| 377 | Money | Xavier | xavier.money377@example.net | … | 2022-04-02 |
| 266 | Blind | Rob | rob.blind266@example.net | … | 2022-01-01 |
| ~ 34 行 ~ |

一些开发者倾向于无论是否需要，总是包含括号，以使观点更清晰。无论哪种方式，请记住 SQL 有明确定义的解释混合逻辑操作符的方式。


### IN 操作符

使用`OR`时，有一种特殊情况。你可能需要测试一个表达式是否精确匹配多个不同的值，例如：

```
--  Change Precedence
SELECT *
FROM customers
WHERE state='VIC' OR state='QLD' OR state='WA'; --  等等
```

你将得到以下结果组：

| id | familyname | givenname | state |
| --- | --- | --- | --- |
| 186 | Gunn | Ray | VIC |
| 179 | Inkling | Ivan | WA |
| 475 | Blood | Drew | QLD |
| 523 | Sights | Seymour | VIC |
| 588 | Skies | Grace | QLD |
| 191 | Moss | Pete | WA |
| ~ 151 rows ~ |

这里，表达式`state`被测试是否匹配多个值。你可以使用`IN`表达式重写这个测试：

```
SELECT *
FROM customers
WHERE state IN ('VIC','QLD','WA');
```

使用这个表达式有两个要求：

*   测试是针对单个列或类似表达式的。在此例中，它测试的是`state`列。
*   测试是针对一个离散的可能性列表的。在此例中，列表是一组州值。

`IN`有两种形式。在这个例子中，你提供了一个括号括起来的硬编码选项列表。用英语说，可以是“where state is in the following list:”，或者更自然地，“where state is one of:”。

你的列表也可以包含不匹配的值或重复的值：

```
SELECT *
FROM customers
WHERE state IN ('VIC','QLD','VIC','ETC');
```

查找不存在的值总是可以的，但通常你不会重复一个值。不过，这种问题可能会间接发生，正如你将在后面看到的。

使用`IN`可以方便地反转条件：

```
--  Change Precedence
SELECT *
FROM customers
WHERE state NOT IN ('VIC','QLD','WA');
```

这将给出其他州的数据：

| id | familyname | givenname | state |
| --- | --- | --- | --- |
| 474 | Free | Judy | NSW |
| 144 | King | Ray | NSW |
| 341 | Idate | Val | NSW |
| 351 | Tate | Dick | NSW |
| 422 | Why | Wanda | TAS |
| 429 | Morrow | Tom | NSW |
| ~ 118 rows ~ |   |   |   |

在前面的例子中，列表是一组硬编码的可能值。你也可以将`IN`表达式与由子查询生成的列表一起使用。

### 派生列表

`IN`子句还有第二种更复杂的形式。

例如，假设你想根据单笔销售找到消费最高的客户。问题在于，销售总额在一个表（`sales`）中，而客户详细信息在另一个表（`customers`）中。幸运的是，`sales`表包含重要的`customerid`，它关联回`customers`表。

要得到结果：

1.  从`sales`表中，获取`total`超过某个值的`customerid`。
2.  从`customers`表中，获取`id`与第一步结果匹配的客户的数据。

对于第一步：

```
SELECT customerid FROM sales WHERE total>1200
```

我们得到一个客户 id 列表：

| **customerid** |
| 2 |
| 10 |
| 19 |
| 46 |
| 24 |
| 69 |
| ~ 147 rows ~ |

（前面的表达式没有分号，因为它将被合并到下一步中）。

对于第二步，使用`IN`表达式将客户与第一步中的多个值进行匹配：

```
SELECT *
FROM customers
WHERE id IN(SELECT customerid FROM sales WHERE total>1200);
```

这给出了匹配的客户：

| id | familyname | givenname | … | registered |
| --- | --- | --- | --- | --- |
| 186 | Gunn | Ray | … | 2021-11-15 |
| 144 | King | Ray | … | 2021-10-18 |
| 179 | Inkling | Ivan | … | 2021-11-08 |
| 351 | Tate | Dick | … | 2022-03-13 |
| 191 | Moss | Pete | … | 2021-11-19 |
| 305 | Net | Clara | … | 2022-01-26 |
| ~ 106 rows ~ |   |   |   |   |

`IN`子句中的`SELECT`语句被称为**子查询**。原则上，它先被求值，结果用于主查询。

如果你将`IN`与子查询一起使用，有另一种可能更直观的表达式：

```
--  PostgreSQL, MySQL/MariaDB, MSSQL, Oracle (not SQLite)
SELECT *
FROM customers
WHERE id = ANY(SELECT customerid FROM sales
WHERE total>1200);
```

类似地，你可以使用子查询来查找荷兰艺术家的所有画作：

1.  找出`nationality`是 Dutch 的艺术家的`id`。
2.  找出`artistid`是前面那些`id`之一的画作：

```
    --  All SQLs
    SELECT *
    FROM paintings
    WHERE artistid IN (SELECT id FROM artists WHERE nationality='Dutch');
    --  not SQLite
    SELECT *
    FROM paintings
    WHERE artistid=ANY(SELECT id FROM artists
    WHERE nationality='Dutch');
```

这应该会给出荷兰艺术家的画作：

| id | artistid | title | year | price |
| --- | --- | --- | --- | --- |
| 81 | 198 | The Garden of Earthly Delights |   |   |
| 1503 | 182 | Breakfast of Crab | 1648 | 160.00 |
| 2128 | 370 | The Geographer |   | 125.00 |
| 264 | 370 | Girl with a Pearl Earring | 1666 | 140.00 |
| 1446 | 266 | Entrance to the Public Garden in Arles | 1888 | 115.00 |
| 968 | 50 | Basket of Fruits | 1622 | 140.00 |
| ~ 172 rows ~ |

请注意，荷兰人实际上并不自称“Dutch”；那是基于与德国人混淆的英语名称。在`artists`表中，一些艺术家被列为`Netherlandish`。你也应该将他们包括进来：

```
SELECT *
FROM paintings
WHERE artistid IN (
    SELECT id FROM artists
    WHERE nationality='Dutch' OR nationality='Netherlandish'
);
```

这扩大了结果组：

| id | artistid | title | year | price |
| --- | --- | --- | --- | --- |
| 541 | 256 | Butcher’s Stall with the Flight into Egypt |   | 110.00 |
| 81 | 198 | The Garden of Earthly Delights |   |   |
| 1503 | 182 | Breakfast of Crab | 1648 | 160.00 |
| 2128 | 370 | The Geographer |   | 125.00 |
| 264 | 370 | Girl with a Pearl Earring | 1666 | 140.00 |
| 1446 | 266 | Entrance to the Public Garden in Arles | 1888 | 115.00 |
| ~ 186 rows ~ |

或者使用另一个`IN`表达式：

```
SELECT *
FROM paintings
WHERE artistid IN (
    SELECT id FROM artists  WHERE nationality IN ('Dutch','Netherlandish')
);
```

你将在本书中看到更多的子查询。在某些情况下，可能会有替代方法，可能更高效，来获得相同的结果，例如连接表。你将在后面学习连接表。

### 通配符匹配

对于字符串，你可以使用`通配符`匹配来扩大搜索范围。例如：

```
SELECT *
FROM customers
WHERE familyname LIKE 'Ring%';
```

这将选择姓氏以`Ring`开头的客户。

| id | familyname | givenname | … |
| --- | --- | --- | --- |
| 90 | Ringer | Belle | … |
| 309 | Ringing | Belle | … |
| 165 | Ring | Wanda | … |
| 164 | Ringing | Isabelle | … |
| ~ 4 rows ~ |

字符串`Ring%`不再是一个简单的字符串：它现在是一个`模式`。

通配符匹配有两个要求：

*   使用`LIKE`关键字来表示后面是一个模式。
*   模式包含特殊字符。

你可以不使用特殊模式字符而使用`LIKE`，但那样模式就只是一个精确匹配。例如：

```
--  Using LIKE
SELECT *
FROM customers
WHERE familyname LIKE 'Ring';
--  Same as simple match
SELECT *
FROM customers
WHERE familyname='Ring';
```

没有多少客户完全匹配这个字符串：

| id | familyname | givenname | … |
| --- | --- | --- | --- |
| `165` | `Ring` | `Wanda` | `…` |

另一方面，如果你不使用`LIKE`关键字，模式字符将被视为普通字符：

```
SELECT *
FROM customers
WHERE familyname='Ring%';   --  nobody called Ring%
```

记住，根据你的排序规则，其他字符可能是区分大小写的，也可能不是。


### 大小写敏感性与模式匹配

请记住，某些数据库管理系统（DBMS）和数据库是区分大小写的，而另一些则不是。

对于默认不区分大小写的 MySQL/MariaDB 和 Microsoft SQL Server，你无需担心，可以直接使用小写字母：
```sql
--  MySQL/MariaDB 和 SQL Server
SELECT *
FROM customers
WHERE familyname LIKE 'ring%';  --  也适用于 'Ring%'
```

SQLite 默认也可能执行不区分大小写的匹配。

对于其他数据库，你可以通过转换大小写（转换为大写或小写）来模拟不区分大小写的匹配：
```sql
--  适用于所有 DBMS:
SELECT *
FROM customers
WHERE lower(familyname) LIKE 'ring%';   --  也适用于 'Ring%'
```

PostgreSQL 有 `ILIKE` 操作符，专门用于不区分大小写的匹配：
```sql
--  PostgreSQL
SELECT *
FROM customers
WHERE familyname ILIKE 'ring%'; --  也适用于 'Ring%'
```

在本章的剩余部分，我们将简单地使用 `LIKE` 并假定其进行不区分大小写的匹配。

### 模式字符

标准 SQL 有两个主要的模式字符，如表 3-4 所示。

表 3-4 通配符字符

| 字符 | 含义 | 文件通配符 (File Glob) |
| --- | --- | --- |
| `%` | **零个**或**多个**字符 | `*` |
| `_` | 恰好**一个**字符 | `?` |

“文件通配符”列显示了如果你在操作系统中尝试使用模式匹配来查找文件时会使用的字符；“glob”是模式匹配的极客术语。你不能在 SQL 中使用那些字符，这里只是作为比较。没人知道为什么 SQL 不使用它们。

注意，`%` 通配符匹配 **零个** 或更多字符。这意味着可能有也可能没有其他字符。例如：
```sql
--  所有 DBMS
SELECT *
FROM customers
WHERE lower(familyname) LIKE 'ring%';
--  不区分大小写的 DBMS：MSSQL, MySQL / MariaDB
SELECT *
FROM customers
WHERE familyname LIKE 'ring%';
--  PostgreSQL
SELECT *
FROM customers
WHERE familyname ILIKE 'ring%';
```

这将匹配任何姓氏以 `Ring` **开头**的人，即使后面没有其他字符。

另一个通配符字符 `_` 匹配**恰好一个**字符。这种匹配可用于填字游戏。例如：
```sql
SELECT *
FROM customers
WHERE familyname LIKE 'R__e';   --  Rate, Rise, Rice, Rowe
```

这会给你四字符的字符串，其中两个是模糊的：

| id | familyname | givenname | … |
| --- | --- | --- | --- |
| 537 | Rise | Theo | … |
| 359 | Rice | Jasmin | … |
| 551 | Rowe | Mike | … |
| 536 | Rate | Amelia | … |

注意下划线 (`_`) 字符。由于历史原因，它比普通字符稍宽，所以相邻的下划线实际上会接触。这使得如果你连续使用超过两个下划线，数起来会有点困难。

一些等宽字体，如 Source Code Pro，为此设计了稍窄的下划线。

当你组合两个通配符时，就创建了“至少”的含义。例如：
```sql
--  至少 4 个字符，以 S 开头：
SELECT *
FROM customers
WHERE familyname LIKE 'S___%';
```

以下是一些使用 `%` 通配符的更多示例及其英文说明：
```sql
--  以 Ring 开头
SELECT *
FROM customers
WHERE familyname LIKE 'Ring%';
--  以 ring 结尾
SELECT *
FROM customers
WHERE lower(familyname) LIKE '%ring';
--  包含 ring
SELECT *
FROM customers
WHERE lower(familyname) LIKE '%ring%';
--  以 S 开头并以 e 结尾
SELECT *
FROM customers
WHERE familyname LIKE 'S%e';
```

你会注意到，当匹配位置可以是任意位置时，我们使用了 `lower()` 函数以确保安全。对于不区分大小写的数据库，这不是必需的。

使用 `_` 通配符：
```sql
--  完全包含 s
SELECT *
FROM customers
WHERE familyname LIKE '%_s_%';
--  恰好 4 个字符
SELECT *
FROM customers
WHERE familyname LIKE '____';
--  恰好 4 个字符，以 R 开头
SELECT *
FROM customers
WHERE familyname LIKE 'R___';
--  至少 4 个字符
SELECT *
FROM customers
WHERE familyname LIKE '____%';
--  至少 4 个字符，以 S 开头
SELECT *
FROM customers
WHERE familyname LIKE 'S___%';
```

使用这两个通配符，你可以在字符串数据上执行相当灵活的搜索。然而，有时你还需要搜索其他类型的数据。

### 非字符串数据的通配符

通常，通配符旨在用于 **字符串**。然而，如果数据可以转换为字符串，某些数据库管理系统对使用通配符的限制较少。

例如，对数字使用通配符：
```sql
--  MySQL/MariaDB, SQLite, MSSQL, Oracle - NOT PostgreSQL
SELECT *
FROM customers
WHERE height LIKE '17%';
```

这会给你身高在 170s 范围内的记录：

| id | familyname | givenname | height |
| --- | --- | --- | --- |
| 144 | King | Ray | 176.8 |
| 179 | Inkling | Ivan | 170.3 |
| 475 | Blood | Drew | 171.0 |
| 341 | Idate | Val | 177.1 |
| 588 | Skies | Grace | 171.5 |
| 191 | Moss | Pete | 172.3 |
| ~ 114 行 ~ |

在上述 DBMS 中，只有 PostgreSQL 对数据类型足够严格，会禁止这种比较并生成错误。

你可能也会在日期上取得一些成功：

| id | familyname | givenname | dob |
| --- | --- | --- | --- |
| 474 | Free | Judy | 1978-04-01 |
| 475 | Blood | Drew | 1989-12-06 |
| 523 | Sights | Seymour | 1965-01-06 |
| 341 | Idate | Val | 1976-06-04 |
| 351 | Tate | Dick | 1969-08-03 |
| 588 | Skies | Grace | 1999-06-28 |
| ~ 206 行 ~ |

```sql
--  MySQL/MariaDB, SQLite*, MSSQL, Oracle* - NOT PostgreSQL
SELECT *
FROM customers
WHERE dob LIKE '19%';
```

在上述 DBMS 中，再次只有 PostgreSQL 禁止这种比较。然而，请注意：
*   SQLite 没有真正的日期类型，并且本示例中的日期是作为字符串存储的。
*   Oracle 的默认日期格式以日号开始，因此它将尝试匹配日号以 `19` 开头的日期。

如果你想让这个功能在 PostgreSQL 中工作，你可以将值 **转换** 为字符串。从技术上讲，这本来就是你应该做的；只是某些 DBMS 会自动完成：
```sql
SELECT *
FROM customers
WHERE cast(height AS VARCHAR(255)) LIKE '17%';
SELECT *
FROM customers
WHERE cast(dob AS VARCHAR(255)) LIKE '19%';
```

请注意：
*   `VARCHAR` 是 SQL 中字符串的术语。
*   `cast()` 将数据从一种类型更改为另一种类型。
*   PostgreSQL 有一个更短的操作符 (`::`) 用于类型转换，但也会使用较长的版本。

在后面的章节中，你会看到更多关于类型转换的内容。

### 通配符的扩展

大多数情况下，这两个通配符就足够了。然而，一些 DBMS 提供了扩展，允许你微调匹配。

许多 DBMS 提供 **正则表达式**，这是一种非常复杂的模式匹配语法。有些，如 MSSQL，只是扩展了 `LIKE` 子句可用的语法；这将在下一节中介绍。

以下是一些实现。


#### 正则表达式（PostgreSQL、MySQL/MariaDB、Oracle）

正则表达式让你能更精确地控制对单个字符的匹配。

例如，要查找以字母 `A - K` 开头，但*不*以 `h` 或 `y` 结尾的姓氏，你会使用模式 `^[A-K][^hy]`：

*   第一个 `^` 表示从字符串的开头开始匹配。
*   `[A-K]` 匹配从 `A` 到 `K`（包含）这个`范围`内的任意字符。
*   `[^hy]` 表示*不*（`^`）匹配字符 `h` 或 `y`。

在不同数据库管理系统中使用正则表达式的方式有所不同。要匹配前面的模式：

```sql
--  PostgreSQL
SELECT *
FROM customers
WHERE familyname ~ '^[A-K][^hy].*';
--  MariaDB/MySQL
SELECT *
FROM customers
WHERE familyname REGEXP '^[A-K][^hy]';
--  Oracle
SELECT *
FROM customers
WHERE REGEXP_LIKE(familyname,'[A-K][^hy].*');
```

结果如下所示：

| id | familyname | givenname | … | registered |
| --- | --- | --- | --- | --- |
| 474 | Free | Judy | … | 2022-06-12 |
| 186 | Gunn | Ray | … | 2021-11-15 |
| 144 | King | Ray | … | 2021-10-18 |
| 179 | Inkling | Ivan | … | 2021-11-08 |
| 475 | Blood | Drew | … | 2022-06-13 |
| 341 | Idate | Val | … | 2022-03-03 |
| ~ 147 行 ~ |   |   |   |   |

正则表达式可能变得非常复杂，不适合胆小的人尝试。你还会发现，在日常的 SQL 中很少需要用到它们。

#### 更简单的扩展（PostgreSQL， MSSQL）

Microsoft SQL 本身不支持正则表达式，但它提供了一个更简单的变体。使用（方）`括号`（`[ … ]`）通配符与 `LIKE` 关键字结合，有以下形式：

*   `[abcde]`：括号内任意单个字符。
*   `[a-e]`：从第一个到最后一个字符`范围`内的任意单个字符。
*   `[^abcde]`, `[^a-e]`：任意*不*匹配后面所跟内容的字符。

```sql
SELECT *
FROM customers
WHERE familyname LIKE '[a-k][^hy]%';
```

请注意，MSSQL 通常不区分大小写，使用小写字母也会匹配大写字母。

PostgreSQL 确实支持正则表达式，但它也提供了一个使用 `SIMILAR TO` 的更简单变体：

```sql
SELECT *
FROM customers
WHERE familyname SIMILAR TO '[A-K][^hy]%';
```

请注意，PostgreSQL 通常区分大小写，所以第一个字符是大写，而第二个字符是小写。当然，你可以使用 `lower()` 函数并全部使用小写。

### 一个简单的模式匹配示例

归根结底，你会发现最常用的是简单的 `%` 通配符。例如，如果你想查找标题中包含 `portrait`（肖像）这个词的画作：

```sql
SELECT * FROM paintings
WHERE title LIKE '%portrait%';
--  WHERE lower(title) LIKE '%portrait%';
```

这些是符合条件的候选记录：

| id | artistid | title | year | price |
| --- | --- | --- | --- | --- |
| 2023 | 58 | Self-Portrait | 1925 |   |
| 1989 |   | Portrait of Trabuc | 1889 | 160.00 |
| 2178 | 102 | Portrait of Hieronymus Holzschuher | 1526 | 145.00 |
| 2244 | 346 | Model with Unfinished Self-Portrait | 1977 | 135.00 |
| 815 | 108 | Portrait of a Cardinal | 1600 | 155.00 |
| 1491 | 333 | Portrait of Sarah Swaim Chase (The Artist’s Mother) | 1892 | 190.00 |
| ~ 100 行 ~ |

如果你想匹配 *self portrait*（自画像），你需要考虑到有时中间有空格，有时有连字符。你可以使用 `_` 通配符来实现这一点：

```sql
SELECT * FROM paintings
WHERE title LIKE '%self_portrait%';
--  WHERE lower(title) LIKE '%self_portrait%'
```

这缩小了结果范围：

| id | artistid | title | year | price |
| --- | --- | --- | --- | --- |
| 2023 | 58 | Self-Portrait | 1925 |   |
| 2244 | 346 | Model with Unfinished Self-Portrait | 1977 | 135.00 |
| 625 |   | Self-portrait |   | 160.00 |
| 2133 | 102 | Self-Portrait at 26 | 1498 | 115.00 |
| 1944 | 233 | Self Portrait with black Vase | 1911 | 105.00 |
| 1054 |   | Self-Portrait | 1889 | 150.00 |
| ~ 35 行 ~ |

请注意，如果你真的在寻找肖像画，这还不够。一些最著名的肖像画，如 *蒙娜丽莎* 和 *戴珍珠耳环的少女*，会漏过这个搜索。

为了真正方便地搜索画作类别，你需要有额外的列，甚至可能需要额外的表来维护类别。

## 总结

当你有大量的行时，可以使用 `WHERE` 子句进行过滤。`WHERE` 子句后面跟一个或多个断言，这些断言的求值结果要么是 `true`（真）要么是 `false`（假），这决定了某一行是否包含在结果集中。

`WHERE` 子句的语法是：

```sql
SELECT columns
FROM table
WHERE conditions;
```

条件是一个或多个断言，即求值结果为 `true` 或 `false` 的表达式。

通常，断言与列数据相关。然而，任何不相关的断言也有效，尽管你可能会得到整张表或者什么也得不到。

### `NULL`

`NULL` 代表一个缺失的值，因此测试它比较棘手。

*   `NULL` *总是* 无法通过比较测试，例如 `=`。
*   测试 `NULL` 需要使用特殊表达式 `IS NULL` 或 `IS NOT NULL`。

### 数字

数字字面量以裸写方式表示：它们没有任何形式的引号。

*   数字按照数轴顺序进行比较，可以使用基本的比较运算符进行过滤。
*   过滤连续值时，需要记得使用相等比较（`<=` 或 `>=`）来包含中间值。

