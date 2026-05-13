# 近似函数

还有一些函数可以给出一个十进制数的*近似*值。下面是一些示例：

```sql
SELECT
--  非 MariaDB/MySQL 或 Oracle
200/7 AS integer_result,
--  所有 DBMS
200/7.0 AS decimal_result,
--  Oracle: ceil(200/7.0)
--  SQLite: round(200/7.0 + 0.5)
ceiling(200/7.0) AS ceiling,
--  SQLite: round(200/7.0 - 0.5)
floor(200/7.0) AS floor,
--  非 MSSQL
round(200/7.0,0) AS rounded_integer,
--  所有 DBMS
round(200/7.0,2) AS rounded_decimal
--  FROM DUAL   -- Oracle
;
```

同样，不同的 DBMS 之间存在差异。前两个计算用于比较，以查看原始小数值的样子。请注意，表达式 `200/7` 可能会给出一个截断的整数，具体取决于 DBMS。

`round()` 函数将小数四舍五入到指定的位数。如果指定位数为 `0`，则四舍五入到最接近的整数。在大多数 DBMS 中，可以省略 `0`。

`ceiling()` 函数总是将小数*向上*舍入到下一个整数，无论小数部分有多小；而 `floor()` 函数总是将小数*向下*舍入到整数，无论小数部分有多大。这些函数在 SQLite 中不可用，但很容易模拟。

