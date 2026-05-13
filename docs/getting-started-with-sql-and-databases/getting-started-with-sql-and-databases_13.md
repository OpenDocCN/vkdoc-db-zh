# 4. 对结果进行排序

关系数据库基于一系列数学原理，其中包括表是一行的 `集合` 这一概念。数学集合有两个重要特性：

*   集合没有重复项。
*   集合是无序的。

我们稍后会讨论重复项的问题，但现在先看看行的顺序。

SQL 不规定数据如何存储，那是 DBMS 软件的事情。它也不规定数据应以何种顺序被获取。

不过，你确实可以选择使用 `ORDER BY` 子句来指定行的顺序。

请注意，一旦你指定了行顺序，严格来说结果就不再是集合了。通常这无关紧要，但有时你会无法在更复杂的 SQL 语句中使用这个结果。

在本章中，我们将探讨如何对 `SELECT` 语句的结果进行排序。我们将了解数据如何根据数据类型进行排序，还将学习如何对一个或多个列进行排序、控制排序方向，以及如何对派生或计算出的数据进行排序。

我们还会探讨使用排序来生成分页结果、获取随机数据以及执行非标准的排序顺序。

## 使用 ORDER BY 子句

你可以在 `SELECT` 语句末尾使用 `ORDER BY` 来对结果集进行排序：

```sql
SELECT *
FROM customers
ORDER BY id;
```

`ORDER BY` 后面跟的是一个或多个要排序的列。

| id | familyname | givenname | … |
| --- | --- | --- | --- |
| 2 | Wreath | Laurel | … |
| 8 | Leaves | Russell | … |
| 9 | Downe | Ida | … |
| 10 | Fied | Terry | … |
| 11 | Onair | Deb | … |
| 15 | Second | Millie | … |
| ~ 304 行 ~ |

注意

*   `ORDER BY` 是 `SELECT` 语句中 `最后一个` 主要子句。
*   `ORDER BY` 也最后被求值。

稍后，你将使用另一个子句来限制结果数量，该子句可被视为 `ORDER BY` 子句的一部分。

`ORDER BY` 子句独立于 `SELECT` 子句，所以你不必选择你要用来排序的数据：

```sql
SELECT givenname, familyname
FROM customers
ORDER BY id;
```

这会给出以下结果，有点令人费解：

| familyname | givenname |
| --- | --- |
| Wreath | Laurel |
| Leaves | Russell |
| Downe | Ida |
| Fied | Terry |
| Onair | Deb |
| Second | Millie |
| ~ 304 行 ~ |   |

如果特定的顺序对于排序很重要，你很可能希望把它包含进来：

```sql
SELECT
id, --  包含排序列
givenname, familyname
FROM customers
ORDER BY id;
```

稍后你会看到，排序列不一定是原始列：它们也可以是计算出的值。

### 排序方向

默认情况下，排序顺序是 `升序`，即递增。你可以通过包含 `ASC` 关键字来明确这一点：

```sql
SELECT *
FROM customers
ORDER BY id ASC;                        --  ASC 是冗余的
```

然而，既然默认方向就是升序，`ASC` 关键字就是冗余的，大多数 SQL 开发人员不会使用它。

另一方面，你可以通过附加 `DESC`（降序）来反转排序顺序：

```sql
SELECT *
FROM customers
ORDER BY id DESC;
```

这会以相反的顺序给出客户：

| id | familyname | givenname | … |
| --- | --- | --- | --- |
| 595 | Time | Mark | … |
| 594 | Mander | Sally | … |
| 592 | Highwater | Camilla | … |
| 589 | O’Shea | Rick | … |
| 588 | Skies | Grace | … |
| 583 | Knife | Jack | … |
| ~ 304 行 ~ |   |   |   |

我们稍后会看到更多关于排序方向的内容。

## 缺失数据 (NULL)

你的一些数据可能在某些列中包含 `NULL`。这代表一个缺失值。

SQL 标准对于 `ORDER BY` 应该如何处理 `NULL` 的规定很模糊；唯一的要求是它们应该被分组在一起，要么在开头，要么在末尾。

```sql
SELECT *
FROM customers
ORDER BY height;
```

因为身高列中有许多 `NULL`，你会看到它们要么集中在数据的开头，要么在末尾。

| id | familyname | givenname | … | height | … |
| --- | --- | --- | --- | --- | --- |
| 517 | Open | Doris | … | 150.3 | … |
| 42 | Worry | Donna | … | 150.3 | … |
| 154 | Pan | Sam | … | 154.2 | … |
| 551 | Rowe | Mike | … | 155.6 | … |
| 216 | Driver | Laurie | … | 156.3 | … |
| 330 | Fied | Clara | … | 156.6 | … |
| ~ 304 行 ~ |

不同的 DBMS 对将 `NULL` 分组放在哪里有不同的做法。例如：

| DBMS | NULL 的位置 |
| --- | --- |
| SQLite | 低位 |
| MSSQL | 低位 |
| MySQL/MariaDB | 低位 |
| PostgreSQL | 高位 |
| Oracle | 高位 |

PostgreSQL 和 Oracle 将 `NULL` 与最高值分组，而 MySQL/MariaDB、SQLite 和 Microsoft SQL 将它们与最低值分组。

如果你反转排序方向：

```sql
SELECT *
FROM customers
ORDER BY height DESC;
```

你会看到 `NULL` 出现在另一端：

| id | familyname | givenname | … | height | … |
| --- | --- | --- | --- | --- | --- |
| 95 | Banks | Bonnie | … |   | … |
| 377 | Money | Xavier | … |   | … |
| 321 | King | May | … |   | … |
| 46 | Ering | Hank | … |   | … |
| 350 | Bea | May | … |   | … |
| 500 | Mentary | Rudi | … |   | … |
| ~ 304 行 ~ |

对于某些 DBMS，你可以使用 `NULLS FIRST|LAST` 子句覆盖它们的默认放置位置：

```sql
--  PostgreSQL, Oracle & SQLite:
SELECT *
FROM customers
ORDER BY height NULLS FIRST;    --  或 NULLS LAST
```

这对于 PostgreSQL、Oracle 和 SQLite 是可用的。

对于其他不支持此选项的 DBMS，你可以使用 coalesce 来模拟这个选项：

```sql
--  模拟 NULLS FIRST
SELECT *
FROM customers
ORDER BY coalesce(height,0);
--  模拟 NULLS LAST
SELECT *
FROM customers
ORDER BY coalesce(height,1000);
```

`coalesce()` 函数用一个替代值替换 `NULL`。如果你使用一个夸张的值，它就会将 `NULL` 值放在开头或结尾。稍后你会看到更多关于 `coalesce()` 的内容。




### 数据类型

哪个排在前面？一月还是二月？如果这看起来像个脑筋急转弯，那它确实就是。答案取决于我们谈论的是日期还是单词。

在 SQL 中，数据有多种类型，但大多数类型都是基于以下三种核心类型的变体：

*   `数字`（Numbers）用于测量或计数。
*   `日期`（Dates）标记某个时间点。
*   `字符串`（Strings）是任何杂项文本——它们是字符序列；SQL 也将此称为 `字符型`（`character`）数据。

下表 `sorting` 包含不同类型数据的示例：

```sql
SELECT * FROM sorting;
```

如果你检查定义该表的 SQL 语句，你会看到以下主要数据类型：

| 列名 | 类型 |
| --- | --- |
| id | number |
| numbervalue | number |
| datevalue | date |
| stringvalue | string |
| datestring | string |
| numberstring | string |
| numbername, fullname, email, firstname, lastname | string |

其中两列，`datestring` 和 `numberstring`，将日期和数字存储为字符串，而非更合适的类型。

数据类型对于确保数据有效性很重要，同时它也决定了排序顺序：

*   数字当然是按 `数字` 顺序排序；负数小于正数。
*   日期按 `历史` 顺序排序，较早的日期在较新的日期之前。

    请注意，在大多数 DBMS 中，日期通常以 `yyyy-mm-dd` 格式出现，更准确地称为 `ISO 8601` 格式；Oracle 通常使用不同的格式。你可以使用各种函数来更改格式。
*   字符串按 `字母` 顺序排序。

    请注意，有些字符串以大写字母开头，有些以小写字母开头。`根据你的 DBMS 和排序规则（collation），它们可能被一起排序或分开排序。`

数据类型影响排序顺序这一事实，是在设计数据库时确保数据类型正确的原因之一。有时，人们会倾向于取巧，将所有数据存储为字符串，因为字符串可以接受任何类型的值。然而，这会导致排序顺序混乱。

例如：

```sql
--  按数字排序
SELECT * FROM sorting ORDER BY numbervalue;
--  按字符串排序
SELECT * FROM sorting ORDER BY numberstring;
```

你会看到，字符串版本严格按照字母顺序从左到右排序，而使用了合适数据类型的版本则排序更合理。

| id | numberstring |
| --- | --- |
| 3 | 0 |
| 8 | 1024 |
| 5 | 16 |
| 2 | 32 |
| 1 | -4 |
| 4 | 4 |
| 7 | -8 |
| 6 | 8 |

如果你有一个列，里面全是*本应是*数字的字符串，你可以使用 `cast` 函数：

```sql
--  按数字排序
SELECT * FROM sorting ORDER BY cast(numberstring as int);
```

`cast` 函数将数据类型转换为另一种类型。这里，类型被设置为 `INT`，这是 `INTEGER`（整数）的缩写。数据一旦被转换，就会相应排序。

对日期你也可以做同样的操作：

```sql
--  按日期排序
SELECT * FROM sorting ORDER BY datevalue;
--  按字符串排序
SELECT * FROM sorting ORDER BY datestring;
--  按日期排序
SELECT * FROM sorting ORDER BY cast(datestring as date);
```

使用 `cast()`，你会看到结果按正确的日期顺序排列：

| id | dateString |
| --- | --- |
| 1 | 10 Dec 1815 |
| 4 | 12 May 1820 |
| 3 | 13 Sep 1819 |
| 5 | 15 Sep 1890 |
| 8 | 16 Dec 1775 |
| 2 | 29 Sep 1794 |
| 6 | 30 Jun 1917 |
| 7 | 7 Nov 1867 |

稍后你会了解更多关于 `cast()` 的内容。

`cast()` 函数在面对看起来不像数字的值时往往反应过度。MySQL/MariaDB 和 SQLite 会返回 `0`。SQL Server 有一个 `try_cast()` 函数，会返回 `NULL`。Oracle 有一个选项可以在错误时返回某个值。PostgreSQL 和 SQL Server 在使用 `cast()` 函数时会返回致命错误。PostgreSQL 没有友好的替代方案，但编写一个函数很容易。

### 大小写敏感性与排序规则

如果你运行以下查询：

```sql
SELECT * FROM sorting ORDER BY stringvalue;
```

你的结果可能因 DBMS 和数据库的不同而不同。例如：

| id | stringvalue |
| --- | --- |
| 1 | apple |
| 6 | Apple |
| 7 | banana |
| 5 | Banana |
| 3 | cherry |
| 2 | Cherry |
| 4 | date |
| 8 | Date |
| ~ 8 行 ~ |

可能影响你结果的一个因素是数据是否被视为大小写敏感，也就是说，大写和小写字母是否被视为相同。DBMS 如何看待大小写字母以及带变音符号的字母，称为 `排序规则`（`collation`）。

广义上讲，排序规则指的是如何处理同一字母的不同变体。这包括这些变体是否被视为同一字符，如果不是，哪个排在前面。

需要考虑的两种主要变体是：

*   大小写：大写和小写
*   变音符号：带变音符号的变体，例如法语中的 `e`, `é`, `ê`, `è`

你表中数据的排序方式受以下因素影响：

*   DBMS 的默认排序规则
*   表或列指定的排序规则（如果有的话）
*   `ORDER BY` 子句后可选的 `COLLATE` 子句

大小写敏感性问题在过滤数据时也很重要，因为它影响字符串是否匹配。




## 多列排序

如果你试图按一个复合列（包含多个值的列）进行排序，会遇到一些问题：

```sql
SELECT id, firstname, lastname, fullname
FROM sorting
ORDER BY fullname;
```

你会得到一个结果，但很可能不是你想要的：

| id | firstname | lastname | fullname |
| --- | --- | --- | --- |
| 1 | Ada | Lovelace | Ada Lovelace |
| 5 | Agatha | Christie | Agatha Christie |
| 3 | Clara | Schumann | Clara Schumann |
| 4 | Florence | Nightingale | Florence Nightingale |
| 8 | Jane | Austen | Jane Austen |
| 6 | Lena | Horne | Lena Horne |
| 7 | Marie | Curie | Marie Curie |
| 2 | Rose | de Freycinet | Rose de Freycinet |

组合数据时存在诸多问题之一就是，如果不付出额外努力在需要时将其拆分，你就无法对其进行正确排序。

要正确排序数据，你需要将姓名拆分成独立的部分：

| id | firstname | lastname | fullname |
| --- | --- | --- | --- |
| 8 | Jane | Austen | Jane Austen |
| 5 | Agatha | Christie | Agatha Christie |
| 7 | Marie | Curie | Marie Curie |
| 2 | Rose | de Freycinet | Rose de Freycinet |
| 6 | Lena | Horne | Lena Horne |
| 1 | Ada | Lovelace | Ada Lovelace |
| 4 | Florence | Nightingale | Florence Nightingale |
| 3 | Clara | Schumann | Clara Schumann |

```sql
SELECT id, firstname, lastname, fullname
FROM sorting
ORDER BY lastname, firstname;
```

回到真实数据，你可以对 `paintings` 表中的详细信息执行相同操作，其中的列应该是彼此独立的：

```sql
SELECT *
FROM paintings
ORDER BY price, title;
```

这个结果就更有意义了：

| id | artistid | title | year | price |
| --- | --- | --- | --- | --- |
| 681 | 163 | Amerika (Baseball) | 1983 | 100.00 |
| 1928 | 237 | Arrangement in Yellow and Grey: Effie Deans |   | 100.00 |
| 1820 | 188 | Bacchus | 1638 | 100.00 |
| 2367 | 188 | Battle of the Amazons | 1618 | 100.00 |
| 2039 | 18 | Bords d’une rivière (Riverbanks) | 1904 | 100.00 |
| 1269 | 67 | Bouquet of Spring Flowers (Spring Bouquet) | 1866 | 100.00 |
| ~ 1273 rows ~ |

通常，数据首先按第一列（`price`）排序；然后，如果有两条或多条数据的值相同，则会进一步按下一列（`title`）排序。下一列有时被称为*决胜列*。

如果下一列中也存在重复值怎么办？有两种可能：

*   如果有更多 `ORDER BY` 列，则数据会进一步排序。
*   如果没有更多 `ORDER BY` 列，*则无法保证顺序*。

除了你在 `ORDER BY` 子句中指定的内容外，SQL 不对排序顺序做任何承诺。如果你想保证一个确定的排序顺序，你需要在 `ORDER BY` 列表的末尾加上一个保证不重复的列：

```sql
SELECT *
FROM paintings
ORDER BY price, title, id;
```

这里，`id` 是最后的手段；作为主键，它不能重复，因此它的顺序是有保证的。在某些情况下，你也可以使用一个非主键列，比如 `customers` 表中的 `email`，因为在该特定表中，它也被要求是唯一的。

### 列的相互独立性

在前面的例子中，两个主要的列 `price` 和 `title` 是彼此独立的。

从技术上讲，没有理由不能反向对它们进行排序：

```sql
SELECT *
FROM paintings
ORDER BY title, price;
```

总的来说，这个选择是个偏好问题。更准确地说，选择取决于谁想知道这些信息。例如，收藏家可能更喜欢看按标题排序的列表，而销售经理可能更喜欢按价格排序。

有时，各列是相关的^(^(⁶))。例如：

| id | givenname | familyname | town | state |
| --- | --- | --- | --- | --- |
| 234 | Nat | Ering | Bald Hills | NSW |
| 592 | Camilla | Highwater | Bald Hills | NSW |
| 342 | Hugh | Morris | Bald Hills | NSW |
| 91 | Cat | Nip | Bald Hills | NSW |
| 308 | Noah | Vale | Bald Hills | NSW |
| 10 | Terry | Fied | Bald Hills | NSW |
| ~ 304 rows ~ |

```sql
SELECT *
FROM customers
ORDER BY state, town;
```

在这种情况下，`state` 几乎总是会首先排序，因为它被视为城镇的集合：你通常先排大组，再排小组。同样，你也可以反向排序，但大多数人不会期望那样。

前面的讨论有一个例外。历史上，我们按姓氏在前的方式对人名排序：

```sql
SELECT *
FROM customers
ORDER BY familyname, givenname;
```

这在技术上没有特殊原因，因为姓氏不一定是一种分组。

### 多列排序方向

如果你按多列排序，请注意每一列都有自己的排序方向。例如：

```sql
SELECT *
FROM paintings
ORDER BY price, title DESC;
```

结果可能有点令人困惑：

| id | artistid | title | year | price |
| --- | --- | --- | --- | --- |
| 2121 | 252 | Winter Scene on a Canal |   | 100.00 |
| 1938 | 135 | Un village (Le village de Maurecourt) |   | 100.00 |
| 787 | 235 | Untitled (The Hotel Eden) | 1945 | 100.00 |
| 1960 | 235 | Untitled (Paul and Virginia) | 1948 | 100.00 |
| 1262 | 334 | The Young Beggar |   | 100.00 |
| 173 | 355 | The Toreador | 1873 | 100.00 |
| ~ 1273 rows ~ |

这反转了标题的顺序，但主顺序，价格，仍然是升序。如果你包含默认的 `ASC`，会更明显：

```sql
SELECT *
FROM paintings
ORDER BY price ASC, title DESC;
```

然而，没人会这样写，它可能会给人一种 `ASC` 关键字确实做了不同事情的错误印象。最好习惯默认行为。

如果你真的想完全反转数据，你需要对两列都使用 `DESC`：

```sql
SELECT *
FROM paintings
ORDER BY price DESC, title DESC;
```

现在这给了你期望的顺序。

| id | artistid | title | year | price |
| --- | --- | --- | --- | --- |
| 379 | 111 | Woman with a Yellow Bodice | 1899 |   |
| 209 | 192 | Woman with a Mandolin | 1760 |   |
| 461 | 39 | Woman with a Basket |   |   |
| 1438 | 252 | Winter landscape with a frozen river and figures | 1620 |   |
| 1281 | 266 | Village Street in Auvers | 1890 |   |
| 2454 | 266 | Vegetable Gardens in Montmartre | 1887 |   |
| ~ 1273 rows ~ |

请注意，在此示例中，`NULL` 值被排在最后。

记住，每一列都有自己独立的排序方向。



