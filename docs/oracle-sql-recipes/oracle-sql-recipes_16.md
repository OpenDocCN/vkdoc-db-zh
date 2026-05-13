# 第一章

■ ■ ■

## 基础知识

本章提供了许多基础的“配方”，帮助你开始——或重温旧日记忆——掌握 SQL 语句的核心构建块。我们将为你展示从 Oracle 数据库表中选择、更改和删除数据的配方，以及执行此类工作时通常需要包含的一些常用选项。

对于那些已扎实掌握 SQL 的读者，可以随意以**自选**的方式深入阅读本章。我们在这个早期阶段包含了一两个巧妙的配方，以确保你能从第一章开始就从《Oracle SQL 配方》中获得最大收益。继续使用菜单的比喻，你可以按任何顺序享用本书中的配方，并跨章节混合搭配代码片段和技术，从而衍生出你自己的配方。

■ **注意** 我们将快速讲解基础知识，并省略诸如语法图示和讨论每个配方选项的每一种排列组合等细节。如果你看到立即无法理解的命令和技术，请不要烦恼。在 Oracle 测试实例中亲自尝试它们，以获得直观感受，并且请记住，像《Begining Oracle SQL》（de Haan, Fink, Jørgensen, Morton）和《Beginning SQL Queries》（Churcher）这样的配套书籍有助于学习 SQL 本身。



##### 1-1. 从表中检索数据

## 问题

你想要从一个表中检索特定的行和列数据。

## 解决方案

执行一条包含 `WHERE` 子句的 `SELECT` 语句。下面是一个简单的 `SELECT` 语句示例，它针对数据库表中符合定义条件的行，查询特定的列值：

[www.it-ebooks.info](http://www.it-ebooks.info/)

**第 1 章 ■ 基础知识**

```sql
select employee_id, first_name, last_name, hire_date, salary
from hr.employees
where department_id = 50
and salary < 7500;
```

当在加载了示例 `HR` 模式的 Oracle 11g 数据库上运行时，该 `SELECT` 语句返回 45 行。以下是你会看到的前 10 行。

```
EMPLOYEE_ID FIRST_NAME LAST_NAME HIRE_DATE SALARY
----------- ---------- --------- --------- ------
        198 Donald     OConnell  21-JUN-99   2600
        199 Douglas    Grant     13-JAN-00   2600
        123 Shanta     Vollman   10-OCT-97   6500
        124 Kevin      Mourgos   16-NOV-99   5800
        125 Julia      Nayer     16-JUL-97   3200
        126 Irene      Mikkilineni 28-SEP-98   2700
        127 James      Landry    14-JAN-99   2400
        128 Steven     Markle    08-MAR-00   2200
        129 Laura      Bissot    20-AUG-97   3300
        130 Mozhe      Atkinson  30-OCT-97   2800
...
```

## 工作原理

该 `SELECT` 语句的格式旨在帮助你理解（或复习）构成一个可工作查询的基本元素。查询的第一行明确说明了我们希望包含在结果中的五个列。

```sql
select employee_id, first_name, last_name, hire_date, salary
```

下一行是 `FROM` 子句，指定了要引用的表（或多表）以提取我们希望看到的列：

```sql
from hr.employees
```

在这个例子中，我们使用了双部分对象命名法，同时指定了表名 `EMPLOYEES` 以及 `employees` 所属的模式 `HR`。这样可以区分 `HR` 模式中的 `EMPLOYEES` 表与任何其他模式中可能存在的其他 `employees` 表，更重要的是，可以避免在未指定模式时，默认隐式选择用户自身模式或当前设置模式中的表。

接下来我们列出 `WHERE` 子句，其中包含两个必须满足才能将行包含在我们结果中的条件。

```sql
where department_id = 50
and salary < 7500
```

`EMPLOYEES` 表中的行必须同时满足这两个测试才能包含在我们的结果中。只满足一个条件而不满足另一个，将无法满足我们的查询，因为我们使用了 `AND` 布尔运算符来组合条件。用非技术术语来说，这意味着一名员工必须列在 ID 为 `50` 的部门中，并且工资低于 `7500`。

[www.it-ebooks.info](http://www.it-ebooks.info/)

**第 1 章 ■ 基础知识**

其他布尔运算符是 `OR` 和 `NOT`，它们遵循正常的布尔优先级规则，并且可以使用括号来明确修改逻辑。我们可以修改第一个例子来展示这些运算符的组合使用：

```sql
select employee_id, first_name, last_name, hire_date, salary
from hr.employees
where department_id = 50
and (salary < 2500 or salary > 7500);
```

此查询寻求与第一个查询相同的列，但这次关注的是部门 `50` 中工资低于 `2500` 或高于 `7500` 的成员。

```
EMPLOYEE_ID FIRST_NAME LAST_NAME HIRE_DATE SALARY
----------- ---------- --------- --------- ------
        120 Matthew    Weiss     18/JUL/96   8000
        121 Adam       Fripp     10/APR/97   8200
        122 Payam      Kaufling  01/MAY/95   7900
        127 James      Landry    14/JAN/99   2400
        128 Steven     Markle    08/MAR/00   2200
        132 TJ         Olson     10/APR/99   2100
        135 Ki         Gee       12/DEC/99   2400
        136 Hazel      Philtanker 06/FEB/00   2200
```

在示例 `HR` 模式中，只有 8 行匹配这些条件。

##### 1-2. 从表中选择所有列

## 问题

你想要从表中检索所有列，但不想花时间键入所有列名作为 `SELECT` 语句的一部分。

## 解决方案

使用星号 (`*`) 占位符来表示表的所有列。例如：

```sql
select *
from hr.employees
where department_id = 50
and salary < 7500;
```

我们的结果显示了换行且仅列出了部分数据以节省空间，但你可以看到星号对所选列的效果：

[www.it-ebooks.info](http://www.it-ebooks.info/)

**第 1 章 ■ 基础知识**



