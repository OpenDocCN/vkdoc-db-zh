# 第 2 章 ■ 数据汇总与聚合

`GROUP BY`子句决定了目标表的行应归入哪些组，以供后续的聚合函数使用。出于性能考虑，Oracle 会隐式地对数据进行排序以符合所需的分组。通常，从`SELECT`语句的第一行就能推断出`GROUP BY`子句中需要的列。

[www.it-ebooks.info](http://www.it-ebooks.info/)

```sql
select department_id, avg(salary)
```

我们告诉 Oracle 希望将个人的薪资数据聚合成平均值，但我们还没有说明如何处理每行中单独的`DEPARTMENT_ID`值。我们是否应该显示每个`DEPARTMENT_ID`条目（包括重复项），并对每个条目显示平均值？显然，这会导致输出重复且浪费——同时也会让你疑惑是否还有更复杂的东西需要理解。通过在`GROUP BY`子句中使用“未聚合”的字段，我们指示 Oracle 如何根据其计算出的聚合值来折叠或分组单个行值。

```sql
group by department_id
```

这意味着我们`SELECT`语句中的所有值要么是聚合值，要么被`GROUP BY`子句所覆盖。编写语法正确的`GROUP BY`语句的关键在于始终记住：值要么被分组，要么被聚合——不允许有“漏网之鱼”。在一个按多个值分组的查询中，你可以清楚地看到这一点。

##### 2-3. 按多个字段分组数据

### 问题

你需要同时按多个值分组报告数据。例如，人力资源部门可能需要按`DEPARTMENT_ID`和`JOB_ID`报告最低、平均和最高`SALARY`。

### 解决方案

Oracle 的`GROUP BY`功能可以扩展到任意数量的列和表达式，因此我们可以扩展之前的方案以满足新的分组需求。我们知道需要聚合什么：`SALARY`值以三种不同的方式聚合。剩下的`DEPARTMENT_ID`和`JOB_ID`则用于分组。我们还希望结果排序，以便能在同一部门内查看不同的`JOB_ID`值及其上下文，并按薪资从高到低排列。接下来的 SQL 语句通过向`GROUP BY`和`ORDER BY`子句添加必要的条件实现了这一点。

```sql
Select department_id, job_id, min(salary), avg(salary), max(salary)
From hr.employees
Group by department_id, job_id
Order by department_id, max(salary) desc;
```

```text
DEPARTMENT_ID JOB_ID MIN(SALARY) AVG(SALARY) MAX(SALARY)
------------- ---------- ----------- ----------- -----------
           10 AD_ASST        4400        4400        4400
           20 MK_MAN        13000       13000       13000
           20 MK_REP         6000        6000        6000
           30 PU_MAN        11000       11000       11000
           30 PU_CLERK       2500        2780        3100
...
20 rows selected.
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 工作原理

当按聚合函数或其他函数排序时，你可以利用 Oracle 对排序列的简写表示法。因此，你可以编写语句按数据的基于列的位置排序，而不必写出聚合表达式繁琐的完整文本。

```sql
Select department_id, job_id, min(salary), avg(salary), max(salary)
From hr.employees
Group by department_id, job_id
Order by 1, 5 desc;
```

在处理分组结果时，你可以在排序中混合使用列名、聚合表达式、数字位置，甚至是`SELECT`子句中的列别名，如下一个 SQL 语句所示。

```sql
Select department_id, job_id, min(salary), avg(salary), max(salary) Max_Sal
From hr.employees
Group by department_id, job_id
Order by 1, job_id, Max_Sal desc;
```

灵活性是这里的关键，随着你创建和使用更复杂的排序与分组表达式，你会发现别名和序数表示法很有帮助。然而，为了可读性，你应该尽量保持一致。

##### 2-4. 在聚合数据集中忽略分组

### 问题

你希望根据聚合函数或分组操作的结果忽略某些数据组。实际上，你真正需要的是在`GROUP BY`子句之后再运行一个`WHERE`子句，用于在组级别或聚合级别提供筛选条件。

### 解决方案

SQL 提供了`HAVING`子句来对分组数据应用条件。对于我们的方案，我们解决了在`HR.EMPLOYEES`表中，找出每个部门中从事相同工作的人员的最低、平均和最高薪资的问题。重要的是，我们只想看到那些在给定部门中有超过一人从事相同工作的聚合值。接下来的 SQL 语句在`HAVING`子句中使用一个表达式来解决我们的问题。

```sql
select department_id, job_id, min(salary), avg(salary), max(salary), count(*)
from hr.employees
group by department_id, job_id
having count(*) > 1;
```

我们的方案产生如下汇总结果。

[www.it-ebooks.info](http://www.it-ebooks.info/)

```text
DEPARTMENT_ID JOB_ID MIN(SALARY) AVG(SALARY) MAX(SALARY) COUNT(*)
------------- ---------- ----------- ----------- ----------- ----------
           90 AD_VP        17000       17000       17000          2
           50 ST_CLERK      2100        2800        3600         19
           80 SA_REP        6100  8396.55172       11500         29
           50 ST_MAN        3750  6691.66667        8200          6
           80 SA_MAN       10500       12200       14000          5
           50 SH_CLERK      2500        3215        4200         20
           60 IT_PROG       4200        5760        9000          5
           30 PU_CLERK      2500        2780        3100          5
          100 FI_ACCOUNT    6900        7920        9000          5
9 rows selected.
```

### 工作原理

你可以立即看到，我们只有九组员工的结果。与之前方案返回的 20 组相比，显然`HAVING`子句起了作用——但它做了什么呢？

`HAVING`子句是在所有分组和聚合操作完成后进行评估的。Oracle 内部会先生成如下结果：

```text
DEPARTMENT_ID JOB_ID MIN(SALARY) AVG(SALARY) MAX(SALARY) COUNT(*)
------------- ---------- ----------- ----------- ----------- ----------
          110 AC_ACCOUNT    8300        8300        8300          1
           90 AD_VP        17000       17000       17000          2
           50 ST_CLERK      2100        2800        3600         19
           80 SA_REP        6100  8396.55172       11500         29
          110 AC_MGR       12000       12000       12000          1
...
```

这实际上就是我们没有`HAVING`子句的方案结果。然后，Oracle 应用`HAVING`条件 `having count(*) > 1`

之前结果中加粗的行的计数为 1（在相应的`DEPARTMENT_ID`中只有一名该`JOB_ID`的员工），这意味着它们未能满足`HAVING`子句的标准，因此被排除在最终结果之外，从而得到了我们上面看到的解决方案。

`HAVING`子句的条件可以任意复杂，因此你可以使用多种不同类型的条件。

```sql
select department_id, job_id, min(salary), avg(salary), max(salary), count(*)
from hr.employees
group by department_id, job_id
having count(*) > 1
   and min(salary) between 2500 and 17000
   and avg(salary) != 5000
   and max(salary)/min(salary) < 2
;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

```text
DEPARTMENT_ID JOB_ID MIN(SALARY) AVG(SALARY) MAX(SALARY) COUNT(*)
------------- ---------- ----------- ----------- ----------- ----------
           90 AD_VP        17000       17000       17000          2
           80 SA_REP        6100  8396.55172       11500         29
           80 SA_MAN       10500       12200       14000          5
           50 SH_CLERK      2500        3215        4200         20
           30 PU_CLERK      2500        2780        3100          5
          100 FI_ACCOUNT    6900        7920        9000          5
6 rows selected.
```

##### 2-5. 在多个层级聚合数据

### 问题

你想要为报表计算总计、平均值和其他聚合数据，以及各种维度的子汇总。你希望用尽可能少的语句实现这一点，最好只用一条，而不是必须执行单独的语句来获取沿途每个中间子汇总。

### 解决方案



在 Oracle 中，你可以使用 `CUBE`、`ROLLUP` 和分组集功能来计算小计或其他中间聚合值。对于这个示例，我们假设一些现实需求。我们希望按 `department_id` 和 `job_id` 查找平均薪资和总薪资，并在部门级别显示有意义的更高级别平均值和小计（无论工作类别如何），以及整个组织的总计和公司范围内的平均值。

```sql
select department_id, job_id, avg(salary), sum(salary)
from hr.employees
group by rollup (department_id, job_id);
```

我们的结果（显示部分输出）包括按 `DEPARTMENT_ID` 和 `JOB_ID` 的总和与平均值、按 `DEPARTMENT_ID` 的汇总聚合值，以及所有数据的总计。

```
DEPARTMENT_ID JOB_ID AVG(SALARY) SUM(SALARY)
------------- ---------- ----------- -----------
...
80 SA_MAN 12200 61000
80 SA_REP 8396.55172 243500
80 8955.88235 304500
90 AD_VP 17000 34000
90 AD_PRES 24000 24000
90 19333.3333 58000
100 FI_MGR 12000 12000
100 FI_ACCOUNT 7920 39600
100 8600 51600
[www.it-ebooks.info](http://www.it-ebooks.info/)
第二章 ■ 数据汇总与聚合
110 AC_MGR 12000 12000
110 AC_ACCOUNT 8300 8300
110 10150 20300
6473.36449 692650
已选择 33 行。
```

### 工作原理

`ROLLUP` 函数执行多级分组，采用一种从右至左的方式，通过中间级别向上汇总到任何总计或求和。在我们的示例中，这意味着在按 `DEPARTMENT_ID` 和 `JOB_ID` 执行正常分组后，`ROLLUP` 函数会汇总所有 `JOB_ID` 值，以便我们查看特定部门内所有工作的 `DEPARTMENT_ID` 级别的平均值和总和。

然后，`ROLLUP` 汇总到我们示例中的下一个（也是最高）级别，汇总所有部门，实际上提供了组织范围内的汇总。你可以在输出中看到汇总后的行。

执行此汇总相当于运行三个独立的语句，例如下面的三个，并使用 `UNION` 或应用程序级别的代码将结果拼凑在一起。

```sql
select department_id, job_id, avg(salary), sum(salary)
from hr.employees
group by department_id, job_id;
```

```sql
select department_id, avg(salary), sum(salary)
from hr.employees
group by department_id;
```

```sql
select avg(salary), sum(salary)
from hr.employees;
```

当然，在这样做时，你需要自己负责在输出的直观位置插入小计。你可以尝试编写一个三向的 `UNION` 子查询，并使用外部 `SELECT` 来排序。如果这听起来越来越复杂，那么值得庆幸的是，`ROLLUP` 命令及其关联命令 `CUBE` 已经取代了执行此类笨拙计算的需要。

细心的观察者会注意到，由于 `ROLLUP` 是从右至左处理给定的列，我们看不到按职位汇总部门的值。我们可以使用此版本的示例来实现这一点。

```sql
select department_id, job_id, avg(salary), sum(salary)
from hr.employees
group by rollup (job_id, department_id);
```

这样做，我们得到了所需的 `DEPARTMENT_ID` 中间汇总，但失去了 `JOB_ID` 中间汇总，只看到 `JOB_ID` 在最终级别汇总。要在所有维度上进行汇总，请将示例改为使用 `CUBE` 函数。

```sql
select department_id, job_id, min(salary), avg(salary), max(salary)
from hr.employees
group by cube (department_id, job_id);
```

结果显示了每个级别的汇总，部分结果如下所示。

```
DEPARTMENT_ID JOB_ID MIN(SALARY) AVG(SALARY) MAX(SALARY)
------------- ---------- ----------- ----------- -----------
7000 7000 7000
2100 6392.27273 24000
AD_VP 17000 17000 17000
AC_MGR 12000 12000 12000
FI_MGR 12000 12000 12000
...
10 4400 4400 4400
10 AD_ASST 4400 4400 4400
20 6000 9500 13000
20 MK_MAN 13000 13000 13000
20 MK_REP 6000 6000 6000
...
```



