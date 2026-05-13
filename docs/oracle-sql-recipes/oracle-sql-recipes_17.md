# SQL 基础操作示例

## 数据查询结果示例

```
EMPLOYEE_ID FIRST_NAME LAST_NAME EMAIL PHONE_NUMBER HIRE_DATE JOB_ID SALARY
COMMISSION_PCT MANAGER_ID DEPARTMENT_ID
----------- ---------- --------- -------- ------------ ---------- --------- ------
-------------- ---------- -------------
198 Donald OConnell DOCONNEL 650.507.9833 21-JUN-99 SH_CLERK 2600
124 50
199 Douglas Grant DGRANT 650.507.9844 13-JAN-00 SH_CLERK 2600
124 50
123 Shanta Vollman SVOLLMAN 650.123.4234 10-OCT-97 ST_MAN 6500
100 50
```

## 如何工作

在 SQL 中，`*` 是一个表示表中所有列名的快捷方式。当 Oracle 的解析器在查询中看到 `SELECT *` 时，解析器会将 `*` 替换为所有可能的列名列表（标记为隐藏的列除外）。使用 `SELECT *` 是构建临时查询的快速方法，因为您无需查找并输入所有正确的列名。

在您忍不住对所有查询使用星号之前，重要的是要记住，良好的设计通常意味着只显式命名您感兴趣的那些列。这会带来更好的性能，因为 Oracle 否则必须读取系统目录来确定要包含在查询中的字段。指定显式列还可以保护您免受未来可能添加列到表中的问题。如果您编写的代码期望使用星号带来的隐式顺序，当该顺序意外更改时（例如以不同方式删除并重新创建表，或向表中添加新列），您可能会遇到令人不快的意外。

在大多数情况下，星号最好保留用于交互式查询和临时分析。

##### 1-3. 对结果进行排序

### 问题

用户希望以特定方式查看查询排序后的数据。例如，他们希望看到员工按姓氏字母顺序排序，然后是名字。

### 解决方案

Oracle 使用标准的 `ORDER BY` 子句来允许您对查询结果进行排序。

```sql
select employee_id, first_name, last_name, hire_date, salary
from hr.employees
where salary > 5000
order by last_name, first_name;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 第 1 章 ■ 基础知识

结果如下。

```
EMPLOYEE_ID FIRST_NAME LAST_NAME HIRE_DATE SALARY
----------- ---------- --------- --------- ------
174 Ellen Abel 11/MAY/96 11000
166 Sundar Ande 24/MAR/00 6400
204 Hermann Baer 07/JUN/94 10000
167 Amit Banda 21/APR/00 6200
172 Elizabeth Bates 24/MAR/99 7300
151 David Bernstein 24/MAR/97 9500
169 Harrison Bloom 23/MAR/98 10000
148 Gerald Cambrault 15/OCT/99 11000
154 Nanette Cambrault 09/DEC/98 7500
110 John Chen 28/SEP/97 8200
…
```

### 如何工作

此解决方案中的 `ORDER BY` 子句指示 Oracle 按 `LAST_NAME` 列排序，当 `LAST_NAME` 值匹配时，再按 `FIRST_NAME` 列排序。除非另有指示，Oracle 隐式使用升序排序，因此数字从零到九排序，字母从 A 到 Z 排序，依此类推。您可以使用 `ASC` 和 `DESC` 选项显式控制升序和降序排序方向。

这是一个例子：

```sql
select employee_id, first_name, last_name, hire_date, salary
from hr.employees
where salary > 5000
order by salary desc;
```

我们对薪水进行的显式降序排序结果如下：

```
EMPLOYEE_ID FIRST_NAME LAST_NAME HIRE_DATE SALARY
----------- ---------- --------- --------- ------
100 Steven King 17-JUN-87 24000
102 Lex De Haan 13-JAN-93 17000
101 Neena Kochhar 21-SEP-89 17000
145 John Russell 01-OCT-96 14000
146 Karen Partners 05-JAN-97 13500
…
```

##### 1-4. 向表中添加行

### 问题

您需要向表中添加新的数据行。例如，一位新员工加入公司，需要将他的数据添加到 `HR.EMPLOYEES` 表中。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 第 1 章 ■ 基础知识

### 解决方案

使用 `INSERT` 语句向表中添加新行。要向表添加新行，请为所有必填列提供值，以及任何可选列的值。这是一条添加新员工的语句：

```sql
insert into hr.employees
(employee_id, first_name, last_name, email, phone_number, hire_date, job_id, salary, commission_pct, manager_id, department_id)
values
(207, 'John ', 'Doe ', 'JDOE ', '650.555.8877 ', '25-MAR-2009 ', 'SA_REP ', 3500, 0.25, 145, 80);
```

### 如何工作

`INSERT` 语句将值列表与列列表关联起来。它基于该关联创建一行，并将该行插入到目标表中。

Oracle 将检查 `NULL` 约束以及主键、外键和其他已定义约束，以确保插入数据的完整性。有关确定表上约束状态以及它们可能如何影响插入新数据的配方，请参见第 10 章。

您可以通过检查表的描述来检查哪些字段是必填的（定义为 `NOT NULL`）。您可以从 SQL Developer 或 SQL*Plus 中发出 `DESCRIBE` 命令（可以缩写为 `DESC`）来实现。例如：

```sql
desc hr.employees;
```

```
Name Null Type
------------------------------ -------- ------------
EMPLOYEE_ID NOT NULL NUMBER(6)
FIRST_NAME VARCHAR2(20)
LAST_NAME NOT NULL VARCHAR2(25)
EMAIL NOT NULL VARCHAR2(25)
PHONE_NUMBER VARCHAR2(20)
HIRE_DATE NOT NULL DATE
JOB_ID NOT NULL VARCHAR2(10)
SALARY NUMBER(8,2)
COMMISSION_PCT NUMBER(2,2)
MANAGER_ID NUMBER(6)
DEPARTMENT_ID NUMBER(4)
11 rows selected
```

您可以通过不枚举列名列表，并为当前表定义中的*每一列*按正确顺序提供数据来编写更短的 `INSERT` 语句。这是一个例子：

```sql
insert into hr.employees
values
(208, 'Jane ', 'Doe ', 'JADOE ', '650.555.8866 ', '25-MAR-2009 ', 'SA_REP ', 3500, 0.25, 145, 80)
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 第 1 章 ■ 基础知识

**注意** 省略列名列表很少（如果有的话）是一个好主意——这是出于一些相当严重的原因。您不知道未来可能会对表进行哪些更改，并且通过假设隐式列顺序，您会使您的 SQL 对未来的架构更改变得脆弱。也许可以说明可能出现问题的最佳例子是静默逻辑损坏。如果有人以不同的列顺序重建基础表，但您的 `INSERT` 语句通过了数据类型和其他检查，您可能会发现自己将数据静默地插入到了错误的列中，带来灾难性的后果。我们强烈建议始终在 `INSERT` 语句中枚举列。

##### 1-5. 将行从一个表复制到另一个表

### 问题

您想将信息从一个表复制到另一个表。

### 解决方案

使用带有 `SELECT` 选项的 `INSERT` 语句将数据从一个表复制到另一个表。假设您有一个候选职位申请表，其中许多详细信息与 `HR.EMPLOYEES` 表相同。这个 `INSERT` 语句将基于对 `CANDIDATES` 表的 `SELECT` 语句插入到 `HR.EMPLOYEES` 表中。

```sql
insert into hr.employees
(employee_id, first_name, last_name, email, phone_number, hire_date, job_id, salary, commission_pct, manager_id, department_id)
select 210, first_name, last_name, email, phone_number, sysdate, 'IT_PROG', 3500, NULL, 103, 60
from hr.candidates
where first_name = 'Susan'
and last_name = 'Jones';
```

### 如何工作

此配方将要插入到 `HR.EMPLOYEES` 表中的值播种为对 `CANDIDATES` 表进行 `SELECT` 的结果。`SELECT` 语句可以单独运行以查看传递给 `INSERT` 语句的数据。

```sql
select 210, first_name, last_name, email, phone_number, sysdate, job_id, 3500, NULL,
'IT_PROG', 103
from hr.candidates
where first_name = 'Susan'
and last_name = 'Jones';
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 第 1 章 ■ 基础知识

```
210 FIRST_NAME LAST_NAME EMAIL PHONE_NUMBER SYSDATE 'IT_PRO 3500 N 103 60
--- ---------- --------- ------ ------------ ------- ---- - --- --
210 Susan Jones SJONES 650.555.9876 30-MAR-09 IT_PROG 3500 103 60
```



##### 1-6. 批量将数据从一个表复制到另一个表

### 问题

你需要将多行数据从一个表复制到另一个表。

### 解决方案

`INSERT INTO … SELECT …` 方法能够插入多行数据。关键在于 `SELECT` 语句中使用所需的条件来返回你希望插入的行。我们可以修改之前的方案来处理多行。以下是一个多行 `INSERT` 的实际示例。

```sql
select candidate_id, first_name, last_name, email, phone_number, sysdate, job_id, 3500, NULL, 'IT_PROG', 103
from hr.candidates;
```

此方案依赖于与前一个方案中相同的 `HR.CANDIDATES` 表的存在。如果你在本地练习，请确保先创建此表。

### 工作原理

此方案同样从 `HR.CANDIDATES` 表中选取要插入的值。由于 `SELECT` 语句部分没有 `WHERE` 子句，`CANDIDATES` 表中的所有行都将被选中，因此这些行对应的记录都将被插入到 `HR.EMPLOYEES` 表中。

##### 1-7. 修改行中的值

### 问题

你想更改表中一个或多个行的部分数据。

[www.it-ebooks.info](http://www.it-ebooks.info/)

第一章 ■ 基础知识

### 解决方案

顾名思义，`UPDATE` 语句就是用来更改数据的。针对此问题，假设我们想将部门 50 所有员工的薪资提高百分之五。下面的 `UPDATE` 语句实现了此更改：

```sql
update hr.employees
set salary = salary * 1.05
where department_id = 50;
```

### 工作原理

`UPDATE` 语句的基本设计以 `UPDATE` 子句本身开始 `update hr.employees`。这告诉 Oracle 要更新哪个表的行。接着，`SET` 子句指定了要更改的列以及如何更新它们——可以通过字面值、计算、函数结果、子查询或其他方法。

`set salary = salary * 1.05`

在此例中，我们使用了一个自引用计算。新的 `SALARY` 值将是当前值的 1.05 倍，相对于每一行（也就是说，Oracle 将为受影响的每一行执行此计算）。

最后，`WHERE` 子句的作用是提供你已经从 `SELECT` 语句中熟悉的常规过滤谓词。只有那些 `DEPARTMENT_ID` 为 50 的行会受到影响。

##### 1-8. 用单条语句更新多个字段

### 问题

你想更改表中一个或多个行的多个列。

### 解决方案

`update` 语句设计为可以在一条语句中更新任意多行。这意味着你可以在一条 `UPDATE` 语句中使用多个 `column = value` 子句来更改任意多的字段。例如，要更改员工 James Marlow（`EMPLOYEE_ID` 为 131）的电话号码、职位和薪资，我们可以使用一条 `UPDATE` 语句。

```sql
update hr.employees
set job_id = 'ST_MAN',
    phone_number = '650.124.9876',
    salary = salary * 1.5
where employee_id = 131;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

第一章 ■ 基础知识

### 工作原理

Oracle 正常评估 `UPDATE` 语句的谓词。在此例中，它定位 `EMPLOYEE_ID` 为 131 的行。然后根据给定的 `SET` 标准更改每个字段。它在对行的单次处理中完成此操作。

Oracle 还支持在括号中对要更新的列和值进行分组，方式与其他数据库类似，但有一个不同之处。

```sql
update hr.employees
set (job_id,Phone_number,Salary)
    = (select 'ST_MAN','650.124.9876',salary * 1.5 from dual)
where employee_id = 131;
```

`注意` 如果子查询没有返回数据，set 子句中指定的值将被设置为 null。



##### 1-9. 从表中移除不需要的行

## 问题
一名员工已入职另一家公司，你需要将其详细信息从 `HR.EMPLOYEE` 表中移除。

## 解决方案
假设 James Landry，其 `EMPLOYEE_ID` 为 127，已入职另一家公司。你可以使用 `DELETE` 语句来移除 James 的详细信息。

```sql
delete
from hr.employees
where employee_id = 127;
```

## 工作原理
本方法演示了基本的 `DELETE` 语句，在 `FROM` 子句中目标表为 `HR.EMPLOYEES`。`WHERE` 子句提供了用于仅影响符合指定条件行的谓词。

可以构建没有 `WHERE` 子句的 `DELETE` 语句，这实际上会匹配表中的所有行。在这种情况下，表中的所有行都将被删除。

```sql
delete
from hr.employees;
```

**注意**：如果你确实运行了上述任一 `DELETE` 语句，并且希望在后续方法中继续使用 `HR.EMPLOYEES` 数据，请不要忘记回滚。

##### 1-10. 从表中移除所有行

## 问题
你希望移除给定表的所有数据。

## 解决方案
使用 `DELETE` 语句删除表中的所有行，允许 Oracle 完整记录该操作，这样如果你意外执行了此语句，可以进行回滚。但同样的记录操作及其所需时间，意味着使用 `DELETE` 语句有时对于期望的结果来说太慢了。

Oracle 提供了一种补充技术来移除表中的所有数据，称为 `TRUNCATE`。要截断 `HR.EMPLOYEES` 表，我们使用以下 SQL 语句：

```sql
truncate table hr.employees;
```

## 工作原理
执行 `TRUNCATE` 语句时不使用 `WHERE` 子句谓词或其他修饰符。该语句被视为 DDL（数据定义语言）SQL，这对事务有其他影响，例如隐式提交其他未完成的事务。有关事务控制和监控方法的更多详情，请参阅第 9 章。

使用 `TRUNCATE` 命令还会重置表的高水位线，这意味着当 Oracle 执行优化器计算（包括判断表中已用容量的影响）时，它会将该表视为从未有数据占用过任何空间。

##### 1-11. 从另一个查询的结果中选择

## 问题
你需要将查询的结果视作表的内容。你不希望存储中间结果，因为每次运行查询时都需要重新生成它们。

## 解决方案
Oracle 的内联视图功能允许将查询包含在语句的 `FROM` 子句中，并使用表别名来引用结果。

```sql
select d.department_name
from
(select department_id, department_name
from hr.departments
where location_id != 1700) d;
```

结果将如下所示。

```
DEPARTMENT_NAME
---------------
Marketing
Human Resources
Shipping
IT
Public Relations
Sales

6 rows selected.
```

## 工作原理
本方法将针对 `HR.DEPARTMENTS` 表的 `SELECT` 语句的结果视为一个内联视图，并为其命名为 “d”。虽然在这种情况下提供别名是可选的，但它使得 SQL 更具可读性、可测试性，并且更符合标准。然后，你几乎可以像引用普通语句一样引用该语句的结果。以下是内联视图的 `SELECT` 语句的结果。

```
DEPARTMENT_ID DEPARTMENT_NAME
------------- -----------------
20 Marketing
40 Human Resources
50 Shipping
60 IT
70 Public Relations
80 Sales

6 rows selected.
```

该内联视图现在的作用就像你有一个表 `D`，其结构定义如下。

```
Name                                      Null?    Type
----------------------------------------- -------- ----------------------------
DEPARTMENT_ID                             NOT NULL NUMBER(4)
DEPARTMENT_NAME                           NOT NULL VARCHAR2(30)
```




##### 1-12. 基于查询设置 Where 条件

`DEPARTMENT_ID` NOT NULL NUMBER(4)
`DEPARTMENT_NAME` NOT NULL VARCHAR2(30)

这些是基础表 `HR.DEPARTMENTS` 的定义，我们的内联视图隐式地继承了这些定义。外部的 `SELECT` 语句查询结果集，就好像有一个真实的表 `d`，并且你是针对它编写了这条语句。

```
select department_name
from d;
```

### 问题

你需要从表中查询数据，但其中一个查询条件依赖于另一张表在查询运行时的数据——并且你想避免硬编码条件。具体来说，你希望找出所有在北美设有办公室的部门，以便用于报告。

### 解决方案

在 HR 模式中，`DEPARTMENTS` 表列出了部门，`LOCATIONS` 表列出了地点。

```
select department_name
from hr.departments
where location_id in
  (select location_id
   from hr.locations
   where country_id = 'US'
      or country_id = 'CA');
```

我们的结果如下：

DEPARTMENT_NAME
---------------
IT 部
运输部
行政部
采购部
执行部
…
[www.it-ebooks.info](http://www.it-ebooks.info/)
第一章 ■ 基础知识

### 原理

我们的 `in` 谓词右侧读取子 `SELECT` 的结果，提供值用于与 `HR.DEPARTMENTS` 表的 `LOCATION_ID` 值进行比较。运行子 `SELECT` 查询本身就可以看到生成了哪些值。

```
select location_id
from hr.locations
where country_id = 'US'
   or country_id = 'CA';
```

结果如下：

LOCATION_ID
-----------
1400
1500
1600
1700
1800
1900
已选择 6 行。

外部的 `SELECT` 将 `LOCATION_ID` 值与这个动态查询出的列表进行比较。这与运行以下静态查询类似。

```
select department_name
from hr.departments
where location_id in (1400,1500,1600,1700,1800,1900);
```

此方法的关键优势在于，如果我们开设新办公室并更改了北美所有的 `LOCATION_ID`，我们无需重写查询：子查询将动态输出必要的新值。

##### 1-13. 查找和消除查询中的 NULL 值

### 问题

你需要报告有多少员工的薪酬中包含佣金百分比，以及有多少员工只领取固定工资。你通过 `HR.EMPLOYEES` 表的 `COMMISSION_PCT` 字段来追踪这个信息。
[www.it-ebooks.info](http://www.it-ebooks.info/)
第一章 ■ 基础知识

### 解决方案

`HR.EMPLOYEE` 表的结构允许 `COMMISSION_PCT` 为 `NULL`。可以使用两个查询来查找佣金百分比为 `NULL` 的员工和非 `NULL` 的员工。

首先，这是查找佣金百分比为 `NULL` 的员工的查询：

```
select first_name, last_name
from hr.employees
where commission_pct is null;
```

FIRST_NAME           LAST_NAME
-------------------- -------------------------
Donald               OConnell
Douglas              Grant
Jennifer             Whalen
Michael              Hartstein
Pat                  Fay
…
已选择 72 行。

现在，这是查找持有非 `NULL` 佣金百分比的员工的查询：

```
select first_name, last_name
from hr.employees
where commission_pct is not null;
```

FIRST_NAME           LAST_NAME
-------------------- -------------------------
John                 Russell
Karen                Partners
Alberto              Errazuriz
Gerald               Cambrault
Eleni                Zlotkey
…
已选择 35 行。

### 原理

我们的第一个 `SELECT` 语句使用 `COMMISSION_PCT IS NULL` 子句来测试 `NULL` 条目。这只有两种结果：要么该列条目为 `NULL`，即没有值，满足测试条件；要么它有某个值。

第二个语句使用 `COMMISSION_PCT IS NOT NULL` 子句，这将找到任何 `COMMISSION_PCT` 有实际值的员工。
[www.it-ebooks.info](http://www.it-ebooks.info/)
第一章 ■ 基础知识

#### Oracle 对空字符串的非标准处理

Oracle 偏离了 SQL 标准，它将空字符串（或零长度字符串）隐式地视为 `NULL` 的替代符。这是出于一系列历史和实用原因，但这一点很重要，需要牢记。几乎所有其他 SQL 实现都将空字符串视为一个独立、已知的值。



有编程背景的读者会发现，零长度字符串的概念与此类似——字符串内存中只有字符串终止符（`\0`）作为其唯一组成部分。相比之下，未实例化的字符串则没有已知状态……甚至连终止符都没有。你不会将零长度字符串和未实例化字符串互换使用，但这与 Oracle 处理 `NULL` 的方式类似。

## `NULL` 没有等价物

在 SQL 中使用 `NULL` 的日常操作（或常规工作）常会让人困惑。SQL 表达式是三值逻辑，意味着每个表达式可以是 `true`、`false` 或 `NULL`。正如你已看到的，这会影响所有类型的比较、运算符和逻辑判断。但这种逻辑的一个细微之处常被遗忘，因此我们在此明确重申：`NULL` 没有等价物。没有任何值与 `NULL` 相同，*甚至其他的 `NULL` 值也不例外*。如果你运行以下查询，能猜到结果吗？

```sql
select first_name, last_name
from hr.employees
where commission_pct = NULL;
```

答案是不会选出任何行。尽管你从本示例中上面的 `SELECT` 语句已看到有 72 名员工的 `COMMISSION_PCT` 为 `NULL`，但没有任何 `NULL` 值等于另一个 `NULL`，因此 `COMMISSION_PCT = NULL` 这个条件永远不会匹配，你也就永远无法从此查询中看到结果。

务必使用 `IS NULL` 和 `IS NOT NULL` 来查找或排除 `NULL` 值。

##### 1-14. 按照人的预期进行排序

### 问题

你的文本数据以大小写和句子形式混合存储。你需要以通常人们自然的方式，按不区分大小写的字母顺序对这些数据进行排序。

### 解决方案

为了向 `HR.EMPLOYEES` 表中引入一些混合大小写的数据，让我们运行以下 `UPDATE` 语句，将 William Smith 的姓氏改为大写。

```sql
update hr.employees
set last_name = 'SMITH'
where employee_id = 171;
```

这条 `select` 语句展示了 Oracle 对员工姓氏的默认排序。

```sql
select last_name
from hr.employees
order by last_name;
```

```
LAST_NAME
…
Rogers
Russell
SMITH
Sarchand
Sciarra
Seo
Sewall
Smith
Stiles
Sullivan
…
```

敏锐的读者可能已经预料到这个结果。默认情况下，Oracle 使用二进制排序顺序。这意味着在这个简单示例中，文本是根据所用代码页（`US7ASCII`、`WEISO8859P1` 等）中的数值等价项进行排序的。在这些代码页中，大小写字母具有不同的值，大写字母排在前面。这就是为什么大写的 `SMITH` 排在所有以大写 `S` 开头的其他姓名之前。

大多数人期望看到的是两个“Smith”值排在一起，而不区分大小写。下面的 `NLS` 指令可以实现这一结果。

```sql
alter session set NLS_SORT='BINARY_CI';

select last_name
from hr.employees
order by last_name;
```

```
LAST_NAME
…
Rogers
Russell
Sarchand
Sciarra
Seo
Sewall
Smith
SMITH
Stiles
Sullivan
…
```

### 工作原理

Oracle 支持区分大小写和不区分大小写的排序顺序。默认情况下，你处于名为 `BINARY` 的区分大小写排序环境中。对于每一种这样的排序顺序，都存在一个使用后缀 `_CI` 的等效不区分大小写顺序。我们改用 `BINARY_CI` 排序顺序，并重新运行查询，以看到普通用户预期的结果顺序。

顾名思义，`NLS_SORT` 选项仅影响排序。它并不影响大小写敏感性的其他方面。使用 `NLS_SORT='BINARY_CI'` 时，尝试以不区分大小写的方式比较数据仍然会表现出 Oracle 的默认行为。

```sql
select first_name, last_name
from hr.employees
where last_name like 's%';
```

```
no rows selected
```

别担心。Oracle 提供了一个类似的选项，可以实现这种不区分大小写的比较。

**提示** 更传统的方法可以在不更改任何 `NLS` 参数的情况下解决此问题。你可以使用 Oracle 的 `UPPER` 和 `LOWER` 函数将列名、字面量或两者都转换为大写或小写来进行比较。



这有一个缺点，即阻止了优化器使用标准索引，但你也可以创建基于函数的索引来弥补这一点。

##### 1-15. 启用其他排序和比较选项

### 问题

你需要对以各种大写、小写和句首大写形式存储的文本数据执行不区分大小写的比较和其他排序操作。

### 解决方案

通过激活 Oracle 的语言比较逻辑，你可以使用相同的语句来检索人类期望看到的数据，而无需承担人为区分大小写的负担。

```sql
alter session set NLS_COMP='LINGUISTIC';

select first_name, last_name
from hr.employees
where last_name = 'smith';
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 1 章 ■ 基础知识

```
FIRST_NAME                   LAST_NAME
-------------------- -------------------------
William              SMITH
Lindsey              Smith
```

本方法依赖于与之前方法中使用的相同的大小写转换语句。请务必在测试本方法实际效果之前运行那些语句。

### 工作原理

在设计应用程序时，历史上处理 Oracle 正常区分大小写的文本处理方式有两种方法。一种方法是设计应用程序逻辑以确保数据以一致的方式存储，例如我们在多个方法中使用的 `HR.EMPLOYEES` 表中看到的首字母大写数据。另一种方法允许数据按用户喜好存储，但使用数据库和应用程序逻辑在用户不期望看到的地方隐藏任何大小写差异。

过去这需要付出很大努力，但现在你可以以极少的功夫实现它。以下语句将检索不到任何行，因为在 `HR.EMPLOYEES` 表中输入的 `LAST_NAME` 都没有以小写形式记录。

```sql
Select first_name, last_name
From hr.employees
Where last_name = 'smith';
```
```
no rows selected
```

但我们的方法设法返回了正确的数据，因为它更改了会话设置，指示 Oracle 以语言方式进行比较。`Smith` 的两种变体都不是以我们查询中指定的小写形式存储的，但 `NLS_COMP` 参数控制着 Oracle 中的比较和其他行为，将其设置为 `LINGUISTIC` 会触发典型的用户认为是正常的比较行为。

##### 1-16. 基于存在性条件插入或更新

### 问题

你希望向一个表中插入行，其键标识符可能已经存在。如果标识符不存在，则应创建新行。如果标识符已存在，则应使用新数据更新该行的其他列，而不是创建新行。

### 解决方案

`MERGE` 语句提供了将新数据插入表中的能力，如果提议的新主键在表中尚不存在，则会插入一个新创建的行。如果主键与表中的现有行匹配，则该语句将使用与该键匹配的附加详细信息来更新该行。

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 1 章 ■ 基础知识

在我们的示例中，我们假设要将 `HR.COUNTRIES` 表加载从 `NEW_COUNTRIES` 表来源的经修订的国家详情。

```sql
merge into hr.countries c
using
  (select country_id, country_name
   from hr.new_countries) nc
on (c.country_id = nc.country_id)
when matched then
  update set c.country_name = nc.country_name
when not matched then
  insert (c.country_id, c.country_name)
  values (nc.country_id, nc.country_name);
```

### 工作原理

与简单地将源数据从 `HR.NEW_COUNTRIES` 表直接插入目标 `HR.COUNTRIES` 表——并可能因主键重复错误而失败——不同，`MERGE` 语句根据 `ON` 子句设置逻辑分支来处理匹配和不匹配的行。

通常，`ON` 子句指定如何在源和目标之间匹配主键或唯一键数据。在本方法中，这就是匹配 `COUNTRY_ID` 值，如下所示。
```
on (c.country_id = nc.country_id)
```
其后是两个额外的子句，`WHEN MATCHED THEN` 子句用于匹配 `ON` 条件的值



`WHEN NOT MATCHED THEN` 子句，用于处理需要被视为待插入新数据的不匹配新行。

匹配和不匹配子句还可以包含进一步的筛选条件，甚至包含满足时导致行被删除的条件。

```sql
merge into hr.countries c
using
    (select country_id, country_name, region_id
     from hr.new_countries) nc
on (c.country_id = nc.country_id)
when matched then
    update set c.country_name = nc.country_name,
               c.region_id    = nc.region_id
    delete where nc.region_id = 4
when not matched then
    insert (c.country_id, c.country_name, c.region_id)
    values (nc.country_id, nc.country_name, nc.region_id)
    where (nc.region_id != 4);
```

在这个修改后的配方中，匹配的行将会更新其 `COUNTRY_NAME`，除非新数据的 `REGION_ID` 等于 `4`，在这种情况下，`HR.COUNTRIES` 中的相应行最终将被删除。

不匹配的行将被插入到 `HR.EMPLOYEES` 表中，除非它们的 `REGION_ID` 是 `4`，在这种情况下它们将被忽略。

[www.it-ebooks.info](http://www.it-ebooks.info/)

