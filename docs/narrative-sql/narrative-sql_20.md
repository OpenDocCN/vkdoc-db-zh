# SQL 语法参考

本节总结了 SQL 的核心语法元素，包括聚合函数、分组、连接、子查询、数据修改、窗口函数以及日期时间处理。

### 聚合函数

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| 聚合函数 | 基本聚合 | `SELECT COUNT(*) FROM table_name;`<br>-- 计算行数<br><br>`SELECT COUNT(column1) FROM table_name;`<br>-- 计算非空值<br><br>`SELECT COUNT(DISTINCT column1) FROM table_name;`<br>-- 计算唯一值<br><br>`SELECT SUM(column1) FROM table_name;`<br>-- 求和<br><br>`SELECT AVG(column1) FROM table_name;`<br>-- 平均值<br><br>`SELECT MIN(column1) FROM table_name;`<br>-- 最小值<br><br>`SELECT MAX(column1) FROM table_name;`<br>-- 最大值<br><br>`SELECT STDDEV(column1) FROM table_name;`<br>-- 标准差<br><br>`SELECT VARIANCE(column1) FROM table_name;`<br>-- 方差 |

## GROUP BY

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| `GROUP BY` | 基本分组 | `SELECT column1, COUNT(*)`<br>`FROM table_name GROUP BY column1;` |
| 多个列 | `SELECT column1, column2, SUM(column3)`<br>`FROM table_name`<br>`GROUP BY column1, column2;` |
| 带 `HAVING` (对聚合数据进行过滤) | `SELECT column1, COUNT(*)`<br>`FROM table_name`<br>`GROUP BY column1`<br>`HAVING COUNT(*) > 5;` |

## 连接

表 A-16 展示了不同的 SQL 连接类型，包括 `INNER JOIN`、`LEFT JOIN`、`RIGHT JOIN`、`FULL OUTER JOIN`、`CROSS JOIN` 和 `SELF JOIN`，以及它们的基本语法和用于组合多表数据的用例。

**表 A-16：连接**

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| 基本连接 | `INNER JOIN` | `SELECT t1.column1, t2.column2`<br>`FROM table1 t1`<br>`INNER JOIN table2 t2 ON t1.id = t2.id;` |
| `LEFT JOIN` | `SELECT t1.column1, t2.column2`<br>`FROM table1 t1`<br>`LEFT JOIN table2 t2 ON t1.id = t2.id;` |
| `RIGHT JOIN` | `SELECT t1.column1, t2.column2`<br>`FROM table1 t1`<br>`RIGHT JOIN table2 t2 ON t1.id = t2.id;` |
| `FULL OUTER JOIN` | `SELECT t1.column1, t2.column2`<br>`FROM table1 t1`<br>`FULL OUTER JOIN table2 t2 ON t1.id = t2.id;` |
| `CROSS JOIN` | `SELECT t1.column1, t2.column2`<br>`FROM table1 t1`<br>`CROSS JOIN table2 t2;` |
| 自连接 | 自连接 (将表与其自身连接) | `SELECT a.column1, b.column2`<br>`FROM table_name a`<br>`JOIN table_name b ON a.id = b.parent_id;` |

## 子查询

表 A-17 说明了 SQL 中不同类型子查询，包括 `WHERE` 和 `FROM` 子句中的基本子查询、相关子查询和公共表表达式，并提供了基本和递归 CTE 的语法。

**表 A-17：子查询**

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| 基本子查询 | `WHERE` 子句中的子查询 | `SELECT * FROM table1`<br>`WHERE column1 IN (SELECT column1 FROM table2);` |
| `FROM` 子句中的子查询 | `SELECT a.column1, b.column2`<br>`FROM table1 a, (SELECT * FROM table2) b`<br>`WHERE a.id = b.id;` |
| 相关子查询 | `SELECT *`<br>`FROM table1 t1`<br>`WHERE column1 > (`<br>`  SELECT AVG(column1)`<br>`  FROM table1 t2`<br>`  WHERE t2.category = t1.category`<br>`);` |
| 公共表表达式 | 基本 CTE | `WITH cte_name AS (`<br>`  SELECT column1, column2`<br>`  FROM table_name`<br>`  WHERE condition`<br>`)<br>`SELECT * FROM cte_name;` |
| 多个 CTE | `WITH`<br>`cte1 AS (`<br>`  SELECT * FROM table1 WHERE condition1`<br>`),`<br>`cte2 AS (`<br>`  SELECT * FROM table2 WHERE condition2`<br>`)<br>`SELECT * FROM cte1 JOIN cte2 ON cte1.id = cte2.id;` |
| 递归 CTE | `WITH RECURSIVE cte_name AS (`<br>`  -- 基本情况`<br>`  SELECT * FROM table_name WHERE condition`<br>`  UNION ALL`<br>`  -- 递归情况`<br>`  SELECT t.*`<br>`  FROM table_name t`<br>`  JOIN cte_name c ON c.id = t.parent_id`<br>`)<br>`SELECT * FROM cte_name;` |

## 数据修改

表 A-18 提供了用于数据操作的基本 SQL 命令，包括用于添加数据的 `INSERT`、用于修改记录的 `UPDATE` 以及用于删除数据的 `DELETE`。它涵盖了基本操作的语法，如插入多行、更新多列以及使用子查询进行删除。

**表 A-18：数据修改**

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| `INSERT` | 基本插入 | `INSERT INTO table_name (column1, column2) VALUES (value1, value2);` |
| 多行插入 | `INSERT INTO table_name (column1, column2)`<br>`VALUES`<br>`  (value1, value2),`<br>`  (value3, value4);` |
| 从 `SELECT` 插入 | `INSERT INTO table_name (column1, column2)`<br>`SELECT column1, column2 FROM source_table WHERE condition;` |
| `UPDATE` | 基本更新 | `UPDATE table_name SET column1 = value1 WHERE condition;` |
| 更新多列 | `UPDATE table_name`<br>`SET column1 = value1, column2 = value2`<br>`WHERE condition;` |
| 使用 `JOIN` 更新 | `UPDATE table1 t1`<br>`SET t1.column1 = t2.column1`<br>`FROM table2 t2`<br>`WHERE t1.id = t2.id;` |
| `DELETE` | 基本删除 | `DELETE FROM table_name WHERE condition;` |
| 删除所有行 | `DELETE FROM table_name;` |
| 使用子查询删除 | `DELETE FROM table_name`<br>`WHERE column1 IN (SELECT column1 FROM other_table WHERE condition);` |

### 窗口函数

表 A-19 描述了 SQL 窗口函数，包括 `ROW_NUMBER()` 和 `PARTITION BY`，以及常见的窗口函数如 `SUM()`、`AVG()`、`MIN()`、`MAX()` 和 `COUNT()`。它还涵盖了排名函数，如 `RANK()`、`DENSE_RANK()`、`NTILE()`，以及用于访问前一行或后一行数据的 `LEAD()` 和 `LAG()`。这些函数用于跨分区或有序结果集进行高级数据分析和计算。

**表 A-19：窗口函数**

| 类别 | 基本语法 |
| --- | --- |
| 行号 | `SELECT`<br>`  column1,`<br>`  ROW_NUMBER() OVER (ORDER BY column2) AS row_num`<br>`FROM table_name;` |
| 按分区 | `SELECT`<br>`  column1,`<br>`  column2,`<br>`  ROW_NUMBER() OVER (PARTITION BY column1 ORDER BY column2) AS row_num`<br>`FROM table_name;` |
| 常见窗口函数 | `SELECT`<br>`  column1,`<br>`  column2,`<br>`  SUM(column2) OVER (PARTITION BY column1) AS sum,`<br>`  AVG(column2) OVER (PARTITION BY column1) AS avg,`<br>`  MIN(column2) OVER (PARTITION BY column1) AS min,`<br>`  MAX(column2) OVER (PARTITION BY column1) AS max,`<br>`  COUNT(*) OVER (PARTITION BY column1) AS count`<br>`FROM table_name;` |
| 排名函数 | `SELECT`<br>`  column1,`<br>`  column2,`<br>`  ROW_NUMBER() OVER (ORDER BY column2) AS row_num,`<br>`  RANK() OVER (ORDER BY column2) AS rank,`<br>`  DENSE_RANK() OVER (ORDER BY column2) AS dense_rank,`<br>`  NTILE(4) OVER (ORDER BY column2) AS quartile`<br>`FROM table_name;` |
| 前导和滞后函数 | `SELECT`<br>`  column1,`<br>`  column2,`<br>`  LEAD(column2) OVER (ORDER BY column1) AS next_value,`<br>`  LAG(column2) OVER (ORDER BY column1) AS prev_value`<br>`FROM table_name;` |

## 日期和时间操作

表 A-20 提供了用于处理日期和时间的 SQL 语法，涵盖检索当前日期和时间、执行日期运算、提取日期的特定组成部分以及格式化日期和时间值等操作。

**表 A-20：日期和时间操作**

| 类别 | 基本语法 |
| --- | --- |
| 当前日期和时间 | `SELECT CURRENT_DATE;`<br>-- 当前日期<br><br>`SELECT CURRENT_TIME;`<br>-- 当前时间<br><br>`SELECT CURRENT_TIMESTAMP;`<br>-- 当前日期和时间<br><br>`SELECT NOW();`<br>-- 当前时间戳 |
| 日期/时间运算 | `SELECT date_column + INTERVAL '1 day';`<br>-- 加 1 天<br><br>`SELECT date_column + INTERVAL '2 hours';`<br>-- 加 2 小时<br><br>`SELECT date_column + INTERVAL '3 months';`<br>-- 加 3 个月<br><br>`SELECT date_column - INTERVAL '1 year';`<br>-- 减 1 年 |
| 日期/时间提取 | `SELECT EXTRACT(YEAR FROM date_column);`<br>-- 获取年份<br><br>`SELECT EXTRACT(MONTH FROM date_column);`<br>-- 获取月份<br><br>`SELECT EXTRACT(DAY FROM date_column);`<br>-- 获取日<br><br>`SELECT EXTRACT(HOUR FROM timestamp_column);`<br>-- 获取小时<br><br>`SELECT EXTRACT(DOW FROM date_column);`<br>-- 星期几 (0-6) |
| 日期/时间格式化 | `SELECT TO_CHAR(date_column, 'YYYY-MM-DD');`<br>-- 格式化日期<br><br>`SELECT TO_CHAR(timestamp_column, 'HH24:MI:SS');`<br>-- 格式化时间 |


表 A-21 涵盖了 SQL 字符串操作，例如连接字符串、提取子字符串、计算字符串长度、替换文本以及查找子字符串在字符串中的位置。

## 表 A-21：字符串操作

| 类别 | 基本语法 |
| --- | --- |
| 字符串操作 | `SELECT CONCAT(column1, ' ', column2);` -- 连接<br>`SELECT column1 &#124;&#124; ' ' &#124;&#124; column2;` -- 连接运算符<br>`SELECT UPPER(column1);` -- 转为大写<br>`SELECT LOWER(column1);` -- 转为小写<br>`SELECT INITCAP(column1);` -- 首字母大写<br>`SELECT TRIM(' text ');` -- 去除两端空格<br>`SELECT LTRIM(' text');` -- 去除左侧空格<br>`SELECT RTRIM('text ');` -- 去除右侧空格<br>`SELECT SUBSTRING(column1, 1, 5);` -- 子字符串<br>`SELECT LEFT(column1, 5);` -- 左侧 n 个字符<br>`SELECT RIGHT(column1, 5);` -- 右侧 n 个字符<br>`SELECT LENGTH(column1);` -- 字符串长度<br>`SELECT REPLACE(column1, 'old', 'new');` -- 替换文本<br>`SELECT POSITION('needle' IN column1);` -- 查找位置 |

表 A-22 解释了 SQL 条件表达式，包括用于条件逻辑的 `CASE` 表达式、当两个值相等时返回 `NULL` 的 `NULLIF`、选择第一个非 `NULL` 值的 `COALESCE`，以及查找多个列中最高或最低值的 `GREATEST/LEAST`。

## 表 A-22：条件表达式

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| 条件表达式 | `CASE` 表达式 | <pre><code>SELECT<br>column1,<br>CASE<br>WHEN condition1 THEN result1<br>WHEN condition2 THEN result2<br>ELSE result3<br>END AS new_column<br>FROM table_name;</code></pre> |
|  | `NULLIF` (如果相等则返回 `NULL`) | `SELECT NULLIF(column1, 0) FROM table_name;` |
|  | `COALESCE` (返回第一个非 `NULL` 值) | `SELECT COALESCE(column1, column2, 'default') FROM table_name;` |
|  | `GREATEST/LEAST` | `SELECT GREATEST(column1, column2, column3) FROM table_name;`<br>`SELECT LEAST(column1, column2, column3) FROM table_name;` |

表 A-23 介绍了 SQL 视图，包括如何创建、管理和使用普通视图与物化视图，包含其刷新、修改、查询和删除的语法，以及检索视图元数据和在允许时执行更新。

## 表 A-23：视图

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| 创建视图 | 基本视图创建 | <pre><code>CREATE VIEW view_name AS<br>SELECT column1, column2<br>FROM table_name<br>WHERE condition;</code></pre> |
|  | 创建或替换现有视图 | <pre><code>CREATE OR REPLACE VIEW view_name AS<br>SELECT column1, column2<br>FROM table_name<br>WHERE condition;</code></pre> |
|  | 物化视图 (物理存储，需要刷新) | <pre><code>CREATE MATERIALIZED VIEW mat_view_name AS<br>SELECT column1, column2<br>FROM table_name<br>WHERE condition;</code></pre> |
|  | 带选项创建 | <pre><code>CREATE VIEW view_name WITH (security_barrier=true) AS<br>SELECT column1, column2<br>FROM table_name<br>WHERE condition;</code></pre> |
| 管理视图 | 刷新物化视图 (更新数据) | `REFRESH MATERIALIZED VIEW mat_view_name;` |
|  | 并发刷新 (不阻塞查询) | `REFRESH MATERIALIZED VIEW CONCURRENTLY mat_view_name;` |
|  | 修改视图 | `ALTER VIEW view_name RENAME TO new_name;`<br>`ALTER VIEW view_name SET SCHEMA new_schema;`<br>`ALTER VIEW view_name ALTER COLUMN column_name SET DEFAULT expression;` |
|  | 删除视图 | `DROP VIEW view_name;`<br>`DROP VIEW IF EXISTS view_name;`<br>`DROP VIEW view_name CASCADE;` -- 同时删除依赖对象 |
|  | 删除物化视图 | `DROP MATERIALIZED VIEW mat_view_name;` |
| 视图信息 | 列出当前数据库中所有视图 | `SELECT * FROM pg_views WHERE schemaname = 'public';` |
|  | 列出所有物化视图 | `SELECT * FROM pg_matviews WHERE schemaname = 'public';` |
|  | 获取视图定义 | `SELECT pg_get_viewdef('view_name', true);` |
| 使用视图 | 像查询普通表一样查询视图 | `SELECT * FROM view_name WHERE column1 = value;` |
|  | 与视图进行连接 | <pre><code>SELECT t.column1, v.column2<br>FROM table_name t<br>JOIN view_name v ON t.id = v.id;</code></pre> |
|  | 通过视图更新 (如果可能) | `UPDATE view_name SET column1 = value WHERE condition;` |

表 A-24 涵盖了 SQL 索引，详细说明了如何创建、管理和监控不同类型的索引以优化查询，以及重建、删除索引和检查索引使用情况与大小的语法。

## 表 A-24：索引

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| 创建索引 | 基本索引 | `CREATE INDEX index_name ON table_name (column_name);` |
|  | 多列索引 | `CREATE INDEX index_name ON table_name (column1, column2);` |
|  | 唯一索引 | `CREATE UNIQUE INDEX index_name ON table_name (column_name);` |
|  | 部分索引 (仅索引特定行) | `CREATE INDEX index_name ON table_name (column_name) WHERE condition;` |
|  | 表达式索引 | `CREATE INDEX index_name ON table_name (lower(column_name));` |
|  | 使用特定方法创建索引 | `CREATE INDEX index_name ON table_name USING BTREE (column_name);`<br>`CREATE INDEX index_name ON table_name USING HASH (column_name);`<br>`CREATE INDEX index_name ON table_name USING GIN (column_name);`<br>`CREATE INDEX index_name ON table_name USING GIST (column_name);` |
|  | 并发创建 (不阻塞写入) | `CREATE INDEX CONCURRENTLY index_name ON table_name (column_name);` |
| 索引类型 | B-tree (默认): 适用于等值和范围查询 | `CREATE INDEX index_name ON table_name USING BTREE (column_name);` |
|  | Hash: 仅适用于等值比较 | `CREATE INDEX index_name ON table_name USING HASH (column_name);` |
|  | GiST: 适用于几何数据和全文搜索 | `CREATE INDEX index_name ON table_name USING GIST (geom_column);` |
|  | GIN: 适用于数组和全文搜索 | `CREATE INDEX index_name ON table_name USING GIN (array_column);` |
|  | BRIN (块范围索引): 适用于具有自然顺序的大表 | `CREATE INDEX index_name ON table_name USING BRIN (timestamp_column);` |
| 管理索引 | 重建索引 | `REINDEX INDEX index_name;` |
|  | 重建表上所有索引 | `REINDEX TABLE table_name;` |
|  | 删除索引 | `DROP INDEX index_name;`<br>`DROP INDEX IF EXISTS index_name;`<br>`DROP INDEX CONCURRENTLY index_name;` |
|  | 禁用/启用索引 | `ALTER INDEX index_name SET (fastupdate = off);` |
| 索引信息 | 列出数据库中所有索引 | <pre><code>SELECT<br>indexname,<br>tablename,<br>indexdef<br>FROM pg_indexes<br>WHERE schemaname = 'public'<br>ORDER BY tablename, indexname;</code></pre> |
|  | 列出特定表上的索引 | <pre><code>SELECT<br>indexname,<br>indexdef<br>FROM pg_indexes<br>WHERE tablename = 'table_name';</code></pre> |
|  | 检查索引大小 | `SELECT pg_size_pretty(pg_relation_size(indexname::text)) as index_size FROM pg_indexes WHERE tablename = 'table_name';` |
|  | 检查索引使用情况 | <pre><code>SELECT<br>idstat.relname AS table_name,<br>index.relname AS index_name,<br>idx_scan AS times_used,<br>idx_tup_read AS tuples_read,<br>idx_tup_fetch AS tuples_fetched<br>FROM pg_stat_user_indexes idxstat<br>JOIN pg_stat_user_tables tabstat ON idxstat.relname = tabstat.relname<br>WHERE idxstat.relname = 'table_name'<br>ORDER BY idx_scan DESC;</code></pre> |

本附录作为一个简洁的 SQL 参考指南，涵盖了核心语法、常用命令及其差异。

