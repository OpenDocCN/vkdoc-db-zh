# 创建视图

您的一些计算可能会变得复杂，而您的一些查询肯定也会如此。当您需要重用一个查询时，您可以将其保存为**视图**。

视图是一个保存好的查询。视图本身被永久保存在数据库中。例如，假设您有一个查询，它使用本章中的一些计算来生成一个简单的价格列表：

```sql
SELECT
id,
(SELECT givenname||' '||familyname FROM artists WHERE
artists.id=paintings.artistid) AS artist,
title,
price, price*0.1 AS tax, price*1.1 AS inc,
CASE
WHEN price<170 THEN 'cheap'
WHEN price>170 THEN 'expensive'
ELSE ''
END AS pricegroup,
(SELECT nationality FROM artists
WHERE artists.id=paintings.artistid) AS nationality
FROM paintings;
```

您得到的结果可以作为一个价格列表使用。

| id | artist | title | price | … | pricegroup | nationality |
| --- | --- | --- | --- | --- | --- | --- |
| 1222 | … | … | 125 | … | cheap | French |
| 251 | … | … | 105 | … | cheap | Norwegian |
| 2190 | … | … | 185 | … | expensive | French |
| 1560 | … | … | 125 | … | cheap | French |
| 172 | … | … | 195 | … | expensive | French |
| 2460 | … | … | 165 | … | reasonable | Flemish |
| ~ 1273 rows |

这是您不想每次使用都重新执行一遍的东西。相反，您可以通过创建视图来保存查询：

```sql
CREATE VIEW pricelist AS
SELECT
id,
(SELECT givenname||' '||familyname FROM artists WHERE
artists.id=paintings.artistid) AS artist,
title,
price, price*0.1 AS tax, price*1.1 AS inc,
CASE
WHEN price<170 THEN 'cheap'
WHEN price>170 THEN 'expensive'
ELSE ''
END AS pricegroup,
(SELECT nationality FROM artists
WHERE artists.id=paintings.artistid) AS nationality
FROM paintings;
```

您现在可以像使用另一个表一样使用 `pricelist` 视图：

```sql
SELECT * FROM pricelist;
```

您可以将视图视为一个虚拟表。重要的是，它不是数据的副本；如果您删除视图，除了视图本身，您不会丢失任何东西。原始数据仍然完好无损。

您可以为视图取几乎任何您喜欢的名称，但显然不能有两个同名的视图。不太明显的是，您也不能创建与表同名的视图。这是因为 SQL 将视图视为另一个表。

如果您不再需要该视图，可以使用 `DROP VIEW` 语句：

```sql
DROP VIEW pricelist;
```

为了安全起见，您可以用 `IF EXISTS` 来限定该语句；否则，如果视图一开始不存在，您可能会得到一个错误：

```sql
--  非 Oracle 或旧版 MSSQL
DROP VIEW IF EXISTS pricelist;
```

请注意，Oracle 以及旧版本的 MSSQL 不支持 `IF EXISTS`。

您不必为了节省空间而删除视图，因为它们几乎不占用任何空间。

然而，如果您想*修改*视图，最简单的方法是先删除它，然后再重新创建。

一旦您理解了视图的价值，您就会发现它们是在已有代码基础上构建的一种有用技术。当您投入了大量工作来创建 `SELECT` 语句后，您可以保存它并在需要时随时重用。

唯一的缺点是 `CREATE VIEW` 需要额外的数据库权限，因为您是在对数据库本身进行更改。我们建议您采取一切必要手段来获得这些权限——纠缠、贿赂、要挟，或者任何有效的方法。

## 在 Microsoft SQL Server 中使用视图

在大多数情况下，视图的工作方式与其他数据库管理系统相同。然而，Microsoft 有一些显著的不同之处：

*   视图不能包含 `ORDER BY` 子句，除非您将其与 `OFFSET … FETCH …` 子句或 `TOP` 表达式结合使用。
*   `CREATE VIEW` 语句必须与脚本的其他部分分开。
*   旧版本不支持 `IF EXISTS`；您必须使用更复杂的表达式，例如 `IF OBJECT_ID('something','V') IS NOT NULL DROP VIEW something`。

Microsoft 要求您将大多数 `CREATE something` 语句与脚本的其他部分分开，唯一的例外是 `CREATE TABLE`。这可以使用一个特殊的关键字 `GO` 来完成，例如：

```sql
GO
CREATE VIEW pricelist AS
SELECT
--  等等
FROM paintings;
GO
```

`GO` 实际上是一个指令，用于处理前面的代码并开始一个新的批处理。

`GO` 不是 SQL 标准的一部分（没有其他数据库管理系统需要它），并且它本身也有奇特的行为。通常，它应该单独占一行，可能不缩进，并且关键字后面不应有任何其他内容。



## 摘要

虽然你的数据应以其最简单的形式保存，但你可以通过计算来生成该数据的变体。

你可以在没有表的情况下使用 `SELECT` 子句来试验计算和公式；在 Oracle 中，你需要使用一个名为 `dual` 的虚拟表。

虽然你通常从表中的现有值计算某个值，但有时你可能会使用内置值，例如当前日期和时间。你也可以使用一个固定值。

### 数据类型

在 SQL 中，主要有三种数据类型：数字、字符串和日期。每种数据类型都有其特定的方法和函数来计算值：

*   对于数字，你可以进行简单的算术运算，也可以使用更复杂的函数进行计算。还有一些函数可以近似数字。
*   对于日期，你可以计算日期之间的年龄或对日期进行偏移。你也可以提取日期的各个部分。
*   对于字符串，你可以连接它们、更改字符串的部分内容或提取字符串的部分内容。
*   对于数字和日期，你可以生成一个格式化的字符串，这可能会给你一个更友好的版本。

## 空值

每当计算涉及到 `NULL` 时，它会对结果产生灾难性的影响，结果将是 `NULL`。

在某些情况下，你可以使用 `coalesce()` 来替换一个值，它将用一个合理的替代值替换 `NULL`。当然，你需要确定什么对你而言是“合理的”。

## 别名

每一列都应该有一个唯一的名称。当你计算一个值时，你需要使用 `AS` 提供这个名字作为别名。你也可以对非计算列执行此操作，以提供一个更合适的名称。

别名和其他名称应该是唯一的。它们还应该遵循标准的列命名规则，例如不与 SQL 关键字相同，并且不包含特殊字符。

如果出于任何原因，一个名称或别名需要打破命名规则，你可以将该名称用双引号括起来，或者使用 DBMS 提供的任何替代方案。

## 子查询

一列也可以包含一个源自子查询的值。如果你想包含来自另一个相关表的数据，这尤其有用。

如果子查询涉及到主表中的一个值，则称其为相关子查询。这种子查询可能成本高昂，但仍然是一种有用的技术。

## CASE 表达式

你可以使用 `CASE … END` 来生成类别，它测试一个值与可能的匹配项，并从多个备选值中返回一个结果。

## 类型转换值

你可以使用 `cast()` 来更改值的数据类型：

*   你可以在主类型内更改为具有更多或更少细节的类型。
*   如果值充分类似于另一种类型，有时你可以在主类型之间进行更改。

有时，类型转换是自动执行的，但有时你需要自己动手。

一个你可能需要从字符串进行类型转换的情况是当你需要一个日期字面量时。由于字符串和日期字面量都使用单引号，SQL 可能会将日期误认为是字符串。

## 视图

你可以通过创建视图将 `SELECT` 语句保存到数据库中。视图允许你将复杂的语句保存为虚拟表，以便以后以更简单的形式使用。

视图是构建有用语句集合的好方法。

### 后续内容

到目前为止，大部分工作都涉及单表。每当我们需要包含来自另一个表的数据时，我们只是简单地使用一个子查询，该子查询查询另一个表并返回结果。

在下一章中，我们将通过将两个或多个表组合在所谓的 `join` 中，来探讨如何生成一个新的虚拟表。

