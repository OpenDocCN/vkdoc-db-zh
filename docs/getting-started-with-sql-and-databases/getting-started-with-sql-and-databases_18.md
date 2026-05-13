# 格式化函数

格式化函数改变值的*外观*。与近似函数和其他数学函数不同，格式化函数的结果不是另一个数字，而是一个字符串；这是改变数字显示方式的唯一途径。

同样，不同的 DBMS 有截然不同的函数。

例如，以下是一些将数字格式化为带货币符号和千位分隔符的货币格式的方法。在这个例子中，我们为美元和可能的欧元进行格式化：

```sql
--  PostgreSQL, Oracle
--  当前区域设置
SELECT to_char(total,'FML999G999G999D00')
FROM sales;
--  手动区域设置
SELECT to_char(total,'FM$999,999,999.00')
FROM sales;
--  MariaDB/MySQL
SET SESSION sql_mode = 'ANSI';
--  当前区域设置
SELECT '$'||format(total,2)
FROM sales;
--  手动区域设置
SELECT '€'||format(total,2,'de_DE')
FROM sales;
--  MSSQL
--  当前区域设置
SELECT format(total,'c')
FROM sales;
--  特定区域设置
SELECT format(total,'c','nl-NL'))
FROM sales;
--  手动
SELECT format(total,'€###,###,###.00','de-de')
FROM sales;
--  SQLite
SELECT printf('$%,d.%02d',total,round(total*100)%100)
FROM sales;
```

在所有情况下，我们都试图使用本地货币或特定替代方案来格式化数字。有时，这意味着需要自己添加货币符号。

注意

-   PostgreSQL 和 Oracle 都有一个灵活的 `to_char()` 函数，也可用于格式化日期。
-   MariaDB/MySQL 使用 `format()` 函数，该函数添加千位分隔符和小数位；你还可以让它针对不同的区域设置进行调整。
-   MSSQL 有自己的 `format()` 函数，其格式代码更直观；它也能适应区域设置，并可用于格式化日期。
-   SQLite 只有一个通用的 `printf()` 函数，程序员可能更熟悉；SQLite 假定你将在宿主应用程序（如 PHP 或嵌入 SQLite 的任何地方）中格式化数据。

请注意，如果你确实对一个数字使用了格式化函数，*它就不再是数字了*！如果你只是看看它，那没关系。但是，如果你计划进行进一步的计算，或者对结果进行排序，那么格式化的数字很可能会适得其反。

归根结底，在 SQL 中你可能不会做太多格式化。SQL 的主要目的是*获取*数据并为下一步做准备。格式化通常是最后一步，并且经常在其他软件中完成。

