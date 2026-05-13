# 10. 分析炼金术：将数据转化为金子

使用 SQL 将数据转化为金子，需要掌握高级分析函数，这些函数能帮助你从原始数据中提取更深层次的见解，并将其转化为引人入胜的叙述。PostgreSQL 提供了强大的函数，可用于将原始数据转化为引人入胜的故事。此外，本章还将阐述如何将原始数据转化为引人入胜的故事，以及如何有效地利用递归来构建查询逻辑。

## 函数

函数可以根据其功能和目的进行分类。PostgreSQL 提供了广泛的函数，虽然聚合函数、统计与数学函数、窗口函数是常见的类别，但还有其他类型，包括排名函数、字符串函数、日期与时间函数、JSON 函数、控制函数和系统函数。SQL 函数帮助你执行那些原本需要多个查询和多次数据库交互的操作。通过使用函数，可以复用你的数据库，因为其他应用程序可以直接与你的存储过程交互，而无需中间层或重复代码。这对大多数数据分析任务都非常有价值。

### 聚合函数

聚合函数对多行数据进行操作，并返回一个汇总后的值。它们通常与 `GROUP BY` 一起使用，以对数据子集执行计算。例如，PostgreSQL 聚合函数可用于查找每个产品类别的总收入。聚合函数对多行数据进行操作，并为每个数据组返回一个汇总值。常见的聚合函数包括 `SUM(列名)`（计算一列的总和）和 `AVG(列名)`（确定平均值）。此外，`COUNT(列名)` 用于计算行数，而 `MIN(列名)` 和 `MAX(列名)` 有助于识别数据集中的最小值和最大值。简而言之，聚合函数的关键特性是它们能将多行数据折叠为每组一个结果。这使得它们非常适合高效地汇总大型数据集。表 10-1 总结了 PostgreSQL 中大多数众所周知的聚合函数。

### 表 10-1：PostgreSQL 中常用的聚合函数

| 函数 | 描述 | 示例 |
| --- | --- | --- |
| `AVG(column_name)` | 返回输入值的平均值（算术平均数）。 | `SELECT AVG(salary) FROM employees;` |
| `COUNT(column_name)` | 返回表达式值不为空的输入行数。 | `SELECT COUNT(department) FROM employees;` |
| `COUNT(*)` | 返回输入行数。 | `SELECT COUNT(*) FROM employees;` |
| `MAX(column_name)` | 返回输入值中的最大值。 | `SELECT MAX(salary) FROM employees;` |
| `MIN(column_name)` | 返回输入值中的最小值。 | `SELECT MIN(salary) FROM employees;` |
| `SUM(column_name)` | 返回输入值的总和。 | `SELECT SUM(salary) FROM employees;` |
| `BOOL_AND(column_name)` | 如果所有输入值都为真，则返回 `TRUE`，否则返回 `FALSE`。 | `SELECT BOOL_AND(active) FROM users;` |
| `BOOL_OR(column_name)` | 如果任一输入值为真，则返回 `TRUE`，否则返回 `FALSE`。 | `SELECT BOOL_OR(active) FROM users;` |
| `EVERY(column_name)` | 等同于 `BOOL_AND`。 | `SELECT EVERY(active) FROM users;` |
| `MODE(column_name)` | 返回出现频率最高的输入值（如果多个值频率相同，则返回任意一个）。 | `SELECT MODE() WITHIN GROUP (ORDER BY department) FROM employees;` |
| `PERCENTILE_CONT(fraction)` | 返回一个百分位数，如果需要，在相邻的输入项之间进行插值。 | `SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) FROM employees;` |
| `PERCENTILE_DISC(fraction)` | 返回一个百分位数；结果是实际的输入元素。 | `SELECT PERCENTILE_DISC(0.9) WITHIN GROUP (ORDER BY salary) FROM employees;` |



### 统计和数学函数

统计和数学函数执行高级的数值计算，常用于预测建模和趋势分析。PostgreSQL 中统计和数学函数的一个实际用例是计算广告支出与销售额之间的相关性。

#### 相关系数 (Correlation)

在统计学中，相关系数衡量两个变量之间关系的强度和方向。它量化了一个变量的变化如何与另一个变量的变化相关联。相关系数 (r) 的范围从 -1 到 1：

*   r = 1 → 完全正相关（两者一起增加）。
*   r = -1 → 完全负相关（一个增加，另一个减少）。
*   r = 0 → 无相关（没有关系）。

最常见的相关方法是皮尔逊相关系数，它假设线性关系。其他类型包括用于排序数据的斯皮尔曼等级相关和用于有序数据的肯德尔等级相关。

通过统计和数学函数，我们可以超越简单的聚合，提供统计洞察和数值建模。例如，`REGR_SLOPE(y, x)` 和 `REGR_INTERCEPT(y, x)` 执行线性回归分析，帮助识别两个变量之间的趋势。

#### 线性回归 (Linear Regression)

线性回归分析是一种统计方法，通过拟合一条直线来建模两个变量之间的关系。它根据自变量 (X) 预测因变量 (Y) 如何变化。方程式是：

`Y=mX+b`

其中 `m` 是斜率（变化率），`b` 是截距（当 `X = 0` 时的 Y 值）。线性回归分析有助于趋势分析、预测和决策制定。为了获得准确的预测，其假设包括线性、独立性、同方差性和残差的正态性。

#### 数据离散度 (Data Dispersion)

为了衡量数据离散度，像 `STDDEV(column_name)` 和 `VARIANCE(column_name)` 这样的函数量化了数据集中的变异性。

数据离散度或变异性指的是数据集中值的分散程度。它有助于理解数据内部的变异程度。常见的离散度度量包括范围、方差和标准差。范围是最大值和最小值之间的差值。方差是每个数据点与平均值之间的平均平方偏差，显示数据点与平均值的差异程度。标准差是方差的平方根，提供了单个数据点偏离平均值的程度度量。

较高的离散度表示较大的变异性，而较低的离散度意味着数据点更接近平均值。

数学变换，如 `POWER(x, y)`、`LOG(x)` 和 `SQRT(x)`，进一步有助于调整和规范化数据以供分析。这些函数的关键优势在于它们能够提取对数据驱动决策至关重要的模式、关系和见解。表 10-2 总结了 PostgreSQL 中大多数著名的统计和数学函数。

**表 10-2：常用统计和数学函数**

| 类别 | 函数 | 描述 | 示例 |
| --- | --- | --- | --- |
| 数学 | `ABS(x)` | 返回 `x` 的绝对值。 | `SELECT ABS(-5);` |
| 数学 | `CBRT(x)` | 返回 `x` 的立方根。 | `SELECT CBRT(27);` |
| 数学 | `CEIL(x) 或 CEILING(x)` | 返回大于或等于 `x` 的最小整数。 | `SELECT CEIL(4.2);` |
| 数学 | `EXP(x)` | 返回 e 的 `x` 次方。 | `SELECT EXP(1);` |
| 数学 | `FLOOR(x)` | 返回小于或等于 `x` 的最大整数。 | `SELECT FLOOR(4.8);` |
| 数学 | `LN(x) 或 LOG(x)` | 返回 `x` 的自然对数。 | `SELECT LN(10);` |
| 数学 | `LOG(b, x)` | 返回以 `b` 为底的 `x` 的对数。 | `SELECT LOG(10, 100);` |
| 数学 | `MOD(y, x)` | 返回 `y` 除以 `x` 的余数。 | `SELECT MOD(11, 3);` |
| 数学 | `PI()` | 返回 π 的值。 | `SELECT PI();` |
| 数学 | `POWER(x, y) 或 POW(x, y)` | 返回 `x` 的 `y` 次方。 | `SELECT POWER(2, 3);` |
| 数学 | `ROUND(x)` | 将 `x` 四舍五入到最接近的整数。 | `SELECT ROUND(4.5);` |
| 数学 | `ROUND(x, s)` | 将 `x` 四舍五入到 `s` 位小数。 | `SELECT ROUND(4.567, 2);` |
| 数学 | `SIGN(x)` | 返回 `x` 的符号（-1、0 或 1）。 | `SELECT SIGN(-10);` |
| 数学 | `SQRT(x)` | 返回 `x` 的平方根。 | `SELECT SQRT(16);` |
| 数学 | `TRUNC(x)` | 向零截断 `x`。 | `SELECT TRUNC(4.8);` |
| 数学 | `TRUNC(x, s)` | 将 `x` 截断到 `s` 位小数。 | `SELECT TRUNC(4.567, 2);` |
| 数学 | `RANDOM()` | 返回 0 到 1 之间的伪随机数。 | `SELECT RANDOM();` |
| 统计 | `AVG(expression)` | 返回输入值的平均值（算术平均）。 | `SELECT AVG(salary) FROM employees;` |
| 统计 | `CORR(y, x)` | 返回相关系数。 | `SELECT CORR(sales, advertising) FROM marketing;` |
| 统计 | `COVAR_POP(y, x)` | 返回总体协方差。 | `SELECT COVAR_POP(sales, advertising) FROM marketing;` |
| 统计 | `COVAR_SAMP(y, x)` | 返回样本协方差。 | `SELECT COVAR_SAMP(sales, advertising) FROM marketing;` |
| 统计 | `STDDEV_POP(expression)` | 返回总体标准差。 | `SELECT STDDEV_POP(salary) FROM employees;` |
| 统计 | `STDDEV_SAMP(expression)` | 返回样本标准差。 | `SELECT STDDEV_SAMP(salary) FROM employees;` |
| 统计 | `VARIANCE(expression)` | 返回样本方差。 | `SELECT VARIANCE(salary) FROM employees;` |
| 统计 | `VAR_POP(expression)` | 返回总体方差。 | `SELECT VAR_POP(salary) FROM employees;` |
| 统计 | `VAR_SAMP(expression)` | 返回样本方差。 | `SELECT VAR_SAMP(salary) FROM employees;` |
| 统计 | `REGR_AVGX(y, x)` | 返回自变量 (`x`) 的平均值。 | `SELECT REGR_AVGX(y, x) FROM table;` |
| 统计 | `REGR_AVGY(y, x)` | 返回因变量 (`y`) 的平均值。 | `SELECT REGR_AVGY(y, x) FROM table;` |
| 统计 | `REGR_INTERCEPT(y, x)` | 返回最小二乘回归线的 y 截距。 | `SELECT REGR_INTERCEPT(y, x) FROM table;` |
| 统计 | `REGR_R2(y, x)` | 返回相关系数的平方。 | `SELECT REGR_R2(y, x) FROM table;` |
| 统计 | `REGR_SLOPE(y, x)` | 返回最小二乘回归线的斜率。 | `SELECT REGR_SLOPE(y, x) FROM table;` |
| 统计 | `REGR_COUNT(y, x)` | 返回两个表达式都不为 null 的输入行数。 | `SELECT REGR_COUNT(y, x) FROM table;` |



### 窗口函数

与聚合函数不同，窗口函数不会折叠行，而是在保持对完整数据集访问的同时，为每一行返回一个计算值。与聚合函数相比，窗口函数的关键优势在于它们允许进行行级比较和趋势分析。这使得分析师能够在揭示洞察的同时保持数据的细粒度。表 10-3 总结了 PostgreSQL 中大部分著名的窗口函数。

**表 10-3：常用窗口函数**

| 函数 | 描述 | 示例 |
| --- | --- | --- |
| `ROW_NUMBER()` | 为分区内的每一行分配一个唯一的顺序整数。 | `SELECT ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) FROM employees;` |
| `CUME_DIST()` | 计算值在分区内的累积分布。 | `SELECT CUME_DIST() OVER (PARTITION BY department ORDER BY salary DESC) FROM employees;` |
| `NTILE(n)` | 将分区划分为 n 个近似相等的组，并为每一行分配一个组号。 | `SELECT NTILE(4) OVER (PARTITION BY department ORDER BY salary DESC) FROM employees;` |
| `LAG(expression [, offset [, default]])` | 访问分区中前一行的数据。 | `SELECT LAG(salary, 1, 0) OVER (PARTITION BY department ORDER BY hire_date) FROM employees;` |
| `LEAD(expression [, offset [, default]])` | 访问分区中后一行的数据。 | `SELECT LEAD(salary, 1, 0) OVER (PARTITION BY department ORDER BY hire_date) FROM employees;` |
| `FIRST_VALUE(expression)` | 返回分区中第一行表达式的值。 | `SELECT FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY hire_date) FROM employees;` |
| `LAST_VALUE(expression)` | 返回分区中最后一行表达式的值。 | `SELECT LAST_VALUE(salary) OVER (PARTITION BY department ORDER BY hire_date RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) FROM employees;` |
| `NTH_VALUE(expression, n)` | 返回分区中第 n 行表达式的值。 | `SELECT NTH_VALUE(salary, 3) OVER (PARTITION BY department ORDER BY hire_date) FROM employees;` |
| `COALESCE(value1, value2, ...)` | 返回列表中第一个非 `NULL` 的值。 | `SELECT COALESCE(NULL, 'default', 'other');` (返回 'default') |
| `NULLIF(value1, value2)` | 如果 `value1` 等于 `value2`，则返回 `NULL`；否则返回 `value1`。 | `SELECT NULLIF(10, 10);` (返回 NULL) `SELECT NULLIF(10, 5);` (返回 10) |
| `GREATEST(value1, value2, ...)` | 返回列表中的最大值。 | `SELECT GREATEST(10, 5, 15);` (返回 15) |
| `LEAST(value1, value2, ...)` | 返回列表中的最小值。 | `SELECT LEAST(10, 5, 15);` (返回 5) |
| `ARRAY[value1, value2, ...]` | 从给定值构建一个数组。 | `SELECT ARRAY[1, 2, 3];` (返回 {1, 2, 3}) |
| `GENERATE_SERIES(start, stop [, step])` | 生成一个值序列。 | `SELECT GENERATE_SERIES(1, 5);` (返回 1, 2, 3, 4, 5) |
| `GENERATE_SUBSCRIPTS(array, dimension)` | 为给定数组的指定维度生成有效的下标集。 | `SELECT GENERATE_SUBSCRIPTS(ARRAY['a','b','c'], 1);` (返回 1, 2, 3) |

如表 10-3 所示，一种特殊类型的窗口函数是值函数。使用值函数，您可以操作单个数据值，包括类型转换、`NULL` 处理和数据格式化。这些函数作用于列或表达式中的单个值，从而实现精确的数据操作。

### 排名函数

排名函数对于为查询结果中的行分配顺序至关重要。它们提供了一种根据指定标准对数据进行排名的方法，便于分析和报告。`RANK()` 为每一行分配一个排名，当出现相同排名时会跳过排名。`DENSE_RANK()` 也分配排名，但不会为并列情况跳过排名，确保顺序排名。`ROW_NUMBER()` 为每一行分配一个唯一的顺序整数，无论是否相等。这些函数使用 `OVER()` 子句来定义结果集的排序和分区。它们对于识别表现最佳者、分析销售数据和生成排行榜等任务非常宝贵，能够从有序数据中提供清晰且结构化的洞察。表 10-4 总结了 PostgreSQL 中大部分著名的排名函数。

**表 10-4：常用排名函数**

| 函数 | 描述 | 示例 |
| --- | --- | --- |
| `RANK()` | 为分区内的每一行分配一个排名，并列值之间会有排名间隔。 | `SELECT RANK() OVER (PARTITION BY department ORDER BY salary DESC) FROM employees;` |
| `DENSE_RANK()` | 为分区内的每一行分配一个排名，并列值之间没有排名间隔。 | `SELECT DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) FROM employees;` |
| `ROW_NUMBER()` | 为分区内的每一行分配一个唯一的顺序整数。 | `SELECT ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) FROM employees;` |
| `NTILE(n)` | 将分区划分为 n 个近似相等的组，并为每一行分配一个组号。 | `SELECT NTILE(4) OVER (PARTITION BY department ORDER BY salary DESC) FROM employees;` |
| `PERCENT_RANK()` | 计算分区内每一行的相对排名（0 到 1）。 | `SELECT PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary DESC) FROM employees;` |



### 字符串函数

字符串函数对于文本处理至关重要，支持连接、提取和格式化文本数据等任务。例如，`CONCAT()` 将多个字符串连接成一个，简化了字符串组合。`LOWER()` 和 `UPPER()` 将字符串转换为小写或大写，以确保大小写一致性。`TRIM()` 移除字符串前导和尾随的空格，用于清理文本数据。`LENGTH()` 返回字符串的字符数。`REPLACE()` 替换指定的子字符串，这对于数据更正非常有用。这些函数对于数据清理、格式化和分析不可或缺，能使数据库中的文本数据更易于管理和保持一致性。表 10-5 总结了 PostgreSQL 中大部分众所周知的字符串函数。

表 10-5

常用字符串函数

| 函数 | 描述 | 示例 |
| --- | --- | --- |
| `CONCAT(string1, string2, ...) 或 string1 || string2 || ...` | 连接字符串。 | `SELECT CONCAT('Post', 'greSQL'); (返回 'PostgreSQL')` |
| `LEFT(string, n)` | 返回字符串的前 n 个字符。 | `SELECT LEFT('PostgreSQL', 4); (返回 'Post')` |
| `RIGHT(string, n)` | 返回字符串的最后 n 个字符。 | `SELECT RIGHT('PostgreSQL', 4); (返回 'SQL')` |
| `TRIM([LEADING | TRAILING | BOTH] [characters] FROM string)` | 从字符串中移除前导、尾随或两端的指定字符。 | `SELECT TRIM(' PostgreSQL '); (返回 'PostgreSQL')` |
| `LTRIM(string [, characters])` | 移除字符串的前导字符。 | `SELECT LTRIM(' PostgreSQL'); (返回 'PostgreSQL')` |
| `RTRIM(string [, characters])` | 移除字符串的尾随字符。 | `SELECT RTRIM('PostgreSQL '); (返回 'PostgreSQL')` |
| `UPPER(string)` | 将字符串转换为大写。 | `SELECT UPPER('PostgreSQL'); (返回 'POSTGRESQL')` |
| `LOWER(string)` | 将字符串转换为小写。 | `SELECT LOWER('PostgreSQL'); (返回 'postgresql')` |
| `LENGTH(string)` | 返回字符串的长度。 | `SELECT LENGTH('PostgreSQL'); (返回 10)` |
| `POSITION(substring IN string)` | 返回子字符串在字符串中首次出现的位置。使用 `POSITION` 函数时，返回的位置从 1 开始计数，而非 0。这与许多编程语言（如 C 和 Python）从 0 开始索引的方式不同。 | `SELECT POSITION('gres' IN 'PostgreSQL'); (返回 6)` |
| `STRPOS(string, substring)` | 返回子字符串在字符串中首次出现的位置。 | `SELECT STRPOS('PostgreSQL', 'gres'); (返回 6)` |
| `REPLACE(string, from, to)` | 将所有出现的子字符串替换为另一个子字符串。 | `SELECT REPLACE('PostgreSQL', 'Post', 'New'); (返回 'NewgreSQL')` |
| `INITCAP(string)` | 将每个单词的首字母转换为大写，其余字母转为小写。 | `SELECT INITCAP('postgreSQL database'); (返回 'PostgreSQL Database')` |
| `REPEAT(string, count)` | 将字符串重复指定的次数。 | `SELECT REPEAT('Post', 3); (返回 'PostPostPost')` |
| `SPLIT_PART(string, delimiter, field)` | 使用分隔符将字符串分割成若干部分，并返回指定的部分。 | `SELECT SPLIT_PART('PostgreSQL,Database', ',', 2); (返回 'Database')` |
| `TO_CHAR(value, format)` | 使用指定格式将值转换为字符串。 | `SELECT TO_CHAR(NOW(), 'YYYY-MM-DD'); (例如返回 '2023-10-27')` |
| `MD5(string)` | 计算字符串的 MD5 哈希值。 | `SELECT MD5('PostgreSQL');` |
| `SHA256(string)` | 计算字符串的 SHA256 哈希值。 | `SELECT SHA256('PostgreSQL');` |
| `TRANSLATE(string, from, to)` | 将字符串中的字符替换为其他字符。 | `SELECT TRANSLATE('12345', '13', 'ax'); (返回 'a2x45')` |

### 日期和时间函数

日期和时间函数对于处理时间数据至关重要，支持精确的时间戳操作。`NOW()` 返回当前日期和时间，捕捉当下时刻。`AGE()` 计算两个时间戳之间的间隔，可用于确定经过的时间。`EXTRACT()` 提取特定的日期或时间组成部分，如年、月或小时，用于详细分析。`DATE_TRUNC()` 将时间戳截断到指定的精度，如天或分钟，用于数据聚合。`TO_CHAR()` 将时间戳格式化为自定义的字符串表示形式，便于生成报告。这些函数对于涉及时间序列分析和数据报告的任务必不可少，确保时间数据管理的准确性和高效性。表 10-6 总结了 PostgreSQL 中大部分众所周知的日期和时间函数。

表 10-6

常用日期和时间函数

| 函数 | 描述 | 示例 |
| --- | --- | --- |
| `NOW() 或 CURRENT_TIMESTAMP` | 返回带时区的当前日期和时间。 | `SELECT NOW();` |
| `CURRENT_DATE` | 返回当前日期。 | `SELECT CURRENT_DATE;` |
| `CURRENT_TIME` | 返回带时区的当前时间。 | `SELECT CURRENT_TIME;` |
| `DATE(timestamp)` | 从时间戳中提取日期部分。 | `SELECT DATE('2023-10-27 10:30:00'); (返回 '2023-10-27')` |
| `EXTRACT(field FROM timestamp)` | 从时间戳中提取特定字段（如年、月、日、时、分、秒）。 | `SELECT EXTRACT(YEAR FROM '2023-10-27 10:30:00'); (返回 2023)` |
| `AGE(timestamp1, timestamp2) 或 AGE(timestamp)` | 计算两个时间戳之间的差值，或计算一个时间戳距今的“年龄”。 | `SELECT AGE('2023-10-27', '2023-10-20'); (返回 '7 days')` |
| `DATE_TRUNC(field, timestamp)` | 将时间戳截断到指定字段（如日、月、年）。 | `SELECT DATE_TRUNC('month', '2023-10-27 10:30:00'); (返回 '2023-10-01 00:00:00')` |
| `TO_CHAR(timestamp, format)` | 根据格式字符串将时间戳格式化为字符串。 | `SELECT TO_CHAR('2023-10-27 10:30:00', 'YYYY-MM-DD HH24:MI:SS'); (返回 '2023-10-27 10:30:00')` |
| `TO_TIMESTAMP(string, format)` | 根据格式字符串将字符串转换为时间戳。 | `SELECT TO_TIMESTAMP('2023-10-27 10:30:00', 'YYYY-MM-DD HH24:MI:SS');` |
| `MAKE_DATE(year, month, day)` | 从年、月、日构造一个日期。 | `SELECT MAKE_DATE(2023, 10, 27); (返回 '2023-10-27')` |
| `MAKE_TIME(hour, minute, second)` | 从时、分、秒构造一个时间。 | `SELECT MAKE_TIME(10, 30, 0); (返回 '10:30:00')` |
| `MAKE_TIMESTAMP(year, month, day, hour, minute, second)` | 从年、月、日、时、分、秒构造一个时间戳。 | `SELECT MAKE_TIMESTAMP(2023, 10, 27, 10, 30, 0);` |
| `ISFINITE(timestamp)` | 检查时间戳是否为有限值（非无穷大或负无穷大）。 | `SELECT ISFINITE('2023-10-27'); (返回 true)` |
| `INTERVAL 'value unit'` | 构造一个时间间隔值。 | `SELECT INTERVAL '1 day';` |
| `DATE_PART(text, timestamp)` | 等同于 extract，但返回一个双精度数。 | `SELECT DATE_PART('day', timestamp '2023-10-27 12:00:00');` |
| `JUSTIFY_DAYS(interval)` | 调整时间间隔，使 30 天的时间跨度用月份表示。 | `SELECT JUSTIFY_DAYS(INTERVAL '30 days');` |
| `JUSTIFY_HOURS(interval)` | 调整时间间隔，使 24 小时的时间跨度用天表示。 | `SELECT JUSTIFY_HOURS(INTERVAL '24 hours');` |
| `JUSTIFY_INTERVAL(interval)` | 使用 `JUSTIFY_DAYS` 和 `JUSTIFY_HOURS` 两者来调整时间间隔。 | `SELECT JUSTIFY_INTERVAL(INTERVAL '54 hours 31 days');` |



### JSON 函数

JSON 函数旨在处理 JSON 数据类型，这对于处理半结构化数据至关重要。

> **注意**
> JSON (JavaScript 对象表示法) 是一种轻量级的数据交换格式，非常适合将结构化数据表示为键值对。它具有人类可读性，且易于机器解析。在诸如 PostgreSQL 之类的 SQL 数据库中，JSON 集成允许将半结构化数据与关系数据一同存储和查询。处理灵活数据模型（例如配置设置或文档式数据）的能力至关重要，这些数据无法整齐地放入传统表中。
>
> 在 SQL 中使用 JSON 函数，您可以直接查询和操作 JSON 数据，弥合了结构化与半结构化数据之间的差距。这些函数使分析师能够直接在 PostgreSQL 中查询、修改和聚合 JSON 数据，从而增强了数据管理的灵活性。表 10-7 总结了 PostgreSQL 中大多数常见的 JSON 函数。

表 10-7：常用的 JSON 函数

| 函数 | 描述 | 示例 |
| --- | --- | --- |
| `to_json(anyelement)` | 将 PostgreSQL 值转换为 JSON。 | `SELECT to_json(ARRAY[1, 2, 3]);` (返回 `"[1,2,3]"`) |
| `to_jsonb(anyelement)` | 将 PostgreSQL 值转换为 JSONB。 | `SELECT to_jsonb(ROW(1, 'foo'));` (返回 `'{"f1":1,"f2":"foo"}'`) |
| `json_build_array(VARIADIC "any")` | 从可变参数列表构建 JSON 数组。 | `SELECT json_build_array(1, 'two', null);` (返回 `"[1,\"two\",null]"`) |
| `json_build_object(VARIADIC "any")` | 从可变参数列表构建 JSON 对象。 | `SELECT json_build_object('foo', 1, 'bar', 'baz');` (返回 `'{"foo":1,"bar":"baz"}'`) |

### 控制函数

控制函数使得在数据库函数和触发器中能够嵌入过程式逻辑。`IF-ELSE` 结构支持条件执行，从而能够基于数据值实现分支逻辑。`LOOP` 结构提供迭代能力，便于处理重复性任务。`EXCEPTION` 处理用于捕获和管理错误，确保代码的健壮执行。这些控制结构使开发者能够创建复杂、动态的数据库操作，例如数据验证、条件更新和自定义错误处理。对于需要超越简单 SQL 查询的过程式逻辑的复杂数据库应用构建而言，它们是必不可少的。表 10-8 总结了 PostgreSQL 中大多数常见的控制函数。

表 10-8：常用的控制函数

| 函数 | 描述 | 示例 |
| --- | --- | --- |
| `COALESCE(value1, value2, ...)` | 返回列表中第一个非 `NULL` 值。 | `SELECT COALESCE(NULL, 'default', 'other');` (返回 `'default'`) |
| `NULLIF(value1, value2)` | 如果 `value1` 等于 `value2`，则返回 `NULL`；否则返回 `value1`。 | `SELECT NULLIF(10, 10);` (返回 `NULL`) |
| `GREATEST(value1, value2, ...)` | 返回列表中的最大值。 | `SELECT GREATEST(10, 5, 15);` (返回 `15`) |
| `LEAST(value1, value2, ...)` | 返回列表中的最小值。 | `SELECT LEAST(10, 5, 15);` (返回 `5`) |
| `CASE WHEN condition1 THEN result1 [WHEN condition2 THEN result2 ...] [ELSE resultN] END` | 条件表达式，类似于 `if-then-else` 逻辑。 | `SELECT CASE WHEN age >= 18 THEN 'Adult' ELSE 'Minor' END FROM users;` |
| `ASSERT(condition, message)` | 检查条件，如果为假则引发错误。（主要用于调试。） | `SELECT ASSERT(age > 0, 'Age must be positive') FROM users;` |

### 系统函数

系统函数提供对数据库元数据和系统级操作的访问。`CURRENT_USER` 揭示当前用户名，有助于安全和审计。`VERSION()` 显示 PostgreSQL 服务器的版本，对于兼容性检查至关重要。`pg_sleep()` 将执行暂停指定的时长，对测试和调度很有用。这些函数允许开发者与数据库环境交互、检索系统信息并控制执行流程。表 10-9 总结了 PostgreSQL 中大多数常见的系统函数。

表 10-9：常用的系统函数

| 函数 | 描述 | 示例 |
| --- | --- | --- |
| `CURRENT_USER` | 返回当前用户名。 | `SELECT CURRENT_USER;` |
| `SESSION_USER` | 返回会话用户名。 | `SELECT SESSION_USER;` |
| `USER` | 等同于 `CURRENT_USER`。 | `SELECT USER;` |
| `CURRENT_DATABASE()` | 返回当前数据库的名称。 | `SELECT CURRENT_DATABASE();` |
| `CURRENT_SCHEMA` | 返回当前模式的名称。 | `SELECT CURRENT_SCHEMA;` |
| `CURRENT_SCHEMAS(boolean)` | 返回搜索路径中的模式名称。如果布尔参数为 `true`，则隐式包含的系统模式也会包含在结果中。 | `SELECT CURRENT_SCHEMAS(false);` |
| `VERSION()` | 返回描述 PostgreSQL 服务器版本的字符串。 | `SELECT VERSION();` |

函数类型之间的差异总结见表 10-10。

表 10-10：函数类型之间的差异

| 函数类型 | 目的 | 返回 |
| --- | --- | --- |
| 聚合函数 | 将多行汇总为一个结果。 | 每组单个值 |
| 统计和数学函数 | 执行数值或统计计算。 | 单个或多行值 |
| 窗口函数 | 跨行计算值而不折叠它们。 | 每行一个值 |
| 值函数 | 处理单个数据值，常用于类型转换或 `NULL` 处理。 | 单个值 |
| 排名函数 | 根据指定标准对结果集中的行进行排名。 | 每行一个值 |
| 字符串函数 | 操作字符串数据。 | 单个值 |
| 日期和时间函数 | 对日期和时间执行计算或转换。 | 单个值或多行值 |
| JSON 函数 | 操作或从 JSON/JSONB 数据类型提取数据。 | 单个或多行值 |
| 控制函数 | 控制 PostgreSQL 中的程序流。 | 取决于函数（例如，布尔型、整型） |
| 系统函数 | 与数据库系统或元数据交互。 | 单个值 |

每种函数类型服务于不同的分析需求；这些需求在表 10-11 中进行了总结和比较。

表 10-11：每种函数满足的不同分析需求

| 函数类型 | 描述 |
| --- | --- |
| 聚合函数 | 对多行进行操作并返回单个结果。 |
| 统计和数学函数 | 用于回归、离差和其他数值分析等计算。 |
| 窗口函数 | 跨与当前行相关的行执行计算，允许进行比较。 |
| 值函数 | 返回或操作值，常用于类型转换或处理 `NULL` 值。 |
| 排名函数 | 根据指定的排序标准对行进行排名。 |
| 字符串函数 | 用于字符串操作任务，例如连接和大小写转换。 |
| 日期和时间函数 | 处理日期和时间类型以进行计算和转换。 |
| JSON 函数 | 操作和提取 JSON/JSONB 列中的数据。 |
| 控制函数 | 控制 PostgreSQL 中的程序流。 |
| 系统函数 | 处理系统级信息或数据库元数据。 |



### 在 PostgreSQL 中创建自定义函数

在 PostgreSQL 中创建自定义函数是一种强大的方式，可以封装可重用的逻辑并扩展数据库的功能。以下是创建自定义 PostgreSQL 函数的基本语法：

```sql
CREATE FUNCTION function_name (parameters)
RETURNS return_type AS $$
DECLARE
-- 变量声明（可选）
BEGIN
-- 函数逻辑
RETURN result;
END;
$$ LANGUAGE plpgsql;
```

其中 `CREATE FUNCTION function_name` 定义了函数名称，`parameters` 是函数接受的输入参数（此为可选），`RETURNS return_type` 指定了函数的返回类型（例如 `INTEGER`、`TEXT`，或对于无返回值使用 `VOID`）。`DECLARE` 是可选的，它声明了函数内使用的任何局部变量。`BEGIN` 和 `END` 标记了函数逻辑的开始和结束，`RETURN` 指定了函数返回的值。

例如，创建一个计算矩形面积的函数可能很有用。以下函数接受两个参数 `width` 和 `height`，并将面积作为 `INTEGER` 返回。

```sql
CREATE FUNCTION calculate_area(width INTEGER, height INTEGER)
RETURNS INTEGER AS $$
BEGIN
RETURN width * height;
END;
$$ LANGUAGE plpgsql;
```

`CREATE FUNCTION calculate_area(width INTEGER, height INTEGER)` 语句创建了一个名为 `calculate_area` 的新函数，它接受两个整数参数：`width` 和 `height`。如 `RETURNS INTEGER` 子句所指定，该函数返回一个整数值。函数体包含在 `AS $$ ... $$` 中，这是一个美元符号引号字符串字面量，无需转义特殊字符即可提高可读性。

注意

PostgreSQL 中的美元符号引号字符串字面量是一种无需转义引号等特殊字符即可定义字符串常量的方式。它被包围在双美元符号 (`$$`) 或自定义标签 (`$tag$`) 中。这在函数、存储过程和动态 SQL 中尤其有用，可以提高可读性并避免转义问题。

美元符号引号确保嵌入的单引号不会干扰 SQL 语法，从而更容易编写复杂的查询和函数体。

函数逻辑放在 `BEGIN` 和 `END` 之间，标记了可执行代码块的开始和结束。在此代码块内，`RETURN width * height;` 执行提供的 `width` 和 `height` 值的乘法，返回计算出的面积。`LANGUAGE plpgsql;` 语句指定了该函数是用 PostgreSQL 的过程语言 `PL/pgSQL` 编写的。创建后，该函数可以如下使用：

```sql
SELECT calculate_area(5, 10);
```

此查询将返回 `50`，因为面积计算为 `5 * 10`。

提供了以下两个查询来说明修改或删除函数的基本语法。

你可以通过删除旧版本并创建新版本来更新函数：

```sql
DROP FUNCTION IF EXISTS function_name;
```

要删除一个函数，请使用此语句：

```sql
DROP FUNCTION function_name;
```

使用 `PL/pgSQL` 在 PostgreSQL 中创建自定义函数，允许你定义复杂的逻辑，提高代码的可重用性，并定制数据库行为以满足特定需求。你可以处理输入/输出参数、错误处理以及返回值，以适应你正在实现的业务逻辑。

### 错误处理

PostgreSQL 中的错误处理对于查询、函数和事务的可靠性至关重要。当发生错误时，除非被显式处理，否则 PostgreSQL 会中止事务。

注意

PostgreSQL 中的事务是作为一个工作单元执行的一系列 SQL 语句的序列。它确保原子性，意味着所有操作要么全部成功完成，如果在发生错误时则全部回滚。事务以 `BEGIN` 开始，使用 `COMMIT` 保存更改。如果发生错误，`ROLLBACK` 会撤销所有更改，维护数据库完整性。通常，PostgreSQL 中的事务通常用于 `INSERT`、`UPDATE` 和 `DELETE` 等写入操作的数据完整性和一致性，但在必要时也可用于数据分析任务。虽然纯数据分析通常涉及 `SELECT` 查询、聚合和报告，不需要事务，但某些情况可能受益于事务。PostgreSQL 与许多其他数据库引擎的不同之处在于其预写式日志 (`WAL`) 机制，该机制支持回滚甚至 `CREATE`、`ALTER` 和 `DROP` 等 `DDL`（数据定义语言）语句。此行为进一步增强了其事务一致性和可靠性。

错误有多种原因，包括无效输入、约束违反和系统故障。PostgreSQL 提供了异常处理机制来管理此类情况，特别是在函数和存储过程中。`RAISE EXCEPTION` 语句允许开发者生成自定义错误消息，而 `EXCEPTION` 块则支持结构化错误处理。适当的错误管理有助于防止意外故障，安全回滚事务，并为用户和应用程序提供有意义的反馈。

#### 基本异常处理

一个函数可以使用 `EXCEPTION` 块来捕获并响应错误。以下是一个防止除零错误的示例：

```sql
CREATE FUNCTION safe_divide(numerator INTEGER, denominator INTEGER)
RETURNS FLOAT AS $$
BEGIN
IF denominator = 0 THEN
RAISE EXCEPTION 'Error: Division by zero is not allowed';
ELSE
RETURN numerator::FLOAT / denominator;
END IF;
END;
$$ LANGUAGE plpgsql;
```

这个名为 `safe_divide` 的函数旨在执行除法运算，同时防止常见的除零错误。该函数接受两个整数参数：`numerator` 和 `denominator`。调用时，它首先使用 `IF` 语句检查分母是否等于 `0`。

注意

通常，`IF ELSE` 是一种条件语句，允许程序根据条件是真还是假来执行不同的代码块。在 PostgreSQL 中，`IF ELSE` 在函数、过程和 `DO` 块内使用。例如，基本语法如下：

```sql
DO $$
BEGIN
IF 10 > 5 THEN
RAISE NOTICE 'Condition is True';
ELSE
RAISE NOTICE 'Condition is False';
END IF;
END $$ LANGUAGE plpgsql;
```

在 PostgreSQL 中，`DO` 块是一个匿名代码块，允许在不创建函数的情况下执行过程逻辑。它对于运行临时脚本、测试逻辑或执行一次性操作很有用。

如果分母为零，函数会引发一个异常，并带有描述性错误消息 `"Error: Division by zero is not allowed"`，这会阻止操作继续进行，并提醒用户注意此问题。如果分母不为零，函数继续执行除法操作，首先将分子转换为浮点数（使用 `::FLOAT` 转换表示法），然后除以分母，结果作为浮点数返回。

注意

在 PostgreSQL 函数中，`::` 类型转换操作符用于在返回值之前显式指定其返回类型。这确保了函数始终返回正确的数据类型，并防止了隐式类型转换问题。回想一下，在 PostgreSQL 中，`::` 是类型转换操作符，用于将一种数据类型显式转换为另一种数据类型。它是 `CAST()` 函数的替代方法。但你应该在函数返回中使用 `::`，原因如下：

1.  确保函数返回正确的类型。
2.  在处理混合数据类型时防止隐式转换错误。
3.  通过明确说明来澄清预期的输出类型。

使用此方法确保了比整数除法更精确的结果。该函数是用 PostgreSQL 的过程语言 (`PL/pgSQL`) 编写的，它通过支持常规 `SQL` 中不可用的控制结构和错误处理功能来扩展标准 `SQL`。



#### 使用 EXCEPTION 块进行错误处理

`EXCEPTION WHEN` 子句用于捕获特定错误并定义替代操作。

```
CREATE FUNCTION insert_employee(name TEXT, age INTEGER)
RETURNS VOID AS $$
BEGIN
INSERT INTO employees (employee_name, employee_age) VALUES (name, age);
EXCEPTION
WHEN unique_violation THEN
RAISE NOTICE 'Employee with the same name already exists.';
WHEN check_violation THEN
RAISE NOTICE 'Age must be a positive number.';
WHEN others THEN
RAISE NOTICE 'An unknown error occurred.';
END;
$$ LANGUAGE plpgsql;
```

`insert_employee` 函数创建员工记录并处理常见错误。该函数接受两个参数：`name`，一个表示员工姓名的文本字符串，和 `age`，一个表示员工年龄的整数值。执行时，它尝试将这些值插入 `Employees` 表，具体是插入到 `employee_name` 和 `employee_age` 列。由于其全面的异常处理，该函数特别健壮。如果插入操作违反了唯一性约束（通常意味着数据库中已存在同名员工），它会捕获 `unique_violation` 错误并向用户显示通知。同样，如果 `age` 参数违反了检查约束（可能要求年龄为正数），它会捕获 `check_violation` 错误并通知用户年龄必须为正。最后，它包含一个用于处理插入过程中可能发生的任何其他意外错误的后备异常处理器。该函数返回 `VOID`，这意味着它不返回任何值，只是执行插入操作并进行适当的错误处理。表 10-12 总结了用于处理异常的不同错误类型和用例。

表 10-12
用于处理异常的错误类型和用例

| 错误类型 | 描述 | 用例 |
| --- | --- | --- |
| `unique_violation` | 捕获在具有 `UNIQUE` 约束的列中插入重复值相关的错误。 | 防止插入重复的主键或唯一值。 |
| `check_violation` | 处理违反 `CHECK` 约束（例如负值）时的错误。 | 确保值遵循预定义规则，例如年龄必须为正。 |
| `others` | 捕获任何与特定类型不匹配的未处理错误。 | 通过优雅地处理意外错误来防止崩溃。 |

## 第一个故事：在线服装市场

玛丽是时尚流（Fashion Flow）在线服装市场的数据分析师。她的工作是使用 SQL 分析销售、客户行为和库存。如表 10-13、10-14 和 10-15 所示，公司的数据库包含订单、客户和产品表。玛丽需要回答几个业务问题并执行一些任务：

表 10-15
订单表

| order_id | customer_id | product_id | Quantity | order_date | total_price |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 | 101 | 2 | 2023-02-05 | 40 |
| 2 | 2 | 102 | 1 | 2023-02-10 | 50 |
| 3 | 3 | 103 | 1 | 2023-02-15 | 100 |
| 4 | 4 | 104 | 2 | 2023-02-20 | 160 |
| 5 | 5 | 105 | 1 | 2023-02-25 | 60 |
| 6 | 1 | 102 | 1 | 2023-03-01 | 50 |
| 7 | 3 | 101 | 3 | 2023-03-05 | 60 |
| 8 | 4 | 105 | 2 | 2023-03-10 | 120 |

表 10-14
产品表

| product_id | Name | Category | Price | Stock | Attributes (JSON) |
| --- | --- | --- | --- | --- | --- |
| 101 | T-Shirt | Clothing | 20 | 100 | `{"color": "red", "size": "M"}` |
| 102 | Jeans | Clothing | 50 | 50 | `{"color": "blue", "size": "L"}` |
| 103 | Jacket | Clothing | 100 | 30 | `{"color": "black", "size": "XL"}` |
| 104 | Sneakers | Footwear | 80 | 40 | `{"color": "white", "size": "10"}` |
| 105 | Hoodie | Clothing | 60 | 25 | `{"color": "gray", "size": "L"}` |

表 10-13
客户表

| customer_id | Name | Email | signup_date | total_spent |
| --- | --- | --- | --- | --- |
| 1 | Alice | `alice@mail.com` | 2022-03-15 | 300.5 |
| 2 | Bob | `bob@mail.com` | 2021-07-10 | 150 |
| 3 | Carol | `carol123@mail.com` | 2023-01-20 | 450.75 |
| 4 | David | `dave99@mail.com` | 2021-12-05 | 1200 |
| 5 | Eve | `eve_wonder@mail.com` | 2022-08-25 | 550.25 |

*   总收入是多少？平均订单价值是多少？
*   每月发生了多少笔销售？
*   哪个产品订购次数最多？
*   按消费额排名的顶级客户是谁？
*   客户的邮箱域名是什么？
*   可用产品的颜色有哪些？
*   确保如果库存数据缺失，则显示“缺货”。
*   玛丽能否创建一个根据客户累计消费总额分配“忠诚度分数”的函数？
*   正在运行的 PostgreSQL 版本是什么？当前用户是谁？

以下查询使用聚合函数计算总收入和平均订单价值：

```
SELECT
SUM(total_price) AS total_revenue,
AVG(total_price) AS avg_order_value
FROM orders;
```

表 10-16 展示了总收入和平均订单价值。

表 10-16
总收入和平均订单价值

| total_revenue | avg_order_value |
| --- | --- |
| 640 | 80.00000 |

以下查询计算每月的销售趋势：

```
SELECT
DATE_TRUNC('month', order_date) AS month,
COUNT(order_id) AS total_orders
FROM orders
GROUP BY month
ORDER BY month;
```

表 10-17 展示了每月的销售趋势。

表 10-17
每月销售趋势

| Month | total_orders |
| --- | --- |
| 2023-02-01 | 5 |
| 2023-03-01 | 3 |

以下查询查找最畅销的产品。

```
SELECT
product_id,
SUM(quantity) AS total_sold,
RANK() OVER (ORDER BY SUM(quantity) DESC) AS rank
FROM orders
GROUP BY product_id;
```

表 10-18 展示了最畅销的产品。

表 10-18
最畅销的产品

| product_id | total_sold | Rank |
| --- | --- | --- |
| 101 | 5 | 1 |
| 105 | 3 | 2 |
| 104 | 2 | 3 |
| 102 | 2 | 3 |
| 103 | 1 | 5 |

以下查询使用 `SUM()` 和 `RANK()` 函数识别消费最高的客户。

```
SELECT
customer_id,
SUM(total_price) AS total_spent,
RANK() OVER (ORDER BY SUM(total_price) DESC) AS rank
FROM orders
GROUP BY customer_id;
```

表 10-19 展示了消费最高的客户。



