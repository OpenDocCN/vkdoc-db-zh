# 在 Oracle 中实现数据库设计

在 SQL DBMS 中创建表涉及以下步骤：

*   为表结构选择一个名称（我们已经在*数据库特征描述*中正式完成）
*   声明各列的名称（我们已经在*特征描述*中正式完成）
*   声明各列的数据类型（同样在*特征描述*中完成）

关于最后一点，遗憾的是，目前对创建可在 `create table` 语句中使用的用户定义类型（用于表示属性值集）的支持尚不成熟。例如，我们无法执行以下操作：

```sql
-- 创建一个类型来表示 EMPNO 属性的值集。
-- 注意：这不是有效的 Oracle 语法。
create type empno_type under number(4,0) check(member > 999);
/

-- 创建一个类型来表示 JOB 属性的枚举值集。
-- 无效语法。
create type job_type under varchar(9)
check(member in ('ADMIN','TRAINER','SALESREP','MANAGER','PRESIDENT));
/

-- 现在在 create table 语句中使用 empno_type 和 job_type 类型。
create table EMP
(empno empno_type not null
,job job_type not null
,...);
```

在实际操作中，我们被迫使用 SQL DBMS 提供的内置数据类型。对于 Oracle 而言，最常用的内置数据类型是 `varchar`、`number` 和 `date`。这些对于我们的示例数据库设计来说已经足够。

在下一节（“实现属性约束”）中，你将看到，通过使用 `SQL alter table add constraint` 语况，我们仍然可以有效地实现属性值集。在 `create table` 语句中声明的列数据类型充当了属性值集的超集，而通过添加约束，我们可以将这些超集精确地缩小到表结构特征描述中指定的属性值集。

清单 11-4 至 11-13 展示了 UEX 示例数据库设计的 `create table` 语句。

**清单 11-4.** *为 GRD 表结构创建表*

```sql
create table grd
( grade number(2,0) not null
, llimit number(7,2) not null
, ulimit number(7,2) not null
, bonus number(7,2) not null);
```

**注意**：每个列旁的 `not null` 表示该列不允许出现 `NULL`。在这种情况下，我们说该列是*必填*的。

**清单 11-5.** *为 EMP 表结构创建表*

```sql
create table emp
( empno number(4,0) not null
, ename varchar(8) not null
, job varchar(9) not null
, born date not null
, hired date not null
, sgrade number(2,0) not null
, msal number(7,2) not null
, username varchar(15) not null
, deptno number(2,0) not null);
```

**清单 11-6.** *为 SREP 表结构创建表*

```sql
create table srep
( empno number(4,0) not null
, target number(6,0) not null
, comm number(7,2) not null);
```

**清单 11-7.** *为 MEMP 表结构创建表*

```sql
create table memp
( empno number(4,0) not null
, mgr number(4,0) not null);
```

**清单 11-8.** *为 TERM 表结构创建表*

```sql
create table term
( empno number(4,0) not null
, left date not null
, comments varchar(60));
```


#### 实现属性约束

**注意** 在我们的示例中，当员工已离职时，用户并不总是希望为 `COMMENTS` 列提供值。Oracle 不允许在强制列中存储空字符串。为了防止用户在此类情况下输入一些随机的 varchar 值（例如一个空格），该列已被设为可选（有时称为可为空），这意味着该列允许出现 `NULL`。

**代码清单 11-9. 用于 HIST 表结构的 CREATE TABLE 语句**
```sql
create table hist
( empno number(4,0) not null
, until date not null
, deptno number(2,0) not null
, msal number(7,2) not null);
```

**代码清单 11-10. 用于 DEPT 表结构的 CREATE TABLE 语句**
```sql
create table dept
( deptno number(2,0) not null
, dname varchar(12) not null
, loc varchar(14) not null
, mgr number(4,0) not null);
```

**代码清单 11-11. 用于 CRS 表结构的 CREATE TABLE 语句**
```sql
create table crs
( code varchar(6) not null
, descr varchar(40) not null
, cat varchar(3) not null
, dur number(2,0) not null);
```

**代码清单 11-12. 用于 OFFR 表结构的 CREATE TABLE 语句**
```sql
create table offr
( course varchar(6) not null
, starts date not null
, status varchar(4) not null
, maxcap number(2,0) not null
, trainer number(4,0)
, loc varchar(14) not null);
```

**注意** 出于我们将在“实现数据库约束”一节中讨论的原因，`TRAINER` 属性被定义为可为空；我们将使用 `NULL` 来代替该属性的属性值集中引入的特殊值 `-1`。

**代码清单 11-13. 用于 REG 表结构的 CREATE TABLE 语句**
```sql
create table reg
( stud number(4,0) not null
, course varchar(6) not null
, starts date not null
, eval number(1,0) not null);
```

在关系数据模型中，数据库设计中的所有属性都是强制性的。因此，你可能会感到失望，在 SQL 标准中，默认情况下列是可为空的；SQL 要求我们必须在每一列旁边显式添加 `not null` 才能使其变为强制性的。

下一节将讨论在 `create table` 语句中使用的内置数据类型的前述“窄化”问题。

#### 实现属性约束

我们现在重新审视“约束分类模式”一章第 7 节中提到的术语“属性约束”。

形式上，特征描述只是将属性值集附加到属性上。将属性值集附加到属性可以被视为一个属性约束。然而，在实践中，你是在 SQL DBMS 中实现数据库设计，而这些 DBMS 以对用户定义类型支持不佳而闻名。用户定义类型本应是实现属性值集的理想选择。但是，正如“实现表结构”一节所讨论的，你不能用它们来实现这一点。相反，你必须使用适当的超集（如前一节所示的某种内置数据类型）作为给定属性的属性值集。幸运的是，你可以使用声明式 SQL `check` 约束将这些超集窄化为特征描述中指定的确切属性值集。在实现过程中，我们将这些声明式 `check` 约束称为属性的属性约束。

所有属性约束都可以——并且根据我们的策略偏好，*应该*——被声明为声明式 `check` 约束。你可以使用 `alter table add constraint` 语句来声明这些约束。

代码清单 11-14 展示了六个 `check` 约束的声明，这些约束是声明式实现代码清单 7-2 中 `chr_EMP` 定义的 `EMP` 表结构属性值集所必需的。我们将在代码清单之后讨论每一个约束。

**代码清单 11-14. EMP 表结构的属性约束**
```sql
alter table EMP add constraint emp_chk_empno check (empno > 999);
alter table EMP add constraint emp_chk_job
    check (job in ('PRESIDENT','MANAGER','SALESREP'
                  ,'TRAINER','ADMIN' ));
alter table EMP add constraint emp_chk_brn check (trunc(born) = born);
alter table EMP add constraint emp_chk_hrd check (trunc(hired) = hired);
alter table EMP add constraint emp_chk_msal check (msal > 0);
alter table EMP add constraint emp_chk_usrnm check (upper(username) = username);
```

从这个代码清单可以看出，所有的 `check` 约束都被赋予了名称。第一个约束的名称是 `emp_chk_empno`。它将 `empno` 列声明的数据类型 `number(4,0)` 窄化为仅包含大于 `999` 的四位数字。

一旦该约束被声明并存储在 Oracle SQL DBMS 的数据字典中，每当 `EMP` 表中出现新的 `EMPNO` 值（通过 `INSERT` 语句），或 `EMP` 表中现有的 `EMPNO` 值被更改（通过 `UPDATE` 语句）时，DBMS 就会运行必要的 DI 代码。DBMS 将在错误消息中使用该约束名，每当尝试存储在 `EMP` 表中的 `EMPNO` 值不满足此约束时，你都会收到通知。

约束 `emp_chk_job`（前面代码清单中的第二个）确保只有列出的五个值可以作为 `JOB` 列的值。

约束 `emp_chk_brn` 和 `emp_chk_hrd` 确保一个日期值（在 Oracle SQL DBMS 中，日期值总是包含一个时间成分）只有在时间成分被截断（即设置为午夜 `0:00`）时，才被允许作为 `BORN` 或 `HIRED` 列的值。

约束 `emp_chk_msal` 确保只有正数——在 `number(7,2)` 超集范围内——才被允许作为 `MSAL` 列的值。

最后，约束 `emp_chk_usrnm` 确保 `USERNAME` 列的值始终为大写。

代码清单 11-15 到 11-23 提供了 UEX 数据库设计中其他表结构的属性约束。

**代码清单 11-15. GRD 表结构的属性约束**
```sql
alter table GRD add constraint grd_chk_grad check (grade > 0);
alter table GRD add constraint grd_chk_llim check (llimit > 0);
alter table GRD add constraint grd_chk_ulim check (ulimit > 0);
alter table GRD add constraint grd_chk_bon1 check (bonus > 0);
```

**代码清单 11-16. SREP 表结构的属性约束**
```sql
alter table SREP add constraint srp_chk_empno check (empno > 999);
alter table SREP add constraint srp_chk_targ check (target > 9999);
alter table SREP add constraint srp_chk_comm check (comm > 0);
```

**代码清单 11-17. MEMP 表结构的属性约束**
```sql
alter table MEMP add constraint mmp_chk_empno check (empno > 999);
alter table MEMP add constraint mmp_chk_mgr check (mgr > 999);
```

**代码清单 11-18. TERM 表结构的属性约束**
```sql
alter table TERM add constraint trm_chk_empno check (empno > 999);
alter table TERM add constraint trm_chk_lft check (trunc(left) = left);
```

**代码清单 11-19. HIST 表结构的属性约束**
```sql
alter table HIST add constraint hst_chk_eno check (empno > 999);
alter table HIST add constraint hst_chk_unt check (trunc(until) = until);
alter table HIST add constraint hst_chk_dno check (deptno > 0);
alter table HIST add constraint hst_chk_msal check (msal > 0);
```


### 第 11 章 在 Oracle 中实现数据库设计

### 清单 11-20. `DEPT` 表结构的属性约束

```sql
alter table DEPT add constraint dep_chk_dno check (deptno > 0);
alter table DEPT add constraint dep_chk_dnm check (upper(dname) = dname);
alter table DEPT add constraint dep_chk_loc check (upper(loc) = loc);
alter table DEPT add constraint dep_chk_mgr check (mgr > 999);
```

### 清单 11-21. `CRS` 表结构的属性约束

```sql
alter table CRS add constraint reg_chk_code check (code = upper(code));
alter table CRS add constraint reg_chk_cat check (cat in ('GEN','BLD','DSG'));
alter table CRS add constraint reg_chk_dur1 check (dur between 1 and 15);
```

### 清单 11-22. `OFFR` 表结构的属性约束

```sql
alter table OFFR add constraint ofr_chk_crse check (course = upper(course));
alter table OFFR add constraint ofr_chk_strs check (trunc(starts) = starts);
alter table OFFR add constraint ofr_chk_stat check (status in ('SCHD','CONF','CANC'));
alter table OFFR add constraint ofr_chk_trnr check (trainer > 999);
alter table OFFR add constraint ofr_chk_mxcp check (maxcap between 6 and 99);
```

**注意** 你可能想知道，每当在 `TRAINER` 列中遇到 `NULL` 时，SQL DBMS 如何处理约束 `ofr_chk_trnr`。我们将在本节末尾讨论这一点。

### 清单 11-23. `REG` 表结构的属性约束

```sql
alter table REG add constraint reg_chk_stud check (stud > 999);
alter table REG add constraint reg_chk_crse check (course = upper(course));
alter table REG add constraint reg_chk_strs check (trunc(starts) = starts);
alter table REG add constraint reg_chk_eval check (eval between -1 and 5);
```

如果声明式的 `check` 约束求值为 `UNKNOWN`（通常由使用 `NULL` 引起），那么 SQL 标准认为该约束已满足；`check` 求值为 `TRUE`。注意；在 PL/SQL 编程语言中，你会观察到相反的行为。在这里，一个求值为未知的布尔表达式会被视为 `FALSE`。为了说明这一点，请看以下触发器定义；它*并不*等同于 `check` 约束 `ofr_chk_trnr`：

```sql
create trigger ofr_chk_trnr
after insert or update on OFFR
for each row
begin
if not (:new.trainer > 999)
then
raise_application_error(-20999,'Value for trainer must be greater than 999.');
end if;
end;
```

声明式的 `check` 约束将允许 `TRAINER` 列中出现 `NULL`，而前面的触发器则不允许 `TRAINER` 列中出现 `NULL`。你可以通过将前面触发器定义中的第五行更改为以下内容来修复此差异：

```sql
if not (:new.trainer > 999 or :new.trainer IS NULL)
```

现在该触发器就等同于声明式的 `check` 约束了。

我们继续研究如何在 Oracle 的 SQL DBMS 中实现元组约束（属性约束之后的下一级）。

#### 实现元组约束

在处理元组约束的实现之前，我们需要坦白一些事情。本书中开发的形式化方法基于二值逻辑 (`2VL`)。`2VL` 的科学性是可靠的；我们已在第 1 章和第 3 章探讨了命题和谓词，并基于此开发了一些重写规则。然而，在本章中，我们将对以 SQL 表达的谓词做出各种陈述。如前面的触发器和清单 11-22 中的属性约束 `ofr_chk_trnr` 所示，由于可能存在 `NULL`，SQL 并不应用 `2VL`；而是应用三值逻辑 (`3VL`)。`3VL` 中最关键的假设是，除了 `TRUE` 和 `FALSE` 这两个真值之外，第三个值表示“可能”或 `UNKNOWN`。

**注意** 与经典的 `2VL` 相反，`3VL` 是反直觉的。我们在此不深入讨论 `3VL`；你可以在附录 D 中找到对 `3VL` 的简要探讨。

我们事先承认，在本章中我们采取了一些自由处理的方式。通过在示例数据库设计的 SQL 实现中几乎在所有列上使用 `NOT NULL`，我们实际上是在避免 `3VL` 问题。如果不使用 `NOT NULL`，本章中我们关于逻辑表达式的各种陈述都将受到质疑。

正如你在第 1 章看到的，合取、析取和否定是*真值函数完备的*。因此，你可以将每个正式指定的元组约束重写为等效的规范，该规范仅使用 SQL 中可用的三种连接词。

一旦以这种方式转换，所有元组约束都可以——因此也应该——声明式地表述为 `check` 约束。你可以使用 `alter table add constraint` 语句向 DBMS 声明它们。让我们以 `EMP` 表结构的元组约束为例来演示这一点。为方便起见，我们在此重复元组 universe 定义 `tup_EMP`：

```
tup_EMP :=
{ e | e∈Π(chr_EMP) ∧
/* We hire adult employees only */
e(BORN) + 18 ≤ e(HIRED) ∧
/* Presidents earn more than 120K */
e(JOB) = 'PRESIDENT' ⇒ 12*e(MSAL) > 120000 ∧
/* Administrators earn less than 5K */
e(JOB) = 'ADMIN' ⇒ e(MSAL) < 5000 }
```

前面的三个元组约束可以表述如下（参见清单 11-24）。

**注意** 前面的三个约束是用 `2VL` 正式表达的，但清单 11-24 中用 SQL 表达的三个约束是 `3VL` 的。在这种情况下，`3VL` 表达的约束之所以等同于正式表达的约束，仅仅是因为我们小心地将所有涉及的列都声明为必填 (`NOT NULL`)。

### 清单 11-24. `EMP` 表结构的元组约束

```sql
alter table EMP add constraint emp_chk_adlt
check ((born + interval '18' year) <= hired);
alter table EMP add constraint emp_chk_dsal
check ((job <> 'PRESIDENT') or (msal > 10000));
alter table EMP add constraint emp_chk_asal
check ((job <> 'ADMIN') or (msal < 5000));
```

第一个名为 `emp_chk_adlt` 的约束的实现使用了日期算术（`+ interval` 运算符）在给定的 `born` 日期值上加上 18 年。

因为 SQL 只提供了三种逻辑连接词 (`and`, `or`, `not`)，所以你被迫将第二和第三个元组约束——两者都涉及蕴涵连接词——转换为析取式。万一你忘记了实现这一点的重要重写规则，这里再次列出：

```
( P ⇒ Q ) ⇔ ( ( ¬P ) ∨ Q )
```

你再次需要注意，这种转换在一般情况下可能不安全，因为当你使用 SQL 时，你处于 `3VL` 的世界，而不是该重写规则所来源的 `2VL` 世界。如果允许 `NULL` 出现在任何涉及的列中，你将需要考虑这些约束在 SQL 的 `3VL` 逻辑中如何工作。

给定元组约束以清单 11-24 所示的方式声明，DBMS 将确保拒绝违反其中任何约束的行。

在清单 11-25 到 11-28 中，你可以找到表结构 `GRD`、`MEMP`、`CRS` 和 `OFFR` 的元组约束的实现。示例数据库设计中其他剩余的表结构没有元组约束。


### **GRD、MEMP、CRS 和 OFFR 表结构的元组约束**

**代码清单 11-25.** GRD 表结构的元组约束
```sql
alter table GRD add constraint grd_chk_bndw check (llimit <= (ulimit - 500));
alter table GRD add constraint grd_chk_bon2 check (bonus < llimit);
```

**代码清单 11-26.** MEMP 表结构的元组约束
```sql
alter table MEMP add constraint mmp_chk_cycl check (empno <> mgr);
```

**代码清单 11-27.** CRS 表结构的元组约束
```sql
alter table CRS add constraint reg_chk_dur2 check ((cat <> 'BLD') or (dur <= 5));
```

**代码清单 11-28.** OFFR 表结构的元组约束
```sql
alter table OFFR add constraint ofr_chk_trst check (trainer is not null or status in ('CANC','SCHD'));
```

代码清单 11-28 中所述元组约束的伴随形式化规范如下：

```
tup_OFFR :=
{ o | o∈P(chr_OFFR) ∧
/* 未分配的 TRAINER 仅允许用于特定的 STATUS 值 */
o(TRAINER) = -1 ⇒ o(STATUS)∈{'CANC','SCHD'}
}
```

将蕴涵式重写为析取式后，变为以下形式：

```
tup_OFFR :=
{ o | o∈P(chr_OFFR) ∧
/* 未分配的 TRAINER 仅允许用于特定的 STATUS 值 */
o(TRAINER) ≠ -1 ∨ o(STATUS)∈{'CANC','SCHD'}
}
```

由于我们决定在 `OFFR` 表结构的实现中使用 `NULL` 来表示 `-1`（原因将在后文解释），前面检查约束中的第一个析取项变为了 `trainer is not null`。

在结束关于实现元组约束的本节之前，我们提出一个观察结论，该结论对于其后的约束类别同样有效。

最佳实践是将所有元组约束写成**合取范式**（CNF；参见第 3 章“范式”一节）。这可能需要你首先应用各种重写规则。通过将约束重写为 CNF，你将得到尽可能多的合取项，每个合取项代表一个可单独实现的约束。对于元组约束，你会为每个合取项创建一个声明式检查约束。这反过来具有一个优点，即 DBMS 会以*尽可能详细的方式*报告对元组约束的违反。

让我们解释一下。

**注意：** 我们再次假设 SQL 的三值逻辑（3VL）以二值逻辑（2VL）的方式运行，因为约束中涉及的所有列都是强制性的（非空）。

假设你为一个在重写成 CNF 后形式为 `A ∧ B` 的元组约束创建了一个检查约束。当该检查约束被违反时，（在 2VL 中）你只知道 `A ∧ B` 不是 `TRUE`。根据德摩根定律，这转化为：要么 `A` 不是 `TRUE`，要么 `B` 不是 `TRUE`。如果你能确切知道两者中哪一个为 `FALSE`，岂不是更好？如果你当初创建了两个独立的检查约束（一个用于 `A`，一个用于 `B`），DBMS 就能报告是两者中的哪一个导致了违反（或者可能两者都是）。换句话说，通过将约束规范重写为 CNF 并单独实现每个合取项，你将获得更详细的错误信息。

如前所述，这个观察结论也适用于后续的约束类别（表、数据库和过渡约束）。

### **表约束实现问题**

到目前为止，关于数据完整性约束的实现都很直接。然而，当你的范围从元组约束扩展到表约束，从而开始处理跨越多个元组的约束时，实现高效的 DI（数据完整性）代码迅速变得复杂得多。

这种复杂性的主要原因在于向 DBMS 声明这些约束的支持力度不足。你只能以声明方式指定两种类型的表约束：唯一标识属性（键）和引用回同一表的子集要求（此时子集要求是表约束，即指向同一表的外键）。

实现所有其他类型的表约束都需要你开发过程式的 DI 代码。在实践中，这意味着你常常不得不诉诸于**触发式过程策略**。

**注意：** 我们认为 DBMS 供应商之所以只提供如此有限的声明式支持，是有原因的。我们将在本节过程中揭示这个原因。

我们将通过阐述不同的 DI 代码*执行模型*，向你介绍实现表约束所涉及的复杂性。在接下来的第一个（相当长的）子节中，我们将说明六种不同的执行模型，从非常低效到更高效。

正如你将看到的，为 DI 代码实现更高效的执行模型也更加复杂。

为了清晰地解释每个执行模型，我们将使用两个示例表约束，并展示它们如何在每个执行模型中实现。我们将使用的约束是代码清单 7-26 中表 universe `tab_EMP` 指定的最后一个约束，以及代码清单 7-30 中表 universe `tab_DEPT` 指定的最后一个约束。为方便起见，我们在此重复这两个约束的形式化规范（注意，在这些规范中，`E` 代表员工表，`D` 代表部门表）。

```
/* 雇佣了总裁或经理的部门 */
/* 也应至少雇佣一名管理员 */
( ∀d∈E⇓{DEPTNO}:
  ( ∃e2∈E: e2(DEPTNO) = d(DEPTNO) ∧ e2(JOB) ∈ {'PRESIDENT','MANAGER'} )
  ⇒
  ( ∃e3∈E: e3(DEPTNO) = d(DEPTNO) ∧ e3(JOB) = 'ADMIN' )
)

/* 一个人最多只能管理两个部门 */
( ∀m∈D⇓{MGR}: #{ d | d∈D ∧ d(MGR) = m(MGR) } ≤ 2 )
```

除了实现高效的执行模型之外，在为表约束实现 DI 代码时，另一个相当严重的问题也会出现。这涉及到*事务串行化*。鉴于 Oracle 的 SQL DBMS 可以并发执行事务，你必须确保 DI 代码内部的查询以可串行化的方式执行：Oracle 的 SQL DBMS 并不保证串行化。我们将在“DI 代码串行化”一节中详细解释这个问题。

### **DI 代码执行模型**

本节将讨论按照触发式过程策略实现表约束 DI 代码的各种执行模型。但在讨论之前，我们首先就 DI 代码执行时间与 DML 语句执行时间的关系提供一些初步的观察结论。

#### **一些观察结论**

使用 SQL DBMS 时，你通过执行 `INSERT`、`UPDATE` 或 `DELETE` 语句来更新数据库。这些语句中的每一个都只针对一个目标表结构以一种方式操作——要么是 `INSERT`，要么是 `UPDATE`，要么是 `DELETE`。通常，事务需要更改多个表，或者可能只是一个表但以多种方式更改。因此，你的事务通常由多个 DML 语句组成，这些语句一个接一个地串行执行。

当执行第一条 DML 语句时，你隐式地启动了一个事务。事务通过执行 `COMMIT` 语句（请求 DBMS 持久存储此事务所做的更改）或执行 `rollback` 语句（请求 DBMS 中止并撤消当前事务所做的更改）来显式结束。结束一个事务后，你可以通过执行另一条（第一条）DML 语句再次启动一个新事务。一个尚未提交的事务所做的所有更改仅对该事务可见；其他事务无法看到这些更改。一旦事务提交，它所做的更改就对其他事务可见。

**注意：** 这里我们假设 DBMS 运行在**读已提交**隔离模式下——这是 Oracle 已安装数据库中最常用的模式。

当然，所有约束必须在事务结束时（提交时）得到满足。


也就是说，你不希望一个事务成功提交，而其 DML 语句到目前为止串行执行所产生的数据库状态却违反了某条约束。

但是，在同一个事务内部，两条 DML 语句执行之间存在的那些数据库状态呢？是否这些中间数据库状态也必须始终满足所有约束？还是说，只要最终的数据库状态（事务提交时的状态）满足所有约束，中间状态可以违反某些约束？

目前，我们不允许这些中间数据库状态违反任何数据库约束。

##### 注意
不过，在“引入延迟检查”一节中，我们会重新审视这个问题。届时你会看到，为了使用 Oracle 的 SQL DBMS 实现某些事务，我们必须允许某些约束在某个中间数据库状态中被暂时违反。

一条试图创建违反约束的数据库状态的`DML`语句将会失败；在接下来的执行模型中，我们会确保此类`DML`语句的更改会被立即回滚，同时保留该事务内先前成功执行的`DML`语句所做的数据库状态更改。

在表 11-1 中，你可以找到一个执行了四条`DML`语句的示例事务；该表展示了在此事务内发生的数据库状态转换。

**表 11-1.** 事务内的数据库状态转换

| 时间 | 起始数据库状态 | DML | 结束数据库状态 | 注释 |
| :--- | :--- | :--- | :--- | :--- |
| | `dbs0` | `DML0;` | `dbs1` | 事务开始。`dbs1`是一个有效状态。 |
| | `dbs1` | `DML1;` | `dbs2` | `dbs2`违反约束；`DML1`被回滚。 |
| | `dbs1` | `DML2;` | `dbs3` | `dbs3`是一个有效状态。 |
| | `dbs3` | `DML3;` | `dbs4` | `dbs4`违反约束；`DML3`被回滚。 |
| | `dbs3` | `commit;` | `dbs3` | `dbs3`被提交并对其他事务可见。 |

我们的执行模型将基于触发器。如前所述，触发器与一个表结构相关联，如果一条`DML`语句更改了该表的内容，`DBMS`会自动执行（“触发”）该触发器。触发器主体内的代码可以检查新的数据库状态是否满足所有约束。如果该状态不满足所有约束，那么此代码将强制触发它的`DML`语句失败；随后`DBMS`会确保其更改被回滚。

你应该意识到`行级触发器`（为每个受影响行触发的触发器）存在一个限制。这些触发器只允许查询*其他*表结构的状态；也就是说，它们不允许查询触发它的`DML`语句当前正在操作的表结构。如果你尝试这样做，就会遇到臭名昭著的`变异表`错误（`ORA-04091: table ... is mutating, trigger/function may not see it`）。

Oracle 的`SQL DBMS`不允许你这样做的非常合理的原因是为了防止`非确定性行为`。这是因为如果你的`行级触发器`被允许查询一个`DML`语句正在修改的表结构，那么这些查询将在事务内执行脏读。这些查询看到的中间表状态仅在触发它的`DML`语句逐行执行时存在。根据 SQL 优化器处理行的顺序，这些查询的结果可能不同。这将导致`非确定性行为`，这就是 Oracle 的`DBMS`不允许你查询“变异”表的原因。

考虑到表约束的本质——即它涉及表中的多行——表约束的`DI`代码总是需要你对已被修改的表执行查询；然而，`变异表`错误阻止你这样做。因此，`行级触发器`不适合作为表约束`DI`代码的容器。

`语句前触发器`看到的是`DML`语句开始执行时的起始数据库状态。`语句后触发器`看到的是由`DML`语句执行创建的结束数据库状态。由于`DI`代码需要验证`DML`语句的结束状态，因此你只剩下每个表结构（`插入`、`更新`和`删除`）的三个`语句后触发器`可以用来构建执行模型。

基于这些观察，我们现在可以继续说明`DI`代码的六种不同执行模型。在讨论执行模型时，我们有时会扩大范围，也将数据库约束包括在内。

## 执行模型 1：总是执行
在第一个执行模型（`EM1`）中，每当执行一条`DML`语句，相应的`语句后触发器`将包含按顺序执行每个约束的`DI`代码。在此模型中，每个中间数据库状态（包括最后一个）都被验证是否满足所有约束。

这个执行模型仅作为起点；你绝不会想使用此模型，因为它效率极低。例如，如果一条`DML`语句更改了`EMP`表结构，那么此模型还会运行`DI`代码来检查那些不涉及`EMP`表结构的约束。显然，这完全是没必要的，因为其他表结构保持不变；不涉及触发它的`DML`语句所操作的表结构的约束无需验证。

让我们快速忘记这个模型，转向一个更高效的模型。

## 执行模型 2：基于涉及的表
这个执行模型（`EM2`）与`EM1`非常相似。唯一的区别是你现在利用了每个约束所涉及表结构的知识。只有当`DML`语句正在更改的表结构参与了某个约束时，你才运行该约束的`DI`代码（因此本节标题中有了“基于涉及的表”）。

让我们仔细看看`EMP`表结构的示例表约束在此执行模型中是如何实现的。记住，该约束是：“雇佣总裁或经理的部门也应至少雇佣一名管理员。”你可以形式化地推导出为验证新的数据库状态是否仍满足此约束而需要执行的约束验证查询。方法是将形式化规范转换为 SQL 的`WHERE`子句表达式，然后执行一个评估该表达式真值的查询。你可以使用`DUAL`表来评估该表达式。让我们演示一下。以下是此表约束的形式化规范：

```
( ∀d∈E⇓{DEPTNO}:
  ( ∃e2∈E: e2(DEPTNO) = d(DEPTNO) ∧ e2(JOB) ∈ {'PRESIDENT','MANAGER'} )
  ⇒
  ( ∃e3∈E: e3(DEPTNO) = d(DEPTNO) ∧ e3(JOB) = 'ADMIN' )
)
```

在将形式化规范转换为 SQL 表达式之前，你需要消除全称量词和蕴涵。以下是重写后的规范版本。

## 提示
试着自己重写这个规范；从前述规范前面加上双重否定开始。

```
¬ ( ∃d∈E⇓{DEPTNO}:
  ( ∃e2∈E: e2(DEPTNO) = d(DEPTNO) ∧ e2(JOB) ∈ {'PRESIDENT','MANAGER'} )
  ∧
  ¬ ( ∃e3∈E: e3(DEPTNO) = d(DEPTNO) ∧ e3(JOB) = 'ADMIN' )
)
```

这现在可以轻松地转换为`SQL`（我们将此约束命名为`EMP_TAB03`）。


## Oracle 中的数据库设计实现

`■Note` Oracle 中的`DUAL`表是一个单列单行的系统表。它最常用于让 SQL 引擎计算`SELECT`子句表达式或`WHERE`子句表达式。下面的代码展示了后一种用法。
```sql
select 'Constraint EMP_TAB03 is satisfied'
from DUAL
where not exists(select d.DEPTNO
                from EMP d
                where exists(select e2.*
                             from EMP e2
                             where e2.DEPTNO = d.DEPTNO
                             and e2.JOB in ('PRESIDENT','MANAGER'))
                and not exists(select e3.*
                               from EMP e3
                               where e3.DEPTNO = d.DEPTNO
                               and e3.JOB = 'ADMIN'))
```

在 EM2 中，您将仅为涉及此约束的`EMP`表结构创建三个`after statement`触发器。这些触发器包含前面的查询，以验证新的数据库状态是否仍满足此约束。清单 11-29 展示了将这三个触发器合并为一个`create trigger`语句。

**清单 11-29.** *约束`EMP_TAB03`的 EM2 DI 代码*
```sql
create trigger EMP_AIUDS_TAB03
after insert or update or delete on EMP
declare pl_dummy varchar(40);
begin
--
  select 'Constraint EMP_TAB03 is satisfied' into pl_dummy
  from DUAL
  where not exists(select d.DEPTNO
                   from EMP d
                   where exists(select e2.*
                                from EMP e2
                                where e2.DEPTNO = d.DEPTNO
                                and e2.JOB in ('PRESIDENT','MANAGER'))
                   and not exists(select e3.*
                                  from EMP e3
                                  where e3.DEPTNO = d.DEPTNO
                                  and e3.JOB = 'ADMIN'));
--
exception when no_data_found then
--
  raise_application_error(-20999,'Constraint EMP_TAB03 is violated.');
--
end;
```

`■Note` 通过解析约束声明的形式规范，DBMS 可以*计算*出涉及的表。此外，DBMS 可以*计算*需要运行的验证查询，以验证约束是否仍然满足（这只需要应用重写规则，最终得到一个可以转换为 SQL 表达式的规范）。因此，DBMS 可以生成前面的触发器。换句话说，DBMS 供应商可以*以声明方式*完全支持这种执行模型！

清单 11-30 展示了使用此执行模型表示约束`DEPT_TAB01` DI 代码的三个触发器。

**清单 11-30.** *约束`DEPT_TAB01`的 EM2 DI 代码*
```sql
create trigger DEPT_AIUDS_TAB01
after insert or update or delete on DEPT
declare pl_dummy varchar(40);
begin
--
  select 'Constraint DEPT_TAB01 is satisfied' into pl_dummy from DUAL
  where not exists(select m.DEPTNO
                   from DEPT m
                   where 2 < (select count(*)
                               from DEPT d
                               where d.MGR = m.MGR));
--
exception when no_data_found then
--
  raise_application_error(-20999,'Constraint DEPT_TAB01 is violated.');
--
end;
/
```

这种执行模型仍然效率低下。例如，当您更新员工姓名时，此执行模型将验证约束`EMP_TAB03`。显然，因为`ENAME`列完全不涉及在此约束中，更新`ENAME`的`DML`语句绝不应该引发检查约束`EMP_TAB03`的需要。类似的低效也适用于约束`DEPT_TAB01`；例如，当您更新部门位置（而属性`LOC`不涉及在约束`DEPT_TAB01`中）时，EM2 将验证此约束。下一个执行模型解决了这种低效问题。

### 执行模型 3：按涉及列

这种执行模型（EM3）在`DML`语句是`INSERT`或`DELETE`的情况下与 EM2 相同。但是，当发生更新时，您现在还利用了每个约束（按表结构）涉及哪些属性的知识。对于给定的约束，只有当涉及的属性被`UPDATE`语句修改时，才需要验证此约束。`INSERT`和`DELETE`语句总是涉及所有属性，因此无论约束涉及哪些属性，每当它们发生时，都会引发需要验证涉及表结构的约束。

让我们再次使用示例表约束`EMP_TAB03`来演示。在这种情况下，插入和删除触发器与 EM2 中的保持相同。您只需更改更新触发器以提高效率。仅通过扫描约束`EMP_TAB03`的形式规范，您就可以发现涉及的属性；对于此约束，它们是`DEPTNO`和`JOB`。

每当发生`UPDATE`语句时，您现在仅当`UPDATE`语句更改了`DEPTNO`或`JOB`（或两者）属性时，才执行在 EM2 中开发的 DI 代码。

`■Note` 您可能很容易理解，更新`JOB`可能会违反此约束。例如，在给定部门中，当您将一名管理员提升为培训师时，很可能您刚刚“移除”了该部门要求存在的唯一管理员，因为该部门还雇佣了总裁或经理。此外，例如，当您将一名培训师提升为经理时，这可能是该部门的第一位经理。这现在将要求该部门还雇佣一名管理员。类似的情况也适用于更新`DEPTNO`；如果您将目前在部门 10 工作的经理切换到部门 20 工作，那么该经理可能是部门 20 的第一位经理，因此……，等等。

SQL 标准允许您在更新触发器中指定这些列。然后，更新触发器仅在其中一个涉及的列被更改时才会触发。以下是您如何在 Oracle 中编写`after statement update`触发器（参见清单 11-31）。

**清单 11-31.** *约束`EMP_TAB03`的 EM3 更高效更新触发器*
```sql
create trigger EMP_AUS_TAB03
after update of DEPTNO,JOB on EMP
declare pl_dummy varchar(40);
begin
--
  select 'Constraint EMP_TAB03 is satisfied' into pl_dummy
  from DUAL
  where not exists(select d.DEPTNO
                   from EMP d
                   where exists(select e2.*
                                from EMP e2
                                where e2.DEPTNO = d.DEPTNO
                                and e2.JOB in ('PRESIDENT','MANAGER'))
                   and not exists(select e3.*
                                  from EMP e3
                                  where e3.DEPTNO = d.DEPTNO
                                  and e3.JOB = 'ADMIN'));
--
exception when no_data_found then
--
  raise_application_error(-20999,'Constraint EMP_TAB03 is violated.');
--
end;
/
```

清单 11-31 中的第二行指定了涉及的列。

`■Note` 与 EM2 的情况一样，DBMS 供应商也可以轻松地以声明方式支持此执行模型。与 EM2 相比，DBMS 为 EM3 需要做的额外工作是解析所有约束。它这样做不仅是为了确定 EM2 涉及的表结构，还要确定每个表结构涉及的属性是什么。DBMS 然后可以自动实现更复杂的更新触发器。

清单 11-32 展示了在`UPDATE`语句执行情况下，约束`DEPT_TAB01`的优化 DI 代码（请注意，只有`MGR`属性涉及在此约束中）。

**清单 11-32.** *更新情况下`DEPT_TAB01`的 DI 代码*
```sql
create trigger DEPT_AUS_TAB01
after update of MGR on DEPT
declare pl_dummy varchar(40);
begin
--
  select 'Constraint DEPT_TAB01 is satisfied' into pl_dummy from DUAL
```



```sql
where not exists(select m.DEPTNO
                 from DEPT m
                 where 2 < (select count(*)
                            from DEPT d
                            where d.MGR = m.MGR));
```

```sql
exception
when no_data_found then
   raise_application_error(-20999,'Constraint DEPT_TAB01 is violated.');
end;
/
```

有一种方法可以进一步提高此执行模型的效率。对于给定的约束，你有时可以推断出，一条 `INSERT` 语句（插入到涉及的某个表结构中）或一条 `DELETE` 语句（从涉及的某个表结构中删除）永远不会违反该约束。在表约束 `EMP_TAB03` 的情况下，两者都无法被推断出来。你可能会插入一名总裁，这种情况下应该验证约束。或者，你可能会删除一名管理员，这种情况下同样应该验证约束。然而，在表约束 `DEPT_TAB01` 的情况下，你可以推断删除一个部门永远不会违反此约束。

下一个执行模型针对 `INSERT` 或 `DELETE` 语句进一步探讨了这种优化。

## 执行模型 4：基于涉及列 + 涉及表的极性

此执行模型 (`EM4`) 在触发 `DML` 语句是 `UPDATE` 语句时与 `EM3` 相同。但是，当发生插入或删除时，你现在还需要利用关于每个约束下涉及表结构的 *极性* 的知识。

对于给定的约束，一个表结构的**极性**定义如下：如果向该表结构的插入可能违反约束而删除不能，则为**正向**；如果从该表结构的删除可能违反约束而插入不能，则为**负向**；如果插入和删除都导致需要验证约束，则为**中性**；如果该表结构不参与该约束，则为**未定义**。

如果对于给定的约束，一个表结构的极性是 `中性`，那么 `EM4` 等同于 `EM3`；与 `EM3` 相比，你没有机会进一步优化插入或删除触发器。然而，如果它是 `正向` 或 `负向`，那么你可以分别进一步优化 `DELETE` 或 `INSERT` 语句的触发器；实际上，你可以直接移除它们。

让我们通过检查约束 `DEPT_TAB01` 的 `DI` 代码来演示这一点。如前所述，只有插入部门才可能违反此约束；如果当前所有部门经理管理的部门都不超过两个，那么删除部门永远不会违反此约束。我们说 `DEPT` 表结构对于 `DEPT_TAB01` 约束具有 `正向` 极性。在这种情况下，你可以优化删除触发器，使其根本不为 `DEPT_TAB01` 运行任何 `DI` 代码；你根本不需要创建在 `EM3` 中为约束 `DEPT_TAB01` 所创建的那个 `DEPT` `DELETE` 语句触发器。

> **Note** 科学研究表明，`DBMS` 也可以相当轻松地计算出给定（形式化指定的）约束下涉及表结构的极性。这再次意味着，`DBMS` 供应商应该能够使用执行模型 `EM4` 为我们提供完整的声明式多元组约束支持。

尽管如此，执行模型 `EM4` 有时会在没有必要时运行 `DI` 代码。例如，考虑约束 `EMP_TAB03`，当你插入一名*销售代表*，或删除一名*培训师*时，`EM4` 将为约束 `EMP_TAB03` 运行 `DI` 代码。但显然，因为销售代表和培训师在此约束中都不扮演任何角色，所以他们的插入和/或删除绝不应导致需要检查约束 `EMP_TAB03`。下一个执行模型解决了这种低效问题。

## 执行模型 5：基于转换效应属性

此执行模型 (`EM5`) 假设可以获取给定 `DML` 的*转换效应*



