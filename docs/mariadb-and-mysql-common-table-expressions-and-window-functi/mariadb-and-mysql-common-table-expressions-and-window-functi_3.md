# 第一部分 公用表表达式

## 1. 公用表表达式基础

公用表表达式 (`CTEs`) 是在 `MariaDB 10.2` 和 `MySQL 8.0` 中引入的新 `SQL` 功能之一。本章将介绍 `CTEs`，描述其两种类型，并解释基本语法。`CTEs` 是命名的临时结果集，其生命周期仅限于它们所在的查询。在某些方面，它们类似于派生表，但功能更强大。它们可以递归地引用自身，并且可以在同一查询中被多次引用。它们还支持列分组，并且可以作为视图的替代方案使用，而无需我们拥有 `CREATE VIEW` 权限。`CTEs` 最初是作为 `SQL99` 标准的一部分引入的。


### 开始之前

在深入了解`CTE`是什么以及它们能做什么之前，本章中的示例使用了一些数据，你可以利用这些数据来跟随文本并自行实验`CTE`。本章使用的表名为 `employees` ，可以通过以下查询创建：

```sql
CREATE TABLE employees (
id serial primary key,
name VARCHAR(150) NOT NULL,
title VARCHAR(100),
office VARCHAR(100)
);
```

数据本身位于一个名为 `bartholomew-ch01.csv` 的`CSV`文件中。你可以使用类似以下内容的查询来加载它（假设文件位于运行`MariaDB`或`MySQL`服务器的计算机的 `/tmp/` 文件夹中）：

```sql
LOAD DATA INFILE '/tmp/bartholomew-ch01.csv'
INTO TABLE employees
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"';
```

如果你使用的是`MySQL 8.0`，那么 `secure_file_priv` 设置默认是启用的。在这种情况下，你需要将文件移动到配置指定的位置，或者在你的 `my.cnf` 或 `my.ini` 文件中关闭该设置。

在`Linux`上，`secure_file_priv`的默认位置是 `/var/lib/mysql-files/`，在`Windows`上是 `C:\ProgramData\MySQL\MySQLServer 8.0\Uploads\`；在运行 `LOAD DATA` 命令之前，你需要将文件移动到该位置，然后修改命令以指向该位置，而不是 `/tmp/` 文件夹。

你可以使用以下命令找出本地`MySQL`安装的 `secure_file_priv` 设置是什么：

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
```

我们现在准备开始了。

> **提示**
>
> 在`Windows`上使用`MySQL`命令行客户端时，可以在 `LOAD DATA` 命令中使用`Linux`风格的路径。值得一提的是，如果你选择使用`Windows`风格的路径，则需要使用双反斜杠（`\\`），因为反斜杠字符用于转义其他字符。例如，以下内容是等效的：

```sql
LOAD DATA INFILE '/ProgramData/MySQL/MySQL Server 8.0/Uploads/file.csv'
LOAD DATA INFILE 'C:\\ProgramData\\MySQL\\MySQL Server 8.0\\Uploads\\file.csv'
```

## 什么是公共表表达式？

`公共表表达式`通常被称为`CTE`。你可以将它们视为一个具有名称的查询结果，你可以在查询的后面部分引用该结果。如果命名结果集听起来有点像视图（`view`）或派生表（`derived table`），那是因为它确实是，但有一些显著的区别。我们马上就会讲到这些区别。首先，`CTE`是什么样子的？

### 基本 CTE 语法

`公共表表达式`的通用语法如下：

```sql
WITH <cte_name> AS (
    <cte_body>
)
<cte_query>
```

`WITH` 和 `AS` 关键字是将`CTE`与普通查询区分开来的标志。如果你看到一个以 `WITH ... AS` 开头的查询，那么你看到的就是一个`CTE`。尖括号 `<>` 中的部分是你需要提供的。现在让我们逐一讲解不同的部分：

*   `<cte_name>` 是我们将在查询后面部分用来引用`CTE`的名称；它可以是任何有效的名称，即不是保留字或函数名。
*   `<cte_body>` 是一个产生结果的 `SELECT` 语句。这部分被包裹在圆括号 `()` 中。
*   `<cte_query>` 是我们在`SQL`查询中引用 `<cte_name>` 的地方。例如：

    ```sql
    SELECT <select_criteria> FROM <cte_name> [WHERE ...]
    ```

*   `<select_criteria>` 是你正常的 `SELECT` 查询标准，带有可选的 `WHERE` 和其他子句。

这可能有点难以想象，所以这里有一个基本有效的`CTE`示例，我们在其中定义了一个 `<cte_name>`，然后从中 `SELECT`。这个`CTE`使用了我们在本章开头加载的示例数据，所以请随意在你的`MariaDB`或`MySQL`服务器上运行它。

```sql
WITH emp_raleigh AS (
SELECT * FROM employees
WHERE office='Raleigh'
)
SELECT * FROM emp_raleigh
WHERE title != 'salesperson'
ORDER BY title;
```

让我们分解一下这里发生了什么。我们的 `<cte_name>` 是 `emp_raleigh`，它是一个简单的 `SELECT` 语句，选择 `employees` 表中`WHERE`办公室为`Raleigh`的每一行。你可以把这个`CTE`看作是`employees`表的一个视图或过滤器。然后，在 `<cte_query>` 部分，我们将 `<cte_name>` 用作一个简单查询的一部分，该查询查找职位不是销售人员的每个员工条目，最后按他们的职位对结果进行排序。因为 `<cte_query>` 使用了我们的 `emp_raleigh` `<cte_name>`，所以我们的结果只来自`employees`表中 `WHERE office='Raleigh'` 的记录。

使用我们的示例数据，结果是：

```sql
+-----+-----------------+------------+---------+
| id  | name            | title      | office  |
+-----+-----------------+------------+---------+
|  73 | Mark Hamilton   | dba        | Raleigh |
|  77 | Nancy Porter    | dba        | Raleigh |
| 135 | Pauline Neal    | dba        | Raleigh |
|  68 | Edmund Hines    | manager    | Raleigh |
|  28 | Marc Greene     | programmer | Raleigh |
|  96 | Mary Walker     | programmer | Raleigh |
| 100 | Freida Duchesne | programmer | Raleigh |
+-----+-----------------+------------+---------+
```

因此，除了可能存在的销售人员之外，我们的`Raleigh`办公室看起来非常技术化，除了一个经理外，都是`DBA`和程序员。

### CTE 的动机

我们简单示例的结果看起来很像你可能使用派生表（又名内联视图）或视图来获得的东西，这两者在`MariaDB`和`MySQL`中已经存在多年了。为什么有人会想使用`CTE`而不是更熟悉的视图或派生表呢？以下是一些原因。

#### 临时性

首先，`CTE`是临时的。一个`CTE`在同一个查询中定义和使用。而视图则更持久，可以被认为是一个某种程度上持久的虚拟表。`CTE`的这种临时性可能是件好事。因为`CTE`和使用它的查询都在一起定义，所以修改它以跟上更新的业务需求很容易。这与视图形成对比，视图需要与使用它的查询分开更新。

这种临时性是派生表如此受欢迎的部分原因；它们让你快速生成一个有用的临时结果集，你可以对其执行操作。`CTE`在此基础上构建了更强大的功能集。

#### 可读性

使用`CTE`的一个很大原因是它们通常更具可读性。复杂的视图或嵌套的派生表查询可能很难被普通人解析，通常需要从内到外、从后到前或以其他不自然的顺序阅读查询。与`CTE`形成对比，`CTE`通常可以从上到下阅读。

例如，这是我们将简单的`CTE`示例重写为派生表的版本：

```sql
SELECT * FROM (
SELECT * FROM employees
WHERE office='Raleigh'
) AS emp_raleigh
WHERE title != 'salesperson'
ORDER BY title;
```

为了方便你不必回头找，这里是`CTE`版本：

```sql
WITH emp_raleigh AS (
SELECT * FROM employees
WHERE office='Raleigh'
)
SELECT * FROM emp_raleigh
WHERE title != 'salesperson'
ORDER BY title;
```

毫不奇怪，使用我们的示例数据，两个查询的输出是相同的。然而，当你比较它们时，`CTE`只需从头到尾阅读即可理解。另一方面，要理解派生表，你需要首先阅读内部的 `SELECT` 语句，然后跳到外部的 `SELECT` 语句，然后再回到末尾。在像这样的简单示例上，与`CTE`相比的额外困难是最小的，但是随着派生表查询变得更加复杂，阅读它的难度会呈指数级增长。而对于`CTE`来说，难度的增长更线性，因为你总是可以从头到尾自然地阅读。


### 在单处或多处使用

承接上一节，使用 CTE 的另一个原因是：当你仅为**一个查询**需要某些复杂逻辑，而不是在许多不同查询中**多次复用**时。例如，如果你的底层销售表使用 Unix 时间戳存储发票日期，但你的几个应用程序在查询该表时都期望得到 `YYYY-MM-DD` 格式，那么使用视图会是一个绝佳的解决方案；只需定义视图，然后让你的应用程序调用它即可。另一方面，一个仅在单个应用程序中使用一次的复杂视图，如果重写为一个更易于阅读的 CTE，可能更具可维护性。

### 权限

当你在自己的工作站或服务器上处理个人数据库时，你的数据库用户通常拥有 `ALL PRIVILEGES WITH GRANT OPTION` 权限，这意味着你可以对你的表和数据库执行任何需要或想要的操作，包括 `CREATE`、`UPDATE`、`INSERT`、`DELETE` 等。或者，你可能经常直接以 root 数据库用户身份登录，该用户自动拥有所有权限。然而，生产环境中使用的数据库通常定义了更细粒度的访问控制。有些用户只能对某些数据库中的表执行 `SELECT` 操作，而其他用户可以在一些表中插入数据但不能在另一些表中操作，还有一些用户则根据其不同的工作职能被授予或多或少的权限。

你可能会遇到这样的情况：你需要对某个表创建视图，但你没有该表的 `CREATE VIEW` 权限。CTE 只需要 `SELECT` 权限，因此在这种情况下，使用 CTE 是一个绝佳的方式，让你能够获得所需的功能，而无需去麻烦一位数据库管理员（DBA）来为你创建所需的视图，或者请求他们授予你在该表上的 `CREATE VIEW` 权限（他们可能因公司政策而无法这样做）。

### 嵌套

CTE 为我们的 DBA 工具箱带来了几个新技巧，其中之一是：在每个独立的 `<cte_body>` 中，我们都可以引用其他的 CTE。这解决了一个嵌套派生表的重大难题——嵌套的每一层都会极大地增加复杂度。

例如，让我们基于简单的 CTE 示例进一步展开，深入分析数据：定义第二个 `<cte_name>` 及其对应的 `<cte_body>`，仅选择 Raleigh 办公室的 DBA：

```
WITH emp_raleigh AS (
SELECT * FROM employees
WHERE office='Raleigh'
),
emp_raleigh_dbas AS (
SELECT * from emp_raleigh
WHERE title='dba'
)
SELECT * FROM emp_raleigh_dbas;
```

查看这段代码，我们有原始的 `<cte_name>` `emp_raleigh` 及其 `<cte_body>`。然后我们定义了第二个 `<cte_name>` `emp_raleigh_dbas` 及其 `<cte_body>`。`emp_raleigh_dbas` 建立在 `emp_raleigh` 之上，仅查找 `WHERE title='dba'` 的记录。最后，在 `<cte_query>` 部分，我们从 `emp_raleigh_dbas` 中 `SELECT` 所有内容。从语法上看，这比使用嵌套派生表编写的等效查询**可读性高得多**。

使用我们的示例数据，结果如下：

```
+-----+---------------+-------+---------+
| id  | name          | title | office  |
+-----+---------------+-------+---------+
|  73 | Mark Hamilton | dba   | Raleigh |
|  77 | Nancy Porter  | dba   | Raleigh |
| 135 | Pauline Neal  | dba   | Raleigh |
+-----+---------------+-------+---------+
```

### 多路复用

基于在同一查询中定义多个 `<cte_name>` 及其对应 `<cte_body>` 的能力，我们可以在后续的 `<cte_body>` 部分或 `<cte_query>` 部分多次引用一个给定的 `<cte_name>`。作为一个多次引用单个 `<cte_name>` 的示例，下面是一个反自连接查询，用于查找在其各自办公室中**唯一的** DBA：

```
WITH dbas AS (
SELECT * FROM employees
WHERE title='dba'
)
SELECT * FROM dbas A1
WHERE NOT EXISTS (
SELECT 1 FROM dbas A2
WHERE
A2.office=A1.office
AND
A2.name <> A1.name
);
```

在这里，我们的 `dbas` `<cte_name>` 简单地选择了公司所有的 DBA，然后在我们最终的 `<cte_query>` 中，我们引用了 `dbas` **两次**，以便过滤掉所有我们不感兴趣的 DBA，只留下目标记录。

使用我们的示例数据，结果如下：

```
+----+---------------+-------+---------+
| id | name          | title | office  |
+----+---------------+-------+---------+
|  6 | Toby Lucas    | dba   | Wichita |
| 16 | Susan Charles | dba   | Nauvoo  |
+----+---------------+-------+---------+
```

我认为管理层应该确保 Toby 和 Susan 每年参观公司其他办公室几次，这样他们就不会感到与公司的其他 DBA 孤立了。

### 递归

CTE 如此有用的最后一个原因是它们可以是**递归**的。在它们自己的 `<cte_body>` 内部，它们可以调用自身。这项技术提供了强大的能力。但我们现在先不深入讨论——有一整章专门介绍这类 CTE 查询，因此我们这里不再赘述。

### 总结

在本章中，我们介绍了 CTE 的基本语法，以及为什么该功能被添加到 SQL 标准中，并现在正被添加到 MariaDB 和 MySQL 中的一些原因。我们还通过了几个非递归 CTE 的简单示例。

在下一章中，我们将更深入地探讨非递归 CTE，然后在第 3 章讨论递归 CTE。

## 2. 非递归公用表表达式

在上一章你已经初步体验了非递归 CTE。本章将基于前面的示例进行扩展，展示更多你可以用非递归 CTE 实现的功能。在本章中，我们将介绍 CTE 的一些常见用法，并最终讲解如何将使用子查询的现有查询转换为使用 CTE 的查询。

### 开始之前

与上一章一样，本章的示例使用了示例数据。除了我们之前使用的 `employees` 表外，本章我们还将使用一个名为 `commissions` 的表。该表可以通过以下查询创建：

```
CREATE TABLE commissions (
id serial primary key,
salesperson_id BIGINT(20) NOT NULL,
commission_id BIGINT(20) NOT NULL,
commission_amount DECIMAL(12,2) NOT NULL,
commission_date DATE NOT NULL
);
```

数据在一个名为 `bartholomew-ch02.csv` 的 CSV 文件中。可以使用类似于下面的查询来加载数据（假设该文件位于运行 MariaDB 或 MySQL 服务器的计算机的 `/tmp/` 文件夹中）：

```
LOAD DATA INFILE '/tmp/bartholomew-ch02.csv'
INTO TABLE commissions
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"';
```

注意

有关在 Windows 上加载文件以及解决 `secure_file_priv` 问题的额外信息，请参阅第 1 章的“开始之前”部分。

我们现在可以开始了。



### 使用 CTE 进行年度同比比较

#### 传统 SQL 查询方法

在上一章中，我们介绍了在单个查询中多次引用 CTE 的能力。让我们来看一个更实质性的例子。

许多公司喜欢追踪的一项指标是销售额的年度同比增减情况。在我们的示例 `commissions` 表中，我们跟踪了公司 55 名销售人员各自赚取的佣金及赚取时间。某天，CEO 找到我们说，他想要比较销售人员在不同年份的表现情况。一个相当直接的传统 SQL 查询就能轻松地获取我们想要的数据，并按销售人员和年份分组：

```sql
SELECT
    salesperson_id,
    YEAR(commission_date) AS year,
    SUM(commission_amount) AS total
FROM
    commissions
GROUP BY
    salesperson_id, year;
```

我们的示例数据包含 2016 和 2017 年的佣金数据，因此此查询会返回 110 行结果——我们 55 位销售人员每人两行。以下是截断后的结果：

```
+----------------+------+----------+
| salesperson_id | year | total    |
+----------------+------+----------+
|              3 | 2016 | 2249.93  |
|              3 | 2017 | 3449.67  |
|              7 | 2016 | 1088.32  |
|              7 | 2017 | 3197.25  |
|              8 | 2016 | 4514.73  |
|              8 | 2017 | 5178.19  |
|             10 | 2016 | 9433.58  |
|             10 | 2017 | 8479.05  |
...
+----------------+------+----------+
```

我们可以将此视为完成，并将数据发送给 CEO。

#### 使用 CTE 进行改进

使用我们最初的查询作为 `<cte_body>`，我们可以从中选择两次，使用 `WHERE` 子句来设定条件，以同时选择给定年份及其前一年的数据。它可能看起来像这样：

```sql
WITH commissions_year AS (
    SELECT
        salesperson_id,
        YEAR(commission_date) AS year,
        SUM(commission_amount) AS total
    FROM
        commissions
    GROUP BY
        salesperson_id, year
)
SELECT *
FROM
    commissions_year CUR,
    commissions_year PREV
WHERE
    CUR.salesperson_id=PREV.salesperson_id AND
    CUR.year=PREV.year + 1;
```

设置好名为 `commissions_year` 的 CTE 后，我们从中选择了两次——一次作为 `CUR`，一次作为 `PREV`。`WHERE` 子句用于匹配两者的 `salesperson_id` 字段，并设定条件：我们正在比较某一年与比它大一（`+1`）的另一年。

这一次，输出结果如下所示：

```
+----------------+------+----------+----------------+------+----------+
| salesperson_id | year | total    | salesperson_id | year | total    |
+----------------+------+----------+----------------+------+----------+
|              3 | 2017 | 3449.67  |              3 | 2016 | 2249.93  |
|              7 | 2017 | 3197.25  |              7 | 2016 | 1088.32  |
|              8 | 2017 | 5178.19  |              8 | 2016 | 4514.73  |
|             10 | 2017 | 8479.05  |             10 | 2016 | 9433.58  |
...
+----------------+------+----------+----------------+------+----------+
```

这个呈现方式更好了，但我们可以去掉重复的 `salesperson_id` 列，同时，我们应该 `JOIN` `employees` 表以在输出中包含员工姓名。

#### 通过 JOIN 添加员工信息

我们将在 `<cte_body>` 的 `FROM` 子句中添加 `JOIN`，然后从 `<cte_query>` 部分仅选择我们想要的列放入输出。经过这些修改后，我们完整的 CTE 现在看起来像这样：

```sql
WITH commissions_year AS (
    SELECT
        employees.id AS sp_id,
        employees.name AS salesperson,
        YEAR(commission_date) AS year,
        SUM(commission_amount) AS total
    FROM
        commissions LEFT JOIN employees
            ON commissions.salesperson_id = employees.id
    GROUP BY
        sp_id, year
)
SELECT CUR.sp_id, CUR.salesperson, PREV.year, PREV.total, CUR.year, CUR.total
FROM
    commissions_year CUR,
    commissions_year PREV
WHERE
    CUR.sp_id=PREV.sp_id AND
    CUR.year=PREV.year + 1;
```

而我们现在的输出如下所示：

```
+-------+---------------------+------+----------+------+----------+
| sp_id | salesperson         | year | total    | year | total    |
+-------+---------------------+------+----------+------+----------+
|     3 | Evelyn Alexander    | 2016 | 2249.93  | 2017 | 3449.67  |
|     7 | John Conner         | 2016 | 1088.32  | 2017 | 3197.25  |
|     8 | Leo Gutierrez       | 2016 | 4514.73  | 2017 | 5178.19  |
|    10 | Ryan Fletcher       | 2016 | 9433.58  | 2017 | 8479.05  |
...
+-------+---------------------+------+----------+------+----------+
```

#### 分析结果与进一步筛选

现在，CEO 可以一目了然地看到，尽管 Evelyn、John 和 Leo 的佣金从 2016 年到 2017 年有所增加，但 Ryan 的佣金却比 2016 年减少了约 1000 美元。

如果 CEO 回来并想要一个经过筛选的列表，只显示那些销售额逐年下降的销售人员，我们只需在 `<cte_query>` 部分的末尾添加以下内容：

```sql
AND CUR.total < PREV.total;
```

使用派生表编写等效的分析查询将会庞大得多，并且远没有那么简洁易读。



#### 个体与群体对比

使用子查询时，一个恼人的问题是你需要在查询中多次复制粘贴它们。这些重复的 `FROM (SELECT ...)` 语句是错误的主要来源，尤其是在某些内容发生变化，你需要更新每一个副本时。CTE（公共表表达式）提供了一种消除这种重复的方法。一个给定的 `<cte_body>` 只定义一次，并与一个唯一的 `<cte_name>` 绑定。每当需要时，你只需引用该 `<cte_name>`，如果有什么需要更新，你只需在一处更新 `<cte_body>` 即可。

使用与上一个示例相同的基准 CTE，我们可以修改其后的 `SELECT` 语句，轻松执行一种不同的分析查询，这种查询在传统上需要使用重复的 `FROM (SELECT...)` 语句。这次，我们不是将销售人员与他们自己过去一年的业绩进行比较，而是将他们与所有销售人员进行比较。具体来说，在他下一次发给全公司的电子邮件中，CEO 想要表扬所有在 `2017` 年赚取的佣金至少占全公司所有销售人员总佣金 2% 的销售人员。

```sql
WITH commissions_year AS (
    SELECT
        employees.id AS sp_id,
        employees.name AS salesperson,
        YEAR(commission_date) AS year,
        SUM(commission_amount) AS total
    FROM
        commissions LEFT JOIN employees
            ON commissions.salesperson_id = employees.id
    GROUP BY
        sp_id, year
)
SELECT *
FROM
    commissions_year C1
WHERE
    total > ( SELECT
                0.02*SUM(total)
              FROM
                commissions_year C2
              WHERE
                C2.year = C1.year
                AND C2.year = 2017)
ORDER BY
    total DESC;
```

`commissions_year` 的 `<cte_body>` 与我们之前的示例相比没有变化，因此无需赘述。差异全在 `<cte_query>` 部分。我们的 `SELECT` 语句查找的是个人总额，这些总额至少占 `2017` 年所有佣金总额的 2%，然后将所有结果按 `DESC`（降序）排序，以便收入最高者排在最前。

用传统方式进行这种查询意味着 `<cte_query>` 部分中的每个 `FROM` 都将是一个复制粘贴的 `FROM (SELECT...)` 语句。

如果你一直跟着示例操作，上面代码的输出将如下所示：

```
+-------+------------------+------+----------+
| sp_id | salesperson      | year | total    |
+-------+------------------+------+----------+
|   116 | Christian Reeves | 2017 | 13856.74 |
|    69 | Luis Vaughn      | 2017 | 12570.95 |
|   128 | Stephanie Dawson | 2017 | 12253.44 |
|    38 | Dorothy Anderson | 2017 | 12010.91 |
|    78 | Louis Santiago   | 2017 | 11423.48 |
|   131 | Rene Gibbs       | 2017 | 11147.38 |
|   121 | Christina Terry  | 2017 | 10979.07 |
|    53 | Jennifer Moore   | 2017 | 10967.64 |
|   114 | Veronica Boone   | 2017 | 10651.10 |
|    41 | Terrance Reese   | 2017 | 10219.33 |
|   132 | Alan Carroll     | 2017 | 10066.97 |
|    34 | Bobby French     | 2017 | 9928.69  |
|   105 | Alonzo Page      | 2017 | 9782.69  |
|    66 | Kathryn Barnes   | 2017 | 9433.56  |
|   106 | Bradley Black    | 2017 | 9387.77  |
|   118 | Deborah Peterson | 2017 | 9265.96  |
|    79 | Rafael Sandoval  | 2017 | 9055.54  |
+-------+------------------+------+----------+
```

让我们为 2017 年“2% 俱乐部”的成员们鼓掌！

#### 将子查询转换为 CTE

让我们换个话题，谈谈将现有查询转换为 CTE 的过程。这相当简单。为了说明这一点，这里有一个查询，它与上一个示例中的查询等效，但它没有使用 CTE，而是使用了两个相同的 `FROM (SELECT...)` 语句形式的子查询：

```sql
SELECT *
FROM (
    SELECT
        employees.id AS sp_id,
        employees.name AS salesperson,
        YEAR(commission_date) AS year,
        SUM(commission_amount) AS total
    FROM
        commissions LEFT JOIN employees
            ON commissions.salesperson_id = employees.id
    GROUP BY
        sp_id, year
    ) AS C1
WHERE
    total > ( SELECT
                0.02*SUM(total)
              FROM (
                    SELECT
                        employees.id AS sp_id,
                        employees.name AS salesperson,
                        YEAR(commission_date) AS year,
                        SUM(commission_amount) AS total
                    FROM
                        commissions LEFT JOIN employees
                            ON commissions.salesperson_id = employees.id
                    GROUP BY
                        sp_id, year
                  ) AS C2
              WHERE
                C2.year = C1.year
                AND C2.year = 2017)
ORDER BY
    total DESC;
```

由于存在重复的 `FROM (SELECT...)` 语句，这个查询有 33 行，而执行相同功能的 CTE 版本只有 25 行。八行的差异不算大，但更复杂的查询可能包含五个、七个、十一个甚至更多重复的子查询，导致查询规模几乎呈指数级增长。如果底层表发生变化而我们需要更新查询，这可能迅速变成一场维护噩梦。

将包含重复子查询的查询转换为 CTE 只需三个步骤（如果你需要为额外的重复子查询重复此过程，则为四个）：
1.  找到派生表查询的第一次出现，将其复制到 `SELECT` 行上方，用 `WITH <cte_name> AS (` 包裹开头，并以 `)` 结尾。
2.  将该第一次出现的子查询替换为我们指定的 `<cte_name>`。
3.  遍历查询的其余部分，找到其他相同的子查询，并将它们也替换为 `<cte_name>`。
4.  （可选）如果查询中还有其他重复的子查询，重复此过程。

并非每个查询都完美适合这个过程，但许多查询是适合的。

### 总结

在本章中，我们扩展了前一章的示例，并通过几个更实质性的例子说明了 CTE 相较于子查询的优势，特别是在帮助我们避免难以修改和维护的重复子查询方面。最后，我们讨论了如何将使用子查询的查询转换为使用 CTE 的查询。

为了圆满结束本书关于 CTE 的部分，下一章将介绍我认为 CTE 最令人兴奋的部分——递归。

