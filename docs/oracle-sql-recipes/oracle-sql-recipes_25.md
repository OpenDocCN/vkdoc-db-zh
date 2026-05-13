# 第二章：汇总与聚合数据

##### 2-13. 在组内查找第一个和最后一个值

### 问题

你想为一个组计算并显示聚合信息（如最小值和最大值），同时显示该组每个成员的详细信息。你希望避免重复显示聚合值和详细信息的工作。

### 解决方案

Oracle 提供了分析函数 `FIRST` 和 `LAST` 来计算任何有序序列中的起始值和结束值。重要的是，与 `MIN` 和 `MAX` 这类显式的聚合函数不同，这些分析函数的使用**不依赖**于分组（`GROUP BY`）。

对于我们的示例，我们假设一个具体问题：显示员工的薪资，同时显示其所在部门同事的最低和最高薪资。以下 `SELECT` 语句可以完成这项工作。

```sql
select department_id, first_name, last_name,
       min(salary)
         over (partition by department_id) "MinSal",
       salary,
       max(salary)
         over (partition by department_id) "MaxSal"
from   hr.employees
order  by department_id, salary;
```

这段代码输出所有员工，并显示他们的薪资，该薪资介于其部门内的最低和最高薪资之间，如下方的示例输出所示。

```
DEPARTMENT_ID FIRST_NAME LAST_NAME   MinSal  SALARY    MaxSal
------------- ---------- ---------- ------- ------- ----------
           10 Jennifer   Whalen        4400    4400       4400
           20 Pat        Fay           6000    6000      13000
           20 Michael    Hartstein     6000   13000      13000
           30 Karen      Colmenares    2500    2500      11000
           30 Guy        Himuro        2500    2600      11000
           30 Sigal      Tobias        2500    2800      11000
           30 Shelli     Baida         2500    2900      11000
           30 Alexander  Khoo          2500    3100      11000
           30 Den        Raphaely      2500   11000      11000
           40 Susan      Mavris        6500    6500       6500
…
107 rows selected.
```

### 工作原理

`FIRST` 和 `LAST` 分析函数的关键在于，它们允许你根据一组条件进行分组和排序，同时在查询的主体中自由地按不同方式排序，并可选择性地根据其他因素进行或不进行分组。

`OLAP` 窗口通过 `OVER` 子句 `over (partition by department_id) "MinSal"` 在每个部门上进行分区。

##### 2-14. 在移动窗口上执行聚合

### 问题

你需要基于相同的数据提供静态和移动的摘要或聚合。例如，在销售报告的一部分中，你需要提供销售订单金额的月度汇总，以及用于比较的三个月的移动平均销售额。

### 解决方案

Oracle 作为其分析函数集的一部分，提供了移动或滚动窗口函数。这使你能够引用结果集中的任意数量的前置行、当前行以及任意数量的后置行。我们最初的示例使用当前行和前三行来计算订单金额的滚动平均值。

```sql
select to_char(order_date, 'MM') as OrderMonth,
       sum(order_total) as MonthTotal,
       avg(sum(order_total))
         over
           (order by to_char(order_date, 'MM') rows between 3 preceding and current row) as RollingQtrAverage
from   oe.orders
where  order_date between '01-JAN-1999' and '31-DEC-1999'
group  by to_char(order_date, 'MM')
order  by 1;
```

我们在结果中看到了月份、相关合计以及计算出的三个月滚动平均值。

```
OR MONTHTOTAL ROLLINGQTRAVERAGE
-- ---------- -----------------
02      120281.6         120281.6
03      200024.1         160152.85
04        1636         107313.9
05      165838.2         121944.975
06      350019.9         179379.55
07      280857.1         199587.8
08      152554.3         237317.375
09      460216.1         310911.85
10       59123.6         238187.775
11      415875.4         271942.35
12       338672         318471.775
11 rows selected.
```

你可能注意到一月（`OrderMonth` 01）缺失了。这不是此方法的特性，而是因为 `OE.ORDERS` 表在 1999 年没有记录该月的订单。

### 工作原理

我们的滚动平均值 `SELECT` 语句首先选择了一些直接的数值。使用带有 `MM` 格式字符串的 `TO_CHAR()` 函数从 `ORDER_DATE` 字段中提取月份数字。我们选择月份数字而非名称，是为了让输出按预期的顺序排序。

接下来是对 `ORDER_TOTAL` 字段使用传统的 `SUM` 函数进行常规聚合。这部分很简单。然后我们引入了 `OLAP AVG` 函数，这是滚动平均值计算的核心。该语句部分如下所示：

```sql
avg(sum(order_total)) over (order by to_char(order_date, 'MM')
     rows between 3 preceding and current row) as RollingQtrAverage
```

所有这些文本都是为了生成结果列 `ROLLINGQTRAVERAGE`。分解各部分将说明每一部分如何为解决方案做出贡献。前导函数 `AVG(SUM(ORDER_TOTAL))` 表明我们将对 `ORDER_TOTAL` 值求和然后取平均。这在某种程度上是正确的，但 Oracle 计算的不是普通的平均值或总和。这些是 `OLAP AVG` 和 `SUM` 函数，因此它们的范围由 `OVER` 子句管理。

`OVER` 子句首先指示 Oracle 基于格式化后的 `ORDER_DATE` 字段的顺序执行计算——这就是 `ORDER BY TO_CHAR(ORDER_DATE, 'MM')` 所实现的——实际上按 02 到 12 的值排序计算（记住，数据库中没有 1999 年 1 月的数据）。最后，也是最重要的，`ROWS` 元素告诉 Oracle 应该在其上计算核心 `OLAP` 聚合函数的行窗口大小。在我们的案例中，这意味着应该对多少个月的 `ORDER_TOTAL` 值进行求和然后平均。我们的示例指示 Oracle 使用从倒数第三行到当前行的结果。这是对三个月滚动平均值的一种解释，尽管从技术上讲，它实际上生成的是四个月的平均值。如果你想要的是真正的三个月平均值——即前两个月加上当前月——你应该将 `ROWS BETWEEN` 元素改为：

```sql
rows between 2 preceding and current row
```

这引出了一个有趣的点。此示例假设你想基于历史数据计算滚动平均值。但一些业务需求要求一个滚动窗口来跟踪基于某个时间点之前**和**之后的数据的趋势。例如，我们可能想使用三个月窗口，但基于前一个月、当前月和后一个月的数据。示例的下一个版本正好展示了窗口函数的这个能力，关键更改已加粗显示。

```sql
select to_char(order_date, 'MM') as OrderMonth,
       sum(order_total) as MonthTotal,
       avg(sum(order_total))
         over
           (order by to_char(order_date, 'MM')
            rows between 1 preceding and 1 following) as AvgTrend
from   oe.orders
where  order_date between '01-JAN-1999' and '31-DEC-1999'
group  by to_char(order_date, 'MM')
order  by 1
/
```

我们的输出正如预期那样发生了变化，因为月度的 `ORDER_TOTAL` 值现在为了计算而被以不同的方式分组。

```
OR MONTHTOTAL  AVGTREND
-- ---------- ----------
02      120281.6 160152.85
03      200024.1 107313.9
04        1636  122499.433
05      165838.2 172498.033
06      350019.9 265571.733
07      280857.1 261143.767
08      152554.3 297875.833
09      460216.1 223964.667
10       59123.6 311738.367
11      415875.4 271223.667
12       338672  377273.7
11 rows selected.
```


##### 2-15. 基于列子集删除重复行

### 问题

需要根据表中仅在部分列上存在的重复值，对数据进行清洗。

### 解决方案

历史上，针对此问题有使用 `ROWNUM` 特性的 Oracle 专有解决方案。然而，如果你有多个重复组并且希望一次操作中删除多余数据，这种方法可能会变得笨拙和复杂。作为替代方案，你可以使用 Oracle 的 `ROW_NUMBER` OLAP 函数结合 `DELETE` 语句，以高效地在一次操作中删除所有重复项。

为了演示我们的方法，我们将首先引入几名新员工，他们的 `FIRST_NAME` 和 `LAST_NAME` 与一些现有员工相同。这些 `INSERT` 语句创建了有问题的重复数据。

```sql
insert into hr.employees
(employee_id, first_name, last_name, email, phone_number, hire_date, job_id, salary, commission_pct, manager_id, department_id)
Values
(210, 'Janette', 'King', 'JKING2', '650.555.8880', '25-MAR-2009', 'SA_REP', 3500, 0.25, 145, 80);

Insert into hr.employees
(employee_id, first_name, last_name, email, phone_number, hire_date, job_id, salary, commission_pct, manager_id, department_id)
Values
(211, 'Patrick', 'Sully', 'PSULLY2', '650.555.8881', '25-MAR-2009', 'SA_REP', 3500, 0.25, 145, 80);

Insert into hr.employees
(employee_id, first_name, last_name, email, phone_number, hire_date, job_id, salary, commission_pct, manager_id, department_id)
Values
(212, 'Allen', 'McEwen', 'AMCEWEN2', '650.555.8882', '25-MAR-2009', 'SA_REP', 3500, 0.25, 145, 80);

commit;
```

为了证明我们确实存在重复项，一个快速的 `SELECT` 查询显示了有问题的行。

```sql
select employee_id, first_name, last_name
from hr.employees
where first_name in ('Janette','Patrick','Allan')
  and last_name in ('King','Sully','McEwen')
order by first_name, last_name;
```

```
EMPLOYEE_ID FIRST_NAME LAST_NAME
----------- ----------- ----------
        158 Allan      McEwen
        212 Allan      McEwen
        210 Janette    King
        156 Janette    King
        211 Patrick    Sully
        157 Patrick    Sully
```

如果你在人力资源部门工作，或者你就是这些人中的一员，你可能会担心不可预测的后果，并希望看到重复项被移除。现在问题数据已经存在，我们可以引入 SQL 来删除“多余”的 Janette King、Patrick Sully 和 Allen McEwen。

```sql
delete from hr.employees
where rowid in
  (select rowid
    from
      (select first_name, last_name, rowid,
              row_number() over
                (partition by first_name, last_name order by employee_id)
                staff_row
        from hr.employees)
    where staff_row > 1);
```

运行后，这段代码确实声称删除了三行，大概就是我们的重复项。为了验证，我们可以重复之前的快速查询，查看哪些行匹配这三个名字。我们看到如下结果集。

```
EMPLOYEE_ID FIRST_NAME LAST_NAME
----------- ----------- -------------------------
        158 Allan      McEwen
        156 Janette    King
        157 Patrick    Sully
```

我们的 `DELETE` 操作成功了，它仅基于部分列子集找到了重复项。

### 工作原理

我们的方法同时使用了 `ROW_NUMBER` OLAP 函数和 Oracle 内部的 `ROWID` 值（用于唯一标识表中的行）。查询以你预想中的 `DELETE` 语法开始。

```sql
delete from hr.employees
where rowid in
  (… 嵌套子查询在这里 …)
```


正如您所料，我们要求 Oracle 删除`HR.EMPLOYEES`表中那些`ROWID`值与我们检测到的重复项匹配的行，检测标准基于对部分列的评估。在我们的案例中，我们使用子查询根据`FIRST_NAME`和`LAST_NAME`来精确识别重复项。

要理解嵌套子查询的工作原理，最简单的方法是从最内层的子查询开始，它看起来像这样。

```sql
select first_name, last_name, rowid,
row_number() over
(partition by first_name, last_name order by employee_id)
staff_row
from hr.employees
```

我们有意将`FIRST_NAME`和`LAST_NAME`列添加到这个最内层的子查询中，以便在梳理其逻辑时使这个方案更易于理解。严格来说，这些列对于逻辑来说是多余的，最内层的子查询完全可以不包含它们而达到相同效果。如果我们只执行这个最内层的查询（为清晰起见选择了额外的列），我们会看到以下结果。

```
FIRST_NAME    LAST_NAME       ROWID              STAFF_ROW
-----------   ------------    ------------------ ----------
...
Alexander     Khoo            AAARAgAAFAAAABYAAP          1
Janette       King            AAARAgAAFAAAABXAAD          1
Janette       King            AAARAgAAFAAAABYAA4          2
Steven        King            AAARAgAAFAAAABYAAA          1
...
Samuel        McCain          AAARAgAAFAAAABYABe          1
Allan         McEwen          AAARAgAAFAAAABXAAF          1
Allan         McEwen          AAARAgAAFAAAABYAA6          2
Irene         Mikkilineni     AAARAgAAFAAAABYAAa          1
...
...
Martha        Sullivan        AAARAgAAFAAAABYABS          1
Patrick       Sully           AAARAgAAFAAAABXAAE          1
Patrick       Sully           AAARAgAAFAAAABYAA5          2
Jonathon      Taylor          AAARAgAAFAAAABYABM          1
...
```

`110 rows selected.`

`HR.EMPLOYEES`表中的全部 110 名员工的`FIRST_NAME`、`LAST_NAME`和`ROWID`都被返回。`ROW_NUMBER()`函数随后根据`PARTITION BY`指令，在由`FIRST_NAME`和`LAST_NAME`驱动的集合上进行计算。这意味着，对于每一个唯一的`FIRST_NAME`和`LAST_NAME`组合，`ROW_NUMBER`都会开始一个行计数，我们将其别名为`STAFF_ROW`。当观察到一个新的`FIRST_NAME`和`LAST_NAME`组合时，`STAFF_ROW`计数器会重置为 1。

这样，第一个 Janette King 的`STAFF_ROW`值为 1，第二个 Janette King 条目的`STAFF_ROW`值为 2，如果有第三个和第四个这样的重复名字，它们将分别具有 3 和 4 的`STAFF_ROW`值。在为同名的员工编号后，我们进入下一个外层的子查询，它查询上面的结果。

```sql
select rowid
from
(select rowid,
row_number() over
(partition by first_name, last_name order by first_name, last_name)
staff_row
from hr.employees)
where staff_row > 1
```

这个外层查询看起来很简单，因为它确实很简单！我们只是从内层查询的结果中`SELECT`出那些计算出的`STAFF_ROW`值大于 1 的`ROWID`值。这意味着我们只选择第二个 Janette King、Allan McEwen 和 Patrick Sully 的`ROWID`值，就像这样。

```
ROWID
AAARAgAAFAAAABYAA4
AAARAgAAFAAAABYAA6
AAARAgAAFAAAABYAA5
```

有了这些`ROWID`值，`DELETE`语句就能准确知道哪些行是重复的，而这仅仅基于对`FIRST_NAME`和`LAST_NAME`的比较和计数。

这个方案的精妙之处在于，其基本结构可以很容易地转换为基于任意此类列子集的重复项来删除数据。格式保持不变，只需要更改表名和几个列名。可以将其视为适用于所有此类情况的伪 SQL 模板。

```sql
delete from <your_table_here>
where rowid in
(select rowid
from
(select rowid,
row_number() over
(partition by <first_duplicate_column>, <second_duplicate_column>, <etc.> order by <desired ordering column> )
duplicate_row_count
from <your_table_here> )
where duplicate_row_count > 1)
/
```

只需将占位符`<your_table_here>`替换为你的表名，并将用于确定重复项的列替换到相应的列占位符中，就大功告成了！

### 2-16\. 查找表中的序列缺口

**问题**

您想查找数据中数字序列或日期时间序列的所有缺口。这些缺口可能出现在记录某个操作发生的日期中，或者其他具有逻辑连续性的数据中。

**解决方案**

Oracle 的`LAG`和`LEAD` OLAP 函数允许您将当前结果行与前一行或后一行进行比较。

`LAG`的通用格式如下：

`LAG(列或表达式, 前置行偏移量, 第一行的默认值)`

`列或表达式`是要与滞后（前置）值进行比较的值。`前置行偏移量`表示`LAG`应该针对当前行之前的第几行。我们在下面的代码中使用'1'来表示当前行的前一行。`第一行的默认值`指定了作为第一行的前导值，因为表中没有第 0 行。我们指示 Oracle 使用 0 作为锚点默认值，以处理我们查找一个月第一天之前日期的情况。

`WITH`查询别名方法几乎可以在所有使用子查询的情况下使用，将子查询细节提前到主查询之前。这有助于提高可读性，并方便日后需要时进行代码重构。

这个方案用于查找 1999 年 11 月订单序列中的日期缺口：

```sql
with salesdays as
(select extract(day from order_date) next_sale,
lag(extract(day from order_date),1,0)
over (order by extract(day from order_date)) prev_sale
from oe.orders
where order_date between '01-NOV-1999' and '30-NOV-1999')
select prev_sale, next_sale
from salesdays
where next_sale - prev_sale > 1
order by prev_sale;
```

我们的查询揭示了 1999 年 11 月订单之间的日期缺口。

```
PREV_SALE   NEXT_SALE
----------  ----------
1          10
10          14
15          19
20          22
```

结果显示，在月初第一天记录了一个订单后，直到 10 号才有后续订单。然后出现了为期四天的缺口直到 14 号，依此类推。精明的销售经理很可能会利用这些数据来询问销售团队在这些缺口日做了什么，以及为什么没有订单进来！

**工作原理**

查询首先使用`WITH`子句以一种"先别名后引用"的方式为一个子查询命名。然后，这个子查询通过一个别名（本例中为`SALESDAYS`）被引用。

`SALESDAYS`子查询计算两个字段。首先，它使用`EXTRACT`函数从`ORDER_DATE`日期字段中返回数字形式的日值，并将此数据标记为`NEXT_SALE`。然后，`lag` OLAP 函数用于计算结果中前一行`ORDER_DATE`在当月的日期数字（同样使用`EXTRACT`方法），这成为`PREV_SALE`结果值。当您将子查询 select 语句的输出可视化时，这一点会更清楚：

```sql
select extract(day from order_date) next_sale,
lag(extract(day from order_date),1,0)
over (order by extract(day from order_date)) prev_sale
from oe.orders
where order_date between '01-NOV-1999' and '30-NOV-1999'
```

如果独立执行，结果将如下所示。

```
NEXT_SALE   PREV_SALE
----------  ----------
1           0
10          1
10          10
10          10
14          10
14          14
15          14
19          15
…
```

从滞后中的锚点值 0 开始，我们看到一笔销售的当月日期作为`NEXT_SALE`，而上一笔销售的日期作为`PREV_SALE`。您可能已经能直观地发现缺口，但让 Oracle 为您完成这项工作要容易得多。这就是我们外层查询执行其非常简单的算术运算的地方。

驱动`SALESDAYS`子查询的查询基于以下条件从结果中选择`PREV_SALE`和`NEXT_SALE`值：

```sql
where next_sale - prev_sale > 1
```



我们知道，如果销售额之间的间隔超过一天，那么这些销售日就不是连续的。最后，我们通过按`PREV_SALE`列对结果进行排序来结束，这样我们就能得到一个从月初到月末的自然顺序。

我们的查询也可以按照传统方式编写，将子查询放在`FROM`子句中，如下所示。

```sql
select prev_sale, next_sale
from (select extract(day from order_date) next_sale,
      lag(extract(day from order_date),1,0)
      over (order by extract(day from order_date)) prev_sale
      from oe.orders
      where order_date between '01-NOV-1999' and '30-NOV-1999')
where next_sale - prev_sale > 1
order by prev_sale
/
```

所采取的方法主要是一个风格和可读性的问题。在那些能极大提高 SQL 语句可读性的情况下，我们更倾向于使用`WITH`方法。

[www.it-ebooks.info](http://www.it-ebooks.info/)

[www.it-ebooks.info](http://www.it-ebooks.info/)

`第三章`

■ ■ ■

`多表查询`

从 Oracle 表中查询数据可能是作为开发人员或数据分析师，甚至可能是作为 DBA（尽管对于 ETL（提取、转换和加载）工具专家来说可能不是）将执行的最常见任务。通常，您可能只查询一个表的一小部分行，但迟早您必须将多个表连接在一起。这就是关系数据库的魅力所在，因为访问数据的路径不是固定的：您可以连接具有公共列的表，甚至可以连接没有公共列的表（但您需要自担风险！）。

在本章中，我们将介绍连接两个或更多表并根据所需行在两个表中是否存在（等值连接）、可能仅存在于一个表或另一个表中的行（左外连接或右外连接）或连接两个表并包括两个表中的所有行并在可能时匹配（全外连接）来检索结果的解决方案。

但是等等，还有更多！Oracle（以及 SQL 语言标准）包含许多构造，帮助您根据另一个表中具有与查询中选定行相同列值的相同行的存在，从表中检索行。这些构造包括`INTERSECT`、`UNION`、`UNION ALL`和`MINUS`操作符。在某些情况下，使用这些操作符的查询结果可以通过标准的表连接语法获得，但如果您处理的列不止几个，查询就会变得笨拙、难以阅读和维护。

您可能还需要根据另一个表中的匹配或不匹配值来更新一个表中的行，因此我们还将提供一些关于使用`IN`/`EXISTS` SQL 构造进行关联查询和关联更新的方法。

当然，关于表操作的讨论如果不深入查询世界的“野孩子”——笛卡尔连接，就不算完整。有些情况下，您希望在没有连接条件的情况下连接两个或多个表，我们将为您提供针对该场景的解决方案。

本章中的大多数示例都基于在安装 Oracle 数据库时指定“Include Sample Schemas”时创建的`EXAMPLE`表空间中的模式。要理解本章中的解决方案，不需要这些示例模式，但它们为您提供了在预填充的表集上试用解决方案的机会，甚至可以进一步深入研究表连接的复杂性。

[www.it-ebooks.info](http://www.it-ebooks.info/)

`第三章 ■ 多表查询`

##### 3-1. 连接两个或多个表中的对应行

**问题**

您希望返回两个或多个表中具有一个或多个公共列的行。例如，您可能希望根据公共列（但不是所有公共列）连接`EMPLOYEES`和`DEPARTMENTS`表，并返回员工及其部门名称的列表。

**解决方案**

如果您使用的是 Oracle 数据库 9*i*或更高版本，可以使用带`USING`子句的 ANSI SQL 99 连接语法。例如，Oracle 示例模式中的`EMPLOYEES`和`DEPARTMENTS`表具有公共的`DEPARTMENT_ID`列，因此查询如下所示：

```sql
select employee_id, last_name, first_name, department_id, department_name
from employees
join departments using(department_id)
;
```

```
EMPLOYEE_ID LAST_NAME                 FIRST_NAME           DEPARTMENT_ID DEPARTMENT_NAME
------------ ------------------------- -------------------- ------------------ --------------------
         200 Whalen                    Jennifer                       10 Administration
         201 Hartstein                 Michael                        20 Marketing
         202 Fay                       Pat                            20 Marketing
         114 Raphaely                  Den                            30 Purchasing
         115 Khoo                      Alexander                      30 Purchasing
. . .
         113 Popp                      Luis                          100 Finance
         205 Higgins                   Shelley                       110 Accounting
         206 Gietz                     William                       110 Accounting
106 rows selected
```

该查询从`EMPLOYEES`表中检索大部分结果，并从`DEPARTMENTS`表中检索`DEPARTMENT_NAME`列。

**工作原理**

Oracle 数据库默认安装时提供的示例模式为尝试一些 Oracle 功能提供了一个良好的起点。Oracle 数据库附带了几个示例模式，如`HR`、`OE`和`BI`，不仅展示了数据库模式之间的关系，还展示了一些不同的功能，如索引组织表（IOT）、基于函数的索引、物化视图、大对象（`BLOB`和`CLOB`）以及 XML 对象。

以下是`EMPLOYEES`和`DEPARTMENTS`表的结构：

[www.it-ebooks.info](http://www.it-ebooks.info/)

`第三章 ■ 多表查询`

```sql
describe employees
```

```
Name                                      Null?    Type
----------------------------------------- -------- ----------------------------
EMPLOYEE_ID                               NOT NULL NUMBER(6)
FIRST_NAME                                         VARCHAR2(20)
LAST_NAME                                NOT NULL VARCHAR2(25)
EMAIL                                    NOT NULL VARCHAR2(25)
PHONE_NUMBER                                      VARCHAR2(20)
HIRE_DATE                                NOT NULL DATE
JOB_ID                                   NOT NULL VARCHAR2(10)
SALARY                                            NUMBER(8,2)
COMMISSION_PCT                                    NUMBER(2,2)
MANAGER_ID                                        NUMBER(6)
DEPARTMENT_ID                                     NUMBER(4)

11 rows selected
```

```sql
describe departments
```

```
Name                                      Null?    Type
----------------------------------------- -------- ----------------------------
DEPARTMENT_ID                             NOT NULL NUMBER(4)
DEPARTMENT_NAME                           NOT NULL VARCHAR2(30)
MANAGER_ID                                         NUMBER(6)
LOCATION_ID                                        NUMBER(4)

4 rows selected
```

还有其他三种基本方法可以在`DEPARTMENT_ID`列上连接这两个表。其中一种是 ANSI SQL 99 之前的方法，另一种更适用于连接表中列名不一致的情况，还有一种则非常危险，正如您将看到的。

使用“旧式”连接语法时，将连接条件包含在`WHERE`子句中，如下所示：

```sql
select employee_id, last_name, first_name, e.department_id, department_name
from employees e, departments d
where e.department_id = d.department_id
;
```

虽然这种方法有效，并且从执行计划的角度来看效率很高，但旧语法可能难以阅读，因为您将表连接条件与任何过滤条件混合在一起。它还迫使您在`SELECT`子句中为连接列指定限定符。

另一种 ANSI SQL 99 连接表的方法使用`ON`子句，如下所示：

```sql
select employee_id, last_name, first_name, e.department_id, department_name
from employees e
join departments d
on e.department_id = d.department_id
;
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

`第三章 ■ 多表查询`

■ **注意** 在多表查询中，如果您希望更加详细或希望非常清楚地表明查询是等值连接而非外连接或笛卡尔积，也可以使用`INNER JOIN`关键字代替简单的`JOIN`。

当连接的列具有相同的名称时，与`USING`子句相比，`ON`子句的可读性较差（通常需要输入更多字符！）。它与使用 ANSI SQL 99 之前语法具有相同的缺点，即您必须使用别名来限定连接列或任何其他同名的列。

最后，通过使用`NATURAL JOIN`子句代替`JOIN ... USING`子句，可以使`EMPLOYEE/DEPARTMENTS`查询的语法更加简单，如本示例所示：

```sql
select employee_id, last_name, first_name, department_id, department_name
from employees natural join departments
;
```



