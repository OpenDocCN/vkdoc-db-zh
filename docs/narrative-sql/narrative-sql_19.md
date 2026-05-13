# 附录 A：SQL 语法参考指南

本附录旨在为使用 SQL 进行数据分析的读者提供快速参考指南。它侧重于核心元素，如基本 SQL 语法、关键 SQL 关键字和标准命令结构。本指南不深入探讨详尽的理论细节，而是重点介绍在查询、操作和管理数据时常用的实用语法模式。


## SQL 术语

每当涉及 SQL 时，必须区分构成其语言结构和功能的几个核心组件。最高层级是语句（Statements），它们是在数据库上执行操作的完整指令。这些指令包括用于检索数据的 `SELECT`、用于添加数据的 `INSERT`、用于修改数据的 `UPDATE`、用于删除数据的 `DELETE` 以及用于定义 SQL 结构的 `CREATE TABLE`。

在这些语句内部是子句（Clauses），它们作为构建块来细化语句的意图。诸如 `FROM`、`WHERE`、`GROUP BY`、`ORDER BY` 和 `HAVING` 之类的子句指定了数据源、过滤条件和分组逻辑。操作（Operations）指应用于数据的特定动作，包括算术运算（+、-）、比较运算（=、<、>）和逻辑运算（`AND`、`OR`、`NOT`）。SQL 还包含函数（Functions），它们是用于计算和转换的预定义例程——例如 `SUM()`、`AVG()`、`COUNT()`、`MAX()` 以及像 `UPPER()` 或 `NOW()` 这样的字符串和日期函数。这些组件通常组合在表达式（Expressions）中，表达式结合值、运算符和函数以产生单一结果。术语 *命令*（*command*）有时与语句（statement）互换使用，尤其是在与数据库系统进行交互的命令行界面中。表 A-1 展示了这些术语。

表 A-1: SQL 术语

| 组件 | 描述 | 示例 |
| --- | --- | --- |
| 语句 | 在数据库上执行操作的完整指令。 | `SELECT` 检索数据 |
| | | `INSERT` 添加新数据 |
| | | `UPDATE` 修改现有数据 |
| | | `DELETE` 删除数据 |
| | | `CREATE TABLE` 定义新表 |
| 子句 | SQL 语句的组成部分，用于指定数据源、条件或操作。 | `FROM` 指定源表 |
| | | `WHERE` 根据条件过滤行 |
| | | `GROUP BY` 对具有相似值的行进行分组 |
| | | `ORDER BY` 对结果集排序 |
| | | `HAVING` 过滤分组后的结果 |
| 操作 | 应用于表达式中值的动作。 | 算术：+、-、*、/ |
| | | 比较：=、<、>、!= |
| | | 逻辑：`AND`、`OR` 和 `NOT` |
| 函数 | 用于计算或数据处理的预定义例程。 | `SUM()` 对值求和 |
| | | `AVG()` 计算平均值 |
| | | `COUNT()` 计算行数 |
| | | `MAX()`、`MIN()` 查找极值 |
| | | `UPPER()`、`LOWER()` 更改大小写 |
| | | `NOW()`、`DATEADD()` |

## SQL 基础

SQL 语句根据其在数据库管理中的作用被分类为不同的功能组。表 A-2 说明了四种主要的 SQL 语句类型，按其用途、描述和数据库操作中使用的常见命令进行分类。

表 A-2: 四种主要的 SQL 语句类型

| 类别 | 描述 | 常用命令 |
| --- | --- | --- |
| 数据定义语言 | 定义和修改数据库模式及结构。 | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| 数据操作语言 | 管理表中的数据。 | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| 数据控制语言 | 控制访问和权限。 | `GRANT`, `REVOKE` |
| 事务控制语言 | 管理数据库内的事务。 | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

在所有上述主要的 SQL 语句类型中，DML 是数据分析中最常用的。表 A-3 展示了四个基本的 DML 命令，重点说明了它们的用途、基本语法以及在数据库中管理数据的示例用例。

表 A-3: 基本 DML 命令

| 命令 | 用途 | 基本语法 | 示例用例 |
| --- | --- | --- | --- |
| `SELECT` | 从一个或多个表中检索数据。 | `SELECT column1, column2 FROM table_name WHERE condition;` | 从 `Customers` 表中获取所有客户姓名。 |
| `INSERT` | 向表中添加新数据行。 | `INSERT INTO table_name (column1, column2) VALUES (value1, value2);` | 向 `Products` 表添加一个新产品。 |
| `UPDATE` | 修改表中的现有数据。 | `UPDATE table_name SET column1 = value1 WHERE condition;` | 更改 `Customers` 表中客户的电子邮件地址。 |
| `DELETE` | 从表中删除行。 | `DELETE FROM table_name WHERE condition;` | 从 `Products` 表中移除已停产的产品。 |

表 A-4 说明了关键的 DDL 命令，概述了它们的用途、基本语法以及用于定义和修改数据库结构的示例用例。

表 A-4: 基本 DDL 命令

| 命令 | 用途 | 基本语法 | 示例用例 |
| --- | --- | --- | --- |
| `CREATE TABLE` | 定义一个新表及其结构。 | `CREATE TABLE table_name (column1 datatype, column2 datatype, ...);` | 创建一个包含 ID、姓名和电子邮件的用户表。 |
| `ALTER TABLE` | 修改现有表。 | `ALTER TABLE table_name ADD column_name datatype;`<br>`ALTER TABLE table_name DROP COLUMN column_name;` | 向 `Users` 表添加一个 `Age` 列。 |
| `DROP TABLE` | 永久删除一个表及其数据。 | `DROP TABLE table_name;` | 删除 `archived_orders` 表。 |
| `TRUNCATE TABLE` | 从表中删除所有行（比 `DELETE` 更快）。 | `TRUNCATE TABLE table_name;` | 快速清空 `Logs` 表中的所有数据。 |

SQL 支持一系列数据类型来表示不同种类的值。表 A-5 总结了常用的、与 PostgreSQL 兼容的 SQL 数据类型。

表 A-5: 常见数据类型

| 数据类型 | 描述 | 示例用例 |
| --- | --- | --- |
| `INT, INTEGER` | 整数 | 计数记录，ID |
| `DECIMAL(p,s), NUMERIC(p,s)` | 精确小数 | 财务数据，货币 |
| `REAL, DOUBLE PRECISION` | 近似数值 | 科学或统计数据 |
| `CHAR(n), VARCHAR(n)` | 固定或可变长度字符串 | 姓名，电子邮件地址 |
| `TEXT` | 大文本字符串 | 评论，描述 |
| `DATE, TIME, TIMESTAMP` | 日期和时间值 | 出生日期，交易日志 |
| `BOOLEAN` | 真/假值 | 标志，二元状态 |
| `BYTEA` | 二进制数据 | 存储图像，文件，文档 |



## 标识符与命名规范

在 SQL 中，标识符指的是表、列、索引和函数等数据库对象的名称。统一且正确地命名这些对象可以确保更好的代码可读性、减少语法错误并提高可维护性。表 A-6 总结了 PostgreSQL 中与标识符相关的基本规则和最佳实践。

**表 A-6**
PostgreSQL 中与标识符相关的基本规则和最佳实践

| 类别 | 规则/最佳实践 | 描述 | 示例 |
| --- | --- | --- | --- |
| 基本结构 | 以字母或下划线开头 | 标识符必须以字母字符（a-z, A-Z）或下划线（_）开头。 | `users`, `product_details`, `_temp_data` |
|  | 包含字母、数字和下划线 | 后续字符可以是字母、数字（0-9）或下划线。其他字符通常不允许，除非使用引号括起来。 | `user_id`, `order2023`, `item_count` |
| 大小写敏感性（未加引号） | 未加引号的标识符会被转换为小写。 | 因此，`TableName`、`tablename` 和 `TABLENAME` 都会被解释为 `tablename`。以 `MyTable` 创建的表将被引用为 `mytable`。 |
| 大小写敏感性（加引号） | 用双引号（"）括起来的标识符被视为区分大小写。 | `"MyTable"` 不同于 `"mytable"`。 `SELECT "FirstName" FROM "Users";` |
| 保留字 | 避免使用未加引号的 SQL 保留关键字 | 不要使用未加引号的 SQL 关键字（例如 `SELECT`、`FROM`、`WHERE`、`USER`）作为标识符。这将导致语法错误。避免：`SELECT, user, group`（未加引号）。 |
|  | 使用双引号来使用保留字 | 如果必须将保留字用作标识符，请将其用双引号括起来。这通常不鼓励。示例：`"SELECT"`、`"user"`、`"group"`。 |
| 长度限制 | 最大长度 | PostgreSQL 对标识符有最大长度限制，通常约为 63 字节。虽然有时可以超过，但最好保持名称长度合理。考虑使用更短但更具描述性的名称。 |
| 最佳实践 | 具有描述性和意义 | 选择能清晰表明数据库对象用途的名称。不要用 `t1`，而要用 `customers`。不要用 `col1`，而要用 `customer_name`。 |
|  | 保持一致性 | 在整个数据库模式中采用一致的命名约定（例如，snake_case、camelCase）。`customer_order`（snake_case）与 `customerOrder`（camelCase）。 |
|  | 使用下划线分隔 | 下划线通常用于分隔标识符中的单词，以提高可读性。示例：`product_category`、`order_item_id`。 |
|  | 避免特殊字符（未加引号） | 虽然从技术上讲，某些特殊字符在加引号后可能是允许的，但为了简单性和可移植性，最好通常避免使用它们。避免：`data-value, user name`（未加引号）。 |
|  | 考虑表名的复数与单数形式 | 没有严格规定，但要保持一致。复数形式常用于表示实体集合的表。示例：`products`、`orders`。 |
|  | 表示单个值的列名使用单数形式 | 列名通常表示行的一个属性。示例：`customer_id`、`product_name`、`order_date`。 |

## 连接与子查询

表 A-7 展示了各种 SQL `JOIN` 类型和子查询概念，详细说明了它们的目的、基本语法以及如何用于跨表检索和关联数据。

**表 A-7**
SQL 连接类型与子查询概念

| 概念 | 目的 | 基本语法 |
| --- | --- | --- |
| `INNER JOIN` | 仅返回两个表中值匹配的行。 | `SELECT * FROM table1 INNER JOIN table2 ON table1.id = table2.id;` |
| `LEFT JOIN` | 返回左表中的所有行以及右表中的匹配行。 | `SELECT * FROM table1 LEFT JOIN table2 ON table1.id = table2.id;` |
| `RIGHT JOIN` | 返回右表中的所有行以及左表中的匹配行。 | `SELECT * FROM table1 RIGHT JOIN table2 ON table1.id = table2.id;` |
| `FULL JOIN` | 当任一表中存在匹配时，返回所有行。 | `SELECT * FROM table1 FULL JOIN table2 ON table1.id = table2.id;` |
| `CROSS JOIN` | 返回两个表的笛卡尔积。 | `SELECT * FROM table1 CROSS JOIN table2;` |
| `SELF JOIN` | 使用表别名将表与自身连接。 | `SELECT a.name, b.name FROM employees a JOIN employees b ON a.manager_id = b.id;` |
| 非关联子查询 | 独立于外部查询执行。 | `SELECT name FROM employees WHERE dept_id IN (SELECT id FROM departments);` |
| 关联子查询 | 在子查询内部引用外部查询的列。 | `SELECT name FROM employees e WHERE salary > (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id);` |

## 聚合函数与分组

表 A-8 说明了关键的 SQL 聚合函数和分组子句，解释了它们的目的、语法以及如何用于在查询中汇总和筛选数据。

**表 A-8**
关键 SQL 聚合函数与分组子句

| 概念 | 目的 | 基本语法 |
| --- | --- | --- |
| `COUNT()` | 计算行数或非 `NULL` 值的数量。 | `SELECT COUNT(*) FROM table;` |
| `SUM()` | 计算数值列的总和。 | `SELECT SUM(column) FROM table;` |
| `AVG()` | 计算数值列的平均值。 | `SELECT AVG(column) FROM table;` |
| `MIN()` | 查找列中的最小值。 | `SELECT MIN(column) FROM table;` |
| `MAX()` | 查找列中的最大值。 | `SELECT MAX(column) FROM table;` |
| `GROUP BY` 子句 | 基于一个或多个列对行进行分组。 | `SELECT column, AGG_FUNC(column) FROM table GROUP BY column;` |
| `HAVING` 子句 | 基于聚合条件对分组进行筛选。 | `SELECT column, AGG_FUNC(column) FROM table GROUP BY column HAVING condition;` |

## 条件逻辑与控制流

表 A-9 展示了 SQL 条件表达式和函数，描述了它们的目的、语法以及如何帮助在查询中处理逻辑和 `NULL` 值。

**表 A-9**
SQL 条件表达式与函数

| 概念 | 目的 | 基本语法 |
| --- | --- | --- |
| `CASE` 语句 | 对条件进行求值并根据这些条件返回值。 | `SELECT CASE WHEN condition THEN result ELSE alternative END FROM table;` |
| `COALESCE()` | 返回表达式列表中的第一个非 `NULL` 值。 | `SELECT COALESCE(column1, column2, default_value) FROM table;` |
| `NULLIF()` | 如果两个表达式相等则返回 `NULL`；否则返回第一个表达式。 | `SELECT NULLIF(expression1, expression2) FROM table;` |
| 条件表达式 | 在 `WHERE`、`HAVING` 和其他子句中用于筛选或求值逻辑。 | `SELECT * FROM table WHERE condition;` |

## 公用表表达式与视图

表 A-10 说明了 CTE 和视图在 SQL 中的使用，包括它们的目的、语法以及如何支持查询模块化和虚拟表创建。

**表 A-10**
SQL 中的公用表表达式与视图

| 概念 | 目的 | 基本语法 |
| --- | --- | --- |
| `WITH`（非递归 CTE） | 定义一个临时结果集，供更大的查询使用。 | `WITH cte_name AS (SELECT ... FROM table) SELECT ... FROM cte_name;` |
| `WITH RECURSIVE` | 定义一个递归 CTE 以处理分层或迭代数据。 | `WITH RECURSIVE cte_name AS (SELECT ... UNION ALL SELECT ... FROM cte_name ...) SELECT ... FROM cte_name;` |
| `CREATE VIEW` | 基于查询结果创建一个虚拟表。 | `CREATE VIEW view_name AS SELECT ... FROM table;` |
| 使用/查询视图 | 像查询表一样从创建的视图中查询数据。 | `SELECT ... FROM view_name;` |
| `DROP VIEW` | 从数据库中删除视图。 | `DROP VIEW view_name;` |



## 索引与优化

表 A-11 展示了 SQL 索引与性能优化技术，包括索引创建以及如 `EXPLAIN` 和 `ANALYZE` 等用于提升查询效率的分析工具。

表 A-11： SQL 索引与性能优化技术

| 概念 | 用途 | 基本语法 |
| --- | --- | --- |
| `CREATE INDEX` | 在一个或多个列上创建索引，以加快查询性能。 | `CREATE INDEX index_name ON table (column);` |
| `DROP INDEX` | 从数据库中移除一个现有索引。 | `DROP INDEX index_name;` |
| 查询性能建议 | 通过在 `WHERE` 子句中使用选择性列、避免 `SELECT *` 以及规范化数据来提高效率。 | `(最佳实践；非基于语法)` |
| `EXPLAIN` | 显示 SQL 查询的执行计划，帮助分析性能。 | `EXPLAIN SELECT ... FROM table WHERE condition;` |
| `ANALYZE` | 执行查询并提供详细的性能统计数据。 | `EXPLAIN ANALYZE SELECT ... FROM table WHERE condition;` |

## 存储过程与函数

表 A-12 说明了 SQL 中存储过程和函数的用法，包括它们的创建、调用以及用于封装任务和返回值的示例用例。

表 A-12： SQL 中存储过程与函数的用法

| 概念 | 用途 | 基本语法 | 示例 |
| --- | --- | --- | --- |
| 创建存储过程 | 封装一系列 SQL 语句以执行任务。 | `CREATE PROCEDURE procedure_name() LANGUAGE plpgsql AS $$ BEGIN --语句 END; $$;` | `CREATE PROCEDURE log_action() LANGUAGE plpgsql AS $$ BEGIN INSERT INTO logs(action) VALUES ('test'); END; $$;` |
| 调用存储过程 | 执行先前定义的存储过程。 | `CALL procedure_name();` | `CALL log_action();` |
| 创建函数 | 返回一个值，可用于 SQL 表达式中。 | `CREATE FUNCTION function_name() RETURNS return_type LANGUAGE plpgsql AS $$ BEGIN RETURN value; END; $$;` | `CREATE FUNCTION add_numbers(a INT, b INT) RETURNS INT LANGUAGE plpgsql AS $$ BEGIN RETURN a + b; END; $$;` |
| 使用函数 | 在查询或语句中调用该函数。 | `SELECT function_name();` | `SELECT add_numbers(3, 5);` |

表 A-13 重点比较了不同数据库系统（MySQL、PostgreSQL、SQL Server 和 Oracle）中 SQL 方言的差异，展示了在自动递增、字符串函数、`NULL` 处理和查询语法等特性上的不同。

表 A-13： SQL 方言差异

| 特性 | MySQL | PostgreSQL | SQL Server | Oracle |
| --- | --- | --- | --- | --- |
| 自动递增 | `AUTO_INCREMENT` | `SERIAL / BIGSERIAL` | `IDENTITY` | `SEQUENCE` 配合 `NEXTVAL` |
| 限制行数 | `LIMIT n` | `LIMIT n` | `TOP n` | `FETCH FIRST n ROWS ONLY (12c+)` |
| `NULL` 处理 | `IFNULL(expr, value)` | `COALESCE(expr, value)` | `ISNULL(expr, value)` | `NVL(expr, value)` |
| 字符串函数 | `LOWER(str), UPPER(str)` | `LOWER(str), UPPER(str)` | `LOWER(str), UPPER(str)` | `LOWER(str), UPPER(str)` |
| 字符串连接 | `CONCAT(str1, str2) or 'str` |   | `str2'` | `'str1` |
| 日期函数 | `CURDATE(), NOW()` | `CURRENT_DATE, CURRENT_TIMESTAMP` | `GETDATE(), GETUTCDATE()` | `SYSDATE, CURRENT_DATE` |
| `SELECT` 中的子查询 | 支持 | 支持 | 支持 | 支持 |
| 布尔数据类型 | `TINYINT(1) 或 BOOLEAN (别名)` | `BOOLEAN` | `BIT` | 使用 `NUMBER(1)` 或 `CHAR(1)` 加约束 |
| 大小写敏感性 | 取决于操作系统 | 大小写敏感 | 默认不区分大小写 | 默认不区分大小写 |
| 连接语法 | 标准 SQL | 标准 SQL | 标准 SQL | 标准 SQL |

如表 A-13 所示，此表对比了 MySQL、PostgreSQL、SQL Server 和 Oracle 之间关键 SQL 语法和行为的差异。对于自动递增列，MySQL 使用`AUTO_INCREMENT`，PostgreSQL 使用`SERIAL`，SQL Server 使用`IDENTITY`，Oracle 使用`SEQUENCE`。行限制在 MySQL/PostgreSQL 中使用`LIMIT`完成，在 SQL Server 中使用`TOP`完成。每个系统对于`NULL`处理和日期操作使用不同的函数。像`LOWER()`和`UPPER()`这样的字符串函数在所有系统中都很常见。子查询普遍得到支持。布尔类型各不相同——PostgreSQL 支持`BOOLEAN`，而 SQL Server 使用`BIT`。大小写敏感性和`JOIN`语法相似，尽管大小写行为取决于系统或配置。

## PostgreSQL 速查表

此 PostgreSQL 速查表提供了对 PostgreSQL 数据库中最常见 SQL 命令、函数和特性的简明实用参考。表 A-14 展示了基础 SQL 查询，涵盖了一系列操作，如选择、过滤、排序和条件检查，并附有其语法和用法示例。

表 A-14： 基础查询

| 类别 | 类型 | 基本语法 |
| --- | --- | --- |
| `SELECT` | 基础 `SELECT` | `SELECT column1, column2 FROM table_name;` |
| 选择所有列 | `SELECT * FROM table_name;` |
| 选择并起别名 | `SELECT column1 AS col1, column2 AS "Column 2" FROM table_name;` |
| 选择不重复值 | `SELECT DISTINCT column1 FROM table_name;` |
| 带条件的选择 | `SELECT * FROM table_name WHERE condition;` |
| 限制结果数量 | `SELECT * FROM table_name LIMIT 10;` |
| 结果偏移 | `SELECT * FROM table_name OFFSET 10 LIMIT 10;` |
| 过滤 | 基础比较运算符 | `SELECT * FROM table_name WHERE column1 = value;  -- 等于`<br>`SELECT * FROM table_name WHERE column1 > value;  -- 大于`<br>`SELECT * FROM table_name WHERE column1 >= value;  -- 大于等于`<br>`SELECT * FROM table_name WHERE column1 < value;  -- 小于`<br>`SELECT * FROM table_name WHERE column1 <= value;  -- 小于等于`<br>`SELECT * FROM table_name WHERE column1 != value; -- 不等于`<br>`SELECT * FROM table_name WHERE column1 <> value;  -- 不等于（替代写法）` |
| 逻辑运算符 | `SELECT * FROM table_name WHERE condition1 AND condition2;  -- 与`<br>`SELECT * FROM table_name WHERE condition1 OR condition2;  -- 或`<br>`SELECT * FROM table_name WHERE NOT condition;  -- 非` |
| `NULL` 检查 | `SELECT * FROM table_name WHERE column1 IS NULL;`<br>`SELECT * FROM table_name WHERE column1 IS NOT NULL;` |
| 模式匹配 | `SELECT * FROM table_name WHERE column1 LIKE 'pattern%';  -- 以 'pattern' 开头`<br>`SELECT * FROM table_name WHERE column1 LIKE '%pattern';  -- 以 'pattern' 结尾`<br>`SELECT * FROM table_name WHERE column1 LIKE '%pattern%';  -- 包含 'pattern'`<br>`SELECT * FROM table_name WHERE column1 ILIKE '%pattern%';  -- 不区分大小写的 LIKE` |
| 范围检查 | `SELECT * FROM table_name WHERE column1 BETWEEN value1 AND value2;`<br>`SELECT * FROM table_name WHERE column1 NOT BETWEEN value1 AND value2;` |
| 列表检查 | `SELECT * FROM table_name WHERE column1 IN (value1, value2, value3);`<br>`SELECT * FROM table_name WHERE column1 NOT IN (value1, value2, value3);` |
| 排序 | 基础排序 | `SELECT * FROM table_name ORDER BY column1;                -- 默认为升序(ASC)`<br>`SELECT * FROM table_name ORDER BY column1 ASC;            -- 升序`<br>`SELECT * FROM table_name ORDER BY column1 DESC;           -- 降序`<br>`SELECT * FROM table_name ORDER BY column1 DESC, column2 ASC;  -- 多列排序` |

表 A-15 说明了 SQL 聚合与分组函数，展示了如何执行计数、求和、平均值等数据操作，以及如何对聚合结果进行分组和过滤。

表 A-15： 聚合与分组


