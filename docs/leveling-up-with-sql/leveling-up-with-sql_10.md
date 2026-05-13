# 4. 处理计算数据

毫无疑问，你之前一定见过计算。SQL 允许你在查询中包含计算数据。

在本章中，我们将探讨关于不同类型数据及其计算方式的一些重要概念。

不要过度沉迷于在 SQL 中进行计算。数据库更关心的是维护和访问原始数据。然而，能够将你的原始数据转化为对当前任务更有用的形式，是很有帮助的。

不同的数据库管理系统在执行计算的能力上差异很大。尤其是在函数方面，它们不仅在功能范围上有所不同，甚至在数据库管理系统对它们的称呼上也不一样。

特别是，SQLite 执行计算的能力非常有限，尤其是在函数方面。

在本章中，我们将处理各种类型的数据，包括字符串。如果你使用的是 MariaDB/MySQL，我们强烈建议你会话设置为 ANSI 模式，这样字符串的行为就能与标准 SQL 保持一致。

你可以用 `SET SESSION sql_mode = 'ANSI';` 来开始你的会话。

## 计算基础

我们稍后会看更多细节，但这里概述一下 SQL 中计算的工作原理。

你可以基于单个列或多个列来计算值。例如：

```sql
SELECT
    height/2.54,                       --  单列
    givenname||' '||familyname         --  多列
    --  givenname+' '+familyname       --  MSSQL
FROM customers;
```

这会得到：

| **?column?** | **?column?** |
| --- | --- |
| 66.339 | May Knott |
| 67.283 | Rick Shaw |
| 60.236 | Ike Andy |
| ~ 303 行 ~ |

你也可以“硬编码”值或从子查询中获取它们：

```sql
SELECT
    'active',                   --  硬编码
    (SELECT name FROM towns WHERE id=townid)  --  子查询
FROM customers;
```

你会得到：

| **?column?** | **?column?** |
| --- | --- |
| active | Kings Park |
| active | Richmond |
| ~ 303 行 ~ |

还有一些内置函数：

```sql
SELECT
    upper(familyname)          --  大写函数
FROM customers;
```

这会得到：

| **?column?** |
| --- |
| KNOTT |
| SHAW |
| ~ 303 行 ~ |

在所有情况下，你会注意到计算值都没有一个合适的名称。

### 使用别名

计算列给 SQL 带来了一个小麻烦。通常，每一列都应该有一个唯一的名称，但 SQL 不清楚该如何称呼新生成的列。

有些 SQL 客户端会让计算列保持无名，而有些则会生成一个虚拟名。在尝试简单的 `SELECT` 语句时，这没问题，但当要认真对待该语句时（例如，你计划稍后使用其结果），你就需要为每一列指定一个更好的名称。

**别名** 是列的新名称，无论它是计算列还是原始列。你使用 `AS` 关键字来创建别名。例如：

```sql
SELECT
    id AS customer,
    height/2.54 AS height,
    givenname||' '||familyname AS fullname,
    --  givenname+' '+familyname AS fullname    --  MSSQL
    'active' AS status,
    (SELECT name FROM towns WHERE id=townid) AS town,
    length(email) AS length
    --  len(email) AS length           --  MSSQL
FROM customers;
```

这样看起来好多了：

| **customer** | **Height** | **fullname** | **status** | **town** | **length** |
| --- | --- | --- | --- | --- | --- |
| 42 | 66.339 | May Knott | active | Kings Park | 23 |
| 459 | 67.283 | Rick Shaw | active | Richmond | 24 |
| ~ 303 行 ~ |

注意：
- `id` 列即使没有被计算，也被设置了别名。
- `height` 计算被别名为 `height`；这没问题，因为它仍然表示相同的意思，只是单位不同。

除了每个计算列必须有唯一名这一事实外，包含别名的其他原因如下：
- 有时，你只是需要重命名列，以获得更好的含义或适应后续使用。
- 有时，你需要将列格式化或转换为更合适的形式，但仍希望保留其原始名称。

此时，我们并不关心前面的别名是否是列的最佳名称；我们只是看看它们是如何工作的。

#### 别名名称

总的来说，别名名称的规则与列名的规则相同。这意味着：
- 别名和原始列名必须是唯一的。
- 别名不应包含空格，不能以数字开头，也不能包含其他特殊字符。
- 别名不应是 SQL 关键字。

如果你确实需要绕过上面第二和第三条规则，你可以将别名用双引号括起来。例如：

```sql
SELECT
    ordered AS "order",
    shipped AS "shipped date"
FROM sales;
```

这里，名称 `order` 是一个 SQL 关键字，而 `shipped date` 包含一个空格。

| **Order** | **shipped date** |
| --- | --- |
| 2022-05-15 21:12:07.988741 | 2022-05-23 |
| 2022-05-16 03:03:16.065969 | 2022-05-24 |
| ~ 5549 行 ~ |

你应该克制这样做的冲动。别名，如同列名，是为了技术用途而非美观。`SELECT` 语句实际上不是报表。

有些数据库管理系统为特殊名称提供了双引号的替代方案：
- Microsoft SQL 提供方括号作为替代：`[shipped date]`。没有理由更倾向于使用它而不是双引号。
- MySQL/MariaDB 使用“反引号”作为替代：`` `shipped date` ``。在 ANSI 模式下，这是不必要的，但在传统模式下，你只能用这个。

无论你选择什么名称，请记住它们应该是纯粹功能性的。不要试图使用大小写、空格或其他可能看起来更好的东西。那是处理查询输出的软件的事情。在 SQL 中，你只需要一个合适的名称来引用数据。

#### AS 是可选的

你很快就会发现 `AS` 是可选的：

```sql
SELECT
    id customer,
    height/2.54 height,
    givenname||' '||familyname fullname,
    --  givenname+' '+familyname fullname   --  MSSQL
    'active' status,
    (SELECT name FROM towns WHERE id=townid) town,
    length(email) length
    --  len(email) length         --  MSSQL
FROM customers;
```

有些开发者认为省略 `AS` 可以节省时间或让他们看起来更专业。然而，你很快也会犯这种错误：

```sql
SELECT
    id,
    email
    givenname, familyname,
    height,
    dob
FROM customers;
```

这会给你一个令人困惑的结果：

| **id** | **givenname** | **familyname** | **height** | **dob** |
| --- | --- | --- | --- | --- |
| 42 | may.knott61@example.net | Knott | 168.5 | [NULL] |
| 459 | rick.shaw459@example.net | Shaw | 170.9 | 1945-07-03 |
| ~ 303 行 ~ |

乍一看，这似乎没问题，因为它没有技术错误。然而，仔细检查后，你会发现 `email` 被别名为了 `givenname`，因为它们之间没有逗号。将一列别名为另一列是合法的，尽管你并不经常真正想这么做。

你无法阻止 SQL 允许这种做法，但如果你养成一个总是用 `AS` 来指定别名的模式，就能让这类错误更容易被发现。


#### 别名在查询后续部分不可用

回顾基础 SQL 子句的处理顺序：

1.  `FROM`
2.  `WHERE`
3.  `GROUP BY`
4.  `HAVING`
5.  `SELECT`
6.  `ORDER BY`

这与你编写 SQL 的方式不同，因为你*先写* `SELECT` 子句。

这在下面这样的语句中会造成主要混淆点：

```sql
SELECT id, title, price, price*0.1 AS tax
FROM books
WHERE tax<1.5;
```

这将导致错误，因为尽管 `price*0.1 AS tax` 表达式写在第一个子句中，但它实际上在 `WHERE` 子句之后才被处理。因此，`tax` 在 `WHERE` 子句中尚不可用。

如果你将计算结果别名为原始列名，情况会变得更令人困惑：

```sql
SELECT
id, title,
price*1.1 AS price  -- 调整以含税
FROM books
WHERE price<15;
```

这将生效。这里，`price` 被增加以包含税，并被别名为原始名称，这是合法的。

| **id** | **书名** | **price** |
| --- | --- | --- |
| 2078 | The Duel | 13.75 |
| 1530 | Robin Hood, The Prince of Thieves | 13.75 |
| 982 | Struwwelpeter: Fearful Stories and Vile Pictures … | 12.65 |
| 573 | The Nose | 11 |
| 1573 | Rachel Ray | 11 |
| 532 | Elective Affinities | 12.65 |
| ~ 521 行 ~ |

然而，`WHERE` 子句将基于*原始* `price` 列进行过滤，*而不是*调整后的版本。

同样，你对此直接能做的并不多，因为你不能选择将 `SELECT` 子句写在更靠后的位置，也不能在其他任何子句中创建别名。

稍后，我们将看到使用公共表表达式如何有助于预处理计算列。

如果你计划稍后使用某个计算结果，将其别名为原始列名可能不是一个好主意。

SQL 对如何处理别名有清晰的规定，但人类读者很可能会感到困惑。

### 处理 NULL 值

迟早，你会在计算中遇到可怕的 `NULL`。实际上，`NULL` 本身并没有问题，但它会彻底搞乱你的计算。

任何涉及 `NULL` 的计算结果都将是 `NULL`。可以说 `NULL` 对于计算极具破坏性。除非你将数据通过一个或两个能够处理 `NULL` 的表达式。

Oracle 字符串是例外。Oracle 将 `NULL` 字符串视为空字符串。一方面，这很方便；另一方面，有时你确实需要 `NULL` 的行为像 `NULL`。

如果你正在对包含 `NULL` 的单个列进行计算，结果也是 `NULL` 是合理的。例如：

```sql
SELECT
id, givenname, familyname,
height/2.54 AS height       -- 有时为 NULL
FROM customers;
```

有时你会得到 `NULL` 结果：

| **id** | **Givenname** | **familyname** | **height** |
| --- | --- | --- | --- |
| 101 | Artie | Chokes | 63.858 |
| 489 | Justin | Case | [NULL] |
| 59 | Leigh | Don | 66.693 |
| 593 | Luke | Warm | [NULL] |
| 170 | Dan | Dee | 65.039 |
| 541 | Neil | Downe | 64.606 |
| ~ 303 行 ~ |

由于我们所做的只是转换单个值，因此将 `NULL` 保留原样是完全可接受的——如果你不知道身高（厘米），那么你仍然不知道它（英寸）。然而，我们很快会看到，有时你可能用你觉得更好的值替换 `NULL`。

另一方面，如果你在多个列上进行计算，且其中大多数不是 `NULL`，这种行为会变得更加麻烦：

```sql
SELECT
id, givenname, othernames, familyname,
givenname||' '||othernames||' '||familyname AS fullname
--  MSSQL:
--  givenname+' '+othernames+' '+familyname AS fullname
FROM authors;
```

除了 Oracle，你会得到很多 `NULL`：

| **id** | **givenname** | **Othernames** | **familyname** | **fullname** |
| --- | --- | --- | --- | --- |
| 464 | Ambrose | [NULL] | Bierce | [NULL] |
| 858 | Alexander | [NULL] | Ostrovsky | [NULL] |
| 525 | Francis | [NULL] | Beaumont | [NULL] |
| 479 | C.E. | Van | Koetsveld | C.E. van Koetsveld |
| 703 | Friedrich | [NULL] | Engels | [NULL] |
| ~ 488 行 ~ |

在前面的例子中，大多数作者没有 `othernames` 的值，因此它是 `NULL`。有些甚至没有 `givenname` 值。大多数情况下，`givenname` 或 `familyname` 没问题，但 `othernames` 的 `NULL` 毁掉了整个计算。

然而，在 Oracle 中，你不会得到 `NULL`。但是，你会在缺失名字的位置看到额外的空格。

从技术上讲，结果是正确的。如果你不知道某些名字，那么你就不知道全名。然而，这没有帮助。

#### Coalesce

SQL 有一个名为 `coalesce()` 的函数，可以用首选的替代值替换 `NULL`。单词 “coalesce” 实际上意味着合并，但它如何成为此操作的名称是那些在古代历史深处丢失的谜团之一。

该函数这样使用：

```sql
coalesce(expression,planB)
```

如果前面的 `expression` 恰好是 `NULL`，则将使用 `planB`。

你可以有多个替代值：

```sql
coalesce(expression,planB,planC, ... , planZ)
```

如果 `planB` 也恰好是 `NULL`，`coalesce()` 将尝试下一个替代值，依此类推，直到找到实际值，或者替代值用尽。

你可以在缺失的电话号码上看到 `coalesce` 的实际应用：

```sql
SELECT
id, givenname, familyname,
phone
FROM employees;
```

在 `employees` 表中，有些人缺失电话号码。

| **id** | **Givenname** | **familyname** | **phone** |
| --- | --- | --- | --- |
| 7 | Ebenezer | Splodge | 0491577644 |
| 4 | Gladys | Raggs | 0491573087 |
| 28 | Cornelius | Eversoe | [NULL] |
| 32 | Clarisse | Cringinghut | 0491571804 |
| 33 | Will | Power | 0491576398 |
| 26 | Fred | Kite | 0491572983 |
| ~ 34 行 ~ |

将这些缺失的电话号码替换为公司的总机号码是合理的：

```sql
SELECT
id, givenname, familyname,
coalesce(phone,'1300975711')  -- 合并至总机号码
FROM employees;
```

这里，缺失的号码已被合并：

| **id** | **Givenname** | **familyname** | **coalesce** |
| --- | --- | --- | --- |
| 7 | Ebenezer | Splodge | 0491577644 |
| 4 | Gladys | Raggs | 0491573087 |
| 28 | Cornelius | Eversoe | 1300975711 |
| 32 | Clarisse | Cringinghut | 0491571804 |
| 33 | Will | Power | 0491576398 |
| 26 | Fred | Kite | 0491572983 |
| ~ 34 行 ~ |

关于 `coalesce()` 的问题是，你并不总能蒙混过关。你需要确保你的替代有意义，并且你的猜测是个好猜测。有很多时候它没有意义，例如一本书缺失的价格或作者的出生日期；`NULL` 通常是你能做的最好的事情。

在第 2 章中，你使用 `coalesce()` 猜测了一个缺失的数量，然后修复了它，使得该数量在未来不能为 `NULL`。有时，那是最好的解决方案。

#### 修复作者姓名

使用 `coalesce()` 函数，你可以用一个替代值来填补缺失的作者姓名。这里需要考虑两点：

*   你不能虚构一个缺失的姓名，因此替代值必须是一个空字符串。
*   你还应该去掉缺失姓名后面的空格。

对于第二点，我们不会单独对姓名进行 coalesce，而是对姓名和空格的组合进行处理，这个组合也应该为 `NULL`——除了 Oracle，它的行为方式不同。我们会对 Oracle 采取不同的方法。

要将姓名和空格 coalesce 成为空字符串，我们可以使用

```sql
--  PostgreSQL, MariaDB/MySQL, SQLite
SELECT
id, givenname, othernames, familyname,
coalesce(givenname||' ','')
||coalesce(othernames||' ','')
||familyname AS fullname
FROM authors;

--  MSSQL
SELECT
id, givenname, othernames, familyname,
coalesce(givenname+' ','')
+coalesce(othernames+' ','')
+familyname AS fullname
FROM authors;
```

这会得到

| **id** | **givenname** | **Othernames** | **familyname** | **fullname** |
| --- | --- | --- | --- | --- |
| 464 | Ambrose | [NULL] | Bierce | Ambrose Bierce |
| 858 | Alexander | [NULL] | Ostrovsky | Alexander Ostrovsky |
| 525 | Francis | [NULL] | Beaumont | Francis Beaumont |
| 479 | C.E. | Van | Koetsveld | C.E. van Koetsveld |
| 703 | Friedrich | [NULL] | Engels | Friedrich Engels |
| ~ 488 rows ~ |

由于 Oracle 会很乐意连接一个 `NULL` 字符串，我们不能使用 `coalesce()`。相反，我们将使用 `ltrim()` 函数。这个函数会移除字符串的前导空格。因为我们是在字符串的*末尾*添加一个空格，只有当姓名为空时，它才会成为一个前导空格。这会给我们

```sql
--  Oracle
SELECT
id, givenname, othernames, familyname,
ltrim(givenname||' ')||ltrim(othernames||' ')
||familyname AS fullname
FROM authors;
```

这应该会给出与之前相同的结果。

### 在其他子句中使用计算

在本章中，我们主要在 `SELECT` 子句中使用计算。当然，任何包含值的子句都可以使用计算出的值。我们在这里看几个例子。

计算的一个明显用途是在 `WHERE` 子句中。例如，你可以查找标题较短的书籍：

```sql
SELECT *
FROM books
WHERE length(title)<24;     --  MSSQL: len(title)
```

给出

| **id** | **authorid** | **Title** | **published** | **price** |
| --- | --- | --- | --- | --- |
| 2078 | 765 | The Duel | 1811 | 12.50 |
| 503 | 128 | Uncle Silas | 1864 | 17.00 |
| 2007 | 99 | North and South | 1854 | 17.50 |
| 702 | 547 | Jane Eyre | 1847 | 17.50 |
| 1759 | 17 | La Curée | 1872 | 16.00 |
| 205 | 436 | Shadow: A Parable | [NULL] | 17.50 |
| ~ 762 rows ~ |

如果你的数据库区分大小写，并且你需要匹配一个大小写未知的字符串，你可能需要这个：

```sql
SELECT *
FROM books
WHERE lower(title) LIKE '%journey%';
```

给出

| **id** | **authorid** | **Title** | **published** | **price** |
| --- | --- | --- | --- | --- |
| 880 | 777 | The Journey of Niels Klim to the Wor … | 1741 | 12.50 |
| 946 | 704 | Following the Equator: A Journey Aro … | 1897 | 19.50 |
| 1314 | 606 | Mozart’s Journey to Prague | 1856 | 17.00 |
| 1092 | 295 | A Journey to the Western Islands of … | 1775 | 14.50 |
| 502 | [NULL] | Journey to the Center of the Earth | 1864 | 15.50 |
| 1454 | 914 | A Sentimental Journey | 1768 | 13.50 |

你也可以在子查询中计算聚合值：

```sql
SELECT *
FROM customers
WHERE height<(SELECT avg(height) FROM customers);
```

给出

| **id** | **familyname** | **Givenname** | **…** | **height** | **…** |
| --- | --- | --- | --- | --- | --- |
| 42 | Knott | May | … | 168.5 | … |
| 597 | Andy | Ike | … | 153.0 | … |
| 352 | Isk | Basil | … | 156.4 | … |
| 526 | Coming | Seymour | … | 163.5 | … |
| 26 | Twishes | Bess | … | 164.6 | … |
| 91 | North | June | … | 164.5 | … |
| ~ 128 rows ~ |

你也可以在 `ORDER BY` 子句中使用计算，例如当你想按标题长度排序时：

```sql
SELECT *
FROM books
ORDER BY length(title);     --  MSSQL: length(title)
```

这会给出

| **id** | **authorid** | **title** | **published** | **price** |
| --- | --- | --- | --- | --- |
| 385 | 971 | She | 1887 | 11.00 |
| 488 | 478 | Mumu | 1852 | 18.00 |
| 728 | 534 | Emma | 1815 | 10.00 |
| 1625 | 496 | Lenz | 1835 | 18.50 |
| 317 | 99 | Ruth | 1853 | 16.50 |
| 2140 | 17 | Nana | 1880 | 12.50 |
| ~ 1200 rows ~ |

然而，你很可能想要选择你排序的依据，因此在 `SELECT` 子句中计算该值并按结果排序会更有意义：

```sql
SELECT id, authorid, title, length(title) AS len, published, price
FROM books
ORDER BY len;       --  MSSQL: length(title)
```

这更具信息性：

| **id** | **authorid** | **Title** | **len** | **published** | **price** |
| --- | --- | --- | --- | --- | --- |
| 385 | 971 | She | 3 | 1887 | 11.00 |
| 488 | 478 | Mumu | 4 | 1852 | 18.00 |
| 728 | 534 | Emma | 4 | 1815 | 10.00 |
| 1625 | 496 | Lenz | 4 | 1835 | 18.50 |
| 317 | 99 | Ruth | 4 | 1853 | 16.50 |
| 2140 | 17 | Nana | 4 | 1880 | 12.50 |
| ~ 1200 rows ~ |

这里有一个在 `ORDER BY` 子句中使用 `coalesce` 的有趣用法。一些数据库管理系统支持 `NULLS FIRST` 或 `NULLS LAST` 来决定在排序顺序中放置 `NULL` 的位置。如果你的数据库管理系统不支持，你可以将该列 coalesce 为一个极值。例如：

```sql
SELECT *
FROM customers
ORDER BY coalesce(height,0);    --  NULLS FIRST

SELECT *
FROM customers
ORDER BY coalesce(height,1000); --  NULLS LAST
```

通过将所有 `NULL` coalesce 为一个极值，SQL 会相应地将它们排序到一端或另一端。

至于 `FROM` 子句，你需要一个生成虚拟表的计算。这通常会是一个视图、一个连接，甚至是一个子查询。在这个上下文中，一个公用表表达式就像一个子查询。我们稍后会做更多这类事情。

### 关于计算的更多细节

SQL 数据库通常理解三种主要数据类型：数字、字符串和日期。这些类型有一些变体，例如数字是否包含小数，字符串的长度，或者日期是否包含时间。还有一些其他类型，例如布尔值（仅限于 `true` 或 `false`）或二进制数据（有时称为二进制大对象，或简称 BLOBs），在不同数据库管理系统中的支持程度各不相同。

在这里，我们将看看使用主要数据类型进行计算的一些细节。

通常，值有三种形式：

*   一个 `存储的` 值可能来自变量或列。
*   一个值可能是 `计算出的` 或来自内置函数。
*   一个 `字面` 值可以直接在代码中输入。

SQL 和大多数编程语言一样，需要一些帮助来区分某些字面值与其他代码。数字字面值按原样输入（裸量），因为它们显然不可能是其他东西。

而字符串或日期字面值则用单引号（`' ... '`）包裹以标记出来。这是为了让 SQL 能够区分字符串和其他单词，例如 SQL 关键字或表和列名。

字符串或日期字面值的实际值不包括引号。但是，在将它们写入代码时需要引号。

在接下来的大部分内容中，我们将使用字面值作为示例。


### 类型转换

`cast()` 函数用于将一个值解释为不同的数据类型。回想一下，SQL 有三种主要数据类型：数字、字符串和日期。你可以使用 `cast` 来做以下两件事之一：

*   你可以尝试将一种主要类型转换为另一种主要类型。
    *   转换为字符串应该相当容易，但转换为另一种类型需要 SQL 知道如何解释该值。不同的数据库管理系统在转换失败时的反应不同。

*   你可以在同一种主要类型内进行转换。例如，你可以在整数和十进制数之间转换，或者在日期和日期时间之间转换。
    *   如果你将十进制数转换为整数、将日期时间转换为日期，或者将字符串转换为更短的字符串，你自然会丢失精度。如果你向相反方向转换，额外的精度将用相当于“无”的内容填充。
    *   如果你确实转换为更窄的类型，它可能会正常工作，但不要过于冒险。例如，将数字 `123.45` 转换为 `decimal(4,2)` 将会失败，因为你没有允许足够的位数；你会得到一个溢出错误。

对于接下来的内容，请记住 SQLite 没有日期类型，所以你不用担心那种转换。稍后，我们会快速看一下 SQLite 中的等效方法。

以下是一些在类型内进行转换的示例：

```sql
--  更短的日期和数字
SELECT
--  非 SQLite:
cast(ordered as date) AS ordered_date,
cast(total AS integer) AS whole_dollars
FROM sales;
--  更短的字符串
SELECT cast(title AS varchar(16)) AS short_title
FROM books;
--  更宽泛的日期和数字
SELECT
--  SQLite: 没有日期类型
--  PostgreSQL, Oracle
cast(dob as timestamp) as long_dob,
--  MariaDB / MySQL, MSSQL
cast(dob as datetime) as long_dob,
cast(height as decimal(5,2)) as long_height
FROM customers;
```

如果你将一个字符串转换为更长的类型，将会发生以下两种情况之一。如果你将其转换为 `CHAR`（固定长度）类型，额外的长度将用空格填充。如果你将其转换为 `VARCHAR` 类型，字符串将保持不变。但是，该字符串将被允许增长到更长的字符串。

在不同类型之间转换是另一回事。大多数数据库管理系统会在必要时自动转换为字符串。例如：

```sql
--  非 MSSQL
SELECT id || ': ' || email
FROM customers;
```

你会得到类似这样的结果：

| **?column?** |
| 42: may.knott61@example.net |
| 459: rick.shaw459@example.net |
| 597: ike.andy597@example.com |
| 186: pat.downe186@example.net |
| 352: basil.isk352@example.net |
| 576: pearl.divers576@example.com |
| ~ 303 rows ~ |

如你所见，MSSQL 不会自动这样做，可能是因为与它们的连接运算符（`+`）混淆。在那里，你必须强制进行：

```sql
--  MSSQL
SELECT cast(id as varchar(5)) + ': ' + email
FROM customers;
```

你也可以对日期做同样的事情。我们将对客户的出生日期这样做，但我们会遇到一些出生日期缺失的复杂情况。使用 `coalesce` 应该可以解决这个问题：

```sql
--  PostgreSQL, MariaDB/MySQL, SQlite
SELECT
id || ': ' || email
|| coalesce('  Born: ' || dob,'')
FROM customers;
--  MSSQL
SELECT
cast(id as varchar(5)) + ': ' + email
+ coalesce('  Born: ' + cast(dob as char(10)),'')
FROM customers;
--  非 Oracle
```

对于 SQLite，这并不费力，因为我们无论如何都是将日期存储为字符串。

在这里，我们对整个连接值 `' Born: ' || dob` 使用了 `coalesce`。这是因为我们希望在 `dob` 缺失时用空字符串替换整个表达式。与 `NULL` 连接应该会导致 `NULL`。

对于 Oracle，你会再次遇到将 `NULL` 字符串视为空字符串的怪癖，因此它们不会被合并。我们可以使用 `CASE` 来解决这个问题：

```sql
--  Oracle
SELECT
id || ': ' || email
|| CASE
WHEN dob IS NOT NULL THEN '  Born: ' || dob
END
FROM customers;
```

基本上，你可以将 `coalesce` 视为简化的 `CASE` 表达式。对于 Oracle，你需要更明确地写出来。

你可能想要更改数据类型的一个原因是为了将它们与其他值混合，例如连接前面的字符串。当我们想要组合来自多个表或虚拟表（例如连接和联合）的数据时，也会看到使用类型转换。

另一个改变数据类型的原因是排序。所有字符串数据通常按字母顺序排序，但你可能需要将它们转换为非字符串类型以进行排序。例如：

```sql
--  整数
SELECT * FROM sorting
ORDER BY numberstring;
SELECT * FROM sorting
ORDER BY cast(numberstring as int);         --  非 MySQL
--  ORDER BY cast(numberstring as signed);  --  MySQL
--  日期 (非 SQLite)
SELECT * FROM sorting
ORDER BY datestring;
SELECT * FROM sorting
ORDER BY cast(datestring as date);
```

在 `sorting` 表中，有一些存储为字符串的值代表数字或日期。对它们进行正确排序的唯一方法是先进行类型转换。

请注意，MySQL 不允许直接转换为整数。你必须使用 `SIGNED`（意思相同）或 `UNSIGNED`。MariaDB 可以接受整数。

并非所有从字符串的转换都会成功，因为字符串可能与正确的类型不相似。例如：

```sql
--  这个可以：
SELECT cast('23' as int)    --  MySQL: as signed
--  FROM dual   --  Oracle
;
--  这个不行：
SELECT cast('hello' as int) --  MySQL: as signed
--  FROM dual   --  Oracle
;
```

接下来会发生什么取决于数据库管理系统：

*   MariaDB/MySQL 都会给出一个 `0`，这比较宽容。
*   MSSQL 会给出一个错误。
    *   不过，你可以使用一个名为 `try_cast` 的替代方法，它只会给出一个 `NULL`。如果你愿意，你可以合并结果。
*   Oracle 也会给出一个错误。
    *   不过，在这个形式中有一个可选的默认值：`cast('hello' as int DEFAULT 0 ON CONVERSION ERROR)`。它比较冗长，但它允许使用 `0` 或其他你喜欢的值作为替代。
*   PostgreSQL 直接给出错误。
    *   可以编写一个函数来绕过它。

### 数值计算

数字通常用于计数某些东西——它是对“多少”这个问题的答案。例如，客户身高有多少厘米，或者这件物品支付了多少美元？

数字并不总是那样使用。有时，它们被用作令牌或代码。你可能对数字执行的计算将取决于数字的使用方式。

#### 基本算术

你总是可以对数字执行基本操作：

```sql
SELECT
3*5 AS multiplication,
4+7 AS addition,
8-11 AS subtraction,
20/3 AS division,
20%3 AS remainder,  --  Oracle: mod(20,3),
24/3*5 AS associativity,
1+2*3 AS precedence,
2*(3+4) + 5*(8-5) AS distributive
--  FROM dual   --  Oracle
;
```

此示例说明了主要操作：

| **mul…** | **add…** | **sub…** | **div…** | **rem…** | **ass…** | **pre…** | **dis…** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 15 | 11 | -3 | 6 | 2 | 40 | 7 | 29 |

请注意，如果你在 Oracle 中测试这个，需要添加 `FROM dual`。

另请注意
*   不同的数据库管理系统对整数除法的态度不同。在某些情况下，`20/3` 会给你一个 `6` 的结果，丢弃小数部分。在其他情况下，你会得到像 `6.66...7` 这样的十进制数。
*   `%` 运算符计算整数除法后的 **余数**。Oracle 使用 `mod()` 函数。
*   当混合操作时，SQL 遵循你在学校学到的关于优先级（哪些运算符先计算）和结合性（从左到右计算）的规则。SQL 也允许你使用括号先计算表达式。

如果你认识忘记算术基本规则的人，你可以告诉他们
1.  先算括号里的。
2.  先算乘法 | 除法，再算加法 | 减法（优先级）。
3.  相同优先级的操作从左到右计算（结合性）。

当然，无论值是字面量还是某些存储或计算的值，这些表达式的工作方式都是一样的。


## 数学函数

也有一些数学函数。大多数情况下，除非你从事相当专业的工作，否则数学函数不会被大量使用。

```sql
SELECT
pi() AS pi,            --  非 Oracle
sin(radians(45)) AS sin45,  --  非 Oracle
sqrt(2) AS root2,      --  √2
log10(3) AS log3,
ln(10) AS ln10,        --  自然对数
power(4,3) AS four_cubed    --  4³
--  FROM dual                  --  Oracle
;
--  Oracle 的三角函数不太方便
SELECT
acos(-1) AS pi,
sin(45*acos(-1)/180) AS sin45
FROM dual;
```

结果大致如下：

| `pi` | `sin45` | `root2` | `log3` | `ln10` | `four_cubed` |
| --- | --- | --- | --- | --- | --- |
| 3.142 | 0.707 | 1.414 | 0.477 | 2.303 | 64 |

所以，现在你可以用 SQL 来计算靠墙梯子的长度，或者计算海上两艘失联船只之间的距离。

## 近似函数

还有一些函数可以给出一个十进制数的近似值。以下是一个示例，其中展示了不同数据库管理系统之间的差异：

```sql
SELECT
ceiling(200/7.0) AS ceiling,
--  SQLite: round(200/7.0 + 0.5),
--  Oracle: ceil(200/7.0),
floor(200/7.0) AS floor,
--  SQLite: round(200/7.0 - 0.5),
round(200/7.0,0) AS rounded_integer,
--  或者 round(200/7), --  非 MSSQL
round(200/7.0,2) AS rounded_decimal
--  FROM DUAL   -- Oracle
;
```

如你所见，这些函数都会损失精度：

| `ceiling` | `floor` | `rounded_integer` | `rounded_decimal` |
| --- | --- | --- | --- |
| 29 | 28 | 29 | 28.57 |

如果你使用 `cast()` 函数转换为另一种更窄的数值类型，你也会损失精度。然而，接下来会发生什么取决于具体的数据库管理系统：

| `DBMS` | `castint` | `castdec` |
| --- | --- | --- |
| PostgreSQL | 235 | 234.57 |
| MariaDB/MySQL | 235 | 234.57 |
| Oracle | 235 | 234.57 |
| MSSQL | 234 | 234.57 |
| SQLite | 234 | 234.567 |

```sql
SELECT
cast(234.567 AS int) AS castint,
--  cast(234.567 AS unsigned),  --  MySQL
cast(234.567 AS decimal(5,2)) AS castdec
--  FROM dual               --  Oracle
;
```

*   对于 PostgreSQL、Oracle 和 MariaDB/MySQL，转换为整数或更短的小数会对数字进行四舍五入。
*   对于 MSSQL，转换为更短的小数会四舍五入，但转换为整数则会截断。如果你想要截断为整数，可以使用类似 `decimal(3,0)` 的方法。
*   对于 SQLite，转换为整数会截断，而转换为小数则会被忽略并保留原始值。

## 数字格式化

格式化函数改变数字的显示外观。与其他近似函数不同，格式化函数的结果不是数字，而是字符串；这是你改变数字外观的唯一方式。

对于数字，你最常想做的是改变小数位数、显示千位分隔符，以及可能添加货币符号。

同样，不同的数据库管理系统有截然不同的函数。例如，以下是一些将数字格式化为带千位分隔符的货币的方法：

```sql
--  PostgreSQL, Oracle
SELECT
to_char(total,'FM999G999G999D00') AS local_number,
to_char(total,'FML999G999G999D00') AS local_currency
FROM sales;
SELECT to_char(total,'FM$999,999,999.00') FROM sales;

--  MariaDB/MySQL
SELECT
format(total,2) AS local_number,
format(total,2,'de_DE') AS specific_number
FROM sales;

--  MSSQL
SELECT
format(total,'n') AS local_number,
format(total,'c') AS local_currency
FROM sales;

--  SQLite
SELECT printf('$%,d.%02d',total,round(total*100)%100)
FROM sales;
```

你会得到类似下面这样的结果：

| `local_number` | `local_currency` |
| --- | --- |
| 28.00 | $28.00 |
| 34.00 | $34.00 |
| 58.50 | $58.50 |
| 50.00 | $50.00 |
| 17.50 | $17.50 |
| 13.00 | $13.00 |
| ~ 5549 行 ~ |

注意

*   PostgreSQL 和 Oracle 都有灵活的 `to_char()` 函数，该函数也可用于格式化日期。
*   MariaDB/MySQL 使用 `format()` 函数，它添加千位分隔符和小数位；你还可以指定它适应不同的区域设置。
*   MSSQL 有其自己的 `format()` 函数，具有更直观的格式代码；它也能适应区域设置，并可用于格式化日期。
*   SQLite 只有一个通用的 `format()` 函数，也称为 `printf()` 函数，这对程序员来说可能更熟悉；SQLite 假定你会在宿主应用程序（如 PHP 或任何嵌入了 SQLite 的地方）中格式化数据。

请注意，如果你确实对一个数字使用了格式化函数，它就不再是数字了！如果你只是查看它，那没关系。但是，如果你计划进行任何进一步的计算，或者对结果进行排序，那么格式化的数字很可能会适得其反。

说到底，格式化可能是你在 SQL 中不会做太多的事情。SQL 的主要目的是获取数据并为下一步做好准备。格式化是最后一步，通常在其他软件中完成。

### 字符串计算

字符串就是字符的序列，因此得名。在 SQL 中，这被称为字符数据。

传统上，SQL 有两种主要的字符串数据类型：

*   字符型：`CHAR(length)` 是固定长度的字符串。如果你输入的字符少于指定长度，字符串将在右侧填充空格。这可能解释了为什么标准 SQL 在比较字符串时会忽略尾随空格。
*   字符可变型：`VARCHAR(length)` 是有长度限制的字符串。如果你输入较短的字符串，它不会被填充。

原则上，`CHAR()` 在处理上更高效，因为它总是相同的长度，数据库管理系统不需要担心计算大小并固定内容。`VARCHAR()` 理应存储效率更高。

实际上，现代数据库管理系统比它们的前辈聪明得多，这两种类型之间的差异已不再那么重要。例如，PostgreSQL 建议始终使用 `VARCHAR`，因为它实际上能更高效地处理该类型。

大多数数据库管理系统提供第三种类型，`TEXT`，它在原则上长度是无限的。同样，现代数据库管理系统允许的标准字符串长度比过去更长，所以这也不那么重要了。微软已弃用 `TEXT`，转而使用功能相同的 `VARCHAR(MAX)`。

字符串字面量写在单引号之间：

```sql
SELECT 'hello'; --  Oracle: FROM dual;
```

使用字符串时，你通常只是想保存和获取它们。然而，你也可以处理字符串本身。这通常是以下操作之一：

*   **串联** 意味着将字符串连接在一起。
    串联是字符串上唯一的直接操作。所有其他操作都使用函数。
*   一些函数会修改字符串。它们并不实际改变字符串，而是返回一个修改后的字符串版本。
*   一些函数可用于提取字符串的部分内容。
*   一些函数更关注字符串的各个字符。


#### 大小写敏感性

SQL 会按预期存储大写/小写字符，但你在搜索它们时可能会遇到困难。这是因为某些数据库忽略大小写，而另一些则不忽略。

数据库如何处理大小写是一个**排序规则**的问题。排序规则指的是它如何解释字母的变体。在英语中，唯一需要担心的变体是大写或小写，但其他语言可能有更多的变体，例如法语或德语中的带重音字母。

排序规则会影响字符串的排序和比较方式。在英语中，你主要关心的是大写字符串是否与小写字符串匹配，以及大写和小写字符串是混合排序还是分开排序。在其他一些语言中，同样的问题可能适用于带重音字符和不带重音字符是否匹配，以及它们如何排序。

你可以在创建数据库或表时设置排序规则，但如果你不关心这个，DBMS 会为新数据库分配一个默认的排序规则。

在 PostgreSQL、Oracle 和 SQLite 中，默认的排序规则是大小写敏感的，因此大写和小写不会匹配。而对于 MySQL/MariaDB 和 MSSQL，默认的排序规则是大小写不敏感的，因此它们会匹配。

如果你不确定你特定的数据库是否区分大小写，可以尝试这个简单的测试：
```sql
SELECT * FROM customers WHERE 'a'='A';
```
如果数据库是大小写敏感的，你将不会得到任何行，因为 `a` 不会匹配 `A`；如果不是，你将得到整个表的内容。

#### ASCII 与 Unicode

传统上，字符串使用 ASCII（美国信息交换标准代码）进行编码。每个字符都有一个从 32 到 126 的数字，存储在一个字节中。例如，`A` 被编码为 65，而 `a` 被编码为 97。特殊字符包括空格（32）、感叹号（33），甚至数字 `0`–`9`（48–57）。

由于 32 到 126 之间只有 95 个值，ASCII 的字符范围有限。一旦你用完了大小写字母和 10 个数字，就已经用掉了 62 个字符，这为标点符号或其他特殊字符留下的空间不多了。（为什么他们会包含像 `~` 和 `` ` `` 这样晦涩的字符，仍然是个谜。）

基本的 ASCII 肯定没有足够的范围来包含更多的标点字符、欧洲带重音字符或希腊/西里尔字母。更不用说日语或中文了。

处理其他语言的一种技术是切换到 ASCII 的不同变体。一个更持久的解决方案是使用 Unicode。

Unicode 是现代标准，用于在单一编码系统中处理多种语言。它通过使用多个字节来实现这一点。具体如何实现可能比较棘手，并且在不同实现中会有所不同，但其理念是相同的。

Unicode 旨在包含 ASCII 码，因此两者之间具有一定的兼容性。然而，一些 Unicode 实现确实比 ASCII 占用更多空间，即使它们编码的是相同的字符。如今，存储空间很便宜，而且数据库软件在高效利用空间方面相当聪明，所以这应该不是什么大问题。

所有现代 DBMS 都支持 Unicode。有些默认支持，有些则需要你主动要求。在某些情况下，你可以为整个数据库、特定表或单个列使用 Unicode。

示例数据库对大多数数据使用 Unicode，但在一些特意限制字符集的情况下可能会使用 ASCII，例如将电话号码存储为字符串时。

一些 DBMS 除了支持 `CHAR` 和 `VARCHAR` 外，还支持 `NCHAR` 和 `NVARCHAR` 数据类型。如果数据库表设置为使用 Unicode，那么 `CHAR` 和 `VARCHAR` 就可以胜任。否则，你可能需要使用 `NCHAR` 和 `NVARCHAR` 在特定列上指定 Unicode。

#### 连接

连接是指将字符串拼接在一起。这是最简单的字符串操作，也是唯一一个不需要函数就能完成的操作。

连接运算符通常是 `||`。Microsoft SQL Server 使用 `+` 代替。例如：
```sql
SELECT
    id,
    givenname||' '||familyname AS fullname
--  givenname+' '+familyname AS fullname    --  MSSQL
FROM customers;
```
这将给出类似这样的结果：

| `Id` | `fullname` |
| --- | --- |
| 42 | May Knott |
| 459 | Rick Shaw |
| 597 | Ike Andy |
| 186 | Pat Downe |
| 352 | Basil Isk |
| 576 | Pearl Divers |
| ~ 303 行 ~ |

请注意，传统模式下的 MySQL 不支持任何形式的连接运算符。在 ANSI 模式下，它支持标准的 `||` 运算符。

许多 DBMS 也支持一个非标准函数 `concat(string,string,...)`。例如：
```sql
--  非 SQLite
SELECT
    id,
    concat(givenname,' ',familyname) AS fullname
FROM customers;
```
这在 SQLite 中不支持。然而，它在 MySQL 中是支持的，因此在传统模式下，这就是你连接字符串的方式。

对于大多数 DBMS，`concat()` 函数和连接运算符之间有一个微妙但重要的区别。使用连接运算符时，如果其中有一个是 `NULL`，结果（自然）也将是 `NULL`。然而，`concat()` 函数会自动将 `NULL` 结果合并为一个空字符串（`''`）。这可能方便也可能不方便，因为有时 `NULL` 是你需要知道的情况。

不过，Oracle 采用了不同的方法。他们将 `NULL` 字符串视为与空字符串 `''` 相同，因此以任何方式连接 `NULL` 都等同于连接一个空字符串。一方面，如果你不想处理合并问题，这很方便；另一方面，有时你需要 `NULL` 就是 `NULL`，所以这可能很麻烦。



#### 字符串函数

对字符串进行其他操作需要使用函数。以下是一些示例。

在接下来的示例中，我们包含了 `SELECT *` 以提供上下文——不过，在 Oracle 中，如果你将其与其他数据混合使用，则需要编写 `SELECT table.*`，因此我们在所有包含 Oracle 的示例中都这样做了。

字符串的长度就是其中的字符数。要查找长度，你可以使用：

```sql
--  PostgreSQL, MySQL/MariaDB, SQLite, Oracle
SELECT customers.*, length(familyname) AS len
FROM customers;
--  MSSQL
SELECT *, len(familyname) AS len FROM customers;
```

要查找字符串中某一部分的位置，你可以使用以下方法：

```sql
--  MySQL/MariaDB, SQLite, Oracle: INSTR('values',value)
SELECT books.*, instr(title,' ') AS space FROM books;
--  PostgreSQL: POSITION(value IN 'values')
SELECT *, position(' ' in title) AS space FROM books;
--  MSSQL: CHARINDEX(value, 'values')
SELECT *, charindex(' ',title) AS space FROM books;
```

你可以使用 `replace` 来替换字符串中的子字符串：

```sql
--  replace(original,search,replace)
SELECT books.*, replace(title,' ','-') AS hyphens
FROM books;
```

要切换大小写，有以下函数：

```sql
--  PostgreSQL, MySQL/MariaDB, SQLite, Oracle, MSSQL
SELECT
  books.*,
  upper(title) AS upper,
  lower(title) AS lower
FROM books;
--  PostgreSQL, Oracle
SELECT books.*, initcap(title) AS lower FROM books;
```

要删除字符串开头或结尾的多余空格，你可以使用 `trim()` 从两端删除，或者使用 `ltrim()` 或 `rtrim()` 分别从字符串的开头或结尾删除：

```sql
WITH vars AS (
  SELECT ' abcdefghijklmnop ' AS string
  --  FROM dual   --  Oracle
)
SELECT
  string,
  ltrim(string) AS ltrim,
  rtrim(string) AS rtrim,
  trim(string) AS trim,
  ltrim(rtrim(string)) AS same
FROM vars;
```

所有现代数据库管理系统都支持 `trim()`，但 MSSQL 在 2017 版本之前并不支持。PostgreSQL 也称之为 `btrim()`。你可能不会注意到右侧的空格被修剪掉了。

你可以根据所使用的数据库管理系统，使用 `substring()` 或 `substr()` 来获取子字符串：

```sql
WITH vars AS (
  SELECT 'abcdefghijklmnop' AS string
  FROM dual   --  Oracle
)
SELECT
  --  PostgreSQL, MariaDB/MySQL, Oracle, SQLite
  substr(string,3,5) AS substr,
  --  PostgreSQL, MariaDB/MySQL, MSSQL, SQLite
  substring('abcdefghijklmnop',3,5) AS substring
FROM vars;
```

一些数据库管理系统包含专门用于获取字符串开头或结尾部分的函数。在某些情况下，你可以使用负的起始位置来获取字符串的末尾部分：

```sql
WITH vars AS (
  SELECT 'abcdefghijklmnop' AS string
  FROM dual   --  Oracle
)
SELECT
  --  左侧
  --  PostgreSQL, MariaDB/MySQL, MSSQL:
  left('abcdefghijklmnop',4) AS lstring
  --  包括 SQLITE 和 Oracle 在内的所有数据库管理系统:
  --  substr(string,1,n) AS lstring,
  --  右侧
  --  PostgreSQL, MariaDB/MySQL, MSSQL:
  right('abcdefghijklmnop',4) AS rstring
  --  MariaDB/MySQL, Oracle, SQLite
  --  substr('abcdefghijklmnop',-4) AS rstring
FROM vars;
```

请注意，如果你花费大量时间从数据中提取子字符串，这可能意味着你试图在一个值中存储过多信息。

另一方面，你通常可以使用子字符串将原始数据重新格式化为更友好的形式。

## 日期操作

从 SQL 的角度来看，日期是有问题的。这是因为，尽管日期在日常生活中无处不在，但日期的计量却相当混乱。

一个问题是我们同时使用多种不兼容的周期来计量日期：日、周、月和年。更糟糕的是，我们生活在不同的时区，所以我们甚至无法就当前时间达成一致。

大多数数据库管理系统都有许多相关的数据类型来管理日期，特别是用于包含时间的日期的 `date`，以及包含时间的 `datetime`。通常，你可以预期这些类型的变体，以及包含时区的能力。

SQLite 是一个例外，它期望你使用数字或字符串，并通过一些函数来处理这些值以进行日期运算。

对于日期和时间，你通常希望进行以下操作：

1.  输入并存储日期/时间
2.  获取当前日期/时间
3.  按日期/时间分组和排序
4.  提取日期/时间的组成部分
5.  对日期/时间进行加减
6.  计算两个日期/时间之间的差值
7.  格式化日期/时间

SQLite 处理日期的方法完全不同。这部分是因为它实际上不支持日期类型。因此，在接下来的大部分讨论中，SQLite 将不会涉及。附录中有关于在 SQLite 中处理日期的信息。

### 输入和存储日期/时间

由于大多数数据库管理系统有自己的存储日期/时间的方式，因此日期存储的具体细节并不重要。重要的是你能够输入数据。

在表中，日期或日期时间列通常定义如下：

| **数据库管理系统** | **日期** | **带时间的日期** |
| --- | --- | --- |
| PostgreSQL | `DATE` | `TIMESTAMP` |
| MariaDB/MySQL | `DATE` | `DATETIME` |
| MSSQL | `DATE` | `DATETIME2` |
| Oracle | `DATE` | `DATETIME` |
| SQLite | `TEXT` | `TEXT` |

输入 `date` 或 `datetime` 字面常量的通常方法是使用以下格式之一：

*   `date`: `'2013-02-15'`
*   `datetime`: `'2013-02-15 09:20:00'`

你也可以省略秒数或包含秒的小数部分。

该格式是 `ISO8601` 格式的变体。在纯 `ISO8601` 格式中，时间会写在 `T` 之后而不是空格之后。

请注意，在 Oracle 中，日期时间字面常量通常使用不同的格式。要使用前面的格式，需要分别在字面常量前加上 `date` 或 `datetime`：

*   `date`: `date '2013-02-15'`
*   `datetime`: `datetime '2013-02-15 09:20:00'`

在 PostgreSQL、MSSQL 和 MySQL/MariaDB 中，你通常也可以输入另一种可读的日期格式，例如 `'15 Feb 2013'`。但是，你*绝不*应该使用在国际上有不同含义的格式 `'2/3/2013'`。

实际上，坚持使用标准格式即可：

```sql
SELECT *
FROM customers
WHERE dob<'1980-01-01'; --  Oracle dob<date '1980-01-01';
```

这会给你较年长的客户：

| **id** | **givenname** | **familyname** | **…** | **dob** | **…** |
| --- | --- | --- | --- | --- | --- |
| 459 | Rick | Shaw | … | 1945-07-03 | … |
| 352 | Basil | Isk | … | 1960-01-13 | … |
| 92 | Nan | Keen | … | 1943-05-18 | … |
| 267 | Boris | Todeath | … | 1969-10-06 | … |
| 91 | June | North | … | 1967-03-22 | … |
| 543 | Nat | Ering | … | 1946-04-30 | … |
| ~ 133 行 ~ |

请注意，在像 `dob<'1980-01-01'` 这样的简单表达式中，SQL 不会混淆表达式是日期还是字符串：上下文已经很清楚了。

### 获取当前日期/时间

有一件事你会想做，就是将某个日期/时间与现在进行比较。在大多数数据库管理系统中，你可以使用：

```sql
SELECT
  current_timestamp AS now,
  current_date AS today       --  非 MSSQL
  --  FROM dual   --  Oracle
;
```

注意

*   MSSQL 也有 `getdate()` 作为 `current_timestamp` 的同义词。尽管名称如此，它返回的不只是日期。
*   MariaDB/MySQL 也有 `now()` 作为 `current_timestamp` 的同义词。
*   Oracle 还有 `systemtimestamp` 和 `systemdate` 用于获取数据库服务器而非客户端上的日期/时间。

如前所述，MSSQL 没有 `current_date` 的版本。无论如何，你可能已有一个现有的 `datetime`，希望将其简化为 `date`。最简单的方法是 `cast` 该 `datetime`：

```sql
--  非 Oracle
SELECT
  current_timestamp AS now,
  cast(current_timestamp as date) AS today
  --  FROM dual   --  Oracle
;
```

这对于 Oracle 不太适用；它确实允许你进行转换，但没有任何效果。相反，你应该使用 `trunc()` 函数：

```sql
--  Oracle
SELECT
  current_timestamp AS now,
  trunc(current_timestamp) AS today
FROM dual   --  Oracle
;
```

这仍然会包含时间部分，但它会被设置为 `00:00`。



## 按日期/时间分组和排序

您可以像对其他任何数据类型一样，按日期/时间进行排序。结果将按历史顺序排列：

```sql
SELECT *
FROM sales
ORDER BY ordered;
```

当然，您也可以使用 `DESC`。

您也可以按 `date` 分组，但您可能不想按 `datetime` 分组，除非您每秒有大量交易。对于 `datetime`，您可以使用公用表表达式（CTE）将其转换为 `date`，然后再对结果进行分组。例如：

```sql
WITH cte AS (
  SELECT
    cast(ordered as date) AS ordered, total  --  非 Oracle
--  trunc(ordered) AS ordered, total     --  Oracle
  FROM sales
)
SELECT ordered, sum(total)
FROM cte
GROUP BY ordered
ORDER BY ordered;
```

这将为您生成以下摘要：

| ordered | sum |
| --- | --- |
| 2022-05-04 | 43.00 |
| 2022-05-05 | 150.50 |
| 2022-05-06 | 110.50 |
| 2022-05-07 | 142.00 |
| 2022-05-08 | 214.50 |
| 2022-05-09 | 16.50 |
| ~ 389 行 ~ |

请记住，在 Oracle 中您需要使用 `trunc()` 函数。

## 提取日期/时间的部分

从技术上讲，`datetime` 代表一个时间点。实际上，我们倾向于从天和年等组成部分来思考。这种情况因以下事实而变得复杂：(a) 各组成部分不同步，并且 (b) 其中一些组成部分的大小会变化。

### 在 PostgreSQL、MariaDB/MySQL 和 Oracle 中提取日期

提取日期部分的标准方法是使用 `extract()` 函数。其形式如下：

```sql
extract(part from datetime)
```

您可以看到 `extract()` 函数的实际应用：

```sql
WITH chelyabinsk AS (
  SELECT
    timestamp '2013-02-15 09:20:00' AS datetime
  FROM dual
)
SELECT
  datetime,
  EXTRACT(year FROM datetime) AS year,
  EXTRACT(month FROM datetime) AS month,
  EXTRACT(day FROM datetime) AS day,
--  非 Oracle 或 MariaDB/MySQL：
  EXTRACT(dow FROM datetime) AS weekday,
  EXTRACT(hour FROM datetime) AS hour,
  EXTRACT(minute FROM datetime) AS minute,
  EXTRACT(second FROM datetime) AS second
FROM chelyabinsk;
```

您将得到以下组成部分：

| datetime | year | month | day | weekday | hour | minute | second |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 2013-02-15 09:20:00 | 2013 | 2 | 15 | 5 | 9 | 20 | 0 |

请注意，Oracle 和 MariaDB/MySQL 没有直接提取星期几的方法，这可能是个问题，例如，如果您想用它进行分组。但是，正如您稍后将看到的，您可以使用格式化函数来获取星期几以及前面的值。

PostgreSQL 还包含一个名为 `date_part('part',datetime)` 的函数，作为前述函数的替代方法。

### 在 Microsoft SQL 中提取日期

Microsoft SQL 有两个主要函数来提取日期部分：

*   `datepart(part,datetime)` 将日期/时间的部分提取为**数字**。
*   `datename(part,datetime)` 将日期/时间的部分提取为**字符串**。对于大多数部分，例如年份，它只是 `datepart` 数字的字符串版本。然而，对于星期几和月份，它实际上是人类友好的名称。

您可以看到这两个函数的实际应用：

```sql
WITH chelyabinsk AS (
  SELECT cast('2013-02-15 09:20' as datetime) AS datetime
)
SELECT
  datepart(year, datetime) AS year,    --  又名 year()
  datename(year, datetime) AS yearstring,
  datepart(month, datetime) AS month,  --  又名 month()
  datename(month, datetime) AS monthname,
  datepart(day, datetime) AS day,      --  又名 day()
  datepart(weekday, datetime) AS weekday, --  星期日=1
  datename(weekday, datetime) AS weekdayname,
  datepart(hour, datetime) AS hour,
  datepart(minute, datetime) AS minute,
  datepart(second, datetime) AS second
FROM chelyabinsk;
```

注意

*   `datename(date,year)` 只给出 `2013` 的字符串版本。
*   有三个短函数——`day()`、`month()` 和 `year()`——它们是 `datepart()` 的同义词。

## 格式化日期

与数字一样，格式化日期会生成一个字符串。

对于 PostgreSQL 和 Oracle，您可以使用 `to_char` 函数。以下是两个有用的格式：

```sql
--  PostgreSQL
WITH vars AS (SELECT timestamp '1969-07-20 20:17:40' AS moonshot)
SELECT
  moonshot,
  to_char(moonshot,'FMDay, DDth FMMonth YYYY') AS fulldate,
  to_char(moonshot,'Dy DD Mon YYYY') AS shortdate
FROM vars;
--  Oracle
WITH vars AS (
  SELECT timestamp '1969-07-20 20:17:40' AS moonshot FROM dual
)
SELECT
  moonshot,
  to_char(moonshot,'FMDay, ddth Month YYYY') AS fulldate,
  to_char(moonshot,'Dy DD Mon YYYY') AS shortdate
FROM vars;
```

您将得到类似以下内容：

| moonshot | full | short |
| --- | --- | --- |
| 1969-07-20 20:17:40 | Sunday, 20th July 1969 | Sun 20 Jul 1969 |

您会注意到 PostgreSQL 和 Oracle 的格式代码之间略有不同。

对于 MariaDB/MySQL，有 `date_format()` 函数：

```sql
WITH vars AS (SELECT timestamp '1969-07-20 20:17:40' AS moonshot)
SELECT
  moonshot,
  date_format(moonshot,'%W, %D %M %Y') AS fulldate,
  date_format(moonshot,'%a %d %b %Y') AS shortdate
FROM vars;
```

对于 Microsoft SQL，`format()` 函数也可用于日期：

```sql
WITH vars AS (SELECT cast('1969-07-20 20:17:40' AS datetime) AS moonshot)
SELECT
  format(moonshot,'dddd, d MMMM yyy') AS fulldate,
  format(moonshot,'ddd d MMM yyy') AS shortdate
FROM vars;
```

SQLite 的格式化功能非常有限，如果没有一些额外的技巧，您肯定无法获取月份或星期的名称。通常最好保留日期不变，让宿主应用程序执行所需的操作。

您可以在以下网址了解有关格式代码的更多信息：

*   PostgreSQL: [`www.postgresql.org/docs/current/functions-formatting.html#FUNCTIONS-FORMATTING-DATETIME-TABLE`](http://www.postgresql.org/docs/current/functions-formatting.html%2523FUNCTIONS-FORMATTING-DATETIME-TABLE)
*   Oracle: [`https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/Format-Models.html`](https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/Format-Models.html%255D)
*   MariaDB: [`https://mariadb.com/kb/en/date_format/`](https://mariadb.com/kb/en/date_format/)
*   MySQL: [`https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html`](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html)
*   Microsoft SQL: [`https://learn.microsoft.com/en-us/dotnet/standard/base-types/custom-date-and-time-format-strings`](https://learn.microsoft.com/en-us/dotnet/standard/base-types/custom-date-and-time-format-strings)


## 日期运算

通常，你对日期要做的两件事是：

*   通过增加或减少一个时间间隔来修改日期。
*   计算两个日期之间的差值。

要修改日期，你可以加上或减去一个时间间隔。一些数据库管理系统为此定义了一种名为 `interval` 的数据类型。例如，要在当前时间上增加四个月，你可以使用：

```sql
-- PostgreSQL
SELECT
    date '2015-10-31' + interval '4 months' AS afterthen,
    current_timestamp + interval '4 months' AS afternow,
    current_timestamp + interval '4' month  -- 也可以
;
-- Oracle
SELECT
    add_months('31 Oct 2015',4) AS afterthen,
    current_timestamp + interval '4' month AS afternow,
    add_months(current_timestamp,4) -- 也可以
FROM dual;
-- MariaDB/MySQL
SELECT
    date_add('2015-10-31',interval 4 month) AS afterthen,
    date_add(current_timestamp,interval 4 month) AS afternow,
    current_timestamp + interval '4' month  -- 也可以
;
```

这将得到类似以下的结果：

| **afterthen** | **Afternow** |
| --- | --- |
| 2016-02-29 00:00:00 | 2023-10-01 16:01:13.691447+11 |

你会注意到 PostgreSQL 和 Oracle 使用加法运算符，而 MariaDB/MySQL 使用一个特殊函数。Oracle 还有一个特殊的 `add_months` 函数用于增加月份。

对于 Microsoft SQL，你使用 `dateadd` 函数，指定单位和单位数量：

```sql
-- MSSQL
SELECT
    dateadd(month,4,'2015-10-31') AS afterthen,
    dateadd(month,4,current_timestamp) AS afternow
;
```

SQLite 使用 `strftime()` 函数将字符串转换为日期，并使用修饰符来调整日期：

```sql
-- SQLite
SELECT
    strftime('%Y-%m-%d','2015-10-31','+4 month') AS afterthen,
    strftime('%Y-%m-%d','now','+4 month') AS afternow
;
```

另一件你想做的事是计算两个日期之间的差值。同样，每个 DBMS 的做法都不同。例如，要找出客户的年龄，你可以使用：

```sql
-- PostgreSQL
SELECT
    dob,
    age(dob) AS interval,
    date_part('year',age(dob)) AS years,
    extract(year from age(dob)) AS samething
FROM customers;
-- MariaDB/MySQL
SELECT
    dob,
    timestampdiff(year,dob,current_timestamp) AS age
FROM customers;
-- MSSQL, 但不完全准确！
SELECT
    dob,
    datediff(year,dob,current_timestamp) AS age
FROM customers;
-- Oracle
SELECT
    dob,
    trunc(months_between(current_timestamp,dob)/12) AS age
FROM customers;
-- SQLite
SELECT
    dob,
    cast(
        strftime('%Y.%m%d', 'now')
        - strftime('%Y.%m%d', dob)
        as int) AS age
FROM customers;
```

对于 PostgreSQL，你将得到以下结果。其他 DBMS 不会有 `age` 列：

| **dob** | **interval** | **Years** | **samething** |
| --- | --- | --- | --- |
| [NULL] | [NULL] | 0 | 0 |
| 1945-07-03 | 77 years 10 mons 29 days | 77 | 77 |
| 1998-08-09 | 24 years 9 mons 23 days | 24 | 24 |
| 1990-04-12 | 33 years 1 mon 19 days | 33 | 33 |
| 1960-01-13 | 63 years 4 mons 19 days | 63 | 63 |
| [NULL] | [NULL] | 0 | 0 |
| ~ 约 303 行 ~ |

在前面的计算中，`MSSQL` 有一个简单的 `datediff` 函数，但它是“短路”的：它只计算年份的差异，如果出生日期在年末而查询日期在年初，结果就会严重偏离。要获得更准确的结果需要更多的工作。

### CASE 表达式

有时简单的表达式无法满足需求，你需要 SQL 做出一些选择。`CASE ... END` 表达式可用于从备选值中进行选择。

例如，你可以根据其他值创建类别：

```sql
SELECT
    id,title,
    CASE
        WHEN price<7 THEN 'expensive'
        -- ELSE NULL
    END AS price
FROM books;
```

你会得到一个简单的价格列表：

| **id** | **Title** | **price** |
| --- | --- | --- |
| 2094 | The Manuscript Found in Saragossa | expensive |
| 336 | The Story of My Life | reasonable |
| 1868 | The Tenant of Wildfell Hall | [NULL] |
| 375 | Dead Souls | reasonable |
| 1180 | Fables | cheap |
| 990 | The History of Pendennis: His Fortun … | cheap |
| ~ 约 1200 行 ~ |

注意，如果所有条件都不满足，结果将是 `NULL`，这在前面被注释掉了。如果你想要一个替代 `NULL` 的值，请使用 `ELSE` 表达式：

```sql
SELECT
    id,title,
    CASE
        WHEN price<7 THEN 'expensive'
        ELSE ''
    END AS price
FROM books;
```

另外，请注意 `CASE` 表达式是“短路求值”的：一旦找到匹配项，它就会停止评估。

### CASE 的各种用法

当你测试一个简单的离散值时，`CASE` 有一种简化的变体。例如：

```sql
SELECT
    c.id,
    givenname||' '||familyname AS name,
    -- givenname+' '+familyname AS name, -- MSSQL
    CASE status
        WHEN 1 THEN 'Gold'
        WHEN 2 THEN 'Silver'
        WHEN 3 THEN 'Bronze'
    END AS status
FROM customers AS c LEFT JOIN VIP ON c.id=vip.id;
-- Oracle:
-- FROM customers c LEFT JOIN VIP ON c.id=vip.id;
```

这将给你：

| **id** | **Name** | **status** |
| --- | --- | --- |
| 69 | Rudi Mentary | [NULL] |
| 182 | June Hills | Bronze |
| 43 | Annie Day | [NULL] |
| 263 | Mark Time | Bronze |
| 266 | Vic Tory | Silver |
| 68 | Phyllis Stein | [NULL] |
| 442 | Herb Garden | Gold |
| 33 | Eileen Dover | [NULL] |
| ~ 约 303 行 ~ |

这种形式并没有短很多，但意图更清晰。

你也可以使用 `IN` 表达式：

```sql
SELECT
    id, givenname, familyname,
    CASE
        WHEN state IN('QLD','NSW','VIC','TAS') THEN 'East'
        WHEN state IN ('NT','SA') THEN 'Central'
        ELSE 'Elsewhere'
    END AS region
FROM customerdetails;
```

这会得到：

| **id** | **Givenname** | **familyname** | **region** |
| --- | --- | --- | --- |
| 137 | Albert | Ross | East |
| 359 | Gail | Warning | Central |
| 40 | Cliff | Face | East |
| 151 | Rick | O’Shea | East |
| 96 | Rob | Blind | Elsewhere |
| 465 | Mary | Christmas | Elsewhere |
| ~ 约 303 行 ~ |

### Coalesce 类似于 CASE 的特殊形式

使用 `coalesce()` 和 `CASE` 之间有相似之处。你可以将 `CASE` 视为 `coalesce` 的一种替代方案：

```sql
SELECT
    id, givenname, familyname,
    coalesce(phone,'-') AS coalesced,
    CASE
        WHEN phone IS NOT NULL THEN phone
        ELSE '-'
    END AS cased
FROM customers;
```

这两个表达式将给出相同的结果：

| **id** | **givenname** | **familyname** | **coalesced** | **cased** |
| --- | --- | --- | --- | --- |
| 42 | May | Knott | 0255509371 | 0255509371 |
| 459 | Rick | Shaw | 0370101040 | 0370101040 |
| 597 | Ike | Andy | - | - |
| 186 | Pat | Downe | 0870105900 | 0870105900 |
| 352 | Basil | Isk | 0255502503 | 0255502503 |
| 576 | Pearl | Divers | 0370107821 | 0370107821 |
| ~ 约 303 行 ~ |

当然，这不一定是一个方便的替代方案，但它有助于理解两者的重叠用途。这在 Oracle 中特别有用，在那里你可以愉快地连接一个 `NULL` 而不会得到 `NULL`，否则很难使用 `coalesce`。


### 嵌套的 CASE 表达式

`CASE` 也可以嵌套额外的 `CASE`。当存在多种可能性时，这很有用。

例如，`sales` 表包含订单下达的日期和时间，以及订单发货（或是否发货）的日期。

我们可以使用 `CASE` 为这些日期生成状态。例如，使用 `shipped` 日期和 `ordered` 日期，可以设置以下标准：

*   已发货：比较 `shipped` 和 `ordered`
    *   超过 14 天 ⇒ `发货延迟`
*   其他情况 ⇒ `已发货`

*   未发货：比较今日与 `ordered`
    *   少于 7 天 ⇒ `当前`
    *   少于 14 天 ⇒ `到期`
*   其他情况 ⇒ `逾期`

但在开始之前，请注意有些销售记录没有 `ordered` 值：

```sql
SELECT * FROM sales;
```

例如，如果客户从未结账，就可能出现这种情况。我们可能应该删除它们，但目前，我们只是将它们过滤掉：

```sql
SELECT * FROM sales WHERE ordered IS NOT NULL;
```

首先，需要计算日期之间的差值。这在不同的数据库管理系统（DBMS）中有所不同：

```sql
--  PostgreSQL, MariaDB / MySQL, Oracle
SELECT
    id, customerid, total,
    cast(ordered as date) AS ordered, shipped,
    current_date - cast(ordered as date) AS ordered_age,
    shipped - cast(ordered as date) AS shipped_age
FROM sales
WHERE ordered IS NOT NULL;
--  MSSQL
SELECT
    id, customerid, total,
    cast(ordered as date) AS ordered, shipped,
    datediff(day,ordered,current_timestamp) AS ordered_age,
    datediff(day,ordered,shipped) AS shipped_age
FROM sales
WHERE ordered IS NOT NULL;
--  SQLite
SELECT
    *,
    julianday('now')-julianday(ordered) AS ordered_age,
    julianday(shipped)-julianday(ordered) AS shipped_age
FROM sales
WHERE ordered IS NOT NULL;
```

你会得到类似以下的结果：

| **id** | **customerid** | **total** | **ordered** | **shipped** | **ordered_age** | **shipped_age** |
| --- | --- | --- | --- | --- | --- | --- |
| 39 | 28 | 28.00 | 2022-05-15 | 2022-05-23 | 382 | 8 |
| 40 | 27 | 34.00 | 2022-05-16 | 2022-05-24 | 381 | 8 |
| 42 | 1 | 58.50 | 2022-05-16 | 2022-05-22 | 381 | 6 |
| 43 | 26 | 50.00 | 2022-05-16 | [NULL] | 381 | [NULL] |
| 45 | 26 | 17.50 | 2022-05-16 | 2022-05-28 | 381 | 12 |
| 668 | 105 | 15.00 | 2022-07-27 | [NULL] | 309 | [NULL] |
| ~ 5295 rows ~ |

请注意，在 SQLite 中，获取年龄的最简单方法是将日期转换为儒略日（Julian date），这是自公元前 4714 年 11 月 24 日正午以来的天数。说来话长。

现在你应该知道，在 `SELECT` 子句的其他部分不能使用计算出的值，所以如果你需要它们，这就很麻烦。然而，你可以分两步完成查询。

如果将前面的查询放在一个公共表表达式（Common Table Expression）中，就可以在主查询中使用其结果。

首先，你需要区分已发货和未发货的记录：

```sql
WITH salesdata AS (
    -- 上述查询之一，末尾不加分号
)
SELECT
    salesdata.*,
    CASE
        WHEN shipped IS NOT NULL THEN
            -- 两种状态之一
        ELSE
            -- 三种状态之一
    END AS status
FROM salesdata;
```

每种情况下的状态是额外的 `CASE` 表达式：

```sql
WITH salesdata AS (
    -- 上述查询之一，末尾不加分号
)
SELECT
    salesdata.*,
    CASE
        WHEN shipped IS NOT NULL THEN
            CASE
                WHEN shipped_age>14 THEN '发货延迟'
                ELSE '已发货'
            END
        ELSE
            CASE
                WHEN ordered_age<7 THEN '当前'
                WHEN ordered_age<14 THEN '到期'
                ELSE '逾期'
            END
    END AS status
FROM salesdata;
```

这将给出类似以下的结果：

| **id** | **cid** | **total** | **ordered** | **shipped** | **ordered_age** | **shipped_age** | **status** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 39 | 28 | 28.00 | 2022-05-15 | 2022-05-23 | 382 | 8 | 已发货 |
| 40 | 27 | 34.00 | 2022-05-16 | 2022-05-24 | 381 | 8 | 已发货 |
| 42 | 1 | 58.50 | 2022-05-16 | 2022-05-22 | 381 | 6 | 已发货 |
| 43 | 26 | 50.00 | 2022-05-16 | [NULL] | 381 | [NULL] | 逾期 |
| 45 | 26 | 17.50 | 2022-05-16 | 2022-05-28 | 381 | 12 | 已发货 |
| 668 | 105 | 15.00 | 2022-07-27 | [NULL] | 309 | [NULL] | 逾期 |
| ~ 5295 rows ~ |

