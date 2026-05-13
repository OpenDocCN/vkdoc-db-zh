# 7. 与子查询的动态对话

本章探讨 SQL 中的动态对话概念，特别是 SQL 中的子查询。如前所述，子查询是强大的工具，允许你将查询嵌套在彼此内部以执行复杂的查询。本章的目的是概述子查询、其类型以及它们在动态对话中使用的叙述性示例。

## 子查询简介

在 SQL 中，`子查询` 是嵌套在另一个查询内部的查询。使用子查询的方式有很多，包括在 `SELECT`、`FROM`、`WHERE` 和 `HAVING` 子句中。根据其结构，子查询可以返回单个值、多个值，甚至整个表。因此，子查询可分为三种基本类型：`单行子查询`，返回单行，适用于比较运算符；`多行子查询`，提供多行，与 `IN`、`ANY` 和 `ALL` 等运算符配合使用；以及 `关联子查询`，基于子查询中的列引用，为外部查询的每一行执行一次。


## 第一个故事：繁忙的办公室

在一间繁忙的办公室里，初级数据分析师萨拉被指派了一项任务：研究员工与部门数据库。萨拉的经理需要针对员工薪资、所属部门和绩效等复杂问题找到答案。萨拉希望通过编写嵌套的 SQL 子查询来满足这些需求。

萨拉利用表 7-1 和 7-2 来提取与她所分配任务相关的洞察。

表 7-2
部门表

| ID | Name |
| --- | --- |
| 1 | IT |
| 2 | Sales |
| 3 | Marketing |

表 7-1
员工表

| ID | Name | Salary | department_id |
| --- | --- | --- | --- |
| 1 | John | 70,000 | 1 |
| 2 | Alice | 80,000 | 2 |
| 3 | Bob | 60,000 | 1 |
| 4 | Emma | 90,000 | 3 |
| 5 | Michael | 55,000 | 2 |

第一个问题是“谁的薪水最高”。萨拉要寻找的是公司里薪水最高的员工。为了找到答案，她使用了以下单行子查询：

```
SELECT name
FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);
```

在此查询中，子查询 (`SELECT MAX(salary) FROM employees`) 计算出表中最高的薪水，外部查询则检索拥有该薪水的员工姓名。如你所料，此查询返回单行结果，如表 7-3 所示。

表 7-3
最高薪资

| Name |
| --- |
| Emma |

第二个问题是“谁在销售部门工作”。萨拉要寻找的是一份包含所有在销售部门工作的员工的列表。萨拉使用了一个多行子查询来匹配部门 ID，从而找到答案：

```
SELECT name
FROM employees
WHERE department_id IN (SELECT id FROM departments WHERE name = 'Sales');
```

子查询 (`SELECT id FROM departments WHERE name = 'Sales'`) 检索销售部门的 ID，外部查询将此 ID 与 `Employees` 表中的 `department_id` 进行匹配。表 7-4 展示了结果。

表 7-4
谁在销售部门工作

| Name |
| --- |
| Alice |
| Michael |

第三个问题是“谁的薪资高于其所在部门的平均薪资”。萨拉要寻找的是一份包含所有薪资高于其所在部门平均薪资的员工的列表。萨拉使用了一个关联子查询来解决这个挑战：

```
SELECT emp.name
FROM employees AS emp
WHERE emp.salary > (
SELECT AVG(sub_emp.salary)
FROM employees AS sub_emp
WHERE sub_emp.department_id = emp.department_id
);
```

子查询 (`SELECT AVG(sub_emp.salary) FROM employees AS sub_emp WHERE sub_emp.department_id = emp.department_id)` 根据外部查询中每位员工的 `department_id` 动态调整，计算每个部门的平均薪资。别名 `emp` 代表外部查询中的 `Employees` 表，别名 `sub_emp` 代表子查询中的同一个表。这些别名清晰地表明了它们在查询中各自的角色。表 7-5 显示了哪些员工的薪资高于其所在部门的平均薪资。

表 7-5
薪资高于部门平均薪资的员工

| Name |
| --- |
| Alice |
| John |

## 动态对话与子查询

在 SQL 中，动态对话用于创建交互式和响应式的查询，这些查询可以根据用户的输入以及其他条件进行修改。由于这种动态特性，子查询扮演了重要的角色。SQL 查询可以利用动态对话来创建，这些对话能够根据参数或条件进行调整。因此，相同的查询可以根据用户的输入或数据库的状态以不同的方式执行。这种适应性对于构建能够实时操作和检索数据的稳健应用程序至关重要。

### 子查询在动态对话中的作用

子查询或嵌套查询是指嵌入在另一个 SQL 查询内部的查询。它们为条件逻辑和适应性提供了强大的机制，能够动态地过滤、聚合或转换数据。集成到动态对话中的子查询允许实现条件逻辑、实时适应和增强的模块化。表 7-6 简要描述了子查询中的条件逻辑、实时适应和增强模块化。

表 7-6
子查询中的条件逻辑、实时适应与增强模块化

| 方面 | 描述 | 示例 |
| --- | --- | --- |
| 条件逻辑 | 基于阈值或特定场景（例如用户偏好）动态调整结果。 | 检索子查询计算阈值的记录：`SELECT name``FROM employees``WHERE salary > (SELECT AVG(salary) FROM employees);` |
| 实时适应 | 将参数或应用程序输入与子查询结合使用，以提供上下文相关的结果。 | `SELECT name``FROM employees``WHERE salary > (SELECT MIN(salary) FROM employees WHERE department_id = 2);` |
| 增强模块化 | 将复杂的操作分解为更小、可重用的子查询，以提高清晰度和可维护性。 | `WITH DeptAvg AS (SELECT department_id, AVG(salary) AS avg_salary FROM employees GROUP BY department_id) SELECT e.name FROM employees e JOIN DeptAvg d ON e.department_id = d.department_id WHERE e.salary > d.avg_salary;` |

如表 7-6 所示，条件逻辑允许基于阈值（例如，将员工薪资与平均值进行比较）动态更改查询结果。另一方面，实时适应结合了用户输入和参数，确保结果能根据具体情况进行定制。例如，你可以对特定部门的员工应用最低工资过滤器。此外，可以使用公用表表达式（CTE）将大型查询划分为可单独重用的部分；这种方法简化了大型查询。例如，可以计算部门平均薪资。本节通过示例阐释了这些子查询，以展示其灵活性、效率和可维护性。



## 将子查询作为会话元素简介

如前所述，子查询是嵌入在另一个查询中的 SQL 查询。因此，它们可以被称为“会话元素”，因为它们允许查询以动态方式交互，从而实现查询之间的信息流动。在对话中，人们常常会提出后续问题以获取更具体的信息。子查询允许 SQL 查询相互构建，通过基于另一个查询的结果来细化或修改主查询，从而实现更复杂的数据检索。

SQL 中的子查询使查询能够动态交互和共享信息。子查询有多种类型，每种在增强数据流检索中扮演着不同的角色。单行子查询为问题提供直接、一次性的答案，而多行子查询则返回针对更广泛查询的一系列可能响应。多列子查询返回多列和多行，通常在比较元组、配对和分组时使用。相关子查询与外部查询进行持续的“对话”，因为它们依赖于每一行的上下文来执行。非相关子查询只执行一次，其结果被外部查询使用。最后，`FROM` 子句中的子查询充当基础上下文，准备主查询可以进一步分析的汇总数据。

通过将任务分解为更小、更易管理的步骤，SQL 能够构建复杂、精细的查询。以下章节将更详细地解释子查询的类型，并使用基于表 7-7 和 7-8 提供数据的查询示例。

表 7-8

部门表

| `department_id` | `department_name` | Location |
| --- | --- | --- |
| 1 | 销售部 | New York |
| 2 | 市场部 | Chicago |
| 3 | 信息技术部 | San Francisco |
| 4 | 人力资源部 | Boston |
| 5 | 运营部 | Los Angeles |
| 6 | 财务部 | New York |
| 7 | 物流部 | Chicago |
| 8 | 研发部 | San Francisco |
| 9 | 法务部 | Boston |
| 10 | 客户支持部 | Los Angeles |

表 7-7

员工表

| `employee_id` | `name` | `department_id` | `job_id` | `Salary` |
| --- | --- | --- | --- | --- |
| 101 | Alice Johnson | 1 | J001 | 55000 |
| 102 | Bob Smith | 2 | J002 | 48000 |
| 103 | Charlie Davis | 1 | J003 | 60000 |
| 104 | David Brown | 3 | J004 | 72000 |
| 105 | Emma Wilson | 2 | J001 | 45000 |
| 106 | Fiona Clark | 4 | J005 | 80000 |
| 107 | George Miller | 1 | J003 | 61000 |
| 108 | Hannah White | 4 | J006 | 85000 |
| 109 | Ian Thompson | 3 | J002 | 70000 |
| 110 | Julia Lewis | 2 | J004 | 52000 |

### 单行子查询

在单行子查询中，只返回一行和一列数据。当外部查询期望一个单一值进行比较时（例如使用 `=`、`<`、`>` 或 `!=` 等比较运算符时），经常会用到它。如果子查询返回多行，则会发生错误。例如，下面的嵌套查询使用一个单行子查询来查找销售部所有员工的姓名。

```sql
SELECT name
FROM employees
WHERE department_id = (SELECT department_id FROM departments WHERE department_name = 'Sales');
```

内部查询查找 `'Sales'` 部门的 `department_id`，外部查询则检索该部门员工的姓名。

如表 7-9 和 7-10 所示，内部查询返回销售部的 `department_id`，外部查询检索 `department_id = 1` 的员工姓名。

表 7-10

外部查询的输出

| `name` |
| --- |
| Alice Johnson |
| Charlie Davis |
| George Miller |

表 7-9

内部查询的输出

| `department_id` |
| --- |
| 1 |

### 多行子查询

多行子查询返回多行，但通常只返回单列。这些子查询与 `IN`、`ANY` 或 `ALL` 等运算符一起使用，以匹配子查询结果中的一个或多个值。要查找所有在位于纽约的部门工作的员工姓名，下面的嵌套查询使用了一个多行子查询。

```sql
SELECT name
FROM employees
WHERE department_id IN (SELECT department_id FROM departments WHERE location = 'New York');
```

子查询检索所有位于 `'New York'` 的部门的 ID，外部查询则查找在这些部门工作的员工。

表 7-11 和 7-12 展示了，内部查询返回了纽约各部门的 ID。在外部查询中，返回了部门 ID 为 1 或 6 的员工姓名。

表 7-12

外部查询的输出

| `name` |
| --- |
| Alice Johnson |
| Charlie Davis |
| George Miller |

表 7-11

内部查询的输出

| `department_id` |
| --- |
| 1 |
| 6 |

### 多列子查询

多列子查询返回多列和多行。它通常用于涉及元组、对或值组的复合比较，常与 `IN` 运算符或在 `JOIN` 条件中结合使用。例如，下面的嵌套查询旨在查找与公司当前空缺职位具有相同 `department_id` 和 `job_id` 的员工姓名。

```sql
SELECT name
FROM employees
WHERE (department_id, job_id) IN (SELECT department_id, job_id FROM job_openings WHERE status = 'Open');
```

子查询检索职位状态为开放的 `department_id` 和 `job_id` 对，外部查询则查找匹配这些对的员工。

表 7-13 显示了名为 `job_openings` 的表中的数据。

表 7-13

job_openings 表

| `department_id` | `job_id` | Status |
| --- | --- | --- |
| 1 | J003 | Open |
| 4 | J006 | Open |

子查询检索开放的职位，外部查询将员工与部门-职位对进行匹配。参见表 7-14 和 7-15。

表 7-15

外部查询输出

| `name` |
| --- |
| Charlie Davis |
| George Miller |
| Hannah White |

表 7-14

内部查询输出

| `department_id` | `job_id` |
| --- | --- |
| 1 | J003 |
| 4 | J006 |



## 关联子查询

关联子查询的执行依赖于外部查询。它会对外部查询处理的每一行数据进行一次求值。关联子查询更像是来回的对话，内部查询持续引用外部查询的列。例如，以下查询将每位员工的薪资与其**所在部门**的平均薪资（而非整个组织的平均薪资）进行比较，以识别薪资高于其所在部门平均水平的员工。

```sql
SELECT e.name
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees WHERE department_id = e.department_id);
```

对于外部查询中的每一位员工，内部查询都会计算该员工所在部门的平均薪资，并将其与该员工的薪资进行比较。为了帮助您更好地理解，表 7-16 展示了每位员工所在部门的平均薪资情况。

在内部查询中，计算每位员工所在部门的平均薪资；而在外部查询中，如表 7-17 所示，员工的薪资会与该平均值进行比较。

表 7-17：关联子查询的输出

| 姓名 |
| --- |
| Charlie Davis |
| George Miller |
| Julia Lewis |

表 7-16：每位员工所在部门平均薪资说明

| 姓名 | department_id | salary | avg_department_salary | salary_comparison |
| --- | --- | --- | --- | --- |
| Alice Johnson | 1 | 55000 | 58667 | 低于平均 |
| Charlie Davis | 1 | 60000 | 58667 | 高于平均 |
| George Miller | 1 | 61000 | 58667 | 高于平均 |
| Emma Wilson | 2 | 45000 | 48333 | 低于平均 |
| Bob Smith | 2 | 48000 | 48333 | 低于平均 |
| Julia Lewis | 2 | 52000 | 48333 | 高于平均 |

注：此表为便于理解而调整。要在 PostgreSQL 中编写查询来访问此表，您可以使用以下查询：

```sql
SELECT
    e.name,
    e.department_id,
    e.salary,
    dept_avg.avg_department_salary,
    CASE
        WHEN e.salary > dept_avg.avg_department_salary THEN '高于平均'
        ELSE '低于平均'
    END AS salary_comparison
FROM employees e
JOIN (
    SELECT department_id, AVG(salary) AS avg_department_salary
    FROM employees
    GROUP BY department_id
) dept_avg ON e.department_id = dept_avg.department_id;
```

## 非关联子查询

非关联子查询独立于外部查询。它只执行一次，其结果供外部查询使用。这类子查询更直接，它们不引用外部查询的列。例如，以下查询先计算整个公司的平均薪资，然后显示薪资高于该平均值的员工。

```sql
SELECT name
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

子查询一次性计算所有员工的平均薪资，外部查询则找出薪资高于该平均值的员工。

如表 7-18 和 7-19 所示，内部查询计算所有员工的平均薪资，外部查询检索薪资高于 62,800 美元的员工。

表 7-19：薪资高于 62,800 美元的员工

| name |
| --- |
| David Brown |
| Fiona Clark |
| Hannah White |
| Ian Thompson |

表 7-18：所有员工的平均薪资

| avg_salary |
| --- |
| 62800 |

## FROM 子句中的子查询

`FROM` 子句中的子查询通常称为派生表。它充当一个临时表，外部查询可以利用它来获取更有意义的洞察。这类子查询对于在主查询中使用之前预聚合数据非常有用。例如，以下嵌套查询计算每个部门的平均薪资，并提供部门名称及其平均薪资。

```sql
SELECT department_name, avg_salary
FROM (SELECT department_id, AVG(salary) AS avg_salary
      FROM employees
      GROUP BY department_id) AS dept_avg
JOIN departments d ON dept_avg.department_id = d.department_id;
```

子查询计算每个部门的平均薪资，外部查询检索部门名称及其对应的平均薪资。

在内部查询中，计算每个部门的平均薪资。然后执行 `JOIN` 操作连接 `Departments` 表。参见表 7-20 和 7-21。

表 7-21：部门名称及其对应的平均薪资

| department_name | avg_salary |
| --- | --- |
| Sales | 58667 |
| Marketing | 48333 |
| IT | 71000 |
| HR | 82500 |

表 7-20：每个部门的平均薪资

| department_id | avg_salary |
| --- | --- |
| 1 | 58667 |
| 2 | 48333 |
| 3 | 71000 |
| 4 | 82500 |

## 复杂对话：嵌套与多级子查询

在 SQL 中，嵌套子查询和多级子查询允许您通过编写多层查询来解决复杂的数据检索问题。每一层都进一步提炼数据，使其更具针对性和相关性。这可以看作类似于一场涉及对先前答案提出后续问题的对话，每一层都依赖于前一层的结果。

如前所述，嵌套子查询是置于另一个子查询或查询内部的子查询。在更复杂的场景中，多个子查询可能相互嵌套，从而形成多级子查询。

注：子查询可以嵌套在 `SELECT`、`FROM` 或 `WHERE` 子句中。多级子查询要求 SQL 先计算最内层的查询，然后将结果传递给下一个查询。最外层的查询使用最终结果来生成所需的输出。

### 两级子查询的通用语法

两级子查询是一种外部查询依赖于子查询结果，而该子查询本身又有另一个内部子查询的查询。

```sql
SELECT column1
FROM table1
WHERE column2 = (
    SELECT column3
    FROM table2
    WHERE column4 = (
        SELECT column5
        FROM table3
        WHERE condition
    )
);
```

首先，最内层的子查询运行并返回一个结果，然后中间子查询筛选其自身的数据，最后外部查询使用最终结果来筛选主表。

### 复杂的多级子查询

要编写复杂的嵌套 SQL 查询，必须从内向外进行，逐步构建和测试每一层以确保正确性。建议从编写和测试最内层的查询开始，一旦确认内部部分工作正常，再逐渐添加外部层。使用有意义的表别名可以提高代码的可读性，并使复杂查询更易于理解和维护。最佳建议是使用描述性别名来表示数据，而不是像 `t1` 或 `t2` 这样的通用名称。然而，必须考虑性能影响，因为嵌套子查询可能消耗大量计算资源。为了优化性能，您需要使用适当的数据库索引，这将在第 9 章介绍。

以下故事阐述了如何使用食品配送平台数据模型编写复杂的多级子查询。这个实际的业务场景展示了多级子查询的工作原理。



## 第二个故事：一个外卖配送平台

吉妮芙，一家外卖配送平台的数据分析师，当营销团队希望为一个忠诚度计划识别高价值客户时，她面临了一项具有挑战性的任务。她需要找到在各自城市消费额高于平均水平的客户，但仅限那些表现出持续订餐行为的客户。

她希望使用 `表 7-22`、`表 7-23` 和 `表 7-24` 来确定每个城市客户的平均总消费金额，找出在各自城市消费超过该平均值的客户，并筛选掉订单少于两单的客户。

### 表 7-22：客户表

| customer_id | 姓名 | 城市 |
| --- | --- | --- |
| 1 | 爱丽丝·约翰逊 | 纽约 |
| 2 | 鲍勃·史密斯 | 芝加哥 |
| 3 | 查理·戴维斯 | 波士顿 |
| 4 | 大卫·布朗 | 纽约 |
| 5 | 伊娃·怀特 | 芝加哥 |
| 6 | 弗兰克·格林 | 波士顿 |
| 7 | 格蕾丝·布莱克 | 纽约 |
| 8 | 汉娜·布鲁 | 芝加哥 |
| 9 | 艾薇·瑞德 | 纽约 |
| 10 | 杰克·格雷 | 芝加哥 |
| 11 | 利亚姆·耶洛 | 波士顿 |
| 12 | 米娅·珀普尔 | 纽约 |
| 13 | 诺亚·奥兰治 | 芝加哥 |
| 14 | 奥利维亚·平克 | 波士顿 |
| 15 | 保罗·西尔弗 | 纽约 |
| 16 | 奎因·戈尔德 | 芝加哥 |

### 表 7-23：订单表

| order_id | customer_id | restaurant_id | total_amount | order_date |
| --- | --- | --- | --- | --- |
| 1001 | 1 | 101 | 50 | 2025-01-01 |
| 1002 | 1 | 102 | 20 | 2025-01-02 |
| 1003 | 2 | 101 | 45 | 2025-01-03 |
| 1004 | 3 | 103 | 35 | 2025-01-04 |
| 1005 | 3 | 102 | 55 | 2025-01-05 |
| 1006 | 4 | 101 | 70 | 2025-01-06 |
| 1007 | 5 | 102 | 30 | 2025-01-07 |
| 1008 | 5 | 101 | 60 | 2025-01-08 |
| 1009 | 5 | 102 | 40 | 2025-01-09 |
| 1010 | 6 | 103 | 25 | 2025-01-10 |
| 1011 | 6 | 102 | 45 | 2025-01-11 |
| 1012 | 7 | 101 | 90 | 2025-01-12 |
| 1013 | 7 | 102 | 50 | 2025-01-13 |
| 1014 | 8 | 101 | 80 | 2025-01-14 |
| 1015 | 8 | 102 | 20 | 2025-01-15 |
| 1016 | 9 | 101 | 100 | 2025-01-16 |
| 1017 | 9 | 102 | 60 | 2025-01-17 |
| 1018 | 10 | 101 | 90 | 2025-01-18 |
| 1019 | 10 | 102 | 70 | 2025-01-19 |
| 1020 | 11 | 103 | 55 | 2025-01-20 |
| 1021 | 11 | 102 | 45 | 2025-01-21 |
| 1022 | 11 | 101 | 120 | 2025-01-22 |
| 1023 | 11 | 102 | 80 | 2025-01-23 |

### 表 7-24：餐厅表

| restaurant_id | 名称 | 城市 |
| --- | --- | --- |
| 101 | 披萨宫 | 纽约 |
| 102 | 寿司点 | 芝加哥 |
| 103 | 汉堡吧 | 波士顿 |

莎拉的业务问题是识别那些消费额超过所在城市平均值，但前提是他们下过超过两个订单的客户。

作为第一步，她将问题分解为以下子问题：

-   找出每个城市客户的平均总消费金额。
-   找出在自己居住城市消费超过该城市平均值的客户。
-   筛选出订单数量少于两单的客户。

为了解决这些问题，将运行多个子查询：

-   最内层的子查询将计算每个城市客户的平均总消费金额。
-   中间层的子查询将识别符合条件的客户。
-   最外层的查询将根据客户的订单数量进行筛选。

以下是对最内层子查询的逐步分解：

```sql
SELECT c.city, AVG(o.total_amount) AS avg_city_spending
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.city;
```

最内层的子查询计算每个城市客户的平均总消费金额。这涉及连接 `Customers` 表和 `Orders` 表，并按 `city` 进行分组。如 `表 7-25` 所示，此查询计算出了每个城市客户的平均总消费金额。

### 表 7-25：各城市客户的平均总消费金额

| 城市 | total_amount |
| --- | --- |
| 纽约 | 70 |
| 芝加哥 | 45 |
| 波士顿 | 90 |

中间层的子查询使用最内层查询的结果来筛选在各自城市消费超过平均值的客户。中间层子查询编写如下：

```sql
SELECT c.customer_id, c.name, c.city, SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city
HAVING SUM(o.total_amount) > (
    SELECT AVG(o2.total_amount)
    FROM customers c2
    JOIN orders o2 ON c2.customer_id = o2.customer_id
    WHERE c2.city = c.city
    GROUP BY c2.city
);
```

此查询识别了在各自城市消费超过该城市平均消费额的客户。它通过连接 `Customers` 表和 `Orders` 表来计算每位客户的总消费额。它按客户详细信息对结果进行分组。子查询通过连接相同的表但仅按 `city` 分组，计算了每个城市的平均消费额。`HAVING` 子句筛选出总消费额超过其所在城市平均消费额的客户。

**注意**

`HAVING` 子句是一个强大的 SQL 功能，用于基于聚合条件过滤 `GROUP BY` 操作的结果。`WHERE` 子句在分组前过滤单个行，而 `HAVING` 在聚合发生后过滤分组。当你需要基于诸如 `SUM()`、`COUNT()`、`AVG()`、`MAX()` 或 `MIN()` 等计算的条件时，`HAVING` 是必不可少的。

主查询和子查询之间的关联通过 `c2.city = c.city` 条件建立，这确保了每个客户只与自己城市的平均值进行比较。这使得该查询成为一个关联子查询，它针对外层查询中的每个客户组执行一次。

该查询计算了每个城市的平均总消费额。最后，最外层的查询如下：

```sql
SELECT c.customer_id, c.name, c.city, SUM(o.total_amount) AS total_spent, COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city
HAVING COUNT(o.order_id) > 2 AND SUM(o.total_amount) > (
    SELECT AVG(o2.total_amount)
    FROM customers c2
    JOIN orders o2 ON c2.customer_id = o2.customer_id
    WHERE c2.city = c.city
    GROUP BY c2.city
);
```

此查询确保只有那些下过超过两个订单——`COUNT(o.order_id) > 2`——且在各自城市消费超过平均值的客户才会被包含在结果中。最外层的查询检索出那些在各自城市消费超过平均值且下过超过两个订单的客户。

当这些层级结合起来时，便以一种为莎拉的问题提供最终答案的方式编写出了一个多层查询：

```sql
SELECT c.customer_id, c.name, c.city, SUM(o.total_amount) AS total_spent, COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city
HAVING COUNT(o.order_id) > 2 AND SUM(o.total_amount) > (
    SELECT AVG(o2.total_amount)
    FROM customers c2
    JOIN orders o2 ON c2.customer_id = o2.customer_id
    WHERE c2.city = c.city
    GROUP BY c2.city
);
```

这个 SQL 查询识别了一个外卖配送平台上的高价值客户。通过连接 `Customers` 表和 `Orders` 表计算每位客户的总消费额。在嵌套的子查询中，订单按 `city` 分组以计算平均总消费额。使用 `HAVING` 子句，主查询将每位客户的总消费额与其所在城市的平均值进行比较，但只包含订单数超过两个的客户。最终输出中，显示了那些在各自城市消费超过平均值的客户的姓名、城市及其总消费金额。见 `表 7-26`。

### 表 7-26：莎拉业务问题的最终答案

| customer_id | 姓名 | 城市 | total_spent | order_count |
| --- | --- | --- | --- | --- |
| 11 | 利亚姆·耶洛 | 波士顿 | 300 | 4 |
| 5 | 伊娃·怀特 | 芝加哥 | 130 | 3 |



为了提升多层嵌套查询的可读性，Sarah 使用了**公共表表达式**。CTE 是由`WITH`子句定义的临时结果集，可以在主查询中引用。通过将复杂逻辑拆分为更小、有名称的代码块，SQL 语句可以更轻松地被阅读、调试和维护。使用 CTE，Sarah 能够分离中间步骤、理清查询流程，并避免深层嵌套的子查询。

```sql
WITH city_spending AS (
SELECT city, AVG(total_customer_spent) AS avg_city_spending
FROM (
SELECT c.customer_id, c.city, SUM(o.total_amount) AS total_customer_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.city ) AS customer_totals
GROUP BY city
),
customer_spending AS (
SELECT c.customer_id, c.name, c.city, SUM(o.total_amount) AS total_spent, COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.city
HAVING COUNT(o.order_id) > 2 )
SELECT cs.customer_id, cs.name, cs.city, cs.total_spent, cs.order_count
FROM customer_spending cs
JOIN city_spending csp ON cs.city = csp.city
WHERE cs.total_spent > csp.avg_city_spending;
```

这个查询使用 CTE 来找出外卖平台上的高价值客户。它将分析分解为两个清晰的步骤：首先，`city_spending`计算每个城市的总消费额。其次，`customer_spending`计算单个客户的消费额，同时筛选出订单数超过两次的客户。最后，它连接这些临时表，以识别消费额超过其所在城市平均消费额的客户。它展示了客户的姓名、城市和总消费金额。该查询使用 CTE 以获得更好的可读性和可维护性，而之前的查询使用嵌套子查询，使其更为复杂。

## 常见误区

在处理 SQL 中的复杂嵌套查询和多层查询时，开发者经常会遇到各种挑战，这些挑战会影响性能、可读性和可维护性。本节详细概述了常见的误区及其解决方案。

### 可读性差

复杂嵌套查询最常见的问题之一是可读性差。当查询嵌套过深或过于复杂时，它们会变得难以理解和维护。这些查询在调试或修改时也可能具有挑战性。使用 CTE 将复杂查询分解为更小、更易管理的部分，可能是解决此问题的一个方法。CTE 能提高可读性，并更容易地隔离和排查查询的不同部分。

### 重复执行子查询

重复执行子查询会导致性能问题，因为在查询执行过程中相同的计算会进行多次。

多次运行子查询的问题会拖慢查询性能。为了解决这个问题，可以计算一次子查询，然后使用派生表或 CTE 在查询中多次引用它。

派生表是临时的、虚拟的表，它们是在 SQL 查询的`FROM`子句中由子查询创建的结果。这些表在数据库模式中并不存在，而是在查询运行时动态生成的。它们通过将复杂查询分解为更小、更易管理的组件来简化查询。

派生表的通用语法如下：

```sql
SELECT derived_table.col1, derived_table.col2
FROM (
SELECT col1, col2
FROM original_table
WHERE col3 > 100
) AS derived_table;
```

### 使用过多子查询而非连接

使用过多嵌套子查询而不是`JOIN`，会使查询不必要地复杂化和低效。使用多个嵌套子查询代替`JOIN`可能导致查询速度变慢、结构更复杂。为了提高性能和简化查询结构，最好在可能的情况下用`JOIN`替代子查询。

### 返回过多数据

返回大量结果集的子查询会对查询性能产生负面影响，因为它会减慢查询执行速度。可以在子查询中添加过滤器和 SQL `WHERE`子句，以限制检索的数据量。

### 忘记使用别名

在嵌套查询中忘记使用别名可能导致列引用不明确，使查询更难以阅读和维护。嵌套查询中不明确的列引用也可能引起混淆和错误。为防止此问题，请使用表别名来明确每个列所属的表。

表 7-27 总结了这些常见误区及其解决方案。

表 7-27：编写复杂嵌套查询和多层查询时的常见误区与解决方案

| 误区 | 问题 | 解决方案 |
| --- | --- | --- |
| 可读性差 | 查询变得难以阅读和维护 | 使用 CTE 提高可读性 |
| 重复执行子查询 | 子查询多次运行，降低性能 | 使用派生表或 CTE |
| 使用过多子查询而非连接 | 在`JOIN`足够时使用嵌套子查询 | 用`JOIN`替换子查询 |
| 返回过多数据 | 子查询返回大量结果集 | 在子查询中添加过滤器 |
| 忘记使用别名 | 嵌套查询中列引用不明确 | 使用表别名以明确 |

## 本章小结

本章探讨了利用子查询实现**动态对话**的概念，重点关注嵌套查询和多层查询。重点阐述了这些查询如何通过允许在单个语句中进行复杂的数据操作来增强 SQL 的灵活性。然而，本章也讨论了常见的误区，例如可读性差、因子查询重复执行导致的性能问题，以及处理大型结果集的挑战。通过应用 CTE 和表别名等最佳实践，分析师可以优化查询性能和可维护性。这些技术有助于编写更高效、可读性更强且更易维护的 SQL 代码，从而确保更好的数据分析和决策结果。

### 关键点

- SQL 中的动态对话涉及使用子查询来创建更灵活、适应性更强的查询，以响应不同的数据输入。这些对话允许以模块化方式构建 SQL 查询，通过在单个查询中实现复杂的数据操作，为 SQL 查询增添了灵活性。
- 多层嵌套查询通过将查询分解为分层结构，为解决复杂问题提供了强大的方法。这些查询使 SQL 用户能够在单个语句中处理多步数据转换，减少了对多个查询的需求。
- 使用 CTE 可以将复杂查询分解为更小、更易管理的部分，从而提高其可读性和可维护性。
- 通过用`JOIN`替换不必要的嵌套子查询来简化查询，可以同时提升性能和清晰度。
- 在子查询中应用过滤器可以减少处理的数据量，提高查询效率并聚焦于相关数据。

### 核心要点

- **利用子查询实现动态对话**：使用子查询来创建能动态响应不同数据输入的自适应查询。这种方法增强了模块化程度，使查询在不同场景下更加灵活和可复用。
- **多层嵌套查询**：可以使用多层嵌套查询在单个查询中处理复杂的数据转换。这种方法支持逐步执行流程，无需单独的查询，同时提升了性能和可维护性。

### 下一章预告

下一章“**条件逻辑在数据绘图中的应用**”将探讨用于动态数据输出的`CASE`语句，并讲解如何基于复杂数据集的分析来定制数据故事。


## 测试你的技能

某城市的一家藏书丰富、用户众多的公共图书馆，聘请杰克担任数据分析师。图书馆管理层希望通过了解借阅模式、热门图书和会员活动来改善用户体验。杰克的目标是使用包含子查询、嵌套查询和多级查询的动态对话，从图书馆数据库中提取洞察。表 7-28 展示了 `BookLoans` 表；表 7-29 展示了 `Borrowers` 表；表 7-30 展示了 `Books` 表。

表 7-30

图书表

| 列名 | 描述 |
| --- | --- |
| `book_id` | 每本书的唯一标识符（主键） |
| `book_title` | 书名 |
| `author` | 作者 |
| `genre` | 图书类型 |
| `published_year` | 出版年份 |

表 7-29

会员表

| 列名 | 描述 |
| --- | --- |
| `member_id` | 每位会员的唯一标识符（主键） |
| `name` | 会员全名 |
| `membership_type` | 会员类型（例如，普通会员、高级会员） |
| `signup_date` | 会员加入图书馆的日期 |

表 7-28

图书借阅表

| 列名 | 描述 |
| --- | --- |
| `loan_id` | 每次图书借阅的唯一标识符 |
| `member_id` | 每位会员的唯一标识符 |
| `book_title` | 所借图书的书名 |
| `borrow_date` | 图书借出日期 |
| `return_date` | 图书归还日期 |
| `loan_duration` | 借阅时长（天数）（`return_date` - `borrow_date`） |
| `borrow_count` | 该图书被借阅的次数 |

1.  确定哪些图书借阅频率最高。尝试编写一个使用子查询的语句，从 `BookLoans` 表中找出借阅次数最多的前五本书。
2.  确定哪些会员的平均借阅时长最长。尝试使用一个多级查询来计算每位会员的平均借阅时长，并识别出平均借阅时长最长的前三名会员。
3.  分析近期的借阅模式。尝试编写一个使用子查询的语句，列出最近的十次借阅记录，包括 `member_id` 和 `book_title`，并按 `borrow_date` 排序。
4.  进行借阅频率分析。尝试使用一个 CTE 来识别借阅图书超过 20 本的会员，并计算他们的平均借阅时长。
5.  比较借阅者在两个时间段的行为。尝试使用一个嵌套查询来比较上半年与下半年的图书借阅数量。

