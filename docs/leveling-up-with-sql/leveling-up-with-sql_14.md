# 6. 使用视图及其相关对象

这不是你第一次在书中看到视图，也不会是最后一次。

本章整合了你可能已经了解到的关于视图和相关概念的知识，并提供了一些关于如何一般性地使用它们的想法。

你可以用一生来编写 SQL 语句，工作也能完成。然而，你可能会达到一个点，即一遍又一遍地写同样的东西失去了魅力，因此你会想要找到重用先前查询的方法。

首先，让我们看看表是什么，以及当你使用 `SELECT` 语句时会发生什么。

SQL 数据库将数据存储在表中。实际上，并非如此——每个表实际上存储在其他某种结构中，例如二进制树，这样更高效。然而，当你看到它时，它将被呈现为一个表，并且在数据库中就是这样称呼的。

表由行和列组成。出于我们的目的，表不一定是永久表，并且有些操作会生成表结构而不一定被永久存储。我们将它们称为 **虚拟表**。

以下是一些生成（虚拟）表的操作，按持久性递增的顺序排列：

*   `SELECT` 语句的结果是一个虚拟表。
*   一个 **连接** 是两个或多个（虚拟）表的组合，以产生一个扩展的虚拟表。
*   一个 **公用表表达式** 生成一个虚拟表，你可以在查询的后面部分使用它。表子查询本质上是相同的东西。
*   一个 **视图** 是一个保存的 `SELECT` 查询，将在调用时重新生成虚拟表。
    一个 **物化视图** 是一种存储其结果的视图，因此不必总是重新生成所有内容。
    一些 DBMS 支持 **表值函数**，这些是生成虚拟表的函数。
*   一个 **临时表** 就像一个真实的表。它可能存储在磁盘上，也可能不存储。当会话结束时，它会自动销毁。

关于表和虚拟表的关键点是，它们都可以在 `FROM` 子句中使用。

你已经知道如何使用连接。你也使用过公用表表达式，我们将在接下来的章节中更深入地讨论它们。

在本章中，我们将研究其余的对象，以及我们如何利用它们来改进我们的工作流程。



## 处理视图

到目前为止，你已经创建过一两个视图了，所以对这部分内容应该比较熟悉。

视图是一个保存在数据库中的 `SELECT` 语句。当视图被保存时，数据库管理系统（DBMS）通常也会保存其执行计划，因此下次规划相同查询时就不会浪费太多时间。

任何你需要从表中读取数据的地方，都可以使用视图。你可以把视图看作一个虚拟表——其行为表现得就像一个表。

要从视图中读取数据，你就像对待表一样对待它：
```
SELECT columns FROM view;
```

请注意，其语法与表的语法完全相同。从 `SELECT` 语句的角度来看，从视图中选择数据和从表中选择数据没有区别。

一个重要的结果是，你不能有与表同名的视图——视图和表共享同一个命名空间。

这并不意味着它们没有区别。数据库管理系统将视图存储为不同类型的对象，并以不同的方式管理它们。然而，一旦创建，你就可以像对待表一样对待视图。

视图可以成为你工作流中的重要组成部分。例如：
-   保存复杂的查询以供简单使用
-   向外部应用程序暴露复杂的查询
-   在现有数据和应用程序之间创建接口
-   作为某些查询的替代方案

创建视图需要权限，而你作为数据库用户可能尚未拥有这些权限。请采取必要行动来获取这些权限——根据需要，去恳求、贿赂或要挟吧。

视图有几个好处：
-   你可以用一个只返回结果虚拟表的视图来隐藏复杂的处理过程。
-   你可以通过创建只包含特定行和列的视图来限制对表中数据的访问。
-   你可以隐藏聚合查询中的细节。

也存在一些限制。主要限制是视图不够灵活。例如，你无法改变视图可能用来计算结果的值。稍后我们讨论表值函数时，你会看到一个可能的解决方案。

其他限制因数据库管理系统而异。一些数据库管理系统支持临时视图。MSSQL 不允许在视图中使用 `ORDER BY`，除非使用一些额外技巧。一些数据库管理系统支持在临时表上创建视图，而另一些则不支持。

然而，在大多数情况下，视图将简化你的工作流程，使你能够转向更复杂的任务。

### 创建视图

视图始于一个简单的 `SELECT` 语句。例如，我们可以开始开发一个 `pricelist` 视图，它将包含有关书籍、其作者和价格（含税）的一些信息：
```
/* 注释
===================================================
MSSQL:  使用 + 进行拼接
Oracle: 表别名不能使用 AS:
FROM books b JOIN authors a ON ...
=================================================== */
SELECT
b.id, b.title, b.published,
coalesce(a.givenname||' ','')
|| coalesce(othernames||' ','')
|| a.familyname AS author,
b.price, b.price*0.1 AS tax, b.price*1.1 AS inc
FROM books AS b LEFT JOIN authors AS a ON b.authorid=a.id
WHERE b.price IS NOT NULL;
```

这应该会给你类似这样的结果：

| `id` | `title` | `pub` | `author` | `price` | `tax` | `inc` |
| --- | --- | --- | --- | --- | --- | --- |
| 2078 | The Duel | 1811 | Heinrich von Kleis ... | 12.50 | 1.25 | 13.75 |
| 503 | Uncle Silas | 1864 | J. Sheridan Le Fan ... | 17.00 | 1.70 | 18.70 |
| 2007 | North and South | 1854 | Elizabeth Gaskell | 17.50 | 1.75 | 19.25 |
| 702 | Jane Eyre | 1847 | Charlotte Brontë | 17.50 | 1.75 | 19.25 |
| 1530 | Robin Hood, The Pr ... | 1862 | Alexandre Dumas | 12.50 | 1.25 | 13.75 |
| 1759 | La Curée | 1872 | Émile Zola | 16.00 | 1.60 | 17.60 |
| ~ 1096 行 ~ |

由于这是一个价格列表，我们过滤掉了 `NULL` 价格。

但是，需要满足某些条件：
-   视图中的所有列都必须有名称，包括计算列。你不能像在简单的 `SELECT` 语句中那样拥有匿名列。
-   列名必须是唯一的。这在连接表时尤其重要，因为列名可能会重复。

换句话说，虚拟表必须符合真实表的规则。

要创建视图，需要在 `SELECT` 语句前加上 `CREATE VIEW ... AS` 子句：
```
/* 注释
===================================================
MSSQL:  使用 + 进行拼接
MSSQL:  用 GO 包围 CREATE VIEW
Oracle: 表别名不能使用 AS:
FROM books b JOIN authors a ON ...
=================================================== */
--  GO
CREATE VIEW aupricelist AS
SELECT
b.id, b.title, b.published,
coalesce(a.givenname||' ','')
|| coalesce(othernames||' ','')
|| a.familyname AS author,
b.price, b.price*0.1 AS tax, b.price*1.1 AS inc
FROM books AS b JOIN authors AS a ON b.authorid=a.id;
--  GO
```

这将被保存到你的数据库中。

我们将这个价格列表命名为 `aupricelist`，因为税率设为 10%，这是澳大利亚的税率。你可以随意使用任何你喜欢的税率和名称。

-   仅在 Microsoft SQL Server 中，你需要将 `CREATE VIEW` 语句与脚本的其余部分分开。通过在前后放置 `GO` 来实现，`GO` 标记了他们所谓的“批处理”。`GO` 必须单独占一行。

现在，你可以像读取表一样读取相应的结果：
```
SELECT * FROM aupricelist;
```

这将给你与之前相同的结果。

你可以像过滤表一样过滤视图：
```
SELECT *
FROM aupricelist
WHERE published BETWEEN 1700 AND 1799;
```

这会得到：

| `id` | `title` | `pub` | `author` | `price` | `tax` | `inc` |
| --- | --- | --- | --- | --- | --- | --- |
| 1608 | The Autobiography ... | 1791 | Benjamin Franklin | 18.50 | 1.85 | 20.35 |
| 2303 | The Metaphysics of ... | 1797 | Immanuel Kant | 12.00 | 1.20 | 13.20 |
| 1305 | An Essay on Critic ... | 1711 | Alexander Pope | 11.00 | 1.10 | 12.10 |
| 1963 | A Treatise of Huma ... | 1740 | David Hume | 18.50 | 1.85 | 20.35 |
| 1196 | Equiano’s Travels: ... | 1789 | Olaudah Equiano | 12.50 | 1.25 | 13.75 |
| 1255 | Discourse on the O ... | 1755 | Jean-Jacques Rouss ... | 19.00 | 1.90 | 20.90 |
| ~ 166 行 ~ |

你也可以对结果进行排序：
```
SELECT * FROM aupricelist ORDER BY title;
```

这会得到：

| `id` | `title` | `pub` | `author` | `price` | `tax` | `inc` |
| --- | --- | --- | --- | --- | --- | --- |
| 541 | 120 Days of Sodom | 1904 | Marquis de Sade | 12.50 | 1.25 | 13.75 |
| 729 | A Cartomante e Out ... | 1884 | Machado de Assis | 16.00 | 1.60 | 17.60 |
| 2092 | A Chaste Maid in C ... | 1613 | Thomas Middleton | 15.00 | 1.50 | 16.50 |
| 1437 | A Child’s Garden o ... | 1885 | Robert Louis Steve ... | 11.00 | 1.10 | 12.10 |
| 454 | A Christmas Carol | 1843 | Charles Dickens | 13.50 | 1.35 | 14.85 |
| 1094 | A Confession | 1882 | Leo Tolstoy | 17.50 | 1.75 | 19.25 |
| ~ 1096 行 ~ |

除了 MSSQL，你本可以将 `ORDER BY` 子句直接包含在视图本身中。虽然这很方便，但这可能不是一个好主意：无论你是否需要，它都会强制数据库管理系统对结果进行排序，而且你可能之后还需要以不同的顺序再次排序。

### 在 MSSQL 中使用 ORDER BY

如前所述，SQL Server 不允许在视图中使用 `ORDER BY`，除非使用一些额外技巧。如果你*真的*想要一个预排序的视图，可以这样做：
```
CREATE VIEW something AS
SELECT columns
FROM table
ORDER BY columns OFFSET 0 ROWS;
```

这样做除了其他好处外，还可以让你创建有序视图，而无需仅仅为了排序而包含额外的列。

但是，你需要意识到有序视图确实会给数据库带来额外负担，因此应仅在需要时使用。

### 使用视图的技巧

视图通常是个好东西，但有一些技巧可以帮助你更可靠地使用它们。一些细节会因数据库管理系统而异，并且数据库管理系统会尽其所能高效工作。尽管如此，牢记这些想法是明智的。


#### 不要过度级联视图

你可以基于其他视图构建视图。这有助于简化工作流程，但需要谨慎：

*   修改底层视图可能会导致该视图混乱。某些数据库管理系统甚至不允许你更改视图，如果另一个视图依赖于它的话。
*   SQL 会尝试优化你的查询，但如果视图嵌套层级过深，它可能无法很好地进行优化。
*   其中一个视图可能包含了新视图所需的更多内容，因此你是在浪费处理时间来生成你不需要的数据。

这并不意味着你不应该在现有视图的基础上构建，而是说你应该明智地这样做。

#### 不要使用 SELECT *

`SELECT *` 很可能无法完全满足你的需求，但即使它能，你可能也应该明确列出所有列。某些数据库管理系统甚至会将 `SELECT *` 转换为列列表。

`SELECT *` 可能不是一个好主意的原因包括：

*   你可能希望自定义列的顺序，而不是使用默认顺序。
*   底层表的结构可能会发生变化，因此你的视图最终可能并非你最初预期的样子。

#### 避免使用 ORDER BY

大多数数据库管理系统允许你在视图中包含 `ORDER BY` 子句，即使在 MSSQL 中，你也可以强制使用它。然而，这可能不是一个好主意。

你很可能会希望以各种方式对视图进行排序。即使你偏好一种排序顺序，你也不希望数据库管理系统在你再次排序之前不必要地对结果进行排序。

然而，你应该做的是确保你的视图包含你以后将用于排序的列。

### 表值函数

视图是一个强大的工具，但它有一个缺点：你不能更改视图中使用的任何值。例如，`aupricelist` 视图中的税率是硬编码的 10%。一种更灵活的视图类型将允许你输入自己的税率。这样的视图将被称为 **参数化视图**。

SQL 通常不支持参数化视图。一些数据库管理系统支持生成虚拟表的函数，称为 **表值函数**，或者如果你赶时间可以叫它 TVF。这将产生大致相同的结果。

在我们常用的数据库管理系统中，只有 PostgreSQL 和 Microsoft SQL Server 支持创建 TVF 的直接方法。我们将在接下来的讨论中探讨这两种。

大多数数据库管理系统允许你创建自定义函数。一个值得注意的例外是 SQLite，不过它允许你在外部创建函数并将其挂钩进来。

一次生成单个值的函数称为 **标量** 函数。像 `lower()` 和 `length()` 这样的内置函数就是标量函数。

在创建函数时，从某种意义上说，存在一个契约。函数定义包括期望的输入数据以及将返回的数据类型。如果输入数据不符合要求，那么就不要期待有结果。

TVF 的工作方式相同：你定义期望的输入，并承诺返回一个结果表。在这里，我们将创建一个更通用的价格表，它允许你告知税率，而不是硬编码。

要使用 TVF，你像使用任何虚拟表一样使用它：

```sql
SELECT *
FROM pricelist(15);
```

这里，TVF 被称为 `pricelist()`，输入参数是 `15`，意思是 15%。代码应负责将其转换为 `0.15`：

| **id** | **Title** | **pub** | **author** | **price** | **tax** | **inc** |
| --- | --- | --- | --- | --- | --- | --- |
| 2078 | The Duel | 1811 | Heinrich von Kleis ... | 12.50 | 1.88 | 14.38 |
| 503 | Uncle Silas | 1864 | J. Sheridan Le Fan ... | 17.00 | 2.55 | 19.55 |
| 2007 | North and South | 1854 | Elizabeth Gaskell | 17.50 | 2.63 | 20.13 |
| 702 | Jane Eyre | 1847 | Charlotte Brontë | 17.50 | 2.63 | 20.13 |
| 1530 | Robin Hood, The Pr ... | 1862 | Alexandre Dumas | 12.50 | 1.88 | 14.38 |
| 1759 | La Curée | 1872 | Émile Zola | 16.00 | 2.40 | 18.40 |
| ~ 1070 rows ~ |

我们接下来将为 PostgreSQL 和 MSSQL 实现这个。

#### PostgreSQL 中的 TVF

PostgreSQL 中 TVF 的大纲如下：

```sql
CREATE FUNCTION pricelist(...)
RETURNS TABLE (...)
LANGUAGE plpgsql AS
$$
BEGIN
RETURN QUERY
SELECT ...
END
$$;
```

在这个大纲中：

*   函数名 `pricelist` 包含了输入参数的名称和类型。
*   该函数将返回一个包含列名和类型的 `TABLE` 结构。
*   编程语言是 `plpgsql`，这是 PostgreSQL 的标准编程语言。
*   实际的代码包含在一个大的字符串中。因为代码中可能还有其他字符串，两端的 `$$` 充当了一个替代的分隔符。
*   然后代码被放置在 `BEGIN` 和 `END` 之间；在这个例子中，它将返回一个 `SELECT` 查询的结果。

填充细节后，我们可以这样写：

```sql
DROP FUNCTION IF EXISTS pricelist(taxrate decimal(4,2));
CREATE FUNCTION pricelist(taxrate decimal(4,2))
RETURNS TABLE (
id int, title varchar, published int, author text,
price decimal(5,2), tax decimal(4,2), inc decimal(5,2)
)
LANGUAGE plpgsql AS $$
BEGIN
RETURN QUERY
SELECT
b.id, b.title, b.published,
coalesce(a.givenname||' ','') || coalesce(othernames||' ','')
|| a.familyname AS author,
b.price, b.price*taxrate/100 AS tax,
b.price*(1+taxrate/100) AS inc
FROM books as b LEFT JOIN authors a ON b.authorid=a.id
WHERE b.price IS NOT NULL;
END; $$;
```

输出表是最繁琐的部分。在其中，我们必须列出我们期望生成的所有列名和类型。

至于计算，我们采取了一种用户友好的方式，允许税率类似于我们在现实生活中可能使用的百分比。我们不能使用 `%`，尤其是因为它有其他含义，但除此之外，我们可以使用这个值。不过，我们需要将其除以 100 以得到其真实值。

#### Microsoft SQL 中的 TVF

在 Microsoft SQL 中创建 TVF 要比在 PostgreSQL 中简单得多。Microsoft SQL 中 TVF 的大纲如下：

```sql
GO
CREATE FUNCTION pricelist(...) RETURNS TABLE AS
RETURN SELECT ...
GO
```

MSSQL 中有两种类型的 TVF。有一种更复杂的类型，但前面提到的简单类型与创建视图非常相似。

在这个大纲中：

*   函数名 `pricelist` 包含了输入参数的名称和类型。
*   该函数将返回一个 `TABLE` 结构。
*   在简单的 TVF 中，只有一个 `SELECT` 语句，该语句会立即作为结果返回。
*   实际的代码与视图的代码几乎相同，只是它会包含来自输入参数的值。

填充细节后，我们可以这样写：

```sql
DROP FUNCTION IF EXISTS pricelist;
GO
CREATE FUNCTION pricelist(@taxrate decimal(4,2))
RETURNS TABLE AS
RETURN SELECT
b.id, b.title, b.published,
coalesce(a.givenname+' ','')
+ coalesce(othernames+' ','')
+ a.familyname AS author,
b.price, b.price*@taxrate/100 AS tax,
b.price*(1+@taxrate/100) AS inc
FROM books as b JOIN authors a ON b.authorid=a.id
WHERE b.price IS NOT NULL;
GO
```

输入参数名为 `@taxrate`。实际上，它的真名是 `taxrate`，但 MSSQL 使用 `@` 字符作为所有变量的前缀。

与 PostgreSQL 版本一样，我们采取了一种用户友好的方式，允许税率类似于我们在现实生活中可能使用的百分比。我们不能使用 `%`，尤其是因为它有其他含义，但除此之外，我们可以使用这个值。不过，我们需要将其除以 100 以得到其真实值。

### 你能用视图做什么？

既然视图是一个虚拟表，那么可以用视图做哪些类型的事情呢？

#### 便捷性

视图最直接的用途是作为一种便捷的方式来封装一个有用的 `SELECT` 查询。例如：

```sql
SELECT * FROM customerdetails;
SELECT * FROM aupricelist;
```

前面的两个视图都包含了连接，其中一个还包含了多项计算。当你需要时，使用已保存的视图要方便得多。

#### 视图的多种用途

## 作为接口

视图的第二个用途是为现有数据提供一个一致的接口。

例如，当我们通过引用另一个表并删除几个列来重构 `customers` 表时，我们面临着使依赖于旧结构的任何其他查询失效的风险。通过创建 `customerdetails` 视图，你有了一个新的虚拟表，它可以像旧表一样被读取。

在你重命名或重新组织表和列的过程中，这也很方便。假设，例如，你正在开发 `customers` 表的新版本，包含以下一些列：

| customerid | firstname | lastname | height_in_inches | au_phone | ... |
| --- | --- | --- | --- | --- | --- |
| ... | ... | ... | ... | ... | ... |

你可以用类似下面的语句来准备它：

```
/* Notes
========================================================
MSSQL:              Use + for concatenation
Oracle, SQLite: Use substr(phone,2) instead of right()
======================================================= */
--  CREATE VIEW newcustomers AS
SELECT
id AS customerid,
givenname AS firstname, familyname AS lastname,
cast(height/2.54 as decimal(3,1))
AS height_in_inches,
'+61' || right(phone,9) AS au_phone
--  etc
FROM customers;
```

这将给你类似下面的结果：

| customerid | firstname | lastname | height_in_inches | au_phone |
| --- | --- | --- | --- | --- |
| 42 | May | Knott | 66.3 | +61255509371 |
| 459 | Rick | Shaw | 67.3 | +61370101040 |
| 597 | Ike | Andy | 60.2 | [NULL] |
| 186 | Pat | Downe | 69.3 | +61870105900 |
| 352 | Basil | Isk | 61.6 | +61255502503 |
| 576 | Pearl | Divers | 69.4 | +61370107821 |
| ~ 303 rows ~ |

（`CREATE VIEW` 子句被注释掉了，因为我们并不真的打算执行它。）

如果你正在为外部应用程序准备数据，这种方法也会很有用。

## 与外部应用程序协作

虽然我们一直在前端客户端上花费时间，但这并非数据的必然归宿。通常，我们会在外部应用程序（如报表软件）中使用数据。

最有意义的是在将数据用于外部应用程序之前，尽可能多地进行预处理。特别是，连接表或运行子查询应该在数据库端完成，而不是之后。

请注意，有些事情数据库无法轻易完成。例如，可能存在数据格式要求或某些数据库服务器不支持的函数。为此，你需要提取原始数据并在外部应用程序中处理。

与外部应用程序协作的例子包括：

-   与文字处理器进行邮件合并
-   在电子表格中使用数据透视表
-   与报表软件配合使用

这类软件通常在数据操作方面能力有限，因此尽可能多地进行预处理是明智的。从外部软件的角度来看，你的视图将被视为单个表（尽管通常它们仍会表明自己实际上是视图）。

## 缓存数据与临时表

一些数据库管理系统提供物化视图。这是一种可以缓存数据的视图类型。其结果是，重复从视图读取数据（不一定）需要重新处理。

当你使用视图时，你可能会给数据库管理系统带来额外的负担。如果视图需要进行一些复杂的处理和连接，那么每次查看视图时，你都会发现自己在重复进行额外的工作。

一些数据库管理系统通过保留一份结果的副本来“作弊”，只要下次访问的时间不太晚，并且相关表没有发生变化。这个副本被称为**缓存**，是否这样做由数据库管理系统决定。

一些数据库管理系统支持物化视图，使这一过程正式化。物化视图在数据库中分配了存储空间来维护结果的副本。这在处理上成本更低，因为数据库管理系统不需要如此频繁地处理相同的数据，但在存储上成本更高。通常，你可能会发现额外的存储比处理能力更便宜。

物化视图并未得到广泛支持，有时用处有限。然而，你可以通过临时表走得很远。

原则上，所有 SQL 表都是临时的，因为总是可以删除一个表——在 SQL 中，如同在生活中，没有什么是真正永恒的。但是，临时表是注定短暂存在的，并且会在你关闭会话时自我销毁。

你可以像创建真实表一样创建临时表，但使用 `TEMPORARY` 前缀：

```
--  PostgreSQL, MariaDB/MySQL, SQLite
CREATE TEMPORARY TABLE somebooks (
id INT PRIMARY KEY,
title VARCHAR(255),
author VARCHAR(255),
price DECIMAL(4,2)
);
--  Oracle
CREATE GLOBAL TEMPORARY TABLE somebooks (
id INT PRIMARY KEY,
title VARCHAR(255),
author VARCHAR(255),
price DECIMAL(4,2)
);
--  MSSQL
CREATE #somebooks (
id INT PRIMARY KEY,
title VARCHAR(255),
author VARCHAR(255),
price DECIMAL(4,2)
);
```

**注意：**

-   Oracle 区分 `GLOBAL` 和 `PRIVATE` 临时表，需要这两个关键字之一。私有临时表还需要特殊的名称。
-   PostgreSQL 允许你使用 `GLOBAL` 和 `LOCAL` 关键字达到相同目的，但会忽略它们；建议省略。
-   MSSQL 使用井号（`#`）表示私有临时表，双井号（`##`）表示全局临时表。

所谓“全局”，是指数据库的其他用户可以访问该临时表。私有临时表，顾名思义，是会话私有的。

如果你非常着急，PostgreSQL 和 SQLite 允许你用 `TEMP` 代替 `TEMPORARY` 来节省时间。你读这段话花的时间可能比这还多。

这个例子中的临时表有一个简单的整数主键。如果你打算在后续添加更多数据，你也可以使用自动递增的主键。

一旦创建了临时表，你就可以使用 `SELECT` 语句将数据复制进去。例如：

```
INSERT INTO somebooks(id,title,author,price)
SELECT id,title,author,price
FROM aupricelist
WHERE price IS NOT NULL;
```

`INSERT ... SELECT ...` 语句将数据复制到一个已存在的表中，无论是临时表还是永久表。

你可以使用以下语句在一个语句中创建新表并填充它：

```
--  PostgreSQL, MariaDB/MySQL, Oracle
CREATE TABLE otherbooks AS
SELECT id,title,author,price
FROM aupricelist
WHERE price IS NULL
;
--  PostgreSQL, SQLite
SELECT id,title,author,price
INTO TEMPORARY otherbooks
FROM aupricelist
WHERE price IS NULL;
--  MSSQL
SELECT id,title,author,price
INTO #otherbooks
FROM aupricelist
WHERE price IS NULL;
```

如你所见，该语句有两种形式；PostgreSQL 两者都支持。

请注意，无论哪种形式，都要求你拥有创建临时表或永久表的权限。

请记住，数据是副本，因此除非你更新它，否则它会过时。

### 为何要使用临时表？

你为什么需要临时表？在我们的示例数据库中，没有任何内容可以被视为真正意义上的“重负载”。然而，在现实世界中，你可能会处理一个涉及海量行、复杂连接、筛选、计算和排序的查询。这最终可能耗费大量的时间和精力，尤其是当你需要不断重新生成数据时。

你选择使用临时表而非视图的原因包括：

*   保存先前生成的结果比重新生成它们更高效。这被称为 `缓存` 结果。
*   有时，你*希望*数据不是最新的，例如当你需要使用当天早些时候数据的 `快照` 时。

如果你需要在未来某个时间点使用该快照，临时表可能就显得“寿命太短”了。我们下面要讨论的所有内容，同样适用于专门创建的永久表。

数据库绝不应保留数据的多个副本。然而，有时你确实需要一个临时表来进行进一步处理、试验，或作为数据迁移过程中的过渡。

#### 计算列

现代 SQL 允许你向表中添加一个原则上不应直接存储在表中的列。`计算`列，或称计算列，是一个基于某些计算值的额外列。仔细想想，这正是你通常会在视图中做的事情。

不妨将计算列理解为在表中嵌入了一个微型视图。如果你经常使用同一种计算，但又不想承担视图的开销，它就特别方便。如果你可以选择缓存结果，它也很有用。

计算列是一个只读的虚拟列。你无法向其中写入任何内容，而且，如果它保存了任何数据，那也只是为了节省日后重新计算的开销而缓存的值。例如，你可以为了方便而存储客户的全名。

你可以在创建表时创建计算列，也可以在事后将其添加到表中。

例如，假设我们想为 `ordered` 日期时间列添加一个缩短形式，只包含日期部分。这对于按天进行汇总非常方便。

你可以按如下方式添加新列：

```sql
--  PostgreSQL >= 12
ALTER TABLE sales
ADD COLUMN ordered_date date
GENERATED ALWAYS AS (cast(ordered as date)) STORED;
--  MSSQL
ALTER TABLE sales
ADD ordered_date AS (cast(ordered as date)) PERSISTED;
--  MariaDB / MySQL
ALTER TABLE sales
ADD ordered_date date
GENERATED ALWAYS AS (cast(ordered as date)) STORED;
--  SQLite>=3.31.0
ALTER TABLE sales
ADD ordered_date date
GENERATED ALWAYS AS (cast(ordered as date)) VIRTUAL;
--  Oracle (STORED)
ALTER TABLE sales
ADD ordered_date date
GENERATED ALWAYS AS (trunc(ordered));
```

如你所见，大多数数据库管理系统都使用标准的 `GENERATED ALWAYS` 语法。然而，MSSQL 使用了其自身更简单的语法，它不指定数据类型，而是从计算中推断出来。

你还会注意到不同类型的计算列：

*   `VIRTUAL` 列不被存储，每次都会重新计算。这是 MSSQL 的默认类型。
*   `STORED` 列保存结果的副本，只有在底层值发生变化时才会重新计算。MSSQL 称之为 `PERSISTED`。在 Oracle 中，这是默认类型。SQLite 也支持此类型，但仅在你以那种方式创建表时；如果后续添加列，则只能是 `VIRTUAL` 类型。

现在你可以获取包含虚拟列的数据了：

```sql
SELECT * FROM sales;
```

这将返回：

| **id** | **...** | **Ordered** | **shipped** | **ordered_date** |
| --- | --- | --- | --- | --- |
| 39 | ... | 2022-05-15 21:12:07.988741 | 2022-05-23 | 2022-05-15 |
| 40 | ... | 2022-05-16 03:03:16.065969 | 2022-05-24 | 2022-05-16 |
| 42 | ... | 2022-05-16 10:09:13.674823 | 2022-05-22 | 2022-05-16 |
| 43 | ... | 2022-05-16 15:02:43.285565 | [NULL] | 2022-05-16 |
| 45 | ... | 2022-05-16 16:48:14.674202 | 2022-05-28 | 2022-05-16 |
| 518 | ... | [NULL] | [NULL] | [NULL] |
| ~ 5549 行 ~ | | | | |

如果可选的话，`STORED` 或其等效类型是更好的选择。它会占用稍多的空间，但能为后续处理节省开销。

