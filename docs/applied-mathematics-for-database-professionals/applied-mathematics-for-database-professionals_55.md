# 第十章 数据操作

#### 规范概述

规范 `FEa` 使用基于 `dbs(REG)` 的查询结果来重新初始化 `REG` 表结构。该查询检索所有应*保留*在数据库状态中的注册记录；也就是说，所有不满足以下条件的元组：注册的学生是 `3124`，且对应一个未来开课、状态为 `confirmed` 或 `scheduled` 的课程。

`FEb` 执行类似操作，只是现在的查询基于 `REG` 和 `OFFR` 的连接。

请注意，与 `FEa` 相比，本规范中我们根据德摩根定律重写了否定条件。该查询现在检索所有满足以下任一条件的注册记录：不是为学生 `3124` 注册的，或者对应一个过去（包括今天）开课的课程，或者状态为 `canceled`。

`FEc` 指定此事务的方式可能是最直观的。它指定一个包含所有需要删除的注册记录的表，然后使用差集运算符 (`−`) 将其从 `dbs(REG)` 中减去。这种指定方式与使用 SQL 指定此事务的方式（参见 `SEI`）非常相似。

正如你现在所理解的，在我们的形式化方法中，事务是通过为每个可能的起始状态定义其终止状态来指定的。这涉及为每个表结构指定一个表。相比之下，使用 SQL 时，你只需指定事务实现的*数据变化*；不需要指定保持不变的数据。

此刻你可能在想，形式化指定事务是相当繁琐的。

然而，你应该意识到，前述事务 `ETX2` 的 SQL 表达式实际上是一种简写表示法。它指定了 SQL 表变量 `REG` 被（重新）赋值为其起始状态的值减去一个查询的结果集，该查询本质上由 `DELETE` 语句的 `WHERE` 子句指定。

##### 注意

使用本书介绍的数学方法，你也可以开发形式化的简写表达式。Bert De Brock 在他的著作《*Foundations of Semantic Databases*》（Prentice Hall, 1995）中为常见类型的事务（例如从表结构中删除元组）开发了这些简写。然而，因为这涉及引入一些更复杂的数学概念，我们不会在本书中开发形式化的简写。我们建议感兴趣的读者参考 De Brock 著作的第 7 章。

#### 事务规范示例

关于在代码清单 10-2 中指定 `ETX2` 的几种方式，有一个要点需要说明。它们都涉及 `OFFR` 表结构来确定课程是否具有 `scheduled` 或 `confirmed` 状态。实际上，这种涉及并非必需。给定约束 `POD6`（已取消的课程不能有注册记录），结合 `OFFR` 中 `STATUS` 属性的属性值集（仅允许三个值：`scheduled`、`confirmed` 和 `canceled`），你可以*推断*出，如果一个课程有注册记录，那么该课程的状态将是 `scheduled` 或 `confirmed`。你无需在事务中指定这一点。

以下是事务 `ETX2` 的等效形式化规范和 SQL 规范：

**`FEa`**:
```math
dbs↓{EMP,SREP,MEMP,TERM,DEPT,GRD,CRS,OFFR,HIST}
∪
{ (REG; { r | r∈dbs(REG) ∧ ¬ ( r(STUD) = 3124 ∧ r(STARTS) > sysdate ) } ) }
```

**`SEII`**:
```sql
delete from REG r
where r.STUD = 3124
and r.STARTS > sysdate
```

你可以认为这些规范更高效；如果以这种不太复杂的方式指定事务，数据库管理系统 (DBMS) 执行它时可能需要的资源更少。

使用已建立的、已知的数据完整性约束知识，将形式化规范——无论是事务、查询，甚至是约束——重写为不太复杂的规范，被称为*语义优化*。

##### 注意

一个真正的关系数据库管理系统应该能够自动为我们执行这种语义优化。不幸的是，能够执行如此复杂语义优化的 DBMS 目前仍不可用。

这主要是因为当前的 DBMS 对声明式指定的数据完整性约束支持仍然很差。这种对声明式支持的不足也是本书设有第 11 章的原因。

#### 更新事务示例

另一个常见的事务是更新数据库中的元组。代码清单 10-4 给出了一个更新某些课程最大容量的示例。它将所有计划在阿姆斯特丹开课的未来课程的最大容量翻倍。

**代码清单 10-4.** *事务 `ETX3(dbs)`*

**`FEa`**:
```math
dbs↓{SREP,MEMP,TERM,EMP,DEPT,GRD,CRS,REG,HIST}
∪
{ (OFFR; { o | o∈dbs(OFFR) ∧ ¬ (o(STARTS) > sysdate ∧ o(LOC) = 'AMSTERDAM') }
∪
{ o↓{COURSE,STARTS,STATUS,TRAINER,LOC}
∪ { (MAXCAP; 2 * o(MAXCAP)) }
| o∈dbs(OFFR) ∧ o(STARTS) > sysdate ∧ o(LOC) = 'AMSTERDAM' }
) }
```

**`SEI`**:
```sql
update OFFR o
set o.MAXCAP = 2 * o.MAXCAP
where o.STARTS > sysdate
and o.LOC = 'AMSTERDAM'
```

规范 `FEa` 清楚地表明，更新元组的子集可以被视为首先删除这些元组，然后使用更新后的属性值重新插入它们。`FEa` 定义第四行中并集运算符的第一个操作数代表 `OFFR` 表，其中未来在阿姆斯特丹开课的课程已被删除。该并集运算符的第二个操作数代表以翻倍最大容量重新插入这些课程。

同样，如表达式 `SEI` 所示，SQL 提供了一种方便的简写来指定更新事务。

我们继续看另一个更新事务示例。代码清单 10-5 中的事务 `ETX4` 更新某些员工的薪资。请注意，状态转换约束 `STC5`（见代码清单 8-8）要求薪资更新必须记录在 `HIST` 表结构中；更准确地说，需要记录*起始状态*的 `MSAL` 值。代码清单 10-5 将所有在位于丹佛的部门工作的培训师的薪资提高 10%。

**代码清单 10-5.** *事务 `ETX4(dbs)`*

**`FEa`**:
```math
dbs↓{SREP,MEMP,TERM,DEPT,GRD,CRS,OFFR,REG}
∪
{ (HIST; dbs(HIST)
∪
{ { (EMPNO; e(EMPNO)
)
,(UNTIL; sysdate )
,(DEPTNO;e(DEPTNO) )
,(MSAL;
e(MSAL)
) }
| e∈dbs(EMP)⊗dbs(DEPT) ∧ e(JOB) = 'TRAINER' ∧ e(LOC) = 'DENVER' }
) }
∪
{ (EMP; { e | e∈dbs(EMP) ∧ ¬ (e(JOB) = 'TRAINER' ∧
↵{ d(LOC) | d∈dbs(DEPT) ∧
d(DEPTNO) = e(DEPTNO) }
= 'DENVER') }
∪
{ e↓{EMPNO,ENAME,BORN,JOB,HIRED,SGRADE,USERNAME,DEPTNO}
∪ { (MSAL; 1.1 * e(MSAL)) }
| e∈dbs(EMP) ∧ e(JOB) = 'TRAINER' ∧
↵{ d(LOC) | d∈dbs(DEPT) ∧ d(DEPTNO) = e(DEPTNO) }
= 'DENVER' }
) }
```

**`SEI`**:
```sql
insert into HIST(EMPNO,UNTIL,DEPTNO,MSAL)
(select e.EMPNO, sysdate, e.DEPTNO, e.MSAL
from EMP e
where e.job ='TRAINER'
and 'DENVER' = (select d.LOC
from DEPT d
where d.DEPTNO = e.DEPTNO));

update EMP e
set e.MSAL = 1.1 * e.MSAL
where e.job ='TRAINER'
and 'DENVER' = (select d.LOC
from DEPT d
where d.DEPTNO = e.DEPTNO)
```

**`SEII`**:
```sql
update EMP e
set e.MSAL = 1.1 * e.MSAL
where e.job ='TRAINER'
and 'DENVER' = (select d.LOC
from DEPT d
where d.DEPTNO = e.DEPTNO);

insert into HIST(EMPNO,UNTIL,DEPTNO,MSAL)
(select e.EMPNO, sysdate, e.DEPTNO, e.MSAL / 1.1
from EMP e
where e.job ='TRAINER'
and 'DENVER' = (select d.LOC
from DEPT d
where d.DEPTNO = e.DEPTNO))
```

事务 `ETX4` 的规范 `FEa` 同时为 `HIST` 和 `EMP` 表结构分配新值。它根据 `EMP` 和 `DEPT` 之间的连接查询结果向 `HIST` 插入数据，从而按照约束 `STC5` 的要求，记录（大致来说）即将被更新的元组的当前 `MSAL` 和 `DEPTNO` 值。它还按要求更新了 `EMP`。这些表（“新” `HIST` 表和“新” `EMP` 表）的规范都引用了 `dbs`，即事务的起始状态。



在 SQL 中，你无法使用一条语句向一个表添加行，同时更改另一个表的行。你必须（分别）指定一条 `INSERT` 语句和一条 `UPDATE` 语句，并选择一个顺序来串行执行这两条语句。我们在 `SEI` 和 `SEII` 中通过分号分隔这两条 DML 语句来表示这种串行执行。

表达式 `SEI` 和 `SEII` 的区别在于两条语句的执行顺序。SQL DBMS 会提供一个中间数据库状态，该状态已经反映了第一条 DML 语句所做的修改，并将其提供给第二条 DML 语句。这条语句（以及嵌入其中的子查询）将“看到”这个中间数据库状态。

**注意** 这通常是 SQL DBMS 的默认行为。我们将在第 11 章中对这些中间数据库状态有更多讨论。

这种副作用在我们的形式化表示中不会出现；`FEa` 仅引用起始状态。正是这种副作用，导致在 `SEII` 中，用于检索需要记录（插入）到 `HIST` 表结构中的元组的子查询需要除以 `1.1`；它看到了先执行的 `UPDATE` 语句所产生的修改（增加的薪资）。

让我们再看一个更新事务的例子。清单 10-6 指定了取消预定课程开设的事务。注意，静态约束 `PODC6`（“被取消的课程开设不能有注册记录；”参见清单 7-43）要求该课程开设的所有注册记录（如果有）必须被删除。清单 10-6 取消了课程 `J2EE` 于 2007 年 2 月 14 日开始的课程开设。

**清单 10-6.** 事务 `ETX5(dbs)`
```
FEa: dbs↓{SREP,MEMP,TERM,DEPT,GRD,CRS,EMP,HIST}

∪

{ (REG; { r | r∈dbs(REG) ∧ (r(COURSE) ≠ 'J2EE' ∨ r(STARTS) ≠ '14-feb-2007') }

) }

∪

{ (OFFR; { o | o∈dbs(OFFR) ∧ ¬ (o(COURSE) = 'J2EE' ∧

o(STARTS) = '14-feb-2007') }

∪

{ o↓{COURSE,STARTS,MAXCAP,TRAINER,LOC}

∪ { (STATUS; 'CANC') }

| o∈dbs(OFFR) ∧ o(COURSE) = 'J2EE' ∧ o(STARTS) = '14-feb-2007' }

) }
```

**SEI**:
```
delete from REG r

where r.COURSE = 'J2EE'

and r.STARTS = '14-feb-2007';

update OFFR o

set o.STATUS = 'CANC'

where o.COURSE = 'J2EE'

and o.STARTS = '14-feb-2007'
```

**SEII**:
```
update OFFR o

set o.STATUS = 'CANC'

where o.COURSE = 'J2EE'

and o.STARTS = '14-feb-2007';

delete from REG r

where r.COURSE = 'J2EE'

and r.STARTS = '14-feb-2007'
```

形式化表达式 `FEa` 到现在应该很直观了；它指定了删除零条或多条注册记录，以及更新单个课程开设。SQL 表达式 `SEI` 和 `SEII` 同样仅在于在 SQL DBMS 中实现此事务所需的两条 DML 语句的执行顺序不同。

与 `ETX4` 相比，其中一种替代执行顺序有一个显著的缺点。`SEII` 创建的中间数据库状态可能违反静态约束。然而，第二条 DML 语句总是会纠正这个违规。

假设 `J2EE` 课程在情人节当天的课程开设已经至少有一个注册记录。那么，第一条 DML 语句——将课程开设状态设置为已取消——将创建一个违反约束 `PODC6` 的数据库状态。第二条 DML 语句——删除相应的注册记录——将修复此问题；它将中间数据库状态修改为符合 `PODC6` 的状态（它“修复了违规”）。

`SEII` 选项中的这种情况是非常危险的。如果其他在同一事务中执行的应用程序代码查询了这个（违反约束的）中间数据库状态，那么这些查询的结果可能是错误的。或者，更糟糕的是，第二条 DML 语句内部的子查询可能产生错误的结果。

**注意** 我们这里假设，在其他事务中执行的应用程序代码永远看不到这些中间数据库状态。在 Oracle 的 SQL DBMS 中确实如此（我们将在第 11 章讨论 Oracle 提供的事务隔离模式时回到这一点）。

例如，假设你想查询至少有一个注册记录的已计划或已确认的课程开设数量。我们称这个查询为 `EXQ11`。以下是 `EXQ11` 的形式化规范：
```
EXQ11(dbs) = #{ o | o∈dbs(OFFR) ∧ o(STATUS)∈{'SCHD','CONF'} ∧

(∃r∈dbs(REG): r↓{COURSE,STARTS} = o↓{COURSE,STARTS}) }
```

在 SQL 中，这个查询可能看起来像这样：
```
select count(*)

from OFFR o

where o.STATUS in ('SCHD','CONF')

and exists (select r.*

from REG r

where r.COURSE = o.COURSE

and r.STARTS = o.STARTS)
```

应用程序开发人员可能非常聪明，并对查询进行了语义优化，将其改写如下：
```
EXQ11(dbs) = #{ r↓{COURSE,STARTS} | r∈dbs(REG) }
```

这对应于以下 SQL 查询：
```
select count(*)

from (select distinct r.COURSE, r.STARTS

from REG r)
```

这里开发人员利用了约束 `PODC6`，该约束意味着如果某个课程开设有注册记录，那么该课程开设的状态不能是已取消。因此，通过统计 `REG` 中不同的（课程开设）外键值，你实际上是在统计至少有一个注册记录且状态为已计划或已确认的课程开设数量。

如果这个经过语义重写的查询在 `SEII` 版本的事务 `ETX5` 执行过程中运行，那么结果显然会多出一个；它也统计了那个其注册记录即将被删除——但尚未被删除——的已取消课程开设。

我们将在第 11 章中对此现象有更多讨论。

让我们再给出两个事务的例子。清单 10-7 定义了一个涉及子查询来确定新属性值的更新事务。清单 10-7 更新销售代表的佣金。对于每个销售代表，将其佣金增加其所在部门所有销售代表（包括正在更新的这位）平均月薪的 2%。

**清单 10-7.** 事务 `ETX6(dbs)`
```
FEa: dbs↓{MEMP,TERM,DEPT,GRD,REG,OFFR,CRS,EMP,HIST}

∪

{ (SREP; { s↓{EMPNO,TARGET}

∪ { (COMM; 1.02 *

(AVG e1∈{ e | e∈dbs(EMP) ∧

e(DEPTNO)∈{ e2(DEPTNO) | e2∈dbs(EMP) ∧

e2(EMPNO) = s(EMPNO) } ∧

e(JOB) = 'SALESREP' }

: e1(MSAL))

) }

| s∈dbs(SREP) }

) }
```

**SEI**:
```
update SREP s

set COMM = (select 1.02 * avg(e1.MSAL)

from EMP e1

where e1.DEPTNO = (select e2.DEPTNO

from EMP e2

where e2.EMPNO = s.EMPNO)

and e1.JOB = 'SALESREP')
```

请注意，与之前的更新事务示例相比，这是一个无限制的更新；`SREP` 的所有行都被修改。

为了方便起见，这里给出了我们在第 5 章中形式化定义的 `AVG` 操作符的结构：
```
(AVG x∈S: f(x))
```

对于可以从集合 `S` 中选择的每个元素 `x`，平均值操作符计算表达式 `f(x)` 的值，并计算所有这些值的平均值。在表达式 `FEa` 中，该操作符的使用方式如下：
```
(AVG e1∈{ 与销售代表 s(EMPNO) 在同一部门工作的所有销售代表 }

: e1(MSAL))
```

如你在表达式 `SEI` 中所见，SQL 允许你在 `UPDATE` 语句的 `SET` 子句中指定子查询。但是请注意，在这些情况下，子查询应始终仅检索单行单列的数据。

清单 10-8 定义了一个 `DELETE` 事务；从 `GRD` 中删除当前未被“使用”的薪资等级。该清单删除了没有员工被分配到的薪资等级。

**清单 10-8.** 事务 `ETX7(dbs)`

**FEa**:
```
dbs↓{SREP,MEMP,TERM,DEPT,REG,OFFR,CRS,EMP,HIST}

∪

{ (GRD; { g | g∈dbs(GRD) ∧ ¬(∃e∈dbs(EMP): e(SGRADE) = g(GRADE) } ) }
```

**SEI**:
```
delete from GRD g

where not exists (select e.*

from EMP e

where e.SGRADE = g.GRADE)
```



尽管存在“不存在”的限制，但考虑到表全域 `tab_GRD` 定义中指定的最后一个表约束（见清单 7-31），事务 `ETX7` 仍然很可能失败。事实上，只有当最低薪资等级（及其零个或多个直接后继）或最高薪资等级（及其零个或多个直接前驱）是唯一被删除的等级时，它才能成功执行。在所有其他情况下，该表约束将被违反，从而导致事务回滚。

本章末尾有一个练习，要求你指定一个修复此类违规的 `UPDATE` 语句。

#### 章节总结

本节以项目符号列表的形式总结了本章内容。在继续下一节的练习之前，你可以用它来检查自己对本章引入的各种概念的理解。

-   你可以将事务形式化地指定为数据库全域上的一个函数。对于每个数据库状态，这个函数返回另一个数据库状态，该状态反映了事务预期的修改。
-   你可以结合第 1 部分介绍的集合论与逻辑，以及第 2 部分介绍的各种表运算符，来指定这个函数。
-   事务起始于一个有效的数据库状态，并且应该始终结束于一个有效的数据库状态；它们必须使数据库处于符合所有约束的状态。
-   每个事务实际上都是一个事务 *尝试*。当结果的数据库状态不符合所有静态约束，或者状态转换未被状态转换全域涵盖时，DBMS 应确保事务被回滚。
-   在 SQL 中，事务被实现为一个或多个数据操作语言（DML）语句，这些语句按你决定的顺序依次执行。
-   你不能随意选择这些 DML 语句的顺序。后续的 DML 语句查询的是已修改的（中间）数据库状态；先前在事务中执行的 DML 语句所做的修改会由 SQL DBMS 对它们可见。
-   有时，中间数据库状态会违反一个或多个数据完整性约束。这种情况是不希望出现的，因为针对此类数据库状态执行的查询可能会给出错误的结果。
-   语义优化的（子）查询在此类情况下产生错误结果的风险尤其高。

##### 练习

**1.** 列出事务 `ETX1` 涉及的所有数据完整性约束。并讨论为使事务 `ETX1` 成功执行，起始状态应具备的属性。

**2.** 事务 `ETX2` 可能违反约束 `PODC4`；如果删除学生 3124 的一门注册导致某个 *已确认* 开课项的注册人数低于六人，则违反了此约束。请修改 `ETX2` 的规范（形式化和 SQL），使得在尽可能多地删除学生 3124 的注册的同时，不违反约束 `PODC4`。

**3.** 确定事务 `ETX4` 的 SQL `UPDATE` 语句可能违反的静态数据完整性约束。

**4.** 假设事务 `ETX7` 创建了一个确实违反所讨论表约束的数据库状态。选择一种策略，通过更新剩余的薪资等级来修复此问题。形式化地指定你的策略，并为其提供一个 SQL `UPDATE` 语句。

**5.** 形式化地并用 SQL 指定以下事务：将部门 40 中所有管理员的月薪提高 5%。如果更新需要修改薪资等级，则也修改薪资等级。

**6.** 形式化地并用 SQL 指定以下事务：删除所有五年前已离职员工的数据。此事务会遇到哪些与约束相关的问题？

