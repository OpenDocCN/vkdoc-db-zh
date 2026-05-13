# 第二部分
窗口函数

## 4. 窗口函数基础

前三章都是关于 CTE 的。从本章开始及接下来的两章，我们将转换主题，探讨窗口函数。与 CTE 类似，它们也是在 MariaDB 10.2 和 MySQL 8.0（自 8.0.2 DMR 起）中引入的。

从本质上讲，窗口函数与任何其他函数一样；它操作数据库中的数据，以一种有用的方式进行处理。主要的语法区别在于，窗口函数与一个自定义 SQL 关键字`OVER`一起使用。实际上，它们能做其他函数只能梦想的事情。本章将使你熟悉基本语法，并快速概述所有各种窗口函数。

## 什么是函数？

在通用的计算机编程术语中，函数可以被视为程序的子程序。它的逻辑与主程序隔离，并在需要时被引用或调用。

MariaDB 和 MySQL 包含许多有用的内置函数，可以执行各种操作。有像`LOWER`、`CONCAT`、`SUBSTR`等函数，帮助操作文本字符串。有像`ABS`、`LOG`、`ROUND`等函数，帮助执行各种数学运算。还有像`SUM`、`AVG`、`MAX`等聚合函数，帮助对行组执行操作。还有用于处理日期、XML、JSON、GIS 和其他数据类型的函数。

窗口函数是一个新的函数类别。当你使用常规函数时，你可以访问当前行的数据，并为结果集中的每一行产生一个结果。当你使用聚合函数时，你可以从结果集中的行组计算出一个结果。窗口函数可以同时完成这两项任务。它们在一定范围的行上计算，但也能为每一行提供结果。

### 窗口函数语法

如前所述，标识窗口函数的关键字是`OVER`。在`SELECT`语句中调用窗口函数的基本语法是：

```
函数名() OVER (
[PARTITION BY ...]
[ORDER BY ...]
[FRAME 子句]
)
...
```

在`OVER`子句内部有三个可能的可选元素：<partition_definition>、<order_definition>和<frame_definition>。我们将分别介绍它们。

`OVER`后的开括号和闭括号是强制性的，除非使用了可选的`WINDOW`子句，即使没有设置分区、排序或帧定义也是如此。有关更多信息，请参阅“WINDOW 子句语法”部分。

#### 分区定义语法

<partition_definition>部分的语法如下所示：

```
PARTITION BY {expr [, expr]...}
```

<partition_definition>部分受所有窗口函数支持。其目的是将给定函数操作的行限制在完整结果集中的特定集合内。

<expression>部分可以是任何有效的表达式，就像在传统查询的`GROUP BY`部分中使用的那样。可以指定多个表达式，用逗号分隔。

#### 排序定义语法

<order_definition>部分的语法如下所示：

```
ORDER BY {expr [ASC|DESC]} [{, expr [ASC|DESC]}...]
```

<order_definition>部分受所有窗口函数支持。其目的是设置在窗口函数运行时，函数自身看到的结果顺序，这先于完整查询中的外部`ORDER BY`子句。

<expression>部分可以是任何有效的 SQL 表达式，就像在传统查询的`ORDER BY`部分中使用的那样。结果也可以按升序（`ASC`）或降序（`DESC`）排序。可以指定多个表达式，用逗号分隔。



### 窗口函数语法

#### 帧定义语法

`<frame_definition>` 部分的语法如下所示：

```
{ROWS|RANGE}
{|}
```

`<frame_definition>` 部分并非所有窗口函数都支持。其目的是定义一个帧，然后函数使用该帧来计算结果。帧随着当前行移动。除了自身定义的边界外，帧还进一步受 `<partition_definition>` 部分的限制。

`<frame_definition>` 部分是窗口函数最强大的特性。下一章包含几个完整的示例，展示了帧移动是如何工作的。

`<frame_definition>` 有两个基本部分。首先，你指定 `ROWS` 或 `RANGE`，然后使用 `<frame_start>` 或 `<frame_between>` 部分指定帧的边界。

`<frame_start>` 部分包含以下之一：

*   `UNBOUNDED PRECEDING`
*   `<expression> PRECEDING`
*   `CURRENT ROW`

`<frame_between>` 部分稍微复杂一些。它包含：

```
BETWEEN  AND 
```

`<frame_boundary1>` 和 `<frame_boundary2>` 部分各自可以包含以下之一：

*   `<frame_start>`
*   `UNBOUNDED FOLLOWING`
*   `<expression> FOLLOWING`

如果这看起来有点复杂，不用担心。在编写 `<frame_definition>` 部分积累一些经验后，就会变得自然而然。

另一种整体查看 `<frame_definition>` 完整语法的方法是使用扩展巴科斯范式（EBNF）表示法。在 EBNF 表示法中，语法如下所示：

```
frame_definition ::= ( ROWS | RANGE ) (
CURRENT ROW
| ( UNBOUNDED | 'value_expr' ) PRECEDING
| ( BETWEEN (
UNBOUNDED PRECEDING
| CURRENT ROW
| 'value_expr' (PREDEDING|FOLLOWING)
)
AND (
UNBOUNDED FOLLOWING
| CURRENT ROW
| 'value_expr' (PRECEDING | FOLLOWING)
)
)
)
```

有一些在线工具，例如 [`http://bottlecaps.de/rr/ui`](http://bottlecaps.de/rr/ui) ，可以将 EBNF 转换为铁路图，从而更好地可视化语法。图 4-1 展示了完整的 `<frame_definition>` 语法的铁路图。

`A454556_1_En_4_Fig1_HTML.jpg`

图 4-1.
`<frame_definition>` 语法的铁路图

铁路图很有用，因为你只需沿着路径从一个元素到下一个元素，就像火车沿着铁轨行驶一样，在路径分叉时选择你想要的分支。

综上所述，以下是一些有效的 `<frame_definition>` 部分示例：

```
ROWS UNBOUNDED PRECEDING
ROWS CURRENT ROW
ROWS 3 PRECEDING
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
RANGE BETWEEN 1 PRECEDING AND CURRENT ROW
ROWS BETWEEN CURRENT ROW AND 5 PRECEDING
```

提示

根据语法定义中的规则，某些组合似乎是可能的，但实际上是不允许的。例如，使用这样的 `<frame_definition>`：

```
ROWS BETWEEN 3 FOLLOWING AND CURRENT ROW
```

将导致以下错误：

```
ERROR 4014 (HY000): Unacceptable combination of window frame bound specifications
```

这里的解决方案是交换两个边界定义的顺序，改为：

```
ROWS BETWEEN CURRENT ROW AND 3 FOLLOWING
```

有关窗口帧在实践中如何工作的简要介绍，请参阅本章后面关于 `SUM()` 函数的条目。在接下来的章节中，我们也将更深入地探讨它们。

### WINDOW 子句语法

窗口函数中 `OVER` 定义的替代方法是在 `SELECT` 语句的 `FROM` 和 `ORDER BY` 子句之间使用 `WINDOW` 子句。`WINDOW` 子句的语法如下：

```
WINDOW  AS (
[]
[]
[]
)[,  AS (...)]...
```

使用 `WINDOW` 子句时，将窗口函数中 `OVER` 后面的 `()`（包括其内部的所有内容）替换为 `<window_name>`。

这是一个假设的例子：

```
SELECT
office, time, amount,
SUM(amount)OVER window1
FROM my_table
WINDOW window1 AS (
PARTITION BY office
ORDER BY time)
ORDER BY office,time;
```

`WINDOW` 子句的目的是帮助保持 `SELECT` 语句中 `SELECT` 和 `FROM` 之间的部分更整洁、更易读。在使用多个具有相似 `OVER` 子句的窗口函数时，它也有助于避免重复。

如果定义了多个 `<window_name>` 部分，则后面的可以继承前面的。例如：

```
WINDOW
window1 AS (PARTITION BY id),
window2 AS (window1 ORDER BY office),
window3 AS (window2 ROWS UNBOUNDED PRECEDING)
```

所以 `window2` 实际上相当于：

```
(PARTITION BY id ORDER BY office)
```

而 `window3` 实际上相当于：

```
(
PARTITION BY id
ORDER BY office
ROWS UNBOUNDED PRECEDING
)
```

添加 `<window_name>` 部分的顺序必须遵循 `OVER` 子句的规则。首先是 `<partition_definition>`，然后是 `<order_definition>`，最后是 `<frame_definition>`。

同样值得注意的是，一个 `WINDOW` 子句在其定义中最多只能继承一个其他的 `WINDOW` 子句。例如，你可能逻辑上认为以下内容是有效的：

```
WINDOW
window1 AS (PARTITION BY id),
window2 AS (ORDER BY office),
window3 AS (window1 window2 ROWS UNBOUNDED PRECEDING)
```

但尝试这样做会给出错误消息。

有关使用 `WINDOW` 子句简化查询的示例，请参见第 6 章。

### 窗口函数参考

为了让你熟悉 MariaDB 和 MySQL 中所有可用的窗口函数，本章的剩余部分将逐一介绍它们及其语法。

本节旨在作为查找特定窗口函数时的参考。虽然你可能（并且可能应该）至少通读一遍，但它不一定是用来从头到尾阅读的。

有些窗口函数作为非窗口函数的任何形式都不存在，因此它们是全新的。其他函数在引入窗口函数之前已经作为聚合函数存在于 MariaDB 或 MySQL 中，例如 `AVG`。因此，虽然你以前可能见过并使用过它们，但当使用 `OVER` 子句调用时，它们具有特殊的窗口函数形式。函数按字母顺序列出，以便于查找。

窗口函数在 MariaDB 10.2 和 MySQL 8.0（自 MySQL 8.0.2 DMR 起）中可用。

#### AVG()

如果包含 `OVER` 子句，`AVG` 聚合函数可用作窗口函数。语法是：

```
AVG() OVER (
[]
[]
[]
)
```

`AVG` 函数返回由 `OVER` 子句视图中的 `<expression>` 的平均值。

#### BIT_AND()

如果包含 `OVER` 子句，`BIT_AND` 聚合函数可用作窗口函数。语法是：

```
BIT_AND() OVER (
[]
[]
[]
)
```

`BIT_AND` 函数返回由 `OVER` 子句视图中的 `<expression>` 的位的按位 `AND`。

#### BIT_OR()

如果包含 `OVER` 子句，`BIT_OR` 聚合函数可用作窗口函数。语法是：

```
BIT_OR() OVER (
[]
[]
[]
)
```

`BIT_OR` 函数返回由 `OVER` 子句视图中的 `<expression>` 的位的按位 `OR`。

#### BIT_XOR()

如果包含 `OVER` 子句，`BIT_XOR` 聚合函数可用作窗口函数。语法是：

```
BIT_XOR() OVER (
[]
[]
[]
)
```

`BIT_XOR` 函数返回由 `OVER` 子句视图中的 `<expression>` 的位的按位 `XOR`（异或）。

#### COUNT()

如果包含 `OVER` 子句，`COUNT` 聚合函数可用作窗口函数。语法是：

```
COUNT() OVER (
[]
[]
[]
)
```

`COUNT` 函数返回由 `OVER` 子句视图中的 `<expression>` 的非 `NULL` 值的计数。


### `CUME_DIST()`

`CUME_DIST()`函数的语法是：

```
CUME_DIST() OVER (
[  ]
[  ]
)
```

`CUME_DIST()`函数返回行的累积分布值。该值使用以下公式计算：

```
(行数 <= 当前行) / (总行数)
```

例如，在值`'1,2,2,3,4'`上使用`CUME_DIST()`，结果如下：

```
+-------+--------------+
| value | cume_dist    |
+-------+--------------+
|     1 | 0.2000000000 |
|     2 | 0.6000000000 |
|     2 | 0.6000000000 |
|     3 | 0.8000000000 |
|     4 | 1.0000000000 |
+-------+--------------+
```

### `DENSE_RANK()`

`DENSE_RANK()`函数的语法是：

```
DENSE_RANK() OVER (
[  ]
[  ]
)
```

`DENSE_RANK()`函数为给定行显示一个从`1`开始的编号，并遵循`<order_definition>`和`<partition_definition>`部分。相同的值会被赋予相同的结果。与`RANK()`函数不同，`DENSE_RANK()`函数在为相同值赋予相同结果后继续编号时不会跳过数字。

例如，在值`'1,2,2,3,4'`上使用`DENSE_RANK()`，结果如下：

```
+-------+------------+
| value | dense_rank |
+-------+------------+
|     1 |          1 |
|     2 |          2 |
|     2 |          2 |
|     3 |          3 |
|     4 |          4 |
+-------+------------+
```

因为`DENSE_RANK()`不跳过值，所以结果最终与`value`列相同。

### `FIRST_VALUE()`

`FIRST_VALUE()`函数的语法是：

```
FIRST_VALUE() OVER (
[  ]
[  ]
)
```

`FIRST_VALUE()`函数返回`OVER`子句视图中的第一行结果。

### `LAG()`

`LAG()`函数的语法是：

```
LAG([,][,]) OVER (
[  ]
[  ]
)
```

`LAG()`函数返回`<expression>`的值，该值偏移给定的`<offset>`量至当前行之前。如果值为`NULL`，可以选择返回`<default>`值代替。如果未指定`<offset>`，则默认为`1`。

`<default>`参数仅在 MySQL 8.0.2 或更高版本中可用；截至 MariaDB 10.2.8，它在 MariaDB 中尚不可用。

### `LAST_VALUE()`

`LAST_VALUE()`函数的语法是：

```
LAST_VALUE() OVER (
[  ]
[  ]
)
```

`LAST_VALUE()`函数返回`OVER`子句视图中的最后一行结果。

### `LEAD()`

`LEAD()`函数的语法是：

```
LEAD([,][,]) OVER (
[  ]
[  ]
)
```

`LEAD()`函数返回`<expression>`的值，该值偏移给定的`<offset>`量至当前行之后。如果值为`NULL`，可以选择返回`<default>`值代替。如果未指定`<offset>`，则默认为`1`。

`<default>`参数仅在 MySQL 8.0.2 或更高版本中可用；截至 MariaDB 10.2.8，它在 MariaDB 中尚不可用。

### `NTH_VALUE()`

`NTH_VALUE()`函数的语法是：

```
NTH_VALUE(, ) OVER (
[  ]
[  ]
)
```

`NTH_VALUE()`函数返回在`OVER`子句中定义的第 n 行结果的值。

例如，如果我们有一个包含`key`和`a`两列的表，其值如下：

```
+-----+------+
| key | a    |
+-----+------+
|   1 |    0 |
|   2 |    0 |
|   3 |    0 |
|   4 |    1 |
|   5 |    1 |
|   6 |    1 |
|   7 |    2 |
|   8 |    2 |
|   9 |    2 |
|  10 |    2 |
|  11 |    2 |
+-----+------+
```

如果我们调用`NTH_VALUE(key, a + 1) OVER (PARTITION BY a ORDER BY key)AS a1`，结果如下：

```
+-----+------+------+
| key | a    | a1   |
+-----+------+------+
|   1 |    0 |    1 |
|   2 |    0 |    1 |
|   3 |    0 |    1 |
|   4 |    1 | NULL |
|   5 |    1 |    5 |
|   6 |    1 |    5 |
|   7 |    2 | NULL |
|   8 |    2 | NULL |
|   9 |    2 |    9 |
|  10 |    2 |    9 |
|  11 |    2 |    9 |
+-----+------+------+
```

因为我们按列`a`分区，所以函数仅基于该分区中的行进行评估。因此，在第 4 行，偏移量`a + 1`等于 2，但由于函数尚未处理第 5 行，该分区的第二行没有值，因此返回的值是`NULL`。第三个分区的情况相同，只是在该情况下`<nth_expression>`等于 3，因此需要等到第三行被处理后才会返回非`NULL`结果。

### `NTILE`

`NTILE()`函数的语法是：

```
NTILE () OVER (
[  ]
[  ]
)
```

此函数返回一个整数，指示某一行所在的组。组的数量由`<ntile_expression>`部分指定，编号从 1 开始。分区中的有序行被划分为指定数量的组，每组的大小尽可能与其他组相等。

例如，在值`'1,2,2,3,4'`上使用`NTILE(2)`，结果如下：

```
+-------+----------+
| value | ntile(2) |
+-------+----------+
|     1 |        1 |
|     2 |        1 |
|     2 |        1 |
|     3 |        2 |
|     4 |        2 |
+-------+----------+
```

在相同值上使用`NTILE(3)`的结果如下：

```
+-------+----------+
| value | ntile(3) |
+-------+----------+
|     1 |        1 |
|     2 |        1 |
|     2 |        2 |
|     3 |        2 |
|     4 |        3 |
+-------+----------+
```

### `PERCENT_RANK()`

`PERCENT_RANK()`函数的语法是：

```
PERCENT_RANK() OVER (
[  ]
[  ]
)
```

`PERCENT_RANK()`函数返回给定行的相对百分比排名。用于计算百分比排名的公式是：

```
(rank - 1) / (窗口或分区中的行数 - 1)
```

例如，在值`'1,2,2,3,4'`上使用`PERCENT_RANK()`，结果如下：

```
+-------+--------------+
| value | percent_rank |
+-------+--------------+
|     1 | 0.0000000000 |
|     2 | 0.2500000000 |
|     2 | 0.2500000000 |
|     3 | 0.7500000000 |
|     4 | 1.0000000000 |
+-------+--------------+
```

### `RANK()`

`RANK()`函数的语法是：

```
RANK() OVER (
[  ]
[  ]
)
```

`RANK()`函数为给定行显示一个从`1`开始的编号，并遵循`<order_definition>`和`<partition_definition>`部分。相同的值会被赋予相同的结果，当在赋予相同结果后恢复编号时，会跳过值，从下一个非相同结果开始编号。

例如，在值`'1,2,2,3,4'`上使用`RANK()`，结果如下：

```
+-------+------+
| value | rank |
+-------+------+
|     1 |    1 |
|     2 |    2 |
|     2 |    2 |
|     3 |    4 |
|     4 |    5 |
+-------+------+
```

因为`RANK()`跳过值，所以在两行值=2 之后恢复排名时，结果跳过了第三个排名，直接到第四个。

### `ROW_NUMBER()`

`ROW_NUMBER()`函数的语法是：

```
ROW_NUMBER() OVER (
[  ]
[  ]
)
```

`ROW_NUMBER()`函数类似于`RANK()`和`DENSE_RANK()`函数，但后两者会根据`ORDER BY`为匹配行分配相同的编号，而`ROW_NUMBER()`函数总是为每一行递增计数。

例如，在值`'1,2,2,3,4'`上使用`ROW_NUMBER()`，结果如下：

```
+-------+------------+
| value | row_number |
+-------+------------+
|     1 |          1 |
|     2 |          2 |
|     2 |          3 |
|     3 |          4 |
|     4 |          5 |
+-------+------------+
```


#### SUM()

`SUM` 聚合函数在包含 `OVER` 子句时可以用作窗口函数。其语法如下：

```
SUM() OVER (
[]
[]
[]
)
```

`SUM` 函数返回 `OVER` 子句所指定窗口中 `<expression>` 值的总和。

以下是一个窗口函数（如 `SUM`）如何与 `<frame_definition>` 部分配合使用的简单示例。假设有一个表 `my_table`，其中包含一个值为 `1,2,2,3,4` 的列 `value`，我们可以使用 `SUM` 函数轻松地将当前行与前两行相加（如果有的话），如下所示：

```
SELECT value,
SUM(value) OVER (
ORDER BY value
ROWS
BETWEEN 2 PRECEDING AND CURRENT ROW
) AS sum
FROM my_table
ORDER BY value;
```

上述查询的结果为：

```
+-------+------+
| value | sum  |
+-------+------+
|     1 | 1    |
|     2 | 3    |
|     2 | 5    |
|     3 | 7    |
|     4 | 9    |
+-------+------+
```

让我们逐行分析结果。

第 1 行，值为`1`，没有前序行，因此`SUM`函数查看的窗口只包含结果的第一行，即`1`。

第 2 行，值为`2`，有一个前序行，因此`SUM`函数查看的窗口包含第 1 行和第 2 行，即`(1+2)`，等于`3`。

第 3 行，值为`2`，有两个前序行，因此`SUM`函数查看的窗口包含结果的第 1、2、3 行，即`1+2+2`，等于`5`。

第 4 行，值为`3`，有三个前序行，但窗口只包含当前行和前两行，因此`SUM`函数查看的行包含第 2、3、4 行，即`(2+2+3)`，等于`7`。

第 5 行的计算方式与第 4 行相同，但相加的值是`(2+3+4)`，等于`9`。

### 小结

在本章中，我们介绍了窗口函数的基本语法，将其分解为组成部分并分别进行了讲解。我们还介绍了所有可用的窗口函数，描述了每个函数及其用途，并在适当的情况下通过简单示例说明了它们的工作原理。我们将在后续章节中介绍更多示例，包括演示窗口及其使用方法。

## 5. 识别窗口函数的应用场景

上一章概述了什么是窗口函数，并详细介绍了其语法。是时候将这些知识付诸实践了。本章将通过一些简单而实用的示例来扩展，说明窗口函数擅长解决的问题类型。我们将涵盖组织结果、维护运行总计和排名结果。

**注意**

如果您使用的是 MySQL 8.0，版本必须至少为 8.0.2。此版本引入了窗口函数。之前的 MySQL 版本，包括 MySQL 8.0.0 和 8.0.1，不包含窗口函数。

所有 MariaDB 10.2 及更高版本都包含窗口函数。

### 分区和排序结果

数据库最重要的目的之一是组织我们周围堆积如山的数据。`OVER` 子句的 `<partition_definition>` 和 `<order_definition>` 部分正是为了帮助我们做到这一点。我们在上一章介绍了它们的语法，但在掌握窗口函数的过程中，对这些部分如何工作的实际演示可能比枯燥的语法图更有用。

使用第 1 章中的 `employees` 表，这是一个 `ROW_NUMBER` 窗口函数的简单示例。首先，我们调用函数时不包含 `OVER` 子句中的任何可选部分：

```
SELECT
ROW_NUMBER() OVER() AS rnum,
name, title, office
FROM employees
WHERE office='Cleveland' OR office='Memphis'
ORDER BY title;
```

结果有点出乎意料：

```
+------+------------------+-------------+-----------+
| rnum | name             | title       | office    |
+------+------------------+-------------+-----------+
|    1 | Eileen Morrow    | dba         | Cleveland |
|    3 | Douglas Williams | dba         | Memphis   |
|    8 | Carol Monreal    | dba         | Cleveland |
|   14 | Elva Garcia      | dba         | Memphis   |
|    5 | Rosemary Bowers  | manager     | Cleveland |
|   12 | Richard Delgado  | manager     | Memphis   |
|    2 | Julius Ramos     | salesperson | Cleveland |
|    7 | Tammy Castro     | salesperson | Memphis   |
|    9 | Joyce Beck       | salesperson | Memphis   |
|   10 | Alonzo Page      | salesperson | Cleveland |
|   11 | Tina Jefferson   | salesperson | Cleveland |
|   13 | Leo Gutierrez    | salesperson | Cleveland |
|   15 | Joann Smith      | salesperson | Memphis   |
|    4 | Janet Edwards    | support     | Memphis   |
|    6 | Louise Lewis     | support     | Cleveland |
+------+------------------+-------------+-----------+
```

查看这个结果，你可能会困惑为什么 `rnum` 列是乱序的。事实上，你的结果可能有完全不同的 `rnum` 列。它可能按顺序从 1 到 15 排列，也可能有完全不同的顺序。这是怎么回事？`rnum` 列乱序是因为窗口函数是在 `SELECT` 语句的其他部分（在我们的例子中是 `name`, `title`, `office` 列）以及任何 `WHERE`、`HAVING` 或 `GROUP BY` 子句被获取之后，但在最终的 `ORDER BY title` 子句之前计算的。事实上，如果在 `OVER` 子句中没有 `<order_definition>` 部分，窗口函数不保证任何特定的顺序。

理想情况下，我们希望 `rnum` 列始终与 `ORDER BY title` 子句的最终输出匹配。为确保这一点，我们在 `OVER` 子句中添加一个 `<order_definition>` 部分，如下所示：

```
SELECT
ROW_NUMBER()
OVER (
ORDER BY title
) AS rnum,
name, title, office
FROM employees
WHERE office='Cleveland' OR office='Memphis'
ORDER BY title;
```

现在，`rnum` 列和最终输出都保证按 `title` 列的内容排序，因此结果将始终在逻辑上合理：



```
+------+------------------+-------------+-----------+
| 编号 | 姓名             | 职位        | 办公室    |
+------+------------------+-------------+-----------+
|    1 | Eileen Morrow    | dba         | Cleveland |
|    2 | Douglas Williams | dba         | Memphis   |
|    3 | Carol Monreal    | dba         | Cleveland |
|    4 | Elva Garcia      | dba         | Memphis   |
|    5 | Rosemary Bowers  | manager     | Cleveland |
|    6 | Richard Delgado  | manager     | Memphis   |
|    7 | Julius Ramos     | salesperson | Cleveland |
|    8 | Tammy Castro     | salesperson | Memphis   |
|    9 | Joyce Beck       | salesperson | Memphis   |
|   10 | Alonzo Page      | salesperson | Cleveland |
|   11 | Tina Jefferson   | salesperson | Cleveland |
|   12 | Leo Gutierrez    | salesperson | Cleveland |
|   13 | Joann Smith      | salesperson | Memphis   |
|   14 | Janet Edwards    | support     | Memphis   |
|   15 | Louise Lewis     | support     | Cleveland |
+------+------------------+-------------+-----------+
```

克利夫兰和孟菲斯办公室之间有 15 名员工。如果我们想为不同办公室的员工分别编号，该怎么办？

这就是 `<partition_definition>` 部分的目的。它允许我们对结果进行分组。在 `OVER` 子句中加入 `<partition_definition>` 部分后，`ROW_NUMBER` 函数将独立地对每个分组执行操作。因此，我们会在 `OVER` 子句中添加 `PARTITION BY office`，像这样：

```sql
SELECT
ROW_NUMBER() OVER (
PARTITION BY office
ORDER BY title
) AS rnum,
name, title, office
FROM employees
WHERE office='Cleveland' OR office='Memphis'
ORDER BY title;
```

不幸的是，输出结果并不像我们预想的那么有用：

```
+------+------------------+-------------+-----------+
| 编号 | 姓名             | 职位        | 办公室    |
+------+------------------+-------------+-----------+
|    1 | Eileen Morrow    | dba         | Cleveland |
|    1 | Douglas Williams | dba         | Memphis   |
|    2 | Carol Monreal    | dba         | Cleveland |
|    2 | Elva Garcia      | dba         | Memphis   |
|    3 | Rosemary Bowers  | manager     | Cleveland |
|    3 | Richard Delgado  | manager     | Memphis   |
|    4 | Julius Ramos     | salesperson | Cleveland |
|    4 | Tammy Castro     | salesperson | Memphis   |
|    5 | Joyce Beck       | salesperson | Memphis   |
|    5 | Alonzo Page      | salesperson | Cleveland |
|    6 | Tina Jefferson   | salesperson | Cleveland |
|    7 | Leo Gutierrez    | salesperson | Cleveland |
|    6 | Joann Smith      | salesperson | Memphis   |
|    7 | Janet Edwards    | support     | Memphis   |
|    8 | Louise Lewis     | support     | Cleveland |
+------+------------------+-------------+-----------+
```

和本章的其他结果一样，您的结果可能与此略有不同。但此处的 `<partition_definition>` 部分工作得非常完美。结果中第一行克利夫兰员工和第一行孟菲斯员工都被赋予了行号 `1`。第二行是 `2`，依此类推。结果后面部分有点混乱，因为克利夫兰的销售人员比孟菲斯多，但所有正确的行号都在那里。我们只需要调整输出使其更易理解。

这再次清楚地表明，**窗口函数不保证排序**。在我们的例子中，解决方案是直接在最终 `ORDER BY` 子句中的 `title` 之前添加 `office` 列，而不是在 `OVER` 子句中添加，如下所示：

```sql
SELECT
ROW_NUMBER() OVER (
PARTITION BY office
ORDER BY title
) AS rnum,
name, title, office
FROM employees
WHERE office='Cleveland' OR office='Memphis'
ORDER BY office, title;
```

现在的输出如下所示：

```
+------+------------------+-------------+-----------+
| 编号 | 姓名             | 职位        | 办公室    |
+------+------------------+-------------+-----------+
|    1 | Eileen Morrow    | dba         | Cleveland |
|    2 | Carol Monreal    | dba         | Cleveland |
|    3 | Rosemary Bowers  | manager     | Cleveland |
|    4 | Julius Ramos     | salesperson | Cleveland |
|    5 | Alonzo Page      | salesperson | Cleveland |
|    6 | Tina Jefferson   | salesperson | Cleveland |
|    7 | Leo Gutierrez    | salesperson | Cleveland |
|    8 | Louise Lewis     | support     | Cleveland |
|    1 | Douglas Williams | dba         | Memphis   |
|    2 | Elva Garcia      | dba         | Memphis   |
|    3 | Richard Delgado  | manager     | Memphis   |
|    4 | Tammy Castro     | salesperson | Memphis   |
|    5 | Joyce Beck       | salesperson | Memphis   |
|    6 | Joann Smith      | salesperson | Memphis   |
|    7 | Janet Edwards    | support     | Memphis   |
+------+------------------+-------------+-----------+
```

成功了！每个办公室的员工都独立编号，并且编号顺序正确。


### 维护运行总计

另一个常见的数据库任务是维护某个值的**运行总计**。这可以是账户余额、一段时间内售出的物品数量，或其他各种数值。

我们可以使用第 2 章中用过的 `commissions` 表来演示这个概念。该表记录了我们虚构公司销售人员的佣金。它记录了销售员 ID、佣金 ID、佣金金额以及佣金日期。

通过以下查询可以查看一个简短的样本数据：
```sql
SELECT
commission_date AS date,
salesperson_id as sp,
commission_id as id,
commission_amount as amount
FROM commissions
ORDER BY sp,date;
```

完整的输出非常长，但以下是前 10 行输出：
```
+------------+----+-------+--------+
| date       | sp | id    | amount |
+------------+----+-------+--------+
| 2016-01-21 |  3 | 15165 | 429.50 |
| 2016-02-09 |  3 | 15231 | 142.37 |
| 2016-02-12 |  3 | 15253 | 184.74 |
| 2016-03-22 |  3 | 15428 | 169.62 |
| 2016-04-01 |  3 | 15476 | 363.53 |
| 2016-05-10 |  3 | 15644 | 358.49 |
| 2016-07-11 |  3 | 15901 | 149.64 |
| 2016-11-25 |  3 | 16465 | 452.04 |
| 2017-01-25 |  3 | 16726 | 145.68 |
| 2017-03-10 |  3 | 16927 | 216.16 |
...
+------------+----+-------+--------+
```

要创建一个运行的佣金总计，我们需要获取匹配给定销售员的第一行金额，并将其输出在一个列中。我们将这个列称为 `total`。然后，对于每一后续行，我们将前一个 `total` 加到当前行的 `commission_amount` 上，作为新的总计，依此类推，直到遇到一个新的 `salesperson_id`，然后我们重新开始这个过程。至少，这是我思考这个问题的方式。实际上，使用传统的 SQL 来解决这个问题有所不同。例如，一种传统的 SQL 解决方法是使用自连接和 `SUM` 函数，如下所示：
```sql
SELECT
commission_date AS date, salesperson_id as sp,
commission_id as id, commission_amount as amount,
(SELECT SUM(commission_amount)
FROM commissions AS c2
WHERE c2.salesperson_id = c1.salesperson_id AND
c2.commission_date <= c1.commission_date) AS total
FROM commissions AS c1
ORDER BY sp,date;
```

在我们的主查询中，有一个子查询，它查找销售员匹配且日期小于等于当前行日期的所有行。对于那些符合条件的行，它将它们全部相加，并将结果输出到 `total` 列中。因此，对于日期戳最早的行，它唯一匹配的行就是它自己，所以对于那一行，`total` 列与 `commission_amount` 列相同。对于日期戳第二早的行，匹配的行将是最老的行和它自己，所以这些行被求和到 `total` 列中。这个过程一直持续到所有匹配给定 `salesperson_id` 的行都被获取，然后针对下一个销售员重新开始这个过程。

结果有几千行长，但这里显示前几行：
```
+------------+-----+-------+--------+----------+
| date       | sp  | id    | amount | total    |
+------------+-----+-------+--------+----------+
| 2016-01-21 |   3 | 15165 | 429.50 | 429.50   |
| 2016-02-09 |   3 | 15231 | 142.37 | 571.87   |
| 2016-02-12 |   3 | 15253 | 184.74 | 756.61   |
| 2016-03-22 |   3 | 15428 | 169.62 | 926.23   |
| 2016-04-01 |   3 | 15476 | 363.53 | 1289.76  |
| 2016-05-10 |   3 | 15644 | 358.49 | 1648.25  |
| 2016-07-11 |   3 | 15901 | 149.64 | 1797.89  |
| 2016-11-25 |   3 | 16465 | 452.04 | 2249.93  |
| 2017-01-25 |   3 | 16726 | 145.68 | 2395.61  |
| 2017-03-10 |   3 | 16927 | 216.16 | 2611.77  |
| 2017-04-05 |   3 | 17046 | 277.36 | 2889.13  |
| 2017-04-11 |   3 | 17072 | 151.36 | 3040.49  |
| 2017-05-29 |   3 | 17272 | 368.20 | 3408.69  |
...
+------------+-----+-------+--------+----------+
```

总而言之，它有效，甚至可以说效果很好。很容易确认 `total` 列正在准确地维护一个运行总计，并且当 `salesperson_id` 变化时，计数会重新开始。

但我们获取总计的过程是**笨拙**的。如果这是一个纸质账本，我们不会不断地重新添加之前的所有内容；我们只会将新值加到旧总计上。

此外，我们查询所做的所有加法和重新加法意味着我们不断地出去获取新的行组来求和。所有这些获取和重新获取都需要时间。索引可以提供帮助，尤其是在大表上，但使用像这样的子查询似乎不是解决我们最初显示运行总计任务的正确方法。

**窗口函数**可以更好地完成这项工作，而且它以一种更**自然的方式**来做。首先，就像我们之前为员工编号的窗口函数示例一样，我们需要在 `OVER` 子句中同时使用 `<partition_definition>` 和 `<order_definition>` 部分。我们将 `PARTITION BY salesperson_id` 并 `ORDER BY commission_date` 来匹配查询的最终 `ORDER BY` 子句。

然后，我们需要告诉窗口函数为我们的 `total` 列将哪些行加在一起。聚合窗口函数（如 `SUM`）可以使用移动窗口框架来快速识别需要处理的数据。为此，我们将使用一个从结果集开始（即 `UNBOUNDED PRECEDING`）到 `CURRENT ROW` 的 `<frame_definition>`，所以我们完整的 `<frame_definition>` 部分将是：
```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

为了将所有这些整合到位，我们采用之前的查询，并将子查询部分替换为我们的 `SUM` 窗口函数，如下所示：
```sql
SELECT
commission_date AS date, salesperson_id as sp,
commission_id as id, commission_amount as amount,
SUM(commission_amount)
OVER (
PARTITION BY salesperson_id
ORDER BY commission_date
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
) AS total
FROM commissions AS c1
ORDER BY sp,date;
```

此查询的结果与我们看到的子查询版本相同，因此无需再次显示。

当这个查询运行时，在第一行，窗口框架只有第一行。随着查询继续，框架会扩展，并且与子查询不同，它不需要重新获取任何数据；它只查看已经存在的数据。然后，当框架跨越我们的分区边界时，它会重置为仅当前行并再次开始增长。与子查询发生的成百上千次往返数据库获取信息不同，窗口函数只需要进行一次遍历。

这种性能差异非常显著，甚至在相对较小的 `commissions` 表上也能看出来。在我的笔记本电脑上，处理表中全部的 3000 多行，运行自连接版本的查询需要 `2.56` 秒。与此形成对比的是，窗口函数版本**瞬间**完成。两秒半不算多，但随着表大小的增加，窗口函数的速度优势会不断增长。一位 MariaDB 工程师在类似但大得多的表上使用类似的查询进行了一些测试。表 5-1 显示了结果。所有时间均以秒为单位。

**表 5-1. 窗口函数与自连接**

| 表中的行数 | 自连接 | 带索引的自连接 | 窗口函数 |
| --- | --- | --- | --- |
| 10,000 | 0.29 | 0.01 | 0.02 |
| 100,000 | 2.91 | 0.09 | 0.16 |
| 1,000,000 | 29.10 | 2.86 | 3.04 |
| 10,000,000 | 346.30 | 90.97 | 43.17 |
| 100,000,000 | 4357.20 | 813.20 | 514.24 |

在没有表索引的情况下，窗口函数版本的查询总是比自连接版本快一个数量级。在表有索引的情况下，自连接版本的查询在一段时间内能跟上窗口函数版本的速度，但在表行数超过一百万后，窗口函数开始领先，速度几乎是自连接的两倍。

注意

