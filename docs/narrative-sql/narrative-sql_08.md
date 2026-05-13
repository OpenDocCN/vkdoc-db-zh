# 2. 从 SELECT 开始

在本章中，你将学习从数据库检索数据的基础 `SQL` 命令。本章通过叙述和实际示例演示 `SELECT` 语句的多功能性，展示其在简单和复杂的数据检索场景中如何使用。

## SELECT 简介

`SELECT` 语句是最基本和最常用的 `SQL` 命令之一，几乎是每个查询操作的核心。对于任何 `SQL` 用户来说，它是从数据库检索数据的必备工具。从本质上讲，`SELECT` 语句允许你精确指定要从哪些表中检索哪些数据，并且可以通过各种子句进行微调，以满足精确的数据检索需求。`SELECT` 不仅仅是将数据从数据库中提取出来；它是选择能够回答特定问题或满足特定信息需求的正确数据片段。因此，`SELECT` 语句可以将原始数据转化为有意义的信息。

### SELECT 在数据叙事中的重要性

在数据叙事方面，`SELECT` 超越了数据检索，成为辅助叙事构建的工具。数据叙事是将数据编织成对受众有意义的叙事，帮助他们通过现实世界的故事理解复杂信息。

例如，为了提取构成这些故事基础的数据点，可以使用 `SELECT` 来提取各种数据点。考虑一个包含多年销售数据的数据库，这些数据是在一段时间内收集的。叙事者可以使用 `SELECT` 命令来检索主要活动或假期期间的总销售额并进行分析。仔细选择数据可以更多地揭示特定时期内的消费者行为。通过选择特定的数据，叙事者可以强调能够说明趋势、支持假设或解释现象的信息，这种方式在视觉和语境上都具有说服力。

### SELECT 语句的剖析

`SELECT` 语句的结构和语法对于最大化 `SQL` 在数据检索和叙事中的能力至关重要。本小节分解了 `SELECT` 语句的基本组成部分，包括如何选择列和使用别名使查询更具可读性。

#### 基本结构和语法

`SELECT` 语句用于查询数据库并检索指定的数据。最基础地，`SELECT` 语句必须指定两个关键信息：要选择什么以及从哪里选择。语法易于理解：

```
SELECT column1, column2, ...
FROM tableName;
```

在此结构中：

*   `SELECT` 表示你即将指定要检索的列或表达式列表。
*   `column1, column2, ...` 是你希望从数据库表中选择的具体列。你也可以使用 `*` 字符来选择表中的所有列。
*   `FROM tableName` 指定从中检索数据的表。

这个简单的语法是构建更复杂查询的起点，包括那些过滤、分组或排序数据的查询。

## 第一个故事：书店周年庆

这个故事基于一个名为 `Customers` 的数据库。每个角色都有独特的背景和故事。John Doe、Jane Smith 和 Alice Johnson 是一家书店的老顾客。书店下个月将庆祝周年纪念日。如果其尊贵的顾客能够参加，这次活动将会取得成功。这需要为每位注册客户准备个性化的邀请函，而这些邀请函所需的关键信息包括每位客户的全名和电子邮件地址。表 2-1 显示了名为 `Customers` 的表。

表 2-1

Customers 表

| CustomerID | FirstName | LastName | Email | DateOfBirth | Neighborhood | Address | ZIP Code | Phone Number |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | John | Doe | `johndoe@example.com` | 1980-05-15 | Downtown | 123 Elm St. | 90210 | (555) 321-9876 |
| 2 | Jane | Smith | `janesmith@example.com` | 1975-07-20 | Midtown | 456 Oak Ave. | 90212 | (555) 654-1234 |
| 3 | Alice | Johnson | `alicej@example.com` | 1990-11-12 | Eastside | 789 Pine Rd. | 90213 | (555) 789-2342 |


### 从表中选择列

在构建叙述或进行分析时，您经常需要从一个或多个表中获取特定的数据片段。选择特定的列可以让您专注于与您的故事或分析相关的数据。例如，如果您有一个名为`Customers`的数据库表，其中包含`CustomerID`、`FirstName`、`LastName`、`Email`、`DateOfBirth`、`Neighborhood`、`Address`、`Zip Code`和`Phone Number`等字段，而您的故事或分析只需要知道客户姓名和电子邮件，那么正确的`SELECT`语句应类似如下：

```sql
SELECT FirstName, LastName, Email
FROM Customers;
```

表 2-2 显示了执行仅选择`FirstName`、`LastName`和`Email`列的查询的结果。

**表 2-2**
SELECT 查询执行结果

| FirstName | LastName | Email |
| --- | --- | --- |
| John | Doe | `johndoe@example.com` |
| Jane | Smith | `janesmith@example.com` |
| Alice | Johnson | `alicej@example.com` |

### 为列引入别名

SQL 中的别名用于重命名 SQL 查询输出中的列或表。它们对于提高查询结果的可读性特别有用。例如，使用`Contact Email`代替单纯的`Email`可以显著增强清晰度。这在连接多个表时同样适用，这将在第 4 章中介绍。当连接多个具有名称相同但含义不同的列的表时，使用别名也很有帮助。`AS`关键字用于列名之后来定义别名。然而，`AS`关键字是可选的。

```sql
SELECT FirstName AS First, LastName AS Last, Email AS ContactEmail
FROM Customers;
```

在此查询中，`FirstName`、`LastName`和`Email`在输出中分别被重命名为`First`、`Last`和`ContactEmail`。别名在报告和导出数据中尤其有用，使其成为数据呈现和讲故事中必不可少的工具。参见表 2-3。

**表 2-3**
带 AS 的 SELECT 查询执行结果

| First | Last | ContactEmail |
| --- | --- | --- |
| John | Doe | `johndoe@example.com` |
| Jane | Smith | `janesmith@example.com` |
| Alice | Johnson | `alicej@example.com` |

### 介绍 CONCAT 函数

为完成此过程，最后一步是构建以下 SQL 查询，该查询将连接姓和名以获取全名和电子邮件地址。

```sql
SELECT CONCAT(FirstName, ' ', LastName) AS FullName, Email
FROM Customers;
```

这条 SQL 语句利用`CONCAT()`函数将`FirstName`和`LastName`合并到一个`FullName`列中，从而简化了每份邀请函的呈现和个性化处理。通过仅选择`FullName`和`Email`列，该查询高效地提取了必要信息，确保创建和发送邀请函的流程得以简化且目标明确。参见表 2-4。

**表 2-4**
SELECT、AS 和 CONCAT 函数查询执行结果

| FullName | Email |
| --- | --- |
| John Doe | `johndoe@example.com` |
| Jane Smith | `janesmith@example.com` |
| Alice Johnson | `alicej@example.com` |

此表高效地呈现了发送书店周年纪念邀请函所需的信息，仅展示每位顾客合并后的全名和电子邮件地址。

注意

`CONCAT`是 SQL 中的一个函数，而非操作。它用于将两个或多个字符串连接或拼接成一个字符串。`CONCAT`函数是合并来自不同列的数据或从数据库中的多个字符串字段创建格式化输出的重要工具。`CONCAT`函数的基本语法如下：

`CONCAT(string_column_1, string_column_2, ..., string_column_N)`

需要注意的是，PostgreSQL 中的`CONCAT`函数可用于多种场景。然而，在某些特定情况下它可能不适用，或者替代方法可能更合适。`||`操作符是 PostgreSQL 中连接字符串的标准方法，可以作为`CONCAT`函数的替代方案。例如，考虑以下使用`CONCAT`函数的示例：

`SELECT CONCAT(string_column_1, string_column_2, ..., string_column_N) AS concatenated_string`

`FROM your_table;`

或者，可以使用`||`操作符实现相同的结果，这是标准的 SQL 字符串连接方法：

`SELECT string_column_1 || string_column_2 || ... || string_column_N AS concatenated_string`

`FROM your_table;`

`||`操作符被各种 SQL 数据库广泛支持，与一些专有函数（如`CONCAT`）相比，使其成为一个可移植性更强的选择。两种方法都能有效地连接字符串，选择哪一种取决于可读性、性能考虑以及特定的数据库要求等因素。

### 使用 SELECT 进行 SQL 数学运算

在 SQL 中，数学运算对于转换和分析数据至关重要。这些操作使您能够跨数据列执行计算，从而增强通过数字传达有意义信息的能力。本节探讨 SQL 中可用的基本数学运算，用汇总表说明这些操作，并展示如何将这些技术应用于讲故事。

SQL 支持多种可以直接在`SELECT`语句中应用的数学运算。包括加法(`+`)、减法(`-`)、乘法(`*`)和除法(`/`)。可以直接在数据库查询中使用这些操作对现有数据进行计算，以生成新值。以下查询提供了一个使用数学运算计算简单乘积的基本示例：

```sql
SELECT column1, column1 * column2 as Product
FROM Table;
```

在此查询中，`column1`和`column2`是`Table`中的现有列，而`Product`是在输出中创建的新列，包含`column1`和`column2`的乘积。

表 2-5 总结了一些最常用的 SQL 数学运算。

**表 2-5**
最实用的 SQL 数学运算示例

| 运算 | SQL 符号 | 示例用法 | 描述 |
| --- | --- | --- | --- |
| 加法 | `+` | `column1 + column2` | 将两列相加。 |
| 减法 | `-` | `column1 - column2` | 将一列减去另一列。 |
| 乘法 | `*` | `column1 * column2` | 将两列相乘。 |
| 除法 | `/` | `column1 / column2` | 将一列除以另一列。 |
| 取模 | `%` | `column1 % column2` | 将一列除以另一列并返回余数。 |

这些操作可用于创建派生列，并更深入地了解您的数据。


## 第二个故事：面包店销售数据分析

有一家小型面包店希望分析其收集到的销售数据。他们有一张名为 `Sales` 的表，其中包含 `CustomerID`、`ProductName`、`Price`、`RegularPrice`、`Discount` 和 `Quantity` 等列。他们在分析中提出了以下问题：
*   哪种口味的甜甜圈最受欢迎？
*   谁是我们最常光顾的顾客？
*   每种产品产生了多少收入？

在回答每个问题之前，请考虑 `Sales` 表，如表 2-6 所示。

表 2-6
销售表

| CustomerID | ProductName | Price | Quantity | RegularPrice | Discount |
| --- | --- | --- | --- | --- | --- |
| 743663 | Glazed Donut | 2.5 | 2 | 3.0 | 0.5 |
| 743663 | Chocolate Muffin | 3.0 | 1 | 3.0 | 0.0 |
| 223424 | Blueberry Muffin | 3.0 | 2 | 3.0 | 0.0 |
| 323423 | Chocolate Chip Cookie | 1.5 | 3 | 2.0 | 0.5 |
| 432424 | Sugar Cookie | 1.0 | 4 | 1.0 | 0.0 |

第一个问题可以用以下查询来回答。他们可以使用简单的 `SELECT` 语句来确定最受欢迎的甜甜圈口味。
```sql
SELECT ProductName, SUM(Quantity) AS TotalSold
FROM Sales
GROUP BY ProductName;
```
此查询选择 `ProductName`，并使用 `SUM(Quantity)` 计算总销售数量，并赋予别名 `TotalSold`。`SUM()` 是 SQL 中的一个函数，用于计算表中数值列的总和。需要注意的是，此函数默认计算指定数值列中所有值的总和，并忽略 `NULL` 值。在此查询中，`GROUP BY ProductName` 按产品名称对所有销售记录进行分组，因此 `SUM(Quantity)` 计算每个独特产品的总销售数量。这有助于按产品汇总数据，而不是对整个表进行汇总。参见表 2-7。

表 2-7
第一个问题查询执行结果

| ProductName | TotalSold |
| --- | --- |
| Chocolate Chip Cookie | 3 |
| Glazed Donut | 2 |
| Blueberry Muffin | 2 |
| Chocolate Muffin | 1 |
| Sugar Cookie | 4 |

注意
在 SQL 中，`GROUP BY` 子句用于将指定列中具有相同值的行分组为汇总行。它通常与聚合函数（如 `SUM()`、`AVG()`、`COUNT()` 和 `MAX()`）结合使用，以对每组数据执行计算。第 5 章将详细介绍 `GROUP BY` 子句，而第 5 和 10 章则涵盖更广泛的聚合函数。

要回答第二个问题（最常光顾的顾客），可以使用以下查询：
```sql
SELECT CustomerID, COUNT(*) AS PurchaseCount
FROM Sales
GROUP BY CustomerID;
```
此查询选择 `CustomerID`，并使用 `COUNT(*)` 计算购买次数，并赋予别名 `PurchaseCount`。SQL 中的 `COUNT(*)` 是一个用于获取表中总行数的函数。需要注意的是，此函数计算表中的所有行，包括重复行和包含 `NULL` 值的行。

注意
`COUNT(*)` 与 `COUNT(column_name)` 不同，后者仅计算特定列中非 `NULL` 值的数量。在 SQL 中，星号 (`*`) 用于选择所有列。星号代表表中的所有列。在 `SELECT` 中使用 `*` 对于大表可能效率低下，因为它会检索所有数据。为获得更好的性能，请考虑选择特定列。

表 2-8 显示了仅选择 `CustomerID` 和 `PurchaseCount` 列的查询执行结果。

表 2-8
第二个问题查询执行结果

| CustomerID | PurchaseCount |
| --- | --- |
| 743663 | 2 |
| 223424 | 1 |
| 323423 | 1 |
| 432424 | 1 |

要回答第三个问题（每种产品产生了多少收入），可以使用以下查询：
```sql
SELECT
CASE
    WHEN ProductName LIKE '%Donut%' THEN 'Donuts'
    WHEN ProductName LIKE '%Muffin%' THEN 'Muffins'
    ELSE 'Cookies'
END AS Category,
SUM(Price * Quantity) AS TotalRevenue
FROM Sales
GROUP BY
CASE
    WHEN ProductName LIKE '%Donut%' THEN 'Donuts'
    WHEN ProductName LIKE '%Muffin%' THEN 'Muffins'
    ELSE 'Cookies'
END;
```
这个查询稍微复杂一些。它使用 `CASE` 语句根据产品名称对产品进行分类。它使用 `SUM(Price * Quantity)` 计算总收入。为了找到答案，查询分析产品名称并将其分类为 `Donuts`（甜甜圈）、`Muffins`（松饼）或 `Cookies`（饼干）。然后，将每件商品的价格乘以销售数量并相加，得到每个类别产生的总收入。通过这种方式，可以计算出每个产品类别产生的收入。

需要注意的是，`%` 符号是一个特殊字符，用于在字符串内执行模式匹配。`%` 在模式中代表任意字符序列，即零个或多个字符。例如，`%Donut%` 是一个匹配模式。在这个特定情况下，查询查找产品名称中任何位置包含子串 `"Donut"` 的记录。因此，它将匹配像 `"Glazed Donut"` 和 `"Chocolate Donut with Sprinkles"` 这样的产品名称。

最后，`GROUP BY` 子句与相同的 `CASE` 表达式一起使用，以便在应用 `SUM` 函数之前，按分配的类别（`Donuts`、`Muffins` 或 `Cookies`）对记录进行分组。这确保了每个产品类别的收入汇总正确。表 2-9 显示了结果。

表 2-9
第三个问题查询执行结果

| Category | TotalRevenue |
| --- | --- |
| Donuts | 5 |
| Muffins | 9 |
| Cookies | 8.5 |

如表 2-9 所示，`CASE` 语句对产品进行了分类。名称中包含 `"Donut"` 的产品成为 `Donuts`，包含 `"Muffin"` 的成为 `Muffins`，其他所有产品则成为 `Cookies`。`SUM(Price * Quantity)` 计算了每个类别的总收入。

## CASE 语句

SQL 中的 `CASE` 语句类似于查询中的 `if-then-else` 逻辑。它评估一系列条件，并根据第一个匹配的条件返回相应的值。
*   `WHEN` 语句指定要评估的条件。它可以是任何返回布尔值（真/假）的有效 SQL 表达式。
*   如果相应的 `WHEN` 条件计算结果为 `true`，则由 `THEN` 返回此值。
*   `ELSE` 是一个可选子句，如果没有任何 `WHEN` 条件满足，则提供默认结果。如果省略且没有条件为真，通常会返回 `NULL`。

```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ...
    ELSE result_else
END
```

对于更复杂的查询，可以嵌套 `CASE` 语句。你可以在第 8 章中特别了解更多关于 `CASE` 和嵌套 `CASE` 的内容。


### 字符串模式

SQL 中的字符串模式允许您基于文本列中的特定模式来搜索和过滤数据。SQL 字符串模式中常用两种通配符：`%`符号代表任意数量的字符，`_`符号代表单个字符。这些通配符为基于不同模式进行数据搜索和过滤提供了灵活性。在您了解了 `WHERE` 和 `LIKE` 语句之后，第 3 章将探讨更复杂的字符串匹配示例。

在回答了三个问题后，莎拉正在为一个促销活动做准备。她的目标是对部分商品提供折扣，但不确定哪些商品会对她的客户产生最大的影响。理想情况下，她希望针对的商品是：

*   `热门`：销售数量高的商品。
*   `尚未打折`：她不想对已经打折的商品再进行折扣。

为了识别这些理想的促销候选商品，莎拉决定分析她的销售数据。使用以下两个 `CASE` 语句，可以实现这个查询。

```sql
SELECT ProductName, Quantity,
CASE
WHEN Quantity >= (SELECT AVG(Quantity) FROM Sales) THEN '热门'
ELSE '不太热门'
END AS Popularity,
CASE WHEN RegularPrice > Price THEN '已折扣' ELSE '原价' END AS Price_Status
FROM Sales
```

在第一个 `CASE` 中，使用子查询计算平均数量，并根据商品数量是否达到或超过平均值进行分类。第二个 `CASE` 语句检查价格状态。这确保了莎拉专注于热门商品的折扣。如果商品是 `热门`（根据外部 `CASE` 判断）且原价 (`RegularPrice`) 高于当前价格 (`Price`)，则意味着该商品已打折。如果商品是 `热门`但没有折扣（原价等于当前价格），则被归类为 `原价`。

注意：子查询（也称为内部查询或嵌套查询）是在彼此内部嵌入 `SELECT` 语句的强大 SQL 工具。它们用于为外部查询检索数据。随着您对其重要性认识的加深，这类查询将在后续章节中更详细地讨论。

表 2-10 是执行查询以选择仅 `ProductName`、`Quantity`、计算出的 `Popularity` 和 `PriceStatus` 列的结果。

表 2-10：促销活动查询执行结果

| 商品名称 | 数量 | 热度 | 价格状态 |
| --- | --- | --- | --- |
| Glazed Donut | 2 | 不太热门 | 已折扣 |
| Chocolate Muffin | 1 | 不太热门 | 原价 |
| Blueberry Muffin | 2 | 不太热门 | 原价 |
| Chocolate Chip Cookie | 3 | 热门 | 已折扣 |
| Sugar Cookie | 4 | 热门 | 原价 |

## 去重选择的艺术

要通过数据构建引人入胜的叙述，有效利用 `DISTINCT` 关键字至关重要。在 SQL 中，`DISTINCT` 关键字通过确保查询结果包含唯一值来优化数据检索。它就像一个过滤器，消除了可能扭曲您的分析或叙述的重复行。其基本语法如下：

```sql
SELECT DISTINCT column
FROM table
```

这从 `table` 中的指定 `column` 仅检索唯一值。

`DISTINCT` 会移除所有选定列的值都相同的行。例如，如果一个表有 `CustomerID` 和 `OrderDate`，`DISTINCT CustomerID` 将返回每个唯一的客户，即使他们下了多个订单。

`DISTINCT` 语句与列的组合一起使用时，可以确保跨多列的唯一性。例如，`SELECT DISTINCT CustomerID, ProductID` 将仅返回客户和产品组合均唯一的行。

您还可以将 `DISTINCT` 与 `SUM` 等聚合函数结合使用，以揭示数据中的趋势。

## 第三个故事：糖果店销售数据

菲比渴望扩展她那充满五彩缤纷糖果的热闹糖果店。她的销售报告包含了今天的购买记录，她很好奇她客户的最爱是什么。表 2-11 显示了销售日志。

表 2-11：销售日志

| 订单 ID | 糖果 | 数量 |
| --- | --- | --- |
| 1 | Gummy Bears | 2 |
| 2 | Lollipops | 1 |
| 3 | Gummy Bears | 3 |
| 4 | Chocolate Bars | 2 |
| 5 | Lollipops | 2 |
| 6 | Gummy Bears | 1 |

日志显示了每一笔销售，但没有显示她客户渴望的独有糖果。她需要的“放大镜”就是 `DISTINCT`：

```sql
SELECT DISTINCT Candy
FROM sales_log
```

此查询揭示了表 2-12 中的数据。

表 2-12：来自销售日志表的独有糖果

| 糖果 |
| --- |
| Gummy Bears |
| Lollipops |
| Chocolate Bars |

注意：在大型数据集上使用 `DISTINCT` 可能会影响性能。重要的是要考虑您是否真的需要所有可能的组合，或者事先进行过滤是否足够。接下来的章节将更详细地讨论数据过滤。

## 使用 SELECT 进行聚合

在 SQL 中，聚合函数对于计算数据集的特征并生成一个总结其特征的值至关重要。这些函数允许分析师从数据中提取更多意义，将原始数据转化为支持决策和叙述的有用见解。SQL 中的数据分析严重依赖聚合函数，这些函数允许跨大量数据进行计算，以创建高效的摘要和高级概览。


### SQL 中常规算术函数与聚合函数的区别

SQL 中的常规算术函数适用于处理单个行内的数据。它们执行加法、减法、乘法等计算，逐行转换或组合值。而聚合函数则着眼于更宏观的层面。汇总整个数据组是其优势所在。`COUNT`、`SUM`、`AVG`、`MIN` 和 `MAX` 等函数将多行数据合并为一个有意义的单一数值，揭示特定列内的总量、平均值以及高低值。与常规算术函数相比，聚合函数有助于你识别数据中的趋势和模式。

以下列表描述了 SQL 中常规算术函数与聚合函数的一些关键区别。

**常规算术函数**：

*   **目的**：执行逐行计算。它们操作单个行内的数据，或组合一行内多个列的值。
*   **示例**：`+`, `-`, `*`, `/`, `MOD (取模)`, `ROUND`, `SQRT`, `ABS (绝对值)` 等。
*   **输出**：为查询中处理的每一行返回一个单一值。
*   **关注单个值**：它们操作行内的单个值，根据需要进行转换或组合。

**聚合函数**：

*   **目的**：通过对整个值组执行计算来汇总数据。它们将多行压缩为一个有意义的单一数值。
*   **示例**：`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`。
*   **输出**：返回一个单一数值，代表跨多行计算的整体结果。
*   **关注汇总**：它们专注于为特定列或列集中的数据提供汇总统计信息（总计、平均值、最小值、最大值等）。

表 2-13 总结了常规算术函数与聚合函数的区别。

表 2-13

常规算术函数与聚合函数的区别

| 特性 | 常规算术函数 | 聚合函数 |
| --- | --- | --- |
| 目的 | 逐行计算 | 汇总数据组 |
| 输入 | 行内的值 | 行组 |
| 输出 | 每行一个值 | 单一值 |
| 重点 | 单个值操作 | 汇总 |

举例来说，为了说明这两者的区别，考虑表 2-14 中所示的 `Orders` 表，它包含 `OrderID`、`CustomerID` 和 `Amount` 列。

表 2-14

订单表

| OrderID | CustomerID | Amount |
| --- | --- | --- |
| 1 | 101 | 100 |
| 2 | 102 | 50 |
| 3 | 103 | 75 |
| 4 | 101 | 275 |

此查询通过逐行乘法计算每个订单的 10%折扣，如表 2-15 所示：

表 2-15

添加了折扣的订单表

| OrderID | CustomerID | Amount | Discount |
| --- | --- | --- | --- |
| 1 | 101 | 100 | 10 |
| 2 | 102 | 50 | 5 |
| 3 | 103 | 75 | 7.5 |
| 4 | 101 | 275 | 27.500 |

```sql
SELECT OrderID, CustomerID, Amount, Amount * 0.1 AS Discount
FROM Orders;
```

该查询通过将 `Amount` 乘以 0.1 来计算每个订单的 10%折扣，并在结果集中添加了一个名为 `Discount` 的新列。

下面的查询使用聚合函数获取所有订单的总订单数 (`COUNT(*)`) 和总销售额 (`SUM(Amount)`)。

```sql
SELECT
CustomerID,
COUNT(*) AS TotalOrders,
SUM(Amount) AS TotalSales
FROM Orders
GROUP BY CustomerID;
```

如表 2-16 所示，此查询使用聚合函数获取所有订单的总订单数和总销售额。

表 2-16

所有订单的总订单数和总销售额

| Customerid | TotalOrders | TotalSales |
| --- | --- | --- |
| 101 | 2 | 375.00 |
| 102 | 1 | 50.00 |
| 103 | 1 | 75.00 |

表 2-17 阐述了在 SQL 数据分析中发挥关键作用的一些主要聚合函数。

表 2-17

一些主要的聚合函数

| 聚合函数 | 描述 |
| --- | --- |
| `COUNT` | 用于计算特定列或数据集中的项目数量，有助于确定数据中各种类别的规模、范围或频率。例如，`COUNT` 可以告诉书店库存中每个类别有多少本书，为库存管理决策提供定量基础。 |
| `AVG` | 用于计算数值列的平均值，这对于在处理价格、年龄或任何可测量的数量等变量时理解典型值至关重要，因为平均值可以洞察正常行为或预期结果。例如，确定其书籍的平均页数可以帮助出版商了解各体裁内的典型出版长度。 |
| `MAX` | 找出列中的最高值。当你需要识别数据集中的峰值或最大水平时，这尤其有用，例如查找商店中售出的最贵物品或个人取得的最高分数。`MAX` 可以在数据分析中突出异常值或特殊情况。 |
| `MIN` | 相反，`MIN` 函数确定列中的最低值。它对于识别最不极端的情况至关重要，例如成本最低的产品，这对于希望向客户营销入门级选项的企业可能很有用。 |
| `STDDEV` | 计算指定数值列的标准差，它衡量一组数值的变化或离散程度。在质量控制、金融以及任何变异性是分析关键的领域中都很有用。 |
| `VAR` | 计算指定数值列的方差，类似于 `STDDEV`，但给出离散度的平方。这在金融和科学计算中至关重要，因为理解变异性是必不可少的。 |



