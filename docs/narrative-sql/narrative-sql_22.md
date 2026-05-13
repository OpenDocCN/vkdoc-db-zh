# 附录 C：PostgreSQL 元素参考

本参考提供了 PostgreSQL 中可用的 SQL 语句、子句、操作和函数的全面按字母顺序排列的列表。

## S

`模式 (Schema)`：包含命名数据库对象的命名空间。

`序列 (Sequence)`：生成唯一数字标识符的对象。

`SERIAL`：用于创建自增整数列的伪类型。

`空间分区 GiST (SP-GiST)`：用于分区搜索树的索引，适用于可划分为非重叠区域的数据。

`统计函数 (Statistical functions)`：用于统计分析的函数，包括相关性、回归等。

`子查询 (Subquery)`：嵌套在另一个查询内部的查询。

## T

`表 (Table)`：以结构化格式保存的相关数据集合。

`表分区 (Table partitioning)`：将大表划分为更小、更易于管理的部分。

`临时表 (Temporary table)`：仅在会话或事务期间存在的表。

`文本搜索 (Text search)`：使用特定语言操作搜索文本数据的功能。

`事务 (Transaction)`：在数据库管理系统中执行的工作单元。

`触发器 (Trigger)`：响应表上某些事件自动执行的函数。

`TRUNCATE`：比 `DELETE` 更快地删除表中所有行的命令。

`元组 (Tuple)`：关系数据库中对行的另一种称呼。

## U

`UNION/UNION ALL`：组合两个或多个 `SELECT` 语句结果的集合操作。

`UNIQUE 约束 (UNIQUE constraint)`：确保列中的所有值都不同。

`UPDATE`：用于修改表中现有记录的语句。

`UPSERT`：插入新行或更新现有行的操作（`INSERT ... ON CONFLICT`）。

`用户定义函数 (User-Defined Function, UDF)`：由数据库用户创建的自定义函数。

`UUID`：通用唯一标识符，一个用作标识符的 128 位数字。

## V

`VACUUM`：回收由死元组占用的存储空间的过程。

`视图 (View)`：基于 SQL 语句结果的虚拟表。

`易变函数 (Volatile function)`：即使使用相同参数，其结果也可能改变的函数。

## W

`WHERE`：用于筛选记录的子句。

`窗口函数 (Window function)`：在一组与当前行相关的表行上执行计算的函数。

`WITH 子句 (WITH clause)`：引入 CTE（公用表表达式）以简化复杂查询。

`预写式日志 (Write-Ahead Logging, WAL)`：通过在应用更改之前将其写入日志来确保数据完整性的方法。

## X

`XML`：用于处理 XML 数据的数据类型和函数。

`XPath`：用于从 XML 文档中选择节点的查询语言。

## Z

`Z 序曲线 (Z-order curve)`：有时用于多维索引策略的空间填充曲线。

`基于零的索引 (Zero-based indexing)`：在 PostgreSQL 中，数组元素和字符串字符的编号从 0 开始。

## A

`ABS()`—返回数字的绝对值。

`ACOS()`—返回数字的反余弦。

`ADD COLUMN`—向表中添加列。

`ALL`—如果所有子查询值都满足条件，则返回 true。

`ALTER DATABASE`—更改数据库属性。

`ALTER SEQUENCE`—更改序列生成器属性。

`ALTER TABLE`—修改表定义。

`ALTER VIEW`—更改视图属性。

`AND`—当两个条件都为 true 时返回 true 的逻辑运算符。

`ANY`—如果任何子查询值满足条件，则返回 true。

`ARRAY`—创建一个数组值。

`ARRAY_AGG()`—将值聚合到一个数组中。

`ARRAY_APPEND()`—将一个元素追加到数组末尾。

`ARRAY_CAT()`—连接两个数组。

`ARRAY_LENGTH()`—返回数组维度的长度。

`AS`—使用别名重命名列或表。

`ASC`—按升序排序结果（默认）。

`ASIN()`—返回数字的反正弦。

`ATAN()`—返回数字的反正切。

`ATAN2()`—返回两个数字的反正切。

`AVG()`—返回数字列的平均值。

## B

`BEGIN`—启动一个事务块。

`BETWEEN`—检查值是否在某个范围内。

`BIGINT`—具有大范围的整数数据类型。

`BIGSERIAL`—具有大范围的自增整数数据类型。

`BIT_AND()`—按位 `AND` 聚合。

`BIT_LENGTH()`—返回字符串中的位数。

`BIT_OR()`—按位 `OR` 聚合。

`BOOLEAN`—逻辑布尔数据类型（true/false）。

`BTRIM()`—从字符串两端移除字符。

## C

`CASE`—在 SQL 中提供条件逻辑。

`CAST`—将值从一种数据类型转换为另一种。

`CBRT()`—返回数字的立方根。

`CEIL()`—将数字向上舍入到最接近的整数。

`CEILING()`—将数字向上舍入到最接近的整数。

`CHAR_LENGTH()`—返回字符串中的字符数。

`CHARACTER VARYING`—变长字符数据类型。

`CHECK`—创建一个约束来检查条件。

`CHR()`—返回给定 Unicode 码点对应的字符。

`CLOSE`—关闭游标。

`CLUSTER`—基于索引物理重新排序表。

`COALESCE()`—返回列表中第一个非 `NULL` 值。

`COLLATE`—为排序操作指定排序规则。

`COLLATION FOR`—返回表达式的排序规则。

`COLUMN`—指定表中的列。

`COMMIT`—提交当前事务。

`CONCAT()`—将字符串连接在一起。

`CONCAT_WS()`—使用分隔符连接字符串。

`CONSTRAINT`—在表上定义约束。

`CORR()`—返回相关系数。

`COS()`—返回数字的余弦。

`COSH()`—返回数字的双曲余弦。

`COUNT()`—计算行数或非 `NULL` 值的数量。

`COVAR_POP()`—返回总体协方差。

`COVAR_SAMP()`—返回样本协方差。

`CREATE DATABASE`—创建一个新数据库。

`CREATE EXTENSION`—安装一个扩展。

`CREATE FUNCTION`—定义一个新函数。

`CREATE INDEX`—创建一个索引。

`CREATE MATERIALIZED VIEW`—创建一个物化视图。

`CREATE RULE`—定义一个新的重写规则。

`CREATE SCHEMA`—创建一个新模式。

`CREATE SEQUENCE`—创建一个序列生成器。

`CREATE TABLE`—创建一个新表。

`CREATE TABLE AS`—根据查询结果创建一个表。

`CREATE TRIGGER`—创建一个触发器。

`CREATE TYPE`—定义一个新数据类型。

`CREATE USER`—创建一个新的数据库用户。

`CREATE VIEW`—创建一个视图。

`CROSS JOIN`—产生两个表的笛卡尔积。

`CURRENT_DATE`—返回当前日期。

`CURRENT_TIME`—返回当前时间。

`CURRENT_TIMESTAMP`—返回当前日期和时间。

`CURRENT_USER`—返回当前用户名。

## D

`DATE`—日期数据类型。

`DATE_PART()`—提取日期/时间值的一部分。

`DATE_TRUNC()`—将日期/时间值截断到指定精度。

`DECIMAL`—具有固定精度和小数位数的精确数值数据类型。

`DECLARE`—声明一个游标。

`DEFAULT`—为列指定默认值。

`DELETE`—从表中删除行。

`DESC`—按降序排序结果。

`DISTINCT`—消除重复行。

`DIV()`—整数除法。

`DROP DATABASE`—删除一个数据库。

`DROP FUNCTION`—删除一个函数。

`DROP INDEX`—删除一个索引。

`DROP TABLE`—删除一个表。

`DROP TRIGGER`—删除一个触发器。

`DROP VIEW`—删除一个视图。

## E

`ELSE`—`CASE` 语句中的替代条件。

`END`—结束条件块或事务。

`EXCEPT`—返回在第一个查询中但不在第二个查询中的行。

`EXISTS`—测试子查询中是否存在记录。

`EXP()`—返回 e 的指定次方。

`EXPLAIN`—显示语句的执行计划。

`EXTRACT()`—提取日期或时间值的一部分。

## F

`FALSE`—逻辑假值。

`FETCH`—从游标中检索行。

`FIRST_VALUE()`—窗口函数，返回有序集中的第一个值。

`FLOAT`—近似数值数据类型。

`FLOOR()`—将数字向下舍入到最接近的整数。

`FOR`—在 PL/pgSQL 中指定循环。

`FOREIGN KEY`—定义外键约束。

`FROM`—指定要查询的表。



## G

`GENERATE_SERIES()`—生成一系列值

`GREATEST()`—从列表中返回最大值

`GROUP BY`—对具有相同值的行进行分组

`GROUPING SETS`—在单个查询中指定多个分组集

## H

`HAVING`—在`GROUP BY`查询中筛选组

`HOUR`—从时间或时间戳中提取小时部分

## I

`IF`—PL/pgSQL 中的条件语句

`ILIKE`—不区分大小写的`LIKE`

`IN`—测试一个值是否匹配列表或子查询中的任何值

`INDEX`—在表的列上创建索引

`INITCAP()`—将每个单词的首字母转换为大写

`INNER JOIN`—当两个表中存在匹配时返回行

`INSERT`—向表中添加新行

`INT` or `INTEGER`—整数数据类型

`INTERSECT`—返回同时存在于两个结果集中的行

`INTERVAL`—时间间隔数据类型

`INTO`—为检索到的数据指定目标

`IS`—测试一个值是否符合布尔真值或`NULL`

`IS NOT NULL`—测试一个值是否不为`NULL`

`IS NULL`—测试一个值是否为`NULL`

## J

`JOIN`—组合来自两个表的行

`JSON_AGG()`—将值聚合为一个 JSON 数组

`JSON_BUILD_ARRAY()`—构建一个 JSON 数组

`JSON_BUILD_OBJECT()`—构建一个 JSON 对象

`JSON_EXTRACT_PATH()`—在指定路径提取 JSON 值

`JSONB_ARRAY_ELEMENTS()`—将 JSON 数组展开为行

`JSONB_EACH()`—将顶级 JSON 对象展开为行

## K

`KEY`—约束定义的一部分

## L

`LAG()`—提供访问前几行的窗口函数

`LANGUAGE`—指定函数的语言

`LAST_VALUE()`—返回有序集合中最后一个值的窗口函数

`LATERAL`—可以引用`FROM`子句中较早出现项的列的子查询

`LEAD()`—提供访问后续几行的窗口函数

`LEAST()`—从列表中返回最小值

`LEFT JOIN`—返回左表中的所有行以及右表中匹配的行

`LENGTH()`—返回字符串中的字符数

`LIKE`—使用通配符进行模式匹配

`LIMIT`—限制返回的行数

`LN()`—返回一个数的自然对数

`LOCALTIME`—返回当前时间

`LOCALTIMESTAMP`—返回当前时间戳

`LOG()`—返回一个数的对数

`LOWER()`—将字符串转换为小写

## M

`MAKE_DATE()`—从年、月、日值创建日期

`MAKE_INTERVAL()`—创建一个间隔值

`MAKE_TIME()`—创建一个时间值

`MAKE_TIMESTAMP()`—创建一个时间戳值

`MAKE_TIMESTAMPTZ()`—创建一个带时区的时间戳值

`MAX()`—返回一组值中的最大值

`MD5()`—计算字符串的 MD5 哈希值

`MIN()`—返回一组值中的最小值

`MOD()`—返回除法运算的余数

`MODE()`—返回一组值中最频繁出现的值

## N

`NEXTVAL()`—从序列中返回下一个值

`NOT`—否定一个条件

`NOT IN`—测试一个值是否不匹配列表或子查询中的任何值

`NOW()`—返回当前日期和时间

`NTH_VALUE()`—返回窗口框架中第 n 个值的窗口函数

`NULLIF()`—如果两个表达式相等则返回`NULL`；否则返回第一个表达式

`NUMERIC`—具有指定精度和范围的精确数值数据类型

## O

`OFFSET`—在开始返回行之前跳过指定行数

`ON`—指定`JOIN`条件

`ON CONFLICT`—为唯一约束冲突指定替代操作

`OR`—当任一条件为真时返回真的逻辑运算符

`ORDER BY`—对结果集进行排序

## P

`PARTITION BY`—为窗口函数将结果集划分为分区

`PERCENTILE_CONT()`—计算连续百分位数

`PERCENTILE_DISC()`—计算离散百分位数

`PG_SLEEP()`—暂停执行指定的秒数

`PI()`—返回圆周率的值

`POSITION()`—查找子字符串在字符串中的位置

`POWER()`—将一个数提升到指定的幂

`PRIMARY KEY`—定义主键约束

## Q

`QUERY`—在各种上下文中用于指代 SQL 语句

## R

`RADIUS`—地理距离函数

`RANDOM()`—返回一个随机数

`RANK()`—为每一行分配排名

`REAL`—单精度近似数值数据类型

`RECURSIVE`—与 CTE 一起使用以定义递归查询

`REFERENCES`—定义外键约束

`REFRESH MATERIALIZED VIEW`—更新物化视图

`REGR_AVGX()`—返回自变量的平均值

`REGR_AVGY()`—返回因变量的平均值

`REGR_COUNT()`—返回输入行的数量

`REGR_SLOPE()`—返回线性回归线的斜率

`RETURN`—从函数中返回

`RETURNING`—从插入、更新或删除的行中返回值

`RIGHT JOIN`—返回右表中的所有行以及左表中匹配的行

`ROLLBACK`—回滚当前事务

`ROUND()`—将一个数四舍五入到指定精度

`ROW_NUMBER()`—为每一行分配一个唯一的数字

## S

`SAVEPOINT`—在事务中创建一个保存点

`SCALE()`—返回数值的小数位数

`SCHEMA`—包含命名数据库对象的命名空间

`SELECT`—从数据库中检索数据

`SEQUENCE`—生成序列号的对象

`SERIAL`—自动递增的整数数据类型

`SESSION_USER`—返回会话用户名

`SET`—更改运行时参数

`SET CONSTRAINTS`—设置约束检查模式

`SETVAL()`—设置序列的当前值

`SHA1()`, `SHA224()`, `SHA256()`, `SHA384()`, `SHA512()`—加密哈希函数

`SHOW`—显示运行时参数的值

`SIGN()`—返回一个数的符号

`SIN()`—返回一个数的正弦值

`SINH()`—返回一个数的双曲正弦值

`SMALLINT`—小范围整数数据类型

`SOME`—如果某些子查询值满足条件则返回真

`SQRT()`—返回一个数的平方根

`STDDEV()`—返回数值列的标准差

`STDDEV_POP()`—总体标准差

`STDDEV_SAMP()`—样本标准差

`STRING_AGG()`—使用分隔符连接值

`SUBSTRING()`—提取子字符串

`SUM()`—计算一组值的总和

## T

`TABLE`—指代数据库中的表

`TAN()`—返回一个数的正切值

`TANH()`—返回一个数的双曲正切值

`TEXT`—可变长度字符数据类型

`THEN`—在`CASE`语句中条件为真时执行

`TIME`—时间数据类型

`TIMESTAMP`—日期和时间数据类型

`TIMESTAMPTZ`—带时区的时间戳数据类型

`TO_CHAR()`—将值转换为带格式的字符串

`TO_DATE()`—将字符串转换为日期

`TO_JSON()`—将值转换为 JSON

`TO_NUMBER()`—将字符串转换为数字

`TO_TIMESTAMP()`—将字符串转换为时间戳

`TRANSACTION`—在数据库系统中执行的工作单元

`TRANSLATE()`—替换字符串中指定的字符

`TRIGGER`—响应事件自动执行的数据库对象

`TRIM()`—从字符串两端移除指定的字符

`TRUE`—逻辑真值

`TRUNC()`—截断数字或日期

`TRUNCATE`—快速删除表中的所有行

## U

`UNION`—组合两个或多个`SELECT`语句的结果集（去除重复项）

`UNION ALL`—组合两个或多个`SELECT`语句的结果集（保留重复项）

`UNIQUE`—强制列中值的唯一性

`UPDATE`—修改表中的现有行

`UPPER()`—将字符串转换为大写

`USER`—返回当前用户名

`USING`—在`JOIN`子句中指定`JOIN`列

`UUID`—通用唯一标识符数据类型

`UUID_GENERATE_V4()`—生成随机的 UUID

## V

`VACUUM`—数据库的垃圾回收和可选分析

`VALUES`—从显式指定的值构造表行

`VARCHAR`—可变长度字符数据类型

`VARIANCE()`—返回数值列的方差

`VAR_POP()`—总体方差

`VAR_SAMP()`—样本方差

`VIEW`—基于`SELECT`查询的虚拟表



## W

`WHEN`—在 `CASE` 语句中指定一个条件

`WHERE`—根据条件筛选行

`WIDTH_BUCKET()`—返回操作数所属的分组桶编号

`WINDOW`—用于窗口函数的命名窗口规范

`WITH`—引入公共表表达式（CTEs）

## X

`XML`—XML 数据类型

`XMLAGG()`—聚合 XML 值

`XMLELEMENT()`—创建一个 XML 元素

`XMLFOREST()`—从列创建 XML 元素

`XMLPARSE()`—将字符串解析为 XML

`XMLROOT()`—更改 XML 值的属性

`XMLSERIALIZE()`—将 XML 转换为字符串

`XMLTABLE()`—从 XML 生成表

## Y-Z

`YEAR`—从日期中提取年份

`ZERO()`—用于设置或测试零值的各种函数

