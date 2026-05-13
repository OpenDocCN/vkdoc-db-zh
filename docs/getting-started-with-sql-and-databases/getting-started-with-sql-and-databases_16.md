# 5. 计算列值

到目前为止，你主要处理的是来自表中直接的数据值。然而，你也可以重新计算值，以获得更适合特定情况的形式。

关于在 SQL 表列中存储的值的类型，有一些相互关联的原则：

*   值不应重复。

    这包括同一值的不同变体，例如以厘米和英寸为单位的身高，或者出生日期和年龄。
*   所有列应彼此独立。

    原则上，更改一列的值不应影响另一列，例如出生日期和电话号码。
*   值应采用其最简形式。

    这尤其适用于格式化字符，例如货币符号或电话号码中的空格。

换句话说，当你确实需要基于存储在数据库中的值进行变体计算时，解决方案是存储最简单的值并计算其余部分。

SQL 具有一定的计算能力。在使用各种 DBMS 时，你很快会注意到：

*   这种能力是有限的。SQL 并非真正的数据处理工具，但它可以进行一些简单的计算。
*   标准 SQL 的局限性更大，而各种 DBMS 通过自己的扩展补充了这一点。这里的问题是，没有两个 DBMS 拥有完全相同的功能。

有时，解决方案是让数据保持其简单形式，而让额外的软件进行更复杂的数据处理。例如：

*   你可以从数据库中提取基本数据，并使用像 PHP（一种用于生成网页的语言）这样的语言来执行更复杂的计算和格式化。
*   会计软件将从数据库中提取数据，并在应用程序内执行其所有的专门处理。
*   存在专门的程序和语言，例如 `R`，它们会从数据库中提取数据并在其上进行复杂的分析。

尽管如此，在将数据交给额外软件处理之前，先了解 DBMS 能为我们做什么是很有用的。

在本章中，我们将了解一些重新计算值的方法。我们将探讨数字、字符串和日期的基本计算，以及处理它们的专用函数。我们还将了解 `NULL` 如何影响计算，以及为计算结果命名的技术。

我们还将了解一些特殊技术，例如在子查询中从其他表获取结果、在 `CASE` 表达式中对值进行分类，以及使用 `case()` 函数重新解释数据类型。

最后，我们将了解如何以视图的形式在数据库中保存复杂的查询。

## 测试计算

大多数 DBMS 包含一项功能，从技术上讲是非标准的，但对于测试计算非常有用：

```
SELECT 2+5 AS result; --  非 Oracle
```

从技术上讲，`SELECT` 语句需要一个 `FROM` 子句。然而，在缺少该子句时，大多数 DBMS 会提供一个虚拟的单行表，但该表本身没有列，结果如下所示：

| result |
| --- |
| 7 |

你甚至可以计算这个虚拟表的行数：

```
SELECT count(*) AS count;    --  1 行
```

Oracle 也提供此功能，但它使用一个名为 `dual` 的虚拟表：

```
--  仅 Oracle
SELECT 2+5 FROM dual;
SELECT count(*) FROM dual;
```

对于大多数示例，我们不会写 `FROM dual`，但如果你使用的是 Oracle，请记得加上它。

当然，如果你没有包含表，就别指望能获取到任何真实数据。你仅限于使用数据字面量和一些内置值，例如当前时间。如果你真的想包含真实数据，你将需要使用稍后会学到的子查询。

只要我们想在不涉及真实表的情况下测试计算，就可以使用此功能。

## 模拟变量

我们的一些示例将使用一组任意值。在编码中，当你要设置临时值时，会创建一个变量。例如，在 JavaScript 中，你可以这样设置一个变量：

```
var a = 23;
```

一些 DBMS 确实有类似的功能，但我们将采用使用纯 SQL 的不同方法。假设，例如，你想设置一个任意税率，并将其应用于价格表。你可以这样做：

```
WITH vars AS (SELECT 0.1 AS taxrate)
SELECT
id, title, price, price*taxrate AS tax
FROM paintings, vars;
```

这给我们提供了一个简单的价格列表：

| id | title | price | tax |
| --- | --- | --- | --- |
| 1222 | Haymakers Resting | 125.00 | 12.500 |
| 251 | Death in the Sickroom | 105.00 | 10.500 |
| 2190 | Cache-cache (Hide-and-Seek) | 185.00 | 18.500 |
| 1560 | Indefinite Divisibility | 125.00 | 12.500 |
| 172 | Girl with Racket and Shuttlecock | 195.00 | 19.500 |
| 2460 | The Procession to Calvary | 165.00 | 16.500 |
| ~ 1273 行 ~ |

*   `WITH` 子句创建了一个（在此例中）一行一列的虚拟表。

    这个虚拟表被称为公用表表达式（CTE）。你可以随意命名它；这里它被称为 `vars`，这是“变量”的常用缩写。
*   `FROM` 子句将真实的 `paintings` 表与虚拟表结合起来，实际上是添加了另一列。

    从技术上讲，这称为交叉连接（CROSS JOIN），它将 `vars` 虚拟表的每一行（一行）与 `paintings` 表的每一行进行组合。

在这个简单的例子中，你本可以在计算中直接使用 `price*0.1` 并省去额外的代码。然而，在更复杂的示例中，以这种方式使用 CTE 可以使你的代码更易于管理和阅读。

你将在第 6 章学习更多关于连接表的内容。在接下来的章节中，你也将学习更多关于使用 CTE 的知识。

## 一些基本计算

一般来说，在计算时可以考虑三种基本数据类型：

*   数字用于计数或测量某物，或指示某种顺序。有些东西看起来像数字，例如电话号码，但实际上并不执行上述任何操作，因此它们不符合条件。
*   日期用于指示某事发生的时间。它们也可能包括时间，在某些情况下还包括时区。
*   其余的基本都是字符串。字符串是一串字符，它们可能具有也可能不具有文本意义。SQL 也将字符串称为字符数据。

### 基本数字计算

对于数字，SQL 支持基本的算术运算。例如：

```
SELECT id, title, price, price*0.1 AS tax
FROM paintings;
```

请注意，有些价格是 `NULL`。自然地，计算结果也是 `NULL`。

根据 DBMS 的不同，可能还有其他数学函数。

### 基本字符串计算

至于字符串，只有一种基本计算：

| fullname |
| Judy Free |
| Ray Gunn |
| Ray King |
| Ivan Inkling |
| Drew Blood |
| Seymour Sights |
| ~ 304 行 ~ |

```
--  标准
SELECT givenname || ' ' || familyname AS fullname
FROM customers;
--  MSSQL
SELECT givenname + ' ' + familyname AS fullname
FROM customers;
--  MySQL
SELECT concat(givenname,' ',familyname) AS fullname
FROM customers;
```

此操作称为串联（concatenation），它将字符串连接在一起。

请注意，Microsoft SQL Server 使用加号（`+`）运算符，这在更复杂的例子中会引起一些混淆。

如果你在 MySQL/MariaDB 中尝试此操作，它可能无法工作。在传统模式下，MySQL/MariaDB 将 `||` 运算符视为逻辑运算符；在 ANSI 模式下，它应该有效。

如果你使用的是 MySQL/MariaDB，请记住你可以通过以下方式切换到 ANSI 模式：

```
SET SESSION sql_mode = 'ANSI';
SELECT givenname || ' ' || familyname AS fullname;
```

`concat()` 函数适用于大多数 DBMS，但 SQLite 不适用。

### 基本日期计算

另一方面，日期的处理要复杂得多。即使是基本操作，你也需要一些额外的工作，这将在后面讨论。


## 处理空值

总是存在你需要与一些 `NULL` 值进行计算的风险。例如：

| id | givenname | familyname | inches |
| --- | --- | --- | --- |
| 474 | Judy | Free |   |
| 186 | Ray | Gunn | 64.48… |
| 144 | Ray | King | 69.60… |
| 179 | Ivan | Inkling | 67.04… |
| 475 | Drew | Blood | 67.32… |
| 523 | Seymour | Sights | 65.86… |
| ~ 304 行 ~ |

```sql
SELECT
  id, givenname, familyname,
  height/2.54 as inches
FROM customers;
```

许多行的 `height` 是 `NULL`。如你所见，它们的结果也是 `NULL`：如果你不知道一个值，那么无论你尝试对它做什么，你仍然不知道。这对于所有数据类型都是正确的。

一个你可以做的事情是过滤掉这些行：

```sql
SELECT
  id, givenname, familyname,
  height/2.54 as inches
FROM customers
WHERE height IS NOT NULL;
```

然而，有时你可以对缺失值进行猜测，或者至少用一个合理的替代值来填充。为了生成替代值，你可以使用 `coalesce(original, alternative)` 函数。

例如，有一个包含许多电话号码的员工表：

```sql
SELECT id, givenname, familyname, phone FROM employees;
```

这给出了名字和一些电话号码：

| id | givenname | familyname | phone |
| --- | --- | --- | --- |
| 26 | Mildred | Thisenthat | 0491570159 |
| 2 | Clarisse | Cringinghut | 0491571491 |
| 3 | Joe | Kerr |   |
| 5 | Norris | Toof |   |
| 20 | Jim | Pills |   |
| 17 | Harold | Prott |   |
| ~ 34 行 ~ |

有些电话号码缺失，但这并不一定意味着无法联系到这些人。对于这些缺失的电话号码，你可以用公司号码替代：

| id | givenname | familyname | phone |
| --- | --- | --- | --- |
| 26 | Mildred | Thisenthat | 0491570159 |
| 2 | Clarisse | Cringinghut | 0491571491 |
| 3 | Joe | Kerr | 1300975707 |
| 5 | Norris | Toof | 1300975707 |
| 20 | Jim | Pills | 1300975707 |
| 17 | Harold | Prott | 1300975707 |
| ~ 34 行 ~ |

```sql
SELECT
  id, givenname, familyname,
  coalesce(phone,'1300975707') AS phone
FROM employees;
```

有时，情况可能更明显。例如，`saleitems` 表包含了每个商品的数量 (`quantity`)：

| id | saleid | paintingid | quantity | price |
| --- | --- | --- | --- | --- |
| 1505 | 619 | 495 | 1 | 190.00 |
| 3278 | 1324 | 806 | 1 | 145.00 |
| 5505 | 2203 | 1585 |   | 105.00 |
| 806 | 329 | 1643 | 1 | 130.00 |
| 5805 | 2321 | 1713 | 3 | 105.00 |
| 1416 | 586 | 2147 | 3 | 160.00 |
| 367 | 147 | 630 | 1 | 200.00 |
| 497 | 202 | 2290 | 2 | 180.00 |
| 188 | 78 | 2186 |   | 155.00 |
| 964 | 395 | 701 | 1 | 190.00 |
| ~ 6315 行 ~ |

```sql
SELECT
  id, saleid, paintingid, quantity, price
FROM saleitems;
```

在这里，你会看到一些数量是 `NULL`。如果不知道数量，包含一个销售项目是没有意义的，但你可以合理地猜测，如果他们没有说明其他情况，那么数量应该是 `1`。你可以使用 `coalesce()` 来实现这个猜测：

```sql
SELECT
  id, saleid, paintingid,
  coalesce(quantity,1) AS quantity, price
FROM saleitems;
```

我们现在为每个项目都有了一个可行的数量：

| id | saleid | paintingid | quantity | price |
| --- | --- | --- | --- | --- |
| 1505 | 619 | 495 | 1 | 190.00 |
| 3278 | 1324 | 806 | 1 | 145.00 |
| 5505 | 2203 | 1585 | 1 | 105.00 |
| 806 | 329 | 1643 | 1 | 130.00 |
| 5805 | 2321 | 1713 | 3 | 105.00 |
| 1416 | 586 | 2147 | 3 | 160.00 |
| 367 | 147 | 630 | 1 | 200.00 |
| 497 | 202 | 2290 | 2 | 180.00 |
| 188 | 78 | 2186 | 1 | 155.00 |
| 964 | 395 | 701 | 1 | 190.00 |
| ~ 6315 行 ~ |

你也可以合理地认为，如果 `1` 是明显的值，那么它应该被内建为表的默认值。然而，你并不总能控制表应该如何设计。

有时，最好的替代是空，或者至少看起来像是空的东西。例如，在 `artists` 表中，有一些缺失的名字。如果你尝试连接它们，你的结果也会是 `NULL`：

```sql
SELECT
  id, givenname, familyname,
  givenname||' '||familyname AS fullname
--  givenname+' '+familyname AS fullname    --  MSSQL
FROM artists;
```

以下是一些缺失名字的结果：

| id | givenname | familyname | fullname |
| --- | --- | --- | --- |
| 215 |   | Cuyp |   |
| 349 | Armand | Guillaumin | Armand Guillaumin |
| 345 | Paolo | Uccello | Paolo Uccello |
| 341 | Auguste | Rodin | Auguste Rodin |
| 284 | Domenico | Ghirlandaio | Domenico Ghirlandaio |
| 2 | Charles | de La Fosse | Charles de La Fosse |
| ~ 187 行 ~ |

虽然从技术上讲，说如果你不知道名字，你就不知道全名是正确的，但更实用的说法是这无关紧要。在这种情况下，我们可以将缺失的名字以及其后的空格 `coalesce` 为空字符串 (`''`)：

```sql
SELECT
  id, givenname, familyname,
  coalesce(givenname||' ','') || familyname AS fullname
--  coalesce(givenname+' ','') + familyname AS fullname --  MSSQL
FROM artists;
```

现在我们为每个人都有了一个名字：

| id | givenname | familyname | fullname |
| --- | --- | --- | --- |
| 215 |   | Cuyp | Cuyp |
| 349 | Armand | Guillaumin | Armand Guillaumin |
| 345 | Paolo | Uccello | Paolo Uccello |
| 341 | Auguste | Rodin | Auguste Rodin |
| 284 | Domenico | Ghirlandaio | Domenico Ghirlandaio |
| 2 | Charles | de La Fosse | Charles de La Fosse |
| ~ 187 行 ~ |

记住以下关于 `coalesce()` 的要点：

*   仅当替代值是明显或无害的时候才使用 `coalesce()`。确定什么是明显或无害的取决于你。
*   一旦你使用了 `coalesce()`，你就无法分辨结果是真实的还是猜测的。

`NULL` 的存在一直是数据库开发人员痛苦的根源。请记住，它代表一个缺失的值，你需要做的一件事就是找出它为什么缺失以及这是否重要。

## 使用别名

当你计算值时，你生成了另一列，但 SQL 不知道该怎么称呼它。如果你在控制台运行 `SELECT` 查询，你可能会看到一些虚拟名称，如“Unnamed Column”，或者你用于计算的表达式。然而，那不是一个真正的名称，当你需要更严肃地对待这个语句时，它会因为缺乏一个合适的名称而失败。

`SELECT` 语句的结果是一个虚拟表，和真实表一样，每一列都必须有一个唯一的名称。为了给计算列一个名称，你使用 `AS` 关键字来创建一个**别名**：

```sql
SELECT
  id, givenname, familyname,
  height/2.54 AS inches
FROM customers;
```

这里，计算的别名是 `inches`。

你可以为别名使用任何*有效*的名称，甚至是另一列的名称：

```sql
SELECT
  id, givenname, familyname,
  height/2.54 AS email    --  probably a bad idea
FROM customers;
```

然而，如果你开始做这种事情，你可能会让包括你自己在内的所有人感到困惑：

| id | givenname | familyname | email |
| --- | --- | --- | --- |
| 474 | Judy | Free |   |
| 186 | Ray | Gunn | 64.48… |
| 144 | Ray | King | 69.60… |
| 179 | Ivan | Inkling | 67.04… |
| 475 | Drew | Blood | 67.32… |
| 523 | Seymour | Sights | 65.86… |
| ~ 304 行 ~ |

你可以为任何你喜欢的列设置别名，即使它不是一个计算值：

```sql
SELECT
  id,
  givenname AS firstname, familyname AS lastname,
  height/2.54 AS email    --  still probably a bad idea
FROM customers;
```

你可能为现有列设置别名的一个原因是，你认为原始的列名不适合你的目的：它们可能不够清晰，或者使用这些数据的软件期望不同的名称。


### 不使用 AS 的别名

此时的 SQL 有一个你可能认为的主要设计缺陷：`AS` 关键字是可选的，并且任何空格都会生成一个别名：

| id | firstname | lastname | email |
| --- | --- | --- | --- |
| 474 | Judy | Free |   |
| 186 | Ray | Gunn | 64.48… |
| 144 | Ray | King | 69.60… |
| 179 | Ivan | Inkling | 67.04… |
| 475 | Drew | Blood | 67.32… |
| 523 | Seymour | Sights | 65.86… |
| ~ 304 行 ~ |

```sql
--  不推荐：
SELECT
id,
givenname firstname, familyname lastname,
height/2.54 email   --  说真的，这很可能是个坏主意
FROM customers;
```

正如示例中第一个注释所示，我们不推荐这种做法。这里有两个你在 SQL 职业生涯中迟早可能会犯的错误：

```sql
--  尾随逗号：
SELECT
id,
givenname,
familyname,
FROM customers;
```

第一个错误是当你得意忘形时，在最后一列之后添加了一个多余的逗号。SQL 会直接拒绝执行，你会得到一个 `语法错误`：SQL 无法理解你的语句。

这是你将会犯的另一个错误：

```sql
--  缺失逗号：
SELECT
id
givenname,
familyname
FROM customers;
```

从技术上讲，这不是一个错误：

| givenname | familyname |
| --- | --- |
| 474 | Free |
| 186 | Gunn |
| 144 | King |
| 179 | Inkling |
| 475 | Blood |
| 523 | Sights |
| ~ 304 行 ~ |

这一次，语句会执行，但你会得到错误的结果。SQL 会将 `givenname` 解释为 `id` 的别名，因为它紧跟在空格之后。这被称为 `逻辑错误`：它能运行，但并非你的本意。

这类错误可能非常难以识别，尤其是当你有大量外观相似的数据列时。

除了非常小心之外，*没有办法* 防止这类错误。我们建议你*始终*使用 `AS` 来定义别名。这虽然不能避免错误，但可能会让缺失的逗号更显眼一些。

### 拗口的别名

你也可以用双引号将别名括起来：

```sql
SELECT
id, givenname, familyname,
height/2.54 AS "inches" --  加引号的别名
FROM customers;
```

在这种情况下，双引号是不必要的，但无疑能明确表示 `inches` 是一个别名。

同样，如果你使用的是 MariaDB/MySQL，你应该将其置于 ANSI 模式。否则，双引号将被解释为字符串。

如果不需要，SQL 为什么允许你用引号括住列名呢？在某些情况下，列名对 SQL 来说可能不清晰。例如，你可能记得有一个名为 `badtable` 的表，里面全是糟糕的列名：

| customer code | customer | order | 1st | 42 | last-date |
| --- | --- | --- | --- | --- | --- |
| 23 | Fred | 42 | 2020-01-01 | Life, … | 2020-01-31 |
| 37 | Wilma | 54 | 2020-02-01 | I think … | 2020-02-29 |

```sql
SELECT * FROM badtable;
```

以下是一些可能出现的问题：

首先，列名可能与 SQL 关键字相同，例如，如果一列名为 `order`，它将与 `ORDER BY` 混淆：

```sql
--  SQL 关键字
SELECT order    --    语法错误
FROM sales;
```

列名可能是一个数字，或以数字开头，这通常是无效的。例如：

```sql
--  数字
SELECT 1st, 42  --    解释为：1 作为 st，42
FROM events;
```

这里，你会得到 *值* `1` 和 `42`，这是合法的；然而，第一个会获得一个别名，而第二个则是匿名的。

列名可能包含空格或其他会被误解的字符。例如：

```sql
--  无效字符
--  解释为：customer 作为 code，last - date
SELECT customer code, last-date
FROM data;
```

它们都被误解了。第二个情况更糟，因为它看起来像是对两个不存在的列进行减法运算。

通常，如果列名有问题，就需要使用双引号。这有时是草率设计的结果，数据库开发者应该学会避免有问题的名称。

一些数据库管理系统有双引号的替代方案：

*   MSSQL 允许方括号（`[ … ]`）作为替代。尽管他们似乎更喜欢方括号，但这在你的代码中造成了另一个不必要的不兼容性，你应该使用双引号。
*   MySQL/MariaDB 允许所谓的反引号（`` ` … ` ``）作为替代。在 ANSI 模式下，你应该避免使用它们，而使用双引号。然而，在传统模式下，你没有其他选择。

在这里，我们将使用双引号，并且仅在需要时才使用。

## 数字计算

SQL 的核心数据类型之一是数字。数字有多种变体：

*   **整数** 数是整数。变体包括数字范围以及是否包含负数。
*   **小数**，也称为 **数值**，有预定义的小数位数。历史上，它们被称为定点数。
*   **浮点数**，也称为 **实数**，具有可变的小数位数。

存在如此多变体的原因是为了存储和处理。你可以根据精度和范围决定为每个值使用多少存储空间。整数也是最易于处理的，其次是定点数，然后是浮点小数。浮点数的精度也较低。

SQL 提供了使用 **算术运算符** 和 **数学函数** 进行计算的能力：

*   算术运算符执行你在学校早期学到的基本算术。
*   数学函数执行更复杂的操作，其中许多可能是你在学校后期学到的。

不同类型名称的命名很不幸：“decimal” 和 “numeric” 过于笼统，数学家认为前面提到的所有类型都是 “实数”。我们经常会在更宽松的意义上使用 “decimal” 一词来包括非整数类型，而 “numeric” 一词通常会在其标准意义上被使用，指代任何与数字相关的事物。

### 算术运算符

你已经见过 SQL 中的一个简单计算：

```sql
SELECT
id, givenname, familyname,
height/2.54 AS inches
FROM customers;
```

所有数据库管理系统都识别四种基本算术运算符：`+`、`-`、`*` 和 `/`。请注意，当你组合这些操作时，SQL 遵循通常的规则：

```sql
SELECT 1+2*3;   --  1+6 = 7         而不是 9
SELECT 12/2*3;  --  6*3 = 18        而不是 2
```

正如你在学校学到的：

*   在加法或减法*之前*进行乘法或除法。这称为运算符优先级。
*   *相同*优先级的运算符从左到右执行。这称为结合性。

当然，你可以随时使用括号 `( … )` 来覆盖这些规则：

```sql
SELECT (1+2)*3;     --  3*3 = 9     而不是 7
SELECT 12/(2*3);    --  12/6 = 2    而不是 18
```

你也可以包含更复杂的括号组合：

```sql
SELECT 2 * (3 + 4) + 3 * (4 + 5)    --  2*7+3*9 = 14+27 = 41
```

这里的原理是，在处理其他部分*之前*先处理括号。


### 整数

从数学上讲，整数是完整的数字，可以是正数、负数或零。你可以将任意两个整数相加、相减或相乘，都会得到另一个整数。问题在于当你尝试将两个整数相除时：有时结果*不是*另一个整数。在行业中，我们说整数在除法运算下不是**封闭**的。

在普通计算器上，它只有一个功能，所有数字都被视为小数；整数是末尾带`.0`的小数。在数据库上，其主要功能是存储和管理数据，小数处理起来更复杂，且与更容易处理的整数相区别。

这一点的实际影响体现在你运行以下语句时：

```sql
SELECT 200/7;
```

你可能得不到预期的结果。

SQLite、PostgreSQL 和 MSSQL 会给你一个 `28` 的结果，这是截短的：`28*7 = 196`。将此计算视为纯整数运算，任何余数，无论多大，都会被忽略。而 MySQL/MariaDB 和 Oracle 则会返回一个小数结果 `28.57…`。

如果你从表中提取真实数据，情况也是如此：

```sql
SELECT
id, quantity, quantity/3 AS third
FROM saleitems;
```

在 `saleitems` 表中，`quantity` 绝对是一个整数，前述的三个数据库管理系统（DBMS）会截断结果，而其他的则会返回小数。

| id | quantity | third |
| --- | --- | --- |
| 2621 | 3 | 1 |
| 5169 | 1 | 0 |
| 667 | 1 | 0 |
| 6905 | 3 | 1 |
| 886 | 1 | 0 |
| 6729 | 2 | 0 |
| ~ 共 6315 行 ~ |

如果你在末尾添加一个 `.0`，可以强制其他数据库管理系统将数字视为小数：

```sql
SELECT 200/7 AS plain, 200.0/7 as decimalised;
```

只要至少有一个数是小数形式，具体哪个数被小数化并不重要。

如果你只使用列数据进行计算，将无法在末尾添加 `.0`。相反，你需要使用 `cast()` 函数将类型从整数更改为小数。例如：

```sql
SELECT cast(200 as float)/cast(7 as float);
```

`float` 数据类型是浮点小数（它可以有任意多的小数位）。

你也可以反过来，将小数转换为整数：

```sql
SELECT cast(6.5 as int);
```

请注意，结果因数据库管理系统而异：

*   对于 PostgreSQL、MySQL/MariaDB 和 Oracle，整数将使用经典的 4/5 规则（即四舍五入）进行舍入；这里的结果将是 7。
*   对于 MSSQL 和 SQLite，小数部分将被丢弃，无论它有多大。这称为截断小数。

如果你确实想截断小数，一些数据库管理系统有一个专门的 `floor()` 函数来完成这项工作。

稍后你将看到更多关于 `cast()` 的内容。

### 余数

如果你正在处理整数除法，有时可能只想要余数。许多数据库管理系统使用 `%` 运算符来实现这一点：

```sql
SELECT 200/7, 200.0/7, 200%7;   --  非 Oracle
```

这*通常*被称为**取模**（"mod"）运算符，尽管严格来说它并不是真正的数学取模运算。区别在于如何处理负数。它实际上是一个**取余**运算符。

如果你在 Oracle 中尝试这个，会因为不支持 `%` 而收到错误。相反，你应该使用 `mod()` 函数：

```sql
--  Oracle
SELECT 200/7, 200.0/7, mod(200,7) FROM dual;
```

同样，这严格来说不是取模运算，而是取余运算。Oracle 确实有一个 `remainder()` 函数，但它既不返回真正的模数，也不返回真正的余数。

为什么取余运算有用？有许多值是周期性的，例如星期几或一年中的月份。要计算 200 天后是星期几，你可以取其 `200%7 = 4` 的余数，并将其加到今天。月份也一样：取余数 `200%12 = 8`，并将其加到当前月份。

### 额外的小数位

当除以 7 时，你会遇到小数的一个基本问题：任何除数（除了 2 或 5）都会导致无限循环小数。根据数据库管理系统，你最终可能会得到几位甚至很多位小数，但这（a）从不精确，并且（b）可能无论如何都超出了你的需要。

从数学上讲，数字 `3`、`3.0` 和 `3.00` 是相同的。然而，当这些数字用于分析时，例如在科学和统计中，小数位数被用来暗示精度：`3.0` 精确到最接近的十分位（`0.1`），但不保证百分位或更进一步的精度。六位小数暗示结果精确到百万分之一。

你可以在 `height` 计算中看到这一点：

```sql
SELECT
id, givenname, familyname,
height/2.54 as inches
FROM customers;
```

由于算术运算，结果有*许多*位小数，但没有人会声称该测量值精确到十分之一英寸以上。

通常的解决方案是将小数四舍五入到几位小数：

```sql
SELECT
id, givenname, familyname,
round(height/2.54,2) as inches
FROM customers;
```

这给出了更现实的结果：

| id | givenname | familyname | inches |
| --- | --- | --- | --- |
| 474 | Judy | Free |   |
| 186 | Ray | Gunn | 64.49 |
| 144 | Ray | King | 69.61 |
| 179 | Ivan | Inkling | 67.05 |
| 475 | Drew | Blood | 67.32 |
| 523 | Seymour | Sights | 65.87 |
| ~ 共 304 行 ~ |

`round()` 函数将第一个数四舍五入到指定的位数。如果后续数字是 5 或更大，它会正确调整最后一位：即所谓的 "`4/5`" 规则（四舍五入）。

一些数据库管理系统，如 SQL Server，可能仍然会显示额外的小数位，即使它们都是零。

接下来你将看到更多关于此函数和其他函数的内容。

### 数学函数

数学函数对数字执行更复杂的操作。SQL 标准对这些函数几乎没有规定，因此你会发现它们的可用性、行为甚至名称在不同数据库管理系统之间可能有所不同。

以下是一些数学函数的示例：

```sql
--  PostgreSQL, MariaDB/MySQL, MSSQL
SELECT
pi() AS pi,
sin(radians(45)) AS sin45,  --  三角函数使用弧度
sqrt(2) AS root2,           --  √2
log10(3) AS log3,
ln(10) AS ln10,             --  自然对数
power(4,3) AS four_cubed    --  4³
;

--  Oracle
SELECT
acos(-1) AS pi,                 --  不同
sin(45*acos(-1)/180) AS sin45,  --  不同
sqrt(2) AS root2,
log(10,3) AS log3,              --  不同
ln(10) AS ln10,
power(4,3) AS four_cubed
FROM dual;
```

如前面的代码所示，Oracle 有一组稍有不同的函数，而 SQLite 则完全没有这些函数。

另外，请注意三角函数不使用角度：那太简单了。相反，它们使用弧度，这涉及到 π 的值（大约 3.142…）。Oracle 通过没有 `pi()` 或 `radians()` 函数使这一点复杂化，因此这是使用 `acos()` 函数来模拟的。


