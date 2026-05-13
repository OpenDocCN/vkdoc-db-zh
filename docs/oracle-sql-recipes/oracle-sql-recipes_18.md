# 第 2 章 ■ 数据汇总与聚合

在本章中，我们将介绍一些在更高层次上处理数据的配方，其中分组、汇总和全局概览非常重要。我们将探讨的许多配方涵盖了常见或棘手的报告场景，这类场景常见于商业或专业环境中。大多数机构都需要关于“谁在何时向谁销售了什么”、“特定时间段内发生了多少人、销售额或活动”、“跨区域或客户群的趋势”等方面的报告。

许多人都熟悉 Oracle 提供的一些用于执行汇总和聚合的基本方法。但开发人员和 DBA 通常会尝试在他们的应用程序中执行更复杂的计算——结果把自己搞得一团糟。如果你在自己的工作中看到以下任何组合：在代码中复制数据库功能、不必要地移动临时数据和结果，以及类似低效的任务，你就能发现这个问题。

Oracle 在这方面功能正呈爆炸式增长，尤其是在过去几个主要版本中引入了在线分析处理（`OLAP`）功能之后。在探索了这些用于汇总和聚合数据的配方后，你会发现 Oracle 是你手中满足大量报告和其他需求的头号工具。

##### 2-1. 汇总列中的值

**问题**

你需要以某种方式汇总某一列的数据。例如，你被要求报告每位员工的平均薪资、薪资预算总额、员工人数、最高和最低收入者等信息。

**解决方案**

你不需要单独计算总和和计数员工人数来确定平均薪资。`AVG` 函数可以为你计算平均薪资，如下一条 `SELECT` 语句所示。

[www.it-ebooks.info](http://www.it-ebooks.info/)

```sql
select avg(salary)
from hr.employees;

AVG(SALARY)
-----------
6473.36449
```

请注意，我们的配方中没有 `WHERE` 子句，这意味着计算总体平均值时评估了 `HR.EMPLOYEES` 表中的所有行。

像 `AVG` 这样的函数被称为 **聚合函数**，你有许多这样的函数可供使用。例如，要计算已支付的总薪资，请使用 `SUM` 函数，如下所示：

```sql
select sum(salary)
from hr.employees;

SUM(SALARY)
-----------
666666
```

要统计领取薪资的人数，你可以简单地使用 `COUNT` 函数统计表中的行数。

```sql
Select count(salary)
From hr.employees;

COUNT(SALARY)
-------------
107
```

最大值和最小值可以使用 `MAX` 和 `MIN` 函数计算。到现在你可能在想 Oracle 为统计函数使用了非常简单的缩写名称，大体上你是对的。`MAX` 和 `MIN` 函数如下所示。

```sql
select min(salary), max(salary)
from hr.employees;

MIN(SALARY) MAX(SALARY)
----------- -----------
2100        24000
```

**工作原理**

Oracle 有大量内置的统计和分析函数来执行常见的汇总任务，例如计算平均值、总和以及最小值和最大值。你不需要手动执行中间计算，尽管如果你想验证 Oracle 的计算过程也可以这样做。

下面的语句将 Oracle 对薪资的平均值计算结果与我们自己用总额除以员工人数的结果进行了比较。

[www.it-ebooks.info](http://www.it-ebooks.info/)

```sql
select avg(salary), sum(salary)/count(salary)
from hr.employees;

AVG(SALARY) SUM(SALARY)/COUNT(SALARY)
----------- -------------------------
6473.36449  6473.36449
```

很高兴知道 Oracle 的计算是正确的。为了完整了解 Oracle 的聚合功能，我们需要考虑当数据包含 `NULL` 值时会发生什么。我们当前的配方聚合了员工的薪资，恰好每位员工都有薪资。但只有销售人员因其销售努力而获得佣金，这在 `HR.EMPLOYEES` 表中体现为 `COMMISSION_PCT` 列有值。非销售人员没有佣金，在 `COMMISSION_PCT` 中体现为 `NULL` 值。那么，当我们尝试对 `COMMISSION_PCT` 值进行平均或计数时会发生什么？下一条 `SQL` 语句展示了这两种聚合的结果。

```sql
select count(commission_pct), avg(commission_pct)
from hr.employees;

COUNT(COMMISSION_PCT) AVG(COMMISSION_PCT)
--------------------- -------------------
38                    .225
```

尽管我们看到了 107 位有薪资的员工，但 `COUNT` 函数忽略了所有 `COMMISSION_PCT` 为 `NULL` 的值，只统计了 38 位有佣金的员工。同样，在计算员工佣金的平均值时，Oracle 也只考虑了那些有实际值的行，忽略了 `NULL` 条目。

只有两种特殊情况，Oracle 会在聚合函数中考虑 `NULL` 值。第一种是 `GROUPING` 函数，用于测试包含 `NULL` 值的分析函数的结果是直接来自基础表中的行，还是来自分析计算的最终聚合“`NULL` 集”。第二种特殊情况是 `COUNT(*)` 函数。因为星号意味着表中的所有列，Oracle 处理行计数时不依赖于任何实际数据值，在这种情况下将 `NULL` 和正常值同等对待。

为了说明这一点，下一条 `SQL` 语句并排显示了 `COUNT(*)` 和 `COUNT(COMMISSION_PCT)` 的区别。

```sql
select count(*), count(commission_pct)
from hr.employees;

COUNT(*) COUNT(COMMISSION_PCT)
-------- ---------------------
107      38
```

我们使用 `COUNT(*)` 统计了表中的所有行，而 `COUNT(COMMISSION_PCT)` 只统计了 `COMMISSION_PCT` 不为 `NULL` 的值。

[www.it-ebooks.info](http://www.it-ebooks.info/)

##### 2-2. 为不同分组汇总数据

**问题**

你想汇总某一列的数据，但你不想对表中的所有行进行汇总。

你想将行分成若干组，然后分别为每组汇总该列。例如，你需要知道每个部门的平均薪资。

**解决方案**

使用 `SQL` 的 `GROUP BY` 功能将共同的数据子集分组在一起，以便应用 `COUNT`、`MIN`、`MAX`、`SUM` 和 `AVG` 等函数。这条 `SQL` 语句展示了如何使用 `GROUP BY` 对数据的子组应用聚合函数。

```sql
select department_id, avg(salary)
from hr.employees
group by department_id;
```

以下是按 `DEPARTMENT_ID` 显示平均值的结果。

```
DEPARTMENT_ID AVG(SALARY)
------------- -----------
100           8600
30            4150
20            9500
70            10000
90            19333.3333
110           10150
50            3503.33333
40            6500
80            8955.88235
10            4400
60            5760

12 rows selected.
```

请注意，结果的第三行显示 `DEPARTMENT_ID` 为 null。

**工作原理**


