# 第四章
## 创建和派生数据
将数据存储在数据库中只是故事的一半。另一半是以用户能够理解的方式使这些数据可用。当你从数据库中提取数据时，无论是在报表格式中，还是在数据维护页面上一次提取一行，每行中的值通常由于多种原因而不适合直接显示。通常，你的数据为了性能和维护原因而高度规范化——将名称的各个部分存储在不同的列中，或者使用数字代码集来引用另一张表中的完整文本描述。其他与规范化相关的场景包括将数量或价格存储在多行或同一行的多个列中，以及不在表中存储总和或其他转换结果。

由于数据库高度规范化，你通常必须执行一些转换，通过即时创建新值，使最终用户能够轻松解读报表结果。你可能希望对单列、单行中的多个列，或者对一个或多个表中的多行中的一个或多个列执行转换。本章包含涵盖所有这些场景的解决方案。


##### 4-1. 派生新列

## 问题

你不想在数据库表中存储冗余数据，但希望能够为行中的列创建总计、派生值或替代格式。

## 解决方案

在 `SELECT` 语句中，对表中的一个或多个列应用 Oracle 内置函数或创建表达式，从而在查询结果中创建一个虚拟列。例如，假设你想要汇总员工的总薪酬，将工资和佣金结合起来。

在 Oracle 默认安装中包含的 OE 用户示例表中，`ORDER_ITEMS` 表包含 `UNIT_PRICE` 和 `QUANTITY` 列，如下所示：

```
select * from order_items;

ORDER_ID LINE_ITEM_ID PRODUCT_ID UNIT_PRICE QUANTITY
--------- ------------ ---------- ---------- --------
2423     7            3290       65         33
2426     5            3252       25         29
2427     7            2522       40         22
2428     11           3173       86         28
2429     10           3165       36         67
. . .
2418     6            3150       17         37
2419     9            3167       54         81
2420     10           3171       132        47
2421     9            3155       43         185
2422     9            3167       54         39

665 rows selected
```

要在查询结果中提供行项目总计，请添加一个将单价乘以数量的表达式，如下所示：

```
select order_id, line_item_id, product_id,
       unit_price, quantity, unit_price*quantity line_total_price
from order_items;

ORDER_ID LINE_ITEM_ID PRODUCT_ID UNIT_PRICE QUANTITY LINE_TOTAL_PRICE
--------- ------------ ---------- ---------- -------- ----------------
2423     7            3290       65         33       2145
2426     5            3252       25         29       725
2427     7            2522       40         22       880
2428     11           3173       86         28       2408
2429     10           3165       36         67       2412
. . .
2418     6            3150       17         37       629
2419     9            3167       54         81       4374
2420     10           3171       132        47       6204
2421     9            3155       43         185      7955
2422     9            3167       54         39       2106

665 rows selected
```

## 工作原理

在 `SELECT` 语句（或任何 DML 语句，如 `INSERT`、`UPDATE` 或 `DELETE`）中，你可以在语句中任何列上引用任何 Oracle 内置函数（或你自己的函数）。你也可以包含聚合函数，如 `SUM()` 和 `AVG()`，只要你在 `GROUP BY` 子句中包含非聚合列。

你还可以在 `SELECT` 语句中嵌套函数。`LINE_TOTAL_PRICE` 列包含了期望的结果，但未格式化得适合向客服代表或客户显示。为了提高输出的可读性，向查询中已有的计算添加 `TO_CHAR` 函数，如下所示：

```
select order_id, line_item_id, product_id,
       unit_price, quantity,
       to_char(unit_price*quantity,'$9,999,999.99') line_total_price
from order_items;

ORDER_ID LINE_ITEM_ID PRODUCT_ID UNIT_PRICE QUANTITY LINE_TOTAL_PRICE
--------- ------------ ---------- ---------- -------- ----------------
2423     7            3290       65         33       $2,145.00
2426     5            3252       25         29       $725.00
2427     7            2522       40         22       $880.00
2428     11           3173       86         28       $2,408.00
2429     10           3165       36         67       $2,412.00
. . .
2418     6            3150       17         37       $629.00
2419     9            3167       54         81       $4,374.00
2420     10           3171       132        47       $6,204.00
2421     9            3155       43         185      $7,955.00
2422     9            3167       54         39       $2,106.00

665 rows selected
```

`TO_CHAR` 函数也可用于将日期或时间戳列转换为更易读或非默认的格式；你甚至可以使用 `TO_CHAR` 将国家字符集（`NCHAR`、`NVARCHAR2`、`NCLOB`）和 `CLOB` 的数据类型转换为数据库字符集，返回一个 `VARCHAR2`。

如果你的用户需要经常运行此查询，可以考虑创建一个数据库视图（`view`），这将使你的分析师和其他最终用户更容易访问数据。新视图将看起来像另一个表，比原始表多一列。使用 `CREATE VIEW` 语句在 `ORDER_ITEMS` 表上创建一个包含派生列的视图：

```
create view order_items_subtotal_vw as
select order_id, line_item_id, product_id,
       unit_price, quantity,
       to_char(unit_price*quantity,'$9,999,999.99') line_total_price
from order_items
;
```

要从视图中访问表的列和派生列，请使用以下语法：

```
select order_id, line_item_id, product_id,
       unit_price, quantity, line_total_price
from order_items_subtotal_vw;
```

下次你需要访问派生的 `LINE_TOTAL_PRICE` 列时，就不再需要记住计算过程或将总价格转换为货币格式的函数了！

在这种场景下使用数据库视图很有用，但它向数据字典添加了另一个对象，因此如果基表发生变化（例如当你修改或删除 `UNIT_PRICE` 或 `QUANTITY` 列时），就会创建另一个维护点。如果你使用的是 Oracle Database 11*g* 或更高版本，可以使用一个称为**虚拟列**（`virtual columns`）的功能，它本质上是在表定义本身内为派生列创建元数据。你可以在创建表时创建虚拟列，也可以像添加任何其他列一样稍后添加它。虚拟列定义本身不占用表中的空间；当你访问包含虚拟列的表时，它会动态计算。要向 `ORDER_ITEMS` 添加虚拟列 `LINE_TOTAL_PRICE`，请使用以下语法：

```
alter table order_items
add (line_total_price as (to_char(unit_price*quantity,'$9,999,999.99')));
```

虚拟列看起来与表中的任何真实列一样，是一个常规列：

```
describe order_items;

Name                                      Null?    Type
----------------------------------------- -------- -----------------
ORDER_ID                                  NOT NULL NUMBER(12)
LINE_ITEM_ID                              NOT NULL NUMBER(3)
PRODUCT_ID                                NOT NULL NUMBER(6)
UNIT_PRICE                                         NUMBER(8,2)
QUANTITY                                           NUMBER(8)
LINE_TOTAL_PRICE                                   VARCHAR2(14)

6 rows selected
```

虚拟列的数据类型由虚拟列定义中计算结果的隐式决定；`LINE_TOTAL_PRICE` 虚拟列中的表达式结果是一个最大宽度为 14 的字符串，因此虚拟列的数据类型是 `VARCHAR2(14)`。

> **提示** 要查明某列是否是虚拟列，可以查询 `*_TAB_COLS` 数据字典视图的 `VIRTUAL_COLUMN` 列。

使用虚拟列有许多优点，例如无需像上一个示例那样在表上创建视图。此外，你可以在虚拟列上创建索引并收集统计信息。以下是在前一个示例的虚拟列上创建索引的示例：

```
create index ie1_order_items on order_items(line_total_price);
```

> **注意** 对于不支持虚拟列的 Oracle 版本（Oracle Database 11*g* 之前），你可以通过在虚拟列使用的表达式上创建基于函数的索引来部分模拟虚拟列的索引功能。


