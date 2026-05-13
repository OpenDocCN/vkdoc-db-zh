# SQL 查询：结果选择与连接类型

## 选择结果

你总是可以使用`SELECT *`来查看情况：

```sql
SELECT *
FROM paintings JOIN artists ON paintings.artistid=artists.id;
```

但是，你可能希望更有选择性地包含列。例如，你可能会选择画作的`id`、`title`，以及艺术家的`givenname`和`familyname`：

```sql
--  这样不行：
SELECT
id,
title,
givenname, familyname
FROM paintings JOIN artists
ON paintings.artistid=artists.id;
```

但它还不能工作。

你得到的错误信息将指向一个`ambiguous`（模糊的）列。在单个表中，不能有两个同名的列，但当你连接两个不同的表时，总是存在这种可能性。

解决方案是包含表名作为前缀：

```sql
SELECT
paintings.id,
title,
givenname, familyname
FROM paintings JOIN artists ON paintings.artistid=artists.id;
```

这样现在就能工作了，因为你已经`qualified`（限定）了那个麻烦的名称；也就是说，你指明了是哪一个。

| id | title | givenname | familyname |
| --- | --- | --- | --- |
| 1222 | Haymakers Resting | Camille | Pissarro |
| 251 | Death in the Sickroom | Edvard | Munch |
| 2190 | Cache-cache (Hide-and-Seek) | Berthe | Morisot |
| 1560 | Indefinite Divisibility | Yves | Tanguy |
| 172 | Girl with Racket and Shuttlecock | Jean-Baptiste-Siméon | Chardin |
| 2460 | The Procession to Calvary | Pieter the Elder | Bruegel |
| ~ 1228 rows ~ |

表示法`table.column`被称为限定名，可以看作是列的全名。

注意，在结果集中，列名仍然是更简单的非限定名。

虽然，如你所见，只有模糊的名称才需要限定，但你可以限定其余的列：

```sql
SELECT
paintings.id,
paintings.title,
artists.givenname, artists.familyname
FROM paintings JOIN artists ON paintings.artistid=artists.id;
```

你将得到完全相同的结果。

事实上，建议你限定所有的列。这是因为：
*   你可以很容易地看出哪个表拥有原始数据。
*   更复杂的连接将更容易管理。
*   如果有人决定在另一个表中添加一个同名的列，你无论如何都需要限定它。

从这里开始，我们将限定所有名称。

### 表别名

如果你打算完全限定每个列名，它很快就会变得非常繁琐。它也使得语句更难阅读，因为你需要费力地浏览多个冗余的表名。你可以通过使用表`alias`（别名）来简化语句，它是表的临时昵称。这类似于计算列值时使用的列别名。

要使用表别名：
*   使用`AS`关键字为表起别名。
*   使用别名*代替*原始名称。

请注意，在 Oracle 中，你将不得不在*不使用*`AS`关键字的情况下为表起别名。

例如：

```sql
--  非 Oracle 数据库
SELECT
p.id,
p.title,
a.givenname, a.familyname
FROM paintings AS p JOIN artists AS a ON p.artistid=a.id;
--  Oracle 数据库，以及其他数据库：
SELECT
p.id,
p.title,
a.givenname, a.familyname
FROM paintings p JOIN artists a ON p.artistid=a.id;
```

如果你不使用 Oracle，我们建议你总是使用`AS`来使你的代码更具可读性并减少错误；对于 Oracle，你没有这个选项。

使用表别名有很多原因，但在本例中，纯粹是为了方便。将表名简化为一个字母更容易输入，也*更*容易扫描。

记住`FROM`在`SELECT`之前被求值，要注意一旦设置了表别名，你就必须在语句的其余部分使用它：你不能将原始名称与别名混合使用。

## 开发价格列表

我们在使用计算时已经开始构建价格列表。使用连接，我们可以轻松地开发一个：

```sql
SELECT
p.id,
p.title,
a.givenname||' '||a.familyname AS artist,
a.nationality,
p.price, p.price*0.1 AS tax, p.price*1.1 AS total
FROM paintings AS p JOIN artists AS a ON p.artistid=a.id;
```

你应该得到相同的结果：

| id | title | artist | price | tax | inc | nationality |
| --- | --- | --- | --- | --- | --- | --- |
| 1222 | Haymakers Restin … | Camille Pissarro … | 125.00 | 12.500 | 137.500 | French |
| 251 | Death in the Sic … | Edvard Munch … | 105.00 | 10.500 | 115.500 | Norwegian |
| 2190 | Cache-cache (Hid … | Berthe Morisot … | 185.00 | 18.500 | 203.500 | French |
| 1560 | Indefinite Divis … | Yves Tanguy … | 125.00 | 12.500 | 137.500 | French |
| 172 | Girl with Racket … | Jean-Baptiste-Si … | 195.00 | 19.500 | 214.500 | French |
| 2460 | The Procession t … | Pieter the Elder … | 165.00 | 16.500 | 181.500 | Flemish |
| ~ 1228 rows ~ |

当然，除了有一些行仍然缺失。

你也可以添加`p.year`或其他列，无需额外工作，因为它们在连接中总是存在的。

### 连接类型

到目前为止，我们忽略了有些行缺失的事实。

原因很简单。一些画作没有匹配的艺术家，一些艺术家没有匹配的画作。

要找到未匹配的（匿名）画作，你可以找到`artistid`为`NULL`的记录：

```sql
SELECT *
FROM paintings
WHERE artistid IS NULL;
```

这会给你匿名画作：

| id | title | artistid | … |
| --- | --- | --- | --- |
| 1989 | Portrait of Trabuc |   | … |
| 1179 | The Green Vineyard |   | … |
| 1774 | The vase with 12 sunflowers |   | … |
| 625 | Self-portrait |   | … |
| 908 | Village Street and Stairs with Figures |   | … |
| 2220 | Noon: Rest from Work (After Millet) |   | … |
| ~ 45 rows ~ |

找到未匹配的艺术家则有点棘手，因为匹配只在`paintings`表中定义：
*   找到那些在`paintings`表中*确实*存在的`artistid`。
*   找到其`id`不在上述列表中的艺术家。

```sql
SELECT * FROM artists
WHERE id NOT IN (
SELECT artistid FROM paintings WHERE artistid IS NOT NULL
);
```

这会给你一个我们尚未包含的艺术家列表：

| id | givenname | familyname | … |
| --- | --- | --- | --- |
| 37 | Francisco | de Zurbarán | … |
| 127 | Pietro | Cavallini | … |
| 38 | Altichiero | Da Verona | … |
| 164 | Alonso | Cano | … |
| 23 | Paul | Klee | … |
| 338 | Gentile | da Fabriano | … |
| ~ 19 rows ~ |

当你连接两个表时，默认情况下，只包含一个表中与另一个表中有匹配行的那些行。在我们的例子中，由于一些画作有`NULL artistid`，SQL 无法在`artists`表中为它们找到匹配项。未匹配的画作被排除在外。这被称为`INNER JOIN`（内连接），是默认的连接方式。

这并不总是你想要的，所以 SQL 提供了多种连接类型：
*   `INNER JOIN`（内连接）只返回匹配的行。这是默认的。
*   `OUTER JOIN`（外连接）类型返回匹配的行以及一些未匹配的行。有三种不同的`OUTER JOIN`类型。
*   `CROSS JOIN`（交叉连接）返回子行和父行的所有可能组合。你很少单独想要`CROSS JOIN`，但它有时用于特殊情况。

在这里，我们将探讨各种类型的连接。为了使它更容易，我们将在`artists`和`paintings`表的微型版本上运行示例，称为`miniartists`和`minipaintings`。这些是你在图 6-1 中看到的表。


### INNER JOIN

默认的连接类型是 `INNER JOIN`：

| id | title | artistid | id | fullname |
| --- | --- | --- | --- | --- |
| 1 | 浴女 | 2 | 2 | 皮埃尔-奥古斯特·雷诺阿 |
| 2 | 星夜 | 1 | 1 | 文森特·梵高 |
| 3 | 暴风雪 | 3 | 3 | 约瑟夫·透纳 |
| 4 | 舞者 | 2 | 2 | 皮埃尔-奥古斯特·雷诺阿 |
| 5 | 月光 | 3 | 3 | 约瑟夫·透纳 |
| 7 | 阿尔勒女人 | 1 | 1 | 文森特·梵高 |
| 8 | 在露台上 | 2 | 2 | 皮埃尔-奥古斯特·雷诺阿 |

```sql
SELECT *
FROM minipaintings AS p
INNER JOIN miniartists AS a ON p.artistid=a.id;
```

这与之前示例的唯一区别是多了 `INNER` 这个词，你可以看到它并没有真正做任何特别的事情。既然是默认的，你完全可以省略它。

请记住，使用 `INNER JOIN` 时，表名的顺序可以互换。你会得到相同的结果，但列的顺序可能会改变。

### LEFT OUTER JOIN 和 RIGHT OUTER JOIN

`OUTER JOIN` 用于包含未匹配的行。它 *总是会包括* `INNER JOIN` 的结果。

由于涉及两个表，你可以选择包含第一个表还是第二个表的未匹配行。如果你真的想包含两者的未匹配行，你会使用 `FULL OUTER JOIN`。

然而，你可能更想包含来自 `minipaintings` 表的未匹配行，即 `minipaintings` 表中那些在 `miniartists` 表中没有匹配行的记录。用英文说，就是匿名画作。

如前所述，表名的顺序可以互换。但是，当涉及到 `OUTER JOIN` 时，你需要记住哪个表是你先写的。

如果你想包含 `minipaintings` 表的 `unmatched` 行，你需要将 `INNER JOIN` 改为相应的 `LEFT OUTER JOIN` 或 `RIGHT OUTER JOIN`。

如果你使用的是 SQLite，你会发现它不支持 `RIGHT JOIN`。他们建议你只需调换两个表的顺序，直到你能用 `LEFT JOIN` 得到想要的结果。

你仍然可以参考后面的解释，但请忽略包含 `RIGHT JOIN` 的示例。

将 `minipaintings` 表放在左边：

```sql
SELECT *
FROM minipaintings AS p LEFT OUTER JOIN miniartists AS a
ON p.artistid=a.id;
```

你会看到现在得到了内连接中缺失的匿名画作：

| id | title | artistid | id | fullname |
| --- | --- | --- | --- | --- |
| 1 | 浴女 | 2 | 2 | 皮埃尔-奥古斯特·雷诺阿 |
| 2 | 星夜 | 1 | 1 | 文森特·梵高 |
| 3 | 暴风雪 | 3 | 3 | 约瑟夫·透纳 |
| 4 | 舞者 | 2 | 2 | 皮埃尔-奥古斯特·雷诺阿 |
| 5 | 月光 | 3 | 3 | 约瑟夫·透纳 |
| 6 | 绿色葡萄园 |   |   |   |
| 7 | 阿尔勒女人 | 1 | 1 | 文森特·梵高 |
| 8 | 在露台上 | 2 | 2 | 皮埃尔-奥古斯特·雷诺阿 |

将 `minipaintings` 表放在右边：

```sql
SELECT *
FROM miniartists AS a RIGHT OUTER JOIN minipaintings AS p
ON a.id=p.artistid;
```

也就是说，根据你想要包含未匹配行的表是在 `JOIN` 关键字的左边还是右边，来选择使用 `LEFT` 还是 `RIGHT`。`ON artists.id=paintings.artistid` 子句的写法仍然可以互换。

与 `INNER JOIN` 一样，`OUTER` 这个词也是可选的，所以你可以省略它：

```sql
--  minipaintings 表在左边
SELECT *
FROM minipaintings AS p LEFT JOIN artists AS a
ON p.artistid=a.id;
--  paintings 表在右边
SELECT *
FROM miniartists AS a RIGHT JOIN minipaintings AS p
ON paintings.artistid=artists.id;
```

对于未匹配的行，相应的数据将填充为 `NULL`。

### “首选”的外连接

在我们的示例中，我们使用 `OUTER JOIN` 来包含所有的画作，但不包含所有的艺术家。这部分是因为我们对画作更感兴趣，因为那是我们销售的商品。这也是由表之间的关系决定的。

注意，`minipaintings` 表有一个指向 `miniartists` 的外键，但反之则不然。我们说这两个表之间存在 `一对多` 的关系。一个艺术家可以有多幅画作，或者，如果你愿意，也可以说多幅画作可以引用同一位艺术家。

有时，我们说 `minipaintings` 表是 `子表`，而 `miniartists` 表是 `父表`。在 SQL 中，子表引用父表，但反之则不然。

当你连接两个表时，你通常想要的是子表的外连接。然而，你也可以得到父表的外连接：

```sql
--  minipaintings 表在左边
SELECT *
FROM minipaintings AS p RIGHT JOIN miniartists AS a
ON p.artistid=a.id;
--  minipaintings 表在右边
SELECT *
FROM artists AS a LEFT JOIN minipaintings AS p
ON p.artistid=a.id;
```

这将产生 `INNER JOIN` 的结果加上未匹配的艺术家。这并不常见，但在少数情况下可能有用。

首先，在这个数据库中，你对画作更感兴趣，并将艺术家信息视为画作的额外细节。然而，如果你是艺术家的经纪人（在本例中可能为时已晚），你会对 `miniartists` 表更感兴趣，那么你可能会对之前提到的父表外连接感兴趣。

其次，你可以使用这种连接来查找未匹配的艺术家。例如：

```sql
SELECT a.*
FROM minipaintings AS p RIGHT JOIN miniartists AS a
ON p.artistid=a.id
WHERE p.id IS NULL;
```

这会给你以下结果：

| id | fullname |
| --- | --- |
| 5 | 列奥纳多·达·芬奇 |
| 4 | 安藤广重 |

在这里，你通过生成外连接并过滤掉已匹配的画作，来寻找没有对应画作的艺术家。这将给你一个与之前“未匹配艺术家”类似的结果。

### 关于 JOIN 的一些建议

在编写连接时，你显然有一些选择。只要能工作，具体采用哪种方式并不重要。然而，一个好的开发人员总是会编写清晰且易于维护的代码。

以下是一些建议。最终，无论你选择什么，都应该保持一致，不要频繁改变规则。

#### （几乎）总是为表使用别名

在极少数情况下，你可能会在不给表起别名的情况下编写连接，但一旦你开始选择特定的列，情况就会变得混乱。给表起别名总是一个好主意。

#### 哪个表放在前面？

显然，我们可以选择先写哪个表。你如何决定？

首先，这无关紧要，所以即使你选择另一种顺序，也不会导致出错。然而，在实践中，你可能对一个表比另一个表更感兴趣。

如果你是艺术家的经纪人，那么你会更倾向于列出艺术家及其画作。在这个示例中，我们销售的是画作，所以我们更倾向于列出画作及其艺术家。

通常，把你更感兴趣的表放在左边。这意味着，当涉及到外连接时，你更有可能使用 `LEFT JOIN`。这也是我们在本书中主要采用的方式。

请记住，SQLite 甚至不支持 `RIGHT JOIN`，并且他们建议你在必要时交换表的顺序以使用 `LEFT JOIN`。显然，没有人怀念 `RIGHT JOIN`。

#### 决定是否使用 INNER 和 OUTER

`INNER` 和 `OUTER` 都是可选的，所以原则上你可以随心所欲。然而，我们建议你选定一个规则并坚持下去。如果你有时使用这些关键字，有时又不使用，可能会引起混淆和误解。

在本书中，我们省略了它们，但在现实中，这取决于你。



