# 字符串

字符串就是一系列字符。在 SQL 中，它通常被描述为 `字符` 数据，你会看到它被定义为 `CHAR`（固定长度）或 `VARCHAR`（可变长度）。

在内部，每个字符都存储为一个数字。这个数字具体是什么可能会有所不同。历史上，字符串是使用 `ASCII` 存储的，这是一种单字节字符编码。例如，字母 `A` 被存储为数字 65。

因为 `ASCII` 是单字节的，它的字符范围有限。如果你需要最基本范围之外的字符，你需要改变它，例如使用字节中的额外位或切换到替代字符集。

现代的数据库管理系统可以使用 `Unicode`，它适用于所有语言和特殊字符。对于传统的 `ASCII` 字符，代码编号是相同的，但 `Unicode` 可以使用多字节编码来扩展范围，为每个字符提供自己唯一的代码，而无需与他人竞争。

一些数据库管理系统总是使用 `Unicode`，而一些使用 `ASCII` 除非另有指定。示例数据库使用 `Unicode`，但现在这并不重要。接下来的内容对两者都同样适用。

你不能直接对字符串做太多操作。有一件事你可以做，那就是将它连接到另一个字符串：

```sql
SELECT 'abc' || 'def';  --  标准
SELECT 'abc' + 'def';   --  仅限 MSSQL
```

这个技术术语叫做串联。

如果你在 MySQL/MariaDB 中尝试这个，它可能不工作。在传统模式下，MySQL/MariaDB 将 `||` 运算符视为逻辑运算符；在 ANSI 模式下，它应该可以工作。如果你被困在传统模式下，你将需要不同的方法。

有一个常见但非标准的函数叫做 `concat()`：

```sql
SELECT concat('abc','def'); --  SQLite 不支持
```

`concat()` 函数在 SQLite 中不可用，但它在 MySQL/MariaDB 以及 PostgreSQL、MSSQL 和 Oracle 中可用。如果你在传统模式下运行 MySQL/MariaDB，这是连接字符串的唯一方法。

当然，如果你使用的是 MySQL/MariaDB，你总是可以切换到 ANSI 模式。

除了串联之外，SQL 还包含一系列处理字符串的函数。大致上，它们分为两类：

*   字符函数：提取字符串的某部分或更改字符串中某些字符的函数
*   格式化函数：将数字和日期转换为格式化字符串的函数

问题是不同的数据库管理系统有自己的字符串函数，所以你会在接下来的内容中看到很多变化。这里的目的不是提供所有可用函数的字典，而是让你了解一下在你的数据库管理系统中可以使用字符串函数做些什么。

## 字符函数

通常，SQL 包含执行以下操作的函数：

*   长度：查找字符串的长度
*   替换：用另一个字符串替换部分字符串
*   查找：在字符串中查找字符或子字符串
*   修剪：删除前导或尾随空格
*   更改大小写：在大写和小写之间切换
*   子字符串：返回部分字符串

以下是这些操作的概述。

### 字符串长度

字符串的长度就是字符串中的字符数。要查找长度，你可以使用

```sql
--  PostgreSQL, MySQL/MariaDB, SQLite, Oracle
SELECT length('abcde');
--  MSSQL
SELECT len('abcde');
```

长度为 0 表示它是一个空字符串。

请注意，字符数不一定等于字节数。如果字符串是 `Unicode`，字符可能占用两个或更多字节。

### 查找子字符串

要查找部分字符串的位置，你可以使用以下方法：

```sql
--  MySQL/MariaDB, SQLite, Oracle: INSTR('值',值)
SELECT instr('abcdefghijklmnop','m');
--  PostgreSQL: POSITION(值 IN '值')
SELECT position('m' in 'abcdefghijklmnop')
--  MSSQL: CHARINDEX(值, '值')
SELECT charindex('m','abcdefghijklmnop');
```

在所有情况下，如果找不到子字符串，结果将是 `0`。

尽管示例是搜索单个字符，但你也可以搜索多字符子字符串，在这种情况下，你会得到子字符串的位置。

这是我们用来按非字母顺序排序字符串技术的一部分。

### 替换

你可以使用 `replace` 来替换字符串中的子字符串：

```sql
--  replace(原始文本,搜索文本,替换文本)
SELECT replace('text with spaces',' ','-')
```

请注意，搜索子字符串是否匹配大小写取决于数据库排序规则，就像 `WHERE` 子句一样。



### 更改大小写

要在大小写之间切换，可以使用

```sql
--  PostgreSQL, MySQL/MariaDB, SQLite, Oracle, MSSQL
SELECT lower('mIxEd cAsE'), upper('mIxEd cAsE');
```

对于 PostgreSQL 和 Oracle，你还可以将每个单词的首字母大写：

```sql
--  PostgreSQL, Oracle
SELECT initcap('mIxEd cAsE')
```

其他数据库管理系统没有如此方便的功能。

### 修剪空格

有时，字符串的开头或结尾会有一些多余的空格。要移除它们，你可以使用 `trim()` 从两端移除，或者使用 `ltrim()` 或 `rtrim()` 从字符串的开头或结尾移除：

```sql
--  PostgreSQL, MySQL/MariaDB, SQLite, Oracle, MSSQL
SELECT rtrim(ltrim(' abcdefghijklmnop '))
--  PostgreSQL, MySQL/MariaDB, SQLite, Oracle, MSSQL>=2017
SELECT trim(' abcdefghijklmnop ');
```

`trim()` 函数不会影响字符串内部的任何空格。

### 子串

子串是字符串的一部分。提取子串最直接的方法是

```sql
Substring
--  PostgreSQL, MariaDB/MySQL, Oracle, SQLite
SELECT substr('abcdefghijklmnop',3,5)
--  MSSQL
SELECT substring('abcdefghijklmnop',3,5)
--  结果为: cdefg
```

在这里，你指定了原始字符串、子串的**起始位置**和子串的**长度**。

一些数据库管理系统提供了专门的函数来获取字符串的开头或结尾部分。在某些情况下，你可以使用负的起始位置来获取字符串的结尾部分：

```sql
--  左侧
--  PostgreSQL, MariaDB/MySQL, MSSQL
SELECT left('abcdefghijklmnop',5);
--  也可以使用 substr(string,1,n)
--  右侧
--  PostgreSQL, MariaDB/MySQL, MSSQL
SELECT right('abcdefghijklmnop',4);
--  MariaDB/MySQL, Oracle, SQLite
substr('abcdefghijklmnop',-4)
```

在大多数情况下，对于普通数据，你并不真正需要提取字符串的一部分。有时你会在数据已被合并，需要再次将其拆开的情况下看到提取操作。然而，如果数据库构建得当，这种情况不会很频繁。

你可能还需要提取字符串的一部分作为格式化过程的一部分。例如，客户电话号码都存储为十位数字字符串，没有空格或其他格式字符。虽然将数据以其最纯粹的形式存储总是最好的，但这并不总是最具可读性的。

你可以使用以下子串来生成 `00 0000 0000` 格式的电话号码：

```sql
--  PostgreSQL, MariaDB/ MySQL
SELECT
id, givenname, familyname,
left(phone,2)||' '||substr(phone,3,4)
||' '||right(phone,4) AS phone
FROM customers;
--  Oracle
SELECT
id, givenname, familyname,
substr(phone,1,2)||' '||substr(phone,3,4)
||' '||substr(phone,-4) AS phone
FROM customers;
--  SQLite
SELECT
id, givenname, familyname,
substr(phone,1,2)||' '||substr(phone,3,4)
||' '||substr(phone,-4) AS phone
FROM customers;
--  MSSQL
SELECT
id, givenname, familyname,
left(phone,2)+' '+substring(phone,3,4)+' '+right(phone,4) AS phone
FROM customers;
```

这现在给出了更具可读性的电话号码。

| id | 名字 | 姓氏 | 电话 |
| --- | --- | --- | --- |
| 474 | Judy | Free |   |
| 186 | Ray | Gunn | 03 5550 5761 |
| 144 | Ray | King | 02 7010 6710 |
| 179 | Ivan | Inkling | 08 7010 1382 |
| 475 | Drew | Blood | 07 5550 8581 |
| 523 | Seymour | Sights | 03 7010 3920 |
| ~ 约 304 行 ~ |

你可能想知道为什么电话号码没有以这种方式存储。问题在于并非每个人都以相同的方式输入电话格式，因此你不确定存储的值是否与你查找的字符串匹配。通过完全移除所有格式，你就有了一个确定的形式，以后可以对其进行格式化。

## 子查询

有时，你需要的数据在另一个表中。将这些数据获取到当前查询中的一种方法是使用**子查询**。你已经在关于数据过滤的第 3 章中见过子查询；在这里，你将把子查询用作计算列。

如果你想在无表查询中包含来自某个表的数据，就可以使用子查询。例如：

```sql
SELECT (SELECT title FROM paintings WHERE id=123);
```

这将给你选定的标题：

| 标题 |
| The Harvest Wagon |

正如你之前看到的，所有子查询都出现在括号内。当然，在前面的例子中，以这种方式使用子查询是没有意义的，但如果你将其与其他数据结合使用，它就更有意义了。

更现实地说，如果你想要来自多个表的数据，可能会使用子查询。例如，假设你想在画作详情中包含艺术家的国籍：

```sql
SELECT
id,
artistid,
title, price,
(SELECT nationality FROM artists
WHERE artists.id=paintings.artistid)
AS nationality
FROM paintings;
```

你现在有了艺术家的国籍：

| id | 艺术家 id | 标题 | 价格 | 国籍 |
| --- | --- | --- | --- | --- |
| 1222 | 147 | Haymakers Resting | 125.00 | French |
| 251 | 40 | Death in the Sickroom | 105.00 | Norwegian |
| 2190 | 135 | Cache-cache (Hide-and-Seek) | 185.00 | French |
| 1560 | 293 | Indefinite Divisibility | 125.00 | French |
| 172 | 156 | Girl with Racket and Shuttlecock | 195.00 | French |
| 2460 | 83 | The Procession to Calvary | 165.00 | Flemish |
| ~ 约 1273 行 ~ |

`WHERE` 子句中的表达式 `artists.id=paintings.artistid` 包含了表名。我们说这些列是**被限定的**。你可以不带表名编写 `WHERE` 子句，但限定列名将有助于你理解和维护查询。

这种类型的子查询被称为**关联**子查询：它包含对主查询的引用。除其他外，这意味着对于主查询的每一行都必须单独评估子查询，因此会涉及一些性能成本。一些子查询，例如第 3 章中用于数据过滤的那些，是非关联的：它们独立于主查询，只评估一次，因此成本要低得多。

至于艺术家的实际姓名，`SELECT` 子句中的子查询只能返回*一个*值。下面的写法是行不通的：

```sql
SELECT
id,
(SELECT givenname, familyname FROM artists
WHERE artists.id=paintings.artistid),
title, price,
(SELECT nationality FROM artists
WHERE artists.id=paintings.artistid) AS nationality
FROM paintings;
```

但是，你可以从子查询返回一个*计算*值：

```sql
--  标准
SELECT
id,
(SELECT givenname||' '||familyname FROM artists
WHERE artists.id=paintings.artistid) AS artist,
title, price,
(SELECT nationality FROM artists WHERE
artists.id=paintings.artistid) AS nationality
FROM paintings;
--  MSSQL
SELECT
id,
(SELECT givenname+' '+familyname FROM artists
WHERE artists.id=paintings.artistid) AS artist,
title, price,
(SELECT nationality FROM artists WHERE
artists.id=paintings.artistid) AS nationality
FROM paintings;
--  SQLite 之外
SELECT
id,
(SELECT concat(givenname,' ',familyname) FROM artists
WHERE artists.id=paintings.artistid) AS artist,
title, price,
(SELECT nationality FROM artists WHERE
artists.id=paintings.artistid) AS nationality
FROM paintings;
```

我们现在有了艺术家的姓名以及国籍：

| id | 标题 | 艺术家 | … | 国籍 |
| --- | --- | --- | --- | --- |
| 1222 | Haymakers Resting | Camille Pissarro | … | French |
| 251 | Death in the Sickroom | Edvard Munch | … | Norwegian |
| 2190 | Cache-cache (Hide-and-Seek) | Berthe Morisot | … | French |
| 1560 | Indefinite Divisibility | Yves Tanguy | … | French |
| 172 | Girl with Racket and Shuttlecock | Jean-Baptiste-Siméon Chardin | … | French |
| 2460 | The Procession to Calvary | Pieter the Elder Bruegel | … | Flemish |
|  ~ 约 1273 行 |

当从同一张子查询表中选择多个值时，你需要使用多个子查询。这对于性能来说可能开始变得非常昂贵，你可能最好使用表连接。你将在下一章中学习表连接。


## CASE 表达式

SQL 提供了一种表达式，可根据值或在某些情况下根据其他类别来生成类别。这就是 `CASE … END` 表达式。

例如，如果你想将绘画作品按价格分组，可以使用

```sql
SELECT
id, title, price,   -- 基本值
CASE
WHEN price<130 THEN '便宜'
END AS price_group
FROM paintings;
```

这将给出你的定价类别：

| id | title | price | price_group |
| --- | --- | --- | --- |
| 1222 | Haymakers Resting | 125.00 | 便宜 |
| 251 | Death in the Sickroom | 105.00 | 便宜 |
| 2190 | Cache-cache (Hide-and-Seek) | 185.00 |   |
| 1560 | Indefinite Divisibility | 125.00 | 便宜 |
| 172 | Girl with Racket and Shuttlecock | 195.00 |   |
| 2460 | The Procession to Calvary | 165.00 |   |
| ~ 1273 行 ~ |

`WHEN` 表达式充当一种 `IF` 操作：如果条件匹配，则使用此值。

请注意，如果条件不匹配，则结果为 `NULL`。无论价格是其他值还是 `NULL`，都是如此。

当然，你不仅限于一个条件：

```sql
SELECT
id, title, price,   -- 基本值
CASE
WHEN price<130 THEN '便宜'
WHEN price<=170 THEN '合理'
END AS price_group
FROM paintings;
```

你现在有两个价格类别：

| id | title | price | price_group |
| --- | --- | --- | --- |
| 1222 | Haymakers Resting | 125.00 | 便宜 |
| 251 | Death in the Sickroom | 105.00 | 便宜 |
| 2190 | Cache-cache (Hide-and-Seek) | 185.00 |   |
| 1560 | Indefinite Divisibility | 125.00 | 便宜 |
| 172 | Girl with Racket and Shuttlecock | 195.00 |   |
| 2460 | The Procession to Calvary | 165.00 | 合理 |
| ~ 1273 行 ~ |

表达式从开头开始计算。即使某个价格，例如 `120`，在技术上小于 `170`，但它匹配第一个条件就足够了，不会进行进一步的测试。我们说这个表达式是 **短路求值** 的。

对于更昂贵的绘画，注意说它们是“其余的”并不正确。因为这包含了你想要排除的 `NULL` 值。

注意到 `NULL` 在比较中会失败，你可以使用

```sql
SELECT
id, title, price,   -- 基本值
CASE
WHEN price < 130 THEN '便宜'
WHEN price <= 170 THEN '合理'
WHEN price > 170 THEN '奢华'
END AS price_group
FROM paintings;
```

现在除了未定价的绘画外，所有作品都有了价格类别。

| id | title | price | price_group |
| --- | --- | --- | --- |
| 1222 | Haymakers Resting | 125.00 | 便宜 |
| 251 | Death in the Sickroom | 105.00 | 便宜 |
| 2190 | Cache-cache (Hide-and-Seek) | 185.00 | 奢华 |
| 1560 | Indefinite Divisibility | 125.00 | 便宜 |
| 172 | Girl with Racket and Shuttlecock | 195.00 | 奢华 |
| 2460 | The Procession to Calvary | 165.00 | 合理 |
| ~ 1273 行 ~ |

或者，你也可以使用 `WHEN price IS NOT NULL`。

在所有示例中，未匹配的价格都会返回 `NULL`。如果需要，你可以显式指定：

```sql
SELECT
id, title, price,   -- 基本值
CASE
WHEN price < 130 THEN '便宜'
WHEN price <= 170 THEN '合理'
WHEN price > 170 THEN '奢华'
ELSE NULL   -- 冗余，因为这是默认值
END AS price_group
FROM paintings;
```

`ELSE` 情况在你想要以一个替代值结束时更有用：

```sql
SELECT
id, title, price,   -- 基本值
CASE
WHEN price < 130 THEN '便宜'
WHEN price <= 170 THEN '合理'
WHEN price > 170 THEN '奢华'
ELSE '-'
END AS price_group
FROM paintings;
```

当值已经处于类别中，但你想重新编码时，也可以使用 `CASE`。例如，`spam` 列设置为 `True/False` 或 `1/0`。你可以使用 `CASE` 使其更清晰：

```sql
SELECT
id, email,
CASE
WHEN spam=1 THEN '是'
WHEN spam=0 THEN '否'
ELSE '' -- 空字符串
END AS spam
FROM customers;
```

这是一个更友好的版本：

| id | email | spam |
| --- | --- | --- |
| 474 | judy.free474@example.net | 否 |
| 186 | ray.gunn186@example.net | 是 |
| 144 | ray.king144@example.net | 否 |
| 179 | ivan.inkling179@example.com | 是 |
| 475 | drew.blood475@example.net | 否 |
| 523 | seymour.sights523@example.net | 是 |
| ~ 304 行 ~ |

请注意，被测试的值是离散的。对于离散情况，有替代语法：

```sql
SELECT
id, email,
CASE spam
WHEN 1 THEN '是'
WHEN 0 THEN '否'
ELSE '' -- 空字符串
END AS spam
FROM customers;
```

这虽然没简短多少，但更清晰地表明你是在从一组离散的备选项中进行选择。

## 转换为不同的数据类型

有时需要改变的不是值本身，而是数据类型。这可能有多种原因，但在组合不同类型的数据时通常很有必要。改变数据类型称为 **类型转换**。

当上下文足够简单时，一些数据库管理系统会自动转换数据：

```sql
-- 非 MSSQL
-- MySQL: SET SESSION sql_mode = 'ANSI';
SELECT id||': '||givenname||' '||familyname AS info
FROM customers;
```

这是组合后的信息：

| info |
| 474: Judy Free |
| 186: Ray Gunn |
| 144: Ray King |
| 179: Ivan Inkling |
| 475: Drew Blood |
| 523: Seymour Sights |
| ~ 304 行 ~ |

虽然 `id` 是整数，但它会被自动转换为字符串以与其他字符串连接。然而，这在 Microsoft SQL Server 中不起作用，因为“+”运算符会将两个数字相加或将两个字符串连接起来，但不知道如何混合它们。

你也可以对日期执行此操作：

```sql
SELECT
id||': '||givenname||' '||familyname||' - '||dob AS info
FROM customers;
```

但是，你可能会得到令人失望的结果：

| info |
| 474: Judy Free - 1978-04-01 |
| [NULL] |
| 523: Seymour Sights - 1965-01-06 |
| ~ 304 行 ~ |

如你所见，当有 `NULL` 会破坏计算时，情况变得复杂。我们将在下一节中解决这个问题，以及如何与 Microsoft SQL Server 协同工作。

### cast( ) 函数

`cast()` 函数用于更改值的 *类型*。总的来说，`cast()` 函数有四种使用方式：

*   你可以转换为同一类型的 *较小* 版本。

例如，可以从浮点数转换为整数，或从日期时间转换为日期。这样做时，你当然会丢失精细的细节。

*   你可以转换为同一类型的 *更宽* 版本。

例如，可以将整数转换为浮点数，或从日期转换为日期时间。如果这样做，更精细的细节将被零或等效值填充。

*   你可以从任何类型转换为字符串。

唯一可能出错的是你指定了太短的字符串类型，例如当你尝试将日期转换为 `varchar(4)` 时。

*   你 *有时* 可以将字符串转换为不同的类型。

如果字符串不符合正确的形式，经典的反应是过度反应，引发错误。一些数据库管理系统提供了更温和的回退方式。

在尝试按具有数字或日期形式的字符串进行排序时，我们已经尝到了 `cast()` 的滋味。


