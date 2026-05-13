# 第 2 章 ■ 汇总与聚合数据

```sql
create or replace function string_to_list
  (input_string varchar2)
  return varchar2
  parallel_enable
  aggregate using t_list_of_strings;
/
```

该函数获得一个名称`STRING_TO_LIST`，它接受一个`VARCHAR2`字符串作为输入，并返回一个作为输出。到目前为止，非常普通。接下来的两行是我们配方聚合行为的关键。

`PARALLEL_ENABLE`子句是完全可选的，它指示 Oracle，如果优化器决定走那条路径，那么底层聚合逻辑可以安全地并行执行。根据你的`ODCIAGGREGATEITERATE`函数中的逻辑，你可能有必须按特定顺序执行的特定操作，因此希望避免并行。启用并行处理也意味着在`ODCIAGGREGATEMERGE`成员函数中具备处理合并结果子集的逻辑。


`AGGREGATE USING T_LIST_OF_STRINGS` 子句是所有魔法发生的地方。这一行指令表明该函数本质上是聚合函数，并且应使用输入参数实例化一个 `T_LIST_OF_STRINGS` 类型的对象，从而启动实际的聚合工作。

■ 注意 Oracle 仅支持创建恰好接受一个输入参数并恰好返回一个输出参数的自定义聚合函数。这就是为什么在我们的示例中，你没有看到任何显式指令将 `INPUT_STRING` 参数传递给实例化的 `T_LIST_OF_STRINGS` 类型，也没有看到将 `T_LIST_OF_STRINGS.ODCIAGGREGATETERMINATE` 成员函数的返回值映射回 `STRING_TO_LIST` 函数返回值的指令。当 Oracle 看到聚合子句时，这些是它唯一能做的事情，因此当你使用聚合功能时，它们都是隐式完成的。

此后，新的 `STRING_TO_LIST` 函数的调用和使用方式就与其他聚合函数（如 `AVG`、`MAX`、`MIN` 等）非常相似。

##### 2-11. 访问后续或先前行中的值

### 问题

你想查询数据以产生有序的结果，但你希望在结果集中包含基于前后行的计算。例如，你想根据在时间上较早和较晚发生的事件，对事件式数据进行计算。

### 解决方案

Oracle 支持 `LAG` 和 `LEAD` 分析函数，利用前置/后置逻辑来访问表或表达式中的多行数据——并且你不需要诉诸于将源数据与自身进行连接 42

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 2 章 ■ 数据汇总与聚合

我们的示例假设你正试图解决一个业务问题：可视化员工随时间变化的雇佣趋势。`LAG` 函数可用于查看哪位员工的雇佣紧随另一位之后，也可用于计算雇佣之间的时间间隔。

```
select first_name, last_name, hire_date,
lag(hire_date, 1, '01-JUN-1987') over (order by hire_date) as Prev_Hire_Date,
hire_date - lag(hire_date, 1, '01-JUN-1987') over (order by hire_date) as Days_Between_Hires
from hr.employees
order by hire_date;
```

我们的查询返回了 107 行数据，按员工被雇佣的顺序（虽然不一定保留用于显示或其他目的的隐式排序）连接员工，并显示了每位员工加入组织之间的时间差。

FIRST_NAME LAST_NAME HIRE_DATE PREV_HIRE DAYS_BETWEEN
----------- ---------- --------- --------- ------------
Steven King 17-JUN-87 01-JUN-87 16
Jennifer Whalen 17-SEP-87 17-JUN-87 92
Neena Kochhar 21-SEP-89 17-SEP-87 735
Alexander Hunold 03-JAN-90 21-SEP-89 104
Bruce Ernst 21-MAY-91 03-JAN-90 503
...
David Lee 23-FEB-00 06-FEB-00 17
Steven Markle 08-MAR-00 23-FEB-00 14
Sundar Ande 24-MAR-00 08-MAR-00 16
Amit Banda 21-APR-00 24-MAR-00 28
Sundita Kumar 21-APR-00 21-APR-00 0

已选择 107 行。

你可以自己计算天数差，以确认 `LAG` 函数和差值算术运算确实如所述那样工作。例如，1990 年 1 月 3 日和 1991 年 5 月 21 日之间确实有 503 天。

### 工作原理

`LAG` 和 `LEAD` 函数与大多数其他分析和窗口函数类似，它们在查询的非分析基础部分完成后才开始操作。Oracle 对中间结果集执行第二次遍历以应用任何分析谓词。实际上，非分析组件会首先被评估，就像已经运行了这个查询一样：

```
select first_name, last_name, hire_date
-- 用于 Prev_Hire_Date 的占位符,
-- 用于 Days_Between_Hires 的占位符
from hr.employees;
```

此时的结果（如果你能看到的话）将如下所示： 43

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 2 章 ■ 数据汇总与聚合


## 第 2 章 ■ 数据汇总与聚合

Neena Kochhar 21-SEP-89（待定）

Alexander Hunold 03-JAN-90（待定）

Bruce Ernst 21-MAY-91（待定）

...

随后处理分析函数，提供您所看到的结果。我们的方案使用 `LAG` 函数来比较当前行结果与前一行的结果。理解 `LAG` 的最佳方式是了解其通用格式，其形式如下：

`lag (列或表达式, 前移位数, 首行默认值)`

`列或表达式` 基本不言自明，因为这是您希望 `LAG` 操作的表数据或计算结果。`前移位数` 部分表示 `LAG` 应参照的、相对于当前行的前几行。在我们的例子中，值 '1' 表示当前行的前一行。`LAG` 的默认值指定当没有“第零行”时，应使用什么值作为首行的参考值。我们选择了一个任意的日期 01-JUN-1987 作为组织成立的假想日期。您可以在此处使用任何日期、日期计算或返回日期的函数。

如果您未指定首行的参考值，Oracle 将提供一个 `NULL` 值。

接着，`OVER` 分析子句规定了应用分析函数时数据的排序方式，以及将数据分割为窗口或子集的任何分区（本方案中未展示）。细心的读者会意识到，这意味着我们的方案本可以包含一个通用的 `ORDER BY` 子句，该子句可以按照与 `LAG` 函数所使用的 `HIRE_DATE` 排序不同的顺序对数据进行排序呈现。这为您处理通用排序以及同一语句中不同的分析滞后和超前需求提供了最大的灵活性。我们将在本章后面展示一个这样的示例。请记住，您绝不应依赖分析函数使用的隐式排序。这种方式未来可能而且将会改变，因此强烈建议您在任何明确需要排序的地方始终包含 `ORDER BY` 子句。

### LEAD 函数

`LEAD` 函数的工作方式与 `LAG` 几乎完全相同，但它跟踪的是后续行而非前序行。我们可以改写我们的方案，以显示员工的雇佣信息以及下一位员工的 `HIRE_DATE`，并计算他们雇佣日期之间类似的经过时间窗口，如下 `SELECT` 语句所示：

```
select first_name, last_name, hire_date,
       lead(hire_date, 1, sysdate) over (order by hire_date) as Next_Hire_Date,
       lead(hire_date, 1, sysdate) over (order by hire_date) - hire_date
           as Days_Between_Hires
  from hr.employees;
```

既然您已经看过 `LAG` 示例，现在日期的排列模式就非常直观了。使用 `LEAD` 时，关键区别在于第三个参数中默认值的效果。

FIRST_NAME LAST_NAME HIRE_DATE NEXT_HIRE DAYS_BETWEEN
----------- ---------- --------- --------- ------------
Steven King 17-JUN-87 17-SEP-87 92
Jennifer Whalen 17-SEP-87 21-SEP-89 735
Neena Kochhar 21-SEP-89 03-JAN-90 104
Alexander Hunold 03-JAN-90 21-MAY-91 503

[www.it-ebooks.info](http://www.it-ebooks.info/)

Bruce Ernst 21-MAY-91 13-JAN-93 603
...
David Lee 23-FEB-00 08-MAR-00 14
Steven Markle 08-MAR-00 24-MAR-00 16
Sundar Ande 24-MAR-00 21-APR-00 28
Amit Banda 21-APR-00 21-APR-00 0
Sundita Kumar 21-APR-00 21-APR-09 3287.98

共选择 107 行。

与 `LAG`（其默认值为首行比较提供一个假想起点）相反，`LEAD` 使用默认值为前向链中的最后一行提供一个假想终点。在此方案中，我们比较的是员工雇佣之间经过了多少天。对我们来说，使用 `SYSDATE` 函数将最后雇佣的员工（在本例中是 Sundita Kumar）与当前日期进行比较是有意义的。这是一个快速简便的收尾，用于计算自雇佣最后一名员工以来经过的天数。

##### 2-12. 为查询结果中的行分配排名值

问题



#### 第 2 章：数据汇总与聚合

查询结果需要分配一个序号来表示它们在结果中的位置。你肯定不希望手动插入并跟踪源数据中的这些序号。

## 解决方案

Oracle 提供了 `RANK` 分析函数，用于为结果集中的行生成排名编号。`RANK` 作为一个常规的 OLAP 风格函数应用于某列或派生表达式。本方案中，我们将假设业务需求是按薪资从高到低对员工进行排名。

以下 `SELECT` 语句使用 `rank` 函数来分配这些排名值。

```sql
select employee_id, salary, rank() over (order by salary desc) as Salary_Rank from hr.employees;
```

我们的查询生成了结果，从月薪最高的 24000 开始，一直到第 107 位月薪 2100 的员工，如下面的节选结果所示。

| EMPLOYEE_ID | SALARY | SALARY_RANK |
|-------------|--------|-------------|
| 100         | 24000  | 1           |
| 101         | 17000  | 2           |
| 102         | 17000  | 2           |
| 145         | 14000  | 4           |
| 146         | 13500  | 5           |
| 201         | 13000  | 6           |
| 205         | 12000  | 7           |
| 108         | 12000  | 7           |
| 147         | 12000  | 7           |
| ...         | ...    | ...         |
| 132         | 2100   | 107         |

共选择了 107 行。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 工作原理

`RANK` 的行为类似于任何其他分析函数，在非分析处理完成后，对结果集进行第二次遍历操作。在本方案中，选取了 `EMPLOYEE_ID` 和 `SALARY` 值（没有 `WHERE` 谓词来过滤表数据，因此我们得到了组织内的每位员工）。然后分析阶段按薪资降序对结果进行排序，并从 1 开始计算结果的排名值。

请注意 `RANK` 函数如何处理相等的值。两名薪资为 17000 的员工被赋予相同的排名 2。下一位薪资为 14000 的员工，排名是 4。这被称为**稀疏排名（sparse ranking）**，其中并列值会“占用”名次位置。实际上，这意味着我们并列第二名的员工占用了第二和第三名的位置，下一个可用的排名就是 4。

你可以使用稀疏排名的替代方案，称为**密集排名（dense ranking）**。Oracle 通过 `DENSE_RANK` 分析函数支持这一点。观察一下当我们在本方案中切换到密集排名时发生的情况。

```sql
select employee_id, salary, dense_rank() over (order by salary desc)
as Salary_Rank
from hr.employees;
```

现在我们看到了“缺失”的连续排名值。

| EMPLOYEE_ID | SALARY | SALARY_RANK |
|-------------|--------|-------------|
| 100         | 24000  | 1           |
| 101         | 17000  | 2           |
| 102         | 17000  | 2           |
| 145         | 14000  | 3           |
| 146         | 13500  | 4           |
| 201         | 13000  | 5           |
| 205         | 12000  | 6           |
| 108         | 12000  | 6           |
| 147         | 12000  | 6           |
| 168         | 11500  | 7           |
| ...         | ...    | ...         |
| 132         | 2100   | 58          |

共选择了 107 行。

不同排名类型的经典使用场景是在体育比赛中。足球和其他运动在追踪球队在积分榜上的胜/负进展时通常使用稀疏排名。另一方面，奥运会等赛事在游泳等项目中出现并列情况时，倾向于使用密集排名。他们希望确保总是有金牌、银牌和铜牌获得者，即使出现并列情况需要颁发更多奖牌。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 注意

作者们一直希望，有一天写 SQL 语句能成为奥运会项目。我们一定会采用密集排名方法，以最大化我们获得奖牌的机会。

我们的方案使用了一个跨越所有员工的简单排名来确定薪资顺序。`RANK` 和 `DENSE_RANK` 都支持常规的分析扩展，允许我们对源数据进行分区，以便为每个数据子集生成排名。延续我们方案的主题，这意味着我们可以为每个部门内的薪资收入者从高到低分配一个排名。将分区引入查询如下所示：

```sql
Select department_id, employee_id, salary, rank() over
(partition by department_id order by salary desc) as Salary_Rank
From hr.employees
;
```

我们的结果现在显示了按部门划分的员工薪资排名。

| DEPARTMENT_ID | EMPLOYEE_ID | SALARY | SALARY_RANK |
|---------------|-------------|--------|-------------|



