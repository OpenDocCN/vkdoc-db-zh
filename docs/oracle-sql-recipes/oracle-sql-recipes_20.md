# 第 2 章 ■ 数据汇总与聚合

ROLLUP 和 CUBE 函数的强大之处在于，它们可以扩展到查询所需的任意多“维度”。诚然，“立方体”（cube）一词意在暗示在三维空间中查看中间聚合的思想，但您的数据通常可能拥有超过三维的维度。扩展我们的食谱，我们可以通过部门、职位、经理和入职年份来“立方体化”平均工资的计算。

```sql
SELECT department_id, job_id, manager_id,
       extract(year from hire_date) as "START_YEAR", avg(salary)
FROM   hr.employees
GROUP  BY cube (department_id, job_id, manager_id, extract(year from hire_date));
```
此食谱的结果是对四个维度平均工资的考察！

##### 2-6. 在其他查询中使用聚合结果

### 问题

您希望将涉及聚合和分组的复杂查询的输出，用作另一个查询的源数据。

### 解决方案

Oracle 允许任何查询用作子查询或内联视图，包括那些包含聚合和分组函数的查询。这在您希望指定组级条件，并与其他表或查询的数据进行比较，且没有现成视图可用时特别有用。

对于我们的食谱，我们将使用一个包含按部门、职位和起始年份进行 rollup 的平均工资计算，如下面的 `SELECT` 语句所示。

```sql
select * from (
  select department_id as "dept", job_id as "job",
         to_char(hire_date,'YYYY') as "Start_Year", avg(salary) as "avsal"
  from   hr.employees
  group  by rollup (department_id, job_id, to_char(hire_date,'YYYY')))
salcalc where salcalc.start_year > '1990'
or salcalc.start_year is null
order by 1,2,3,4;
```

我们的食谱产生以下（节选的）输出：

dept job Start_Year avsal
---------- ---------- ----- ----------
10 AD_ASST     4400
10                  4400
20 MK_MAN 1996 13000
20 MK_MAN      13000
20 MK_REP 1997 6000
20 MK_REP       6000
…
              6473.36449
79 rows selected.

### 工作原理

我们的食谱使用子查询的聚合和分组结果作为内联视图，然后我们从中选择并应用进一步的条件。在这种情况下，我们可以通过使用更复杂的 `HAVING` 子句来避免子查询方法，如下所示。

```sql
having to_char(hire_date,'YYYY') > '1990'
or to_char(hire_date,'YYYY') is null
```

在这里避免子查询之所以可行，仅仅是因为我们将聚合与字面值进行比较。如果我们想查找那些在某个职位上有人曾担任过的部门中，该职位的平均值，我们就需要引用 `HR.JOBHISTORY` 表。根据业务需求，我们可能幸运地能够在一个语句中构建连接、聚合、分组和 `HAVING` 条件。通过将聚合和分组查询的结果视为另一个查询的输入，我们获得了更好的可读性，并且能够编码比 `HAVING` 子句所允许的更复杂的逻辑。

##### 2-7. 统计组和集合中的成员

### 问题

您需要统计一个组的成员、组的组以及其他基于集合的集合。同时，您需要根据其他数据动态地包含或排除单个成员和组。例如，您想根据员工在公司内获得的晋升次数，统计他们在组织任职期间担任过多少个不同的职位。

### 解决方案

Oracle 的 `COUNT` 功能可用于统计物化结果以及表中的实际行。下一个 `SELECT` 语句使用一个子查询来统计跨表持有的职位实例，然后汇总这些计数。实际上，这是对查询结果数据的计数的计数，而不是任何直接存储在 Oracle 中的内容。

```sql
select jh.JobsHeld, count(*) as StaffCount
from
  (select u.employee_id, count(*) as JobsHeld
   from
     (select employee_id from hr.employees
      union all
      select employee_id from hr.job_history) u
   group by u.employee_id) jh
group by jh.JobsHeld;
```

从该 `SELECT` 语句中，我们得到以下简明的摘要。

JOBSHELD STAFFCOUNT
---------- ----------



---------- ----------

1 99

2 5

3 3

大多数员工只有一份工作，有五人担任过两个职位，有三人每人担任过三个职位。

## 工作原理

我们这个方案的关键在于`COUNT`函数的灵活性，它的用途远不止于物理上计算表中的行数。你可以统计任何能在`结果集`中表示的事物。

这意味着你可以统计衍生数据、推断数据以及瞬时计算和判断的结果。

我们的方案使用了嵌套子查询并在两个层级上进行计数，最好从内向外逐步理解。

我们知道员工当前的职位记录在`HR.EMPLOYEES`表中，而在该组织内的每一个先前职位实例则记录在`HR.JOB_HISTORY`表中。我们不能简单地计算`HR.JOB_HISTORY`中的条目数然后为员工的当前职位加一，因为从未换过工作的员工在`HR.JOB_HISTORY`中没有记录。

因此，我们对`HR.EMPLOYEES`和`HR.JOB_HISTORY`中的`EMPLOYEE_ID`值执行`UNION ALL`操作，构建一个基础结果集，该结果集为员工担任过的每一个职位重复一次`EMPLOYEE_ID`。以下仅展示了内部`UNION ALL`语句的部分结果，以帮助你理解逻辑。

`EMPLOYEE_ID`

[www.it-ebooks.info](http://www.it-ebooks.info/)

**第 2 章 ■ 数据汇总与聚合**

…

### Union 与 Union All

记住`UNION`和`UNION ALL`的区别很有用。`UNION`会移除结果集中的重复条目（并在去重过程中对数据进行排序），而`UNION ALL`则保留所有源集中的所有值，包括重复值。在我们的方案中，如果使用`UNION`，会导致每个员工只统计一份工作，无论他们实际有过多少次晋升或换工作。

你会高兴地知道，`UNION`操作符是本书其他章节中许多方案的有用成分。

下一个子查询对最内层子查询派生的值进行聚合和分组，统计每个`EMPLOYEE_ID`出现的次数，以确定每个人担任过多少份工作。这是我们第一次将`COUNT`函数应用于另一个查询的结果，而不是表中的原始数据。

此子查询的部分输出如下所示。

`EMPLOYEE_ID JOBSHELD`

`----------- ----------`

`100 1`

`101 3`

`102 2`

`103 1`

`104 1`

…

我们最外层的查询也执行了直接的聚合和分组，再次对子查询的结果使用了`COUNT`函数——而该子查询本身就在对衍生数据进行计数。

##### 2-8. 查找表中的重复值和唯一值

### 问题

你需要测试某个给定的数据值在表中是否唯一——即它只出现一次。

[www.it-ebooks.info](http://www.it-ebooks.info/)

**第 2 章 ■ 数据汇总与聚合**

### 解决方案

Oracle 支持标准`SELECT`语句的`HAVING`子句和`COUNT`函数，两者结合可以识别表或结果集中的单个数据实例。以下`SELECT`语句解决了在`HR.EMPLOYEES`表中查找姓氏 Fay 是否唯一的问题。

```sql
select last_name, count(*)
from hr.employees
where last_name = 'Fay'
group by last_name
having count(*) = 1;
```

使用此方案，我们得到以下结果：

`LAST_NAME COUNT(*)`

`------------------------- ----------`

`Fay 1`

因为`LAST_NAME`值为`Fay`的恰好只有一个，所以我们得到的计数是 1，因此能看到结果。

### 工作原理

只有唯一的数据组合才会分组后计数为 1。例如，我们可以测试姓氏 King 是否唯一：

```sql
select last_name, count(*)
from hr.employees
where last_name = 'King'
group by last_name
having count(*) = 1;
```

此语句不返回任何结果，这意味着姓氏为`King`的人数不是 1；而是 0、2 或其他数字。该语句首先确定哪些行的`LAST_NAME`值为`King`。然后按`LAST_NAME`分组并统计遇到的匹配次数。最后，`HAVING`子句测试`LAST_NAME`为`King`的行数是否等于 1。只有满足条件的结果才会被返回，因此，只有当你看到结果时，该姓氏才是唯一的。

如果我们移除`HAVING`子句，如下面的`SELECT`语句，我们就能看到`HR.EMPLOYEES`表中有多少个 King。

```sql
select last_name, count(*)
from hr.employees
where last_name = 'King'
group by last_name;
```

`LAST_NAME COUNT(*)`

`------------------------- ----------`

`King 2`

[www.it-ebooks.info](http://www.it-ebooks.info/)

**第 2 章 ■ 数据汇总与聚合**

有两个人的姓氏是`King`，因此它不唯一，也就没有出现在我们之前唯一性的测试中。

同样的技术可以扩展到测试列的唯一组合。我们可以扩展方案来测试某人的完整姓名（基于`FIRST_NAME`和`LAST_NAME`的组合）是否唯一。这个`SELECT`语句在条件中包含了两个列，用于测试 Lindsey Smith 在`HR.EMPLOYEES`表中是否是一个唯一的全名。

```sql
select first_name, last_name, count(*)
from hr.employees
where first_name = 'Lindsey'
and last_name = 'Smith'
group by first_name, last_name
having count(*) = 1;
```

`FIRST_NAME LAST_NAME COUNT(*)`

`-------------------- ------------------------- ----------`

`Lindsey Smith 1`

你可以编写类似的方案，使用字符串拼接、自连接以及许多其他方法。

##### 2-9. 计算总计和小计

### 问题

你需要在各种环境中计算总计和小计，使用 SQL 的最基本功能。例如，你需要计算每个部门的人数以及总人数，并且希望这种方式能在各种版本的 Oracle 中无需更改即可运行。

### 解决方案

在那些你认为无法使用`ROLLUP`和`CUBE`等分析函数，或者受到许可或其他因素限制的情况下，你可以在单独的 SQL 语句中使用传统的聚合和分组技术，并用`UNION`组合结果，将所有逻辑合并到一个语句中。这个`SELECT`语句在一个查询中计算各部门的员工数，在另一个查询中计算所有员工的总数。

```sql
select nvl(to_char(department_id),'-') as "DEPT.", count(*) as "EMP_COUNT"
from hr.employees
group by department_id
union
select 'All Depts.', count(*)
from hr.employees;
```

方案的结果如下所示，为节省空间对输出进行了删节。

[www.it-ebooks.info](http://www.it-ebooks.info/)

**第 2 章 ■ 数据汇总与聚合**

`DEPT. EMP_COUNT`

`----------- ----------`

`10 1`

`100 6`

`110 2`

…

`90 3`

`All Depts. 107`

`13 rows selected.`

### 工作原理

此方案使用单独的查询来计算不同层级的不同聚合值，并使用`UNION`将结果组合成报告式的输出。实际上，生成了两组不同的结果。首先，使用以下`SELECT`语句完成按部门统计员工数。

```sql
select nvl(to_char(department_id),'-') as "DEPT.", count(*) as "EMP_COUNT"
from hr.employees
group by department_id;
```

注意这里使用了`TO_CHAR`函数将整数`DEPARTMENT_ID`值转换为字符等效值。这是为了确保最终的`UNION`操作不会受到隐式转换开销甚至转换错误的困扰。在本方案中，我们知道希望将字面短语`"All Depts."`与总员工数结合使用，并让它与`DEPARTMENT_ID`值显示在同一列中。

如果不进行转换，这会导致尝试从一个定义为整数的列和一个字面字符串形成联合。如果我们不进行转换，将会收到此错误。

`ORA-01790: expression must have same datatype as corresponding expression`

显然这不是一个有用的结果。在处理必须处理`NULL`值的联合时，你经常会看到这个错误。



我们还需要`TO_CHAR`转换与`NVL`空值测试函数协同工作，以将`DEPARTMENT_ID`为空的员工映射为“-”来表示无部门。这纯粹是出于美观原因，但你可以看到它如何为阅读你配方结果的人提供清晰度。这样，他们就不必猜测空白的或空的`DEPARTMENT_ID`意味着什么。

`UNION`中的第二个查询仅计算`HR.EMPLOYEES`表中的员工总数，并利用 Oracle 提供的灵活的字面量处理机制，将总数与值“所有部门”隐式分组。

```sql
select '所有部门', count(*)
from hr.employees
```

然后，`UNION`子句简单地将两个结果拼接在一起，生成你看到的输出。这等同于使用本章前面《在多个层级聚合数据》配方中介绍的`ROLLUP`子句。

[www.it-ebooks.info](http://www.it-ebooks.info/)

