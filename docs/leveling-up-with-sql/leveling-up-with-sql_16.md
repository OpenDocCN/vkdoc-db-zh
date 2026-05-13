# 7. 使用子查询与公共表表达式

运行一个 `SELECT` 语句，假设没有错误，会给出一个结果。该结果是一个虚拟表，它将包含行和列。

就我们的目的而言，我们对三种可能的虚拟表感兴趣：

*   一行一列：你只得到一个值，尽管技术上它仍然在一个表中。我们将其称为单个值。
*   一列多行：到时候，我们会称其为一个列表。
*   多行多列：在此上下文中，具有多列的单行也算作同类事物。这更像是我们在谈论虚拟表时所想到的那种东西。

当然，该结果也可能是空的，但这被视为 `NULL`。

你将从以下示例中获得这些类型的结果。

例如，一行一列：

```sql
SELECT id FROM books WHERE title='Frankenstein';
```

这是一个单值：

| **Id** |
| --- |
| 392 |

一列多行：

```sql
SELECT email FROM customerdetails WHERE state='VIC';
```

这给出一个列表：

| **Email** |
| --- |
| xavier.money17@example.net |
| anne.onymous262@example.net |
| bess.twishes26@example.net |
| judy.free93@example.net |
| peter.off415@example.com |
| moe.grass360@example.com |
| ~ 64 rows ~ |

多行多列：

```sql
SELECT givenname, familyname, email
FROM customerdetails WHERE state='VIC';
```

这给出一个虚拟表：

| **givenname** | **Familyname** | **email** |
| --- | --- | --- |
| Xavier | Money | xavier.money17@example.net |
| Anne | Onymous | anne.onymous262@example.net |
| Bess | Twishes | bess.twishes26@example.net |
| Judy | Free | judy.free93@example.net |
| Peter | Off | peter.off415@example.com |
| Moe | Grass | moe.grass360@example.com |
| ~ 64 rows ~ |

最后一种类别，即虚拟表，也可能是非常宽泛的查询的结果，例如 `SELECT * FROM customerdetails`。其工作方式完全相同。

根据上下文，这些结果中的任何一个都可以用在后续的查询中，该查询期望使用单个值、一个列表或一个（虚拟）表。例如，使用单个值：

```sql
SELECT *
FROM saleitems
WHERE bookid=(SELECT id FROM books WHERE title='Frankenstein');
```

这里，单值查询被括号包裹起来，其用法就像你已经知道要匹配的 `bookid` 值一样：

| **id** | **Saleid** | **bookid** | **quantity** | **price** |
| --- | --- | --- | --- | --- |
| 7234 | 2873 | 392 | 3 | 18.50 |
| 14875 | 5907 | 392 | 2 | 18.50 |
| 11183 | 4448 | 392 | 2 | 18.50 |
| 1312 | 517 | 392 | 2 | 18.50 |
| 9956 | 3948 | 392 | 2 | 18.50 |
| 12636 | 5012 | 392 | 2 | 18.50 |
| ~ 14 rows ~ |

或者你可能期望某种列表：

```sql
SELECT *
FROM books
WHERE authorid IN (
SELECT id FROM authors WHERE born BETWEEN '1700-01-01' AND '1799-12-31'
);
```

`IN` 操作符期望一个值列表，我们从嵌套的 `SELECT` 语句中的那一列获得：

| **id** | **authorid** | **Title** | **published** | **price** |
| --- | --- | --- | --- | --- |
| 2078 | 765 | The Duel | 1811 | 12.50 |
| 2243 | 715 | Vrijmoedige Verhalen; ee … | 1831 | [NULL] |
| 532 | 628 | Elective Affinities | 1809 | 11.50 |
| 1608 | 420 | The Autobiography of Ben … | 1791 | 18.50 |
| 2303 | 633 | The Metaphysics of Moral … | 1797 | 12.00 |
| 1963 | 529 | A Treatise of Human Natu … | 1740 | 18.50 |
| ~ 256 rows ~ |

嵌套的 `SELECT` 语句被称为 `子查询`。`子查询` 是一个 `SELECT` 语句，它被用作另一个查询的一部分。从概念上讲，你使用 SELECT 查询来获取一个或多个结果，这些结果继而会被用于另一个查询中。

子查询可以在 `SELECT`、`WHERE`、`FROM` 甚至 `ORDER BY` 子句中使用。当使用它们时，结果必须与上下文兼容。例如：

*   `SELECT` 子句中的子查询必须始终返回单个值，就像任何其他计算所期望的那样。这可以称为 `标量`（单值）子查询。一些现代 DBMS 现在开始增加对返回多列值的支持，但目前尚未广泛支持。
*   `WHERE` 子句中的子查询在使用比较操作符时必须返回单个值，或在使用 `IN()` 表达式时返回单个列。
*   `FROM` 子句中的子查询必须返回一个结果表。这种类型的子查询有时被称为内联视图。

子查询的一些用途包括：

*   从另一个表中查找相关数据（`SELECT`）
*   查找一个值或一组值以用作过滤器（`WHERE`）
*   生成一个派生的虚拟表（`FROM`）

关于子查询的整个要点是，通过子查询，你可以组合多个部分以构成更复杂的查询：

*   使用子查询，你将能够从其他表获取数据。
*   你可以使用子查询来预处理数据，以便在主查询中使用。
*   子查询将允许你组合聚合查询和非聚合查询，而这通常无法在单个查询中完成。

然而，子查询确实有成本，因为你将为一次查询付出运行两次或更多次查询的代价——有时甚至是为一次查询付出运行数百次查询的代价。我们将在后面将子查询与其他技术进行比较时详细讨论这一点。

毫无疑问，你之前已经见过一些子查询，肯定在本书前面的某些章节中见过。在本章中，我们将更仔细地研究子查询如何工作以及我们可以用它们做什么。


### 关联与非关联子查询

子查询可以是 `关联` 的或 `非关联` 的。关联子查询包含对主查询的引用。非关联子查询则是独立于主查询进行评估的。

以下是一个非关联子查询的例子：

```
--  女性作家的书籍
SELECT *
FROM books
WHERE authorid IN(
SELECT id FROM authors WHERE gender='f'
);
```

你将得到类似这样的结果：

| `id` | `authorid` | `Title` | `published` | `price` |
| --- | --- | --- | --- | --- |
| 2007 | 99 | North and South | 1854 | 17.50 |
| 702 | 547 | Jane Eyre | 1847 | 17.50 |
| 95 | 701 | Silas Marner | 1861 | 18.50 |
| 983 | 211 | East Lynne | 1861 | 16.00 |
| 678 | 547 | Tales of Angria | 1839 | 14.50 |
| 1255 | 608 | Discourse on the Origin … | 1755 | 19.00 |
| ~ 165 rows ~ |

这是另一个在子查询中使用聚合函数的例子：

```
--  最年长的顾客
SELECT *
FROM customers
WHERE dob=(SELECT min(dob) FROM customers);
```

这会给出

| `id` | `givenname` | `familyname` | `…` | `dob` | `…` |
| --- | --- | --- | --- | --- | --- |
| 92 | Nan | Keen | … | 1943-05-18 | … |
| 392 | Daisy | Chain | … | 1943-05-18 | … |

（你会注意到有不止一位最年长的顾客，因为他们恰好在同一天出生。这种情况时有发生。）

在这两种情况下，子查询都被评估一次，其结果被用于主查询。结果可能是一个列表，如女性作家示例，也可能是一个单一值，如最年长的顾客示例。

非关联子查询独立于主查询。如果你只高亮显示子查询并运行它，你也会得到一个结果。

以下是一个关联子查询的例子：

```
--  书籍作者（是的，还有另一种方法可以实现）
SELECT
id, title, (
SELECT coalesce(givenname||' ','')
|| coalesce(othernames||' ','')
|| familyname
FROM authors
WHERE authors.id=books.authorid
) AS author
FROM books;

--  MSSQL
SELECT
id, title, (
SELECT coalesce(givenname+' ',''
+ coalesce(othernames+' ','')
+ familyname
FROM authors
WHERE authors.id=books.authorid
) AS author
FROM books;

--  Oracle
SELECT
id, title, (
SELECT ltrim(givenname||' ')
||ltrim(othernames||' ')
||familyname
FROM authors
WHERE authors.id=books.authorid
) AS author
FROM books;
```

这会给出以下结果：

| `id` | `title` | `author` |
| --- | --- | --- |
| 2078 | The Duel | Heinrich von Kleist |
| 503 | Uncle Silas | J. Sheridan Le Fanu |
| 2007 | North and South | Elizabeth Gaskell |
| 702 | Jane Eyre | Charlotte Brontë |
| 1530 | Robin Hood, The Prince of Thieves | Alexandre Dumas |
| 1759 | La Curée | Émile Zola |
| ~ 1201 rows ~ |

在这种情况下，子查询对每一行都重新评估一次。看一下前面第一个例子中的子查询，为了让它更易读，我们把它展开：

```
(
SELECT
coalesce(givenname||' ','')
|| coalesce(othernames||' ','')
|| familyname
FROM authors
WHERE authors.id=books.authorid
)
```

`SELECT` 子句期望 `author` 列是一个单一值，因此子查询应该提供一个单一值，它确实做到了。在这种上下文中，你不能使用多列，所以你需要将姓名连接起来以给出这个单一值。

同样重要的是，你也不能返回多行。在这里，`WHERE` 子句将结果过滤为一行，其中 `id` 与主查询中的 `authorid` 匹配：`WHERE authors.id=books.authorid`。

对于 `books` 表中的每一行，子查询都会再次运行以匹配下一个 `authorid`。

如果没有匹配项，子查询会返回一个 `NULL`。

你可以通过查询引用了主查询中的某些内容来识别关联子查询。因此，你不能高亮显示子查询并单独运行它，因为它需要那个引用才能完整。

顺便提一下，注意子查询中的 `WHERE` 子句。在某种意义上，它的条件是过强的，我们本可以这样写：`WHERE id=authorid`。尽管 `id` 列在子查询和主查询中都出现了。

当子查询被评估时，列名将由内向外定义。对于 `id` 列，内部的 `authors` 表中有一个，所以 SQL 不会费心去注意外部的 `books` 表中也有一个。对于 `authorid` 列，`authors` 表中没有，所以它就对应到了 `books` 表中的那个。

这就是 SQL 中的工作方式，但像我们在示例中那样限定列名，可能对我们人类读者来说能最大限度地减少混淆。

一般来说，关联子查询是一项昂贵的操作，因为它被频繁地重新评估。这并不意味着你不应该使用它，只是意味着如果有其他选择，你应该考虑替代方案。你通常无法选择需要哪种类型的子查询，但它有助于在决定是否有更好的替代方案时提供参考。

## SELECT 子句中的子查询

正如我们所见，`SELECT`子句中的子查询预期为每一行返回一个**单一值**。但如果你希望使用关联子查询从同一个外部表中获取多个值，这可能会很繁琐：

```sql
SELECT
id, title, (
SELECT coalesce(givenname||' ','')
|| coalesce(othernames||' ','')
|| familyname
FROM authors
WHERE authors.id=books.authorid
) AS author,
(SELECT born FROM authors
WHERE authors.id=books.authorid) AS born,
(SELECT died FROM authors
WHERE authors.id=books.authorid) AS died
FROM books;
```

除了繁琐，这种方式开销也大。当然，有更好的方法可以实现，即使用连接：

```sql
SELECT
id, title,
coalesce(givenname||' ','')
|| coalesce(othernames||' ','')
|| familyname AS author,
born, died
FROM books AS b LEFT JOIN authors AS a ON b.authorid=a.id;
--  Oracle
--  FROM books b LEFT JOIN authors a ON b.authorid=a.id;
```

实际上，你很可能会发现，关联子查询通常最好用连接来替代。连接也有一定成本，但在此之后，其余数据的获取就是“免费”的了。

另一方面，如果子查询是**非关联的**，那么它的开销就不那么大。例如，以下是客户身高与平均身高的差值：

```sql
SELECT
id, givenname, familyname,
height,
height-(SELECT avg(height) FROM customers) AS diff
FROM customers;
```

你应该会看到类似这样的结果：

| **id** | **givenname** | **familyname** | **height** | **Diff** |
| --- | --- | --- | --- | --- |
| 42 | May | Knott | 168.5 | -2.34 |
| 459 | Rick | Shaw | 170.9 | 0.06 |
| 597 | Ike | Andy | 153.0 | -17.84 |
| 186 | Pat | Downe | 176.0 | 5.16 |
| 352 | Basil | Isk | 156.4 | -14.44 |
| 576 | Pearl | Divers | 176.3 | 5.46 |
| ~ 303 rows ~ |

尽管平均值参与了每一行的计算，但在非关联子查询中，它只计算**一次**。

顺便提一下，还有一种替代方法可以实现前面的查询，即使用`窗口函数`，我们将在第 8 章介绍。不过，就这个例子而言，结果差别不大。

你会注意到，在这种情况下，子查询引用的是与主查询相同的表。但这并不使它成为关联子查询，因为它没有引用主查询中的实际*行*。你可以通过高亮选中子查询并单独运行它来验证——它是可以独立工作的。

本例中的子查询是一个聚合查询。你也可以在关联子查询中使用聚合函数。以下是生成累计总和的一种方法：

```sql
--  Oracle: FROM sales ss
SELECT
id, ordered, total,
(SELECT sum(total) FROM sales AS ss
WHERE ss.ordered<=sales.ordered) AS running_total
FROM sales
ORDER BY id;
```

你会得到类似这样的结果：

| id | ordered | total | running_total |
| --- | --- | --- | --- |
| 1 | 2022-05-04 21:53:55.165107 | 43.00 | 43.00 |
| 2 | 2022-05-05 12:39:41.438631 | 54.50 | 97.50 |
| 3 | 2022-05-05 17:48:08.433387 | 96.00 | 193.50 |
| 4 | 2022-05-07 08:29:35.61573 | 17.50 | 321.50 |
| 5 | 2022-05-07 13:10:25.441528 | 63.00 | 384.50 |
| 6 | 2022-05-06 17:23:38.261261 | 18.00 | 211.50 |
| ~ 5549 rows |

我们必须在子查询中将表别名设为`ss`（subsales？），以便与主查询中的同一个表区分开。这样表达式`ss.ordered<=sales.ordered`才能引用正确的表。

在这里，子查询计算的是按`ordered`列排序后，到当前销售记录（包括当前记录）为止所有`total`的总和。

你可能注意到这个查询运行起来花了点时间。正如我们提到的，关联子查询开销很大，而涉及聚合的关联子查询开销尤其大。幸运的是，也有对应的窗口函数可以实现，我们将在下一章看到。

## WHERE 子句中的子查询

子查询也可以用于过滤你的数据。

同样，你可能会发现子查询的替代方案，比如`JOIN`，但一个引人注目的用例是当子查询是一个聚合查询时。

以下是一些子查询能清晰简洁地说明问题的情况。

### 带简单聚合的子查询

子查询有用的一种情况是当你需要混合聚合与非聚合查询时。例如，如果你想找到最年长的客户，你需要分两步完成：

1.  使用聚合查询找到最早的出生日期。
2.  使用聚合结果作为过滤条件。

在这里，你看到聚合查询被用作子查询：

```sql
SELECT *
FROM customers
WHERE dob=(SELECT min(dob) FROM customers);
```

你也可以用同样的方式找到身高低于平均值的客户：

```sql
SELECT *
FROM customers
WHERE height<(SELECT avg(height) FROM customers);
```

在这两种情况下，聚合查询和主查询都基于同一张表。你可能认为可以使用像`WHERE dob=min(dob)`或`WHERE height<avg(height)`这样的表达式，但这行不通；聚合是在`WHERE`子句**之后**计算的。


#### 大额消费者

假设你想识别你的“大额消费者”——那些消费金额最高的客户。为此，你需要用到 `customers` 表和 `sales` 表的数据。

这里，我们将使用子查询作为多步骤过程的一部分。

首先，你需要确定什么才算大额购买：

```
SELECT * FROM sales WHERE total>160;
```

你会得到类似这样的结果：

| **id** | **customerid** | **total** | **…** | **ordered_date** |
| --- | --- | --- | --- | --- |
| 80 | 32 | 168.00 | … | 2022-05-22 |
| 216 | 13 | 160.50 | … | 2022-06-11 |
| 483 | 59 | 176.50 | … | 2022-07-11 |
| 726 | 68 | 173.00 | … | 2022-08-02 |
| 823 | 86 | 165.50 | … | 2022-08-09 |
| 891 | 140 | 162.50 | … | 2022-08-16 |
| ~ 35 rows ~ |

在这里，我们只对 `customerid` 感兴趣，我们将用它来从 `customers` 表中进行选择：

```
SELECT *
FROM customers
WHERE id IN(SELECT customerid FROM sales WHERE total>160);
```

这会得到

| **id** | **…** | **familyname** | **givenname** | **…** |
| --- | --- | --- | --- | --- |
| 42 | … | Knott | May | … |
| 58 | … | Ting | Jess | … |
| 91 | … | North | June | … |
| 140 | … | Byrd | Dicky | … |
| 40 | … | Face | Cliff | … |
| 141 | … | Rice | Jasmin | … |
| ~ 32 rows ~ |

注意 `IN` 操作符需要一个值的 *列表*。在子查询中，这是一个单列的值。

另请注意，你得到的结果可能比前一个查询少；这可能是因为某些客户 ID 出现了不止一次。

SQL 还有一个 `ANY` 操作符，可以完成同样的工作：

```
SELECT *
FROM customers
WHERE id=ANY(SELECT customerid FROM sales WHERE total>=160);
```

你也可以使用 `JOIN`：

```
SELECT DISTINCT customers.*
FROM customers JOIN sales ON customers.id=sales.customerid
WHERE sales.total>=160;
```

为了重现我们上一个查询的结果，我们限定了星号（`customers.*`）并使用 `DISTINCT` 来去除那些可能在列表中出现多次的客户。

使用连接的优势在于你可以随时获取销售数据，因此这会给出稍微更丰富的结果：

```
SELECT *
FROM customers JOIN sales ON customers.id=sales.customerid
WHERE sales.total>=160;
```

在这里，我们移除了 `DISTINCT` 和 `customers.`，所以你会得到大量数据：

| **id** | **…** | **familyname** | **givenname** | **…** | **total** | **…** | **ordered_date** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 32 | … | Cue | Barbie | … | 168.00 | … | 2022-05-22 |
| 13 | … | Fine | Marty | … | 160.50 | … | 2022-06-11 |
| 59 | … | Don | Leigh | … | 176.50 | … | 2022-07-11 |
| 68 | … | Stein | Phyllis | … | 173.00 | … | 2022-08-02 |
| 86 | … | Fied | Molly | … | 165.50 | … | 2022-08-09 |
| 140 | … | Byrd | Dicky | … | 162.50 | … | 2022-08-16 |
| ~ 35 rows ~ |

要找到总销售额很大的客户，需要使用聚合子查询：

```
SELECT *
FROM customers
WHERE id IN(
SELECT customerid FROM sales
GROUP BY customerid HAVING sum(total)>=2000
);
```

这将得到

| **id** | **…** | **familyname** | **givenname** | **…** |
| --- | --- | --- | --- | --- |
| 42 | … | Knott | May | … |
| 58 | … | Ting | Jess | … |
| 26 | … | Twishes | Bess | … |
| 91 | … | North | June | … |
| 69 | … | Mentary | Rudi | … |
| 140 | … | Byrd | Dicky | … |
| ~ 57 rows ~ |

#### 最后订单

这里，我们将再次使用聚合子查询，这次是为了获取每个客户的最后订单。

首先，我们需要获取每个客户的最后日期和时间：

```
SELECT max(ordered) FROM sales GROUP BY customerid
```

这将给我们一个列表，每个客户对应一个日期时间：

| **Max** |
| 2023-05-15 00:46:00.864446 |
| 2023-05-25 00:42:26.783461 |
| 2023-05-16 05:27:53.810977 |
| 2023-05-06 01:40:02.346894 |
| 2023-05-19 07:41:25.104524 |
| 2023-05-07 19:01:06.756387 |
| ~ 269 rows ~ |

我们将使用这个列表来获取匹配的订单：

```
SELECT * FROM sales
WHERE ordered IN(SELECT max(ordered) FROM sales GROUP BY customerid);
```

这将给我们一个销售列表：

| **id** | **customerid** | **total** | **…** | **ordered_date** |
| --- | --- | --- | --- | --- |
| 6168 | 42 | 121.22 | … | 2023-05-28 |
| 4209 | 287 | 50.50 | … | 2023-03-03 |
| 4542 | 26 | 11.00 | … | 2023-03-18 |
| 4793 | 368 | 56.00 | … | 2023-03-28 |
| 4939 | 282 | 39.00 | … | 2023-04-03 |
| 4953 | 395 | 75.50 | … | 2023-04-03 |
| ~ 266 rows ~ |

如果你数一下行数，可能会发现主查询返回的行数比子查询少。如果存在一些 `NULL` 的订单日期时间，就会发生这种情况。在某个时候，我们应该学会忽略这些记录，要么将它们过滤掉，要么直接移除。

问题是，为什么这些销售没有包含在完整的查询中？答案在于 `IN()` 操作符。

记得在第 3 章中，我们讨论了 `NOT IN` 的怪异之处。这个讨论同样适用于普通的 `IN`。子查询中的 `NULL` 日期时间会导致等同于测试 `WHERE ordered=NULL`，众所周知，这总是失败的。

现在我们有了每个客户的销售记录，将其与 `customers` 表连接起来以获取更多细节就很简单了：

```
SELECT *
FROM sales JOIN customers ON sales.customerid=customers.id
WHERE ordered IN(SELECT max(ordered) FROM sales GROUP BY customerid);
```

你会得到类似这样的结果：

| **id** | **cid** | **total** | **…** | **od** | **id** | **email** | **…** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 6168 | 42 | 121.22 | … | 2023-05-28 | 42 | may.knott61@example.net | … |
| 4209 | 287 | 50.50 | … | 2023-03-03 | 287 | judy.free287@example.com | … |
| 4542 | 26 | 11.00 | … | 2023-03-18 | 26 | bess.twishes26@example.net | … |
| 4793 | 368 | 56.00 | … | 2023-03-28 | 368 | sharon.sharalike368@example.net | … |
| 4939 | 282 | 39.00 | … | 2023-04-03 | 282 | howard.youknow282@example.com | … |
| 4953 | 395 | 75.50 | … | 2023-04-03 | 395 | holly.day395@example.net | … |
| ~ 266 rows ~ |

你现在可以提取任何你可能需要处理的客户或销售数据。

#### 重复的客户

我们在第 2 章中已经看到了如何查找重复项。假设，例如，你想查找重复的客户姓名：

```
SELECT
givenname||' '||familyname AS fullname,
--  MSSQL: givenname+' '+familyname AS fullname,
count(*) as occurrences
FROM customers
GROUP BY familyname, givenname
HAVING count(*)>1;
```

你会得到

| **fullname** | **Occurrences** |
| --- | --- |
| Judy Free | 2 |
| Annie Mate | 2 |
| Mary Christmas | 2 |
| Ken Tuckey | 2 |
| Corey Ander | 2 |
| Ida Dunnit | 2 |
| Paul Bearer | 2 |
| Terry Bell | 2 |

请记住，名字相同并不一定意味着它们是重复的。很可能只是巧合。

我们连接名字是因为下一步我们要做的事情。

聚合查询的问题是，你只能选择你正在分组的内容，所以我们看不到客户的其余细节。任何包含它们的尝试都会破坏聚合。

但是，我们可以将重复项查询用作子查询来过滤 `customers` 表：

```
/*  注意
================================================
MSSQL: 使用 givenname+' '+familyname
================================================ */
SELECT *
FROM customers
WHERE givenname||' '||familyname IN (
SELECT givenname||' '||familyname FROM customers
GROUP BY familyname, givenname
HAVING count(*)>1
);
```

这将给我们客户的其余细节。我们必须连接客户姓名的原因是，在 `IN()` 表达式中只能有一个列。


## FROM 子句中的子查询

子查询也可以生成一个虚拟表，这在你实际查询前需要准备数据时非常有用。

例如，假设你想按价格分组查看你的书籍。你可以创建一个像这样的简单查询：

```sql
SELECT
  id, title,
  CASE
    WHEN price > 17 THEN 'expensive'
    WHEN price > 7 THEN 'reasonable'
    ELSE 'cheap'
  END AS price_group
FROM books;
```

你会得到类似这样的结果：

| **id** | **title** | **price_group** |
| --- | --- | --- |
| 2078 | The Duel | Cheap |
| 503 | Uncle Silas | Reasonable |
| 2007 | North and South | Expensive |
| 702 | Jane Eyre | Expensive |
| 1530 | Robin Hood, The Prince of Thieves | Cheap |
| 1759 | La Curée | Reasonable |
| ~ 1201 行 ~ |

现在，假设你想对这个表进行汇总。问题是，你不能这样做：

```sql
-- 这样不行：
SELECT
-- id, title,
  CASE
    WHEN price > 17 THEN 'expensive'
    WHEN price > 7 THEN 'reasonable'
    ELSE 'cheap'
  END AS price_group,
  count(*) as num_books
FROM books
GROUP BY price_group;
```

我们已经注释掉了不用于分组的列，但由于那个烦人的子句顺序问题，它仍然无法工作：别名 `price_group` 是在 `SELECT` 子句中创建的，而 `SELECT` 子句出现在 `GROUP BY` 子句*之后*，因此它尚不能用于分组。当然，你随后可以在 `GROUP BY` 子句中重现这个计算：

```sql
-- 这样可以，但...
SELECT
-- id, title,
  CASE
    WHEN price > 17 THEN 'expensive'
    WHEN price > 7 THEN 'reasonable'
    ELSE 'cheap'
  END AS price_group,
  count(*) as num_books
FROM books
GROUP BY CASE
           WHEN price > 17 THEN 'expensive'
           WHEN price > 7 THEN 'reasonable'
           ELSE 'cheap'
         END;
```

但你肯定不想那么做。

一个解决方案是将原始的 `SELECT` 语句放入一个子查询中：

```sql
SELECT price_group, count(*) AS num_books
FROM (
  SELECT
    id, title,
    CASE
      WHEN price > 17 THEN 'expensive'
      WHEN price > 7 THEN 'reasonable'
      ELSE 'cheap'
    END AS price_group
  FROM books
) AS sq     -- Oracle: ( ... ) sq
GROUP BY price_group;
```

这会给出一个有意义的结果：

| **price_group** | **num_books** |
| --- | --- |
| expensive | 320 |
| [NULL] | 105 |
| reasonable | 467 |
| cheap | 309 |

记住，`CASE` 表达式的默认穿透（fall through）结果是 `NULL`。那些未定价的书籍最终会归入 `NULL` 价格组。根据数据库管理系统的不同，你可能会在结果集的某个地方看到它作为一个单独的分组。

记住，`SELECT` 语句生成的是一个虚拟表。因此，它可以以子查询的形式用在 `FROM` 子句中。

注意，`FROM` 子句中的子查询有一个特殊要求：它必须有一个别名，即使你没有计划使用它。我们在这里没有特殊计划，所以它就随便被叫做 `sq`（代表“SubQuery”）。如果你想，比如说，将这个子查询与另一个表或虚拟表进行连接，那么别名就会很有用。

### 嵌套子查询

子查询是一个带有自己 `FROM` 子句的 `SELECT` 语句。反过来，那个 `FROM` 子句也可能来自另一个子查询。如果你有一个子查询里面又包含一个子查询，这就是一个**嵌套**子查询。

例如，我们再来看重复的客户姓名。你可以用下面的聚合查询找出候选对象：

```sql
SELECT familyname, givenname
FROM customers
GROUP BY familyname, givenname HAVING count(*)>1;
```

这里只有名字。假设你想要更多细节。为此，你可以将 `customers` 表与前面的查询进行连接：

```sql
SELECT
  c.id, c.givenname, c.familyname, c.email
FROM customers AS c JOIN (
  SELECT familyname, givenname
  FROM customers
  GROUP BY familyname, givenname HAVING count(*)>1
) AS n ON c.givenname=n.givenname AND c.familyname=n.familyname;
```

我们之前见过类似的东西。你现在会得到候选客户：

| **id** | **givenname** | **familyname** | **Email** |
| --- | --- | --- | --- |
| 429 | Corey | Ander | corey.ander429@example.net |
| 287 | Judy | Free | judy.free287@example.com |
| 90 | Ida | Dunnit | ida.dunnit90@example.net |
| 488 | Ken | Tuckey | ken.tuckey488@example.net |
| 174 | Paul | Bearer | paul.bearer174@example.com |
| 505 | Annie | Mate | annie.mate505@example.com |
| ~ 16 行 ~ |

为了方便，我们将 `customers` 表别名为 `c`（别忘了 Oracle 中没有 `AS`），而且子查询无论如何也需要个名字，所以我们叫它 `n`。在 `SELECT` 子句中，我们只是提取了 `id`、姓名和电子邮件地址。

现在，让我们在另一个聚合查询中结合这个结果，这将为我们提供每个名字一行，并合并其他详细信息：

```sql
SELECT
  givenname, familyname,
  -- PostgreSQL, MSSQL:
  string_agg(email,', ') AS email,
  string_agg(cast(id AS varchar(3)),', ') AS ids
  -- MariaDB/MySQL:
  group_concat(email SEPARATOR ', ') AS email,
  group_concat(cast(id AS varchar(3)) SEPARATOR ', ') AS ids
  -- SQLite:
  group_concat(email,', ') AS email,
  group_concat(cast(id AS varchar(3)),', ') AS ids
  -- Oracle:
  listagg(email,', ') AS email,
  listagg(cast(id AS varchar(3)),', ') AS ids
FROM (  -- 将之前的 SELECT 作为子查询
  SELECT c.id, c.givenname, c.familyname, c.email
  FROM customers AS c JOIN (
    SELECT familyname, givenname
    FROM customers
    GROUP BY familyname, givenname HAVING count(*)>1
  ) AS n ON c.givenname=n.givenname AND
            c.familyname=n.familyname
) AS sq
GROUP BY familyname, givenname;
```

现在你会得到类似这样的结果：

| **givenname** | **familyname** | **Email** | **ids** |
| --- | --- | --- | --- |
| Corey | Ander | corey.ander429@e …, corey.ander85@ex … | 429, 85 |
| Paul | Bearer | paul.bearer174@e …, paul.bearer482@e … | 174, 482 |
| Terry | Bell | terry.bell402@ex …, terry.bell295@ex … | 402, 295 |
| Mary | Christmas | mary.christmas46 …, mary.christmas59 … | 465, 594 |
| Ida | Dunnit | ida.dunnit504@ex …, ida.dunnit90@exa … | 504, 90 |
| Judy | Free | judy.free93@exam …, judy.free287@exa … | 93, 287 |
| ~ 8 行 ~ |

注意，我们包含了如 `第 5 章` 所述的 `string_agg()` 函数的变体。此外，我们必须将 `id` 转换为字符串，以便将其聚合成一个字段。

如果你觉得最后一个例子有点难以阅读，嗯，这就是嵌套子查询的问题之一。值得庆幸的是，还有公共表表达式（Common Table Expressions）作为替代方案，我们很快会看到。

## 使用 WHERE EXISTS (子查询)

你已经见过在 `WHERE` 子句中使用子查询来过滤你的查询。到目前为止，子查询返回的是单个值或单个列，用于 `IN()` 表达式。

你也可以使用 `WHERE EXISTS (...)` 来过滤一组数据。它看起来有点像这样：

```sql
SELECT ...
FROM ...
WHERE EXISTS(subquery);
```

子查询要么返回结果，要么不返回。如果返回，那么 `WHERE EXISTS` 条件满足，该行被保留；如果不返回，那么 `WHERE EXISTS` 条件不满足，该行将被过滤掉。

例如，你可以用下面的语句测试这个想法：

```sql
-- PostgreSQL, MSSQL, SQLite
SELECT * FROM authors
WHERE EXISTS (SELECT 1 WHERE 1=1);
-- MariaDB/MySQL, Oracle
SELECT * FROM authors
WHERE EXISTS (SELECT 1 FROM dual WHERE 1=1);
```

因为 `1=1` 总是真，所以你会得到 `authors` 表的所有行。

虽然你通常只在 Oracle 中使用 `FROM dual`，但 MariaDB 和 MySQL 也支持这个。在这种情况下，MariaDB 和 MySQL 不喜欢没有 `FROM` 的 `WHERE` 子句，所以我们加上它来让它们高兴。

同样地，你也可以返回空结果：

```sql
-- PostgreSQL, MSSQL, SQLite
SELECT * FROM authors
WHERE EXISTS (SELECT 1 WHERE 1=0);
-- MariaDB/MySQL, Oracle
SELECT * FROM authors
WHERE EXISTS (SELECT 1 FROM dual WHERE 1=0);
```

子查询处于一个特殊位置，因为实际选择哪些列并不重要：重要的是有没有行存在。这就是为什么我们包含了一个虚拟的 `SELECT 1`。

你也可以选择 `SELECT NULL` 甚至 `SELECT 1/0`。前者会给人一种（错误的）印象，好像我们在查找什么都没有，后者如果单独运行会导致错误。虽然很诱人选择一个更有意义的值来更认真地对待它，但其实没有必要。


#### WHERE EXISTS 与非关联子查询

显然，前面的例子会给你“全部”或“全部不给”的结果，这正是你可以从非关联子查询中预期的行为，例如

```
SELECT * FROM authors
WHERE EXISTS (SELECT 1/0 FROM books WHERE price<15);
```

该子查询选取*一些*行，这足以满足 `WHERE` 子句的条件，因此你将获得*所有*作者。如果你尝试 `WHERE price<0`，那么你将*一个*作者也得不到。

#### WHERE EXISTS 与关联子查询

如果使用关联子查询，`WHERE EXISTS` 会更有趣。此时，测试将针对每一行进行评估，因此一些行会通过，而另一些则不会。

例如，如果你想查找所有在 `books` 表中有书的作者，你可以使用

```
SELECT * FROM authors
WHERE EXISTS (
SELECT 1 FROM books WHERE books.authorid=authors.id
);
```

你将得到类似下面的结果：

| **id** | **givenname** | **…** | **familyname** | **…** | **Home** |
| --- | --- | --- | --- | --- | --- |
| 464 | Ambrose | … | Bierce | … | Meigs County, Ohio |
| 858 | Alexander | … | Ostrovsky | … | Moscow |
| 525 | Francis | … | Beaumont | … | Grace-Dieu, Leicestershire |
| 488 | Bashou | … | Matsuo | … | Matsuo Kinsaku |
| 703 | Friedrich | … | Engels | … | Barmen |
| 722 | Stanley | … | Waterloo | … | St. Clair, Mich. |
| ~ 443 rows ~ |

在这里，子查询查找 `books.authorid` 与 `authors.id` 匹配的行，如果存在这样的行（对大多数作者来说确实存在），则会返回该作者行。

#### WHERE EXISTS 与 IN() 表达式的对比

在前面的例子中，你也可以使用 `IN()` 表达式获得相同的结果：

```
SELECT * FROM authors
WHERE id IN(SELECT authorid FROM books);
```

这种写法当然更简单。然而，很可能在底层，SQL 执行的操作完全相同，所以你如何编写它其实是个口味问题。

另一方面，如果你要查找*没有*书的作者（在我们的目录中），那就是另一回事了。

这*不*能工作：

```
SELECT * FROM authors
WHERE id NOT IN(SELECT authorid FROM books);
```

嗯，严格来说，它*能*工作，但不是我们想要的方式。再次回忆第 3 章中的“NOT IN 怪癖”。由于 `authorid` 列中存在一些 `NULL`，`NOT` `IN` 操作符最终会评估类似 `... AND id=NULL AND ...` 的条件。`id=NULL` 总是失败，并且 `... AND ...` 将此失败与其余部分组合起来，导致整个表达式失败。

然而，使用 `WHERE NOT EXISTS` 将能工作：

```
SELECT * FROM authors
WHERE NOT EXISTS (
SELECT 1 FROM books WHERE books.authorid=authors.id
);
```

这是因为 `WHERE EXISTS` 评估的是行，而不是值：

| **id** | **givenname** | **…** | **familyname** | **…** | **Home** |
| --- | --- | --- | --- | --- | --- |
| 479 | C.E. | … | Koetsveld | … | Rotterdam |
| 874 | Henry | … | Savery | … | Somerset, England |
| 429 | Oliné | … | Keese | … | [NULL] |
| 35 | James | … | Lowell | … | Cambridge, Massachusetts |
| 148 | Demetrius | … | Boulger | … | [NULL] |
| 922 | Robert | … | Ingersoll | … | Dresden, New York |
| ~ 45 rows ~ |

在实际应用中，你不会经常看到 `WHERE EXISTS`，因为通常你可以使用连接（join）或 `IN` 操作符完成相同的事情。然而，有时它具有优势或更直观。这尤其是因为当 `NOT IN` 不起作用时，`WHERE EXISTS` 可以表达得更清晰、更具体。

## LATERAL 连接（又称 CROSS APPLY）及其相关功能

此功能在 SQLite 中不可用。MariaDB 中也不可用，尽管它*确实*在 MySQL 中可用。

不过，如果你正在使用其中一种 DBMS，你可能想了解它是怎么回事。无论如何，对于大多数事情都有替代方案，尤其是在下一节中介绍的公用表表达式（Common Table Expressions）。

然而，SQLite 在 `WHERE` 子句方面有一个有趣的特性，如果你继续阅读就会看到。

如果你尝试以下查询：

```
SELECT
id, title,
price, price*0.1 AS tax, price+tax AS inc
FROM books;
```

它将无法工作。这是因为每一列都是独立的，与其他列无关。你不能在 `SELECT` 子句中将别名用作另一个计算的一部分。我们通过单独计算 `inc` 列来绕过这个问题：`price*1.1 AS inc`。

如果你尝试这样的操作，情况会更糟：

```
SELECT
id, title,
price, price*0.1 AS tax
FROM books
WHERE tax>1.5;
```

这里的问题在于 `SELECT` 子句是在 `WHERE` 子句*之后*评估的，因此在 `WHERE` 子句中，`tax` 的别名计算尚不可用。同样，我们可以在 `WHERE` 子句中重新计算该值：`WHERE price*1.1>1.5`。

但 SQLite 除外。你确实可以在 `WHERE` 子句中使用别名，也可以在 `GROUP BY` 子句中使用。

最后，例如，如果你想在 `SELECT` 子句中从子查询获取多个列，这也不起作用：

```
SELECT
id, title,
(SELECT givenname, othernames, familynames
FROM authors WHERE authors.id=books.authorid)
FROM books
WHERE tax>1.5;
```

`SELECT` 子句中的子查询只能返回一个值，如果你将名称连接起来然后返回结果，那是没问题的。否则，你只能使用三个子查询，这既昂贵又繁琐。

SQL 可以通过对每一行应用子查询来解决这个问题。这在某些 DBMS 中被称为 `LATERAL JOIN`，在另一些中则称为 `APPLY`。

### 添加列

在前面的前两个例子中，你可以使用像这样的表达式：

```
--  PostgreSQL, MySQL (非 MariaDB)
SELECT
id, title,
price, tax, inc
FROM
books
JOIN LATERAL(SELECT price*0.1 AS tax) AS sq ON true
JOIN LATERAL(SELECT price+tax AS inc) AS sq2 ON true
WHERE tax>1.5
;
--  MSSQL
SELECT
id, title,
price, tax, inc
FROM books
CROSS APPLY (SELECT price*0.1 AS tax) AS sq
CROSS APPLY (SELECT price+tax AS inc) AS sq2
WHERE tax>1.5
;
--  Oracle (子查询名称中无 AS)
SELECT
id, title,
price, tax, inc
FROM books
CROSS APPLY (SELECT price*0.1 AS tax FROM dual) sq
CROSS APPLY (SELECT price+tax AS inc FROM dual) sq2
WHERE tax>1.5
;
```

你将得到类似这样的结果：

| **id** | **Title** | **price** | **tax** | **inc** |
| --- | --- | --- | --- | --- |
| 503 | Uncle Silas | 17.00 | 1.700 | 18.700 |
| 2007 | North and South | 17.50 | 1.750 | 19.250 |
| 702 | Jane Eyre | 17.50 | 1.750 | 19.250 |
| 1759 | La Curée | 16.00 | 1.600 | 17.600 |
| 205 | Shadow: A Parable | 17.50 | 1.750 | 19.250 |
| 1702 | Philaster | 17.50 | 1.750 | 19.250 |
| ~ 525 rows ~ |

注意

.

*   子查询必须被赋予一个别名，即使它没有被使用。

*   PostgreSQL、MySQL 和 MSSQL 允许你将列别名放在子查询别名中：`(SELECT price*0.1) AS sq(tax)`。Oracle 不行。

*   PostgreSQL 和 MySQL 的例子使用了虚拟条件 `ON true`。MySQL 允许你省略它，但 PostgreSQL 要求必须有它。

尤其要注意，第二个子查询会愉快地计算表达式 `price+tax AS inc`。这是因为子查询是依次评估的，所以表达式可以累积。

`LATERAL` 或 `CROSS APPLY` 子查询应用于主查询的每一行。原则上，这可能相当昂贵，但事实证明，情况并非如此糟糕。如果你需要在更复杂的计算中包含一系列中间步骤，它特别有用——它易于理解和维护。

SQL 还有一种称为 `CROSS JOIN` 的连接类型。在交叉连接中，一个表的每一行与另一个表的每一行连接。此结果也称为笛卡尔积。那是大量的组合，通常不是你想要的。

`CROSS APPLY` 与此不同，尽管它是一种连接类型。它更接近于 `OUTER JOIN`。

稍后当我们与单行虚拟表进行交叉连接时，你会看到交叉连接的一个用途。


## 使用公用表表达式

### 多列查询

如前所述，SQL 不允许在 `SELECT` 子句中从单个子查询获取多列，因为 `SELECT` 子句中的所有内容都应是标量——即单个值。

然而，如果上下文是类似表的结构（例如在 `FROM` 子句中），则可以获取多列。例如：

```sql
--  PostgreSQL, MySQL (非 MariaDB)
SELECT
id, title,
givenname, othernames, familyname
FROM
books
LEFT JOIN LATERAL(
SELECT givenname, othernames, familyname
FROM authors
WHERE authors.id=books.authorid
) AS a ON true;
--  MSSQL
SELECT
id, title,
givenname, othernames, familyname,
home
FROM books
OUTER APPLY (
SELECT givenname, othernames, familyname
FROM authors
WHERE authors.id=books.authorid
) AS a;
--  Oracle: 与 MSSQL 相同，但不使用 AS
```

结果类似于：

| **id** | **标题** | **givenname** | **othernames** | **familyname** |
| --- | --- | --- | --- | --- |
| 2078 | The Duel | Heinrich | [NULL] | von Kleist |
| 503 | Uncle Silas | J. | Sheridan | Le Fanu |
| 2007 | North and South | Elizabeth | [NULL] | Gaskell |
| 702 | Jane Eyre | Charlotte | [NULL] | Brontë |
| 1530 | Robin Hood, The … | Alexandre | [NULL] | Dumas |
| 1759 | La Curée | Émile | [NULL] | Zola |
| ~ 约 1201 行 ~ |

在这种情况下，使用普通的外连接同样可以轻松获得相同的结果：

```sql
SELECT
books.id, title,
givenname, othernames, familyname,
home
FROM books LEFT JOIN authors ON authors.id=books.authorid;
```

后一种形式显然更简单（为了简洁我们省略了表别名，并且出于必要对 `books.id` 列进行了限定）。

另一方面，如果子查询是聚合查询，横向连接就非常方便，因为你反正需要一个子查询：请记住，你不能在单个 `SELECT` 语句中混合聚合数据和非聚合数据。

例如，假设你想要一个客户列表及其各自的总销售额。你需要一个聚合查询来获取总和，再与客户表连接。你可以这样做：

```sql
--  PostgreSQL, MySQL (非 MariaDB)
SELECT
id, givenname, familyname, total
FROM
customers
LEFT JOIN LATERAL(
SELECT sum(total) AS total FROM sales
WHERE sales.customerid=customers.id
) AS totals ON true;
--  MSSQL, Oracle (不使用 AS)
SELECT
id, givenname, familyname, total
FROM
customers
OUTER APPLY(
SELECT sum(total) AS total FROM sales
WHERE sales.customerid=customers.id
) AS totals;
```

这将得到类似这样的结果：

| **Id** | **givenname** | **familyname** | **total** |
| --- | --- | --- | --- |
| 42 | May | Knott | 3437.72 |
| 459 | Rick | Shaw | 461.00 |
| 597 | Ike | Andy | [NULL] |
| 186 | Pat | Downe | 1536.50 |
| 352 | Basil | Isk | 573.00 |
| 576 | Pearl | Divers | [NULL] |
| ~ 约 303 行 ~ |

尽管可能存在替代方案（在使用 SQLite 或 MariaDB 时你将会需要），但横向连接有时可以让这类查询更加直观。

### 语法

我们已经看到可以在 `FROM` 子句中使用子查询，甚至可以嵌套它们。然而，还有一种替代方法可以使处理这些子查询更加自然。

公用表表达式，朋友们简称 CTEs，是生成表结果的子查询。因此，你几乎总是可以用 CTE 替换 `FROM` 子查询。

我们在前面的章节中已经使用过 CTEs，所以如果你有种*似曾相识（déjà vu）*的感觉，那是正常的。

公用表表达式（CTEs）是 SQL 中相对较新的功能，但已存在一段时间，并且在几乎所有现代 DBMS 中都可用。值得注意的落后者是 MariaDB（在 2016 年发布的 10.2 版本中添加了支持）和 MySQL（在 2018 年发布的 8.0 版本中添加了支持）。如果你被困在旧版本的 MariaDB 或 MySQL 上，也许可以学着享受嵌套子查询。

CTE 是在使用前定义的虚拟表。它与子查询相似，因为它包含一个 `SELECT` 语句，但语法不同，并且具有许多优点。

最简单形式的 CTEs 是子查询的替代形式。即便如此，也有一个直接的好处：

-   CTEs 更具可读性且更易于维护，因为它们在查询开头定义，而不是在中间定义。

复杂的子查询可能有一个子查询引用另一个子查询。这涉及子查询的嵌套。

-   CTEs 可以引用先前的 CTEs，而无需嵌套。

这两个好处都与可读性和可维护性有关。第三个好处是普通子查询所不具备的。

-   CTEs 可以引用自身；因此，它们可以是递归的。

一个公用表表达式作为查询的一部分定义在主查询部分之前：

```sql
WITH cte AS (subquery)
SELECT columns FROM cte;
```

CTE 被赋予一个名称，当然不一定非得是 `cte`。之后，它在主查询中被当作普通表使用。你可以如下定义多个 CTEs：

```sql
WITH
cte AS (subquery),
another AS (subquery)
SELECT columns FROM ...;
```


#### 使用公用表表达式（CTE）准备计算

还记得带有价格组的那个子查询吗？

```
SELECT price_group, count(*) AS num_books
FROM (
SELECT
id, title,
CASE
WHEN price < 17 THEN 'cheap'
WHEN price < 30 THEN 'affordable'
WHEN price >= 17 THEN 'expensive'
END AS price_group
FROM books
) AS sq     --  Oracle: ( ... ) sq
GROUP BY price_group;
```

这可以轻松地重写为 CTE：

```
--  准备数据
WITH sq AS (
SELECT
id, title,
CASE
WHEN price < 17 THEN 'cheap'
WHEN price < 30 THEN 'affordable'
WHEN price >= 17 THEN 'expensive'
END AS price_group
FROM books
)
--  使用准备好的数据
SELECT price_group, count(*) AS num_books
FROM sq
GROUP BY price_group;
```

它看起来差别不大，但重要的是你现在将查询分成了两部分：第一部分定义子查询，第二部分使用它。这是一种组织代码的更好方式。

子查询在查询的开头被转移到一个 CTE 中。从那时起，主`SELECT`语句引用该 CTE，就好像它是另一张表一样。

优点是查询是按照计划编写的：先准备数据，然后使用数据。

目前 MSSQL 不要求在语句末尾使用分号，但你应该养成使用它的习惯。

然而，`WITH`子句在前一个`SELECT`语句末尾有另一种含义，因此如果你不以分号结束前一个`SELECT`语句，它将被错误解释。

只需在每个语句末尾使用分号，一切都会很好。不要听信这种无稽之谈：

`;WITH (...)`

这是另一个例子，我们将在接下来的几章中进一步使用它。如果你查看`sales`表：

```
SELECT * FROM sales;
```

你将看到以下内容：

| **id** | **customerid** | **total** | **…** | **ordered_date** |
| --- | --- | --- | --- | --- |
| 39 | 28 | 28.00 | … | 2022-05-15 |
| 40 | 27 | 34.00 | … | 2022-05-16 |
| 42 | 1 | 58.50 | … | 2022-05-16 |
| 43 | 26 | 50.00 | … | 2022-05-16 |
| 45 | 26 | 17.50 | … | 2022-05-16 |
| 518 | 50 | 13.00 | … | [NULL] |
| ~ 5549 行 ~ |

如果你想汇总该表，例如获取每月总计，数据就太细粒度了。相反，你可以通过将`ordered_date`格式化为年-月值来准备数据：

```
WITH salesdata AS (
SELECT
--  PostgreSQL, Oracle
to_char(ordered_date,'YYYY-MM') AS month,
--  MariaDB/MySQL
--  date_format(ordered_date,'%Y-%m') AS month,
--  MSSQL
--  format(ordered_date,'yyyy-MM') AS month,
--  SQLite
--  strftime('%Y-%m',ordered_date) AS month,
total
FROM sales
)
SELECT month, sum(total) AS monthly_total
FROM salesdata
GROUP BY month
ORDER BY month;
```

现在你将得到以下汇总：

| **月份** | **月度总计** |
| --- | --- |
| 2022-05 | 6966.50 |
| 2022-06 | 12733.00 |
| 2022-07 | 17314.00 |
| 2022-08 | 19093.00 |
| 2022-09 | 20295.50 |
| 2022-10 | 27797.50 |
| ~ 14 行 ~ |

在现实生活中，你想汇总的许多内容格式都不正确，但你可以在 CTE 中准备它，使其就绪。

我们将在第 9 章再次讨论 CTE，届时我们将看到更多可应用的技术。

