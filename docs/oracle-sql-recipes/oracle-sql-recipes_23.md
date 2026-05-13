# 第 2 章 ■ 汇总与聚合数据

我们现在可以使用我们的自定义聚合函数，从查询中生成的任何字符串集合或子集来构建列表。在此配方中，我们希望返回分配给特定经理的所有员工的姓氏。

```sql
select manager_id, string_to_list (last_name) as employee_list
from hr.employees
group by manager_id;
```

我们的新自定义聚合函数奏效了，生成了类似列表输出的姓氏以及经理的 ID。

```
MANAGER_ID EMPLOYEE_LIST
---------- ------------------------------------------------------------
       100 Hartstein , Kochhar , De Haan , Fripp , Kaufling , Weiss ...
       101 Whalen , Greenberg , Higgins , Baer , Mavris
       102 Hunold
       103 Ernst , Pataballa , Lorentz , Austin
       108 Faviet , Chen , Sciarra , Urman , Popp
...
```

### 工作原理

不要被跳转到 PL/SQL，或者配方的长度吓到。事实上，上面的大部分代码是 Oracle 提供的样板模板，旨在帮助快速轻松地构建自定义聚合。

这个配方构建了创建自定义聚合所必需的三个组件，然后在 SQL 语句中使用它，按经理列出员工姓氏。首先，它定义了一个自定义类型，其中定义了 Oracle 对自定义聚合要求的四个规定函数。它们是：

`ODCIAggregateInitialize`：此函数在处理最开始时被调用，并设置聚合类型的新实例——在我们的例子中是`T_LIST_OF_STRINGS`。这个新实例初始化了其成员变量（如果有），并为主聚合阶段做好准备。

`ODCIAggregateIterate`：此函数包含了自定义聚合的核心功能。此函数中的逻辑将为驱动查询中传递的每个数据值调用。在我们的例子中，这是获取每个字符串值并将其附加到现有字符串列表的逻辑。

`ODCIAggregateMerge`：这是一个半可选函数，用于控制 Oracle 的行为，如果 Oracle 决定并行执行聚合，使用多个并行从属进程处理部分数据。虽然你不必启用并行选项（见下文），但你仍然需要包含这个函数。在我们的例子中，如果我们的列表创建被拆分成并行任务，我们只需要在并行过程结束时连接子列表。

`ODCIAggregateTerminate`：这是最后一个函数，用于将结果返回给调用的 SQL 函数。在我们的例子中，它返回在迭代和并行合并阶段构建的全局变量`STRING_LIST`。

我们现在已经构建了聚合的机制。配方的下一部分构建了你可以调用以在数据上实际运行你强大的新聚合功能的函数。我们创建了一个具有许多你期望的常规特性的函数：

[www.it-ebooks.info](http://www.it-ebooks.info/)

