# 第五部分 函数

## 22. 在 SQL 中使用函数

SQL 的功能远不止读取存储在表中的数据。你常常需要对数据进行操作或计算。这时，函数就派上用场了。

### 什么是函数？

`函数`是存储在数据库中的一段代码，它允许你执行一项特定任务并返回一个输出。一个函数包含三个部分：

*   输入，即你提供给函数的数据
*   处理过程，即函数执行的操作
*   输出，即函数处理过程的结果

Oracle SQL 内置了大量函数，用于执行各种不同的计算或处理过程。为什么要使用函数？函数用于对数据进行操作，例如：

*   统计符合特定条件的记录数量
*   对一个范围内的数字求和
*   将日期转换到另一个时区
*   在一串字符中查找特定字符

当你需要查看的不仅仅是数据库表中存储的值时，很可能就需要使用函数。

### 可以在哪些地方使用函数？

函数可以在 Oracle SQL 的许多地方使用，例如：

*   `SELECT`语句的`SELECT`子句，用于查看函数的结果
*   `SELECT`语句中`JOIN`的`ON`子句
*   `SELECT`语句的`WHERE`子句，用于根据函数的结果过滤数据
*   `ORDER BY`子句
*   `UPDATE`语句的`SET`子句
*   `DELETE`或`UPDATE`语句的`WHERE`子句
*   `INSERT`语句的`VALUES`子句

你已经学习了如何编写`SELECT`、`INSERT`、`UPDATE`和`DELETE`语句。在本章中，你将看到如何在这些语句中使用函数。

### 混合连接类型

到目前为止的示例表明，你可以使用`INNER JOIN`显示多个表中的所有匹配记录，使用`LEFT JOIN`显示所有表中的匹配记录或`NULL`值。你能混合使用这两种方式吗？答案是肯定的，你可以在一个查询中混合使用连接类型。

假设你想查看所有有办公室的员工，以及他们可能有的任何`sales_meetings`（销售会议）。这可以通过从`employee`表中选择，对`office`表执行`INNER JOIN`（以排除没有办公室的员工和没有员工的办公室），然后再对`sales_meeting`表执行`LEFT JOIN`来实现。

查询如下所示：

```
SELECT e.id,
e.last_name,
o.address,
s.company,
s.meeting_date
FROM employee e
INNER JOIN office o ON e.office_id = o.id
LEFT JOIN sales_meeting s ON e.id = s.employee_id;
```

此查询的结果是：

| ID | LAST_NAME | ADDRESS | COMPANY | MEETING_DATE |
| --- | --- | --- | --- | --- |
| 1 | JONES | 123 Smith Street | (null) | (null) |
| 2 | SMITH | 45 Main Street | BW Signage | 21/Aug/18 |
| 3 | KING | 45 Main Street | (null) | (null) |
| 4 | SIMPSON | 123 Smith Street | (null) | (null) |
| 6 | COOPER | 123 Smith Street | (null) | (null) |
| 7 | ADAMS | 10 Collins Road | (null) | (null) |
| 8 | SMITH | 10 Collins Road | (null) | (null) |
| 9 | PATRICK | 123 Smith Street | (null) | (null) |

你可以看到这个结果与之前的不同。员工 ID 为 5 的（ANDERSON）没有显示，因为他没有`office`。尽管他有一个`sales_meeting`，但由于我们对`employee`和`office`执行了`INNER JOIN`，意味着记录必须存在于两个表中，因此他没有被显示。

这表明，在处理多个表时，你可以混合使用连接类型。你使用的连接类型应取决于你的查询需求以及你想看到的结果。

### 连接四个或更多表

你已经看到了一些连接两个和三个表示例查询。你可以使用相同的技术来连接四个、五个甚至更多的表。虽然我们目前只有三个表，但一旦你开始在工作中或业余项目中处理真实的应用程序，你将会使用远超过三个表。

连接超过三个表时，适用相同的技术。例如，假设你有一个名为`department`的表，存储部门的`id`和`name`，例如会计部或销售部。再假设`employee`表中有一个名为`department_id`的列，存储来自`department`表的`id`。

那么，我们的表和字段将是：

*   `employee`: `id`, `last_name`, `salary`, `job_title`, `office_id`, `department_id`
*   `office`: `id`, `address`
*   `sales_meeting`: `id`, `employee_id`, `company`, `meeting_date`
*   `department`: `id`, `department_name`

你可以像这样连接所有这四个表：

```
SELECT e.id,
e.last_name,
d.department_name,
o.address,
s.company,
s.meeting_date
FROM employee e
INNER JOIN office o ON e.office_id = o.id
INNER JOIN sales_meeting s ON e.id = s.employee_id
INNER JOIN department d ON e.department_id = d.id;
```

这将显示所有四个表中相关联的记录。如果你使用我们目前使用的示例数据来运行此查询，你会得到一个错误，因为`department`表不存在。

随着你向数据库中添加越来越多具有关联列的表，你可以在需要时与它们进行连接。你还可以使用`WHERE`子句在任何这些表上过滤数据，使用`ORDER BY`子句按任何列排序，以及使用你目前学到的其他关键字。

你最终可能会得到一个相当长的查询，就像这里展示的：

```
SELECT e.id AS employee_id,
e.last_name,
e.salary,
e.job_title,
d.department_name,
o.address,
s.company,
s.meeting_date
FROM employee e
INNER JOIN office o ON e.office_id = o.id
INNER JOIN sales_meeting s ON e.id = s.employee_id
INNER JOIN department d ON e.department_id = d.id
WHERE o.address LIKE '%Main Street%'
AND e.salary > 30000;
```

此查询将显示所有`办公室`在`Main Street`且`工资`大于 30000 的员工的`员工`、`部门`、`办公室`和`销售会议`信息。

一旦你设置好数据，编写这样的查询就会变得更容易。

### 总结

你可以使用与连接两个表相同的方法来连接三个、四个或更多表。你可以使用`INNER JOIN`、一种`OUTER JOIN`或连接的组合来连接所有表。具体需要哪种连接将取决于你想查看的数据。

你可以在连接中使用其他关键字和子句，例如用于过滤数据的`WHERE`子句和用于排序数据的`ORDER BY`子句，以获取你所需的确切数据。



### 简单的数值计算

SQL 允许你使用一些标准的数学符号执行简单的数学计算。这些符号是：

*   `+` 用于加法
*   `-` 用于减法
*   `*` 用于乘法
*   `/` 用于除法

仅使用这些符号你可能就能得到所需的结果。让我们来看一个例子。

假设你需要查看每位员工的薪资，以及在他们当前薪资基础上增加 10000 后的数额。你的查询可能如下所示：

```sql
SELECT id, last_name, salary, salary + 10000
FROM employee;
```

此查询显示了员工的 `id`、`last_name` 和其当前的 `salary` 值。它还显示了一个新列，即当前 `salary` 值加上 10000，这正是 `salary + 10000` 所做的。这不会更新表中的任何数据；它只是显示当前值加上 10000 后的结果。

该查询的结果是：

| ID | LAST_NAME | SALARY | SALARY+10000 |
| --- | --- | --- | --- |
| 1 | JONES | 30000 | 40000 |
| 2 | SMITH | 35000 | 45000 |
| 3 | KING | 45000 | 55000 |
| 4 | SIMPSON | 52000 | 62000 |
| 5 | ANDERSON | 31000 | 41000 |
| 6 | COOPER | (null) | (null) |
| 7 | ADAMS | (null) | (null) |
| 8 | SMITH | 62000 | 72000 |
| 9 | PATRICK | 40000 | 50000 |

新列被称为 “SALARY+10000”。Oracle 会使用你提供的表达式来标记该列，并移除空格。这些值显示了薪资加上 10000 后的数额。

任何 `NULL` 值将继续显示为 `NULL`。这是因为 `NULL` 是一个未知值，向一个未知值添加任何东西仍然会使它成为一个未知值。

你也可以使用 `-` 符号进行减法运算。例如，要查看 `salary` 值减少 5000 后的数额，你的查询可能如下所示：

```sql
SELECT id, last_name, salary, salary - 5000
FROM employee;
```

结果将是：

| ID | LAST_NAME | SALARY | SALARY-5000 |
| --- | --- | --- | --- |
| 1 | JONES | 30000 | 25000 |
| 2 | SMITH | 35000 | 30000 |
| 3 | KING | 45000 | 40000 |
| 4 | SIMPSON | 52000 | 47000 |
| 5 | ANDERSON | 31000 | 26000 |
| 6 | COOPER | (null) | (null) |
| 7 | ADAMS | (null) | (null) |
| 8 | SMITH | 62000 | 57000 |
| 9 | PATRICK | 40000 | 35000 |

新列再次显示了此计算的结果。它显示了一个比当前 `salary` 值少 5000 的数额。

你也可以使用 `*` 进行乘法运算。你可以乘以整数或小数。例如，你可以查看 `salary` 增加 10%（乘以 1.1）后的情况，或者查看 `salary` 翻倍（乘以 2）后的情况。

```sql
SELECT id, last_name, salary, salary * 1.1, salary * 2
FROM employee;
```

结果：

| ID | LAST_NAME | SALARY | SALARY*1.1 | SALARY*2 |
| --- | --- | --- | --- | --- |
| 1 | JONES | 30000 | 33000 | 60000 |
| 2 | SMITH | 35000 | 38500 | 70000 |
| 3 | KING | 45000 | 49500 | 90000 |
| 4 | SIMPSON | 52000 | 57200 | 104000 |
| 5 | ANDERSON | 31000 | 34100 | 62000 |
| 6 | COOPER | (null) | (null) | (null) |
| 7 | ADAMS | (null) | (null) | (null) |
| 8 | SMITH | 62000 | 68200 | 124000 |
| 9 | PATRICK | 40000 | 44000 | 80000 |

在之前的章节中，你已经了解了列别名，即你可以为结果中的列指定的名称。在执行这类计算时使用列别名是一个好主意。这个查询使结果更易于阅读：

```sql
SELECT id,
last_name,
salary,
salary * 1.1 AS salary_inc_comm,
salary * 2 AS salary_double
FROM employee;
```

这显示了相同的数据，但表明乘以 1.1 是包含佣金的计算，而乘以 2 是翻倍的计算。

| ID | LAST_NAME | SALARY | SALARY_INC_COMM | SALARY_DOUBLE |
| --- | --- | --- | --- | --- |
| 1 | JONES | 30000 | 33000 | 60000 |
| 2 | SMITH | 35000 | 38500 | 70000 |
| 3 | KING | 45000 | 49500 | 90000 |
| 4 | SIMPSON | 52000 | 57200 | 104000 |
| 5 | ANDERSON | 31000 | 34100 | 62000 |
| 6 | COOPER | (null) | (null) | (null) |
| 7 | ADAMS | (null) | (null) | (null) |
| 8 | SMITH | 62000 | 68200 | 124000 |
| 9 | PATRICK | 40000 | 44000 | 80000 |

最后，你可以使用 `/` 符号对数字进行除法运算。要将员工的 `salary` 除以 2，你可以编写以下查询：

```sql
SELECT id,
last_name,
salary,
salary / 2
FROM employee;
```

结果将是：

| ID | LAST_NAME | SALARY | SALARY/2 |
| --- | --- | --- | --- |
| 1 | JONES | 30000 | 15000 |
| 2 | SMITH | 35000 | 17500 |
| 3 | KING | 45000 | 22500 |
| 4 | SIMPSON | 52000 | 26000 |
| 5 | ANDERSON | 31000 | 15500 |
| 6 | COOPER | (null) | (null) |
| 7 | ADAMS | (null) | (null) |
| 8 | SMITH | 62000 | 31000 |
| 9 | PATRICK | 40000 | 20000 |

这些运算符或符号在执行此类计算时非常有用。然而，对于更复杂的计算，你可以使用 Oracle 的内置函数。

### DUAL 表

在我们探讨 Oracle 中可用的函数之前，有一个概念你应该先了解一下。当你在 Oracle 中编写 `SELECT` 语句时，`SELECT` 和 `FROM` 关键字都是必需的。你不能编写没有 `SELECT` 或没有 `FROM` 的 `SELECT` 语句。

这个查询可以工作：

```sql
SELECT id
FROM employee;
```

但这个查询不行：

```sql
SELECT id;
```

有时你可能不需要从表中选择数据。例如，如果你想要获取当前日期，你不需要为此使用特定的表，因为数据库应该知道日期（我们将在本章后面学习这个函数）。或者你可能想显示一个特定的数字来测试一个函数。例如，如果你想显示值 123，那么你可以编写这个查询：

```sql
SELECT 123
FROM employee;
```

然而，这将会为 employee 表的每一行显示一个 123 的值：

| 123 |
| --- |
| 123 |
| 123 |
| 123 |
| 123 |
| 123 |
| 123 |
| 123 |
| 123 |
| 123 |

这显示了九个结果加上列标题。这不是你想要看到的。如果你从查询中移除表，因为你不想要获取 employee 记录呢？

```sql
SELECT 123;
```

当你运行这个查询时，它会显示一个错误：

```
ORA-00923: FROM keyword not found where expected
00923. 00000 -  "FROM keyword not found where expected"
*Cause:
*Action:
Error at Line: 1 Column: 10
```

这是因为 `FROM` 子句是必需的。那么，你如何才能只看到 123 这个值呢？你可以使用一个名为 `DUAL` 的特殊 Oracle 表。

`DUAL` 表是内置于 Oracle 数据库中的一个表。它包含一行和一列。它的主要目的是允许你运行那些使用不需要任何表数据的值或函数的查询。

你可以通过对其运行查询来看到 `DUAL` 表：

```sql
SELECT *
FROM dual;
```

| DUMMY |
| --- |
| X |

`DUAL` 表在一个名为 `DUMMY` 的列中只有一个值 “X”。这意味着你可以使用 `DUAL` 表来执行你之前的查询，以显示值 123：

```sql
SELECT 123
FROM dual;
```

| 123 |
| --- |
| 123 |

你现在可以看到你选择的值。现在看起来可能不那么有用，但是一旦我们开始使用函数，你就会看到这个 `DUAL` 表的好处。这个表是 Oracle 特有的：它在其他 SQL 供应商（如 Microsoft SQL Server 或 MySQL）中不存在。



### 数字函数

Oracle 有一系列“数字函数”，或称为使用数字执行计算的函数。在 Oracle 中，函数可以像这样被引用：

```
函数名(参数)
```

这包括：

*   `函数名`，即函数的名称
*   左括号，其中包含一个或多个参数，或你提供给函数的输入值
*   右括号

函数的结果将在你指定函数的位置使用。例如，如果你在 `SELECT` 子句中将此函数用作列之一，函数的结果将显示在该列中。让我们看一些例子。

### ROUND 函数

你可能经常执行的一个操作是四舍五入数字。Oracle 允许你在数据库中存储十进制数字，但有时你可能希望以整数形式显示它们。有几个函数可以让你做到这一点。其中一个函数叫做 `ROUND`。我将使用大写字母引用 Oracle 函数，因为它们在 Oracle 文档中就是这样编写的，而且在你的 SQL 代码中也更容易阅读。

`ROUND` 函数接受两个参数或输入：

*   你想要四舍五入的数字
*   要四舍五入到的小数位数

函数的结果是你指定的数字，按你指定的小数位数进行了四舍五入。你可以像下面这样在 SQL 查询中编写它，使用一个示例数字：

```
SELECT ROUND(123.456, 0)
FROM dual;
```

我们使用了本章前面提到的 `DUAL` 表。同时使用了 `ROUND` 函数，它包含两个参数。第一个是你想要四舍五入的数字，即 123.456。第二个是要四舍五入到的小数位数，即 0。这意味着数字 123.456 将被四舍五入到 0 位小数。

如果你运行此查询，你将得到以下结果：

| ROUND(123.456,0) |
| --- |
| 123 |

这个 `ROUND` 函数的结果是 123。这是因为 123.456 被向下舍入，而不是向上舍入。

你也可以使用 `ROUND` 函数配合不同的小数位数。例如，要四舍五入到两位小数，你的查询将如下所示：

```
SELECT ROUND(123.456, 2)
FROM dual;
```

你只需将第二个参数更改为数字 2，以代表两位小数。此查询的结果是：

| ROUND(123.456,2) |
| --- |
| 123.46 |

结果是 123.46，因为 123.456 被向上舍入为 123.46。

### CEILING 和 FLOOR 函数

在 Oracle SQL 中还有另外两种对数字进行取整的方法，它们使用 `CEILING` 和 `FLOOR` 函数。`CEILING` 函数会将数字向上舍入到最接近的整数，而 `FLOOR` 函数会将数字向下舍入到最接近的整数。

这些函数的语法是：

```
CEILING (输入数字)
FLOOR (输入数字)
```

这些函数中的每一个都接受单个参数，即要取整的数字。与 `ROUND` 函数不同，这些函数不需要用于指定位数的第二个参数，因为它们总是取整到整数。

使用这些函数的一个查询如下所示。我使用了列别名以使输出更整洁。

```
SELECT
CEILING(123.456) AS 向上取整结果,
FLOOR(123.456) AS 向下取整结果
FROM dual;
```

| 向上取整结果 | 向下取整结果 |
| --- | --- |
| 124 | 123 |

`CEILING` 函数将数字向上舍入为 124，而 `FLOOR` 函数将数字向下舍入为 123。Oracle 中的其他数字函数的工作方式相同，你提供一个输入值，然后计算出一个结果。

### 对表数据使用数字函数

让我们看一个使用表中某列的函数示例。假设 `employee` 表中存储的 `salary` 是一个年薪值，我们想看看月值会是多少。然而，除了月值之外，我们还想看看如果我们将 `salary` 除以 12 并向下取整后，“剩余”金额会是多少。例如，30/12 的值将是 2（2*12=24），剩余 6（30-24=6）。我们可以在 Oracle 中使用函数来实现这一点：`FLOOR` 函数和 `MOD` 函数。

`MOD` 函数执行*模除运算*，这意味着它将一个数字除以另一个数字并返回余数。

我们的查询将如下所示：

```
SELECT
Id,
last_name,
salary,
FLOOR(salary/12) AS 月薪,
MOD(salary, 12) AS 月剩余金额
FROM employee;
```

此查询包含 `FLOOR` 函数，首先将 `salary` 除以 12，然后将结果四舍五入到最接近的整数。`MOD` 函数获取 `salary` 值，将其除以 12，并显示剩余值。

此查询的结果将是：

| ID | 姓氏 | 年薪 | 月薪 | 月剩余金额 |
| --- | --- | --- | --- | --- |
| 1 | JONES | 30000 | 2500 | 0 |
| 2 | SMITH | 35000 | 2916 | 8 |
| 3 | KING | 45000 | 3750 | 0 |
| 4 | SIMPSON | 52000 | 4333 | 4 |
| 5 | ANDERSON | 31000 | 2583 | 4 |
| 6 | COOPER | (null) | (null) | (null) |
| 7 | ADAMS | (null) | (null) | (null) |
| 8 | SMITH | 62000 | 5166 | 8 |
| 9 | PATRICK | 40000 | 3333 | 4 |

这显示了 `salary`，以及向下取整后的月 `salary`。它还显示了 `salary` 除以 12 后的剩余值。例如，员工 JONES 的年薪为 30000，可以有 2500 的月薪，剩余 0，因为 2500*12 = 30000。



### 字符串的连接

在本章前面，你学习了如何使用符号执行数学运算，例如加法和减法。你可以使用另一个符号对字符串或文本值执行不同的操作：*连接*。

连接是将两个字符串值连接在一起的过程。这样做通常是因为值存储在不同的列或不同的表中，但你希望将它们显示在同一列中。例如，如果我们的员工表中有一个名为 `first_name` 的列，你可能希望将 `first_name` 和 `last_name` 连接起来，组成一个完整的姓名。

如何在 Oracle 中连接字符串？你需要使用两个管道符：`"||"`。将这两个字符放在两个其他字符串之间，结果将是一个组合了它们的单个值。

例如，你可以使用连接将名称 `"John"` 和名称 `"Smith"` 组合起来：

```sql
SELECT 'John' || 'Smith'
FROM dual;
```

结果将是：

| JOHN&#124;&#124;SMITH |
| --- |
| JohnSmith |

这显示了一个包含组合值的单列。但是，值之间没有空格，因为我们在连接时没有指定。连接不会自动添加空格，所以如果你想要一个空格，就需要手动添加。让我们试试这个查询：

```sql
SELECT 'John ' || 'Smith' AS full_name
FROM dual;
```

我还添加了一个列别名，以使结果更易于阅读。结果是：

| FULL_NAME |
| --- |
| John Smith |

现在显示的值中间有了一个空格。你也可以连接表中的值。假设你需要组合 `employee` 表中的 `last_name` 和 `salary` 值：

```sql
SELECT id,
last_name || salary
FROM employee;
```

结果将是：

| ID | LAST_NAME&#124;&#124;SALARY |
| --- | --- |
| 1 | JONES30000 |
| 2 | SMITH35000 |
| 3 | KING45000 |
| 4 | SIMPSON52000 |
| 5 | ANDERSON31000 |
| 6 | COOPER |
| 7 | ADAMS |
| 8 | SMITH62000 |
| 9 | PATRICK40000 |

这个结果集的主要问题是很难读取连接后的值，因为名称直接连到了数字值上。

如何解决名称直接连到数字上的问题？你可以添加一个空格和一个逗号。这可以通过添加第三个连接值来实现。这意味着你的值将显示 `last_name`，然后是一个逗号和一个空格，接着是薪水。执行此操作的查询是：

```sql
SELECT id,
last_name || ', ' || salary
FROM employee;
```

这个查询的结果是：

| ID | LAST_NAME&#124;&#124;,&#124;&#124;SALARY |
| --- | --- |
| 1 | JONES, 30000 |
| 2 | SMITH, 35000 |
| 3 | KING, 45000 |
| 4 | SIMPSON, 52000 |
| 5 | ANDERSON, 31000 |
| 6 | COOPER, |
| 7 | ADAMS, |
| 8 | SMITH, 62000 |
| 9 | PATRICK, 40000 |

这些结果更易于阅读。你添加了一个空格和一个逗号，结果现在位于单列中。如果你需要组合字符串值，或者字符串与数字或日期，都可以使用这种连接技术。

### 字符串函数

Oracle 还提供了一些处理字符串的函数，就像你之前学习处理数字时一样。这类函数有很多，它们允许你对字符串值执行各种操作，例如：

*   更改大小写（大写和小写）
*   在其他字符串中查找字符串并提取位置或字符串本身
*   在字符串的开头或结尾添加或删除字符，例如空格
*   转换 ASCII 码

在本节中，你将看到如何使用其中的几个函数。

#### 更改大小写

在本节中，你将学习两个用于更改字符串大小写的函数：一个用于转换为大写，一个用于转换为小写。这些函数用于将你的字符串或字符数据转换为一致的格式。

为什么你希望数据保持一致的格式？也许你允许用户以混合大小写的形式在表单中输入数据，而你希望以相同的方式存储它，以便日后更易于显示和搜索。

要将字符串值转换为大写，你需要使用 `UPPER` 函数。`UPPER` 函数的语法如下：

```sql
UPPER (input_string)
```

它只接受一个参数，即 `input_string`，也就是你想要转换为大写的字符串。该函数返回相同的大写形式值。

你可以通过查询 `office` 表来了解此函数的工作原理，因为 `address` 字段包含大写和小写的数据：

```sql
SELECT id,
address,
UPPER(address)
FROM office;
```

这个查询的结果是：

| ID | ADDRESS | UPPER(ADDRESS) |
| --- | --- | --- |
| 1 | 123 Smith Street | 123 SMITH STREET |
| 2 | 45 Main Street | 45 MAIN STREET |
| 3 | 10 Collins Road | 10 COLLINS ROAD |

你可以看到 `id` 列、原始的 `address`，以及应用 `UPPER` 函数后的 `address`。内容读起来一样，但只是转换成了大写形式（`123 SMITH STREET` 而不是 `123 Smith Street`）。任何已经是大写的字母保持不变，小写字母被转换为大写字母，数字则保持不变。

你可以使用类似的查询将数据转换为小写。Oracle 的 `LOWER` 函数可以做到这一点，其语法与 `UPPER` 函数类似：

```sql
LOWER (input_string)
```

该函数将所有字符转换为小写字符。对 `office` 表执行此操作的查询如下：

```sql
SELECT id,
address,
LOWER(address)
FROM office;
```

结果是：

| ID | ADDRESS | LOWER(ADDRESS) |
| --- | --- | --- |
| 1 | 123 Smith Street | 123 smith street |
| 2 | 45 Main Street | 45 main street |
| 3 | 10 Collins Road | 10 collins road |

这些结果显示了地址的小写形式。数字和已有的小写字符保持不变，而大写字符被转换为小写。

#### 检查相同大小写的匹配

我见过 `UPPER` 和 `LOWER` 函数的一种常见用法是在比较数据或在表中查找数据时。这通常在 `WHERE` 子句中完成。

为了演示这个例子，你需要向 `employee` 表中插入一条新记录，员工名为“Jones”。

```sql
INSERT INTO employee (id, last_name, salary, office_id)
VALUES (10, 'Jones', 42000, 2);
```

当你在表上运行 `SELECT` 语句时，它看起来像这样：

```sql
SELECT id,
last_name,
salary,
office_id
FROM employee;
```

| ID | LAST_NAME | SALARY | OFFICE_ID |
| --- | --- | --- | --- |
| 1 | JONES | 30000 | 1 |
| 2 | SMITH | 35000 | 2 |
| 3 | KING | 45000 | 2 |
| 4 | SIMPSON | 52000 | 1 |
| 5 | ANDERSON | 31000 | (null) |
| 6 | COOPER | (null) | 1 |
| 7 | ADAMS | (null) | 3 |
| 8 | SMITH | 62000 | 3 |
| 9 | PATRICK | 40000 | 1 |
| 10 | Jones | 42000 | 2 |

现在假设你想查找所有名为 `JONES` 的员工。你已经看过一些记录，并了解到 `employee` 的 `last_names` 是以大写形式存储的，所以你的查询是这样的：

```sql
SELECT id,
last_name,
salary,
office_id
FROM employee
WHERE last_name = 'JONES';
```

当你运行这个查询时，得到的结果是：

| ID | LAST_NAME | SALARY | OFFICE_ID |
| --- | --- | --- | --- |
| 1 | JONES | 30000 | 1 |

那么你刚插入的 `Jones` 新记录呢？它不应该也显示出来吗？在这个案例中，Oracle 数据库显示了正确的结果。这是因为 Oracle 中的 `WHERE` 子句是区分大小写的。这意味着 `"JONES"` 不等于 `"Jones"`，也不等于 `"jones"`。


#### 注意

Oracle 中的 WHERE 子句是区分大小写的，这意味着"JONES"、"Jones"和"jones"都是不同的值。

如何让查询同时显示这两种值呢？你可以使用`UPPER`或`LOWER`函数。与其查找"`last_name`为 JONES 的所有记录"，不如查找"`last_name`大写形式为 JONES 的所有记录"。你也可以查找"`last_name`小写形式为 jones 的所有记录"。它们会显示相同的结果。

实现此功能的查询如下：

```
SELECT id,
last_name,
salary,
office_id
FROM employee
WHERE UPPER(last_name) = 'JONES';
```

这将比较`last_name`值的大写形式与 JONES 的值。将显示以下结果：

| ID | LAST_NAME | SALARY | OFFICE_ID |
| --- | --- | --- | --- |
| 1 | JONES | 20000 | 1 |
| 10 | Jones | 42000 | 2 |

两条"Jones"记录都被显示出来了：一条是大写形式的，另一条是混合大小写形式的。

你也可以通过将两边都转换为小写来获得相同的结果：

```
SELECT id,
last_name,
salary,
office_id
FROM employee
WHERE LOWER(last_name) = 'jones';
```

应该使用哪个函数呢？对我个人而言，我偏好使用`UPPER`。在这种情况下它也更有意义，因为表中大多数值都是大写的。但两者会显示相同的结果。

#### 获取字符串的一部分

另一个对字符串值进行的常见操作是获取字符串的某个部分。这个概念称为`子串`，因为较大字符串的较小部分被称为子串。Oracle 函数`SUBSTR`可以让你从一个字符串中提取一个子串。其语法如下：

```
SUBSTR (string, start_position [, length])
```

此函数有三个参数：

*   `string`：要查找的字符串
*   `start_position`：要提取的子串的起始位置
*   `length`：要返回的子串的长度

在上面的语法中，你会注意到一些方括号"["和"]"。它们表示该参数是可选的。你不必指定`length`参数。这意味着你可以用两种方式运行此函数：

```
SUBSTR (string, start_position)
SUBSTR (string, start_position, length)
```

如果你不添加`length`参数，默认行为是返回字符串的剩余部分。

让我们在`office`表上看一个例子。你可能想要找到`address`字段的前两个字符。你可以使用`SUBSTR`函数，起始位置为 1，长度为 2 来实现。

```
SELECT id,
address,
SUBSTR(address, 1, 2)
FROM office;
```

这将显示以下结果：

| ID | ADDRESS | SUBSTR(ADDRESS,1,2) |
| --- | --- | --- |
| 1 | 123 Smith Street | 12 |
| 2 | 45 Main Street | 45 |
| 3 | 10 Collins Road | 10 |

它提取了`address`字段的前两个字符并显示在新列中。

你可以尝试不带`length`参数的同一函数。假设你想从第 5 个字符开始提取`address`的剩余部分。你的查询可能如下所示：

```
SELECT id,
address,
SUBSTR(address, 5)
FROM office;
```

结果将是：

| ID | ADDRESS | SUBSTR(ADDRESS,5) |
| --- | --- | --- |
| 1 | 123 Smith Street | Smith Street |
| 2 | 45 Main Street | ain Street |
| 3 | 10 Collins Road | ollins Road |

该函数提取了从第 5 个位置开始的整个字符串。

在 Oracle 中，还有更多的字符串函数，可以让你对字符串进行各种转换。这里介绍的只是其中一部分。


### 日期计算

在本章中，你已经学习了如何对字符串和数字进行基本操作。你也可以对日期进行基本操作。给日期加上或减去天数是我见过的最常见的操作，因此在本节中你将学习如何做到这一点。

`sales_meeting` 表有一个 `meeting_date` 列，用于存储日期值。

| ID | EMPLOYEE_ID | COMPANY | MEETING_DATE |
| --- | --- | --- | --- |
| 1 | 5 | ABC Construction | 10/Aug/2018 |
| 2 | 2 | BW Signage | 21/Aug/2018 |
| 3 | (null) | WXC Services | 23/Aug/2018 |

你只需向日期值加上或减去一个数字，即可为该日期值加上或减去天数。例如，要在 `meeting_date` 值上加 7 天，你可以指定 `meeting_date + 7`。执行此操作的查询如下：

```sql
SELECT id,
meeting_date,
meeting_date + 7
FROM sales_meeting;
```

运行此查询后，你将得到以下结果。

| ID | MEETING_DATE | MEETING_DATE+7 |
| --- | --- | --- |
| 1 | 10/Aug/2018 | 17/Aug/2018 |
| 2 | 21/Aug/2018 | 28/Aug/2018 |
| 3 | 23/Aug/2018 | 30/Aug/2018 |

这些日期每一个都加上了 7 天。如果需要，你可以加更长的周期或减去天数。下面是一个查询，显示相同的日期加上 14 天和减去 30 天的结果。

```sql
SELECT id,
meeting_date,
meeting_date + 14,
meeting_date - 30
FROM sales_meeting;
```

结果如下：

| ID | MEETING_DATE | MEETING_DATE+14 | MEETING_DATE-30 |
| --- | --- | --- | --- |
| 1 | 10/Aug/2018 | 24/Aug/2018 | 11/Jul/2018 |
| 2 | 21/Aug/2018 | 4/Sep/2018 | 22/Jul/2018 |
| 3 | 23/Aug/2018 | 6/Sep/2018 | 24/Jul/2018 |

这是查找未来或过去日期的简单方法。你也可以用它来查找未来或过去的小时。请记住，`DATE` 数据类型也包含时间。我们也可以使用相同的过程以及一个名为 `TO_CHAR` 的函数来显示时间。

`meeting_date` 列存储日期和时间值，但只显示日期。查看时间值的一种方法是使用 `TO_CHAR` 函数，该函数将日期值转换为字符串或字符值。`TO_CHAR` 函数语法如下：

```sql
TO_CHAR (input_value [, format_mask [, nls_parameter] ] )
```

这包含几个参数：

*   `input_value`：要转换为字符值的值
*   `format_mask`：`input_value` 应显示的格式。这是可选的，如果省略，将使用默认格式。
*   `nls_parameter`：此可选参数允许你指定任何特定于位置的值。

首先，让我们将 `meeting_date` 值显示为日期和时间值：

```sql
SELECT id,
meeting_date,
TO_CHAR(meeting_date, 'DD/MON/YYYY HH:MI:SS AM') AS meeting_datetime
FROM sales_meeting;
```

第二个参数是格式掩码。这是日期值将显示的格式。它包括：

*   `DD`，表示两位数的日期（例如，27），后跟正斜杠 "/"
*   然后是 `MON`，表示三位字母的月份（例如，AUG），后跟正斜杠 "/"
*   然后是四位数的年份（例如，2018），后跟一个空格
*   然后是两位数的小时，后跟冒号 ":"
*   然后是两位数的分钟，后跟冒号 ":"
*   然后是两位数的秒，后跟一个空格
*   然后是 `AM` 或 `PM` 指示符

此查询的结果是：

| ID | MEETING_DATE | MEETING_DATETIME |
| --- | --- | --- |
| 1 | 10/Aug/18 | 10/Aug/2018 12:00:00 AM |
| 2 | 21/Aug/18 | 21/Aug/2018 12:00:00 AM |
| 3 | 23/Aug/18 | 23/Aug/2018 12:00:00 AM |

如你所见，`meeting_datetime` 列显示了包含时间的 `meeting_date` 值。时间是午夜，因为当未指定时间时，这是默认值。

现在，假设你想给这些日期加上 6 小时。之前你向日期加上整数来增加一天。你可以通过加上数字然后除以 24 来向日期加上小时。例如，向日期加上 1/24 将使其增加 1 小时。

要为每个 `meeting_date` 值加上 6 小时，你的查询如下：

```sql
SELECT id,
TO_CHAR(meeting_date, 'DD/MON/YYYY HH:MI:SS AM') AS meeting_datetime,
TO_CHAR(meeting_date + 6/24, 'DD/MON/YYYY HH:MI:SS AM') AS meeting_datetime_future
FROM sales_meeting;
```

此查询的结果是：

| ID | MEETING_DATE | MEETING_DATETIME |
| --- | --- | --- |
| 1 | 10/Aug/18 | 10/Aug/2018 06:00:00 AM |
| 2 | 21/Aug/18 | 21/Aug/2018 06:00:00 AM |
| 3 | 23/Aug/18 | 23/Aug/2018 06:00:00 AM |

这显示了当前的日期和时间，然后是加上 6 小时后的日期和时间。所有时间都显示为早上 6 点，因为你在提到 `meeting_date` 列的地方加上了“+6/24”。它被添加到函数内部，因为我们希望在 `TO_CHAR` 执行格式化之前先完成加 6 小时的操作。

这些只是你可以在日期和数字之间进行的加减操作的一些例子。使用日期函数可以对日期进行更多操作。

### 日期函数

在本节中，你将学习如何使用两个最常见的日期函数。第一个函数让你查找当前日期和时间。

#### 当前日期和时间

在很多情况下，你可能想知道当前的日期和时间是什么，例如：

*   在应用程序中记录事件发生的时间
*   记录错误发生的时间
*   向用户显示当前日期和时间

有几个函数可以显示当前日期和时间，但我见过最常用的是 `SYSDATE`。此函数将返回数据库服务器的当前日期和时间。这是一种快速简便地获取当前日期和时间的方法。它不涉及任何参数；只需在查询中添加单词 `SYSDATE`。要获取当前日期和时间，请使用以下查询：

```sql
SELECT SYSDATE
FROM dual;
```

你的结果将显示当前日期，例如：

| SYSDATE |
| --- |
| 30/AUG/2018 |

`SYSDATE` 函数实际上包含一个时间部分，但像之前的 `meeting_date` 列一样，默认情况下不显示。你可以通过对 `SYSDATE` 列使用 `TO_CHAR` 函数来显示日期和时间。是的，你可以在其他函数内部使用函数，这是在许多情况下获取所需数据的常用方法。

你的查询可能如下所示：

```sql
SELECT
TO_CHAR(SYSDATE, 'DD/MON/YYYY HH:MI:SS AM') AS time_now
FROM dual;
```

此查询的结果将显示类似以下内容：

| SYSDATE |
| --- |
| 30/AUG/2018 12:30:14 PM |

这显示了当前日期以及时间。每次运行此查询时，时间都会更改为当前时间。

如果需要将当前日期插入列中，可以在 `INSERT` 语句中使用此函数。例如，像这样的查询将插入一条新记录，其会议日期和时间为当前时间：

```sql
INSERT INTO sales_meeting (id, employee_id, company, meeting_date)
VALUES (4, 7, 'McMahon', SYSDATE);
```

因此，无需指定实际的日期值，而是使用 `SYSDATE` 函数。你无需在你的数据库上运行此查询。



#### 添加月份

在本章前面，你已经学习了如何通过简单地在日期上添加一个数字来增加天数：

```sql
SELECT meeting_date + 7
FROM sales_meeting;
```

然而，如果你需要向一个日期添加几个月呢？三个月、六个月，还是 24 个月？你可以通过添加天数来实现。但你会选择哪个数字：30，还是 31？不同的月份有不同的天数，所以添加 30 天这个方法并非在所有情况下都适用。如果你需要日期精确地变为未来的某个月份，这种方法就行不通了。

Oracle 中有一个名为 `ADD_MONTHS` 的函数，它允许你向一个日期添加指定的月份数：

```sql
ADD_MONTHS (input_date, number_of_months)
```

这个函数包含两个参数。`input_date` 是你想要向其添加月份的日期，而 `number_of_months` 是一个表示要添加的月份数的数字。这个函数会将指定的月份数添加到日期中，并确保日保持不变，只改变月份（或年份）。

假设你需要为你的 `meeting_dates` 添加六个月。你的查询将如下所示：

```sql
SELECT
    meeting_date,
    ADD_MONTHS(meeting_date, 6) AS new_date
FROM sales_meeting;
```

该查询的结果是：

| ID | MEETING_DATE | NEW_DATE |
| --- | --- | --- |
| 1 | 10/Aug/2018 | 10/Feb/2019 |
| 2 | 21/Aug/2018 | 21/Feb/2019 |
| 3 | 23/Aug/2018 | 23/Feb/2019 |

`new_date` 列，使用了 `ADD_MONTHS` 函数，显示了 6 个月后的会议日期。它保持了相同的日，但月份和年份已经改变。

如果你想向一个日期添加特定的年数，你可以通过添加等效的月份数来实现。这是可行的，因为一年总是有 12 个月。你可以通过添加年数对应的月份数（例如，24 代表 2 年），或者将年数乘以 12（例如，2 * 12）来完成。这个查询包含了两种示例：

```sql
SELECT
    meeting_date,
    ADD_MONTHS(meeting_date, 24) AS new_date,
    ADD_MONTHS(meeting_date, 2*12) AS new_date_ex2
FROM sales_meeting;
```

结果是：

| ID | MEETING_DATE | NEW_DATE | NEW_DATE_EX2 |
| --- | --- | --- | --- |
| 1 | 10/Aug/2018 | 10/Aug/2020 | 10/Aug/2020 |
| 2 | 21/Aug/2018 | 21/Aug/2020 | 21/Aug/2020 |
| 3 | 23/Aug/2018 | 23/Aug/2020 | 23/Aug/2020 |

`new_date` 和 `new_date_ex2` 两列都显示了 `meeting_date` 值两年后的结果。这是向现有日期值添加月份或年份的一种简单方法。

### 总结

Oracle 中的函数允许你对查询中的列和值执行计算。你可以使用数学符号如 `+`、`-`、`*` 和 `/` 执行一些基本操作。你也可以使用 `||` 连接字符串值。如果你想执行更复杂的操作，可以使用 Oracle 提供的众多函数之一。

Oracle 函数根据它们处理的数据类型分为三组：数字、字符串和日期。这些函数接受参数，执行一个过程，并返回一个值。它们可以用于查询的许多不同部分，例如 `SELECT` 查询中的 `WHERE` 子句，以及 `INSERT` 和 `UPDATE` 查询中。

## 23. 编写条件逻辑

许多编程语言都包含执行条件逻辑的能力，即“如果这个条件为真，则执行某事，否则执行另一事”。SQL 与其他编程语言不同，它不包含很多特性，如变量和循环。然而，SQL 确实具有使用条件逻辑的能力。

这在其他语言中通常被称为 `IF` 语句，其形式如下：

```sql
if (some condition) {
    do_something;
else {
    do_something_else;
}
```

在 SQL 中，你可以做同样类型的事情，但做法略有不同。你可以使用一种称为 `CASE` 语句的语句。

### CASE 语句

在 SQL 中，`CASE` 语句允许你在查询中执行这种“如果-那么-否则”逻辑或条件逻辑。它类似于其他语言中的 `IF` 语句。它被称为语句（statement）而不是函数（function），因为它是一组关键字的组合，而不是一个带有括号内几个参数的关键字。

`CASE` 语句可以用于函数可以使用的许多相同位置，例如：

*   `SELECT` 语句的 `SELECT`、`WHERE` 和 `ORDER BY` 子句
*   `UPDATE` 语句的 `UPDATE`、`SET` 和 `WHERE` 子句
*   `DELETE` 语句的 `WHERE` 子句

它非常适合根据同一行中的另一个值来显示特定的值。因为每一行中的值可能不同，所以有时添加一个 `CASE` 语句来查看它们并在不同情况下显示一个值是很有用的。

`CASE` 语句是什么样子的？

```sql
CASE [expression]
    WHEN condition_1 THEN result_1
    WHEN condition_2 THEN result_2
    ...
    WHEN condition_n THEN result_n
    ELSE result
END case_name
```

该语句的参数是：

*   `expression`：这是 `CASE` 语句将要查找的表达式或条件。如果你将其与 `IF` 语句比较，这相当于 `IF` 后面条件的第一部分。这是一个可选参数。
*   `condition_1/2/n`：这是上述表达式的可能结果。根据你想要处理的不同可能值的数量，你可以在 `CASE` 语句中放入任意多个这样的条件。
*   `result_1/2/n`：这是如果满足上述条件则应显示的值。它位于 `THEN` 关键字之后，相当于其他编程语言中 `IF-THEN-ELSE` 语句的 `THEN` 部分。
*   `ELSE result`：如果 `CASE` 语句中没有任何条件被满足，则显示此值。它相当于 `IF-THEN-ELSE` 语句的 `ELSE` 部分。
*   `case_name`：这是一个可选参数，指示当它在查询中使用时该列应被称为什么。它类似于你在前面章节中使用过的列别名。

这是一个相当长的语句，但如果你需要在查询中执行这种逻辑，它非常有用。在 Oracle SQL 中，你可以使用两种方式的 `CASE` 语句：简单的 `CASE` 语句和搜索的 `CASE` 语句。



### 简单 CASE 语句

### 基本示例

一个*简单 CASE 语句*是指使用了表达式参数的情况。它会对每一行的 `WHEN` 条件检查同一个表达式。

例如，你的语句可能如下所示：

```sql
CASE last_name
WHEN 'JONES' THEN 'The name is Jones'
WHEN 'SMITH' THEN 'The name is Smith'
WHEN 'KING' THEN 'The name is King'
ELSE 'The name is something else'
END namecheck
```

这条语句会检查 `last_name` 列，如果它等于 JONES、SMITH 或 KING，则显示不同的值。如果它不匹配其中任何一个，则显示另一个值。你不需要指定 `WHEN last_name = 'JONES'` 或 `WHEN last_name = 'SMITH'`，因为使用这种方法时，`WHEN` 会自动应用到语句开头提到的 `last_name` 上。

这里也没有提到表名或 `FROM` 子句，因为假定这条语句是另一个包含 `FROM` 子句的语句（例如 `SELECT`）的一部分。例如，这条 `CASE` 语句可以被添加到一个 `SELECT` 查询中：

| LAST_NAME | NAMECHECK |
| --- | --- |
| JONES | The name is Jones |
| SMITH | The name is Smith |
| KING | The name is King |
| SIMPSON | The name is something else |
| ANDERSON | The name is something else |
| COOPER | The name is something else |
| ADAMS | The name is something else |
| SMITH | The name is Smith |
| PATRICK | The name is something else |
| Jones | The name is something else |

```sql
SELECT
last_name,
CASE last_name
WHEN 'JONES' THEN 'The name is Jones'
WHEN 'SMITH' THEN 'The name is Smith'
WHEN 'KING' THEN 'The name is King'
ELSE 'The name is something else'
END namecheck
FROM employee;
```

你可以看到其中一些结果显示了正确描述名字的句子，而另一些显示了“something else”。最后一行显示的是“Jones”，但这个 `CASE` 语句将其显示为“something else”。这是因为 `CASE` 语句和 `WHEN` 子句是区分大小写的，就像你在之前章节中学到的 `WHERE` 子句一样。这意味着 “Jones” 不等于 “JONES”。

### 使用函数处理大小写

好消息是，你可以在 `CASE` 语句中使用函数。你可以在 `CASE` 语句内部使用 `UPPER` 函数将 `last_name` 转换为大写值进行检查：

| LAST_NAME | NAMECHECK |
| --- | --- |
| JONES | The name is Jones |
| SMITH | The name is Smith |
| KING | The name is King |
| SIMPSON | The name is something else |
| ANDERSON | The name is something else |
| COOPER | The name is something else |
| ADAMS | The name is something else |
| SMITH | The name is Smith |
| PATRICK | The name is something else |
| Jones | The name is Jones |

```sql
SELECT
last_name,
CASE UPPER(last_name)
WHEN 'JONES' THEN 'The name is Jones'
WHEN 'SMITH' THEN 'The name is Smith'
WHEN 'KING' THEN 'The name is King'
ELSE 'The name is something else'
END namecheck
FROM employee;
```

使用这个 `UPPER` 函数已将“Jones”转换为“JONES”，并满足了 CASE 语句中 WHEN ‘JONES’ 的条件。

### 多条件简单 CASE

让我们看另一个例子，你需要判断一个 `salary` 是否等于 20000 或 40000，如果是，则符合加薪条件。你的查询会是这样：

```sql
SELECT id,
last_name,
salary,
CASE salary
WHEN 20000 THEN 'Eligible'
WHEN 40000 THEN 'Eligible'
ELSE 'Not Eligible'
END salary_check
FROM employee;
```

这个查询检查 `salary` 列。如果等于 20000，则显示 Eligible 值。如果等于 40000，也显示同样的值。如果它不匹配这两个值中的任何一个，则显示 Not Eligible。如果你运行这个查询，你会得到这个结果：

| ID | LAST_NAME | SALARY | SALARY_CHECK |
| --- | --- | --- | --- |
| 1 | JONES | 30000 | Not Eligible |
| 2 | SMITH | 35000 | Not Eligible |
| 3 | KING | 45000 | Not Eligible |
| 4 | SIMPSON | 52000 | Not Eligible |
| 5 | ANDERSON | 31000 | Not Eligible |
| 6 | COOPER | (null) | Not Eligible |
| 7 | ADAMS | (null) | Not Eligible |
| 8 | SMITH | 62000 | Not Eligible |
| 9 | PATRICK | 40000 | Eligible |
| 10 | Jones | 42000 | Not Eligible |

这显示了 `salary` 为 20000 或 40000 的员工为 Eligible，其他记录为 Not Eligible。然而，`CASE` 语句中的两个 `WHEN` 子句看起来非常相似：

```sql
WHEN 20000 THEN 'Eligible'
WHEN 40000 THEN 'Eligible'
```

我们可以尝试使用 `OR` 关键字将它们合并为一个条件，就像在 `WHERE` 子句中那样。这使代码更易于阅读和编写，因为代码更少。

```sql
WHEN 20000 OR 40000 THEN 'Eligible'
```

让我们将这个添加到整个查询中并运行它。

```sql
SELECT id,
last_name,
salary,
CASE salary
WHEN 20000 OR 40000 THEN 'Eligible'
ELSE 'Not Eligible'
END salary_check
FROM employee;
```

我们得到了这个错误：

```
ORA-00905: missing keyword
00905. 00000 -  "missing keyword"
*Cause:
*Action:
Error at Line: 28 Column: 12
```

这是否意味着我们不能在 `CASE` 语句中使用 `OR` 关键字？我们可以，但不能在简单 CASE 语句中使用。对于比这些简单检查更复杂的任何情况，我们需要使用另一种类型的 CASE 语句。


### 搜索式 CASE 语句

Oracle 中的另一种 `CASE` 语句是“搜索式 `CASE` 语句” (`searched case statement`)。搜索式 `CASE` 语句与简单 `CASE` 语句的区别在于指定表达式的位置。在简单 `CASE` 语句中，`expression` 紧跟在 `CASE` 关键字之后。在搜索式 `CASE` 语句中，`expression` 位于每个 `WHEN` 子句中。

搜索式 `CASE` 语句允许使用更复杂的条件，例如多个条件、通配符检查、大于和小于检查以及对多个字段的检查。你可以看到以下示例，它使用了与之前简单 `CASE` 语句相同的逻辑，但采用了搜索式 `CASE` 语句的格式：

```sql
CASE
WHEN last_name = 'JONES' THEN 'The name is Jones'
WHEN last_name = 'SMITH' THEN 'The name is Smith'
WHEN last_name = 'KING'  THEN 'The name is King'
ELSE 'The name is something else'
END namecheck
```

这种方法需要多打一点字，但它更灵活。让我们将其加入 `SELECT` 查询中，并像之前一样加入 `UPPER` 函数。

| LAST_NAME | NAMECHECK            |
| --------- | -------------------- |
| JONES     | The name is Jones    |
| SMITH     | The name is Smith    |
| KING      | The name is King     |
| SIMPSON   | The name is something else |
| ANDERSON  | The name is something else |
| COOPER    | The name is something else |
| ADAMS     | The name is something else |
| SMITH     | The name is Smith    |
| PATRICK   | The name is something else |
| Jones     | The name is Jones    |

```sql
SELECT
last_name,
CASE
  WHEN UPPER(last_name) = 'JONES' THEN 'The name is Jones'
  WHEN UPPER(last_name) = 'SMITH' THEN 'The name is Smith'
  WHEN UPPER(last_name) = 'KING'  THEN 'The name is King'
  ELSE 'The name is something else'
END namecheck
FROM employee;
```

结果显示与之前的示例相同。`namecheck` 列根据我们编写的 `CASE` 语句显示简短描述。对于一个简单的检查，这种方法可能显得有点过于麻烦，因此在这种情况下，也许简单 `CASE` 语句更受青睐。然而，有些时候搜索式 `CASE` 语句更好。

让我们看另一个例子。假设你被要求根据几位员工当前的薪资来确定他们是否有资格加薪。如果员工的薪水低于 40000，则显示为“低薪”。如果他们的薪水在 40000 到 50000 之间，则显示为“中等薪资”，任何 50000 或更多的应显示为“高薪”。

你的查询将如下所示：

```sql
SELECT id,
       last_name,
       salary,
       CASE
         WHEN salary < 40000                    THEN 'Low salary'
         WHEN salary >= 40000 AND salary < 50000 THEN 'Medium salary'
         WHEN salary >= 50000                   THEN 'High salary'
       END salary_check
FROM employee;
```

我们这里没有添加 `ELSE`。它是一个可选子句。让我们看看运行此查询会发生什么。

| ID  | LAST_NAME | SALARY | SALARY_CHECK   |
| --- | --------- | ------ | -------------- |
| 1   | JONES     | 30000  | Low salary     |
| 2   | SMITH     | 35000  | Low salary     |
| 3   | KING      | 45000  | Medium Salary  |
| 4   | SIMPSON   | 52000  | High salary    |
| 5   | ANDERSON  | 31000  | Low salary     |
| 6   | COOPER    | (null) | (null)         |
| 7   | ADAMS     | (null) | (null)         |
| 8   | SMITH     | 62000  | High salary    |
| 9   | PATRICK   | 40000  | (null)         |
| 10  | Jones     | 42000  | Medium salary  |

你可以看到，一些记录的值为“低薪”、“中等薪资”或“高薪”。其他记录显示为 `NULL`。这是因为他们没有满足 `CASE` 语句中提到的条件。

那么 `salary` 为 40000 的那条记录呢？为什么它没有显示一个值？这是因为你使用的条件是这样的：

```sql
CASE
  WHEN salary < 40000                    THEN 'Low salary'
  WHEN salary > 40000 AND salary < 50000 THEN 'Medium salary'
  WHEN salary >= 50000                   THEN 'High salary'
END salary_check
```

第一个 `WHEN` 子句查找 `salary` 值小于 40000。第二个 `WHEN` 子句查找 `salary` 值大于 40000。其中没有提到 `salary` 等于 40000。有两种方法可以解决此问题：

*   更改符号以包含“等于”
*   更改 `WHEN` 子句使其重叠

第一种解决方法是更改符号以包含“等于”。我们之前使用的描述是：“如果员工的薪水低于 40000，则显示为低薪。如果他们的薪水在 40000 到 50000 之间，则显示为中等薪资。”这表明 `salary` 值为 40000 应标记为‘中等薪资’。你可以更改查询以加入“等于”：

```sql
SELECT id,
       last_name,
       salary,
       CASE
         WHEN salary < 40000                       THEN 'Low salary'
         WHEN salary >= 40000 AND salary <= 50000  THEN 'Medium salary'
         WHEN salary > 50000                       THEN 'High salary'
       END salary_check
FROM employee;
```

| ID  | LAST_NAME | SALARY | SALARY_CHECK   |
| --- | --------- | ------ | -------------- |
| 1   | JONES     | 30000  | Low salary     |
| 2   | SMITH     | 35000  | Low salary     |
| 3   | KING      | 45000  | Medium salary  |
| 4   | SIMPSON   | 52000  | High salary    |
| 5   | ANDERSON  | 31000  | Low salary     |
| 6   | COOPER    | (null) | (null)         |
| 7   | ADAMS     | (null) | (null)         |
| 8   | SMITH     | 62000  | High salary    |
| 9   | PATRICK   | 40000  | Medium salary  |
| 10  | Jones     | 42000  | Medium salary  |

现在显示了正确的值。最后两个 `NULL` 值显示的原因是薪资值为 `NULL`。

解决此问题的另一种方法是利用 Oracle 处理 `CASE` 语句的方式。Oracle 会找到第一个为 `TRUE` 的条件，并将其用于你的 `CASE` 语句。这意味着如果第一个和第二个 `WHEN` 子句都为真，则将使用第一个 `WHEN` 子句的结果。你可以更新查询以使用此方法：

```sql
SELECT id,
       last_name,
       salary,
       CASE
         WHEN salary <= 40000              THEN 'Low salary'
         WHEN salary <= 50000              THEN 'Medium salary'
         ELSE 'High salary'
       END salary_check
FROM employee;
```

| ID  | LAST_NAME | SALARY | SALARY_CHECK   |
| --- | --------- | ------ | -------------- |
| 1   | JONES     | 30000  | Low salary     |
| 2   | SMITH     | 35000  | Low salary     |
| 3   | KING      | 45000  | Medium salary  |
| 4   | SIMPSON   | 52000  | High salary    |
| 5   | ANDERSON  | 31000  | Low salary     |
| 6   | COOPER    | (null) | (null)         |
| 7   | ADAMS     | (null) | (null)         |
| 8   | SMITH     | 62000  | High salary    |
| 9   | PATRICK   | 40000  | Medium salary  |
| 10  | Jones     | 42000  | Medium salary  |

如你所见，显示的值与之前的示例相同。`CASE` 语句是一个非常强大的语句。

在 SQL 中处理条件逻辑还有另一种方法，那就是 `DECODE` 函数。

### DECODE 函数

`DECODE` 是 Oracle 提供的一个函数，用于执行条件逻辑。它与 `CASE` 语句类似。该函数的语法如下所示：

```
DECODE (expression, search, result [, search, result]... [,default] )
```

此 Oracle `DECODE` 函数的参数为：

*   `expression`（必填）：这是函数中要比较的值。
*   `search`（必填）：这是用于与表达式进行比较的值。
*   `result`（必填）：如果搜索值与表达式值匹配，则返回此值。可以有多组搜索和结果值，结果值对应于前一个搜索值。
*   `default`（可选）：如果没有一个搜索值与表达式匹配，则返回此值。如果未提供默认值，则使用 `NULL`。

如果你将它与 `IF-THEN-ELSE` 语句进行比较，它看起来会是这样：

```
IF (expression = search) THEN result
[ELSE IF (expression = search) THEN result]
ELSE default
END IF
```

你也可以将之前的 `CASE` 语句写成 `DECODE` 函数。它看起来像这样：

```
SELECT
last_name,
DECODE(UPPER(last_name), 'JONES', 'The name is Jones', 'SMITH', 'The name is Smith', 'KING', 'The name is King', 'The name is something else') AS namecheck
FROM employee;
```

这个函数全部在一行上，所以可以格式化得更好一些：

```
SELECT
last_name,
DECODE(UPPER(last_name),
'JONES', 'The name is Jones',
'SMITH', 'The name is Smith',
'KING', 'The name is King',
'The name is something else') AS namecheck
FROM employee;
```

你可以运行此查询并得到以下结果。

| LAST_NAME | NAMECHECK |
| --- | --- |
| JONES | 名字是 Jones |
| SMITH | 名字是 Smith |
| KING | 名字是 King |
| SIMPSON | 名字是其他 |
| ANDERSON | 名字是其他 |
| COOPER | 名字是其他 |
| ADAMS | 名字是其他 |
| SMITH | 名字是 Smith |
| PATRICK | 名字是其他 |
| Jones | 名字是 Jones |

此示例中的 `DECODE` 函数看起来类似于 `CASE` 语句，并显示相同的结果。但是，在某些情况下，`DECODE` 函数可能会有点混乱，例如使用 `LIKE` 关键字进行部分匹配或使用大于或小于符号时。

`DECODE` 只能检查完全匹配。你可以使用一些运算符和 `SIGN` 函数来检查是否大于或小于其他值。之前我们有一个对薪资进行分组的 `CASE` 语句：

```
CASE
WHEN salary = 50000 THEN 'High salary'
END salary_check
```

要用 `DECODE` 语句编写相同的逻辑，我们的函数将如下所示：

```
DECODE(SIGN(40000-salary),1,'Low salary', -1, 'Medium or high salary', 'Unsure')
```

此函数首先从 `40000` 中减去 `salary`，并将其括在 `SIGN` 函数中。如果结果为正数，此函数将返回 `1`，如果结果为负数，则返回 `-1`。因此，如果 `salary` 小于 `40000`，则结果值将为正数，`SIGN` 返回 `1`。

然后，你需要将 'Medium or high salary' 替换为另一个 `DECODE` 函数来检查 `50000` 的值：

```
DECODE(SIGN(40000-salary),1,'Low salary', -1, DECODE(SIGN(50000-salary),1,'Medium salary', -1, 'High salary', 'Unsure'),'Unsure')
```

如你所见，这变得相当混乱，函数里面套着函数。你或其他人也很难理解它的含义。

因此，在这种情况下，它可能有效，但使用 `CASE` 语句会更好。

### CASE 还是 DECODE？

如果 `CASE` 语句和 `DECODE` 函数功能相同，应该使用哪一个？我建议在所有情况下都使用 `CASE` 语句，原因如下：

*   `CASE` 是在 Oracle 8（相当长一段时间以前）中引入的，用于替换较旧的 `DECODE` 函数。
*   `CASE` 比 `DECODE` 更灵活。
*   对于简单和复杂的逻辑，`CASE` 都比 `DECODE` 更具可读性。
*   `CASE` 和 `DECODE` 的性能相同。

如果你在所有情况下都使用 `CASE`，那么你就保持了代码的一致性。有时对于简单逻辑，使用 `DECODE` 似乎更容易，但如果你在某些地方使用 `DECODE`，在其他地方使用 `CASE`，会使你的 SQL 代码更难理解。

### 总结

Oracle SQL 包含使用条件逻辑的能力，这等同于 "if then else" 语句。这可以通过两种方式实现：`CASE` 和 `DECODE`。

`CASE` 语句允许你执行条件逻辑，可以是简单的 CASE 语句，也可以是搜索式的 CASE 语句。这些语句几乎相同，只在表达式使用的位置上有所不同，但搜索式 CASE 语句在逻辑上更灵活。

`DECODE` 函数也可以以类似的方式用于条件逻辑。然而，它限制更多、更古老，且更难阅读。我建议使用 `CASE` 语句而不是 `DECODE` 函数。

## 24. 理解聚合函数

在上一章中，你学习了什么是函数。你查看了一些函数示例，例如 `LOWER`，它对特定列的值执行特定操作。`LOWER` 函数用于将每个记录的 `last_name` 值转换为小写版本。

```
SELECT id,
last_name,
LOWER(last_name)
FROM employee;
```

此查询的结果显示小写的 `last_name` 记录。

| ID | LAST_NAME | LOWER(LAST_NAME) |
| --- | --- | --- |
| 1 | JONES | jones |
| 2 | SMITH | smith |
| 3 | KING | king |
| 4 | SIMPSON | simpson |
| 5 | ANDERSON | anderson |
| 6 | COOPER | cooper |
| 7 | ADAMS | adams |
| 8 | SMITH | smith |
| 9 | PATRICK | patrick |
| 10 | Jones | jones |

这个函数，以及我们看过的其他函数，都是对每个记录单独执行的。SQL 中有一些函数允许你对多条记录执行操作。

### 聚合函数

`聚合函数`是 SQL 中一种可以使用来自多于一行数据的函数。它们被称为聚合函数，因为它们聚合数据。虽然你之前学到的函数，如 `ADD_MONTHS` 和 `LOWER`，作用于返回的每一行，但聚合函数会查看多行中的数据并返回单个结果。

为什么这很有用？SQL 中有五个常用的聚合函数，它们允许你执行以下任务：

*   查找列或表中的记录数
*   查找一组数字的总和
*   查找最大值或最高值
*   查找最小值或最低值
*   查找值列表的平均值

在本章中，你将学习这些函数以及如何使用它们。

### SUM 函数

你将学习的第一个聚合函数是 `SUM`。`SUM` 函数用于对数字进行总计或求和。它看起来像这样：

```
SUM ( [DISTINCT] expression)
```

`SUM` 函数接受两个参数：

*   `expression`：这是要相加的表达式，可以是表中的列或其他数值表示形式。
*   `DISTINCT`：这是一个可选参数，指定 SUM 函数只应对表达式中的不同或唯一值求和。我不经常使用它，但知道它是可用的。

