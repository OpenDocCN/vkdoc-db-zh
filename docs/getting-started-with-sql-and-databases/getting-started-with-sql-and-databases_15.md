# 获取随机行

如果你想随机获取一行或多行，例如在测试应用程序或对数据抽样时，你可以在随机排序后限制结果。为了获得随机排序，你将需要一个随机化函数，这些函数因 DBMS 而异：

```sql
--  PostgreSQL, SQLite
SELECT * FROM customers
ORDER BY random();
--  MySQL / MariaDB
SELECT * FROM customers
ORDER BY rand();
--  Oracle
SELECT * FROM customers
ORDER BY dbms_random.value;
--  MSSQL
SELECT * FROM customers
ORDER BY newid();
```

其思想是，该函数为每一行生成一个随机值，用于对数据进行排序。

MSSQL 确实有一个`random()`函数，但它与其他的不同：它只生成一次随机数并将其用于整个表。

你可以将其与`OFFSET 0 FETCH …`（或`LIMIT … OFFSET 0`或`TOP`）结合使用，以获取有限数量的随机行。



## 非字母顺序的字符串排序

如前所述，字符串按字母顺序排序。问题在于，在现实世界中，大多数事物并非按字母顺序排列：一周中的日子、彩虹的颜色、元素周期表中的元素以及铁路线上的车站，都有其固有的顺序，按字母排序只会让人困扰。

SQL 本身没有非字母顺序排序字符串的内在方法。有多种变通方法，包括使用一个单独的表来存储首选顺序的值。然而，你也可以通过创建一个包含按你首选顺序排列的值的字符串，并定位每个值在该字符串中的位置来实现相同的效果。

例如，`sorting` 表有一列 `numbername`，它是以文本形式写出的数字。显然，按字母排序完全没用。要按值排序：

*   创建一个按顺序包含值的字符串：`'One,Two,Three,Four,Five,Six,Seven,Eight,Nine'`
*   使用一个函数来定位你的值在该字符串中的位置。

该函数在不同数据库管理系统（DBMS）中会有所不同。以下是各种 DBMS 的 SQL 示例：

```sql
--  MySQL/MariaDB, SQLite, Oracle:
--  INSTR('values',value)
SELECT *
FROM sorting
ORDER BY
INSTR('One,Two,Three,Four,Five,Six,Seven,Eight,Nine',
numbername);
--  PostgreSQL:
--  POSITION(value IN 'values')
SELECT *
FROM sorting
ORDER BY
POSITION(numbername IN
'One,Two,Three,Four,Five,Six,Seven,Eight,Nine');
--  MSSQL:
--  CHARINDEX(value, 'values')
SELECT *
FROM sorting
ORDER BY
CHARINDEX(numbername,
'One,Two,Three,Four,Five,Six,Seven,Eight,Nine');
```

现在这将按正确的顺序排序：

| id | numbername | … |
| --- | --- | --- |
| 7 | One | … |
| 5 | Two | … |
| 3 | Three | … |
| 2 | Four | … |
| 8 | Five | … |
| 1 | Six | … |
| 6 | Seven | … |
| 4 | Eight | … |

注意

*   字符串中的逗号纯粹是为了可读性。你可以在值之间使用空格、连字符、管道符或任何其他字符。你也可以不加任何分隔符地连接值。
*   在此示例中，值的大小写与字符串兼容。如果值的大小写混合，你可能需要使用 `lower` 函数，并准备一个小写形式的值字符串。

## 特殊字符串

通常，按字母顺序排序是一种约定俗成的做法，但实际的排序顺序没有意义。例如，如果你按星期几、月份名称或数字名称排序，结果不会有任何实际的顺序。

然而，有时你只能使用字符串，但仍需要更有意义地对列进行排序。前面你看到可以使用 `cast()` 来重新解释字符串，但这里有一些准备字符串本身的想法：

*   数字字符串可以进行 `补零`。这意味着在数字开头添加零以将其填充到固定长度。例如，`'1234'`、`'0056'` 和 `'0789'` 将被正确排序。
*   日期字符串可以是 ISO 8601 格式，其组成部分从大到小排列。该格式的一个特点是，按字母排序将得到正确的顺序。

如果你的字符串采用合适的自上而下的格式，那么结果确实会有意义。

## 总结

除非你使用 `ORDER BY` 子句指定顺序，否则 SQL 不保证结果的顺序。这是设计使然，因为 SQL 关注的是数据集，而数据集没有隐含的顺序。

请注意，对数据进行排序被高估了。排序最常见的原因是帮助查找某些内容，而 SQL 已经做到了这一点。特别是，字母顺序被高估了，因为按字母顺序相邻的项目在其他意义上很少相邻。

然而，有时数据在排序后会显得更有条理，并且在你不得不浏览数据的显示版本而无法使用搜索工具时，排序可能会有所帮助。

### 使用 ORDER BY 排序

对表进行排序使用 `ORDER BY` 子句：

*   排序不会改变实际的表，只会改变当前查询结果的顺序。
*   你可以使用原始列或计算值进行排序。
*   你可以使用多个列进行排序，这将有效地对行进行分组；列的顺序是任意的，但会影响分组的效果。
*   每个单独的排序列都可以通过 `DESC` 子句限定，该子句将反转顺序。还有 `ASC` 子句，它不改变任何东西，因为这是默认值。
*   不同的 DBMS 对于放置已排序的 `NULL` 值有自己的处理方式，但它们都会被分组在开头或结尾。
*   数据类型会影响排序顺序。
*   一些 DBMS 会分别对大小写不同的值进行排序。

```sql
SELECT columns
FROM table
ORDER BY …;
```

### 限制结果数量

`SELECT` 语句也可以包含对行数的限制。这个功能非正式地存在了很长时间，但现在已成为一个官方功能。

许多 DBMS 仍然提供它们专有的非正式限制子句。有些现在也提供官方版本。

### 排序字符串

按字母排序大体上是没有意义的。然而，有一些技术可以按更有意义的顺序对字符串进行排序。

### 后续内容

到目前为止，我们主要处理的是原始表数据。在下一章中，我们将更仔细地研究如何根据原始值重新计算值。

脚注 1



