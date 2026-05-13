# 290

## 第 11 章 - 在 ORACLE 中实现数据库设计

我们将以一些关于可序列化的总结性评论来结束本节，进一步阐明为何供应商对声明式策略来实现约束的支持如此欠缺。

回想一下你已经了解的多种执行模型。效率较低的执行模型不仅会比必要更频繁地执行非优化的约束验证查询，它们也会比必要更频繁地获取序列化锁。例如，如果你在使用`EM1`构建的`DI`代码中加入序列化锁调用，实际上你将只允许一次执行一个事务；在这个执行模型中，不可能有任何事务的并发执行。在`EM2`中，这被放宽为每个表结构一次只允许一个事务。进一步遍历所有剩余的执行模型，并发级别每次都在增加。

除了之前关于`DBMS`可能无法计算`TE`查询和优化的约束验证查询的评论外，类似的评论也适用于`DBMS`能够计算出最优的序列化锁调用应该是什么。确保`DI`代码执行正确性的一个惰性策略是为每个约束获取一个独占的应用程序锁（如代码清单 11-38 所示）。然而，你可能不会接受随之而来的`DBMS`并发性不必要下降的问题。

本节广泛研究了使用触发式过程策略为表约束实现高效`DI`代码的相关问题后，下一节将只为你提供一个为表约束实现`DI`代码的指导原则摘要。

#### 实现表约束

正如前一节所展示的，正是在表约束这个级别，事情迅速变得更加复杂。你只能以声明方式规定两种类型的表约束：唯一标识属性（键），以及引用回同一表的子集要求（此时子集要求是一个表约束）。

以下是一个示例，展示了 SQL 主键约束和唯一键约束的使用（见`代码清单 11-39`）。



#### 第 11 章

#### 实现数据库约束

**清单 11-39. DEPT 表结构键的声明性实现**

```sql
alter table DEPT add constraint DEPT_key1 primary key(deptno);
after table DEPT add constraint DEPT_key2 unique(dname,loc);
```

该示例数据库设计不涉及表级别的子集要求。下一节你将看到如何声明性地表述数据库级别的子集要求。

鉴于“表约束实现问题”这一节，你或许现在能意识到为什么数据库管理系统供应商不提供在声明性约束中包含查询的可能性；很可能因为数据库管理系统无法提供比 EM4 更好的执行模型，并且你也可能不会接受 EM4 所带来的强加事务串行化。

例如，以下试图声明性实现 `DEPT_TAB01` 的 SQL 语句是不被允许的：

`7451CH11.qxd 5/15/07 2:26 PM Page 291`

```sql
-- Oracle 的无效 "alter table add constraint" 语法。
alter table DEPT add constraint dept_tab01 check
(not exists(select m.DEPTNO
            from DEPT m
            where 2 < (select count(*)
                       from DEPT d
                       where d.MGR = m.MGR)))
```

你必须为所有其他类型的表约束自己编写过程性的 DI 代码。

开发和实现高效的 DI 代码并非易事。正如研究各种执行模型时所展示的，每个约束都需要以下高级步骤：

1.  将形式化规范转化为约束验证查询。
2.  编写维护转换效应的代码。
3.  设计 TE 查询，确保仅在必要时运行约束验证查询。
4.  寻找一种方法，通过 TE 查询提供可用于验证查询的值来优化约束验证查询。
5.  为 DI 代码设计和添加串行化策略。

然而，你会发现，通过定期实现表约束并熟练掌握该技能，以过程化方式实现表约束通常是完全可行的。

如你所知，实现数据库约束的声明性支持仅适用于子集要求；在 SQL 中，子集要求通过声明 *外键* 来表达。让我们看看如何实现来自示例数据库设计的子集要求 PSSR1：

PSSR1(EMP,DEPT) :=

/* 员工为一个已知的部门工作 */

{ e(DEPTNO) | e∈EMP } ⊆ { d(DEPTNO) | d∈DEPT }

在 SQL 中，此子集要求表述如下：

```sql
alter table emp add constraint emp_fk_dept
foreign key(deptno) references dept(deptno);
```

SQL 要求被引用的属性集构成一个键。在上述例子中，确实如此；`DEPTNO` 在 `DEPT` 中是唯一标识。然而，在以下情况——要求 PSSR7——`LOC` 在 `DEPT` 中不构成键。因此，你不能通过声明外键来实现 PSSR7。

`7451CH11.qxd 5/15/07 2:26 PM Page 292`

PSSR7(OFFR,DEPT) :=

/* 课程在我们有部门的地点进行 */

{ o(LOC) | o∈OFFR } ⊆ { d(LOC) | d∈DEPT }

如果你尝试以下 `alter table` 命令，SQL 数据库管理系统将报错：

```sql
-- 因为 dept(loc) 不是键，所以无效。
alter table offr add constraint offr_fk_dept
foreign key(loc) references dept(loc);
```

还有其他一些情况，无法使用外键来声明性地实现子集要求。示例数据库设计中的一些子集要求引用被引用表结构中的 *子集* 元组；这很常见。这里有一个例子：

PSSR2(DEPT,EMP) :=

/* 部门经理是已知员工，不包括管理员和总裁 */

{ d(MGR) | d∈DEPT } ⊆

{ e(EMPNO) | e∈EMP ∧ e(JOB) ∉ {'ADMIN','PRESIDENT'} }



您可以按如下方式声明从 `DEPT` 到 `EMP` 的外键：`alter table offr add constraint offr_fk_dept foreign key(loc) references dept(loc);`

这个外键将确保只引用已知的员工；但它仍然允许管理员或总裁管理部门。目前无法通过声明方式指定只允许引用员工的一个子集；为此，您需要编写额外的过程式代码。

类似的限制也适用于只需要引用另一表中行键值的行子集的情况。这在子集要求 `PSSR8` 中有所体现：`PSSR8(OFFR,EMP) :=`

`/* 课程授课的讲师是已知的讲师 */`

```
{ o(TRAINER) | o∈OFFR ∧ o(TRAINER) ≠ -1 } ⊆
{ e(EMPNO) | e∈EMP ∧ e(JOB) = 'TRAINER' }
```

无法向 DBMS 声明，只有 `TRAINER≠-1` 的 `OFFR` 行子集中的每一行都引用一个已知的讲师。

不过，这种情况有一种技巧可以应用。通过选择使用 `NULL` 而不是值 `-1` 来表示尚未分配讲师，您至少可以声明一个外键，以强制讲师分配必须引用一个已知的员工。同样，您仍然需要开发额外的过程式代码来在另一端实现限制（只允许引用 `JOB='TRAINER'` 的员工）。

现在让我们花点时间研究一下；在构建额外的过程式代码之前，明智的做法是首先思考一下这里需要实现的剩余谓词是什么（给定外键已经为实现 `PSSR8` 做了一些工作）。更准确地说，您能将 `PSSR8` 重写为一个合取式，其中一个合取项恰好涵盖外键已经通过声明方式实现的内容，而另一个合取项涵盖需要通过过程式代码实现的内容吗？如下所示。

7451CH11.qxd 5/15/07 2:26 PM Page 293

## 第 11 章 ■ 在 ORACLE 中实现数据库设计

293

```
(∀o∈OFFR: o(TRAINER)≠-1 ⇒ (∃e∈EMP: e(empno)=o(trainer))) ∧
(∀t∈(OFFR◊◊{(trainer;empno)})⊗EMP: t(job)='TRAINER')
```

前面的第一个合取项表明，对于每个课程授课，分配的讲师应该是一个已知的员工；这代表了外键所实现的内容。第二个合取项表明，对于 `OFFR`（需要属性重命名）与 `EMP` 的连接中的所有元组，`JOB` 属性应持有值 `'TRAINER'`；这代表了需要通过过程式代码实现的内容。顺便问一下，您认出第二个合取项的谓词模式了吗？它实际上是一个元组连接谓词。

### ■ 注意

您可能凭直觉就能理解，第二个谓词正是为 `PSSR8` 需要通过过程式代码实现的部分。值得一提的是，您实际上可以形式化地证明前面的合取式在逻辑上等同于 `PSSR8` 谓词。这需要开发更多关于量词的重写规则，以及关于如何建立形式化证明的一些概念（这两方面在本书中我们都未涉及）。

过程式地实现数据库约束的方式与表约束类似；只是现在因为涉及多个表结构，需要开发更多的触发器。请看一下 `清单 11-40`。它列出了使用 `EM6`（包括序列化逻辑）实现前述剩余的元组连接谓词以完成 `PSSR8` 实现所需的所有触发器。在这种情况下，您需要一个在 `EMP` 表结构上的 after statement update 触发器，以及一个在 `OFFR` 表结构上的 after `INSERT` statement 触发器和 after statement update 触发器。

`清单 11-40` 没有列出维护转换效果视图所必需的全局临时表、pre-statement 触发器、after row 触发器和视图定义；您需要以类似 `清单 11-33` 所示的方式，为 `OFFR` 和 `EMP` 两个表结构设置这些。



**清单 11-40.** 剩余 PSSR8 谓词 DI 代码的 EM6 实现

```
create trigger EMP_AUS_PSSR8
after update on EMP
declare
  pl_dummy varchar(40);
begin
  -- Changing a trainers job, requires validation of PSSR8.
  for r in (select n_empno as empno
            from v_emp_ute e
            where (e.0_empno=e.n_empno and e.n_job<>'TRAINER' and e.o_job='TRAINER') or (e.0_empno<>e.n_empno and e.n_job<>'TRAINER')) loop
    begin
      -- Acquire serialization lock.
      p_request_lock('PSSR8'||to_char(r.empno));
      --
      select 'Constraint PSSR8 is satisfied' into pl_dummy
      from DUAL
      where not exists(select 'This employee is assigned as a trainer to an offering'
                       from OFFR o
                       where o.TRAINER = r.empno);
      --
    exception when no_data_found then
      --
      raise_application_error(-20999,'Constraint PSSR8 is violated '||
        'for employee '||to_char(r.empno)||'.');
      --
    end;
  end loop;
  --
end;
/

create trigger OFFR_AIS_PSSR8
after insert on OFFR
declare
  pl_dummy varchar(40);
begin
  -- Inserting an offering, requires validation of PSSR8.
  for r in (select distinct trainer
            from v_offr_ite i)
  loop
    begin
      -- Acquire serialization lock.
      p_request_lock('PSSR8'||to_char(r.trainer));
      --
      select 'Constraint PSSR8 is satisfied' into pl_dummy
      from DUAL
      where not exists(select 'This employee is not a trainer'
                       from EMP e
                       where e.EMPNO = r.trainer
                       and e.JOB <> 'TRAINER');
      --
    exception when no_data_found then
      --
      raise_application_error(-20999,'Constraint PSSR8 is violated '||
        'for trainer '||to_char(r.trainer)||'.');
      --
    end;
  end loop;
  --
end;
/

create trigger OFFR_AUS_PSSR8
after update on OFFR
declare
  pl_dummy varchar(40);
begin
  -- Updating the trainer of an offering, requires validation of PSSR8.
  for r in (select distinct n_trainer as trainer
            from v_offr_ute u
            where o_trainer<>n_trainer)
  loop
    begin
      -- Acquire serialization lock.
      p_request_lock('PSSR8'||to_char(r.trainer));
      --
      select 'Constraint PSSR8 is satisfied' into pl_dummy
      from DUAL
      where not exists(select 'This employee is not a trainer'
                       from EMP e
                       where e.EMPNO = r.trainer
                       and e.JOB <> 'TRAINER');
      --
    exception when no_data_found then
      --
      raise_application_error(-20999,'Constraint PSSR8 is violated '||
        'for trainer '||to_char(r.trainer)||'.');
      --
    end;
  end loop;
  --
end;
/
```

我们将以对数据库供应商在多元组约束方面提供的声明性支持不足的观察来结束本节。因为我们相信，对于数据库供应商而言，不可能编程实现一个算法，该算法能接受任意复杂的谓词，然后计算高效的 TE 查询、一个最小的验证查询以及实现执行模型 EM6 的最优序列化代码，所以我们不应期望未来这些供应商能以实用、可用且可接受的方式提供对多元组约束的完全支持。我们所能期望的最好是，数据库研究人员首先提出更常见的约束类别，并为这些类别开发便捷的简写形式。然后，数据库供应商应提供与我们这些简写形式一致的新声明性构造，以便我们能够轻松地向 DBMS 陈述这些常见的约束类别。有了这样一个常见类别的声明，数据库供应商就应该能够编程实现一个算法，在底层为我们提供一个类似 EM6 的执行模型来实现该约束。

`注意` 我们在本书中已经暗示了几种常见的约束类别，数据库供应商应为其提供完全的声明性支持：特化、泛化和连接中的元组约束。此外，在子集需求约束领域——我们目前只有 SQL 外键构造可用——应该提供更多声明性的变体形式。

至此，关于为数据库约束开发 DI 代码的探讨就结束了。下一节将探讨如何实现过渡约束。

### 实现过渡约束

请返回清单 8-8，花点时间回顾一下我们示例数据库宇宙中的过渡约束。你是否注意到，状态过渡约束的指定方式与指定数据库约束的方式并无不同？它们是涉及两个或多个表类型参数的谓词。当然，在过渡约束的情况下，这些参数总是代表所涉及表结构的旧快照和新快照。但从本质上讲，过渡约束是一个涉及多个表结构的谓词。这意味着，为过渡约束实现 DI 代码的复杂性，原则上与数据库约束的复杂性并无不同。

过渡约束处理的是事务的开始状态和结束状态。如前所述，事务在 SQL 中是通过连续执行 DML 语句来实现的。这种实现不仅会创建中间数据库状态，而且——在本节的上下文中——还会创建*中间状态转换*。从形式上讲，你只需要验证从事务开始状态到结束状态的转换。过渡约束的 DI 代码应该只在事务的最后一条 DML 语句创建了结束状态之后执行；也就是说，它的执行应该被推迟到事务结束时。在“引入延迟检查”一节中，你会找到一个关于延迟执行 DI 代码的简要探讨。现在我们将演示如何为过渡约束开发 DI 代码（在触发式过程策略中），使得所有中间状态转换也满足这些约束。

`注意` 正如你可能知道的，当前的 SQL DBMS 中完全没有支持以声明方式实现过渡约束。

如果你开始思考如何实现过渡约束，你立即会碰到这个问题：“在 after 语句触发器中，我如何查询所涉及表结构的旧快照？”答案其实很简单。在 Oracle 的 SQL DBMS 中，你可以使用*闪回查询*来做到这一点：查询表结构的某个旧快照。为了能够对表结构使用闪回查询，你必须以某种方式管理事务开始时的*系统变更号*。你可以使用包变量或会话临时表来做到这一点。请注意，只有那些旧快照参与了任何状态过渡约束的表结构，才需要使用闪回查询进行查询。因此，仅当 DML 语句更改了这些表结构之一时才管理系统变更号是安全的；你可以使用语前触发器来实现这一点，这些触发器通过 `dbms_flashback.get_system_change_number` 确定当前的系统变更号，并将其存储在包变量或会话表中。在下面的例子中，我们将假设这些触发器已经就位。

我们现在将为你展示实现状态过渡约束 `STC5` 的 DI 代码的探讨。

`STC5(HISTB,EMPB,HISTE,EMPE) :=`

`/* 新的历史记录必须准确反映员工的更新 */`

`( ∀h∈(HISTE⇓{EMPNO,UNTIL} − HISTB⇓{EMPNO,UNTIL})⊗HISTE:`

`h(UNTIL) = sysdate ∧`

`( ∃e1∈EMPB, e2∈EMPE:`

`e1↓{EMPNO,MSAL,DEPTNO} = h↓{EMPNO,MSAL,DEPTNO} ∧`

`e2(EMPNO) = h(EMPNO) ∧`

`( e2(MSAL) ≠ e1(MSAL) ∨ e2(DEPTNO) ≠ e1(DEPTNO) ) ) )`


为维持这一约束，您只需在 `HIST` 表结构上开发一个 `INSERT` 语句后的触发器。`代码清单 11-41` 列出了此触发器。请注意，形式化规范再次相对容易地被转化为约束验证查询。以下代码假设存在一个过渡状态视图，并且假设存在函数 `f_get_start_scn` 来检索先前存储的系统变更号。

## 代码清单 11-41：用于 STC5 的 DI 代码在 EM6 中的实现（无序列化）

```
create or replace trigger HIST_AIS_STC5

after insert on HIST

declare pl_dummy varchar(40);

begin

-- 插入历史记录需要验证 STC5。

for r in (select empno,until

from v_hist_ite i)

loop

begin

--

select 'Constraint STC5 is satisfied' into pl_dummy

from DUAL

where exists (select 'The history record is OK'

from HIST h

where h.EMPNO = r.empno

and h.UNTIL = r.until

and h.UNTIL = sysdate

and exists(select 'A corresponding update on EMP'

from EMP as of scn f_get_tx_start_scn e1

,EMP e2

where e1.EMPNO = h.EMPNO

and e1.MSAL = h.MSAL

and e1.DEPTNO = h.DEPTNO

and e2.EMPNO = h.EMPNO

and (e2.MSAL<>e1.MSAL or e2.DEPTNO<>e1.DEPTNO))); exception when no_data_found then

--

raise_application_error(-20999,'Constraint STC5 is violated '||

'for history record '||to_char(r.empno)||'/'||

to_char(r.until)||'.');

--

end;

end loop;

--

end;
```

为过渡约束的 `DI` 代码设计序列化策略并非易事。例如，请考虑表 11-5 中所示的两个并发执行事务 `TX5` 和 `TX6` 的场景。

## 表 11-5：TX5 和 TX6 的序列化

| 时间 | TX5 | TX6 | 说明 |
|------|-----|-----|---------|
| t=0 | `DML1`; | | `TX5` 开始；`DML1` 不涉及 `EMP` 表结构。 |
| t=1 | | `UPDATE` | `TX6` 开始；它更新员工 1042 的工资。 |
| t=2 | | `INSERT` | `TX6` 插入相应的记录；`STC5` 的 `DI` 代码触发。 |
| t=3 | | `COMMIT` | `TX6` 结束。 |
| t=4 | `INSERT` | | `TX5` 为 1042 插入类似的历史记录，`STC5` 的 `DI` 代码触发。 |

考虑到 Oracle 的 `读已提交` 隔离级别，当约束 `STC5` 的 `DI` 代码在 `t=4` 触发时，它能看到由事务 `TX6` 建立的员工 1042 工资变更。`t=0` 时的旧快照没有此变更。新的快照（即 `t=4` 时的当前快照）确实有此变更，因为 `TX6` 已在 `t=4` 提交。因此，`STC5` 的 `DI` 代码可能会批准由 `TX5` 执行的历史记录插入（当然，前提是它正确反映了工资更新）。在这种情况下，添加序列化锁调用无济于事；`TX6` 中运行的 `DI` 代码获取的任何锁在 `t=4` 时都已释放。

显然，当前 `STC5` 的 `DI` 代码设置无法正确工作，是因为它错误地推断它所看到的关于 `EMP` 表结构的变化是由*同一*事务中先前的 `DML` 语句引起的。

我们给出此示例是为了表明，为（至少某些）状态转换约束实现 `DI` 代码绝非易事，绝对需要进一步研究。

### ■注意

在撰写本书时，我们尚未完全研究正确序列化状态转换 `DI` 代码的相关问题。在这种情况下，一种可能的修复方法是让约束验证查询（闪回查询）在 Oracle 的 `可序列化` 隔离级别下运行。在此模式下运行的查询不会看到其他事务在当前事务开始后提交的更改；您只能看到当前事务所做的更改。不幸的是，您无法更改单个查询的隔离模式；您必须转而让整个事务在 Oracle 的 `可序列化` 模式下运行。

在下一节中，您将找到对另一个我们至今尚未忽视的问题的探讨。


