# 2. 使用表设计

SQL 数据库是——或者至少应该是——建立在一些相当强大的原则之上的。尽管这些原则在现实生活中有时会被放宽，但如果尽可能遵循它们，数据库将更可靠、更高效。

在本章中，我们将查看现有数据库的部分内容，并探讨如何使用其中一些原则对其进行改进。我们将探讨：

*   对规范化表的基本理解——即按照核心原则构建或重建的表

*   修改表以使其更严格地遵循这些原则

*   通过添加额外的检查和约束来提高数据库的可靠性和完整性

*   通过添加索引以帮助更高效地查找数据来提升数据库性能

当然，我们无法让表变得完美：这需要很长时间以及对数据库的丰富经验。对于你的数据库，你甚至可能无法做到这一点。然而，我们将能够更好地理解是什么让数据库工作得更好。

## 理解规范化表

首先要考虑的事情之一是个别表的设计和构建方式。目标是确保每条数据都有一个独特且可识别的位置，并且数据处于其最简单的形式。

数学家对任何处于最纯粹形式的事物有一个术语：他们称之为 `normal`（规范化的）。与许多数学术语一样，它的含义可能并非字面所示：它并不意味着“常见”，而是指“确定的”。

开发数据库表通常始于对要存储数据类型的大致想法，然后经历一个 **规范化** 过程：检查表的特性并进行更改，以更好地符合规范化表的要求。

规范化有不同的规则和级别。通常，一个规范化的数据库满足以下要求：

*   数据是原子性的。

    这意味着数据被分解成其最小的实用单元。

*   行是无序的。

    当然，你总是会以某种顺序看到行，但行的顺序并不重要。

*   行是唯一的。

    不应有真正的重复行。巧合情况则另当别论。

*   行是独立的。

    一行中出现的任何内容都不应影响同一表中另一行的内容。

*   列彼此独立。

    更专业地说，列仅依赖于主键。

*   列是单一类型的。

    你不能在单个列中混合数据类型。严格来说，你不应该混合 `domains`（域，即可接受值的集合），但 SQL 很难检查这一点。

*   列名是唯一的。

*   列是无序的。

    同样，列会以某种顺序出现——默认情况下，按表设计的顺序。然而，列的顺序并不重要。

规范化要解决的一个设计问题是处理多值。如果数据要是原子性的，列要是独立的，那么如何管理多个值呢？例如，如何处理有多个销售项目的销售或有多个流派的书籍？

解决方案是将这些值放入一个单独的表中，每行一个值，并让该表引用回第一个表。我们将在下一章探讨表之间的关系，特别是在处理多值时。基本思路是，一个附加表将在多行中保存多个值；然后它将包含一个 `foreign key`（外键），即对第一个表中主键的引用。

在本章中，我们将研究其中一些原则，并看看我们的数据库与之相比如何。当我们确实发现不足时，我们将对表本身进行一些修改，甚至添加几张表，使数据库更符合这些理念。我们还将探讨确保数据更可靠，并在某种程度上更高效的方法。

我们将从解决相互依赖列的问题开始，这意味着要修改一些表。这也意味着要添加额外的表。为了使这些更改不那么麻烦，我们将探讨创建视图来包含这些额外表。

我们还将探讨如何通过添加额外的约束——即数据规则——来首先检查进入表的数据，从而朝着构建更可靠的数据库努力。

最后，我们将探讨添加索引以提高数据库性能。

关于多值的问题，我们将在下一章中看到更多内容，该章将讨论表之间如何相互关联。

没有数据库是完美的，我们的目标也不是让这个变得完美：我们将留下很多未完成的工作。而且，这很可能也不是你的工作。但是，至少我们将能更好地理解是什么造就了一个好的数据库。


### 列应当独立

表格设计的基本原则之一是列之间应相互独立。也就是说，更改某一列不应必然影响另一列。

然而，如果你查看客户地址：

```sql
SELECT
id, givenname, familyname,
street, town, state, postcode
FROM customers;
```

你会发现一些地址列之间确实存在关系。

| `Id` | `...` | `street` | `town` | `state` | `postcode` |
| --- | --- | --- | --- | --- | --- |
| 85 | z | 1313 Webfoot Walk | Kingston | ACT | 2604 |
| 355 | ... | 345 Stonecave Road | Kingston | ACT | 2604 |
| 147 | ... | Apartment 5A, 129 West 81st Street | Kingston | ACT | 2604 |
| 112 | ... | 890 Fifth Avenue | Kingston | ACT | 2604 |
| 489 | ... | Apartment 42, 2630 Hegal Place | Gordon | ACT | 2906 |
| 592 | ... | 0001 Cemetery Lane | Rosebery | NSW | 1445 |
| ~ 303 行 ~ |

例如，如果你将地址从一个城镇更改为另一个，你很可能也需要更改 `postcode`，甚至可能还要改 `state`。除此之外，住在同一个城镇的人可能也有相同的 `postcode`；他们肯定在同一个 `state`。

这就带来了一个维护问题：

*   更改地址需要为单次变更修改*三个*列。
*   可能出现错误，只修改了部分数据；这会导致数据不一致，使得数据失效。

正确的解决方案是将此数据移到另一个表中。

## 添加城镇表

有一个名为 `towns.sql` 的 SQL 文件。它将创建并填充一个 `towns` 表。

该表结构如下：

```sql
CREATE TABLE towns (
id INT PRIMARY KEY, --  自动编号
name VARCHAR(...),
state VARCHAR(...),
postcode CHAR(4),
UNIQUE(name,state,postcode)
);
```

该表还包括一个自动编号的 `id` 列，它是**主键**。主键是唯一标识一行的列。在这里，它是一个任意数字。具体细节将取决于你使用的 DBMS。

虽然 `name`、`state` 和 `postcode` 可能存在重复，但它们的*组合*将是唯一的。

前面的 `UNIQUE` 子句也会创建一个索引，这将使搜索表的速度更快。你稍后会了解更多关于索引的内容。

你现在可以运行此脚本来创建并填充 `towns` 表。

根据你的 DBMS，你可能需要确保将此表安装到正确的数据库中。

## 添加到城镇的外键

下一步是在 `customers` 表中添加一列以引用 `towns` 表中的城镇。这通过 `ALTER` 语句完成：

```sql
ALTER TABLE customers
ADD townid INT
CONSTRAINT fk_customers_town REFERENCES towns(id);
```

请注意，`townid` 列必须与 `towns` 表中的 `id` 列的数据类型匹配，在本例中为整数。

你会注意到它实际上没有使用 `FOREIGN KEY` 这个术语。是关键字 `REFERENCES` 使其成为外键：这里它引用了 `towns` 表中的一个 `id`。

你还会注意到使用 `CONSTRAINT fk_customers_town` 命名外键。每个约束实际上都有一个名称，但如果你愿意让 DBMS 为你生成一个名称，则不必自己命名。如果是这样，你可以使用更简短的形式：

```sql
ALTER TABLE customers
ADD townid INT REFERENCES towns(id);
```

如果你已经有了该列，可以使用以下语句追溯添加外键约束：

```sql
ALTER TABLE customers
ADD CONSTRAINT fk_customers_town FOREIGN KEY(townid)
REFERENCES towns(id);
```

默认情况下，当你创建新列时，它将填充为 `NULL`。你本可以添加一个默认值，但在这种情况下是毫无意义的，因为每个人都住在不同的地方；在某些情况下，我们根本没有客户的地址。

## 更新客户表

现在你有了外键列，你需要用对相应城镇的引用来填充它。

如果你想查看这些外键应该是什么，可以使用子查询：

```sql
SELECT
id, givenname, familyname,
town, state, postcode,    --  现有数据
(SELECT id FROM towns AS t WHERE    --  新数据
t.name=customers.town
AND t.postcode=customers.postcode
AND t.state=customers.state
) AS reference
FROM customers;
```

其中一些结果当然是 `NULL`，因为有些客户没有记录的地址。

| `id` | `givenname` | `familyname` | `town` | `state` | `postcode` | `reference` |
| --- | --- | --- | --- | --- | --- | --- |
| 85 | Corey | Ander | Kingston | ACT | 2604 | 35 |
| 355 | Joe | Kerr | Kingston | ACT | 2604 | 35 |
| 147 | Aiden | Abet | Kingston | ACT | 2604 | 35 |
| 112 | Jerry | Cann | Kingston | ACT | 2604 | 35 |
| 489 | Justin | Case | Gordon | ACT | 2906 | 135 |
| 592 | Paddy | Wagon | Rosebery | NSW | 1445 | 386 |
| ~ 303 行 ~ |

子查询是嵌套在查询内部的查询。在这种情况下，它是一种从另一个表中查找内容的简单方法。

此子查询是一个*关联*子查询：它针对主查询中的每一行运行，使用主查询中的值与子查询进行比较。这通常是一种开销较大的查询类型，但我们不会经常使用它。它对下一步也很有用。

你稍后将了解更多关于子查询的内容。

请注意，我们在子查询中为 `towns` 表起了别名；这是为了使代码更易于阅读和编写。你也可以为 `customers` 表起别名，但这在下一步中并非所有 DBMS 都适用。

我们不仅仅想查看引用：我们想将引用复制到 `customers` 表中。你可以使用 `UPDATE` 语句来完成：

```sql
UPDATE customers
SET townid=(
SELECT id FROM towns AS t
WHERE t.name=customers.town
AND t.postcode=customers.postcode
AND t.state=customers.state
);
```

`UPDATE` 语句用于更改现有表中的值。你可以将值设置为常量值、计算值，或者如本例所示，来自另一个表的值。

这里，使用相同的子查询来获取将被复制到 `townid` 列中的 `id`。

一些 DBMS 允许你为 `customers` 表起别名，这会使 `UPDATE` 语句稍微简单一些。

关联子查询可能开销较大，通常如果可能，最好使用连接（join）。我们本可以在 `SELECT` 语句中使用连接，但并非所有 DBMS 与 `UPDATE` 语句配合得那么好。这里，子查询直观且效果良好，而且由于你只运行一次，开销不算太大。

## 移除旧的地址列

很快，我们将移除旧的地址列，但这将在以后造成不便，因为城镇数据现在在另一个单独的表中。这意味着如果不连接表，我们就无法获取包含完整地址的客户详细信息。为了管理这种不便，创建一个视图来组合客户数据和城镇数据将很有用。



### 创建客户详情视图

视图是一个已保存的查询，我们可以将其用作虚拟表。在此视图中，我们将简单地添加 `customers` 表中的所有列，除了城镇数据。对于城镇数据，我们需要连接到 `towns` 表。

首先，我们用一个简单的 `SELECT` 语句来测试：

```sql
SELECT
c.id, c.email, c.familyname, c.givenname,
c.street,
--  原始值
c.town, c.state, c.postcode,
c.townid,
--  来自 towns 表
t.name AS town, t.state, t.postcode,
c.dob, c.phone, c.spam, c.height
FROM customers AS c LEFT JOIN towns AS t ON c.townid=t.id;
```

如果你是在 Oracle 中操作，请记住不能为表别名使用 `AS`：

```sql
SELECT
...
FROM customers c LEFT JOIN towns t ON c.townid=t.id;
```

请注意：

*   我们使用 `LEFT JOIN` 来包含没有地址的客户。
*   为了方便，我们为 `customers` 和 `towns` 表设置了别名。
*   `towns` 表有一个 `name` 列，而不是 `town` 列。然而，在查询上下文中，将其别名为 `town` 是合理的。
*   我们还包含了 `c.townid` 列，虽然它是冗余的，但可能使维护更容易。

一旦你确认 `SELECT` 语句能完成工作，就可以创建视图了。当然，你应该省略旧的城镇数据，因为整个目的是使用来自连接的数据：

```sql
CREATE VIEW customerdetails AS
SELECT
c.id, c.email, c.familyname, c.givenname,
c.street,
--  省略 c.town, c.state, c.postcode
c.townid, t.name AS town, t.state, t.postcode,
c.dob, c.phone, c.spam, c.height
FROM customers AS c LEFT JOIN towns AS t ON c.townid=t.id;
```

在 Microsoft SQL 中，你需要将 `CREATE VIEW` 语句包裹在一对 `GO` 关键字之间：

```sql
--  MSSQL:
GO
CREATE VIEW customerdetails AS
SELECT
c.id, c.email, c.familyname, c.givenname,
c.street,
c.townid, t.name as town, t.state, t.postcode,
c.dob, c.phone, c.spam, c.height
FROM customers AS c LEFT JOIN towns AS t ON c.townid=t.id;
GO
```

你稍后会了解更多关于视图的知识。

### 删除地址列

要删除旧的地址列，你应该能够运行以下语句：

```sql
--  PostgreSQL, MySQL / MariaDB
ALTER TABLE customers
DROP COLUMN town, DROP COLUMN state, DROP COLUMN postcode;
--  Oracle: 不是 DROP COLUMN
ALTER TABLE customers DROP (town, state, postcode);
--  SQLite: 你需要一次删除一个列
ALTER TABLE customers DROP COLUMN town;
ALTER TABLE customers DROP COLUMN state;
ALTER TABLE customers DROP COLUMN postcode;
--  MSSQL: （目前）不工作
ALTER TABLE customers
DROP COLUMN town, state, postcode;
```

这里，我们使用 `DROP COLUMN` 来移除一个或多个列，当然，以及它们的所有数据，所以你需要确保不再需要它。正如你之前看到的，不同数据库管理系统之间的语法存在一些差异。

在 Microsoft SQL 中，你会收到一个错误，提示无法删除 `postcode` 列，因为存在一个约束。约束是用于有效值的附加规则。

在这种情况下，有一个名为 `ck_customers_postcode` 的约束，要求邮政编码仅由四位数字组成。你现在不再需要这个约束了，特别是因为你正要删除该列。

要移除约束，请运行：

```sql
--  MSSQL
ALTER TABLE customers
DROP CONSTRAINT ck_customers_postcode;
```

成功移除约束后，你现在可以删除列了：

```sql
ALTER TABLE customers DROP COLUMN town, state, postcode;
```

这样你就移除了多余的地址列。

请记住，如果你删除了错误的列，要恢复它是非常棘手或不可能的。

## 更改城镇

当然，整个练习的重点是你现在应该能够通过一个单一的更改来迁移到另一个城镇。我们将用客户 `42` 来尝试这个。

首先，找到客户 `42` 的地址，特别是注意 `townid`：

```sql
SELECT * FROM customerdetails WHERE id=42;
```

你会得到类似这样的结果：

| `id` | `...` | `townid` | `town` | `State` | `postcode` | `...` |
| --- | --- | --- | --- | --- | --- | --- |
| 42 | ... | 846 | Kings Park | NSW | 2148 | ... |

请注意，我们是从 `customerdetails` 视图中读取数据，因为城镇数据已不在 `customers` 表中，尽管 `townid` 还在。

现在，将客户的 `townid` 更改为任何你喜欢的值（只要不超过 `towns` 表中的最高 id）：

```sql
UPDATE customers SET townid=12345 WHERE id=42;
```

如果你现在检查同一个客户：

```sql
SELECT * FROM customerdetails WHERE id=42;
```

你会得到类似这样的结果：

| `id` | `...` | `townid` | `town` | `State` | `postcode` | `...` |
| --- | --- | --- | --- | --- | --- | --- |
| 42 | ... | 12345 | Swan Marsh | VIC | 3249 | ... |

如果你愿意，可以将其设置回原始值。

在这里，我们设置了 `customers` 表中的 `townid`，这是它所属的位置。

一些数据库管理系统允许你采取稍微间接的方法来更改此值，并通过视图进行更改：

```sql
--  不是 PostgreSQL 或 SQLite：
UPDATE customerdetails SET townid = ... WHERE id=42;
```

如你所见，这不包括 PostgreSQL 或 SQLite。

当然，你并不能真正更新视图，因为它实际上只是一个 `SELECT` 语句，不包含任何数据。相反，数据库管理系统会尝试找出特定列属于哪个表，并将更改传递给该表。有时它无法做到这一点，例如当你尝试更新一个计算列时。在这种情况下，更新将失败，你必须直接更新表。


### 提升数据库完整性

## 添加国家信息

为了保证完整性，你可能需要添加国家的参考信息。你可以简单地增加一个包含国家名称的列，但许多国家的名称存在变体，你不会希望同一张表中出现三个不同版本的国家名称。

更好的做法是创建一张独立的国家表，并包含一个指向该表的外键。你可以将这个引用添加到 `customers` 表中，但将其添加到 `towns` 表可能更有意义，因为城镇是位于某个国家的。

你会注意到 `countries` 表包含的内容远超当前所需。你可能不需要国家的人口或面积等信息。然而，在未来某个时候，你可能需要国家的货币、时区或电话区号，所以现在包含这些信息并无坏处。这不是一张很大的表，你可以忽略不需要的部分，但也可以随时删除确实不需要的列。

1.  另有一个名为 `countries.sql` 的 SQL 文件用于创建另一张表。它包含许多细节，但最重要的两个细节是：

    ```
    CREATE TABLE countries (
    id CHAR(2) PRIMARY KEY,
    name VARCHAR(...),
    --  等等
    );
    ```

    注意主键是一个双字符字符串。每个国家都有一个预定义的两字符代码，通常基于该国名称（英文或本国语言）。将其用作主键是合理的，而不是自己编造一个。这就是一个 `自然键` 的例子：一个基于实际数据而非任意代码的主键。

    运行脚本来安装该表。

2.  向 `towns` 表中添加一个 `countryid` 列，类似于你向 `customers` 表添加 `townid` 的方式。记住，数据类型必须与前面的主键匹配：

    ```
    --  PostgreSQL, Oracle, MSSQL, SQLite
    ALTER TABLE towns
    ADD countryid CHAR(2)
    CONSTRAINT fk_town_country REFERENCES countries(id);
    --  MySQL / MariaDB
    ALTER TABLE towns
    ADD countryid CHAR(2) REFERENCES countries(id);
    ```

3.  更新 `towns` 表，将澳大利亚或你选择的其他国家的 `countryid` 值设置为 `'au'`。这比通过子查询设置要简单得多：

    ```
    UPDATE towns SET countryid='au';
    ```

4.  你将需要修改你的视图。首先，删除旧版本：

    ```
    --  非 Oracle 数据库：
    DROP VIEW IF EXISTS customerdetails;
    --  Oracle 数据库：
    DROP VIEW customerdetails;
    ```

5.  接下来，你需要使用国家名称重新创建它：

    ```
    --  非 Oracle 数据库
    CREATE VIEW customerdetails AS
    SELECT
    ...
    c.townid, t.name AS town, t.state, t.postcode,
    n.name AS country
    ...
    FROM
    customers AS c
    LEFT JOIN towns AS t ON c.townid=t.id
    LEFT JOIN countries AS n ON t.countryid=n.id;
    --  Oracle 数据库
    CREATE VIEW customerdetails AS
    SELECT
    ...
    c.townid, t.name AS town, t.state, t.postcode,
    n.name AS country
    ...
    FROM
    customers c
    LEFT JOIN towns t ON c.townid=t.id
    LEFT JOIN countries n ON t.countryid=n.id;
    ```

**注意**

*   这包括了与 `countries` 表的一次额外 `JOIN`；为了适应更长的子句，我们将 `JOIN` 拆分到了多行。
*   `countries` 表的别名被设置为 `n`（代表 Nation）；这仅仅是因为我们不能使用 `c`，因为它已被占用。

## 附加说明

你可能注意到我们没有对 `street` 地址列做任何处理。严格来说，它也存在与其他地址部分相同的问题，因此如果我们采取类似处理会更好。

然而，街道地址要复杂得多，而且我们的客户也不算多，所以我们保持了原样。这使得我们的设计虽不完美，但已大为改进。

### 提升数据库完整性

到目前为止，我们一直专注于通过减少列之间的依赖和数据重复，使表更接近真正的范式。这意味着需要添加其他表和外键。

在这里，我们将探讨对数据库完整性的其他改进。`数据库完整性`指的是数据的质量：数据是否有意义？

你需要记住，DBMS（数据库管理系统）实际上并不了解发生了什么，它也不在乎你是否在说实话。然而，DBMS 非常关心数据是否 *有效*。即，数据是否符合各种规则。

理论上，数据属于一个 `域`——一个有效值的集合。然后你应该能够为一个或多个列定义一个域。实际上，这个特性在大多数 DBMS 中并未得到广泛支持。

另一方面，你可以轻松地对列施加 `约束`。约束就是数据规则。

你已经知道一些标准的约束类型：

*   数据类型，如 `INTEGER` 或 `VARCHAR(16)`，限制了可接受数据的类型和范围。
*   `NOT NULL` 约束意味着值不能为 `NULL`；即，它是必填的。
*   `UNIQUE` 约束将不允许出现重复值，如果另一行在该列（或列组合）中已存在相同的值。
*   `REFERENCES` 约束定义了一个外键；外键 *必须* 匹配另一个键中的现有值。

当然，在所有情况下，都不能保证值是真实的——仅仅是有效的。

如果你想更具体地定义什么是有效，还有 `CHECK` 约束。`CHECK` 是一种杂项约束，允许你使用类似于 `WHERE` 子句的表达式来设置自己的规则。有时，这些被称为 `业务规则`。

在本节中，我们将研究数据库的一些弱点，并通过添加一些约束来填补设计中的一些空白。

以下大部分内容将涉及对现有列进行更改。如果你使用的是 SQLite，那么很遗憾，你无法这样做。SQLite 的 `ALTER TABLE` 功能非常有限，无法修改现有列。如果你确实需要进行此类更改，你将不得不经历一个更复杂的过程，即删除一列并创建一个新列。


#### 修复可空列的问题

`saleitems` 表中包含一个名为 `quantity` 的列——这代表你购买该书的数量：

```sql
SELECT * FROM saleitems ORDER BY saleid,id;
```

你将看到类似以下内容：

| **Id** | **saleid** | **bookid** | **quantity** | **price** |
| --- | --- | --- | --- | --- |
| 1 | 1 | 1403 | 1 | 11.5 |
| 2 | 1 | 1861 | 1 | 13.5 |
| 3 | 1 | 643 | [NULL] | 18 |
| 4 | 2 | 187 | 1 | 10 |
| 5 | 2 | 1530 | 1 | 12.5 |
| 6 | 2 | 1412 | 2 | 16 |
| ~ 13964 rows ~ |

然而，由于疏忽，该列允许 `NULL` 值，如果你仔细查看，会发现很多行中存在这种情况。这不合常理：如果你不知道要购买多少本，就无法生成销售明细。
一个合理的推测是：缺失的 `quantity` 表示数量为 `1`。你可以使用 `coalesce()` 函数来实现这个推测：

```sql
SELECT
id, saleid, bookid,
coalesce(quantity,1) AS quantity, price
FROM saleitems
ORDER BY saleid, id;
```

现在，除了 `NULL` 值被替换为 `1` 之外，我们将得到相同的结果：

| **id** | **saleid** | **bookid** | **quantity** | **Price** |
| --- | --- | --- | --- | --- |
| 1 | 1 | 1403 | 1 | 11.5 |
| 2 | 1 | 1861 | 1 | 13.5 |
| 3 | 1 | 643 | 1 | 18 |
| 4 | 2 | 187 | 1 | 10 |
| 5 | 2 | 1530 | 1 | 12.5 |
| 6 | 2 | 1412 | 2 | 16 |
| ~ 13964 rows ~ |

如同使用 `coalesce()` 函数时通常需要做的那样，你必须检查你的假设。`1` *真的*是一个合理的推测吗？在这种情况下，它不太可能代表零本或其他数字，但这确实取决于具体情况。为了本次练习，我们暂且接受这个推测...
我们当然不希望每次都手动处理，因此我们将修正旧数据并防止未来出现 `NULL` 值。
接下来的操作不适用于 SQLite。不过，在此之后会有一节专门说明如何在 SQLite 中进行相同的更改。

##### 替换 NULL 数量值

首先，我们将禁止 `NULL` 值。稍后，我们将为 `quantity` 列添加一个 `NOT NULL` 约束。然而，在清除现有的 `NULL` 值之前，我们无法这样做，因为即使约束是后来添加的，DBMS（数据库管理系统）也绝不会允许违反约束。
假设这没问题，我们可以将 `NULL` 值替换为 `1`：

```sql
UPDATE saleitems
SET quantity=1
WHERE quantity IS NULL;
```

从现在开始，对于现有数据，我们不再需要使用 `coalesce()`，但我们需要防止未来出现 `NULL` 值。

##### 为数量列设置 NOT NULL 约束

接下来是在该列上设置一个 `NOT NULL` 约束：

```sql
--  PostgreSQL
ALTER TABLE saleitems ALTER COLUMN quantity SET NOT NULL;
--  MySQL/MariaDB
ALTER TABLE saleitems MODIFY quantity INT NOT NULL;
--  MSSQL
ALTER TABLE saleitems ALTER COLUMN quantity INT NOT NULL;
--  Oracle
ALTER TABLE saleitems MODIFY quantity NOT NULL;
--  SQLite 中无法直接实现
```

之前，`ALTER TABLE` 语句被用来添加或删除列。你也可以用它来修改现有列。这里，我们用它来添加一个 `NOT NULL` 约束。
正如你之前所见，每个 DBMS 对 `ALTER TABLE` 语句都有其细微的语法差异。

##### 为数量列设置 DEFAULT 默认值

原则上，导致 `NULL` 值出现的原因可能会再次发生，只是现在会引发错误。更好的做法是，我们应提供一个默认值 `1`，以防未来的交易中再次缺失数量：

```sql
--  PostgreSQL
ALTER TABLE saleitems
ALTER COLUMN quantity SET DEFAULT 1;
--  MySQL/MariaDB
ALTER TABLE saleitems
MODIFY quantity INT DEFAULT 1;
--  MSSQL
ALTER TABLE saleitems
ADD DEFAULT 1 FOR quantity;
--  Oracle
ALTER TABLE saleitems
MODIFY quantity DEFAULT 1;
--  SQLite 中无法直接实现
```

`DEFAULT` 值是在你不提供值时使用的值。该列不一定要是 `NOT NULL`，`NOT NULL` 列也不一定要有 `DEFAULT`。然而，在这种情况下，这是一个合理的组合。
再次注意，每个 DBMS 的语法都有其细微差异。

##### 为数量列添加正数 CHECK 约束

在我们微调 `quantity` 列的行为时，你还可以防止另一种可能的错误。原则上，整数可以包含负值，这对于 `quantity` 来说毫无意义。即使零在当前情况下也不合适。
你可以使用 `CHECK` 约束来防止出现超出范围的值：

```sql
CHECK (quantity>0)
```

你也可以使用 `BETWEEN` 表达式来设置一个上限：

```sql
CHECK (quantity BETWEEN 1 AND 5)
```

请记住 `BETWEEN` 是包含边界的。
然而，在施加任意限制（例如最大为 5）时必须谨慎，因为比如说，购买 6 本并非不可能。我们将这个决定留待将来。
要添加 `CHECK` 约束，你再次需要使用 `ALTER TABLE`：

```sql
--  PostgreSQL
ALTER TABLE saleitems
ADD CHECK (quantity>0);
--  MySQL/MariaDB
ALTER TABLE saleitems
MODIFY quantity INT CHECK(quantity>0);
--  MSSQL
ALTER TABLE saleitems
ADD CHECK(quantity>0);
--  Oracle
ALTER TABLE saleitems
MODIFY quantity CHECK(quantity>0);
--  SQLite 中无法直接实现
```

至此，我们已经使 `quantity` 列更加可靠了。

##### 合并更改

在某些 DBMS 中，可以将这些更改合并到一个 `ALTER TABLE` 语句中：

```sql
--  PostgreSQL
ALTER TABLE saleitems
ALTER COLUMN quantity SET NOT NULL,
ALTER COLUMN quantity SET DEFAULT 1,
ADD CHECK (quantity>0);
--  MySQL/MariaDB
ALTER TABLE saleitems MODIFY quantity INT
NOT NULL
DEFAULT 1
CHECK(quantity>0);
--  Oracle
ALTER TABLE saleitems MODIFY quantity
DEFAULT 1
NOT NULL
CHECK(quantity>0);
--  MSSQL 中无法实现
--  SQLite 中无法实现
```

由于你实际上并不经常进行这类更改，即使将步骤分开执行也没有什么损失。

##### 在 SQLite 中进行更改

如你所见，SQLite 在更改表结构方面能力非常有限。通常，SQLite 只能对表进行以下更改：
*   添加列
*   重命名列
*   删除列

然而，只要我们不介意列的顺序不同，这就足以实现我们想要的所有更改。
要完成前面所有的更改：

1.  重命名原始的 `quantity` 列：
    ```sql
    ALTER TABLE saleitems
    RENAME quantity TO oldquantity;
    ```
2.  添加一个具有所需属性的新 `quantity` 列：
    ```sql
    ALTER TABLE saleitems
    ADD quantity INT NOT NULL DEFAULT 1 CHECK(quantity>0);
    ```
3.  将数据从旧列复制到新列：
    ```sql
    UPDATE saleitems
    SET quantity=oldquantity;
    ```
4.  删除旧列：
    ```sql
    ALTER TABLE saleitems
    DROP oldquantity;
    ```

新列将位于表的末尾，与原始列位置不同，但这并不是什么大问题。



#### 其他调整

正如开发过程中常见的情况，让某些东西运行起来并不难，但主要的精力耗费在于让它运行得恰到好处。以下是一些建议，用于提升数据库的完整性和性能。

我们将在下一节讨论索引：它们有助于让数据更易于搜索或排序。

| **表** | **列** | **建议** |
| --- | --- | --- |
| Customers |
|   | height | `CHECK` (`height>0`) – 或 `height BETWEEN 60 and 260` |
|   | dob | `CHECK` (`dob<current_timestamp`) |
|   | registered | `CHECK` (`registered<current_timestamp`) |
| Authors |
|   | names | `INDEX` |
|   | dates | `CHECK` (`born<died`) |
|   | gender | `CHECK` (`gender IN(‘m‘,‘f‘)`) |
|   |   | `CHECK` (`givenname IS NOT NULL OR familyname IS NOT NULL`) |
| Books |
|   | authorid | `INDEX` |
|   | title | `INDEX` |
|   | published | `CHECK` (`published < year(current_timestamp)`) |
|   | price | `CHECK` (`price>=0`) |
| Sales |
|   | total | `CHECK` (`total>=0`) |
|   | ordered | `CHECK` (`ordered<current_timestamp`) |
|   | customerid | `INDEX` |
|   |   | `CHECK` (`shipped>=ordered`) |
| Saleitems |
|   | saleid | `INDEX` |
|   | bookid | `INDEX` |
|   | quantity | `NOT NULL CHECK(quantity>0) DEFAULT 1` |
|   | price | `CHECK(price>=0)` |

你会注意到，有些 `CHECK` 约束并未关联到单一列。有些约束更关注一列如何与另一列相关联。

我们当然不会在这里处理所有这些建议。毕竟，这不是一个真实的工作数据库，而且这很可能也不是你的工作。我们只会再看两个例子。

##### 确保价格非负

如果能够定义一种非负的数据类型，我们应该对许多列使用这种类型。例如，MySQL/MariaDB 有一种称为 `UNSIGNED INT` 的数据类型：由于是无符号的，它将始终为零或正数，这对于一些计数器以及数量来说非常方便。

当然，另一种方法是使用 `CHECK` 约束来限制取值。我们会将此构建到书籍价格中。

你可能会考虑一个合适的最低或最高价格，为此你可以使用 `BETWEEN` 条件。然而，这可能会随时间变化，所以并不总是实用。

我们在这里要做的只是确保价格从不低于零。我们会允许零，以防有些东西我们愿意免费赠送，但价格绝不应低于零。

添加这个约束很简单，但语法在不同的数据库管理系统之间又有所不同：

```
--  PostgreSQL
ALTER TABLE books ADD CHECK (price>=0);
--  MySQL/MariaDB
ALTER TABLE books MODIFY price INT CHECK(price>=0);
--  MSSQL
ALTER TABLE books ADD CHECK(price>=0);
--  Oracle
ALTER TABLE books MODIFY price CHECK(price>=0);
```

再次，在 SQLite 中执行此操作，你可以遵循之前 `saleitems` 表中 `quantity` 列的步骤。

##### 确保作者生年早于卒年

当你有多个包含类似数据的列时，就存在将它们顺序弄反的风险。

一个众所周知的例子是姓名：你按姓氏 `familyname`，然后名字 `givenname` 排序，因此按这种方式存储是有道理的，但你常常会以相反的顺序查看它。不难想象姓名可能会被错误地输入。

`authors` 表有两个日期：`born`（出生日期）和 `died`（死亡日期）。在这种情况下，它们是按时间顺序存储的，但进行确认仍然是好的。

在这里，我们将添加一个 `table constraint`（表约束）——一个应用于表而不是单个列的约束。列约束也可以通过这种方式添加，但有些约束不适用于单个列。

如果我们有机会从头开始重建表，我们会这样做：

```
CREATE TABLE authors (
id int PRIMARY KEY,     --  按 DBMS 自动编号
--  givenname, othernames, familyname,
born DATE,
died DATE,
--  gender, home,
--  表约束:
CONSTRAINT ck_author_dates CHECK(born<died)
);
```

这里，约束作为一个附加属性出现，通常（但不一定）在最后。

鉴于表已经存在并且已有数据，那个机会已经错过了。然而，我们可以事后添加表约束。

首先，当然，你需要检查是否有任何数据可能违反该约束：

```
SELECT * FROM authors WHERE born<died;
```

不应该有任何数据违反。如果有，那你就得自己处理了。你必须自行研究正确的日期应该是什么，或者，如果你实在没办法，可以将它们设置为 `NULL`。

下一步将是添加表约束：

```
ALTER TABLE authors ADD CHECK (born<died);
```

与添加列约束不同，各种数据库管理系统都使用相同的语法——当然，SQLite 除外。在 SQLite 中，没有简单的方法来添加表约束。复杂的方法包括删除并重建整个表（类似于删除并重建列），或者篡改数据库内部结构，这绝对不适合胆小的人。


## 添加索引

SQL 并不定义表应该以何种顺序存储。这使得数据库管理系统（DBMS）可以自行决定以任何它认为最有效的方式来存储表。

问题在于，当搜索特定行时，它可能在任何位置，唯一的查找方法是遍历整个表，并希望这不会花费太长时间。

另一方面，如果表是有序的，查找所需内容就会容易得多。然而，即使它有序，也可能只是按错误的字段排序。

例如，即使 `customers` 表按 `id` 排序，在按 `familyname` 搜索时也毫无帮助。如果它按 `familyname` 排序，在按 `phone` 搜索时同样无济于事。

解决方案是保持表原样，然后通过一个或多个 `index`（索引）来补充表。索引是一个额外的列表，它按搜索顺序排列，并包含一个指向表中匹配行的指针。

例如，`customers` 表有一个针对 `familyname` 的索引。当需要按 `familyname` 搜索时，DBMS 会自动查找索引，找到所需内容，然后返回真实表中获取其余数据。

使用索引有两个代价：

*   每个索引都会在数据库中占用额外的空间。
*   每次在表中添加或更改一行时，每个索引也需要更新。

因此，只有在表设计中特别要求时，你才会在某个列上找到索引。并且，只有在你认为搜索能力的提升值得付出存储和管理成本时，你才会添加索引。

有两种额外的索引是自动包含的：

*   任何 `UNIQUE`（唯一）列都会被索引；防止重复值的最佳方法是维护一个现有值的有序列表。
*   主键总是被索引；根据定义，它是一个唯一标识符，你可能会经常搜索它。

另一种值得考虑的列类型是外键。这是因为它当然会大量参与搜索和排序。

学术界对于对外键建立索引的优劣存在一些讨论。总的来说，这似乎是个好主意，你最好考虑为每个外键添加一个索引。

任何其他列则需要根据情况判断。至少，将来在某个时间点改变主意添加或删除索引并不困难。

一些 DBMS 确实包含按某一列顺序存储表的能力。这被称为 `聚集索引或索引组织表`。在一些 DBMS 中（如 Microsoft SQL），聚集是永久性的（DBMS 确保表始终按该顺序维护）；在另一些中，它是临时性的（DBMS 对表排序一次，但将来你需要再次执行排序）。

在此，我们忽略聚集索引。无论如何，你仍然无法让表同时保持多种顺序，因此无论如何你都需要索引。

### 为 Books 和 Authors 表添加索引

你可能经常搜索的一个列是 `books` 表中的 `title` 列。要添加索引，请使用 `CREATE INDEX`：

```
CREATE INDEX ix_books_title
ON books(title);
```

`ON` 子句标识了你想要列出的表和列。

可以在单个语句中索引多个列，但这不会创建多个独立的索引。相反，你会创建一个基于组合值的索引。例如：

```
CREATE INDEX ix_authors_name
ON authors(familyname, givenname, othernames);
```

这将创建一个包含作者 `familyname`、`givenname` 和 `othernames` 的单一索引。

即使索引是围绕作者姓名的这三个部分构建的，如果你只搜索，例如 `familyname`，它仍然会被使用。然而，这样使用部分索引的前提是，你至少使用了索引的前几个组成部分，这就是为什么列要按此顺序排列。

请注意，在前面两个语句中，索引都被赋予了一个名称。对于名称应该是什么没有硬性规定，但开发者有自己的模式。例如，前面的模式类似于：

```
ix_table_columns
```

这不是一个严格的规则，但它使操作更容易。

为什么索引需要名称呢？大多数时候，你并不真正关心。然而，有两个原因：

*   存储在数据库中的所有内容，包括维护对象，都必须有一个唯一的名称以供内部管理。
*   如果你需要删除一个索引，你需要使用它的名称来标识它。

即使你成功创建了一个匿名索引，DBMS 也会自动分配它自己的名称，而这个名称通常不太美观。

你可能考虑的另一个索引是 `books` 表中的外键 `authorid`。你可以用以下语句添加它：

```
CREATE INDEX ix_books_authors
ON books(authorid);
```

当然，你也可以考虑为 customer details 或其他细节包含索引。


## 创建唯一索引

有些列你可能不希望出现重复值。例如，在你的客户表中，你不应该看到两个客户拥有相同的电子邮件地址，特别是当他们需要使用电子邮件地址登录时。同样，你可能也不希望不同的客户拥有相同的电话号码。

其他列，如姓氏或出生日期，则应该允许重复：重复的值可能仅仅是巧合。

`customers`表已经通过在电子邮件列上包含`UNIQUE`属性来防止重复地址。我们将对电话号码做同样的处理。

然而，在此之前，你需要确保没有现存的重复项。SQL 会拒绝执行任何违反数据库约束的操作，因此，如果存在重复的电话号码，在解决这些重复项之前，你将无法添加`UNIQUE`约束。

要查找重复项，你需要使用聚合分组查询。例如，你可能想查找名字重复的客户：

```sql
SELECT familyname, givenname, count(*) AS number
FROM customers
GROUP BY familyname, givenname;
```

通过按名字分组，你可以统计它们出现的次数。当然，既然你只对那些出现超过一次的记录感兴趣，你可以使用`HAVING`子句来过滤结果：

```sql
SELECT familyname, givenname, count(*) AS number
FROM customers
GROUP BY familyname, givenname
HAVING count(*)>1;
```

你应该会看到一个候选列表：

| familyname | Givenname | number |
| --- | --- | --- |
| Free | Judy | 2 |
| Mate | Annie | 2 |
| Christmas | Mary | 2 |
| Tuckey | Ken | 2 |
| Ander | Corey | 2 |
| Dunnit | Ida | 2 |
| Bearer | Paul | 2 |
| Bell | Terry | 2 |

在前面的查询中，行按`familyname`和`givenname`分组并汇总。`HAVING`子句过滤出那些出现次数超过一次的分组。然后`SELECT`子句输出这些名字及其出现次数。

你并不一定需要在`SELECT`子句中包含`count(*)`，但这有助于使结果更清晰。

当然，如果你发现重复的姓氏，这通常不是问题：许多人和别人同名。但是，如果你发现重复的电话号码，那就可能是个问题：

| **phone** | **number** |
| --- | --- |
| [NULL] | 17 |

```sql
SELECT phone, count(*) AS number
FROM customers
GROUP BY phone
HAVING count(*)>1;
```

在这种情况下，没有重复项。看起来像是重复的实际上是`NULL`值，因为表中有多个`NULL`。它们不计入重复。

如果你确实发现了重复项，那么你需要费一番功夫来判断这些重复是否合理。你甚至可能得出结论，重复的电话号码是可以接受的，那么就不会进行下一步操作。

假设重复是*不可接受*的，为了防止重复，你需要添加一个`UNIQUE INDEX`：

```sql
--  Not MSSQL
CREATE UNIQUE INDEX uq_customers_phone
ON customers(phone);
```

Microsoft SQL 有一个特点，它将多个`NULL`视为重复值，因此你需要使用这个变通方法：

```sql
--  MSSQL
CREATE UNIQUE INDEX uq_customers_phone
ON customers(phone)
WHERE phone IS NOT NULL;
```

注意这次索引名称以`uq`开头，以提醒这是一个唯一索引。同样，索引的命名没有硬性规定，但这个遵循了一个常见且易于理解的模式。

你是否真的想禁止重复电话号码是另一个问题。来自同一家庭或组织的两个客户很可能共享同一个电话号码，因此禁止重复会造成问题。这是一个关于*如何*禁止重复的练习，但不一定是关于*是否*要禁止重复的决定。这最好由具体数据库的需求来决定。

## 回顾

一个设计良好的 SQL 数据库需要遵循一些规则，以确保数据是可靠的。虽然不能保证数据是*真实*的，但数据至少应该是*有效*的。

### 范式

遵循某些设计原则的表被称为处于某种范式。这并不意味着它是常见的，而是指它处于一种明确的形式。

规范化表具有以下属性：

*   数据是原子的。
*   行是无序的。
*   行是唯一的。
*   行之间是独立的。
*   列之间是独立的。
*   列是单一类型的。
*   列名是唯一的。
*   列是无序的。

### 多值处理

在开发表时，一个主要问题是如何处理多值和重复值。通常的解决方案是使用额外的表，并通过外键将它们连接起来。

### 修改表

在重构或加固数据库时，你需要对现有的表和列进行更改。`ALTER TABLE`语句可用于：

*   添加额外的列，包括外键
*   删除列
*   添加或删除约束
*   添加或删除索引

约束包括添加`NOT NULL`、默认值以及其他`CHECK`约束。

### 视图

视图是一个保存的 SELECT 语句。创建视图的一个原因是为了方便，将一个或多个表的数据集中在一个地方。

有时，当你创建一个包含组合数据的视图时，最终得到的结果可能不再遵循所有规范化规则。在业内，这被称为**反规范化**。

反规范化数据通常是维护数据的一种糟糕方式，但经常是提取数据的一种便捷方式。从这个意义上说，它结合了两者的优点：原始数据在原始表中仍然完好无损。

一些数据库管理系统支持在视图中更新数据。实际上，更新不会影响视图本身，而是传递到底层的表。

