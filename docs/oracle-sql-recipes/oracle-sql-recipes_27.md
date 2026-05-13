# 第 3 章 ■ 从多表查询

`GROUP BY`和`HAVING`子句用于挑选出共同的行。如果某给定行在两个表中各出现一次或数次，则该行的计数将相同，从而会被`HAVING`子句中的条件排除。如果计数不同，该行将随其在一个表中出现而另一个表中未出现的次数（反之亦然）一同显示在结果中。由于我们在此查询中包含了主键，因此您在每个表中最多只会看到一个非共同行。

图 3-4 展示了结果集的维恩图。

**图 3-4.** 用于查找组件查询之间唯一行的 Oracle 集合操作

使用标准集合表示法，您所寻找的结果可更简洁地表达为：

`NOT (Query1 INTERSECT Query2)`

虽然 Oracle 语法确实有`NOT`和`INTERSECT`，但 Oracle 集合运算符不支持使用`NOT`。因此，您必须按照解决方案中所示，使用`UNION ALL`和`GROUP BY`来编写查询，以提供图 3-4 所标识的结果。即使 Oracle 支持在集合运算符中使用`NOT`，您仍需使用提供的解决方案来确定每个非共同行在每个组件查询中存在多少个。

##### 3-10. 生成测试数据

### 问题

您希望组合两个或多个没有共同列的表，以生成测试数据或作为来自两个或多个表的所有可能值组合的模板。

例如，您正在编写一个嵌入式 Oracle 数据库应用程序，以帮助在下一场 21 点游戏中计牌，并且您希望以最少的努力生成 52 张牌组的模板表。

### 解决方案

在四种花色（红桃、梅花、方块和黑桃）的集合与 13 个点数（A、2-10、J、Q、K）之间使用`CROSS JOIN`结构。例如：

```sql
create table card_deck
(suit_rank varchar2(5),
 card_count number)
;

insert into card_deck
select rank || '-' || suit, 0 from
(select 'A' rank from dual
 union all
 select '2' from dual
 union all
 select '3' from dual
 union all
 select '4' from dual
 union all
 select '5' from dual
 union all
 select '6' from dual
 union all
 select '7' from dual
 union all
 select '8' from dual
 union all
 select '9' from dual
 union all
 select '10' from dual
 union all
 select 'J' from dual
 union all
 select 'Q' from dual
 union all
 select 'K' from dual)
cross join
(select 'S' suit from dual
 union all
 select 'H' from dual
 union all
 select 'D' from dual
 union all
 select 'C' from dual)
;

select suit_rank from card_deck;

SUIT_RANK
---------
A-S
2-S
3-S
4-S
5-S
6-S
. . .
10-C
J-C
Q-C
K-C

52 rows selected
```

### `DUAL` 表

`DUAL` 表在许多情况下都很方便。它在过去 20 年的每个 Oracle 版本中都可用，包含一行一列。当您不需要从任何特定表中检索数据而只想返回一个值时，它非常有用。您也可以使用 `DUAL` 表通过内置函数执行临时计算，如本例所示：

```sql
select sqrt(144) from dual;

SQRT(144)
---------
        12
```

如果您已经将花色和点数存储在两个现有表中，这条 SQL 语句同样有效。在这种情况下，您会引用这两个表，而不是编写那些涉及对`DUAL`表进行`UNION`操作的长子查询。例如：

```sql
select rank || '-' || suit, 0 from
  card_ranks
cross join
  card_suits
;
```

### 工作原理

使用`CROSS JOIN`（也称为笛卡尔积）很少是故意的。如果您没有在两个表之间指定连接条件，则结果中的行数将是每个表中行数的乘积。这通常不是期望的结果！作为一个通用规则，对于包含*n*个表的查询，您需要至少指定*n-1*个连接条件以避免笛卡尔积。



如果你的纸牌游戏通常会在牌组中使用一到两张王牌，你可以像这样轻松地将它们附加到末尾：
```sql
select rank || '-' || suit, 0 from
(select 'A' rank from dual
 union all
 select '2' from dual
 . . .
 select 'C' from dual)
union (select 'J1', 0 from dual)
union (select 'J2', 0 from dual)
;
```
使用 Oracle9 *i* 之前的语法（Oracle 专有语法），你可以重写原始查询如下：
```sql
select rank || '-' || suit, 0 from
card_ranks, card_suits
;
```
使用 Oracle 专有语法进行多表连接且没有 `WHERE` 子句虽然看似奇怪，但它确实能产生你想要的结果！

##### 3-11. 基于其他表中的数据更新行

#### 问题
你希望根据同一张表或另一张表中的单个值或聚合值，来更新表中的部分或所有行。用于更新查询的值依赖于被更新表中的列值。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第三章 ■ 从多张表中查询

#### 解决方案
使用关联更新将子查询中的值与 `UPDATE` 子句中的主表链接起来。在下面的示例中，你希望将所有员工的薪资更新为其部门平均薪资的 110%：
```sql
update employees e
set salary = (select avg(salary)*1.10
              from employees se
              where se.department_id = e.department_id)
;
```
```
107 行已更新
```

#### 工作原理
对于 `EMPLOYEES` 表中的每一行，当前薪资都会被更新为该部门平均薪资的 110%。由于该子查询包含聚合但没有 `GROUP BY`，它将始终返回一行。请注意，此查询展示了 Oracle 的读一致性特性正在生效：Oracle 在更新同一 `UPDATE` 语句中每位员工的薪资时，会保留每位员工的原始薪资值来计算平均值。

关联更新与关联子查询（本章其他地方会讨论）非常相似：你将查询主体部分的一个或多个列与子查询中的相应列链接起来。执行关联更新时，请特别注意更新了多少行；错误地编写子查询常常会用 `NULL` 或错误值更新表中的大多数值！

事实上，如果某位员工没有被分配部门，提供的解决方案将会失败。在这种情况下，关联子查询将返回 `NULL`，因为 `NULL` 部门值永远不会匹配任何其他也是 `NULL` 的部门值，因此该员工的 `SALARY` 将被设为 `NULL`。要解决此问题，请在 `WHERE` 子句中添加一个过滤器：
```sql
update employees e
set salary = (select avg(salary)*1.10
              from employees se
              where se.department_id = e.department_id)
where department_id is not null
;
```
```
106 行已更新
```
所有没有分配部门的员工将保留其现有薪资。另一种实现相同效果的方法是在 `SET` 子句中使用 `NVL` 函数来检查 `NULL` 值：
```sql
update employees e
set salary = nvl((select avg(salary)*1.10
                  from employees se
                  where se.department_id = e.department_id),salary)
;
```
```
107 行已更新
```
[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第三章 ■ 从多张表中查询
从 I/O 角度或重做日志文件使用情况来看，此选项可能不太理想，因为无论行是否有关联部门，所有行都将被更新。

##### 3-12. 在连接条件中操作和比较 NULL

#### 问题
你希望将连接列中的 `NULL` 值映射到一个默认值，该值将与被连接表中的一行匹配，从而避免使用外连接。

#### 解决方案
使用 `NVL` 函数将要连接到其父表的表的外键列中的 `NULL` 值进行转换。在此示例中，为每个部门安排了节日聚会，但有几位员工没有分配部门。这是其中一位：
```sql
select employee_id, first_name, last_name, department_id
from employees
where employee_id = 178
;
```
```
EMPLOYEE_ID FIRST_NAME              LAST_NAME                 DEPARTMENT_ID
------------- --------------- --------------- -----------------
        178 Kimberely              Grant
已选择 1 行
```
为确保每位员工都将参加节日聚会，在查询中将 `EMPLOYEES` 表中所有为 `NULL` 的部门代码转换为部门 110（会计部），如下所示：
```sql
select employee_id, first_name, last_name, d.department_id, department_name
from employees e
join departments d
on nvl(e.department_id,110) = d.department_id
;
```
```
EMPLOYEE_ID FIRST_NAME              LAST_NAME                 DEPARTMENT_ID DEPARTMENT_NAME
------------- --------------- --------------- ----------------- ------------------
        200 Jennifer                Whalen                              10 Administration
        201 Michael                 Hartstein                           20 Marketing
        202 Pat                     Fay                                 20 Marketing
        114 Den                     Raphaely                            30 Purchasing
        115 Alexander               Khoo                                30 Purchasing
. . .
[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第三章 ■ 从多张表中查询
        113 Luis                    Popp                               100 Finance
        178 Kimberely              Grant                              110 Accounting
        205 Shelley                 Higgins                            110 Accounting
        206 William                 Gietz                              110 Accounting
已选择 107 行
```

#### 工作原理
在连接条件中将包含 `NULL` 的列映射到非 `NULL` 值以避免使用 `OUTER JOIN`，可能仍然存在性能问题，因为在连接期间将不会使用 `EMPLOYEES.DEPARTMENT_ID` 上的索引（主要是因为 `NULL` 列不会被索引）。你可以通过使用基于函数的索引来解决这个新问题。FBI 在表达式上创建索引，如果该表达式出现在连接条件或 `WHERE` 子句中，优化器可能会使用该索引。以下是如何在 `DEPARTMENT_ID` 列上创建 FBI：
```sql
create index employees_dept_fbi on employees(nvl(department_id,110));
```
■ **提示** 从 Oracle Database 11 *g* 开始，你现在可以使用*虚拟列*作为 FBI 的替代方案。虚拟列源自常量、函数以及同一表中的列。你可以在虚拟列上定义索引，优化器将在任何查询中使用这些索引，就像使用常规列索引一样。

最终，处理连接条件中的 `NULL` 的关键在于了解你的数据。如果可能，避免在可能包含 `NULL` 的列上进行连接，或者确保你将用于连接的每个列在填充时都有默认值。如果该列必须包含 `NULL`，请在连接前使用 `NVL`、`NVL2` 和 `COALESCE` 等函数将 `NULL` 转换掉；你可以创建基于函数的索引来抵消在表达式上连接的任何性能问题。如果无法做到这一点，请了解你的数据库列中 `NULL` 的业务规则含义：它们是表示零、未知还是不适用？你的 SQL 代码必须反映可以包含 `NULL` 的列的业务定义。

[www.it-ebooks.info](http://www.it-ebooks.info/)

[www.it-ebooks.info](http://www.it-ebooks.info/)

