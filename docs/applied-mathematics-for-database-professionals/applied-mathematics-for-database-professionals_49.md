# 第九章 ■ 数据检索

## SEII
`select d.* from DEPT d`

这个查询很直接。它从给定的数据库状态`dbs(DEPT)`中检索部门表。另一种形式化表示方法（FEb）稍微不那么简洁。在 SEII 中，星号（`*`）符号被用作表示部门表所有属性的简写。SEII 实际上列出了所有属性（顺序是任意的）。

#### 代码清单 9-2. 查询 EXQ2(dbs)
此清单给出了月薪超过 4800，或工资级别大于等于 6 的员工的工作。

### FEa:
```
{ e(JOB) | e∈dbs(EMP) ∧ (e(MSAL) > 4800 ∨ e(SGRADE) ≥ 6) }
```

### FEb:
```
{ e↓{JOB} | e∈dbs(EMP) ∧ (e(MSAL) > 4800 ∨ e(SGRADE) ≥ 6) }
```

### FEc:
```
{ e↓(JOB) | e∈dbs(EMP) ∧ (e(MSAL) ≤ 4800 ⇒ e(SGRADE) ≥ 6) }
```

### FEd:
```
{ e↓(JOB) | e∈dbs(EMP) ∧ (e(SGRADE) < 6 ⇒ e(MSAL) > 4800) }
```

### FEe:
```
{ e↓(JOB) | e∈dbs(EMP) ∧ ¬(e(MSAL) ≤ 4800 ∧ e(SGRADE) < 6) }
```

### FEf:
```
{ e↓(JOB) | e∈dbs(EMP) ∧ e(MSAL) > 4800 }
∪
{ e↓(JOB) | e∈dbs(EMP) ∧ e(SGRADE) ≥ 6 }
```

### SEI:
```
select distinct e.JOB from EMP e where e.MSAL > 4800 or e.SGRADE >= 6
```

### SEII:
```
select distinct e.JOB from EMP e where not (e.MSAL <= 4800 and e.SGRADE < 6)
```

### SEIII:
```
select e.JOB from EMP e where e.MSAL > 4800
union
select e.JOB from EMP e where e.SGRADE >= 6
```

你注意到 FEa 和 FEb 之间的细微差别了吗？前者产生的是一个工作值的集合（即这不是一张表）；后者产生的是一个元组集合（即一个在`{JOB}`上的表），因此是形式化指定`EXQ2`的首选方式。

备选方案 FEc、FEd 和 FEe 是 FEb 的变体，其中析取被重写为蕴涵或合取。备选方案 FEf 使用并集运算符；收入超过 4800 的员工的工作与工资级别大于等于 6 的员工的工作进行并集。

**■注意** 这个查询的形式化指定方式意味着我们只检索*不同的*工作值。假设表`dbs(EMP)`当前有二十名员工，其中十名符合谓词“月薪大于 4800 或工资级别等于 6 或更高”。如果这十名员工中有八名的工作是`TRAINER`，另外两名的工作是`MANAGER`，那么 FEa 的结果将是`{'TRAINER','MANAGER'}`：一个只有两个，而不是十个元素的集合。

FEa 和 FEb 演示的细微差别无法用 SQL 表达。规范 SEI 是这两者的 SQL 变体。另请注意，在 SQL 中，必须添加`distinct`关键字以确保查询结果中不出现重复的工作值。

备选方案 FEc 和 FEd 不能直接转换为 SQL，因为 SQL 缺乏蕴涵运算符；在 SQL 中只有析取、合取和否定可用。

规范 SEII 是 FEe 的 SQL 版本，SEIII 代表 FEf 的 SQL 版本。

请注意，在后一种情况下，使用`distinct`关键字不是必需的；当使用集合运算符（`union`、`minus`或`intersect`）时，SQL 总是返回不同的值。

#### 代码清单 9-3. 查询 EXQ3(dbs)
此清单给出了办事员的编号、姓名和工资，包括他们所属工资级别的下限和上限。

### FEa:
```
{ e↓{EMPNO,ENAME,MSAL,LLIMIT,ULIMIT} | e∈dbs(EMP)⊗
(dbs(GRD)◊◊{(SGRADE;GRADE),(LLIMIT;LLIMIT),(ULIMIT;ULIMIT)}) ∧
e(JOB)='CLERK' }
```

### SEI:
```
select e.EMPNO, e.ENAME, e.MSAL, g.LLIMIT, g.ULIMIT
from EMP e, GRD g
where e.SGRADE = g.GRADE
and e.JOB = 'CLERK'
```

这个查询由员工表和工资级别表之间的连接表示。为了能够使用连接运算符，你必须重命名员工表中的属性`SGRADE`或工资级别表中的属性`GRADE`。规范 FEa 执行了后者重命名。你可以直接从 FEa 推导出 SQL 表达式 SEI。

现在你可能在想 SQL 表达式看起来比形式化表达式容易得多，对吧？这是因为这个例子太简单了。当我们看到更复杂的例子时，你会看到这种情况不再适用；那时 SQL 表达式将变得比数学表达式更复杂。

#### 代码清单 9-4. 查询 EXQ4(dbs)
此清单给出了在两个不同地点恰好管理两个部门的员工（`EMPNO`和`ENAME`）。

### FEa:
```
{ e↓{EMPNO,ENAME} | e∈dbs(EMP) ∧
#{ d(LOC) | d∈dbs(DEPT) ∧ d(MGR)=e(EMPNO) } = 2 }
```

### FEb:
```
{ e↓{EMPNO,ENAME} | e∈dbs(EMP) ∧
(∃d1,d2∈dbs(DEPT): d1(MGR)=e(EMPNO) ∧
d2(MGR)=e(EMPNO) ∧
d1(LOC)≠d2(LOC)) }
```

### FEc:
```
{ e↓{EMPNO,ENAME} | e∈(dbs(EMP)⇓{EMPNO,ENAME})⊗
(dbs(DEPT)◊◊{(EMPNO;MGR),(LOC1;LOC)})⊗
(dbs(DEPT)◊◊{(EMPNO;MGR),(LOC2;LOC)}) ∧
e(LOC1)≠e(LOC2) }
```

### SEI:
```
select e.EMPNO, e.ENAME
from EMP e
where 2 = (select count(distinct d.LOC)
from DEPT d
where d.MGR = e.EMPNO)
```

### SEII:
```
select e.EMPNO, e.ENAME
from EMP e
where exists(select d1.*
from DEPT d1
where d1.MGR = e.EMPNO
and exists(select d2.*
from DEPT d2
where d2.MGR = e.EMPNO
and d1.LOC <> d2.LOC))
```

### SEIII:
```
select e.EMPNO, e.ENAME
from EMP e
where exists(select *
from DEPT d1, DEPT d2
where d1.MGR = e.EMPNO
and d2.MGR = e.EMPNO
and d1.LOC <> d2.LOC)
```

### SEIV:
```
select distinct e.EMPNO, e.ENAME
from EMP e, DEPT d1, DEPT d2
where e.EMPNO = d1.MGR
and e.EMPNO = d2.MGR
and d1.LOC <> d2.LOC
```

规范 FEa 指出，由该员工管理的部门地点集合的基数必须为 2。备选方案 FEb 通过对两个由该员工管理的不同地点的部门进行存在性量化来实现这一要求。

**■注意** 在 FEb 中，使用了对同一集合的两个嵌套存在性量词的简写表示法。

你可以直接将这两种形式化规范翻译成 SQL。注意，在 SEI 中，必须在`count`聚合函数内使用`distinct`关键字。规范 SEII 用于演示 SQL 中可用的存在性量词——`exists(...)`表达式的嵌套使用。SEIII 是一个简写版本，类似于形式化规范 FEb 中使用的简写。

规范 FEc 使用连接来定位由该员工管理的两个不同地点的部门。注意，在其 SQL 版本（SEIV）中，必须使用`distinct`关键字，以防止管理两个不同地点部门的员工在结果集中出现两次。

#### 代码清单 9-5. 查询 EXQ5(dbs)
此清单给出了所有收入超过其上司工资 90%的被管理员工的编号和姓名。

### FEa:
```
{ e1↓{EMPNO,ENAME} | e1∈dbs(EMP) ∧ m∈dbs(MEMP) ∧ e2∈dbs(EMP) ∧
e1(EMPNO) = m(EMPNO) ∧ m(MGR) = e2(EMPNO) ∧
e1(MSAL) > 0.9*e2(MSAL) }
```

### FEb:
```
{ e↓{EMPNO,ENAME} | e∈(dbs(EMP)⊗dbs(MEMP))⊗
(dbs(EMP)◊◊{ (MGR;EMPNO), (MGR_MSAL;MSAL) }) ∧
e(MSAL) > 0.9*e(MGR_MSAL) }
```

### FEc:
```
{ e1↓{EMPNO,ENAME} | e1∈dbs(EMP) ∧
(∃m∈dbs(MEMP): m(EMPNO)=e1(EMPNO) ∧
(∃e2∈dbs(EMP): e2(EMPNO)=m(MGR) ∧
e1(MSAL) > 0.9*e2(MSAL) )) }
```

### SEI:
```
select e1.EMPNO, e1.ENAME
from EMP e1, MEMP m, EMP e2
where e1.EMPNO = m.EMPNO
and m.MGR = e2.EMPNO
and e1.MSAL > 0.9 * e2.MSAL
```

### SEII:
```
select e1.EMPNO, e1.ENAME
from EMP e1
where exists (select m.*
from MEMP m
where m.EMPNO = e1.EMPNO
and exists (select e2.*
from EMP e2
where e2.EMPNO = m.MGR
and e1.MSAL > 0.9 * e2.MSAL))
```


通过结合每个 EMP 元组与对应的 MEMP 元组（通过特化 `PSPEC2`），然后将 MEMP 元组合并回 EMP 元组（通过子集要求 `PSSR3`），您可以将受管理员工的月薪与其经理的月薪进行比较。规范 `FEa` 通过绑定两个用于 `EMP` 的元组变量（一个用于员工，一个用于经理）和一个用于 `MEMP` 的元组变量，然后对这些变量施加组合限制来实现这一点。规范 `FEb` 通过绑定一个从执行组合的嵌套连接中提取值的单一元组变量来实现这一点。第一个连接将 `EMP` 与 `MEMP` 的元组合并（以找到经理的员工编号）。第二个连接将 `MEMP` 的元组合并回 `EMP`（以找到经理的薪资）并要求属性重命名。

规范 `SEI` 代表了 `FEa` 和 `FEb` 的 SQL 版本。

规范 `FEc` 采用嵌套存在量化来实现相同的结果。

您可以利用 SQL 中对存在量化的支持，将其直接翻译为 SQL（参见 `SEII`）。

代码清单 9-6 给出了所有课程的代码、描述和类别，这些课程的每个预定授课单元的最大容量都少于十名学生。

**代码清单 9-6.** 查询 `EXQ6(dbs)`

`FEa`:

```
{ c↓{CODE,DESCR,CAT} | c∈dbs(CRS) ∧

(∀o∈dbs(OFFR): (o(COURSE) = c(CODE) ∧

o(STATUS) = 'SCHD') ⇒

o(MAXCAP) < 10 ) }
```

`FEb`:

```
{ c↓{CODE,DESCR,CAT} | c∈dbs(CRS) ∧

¬(∃o∈dbs(OFFR): o(COURSE) = c(CODE) ∧

o(STATUS) = 'SCHD' ∧

o(MAXCAP) ≥ 10 ) }
```

`FEc`:

```
{ c↓{CODE,DESCR,CAT} | c∈dbs(CRS) ∧

c(CODE) ∉ { o(COURSE) | o∈dbs(OFFR) ∧

o(STATUS) = 'SCHD' ∧

o(MAXCAP) ≥ 10 ) } }
```

`FEd`:

```
{ c↓{CODE,DESCR,CAT} | c∈dbs(CRS) ∧

#{ o1 | o1∈dbs(OFFR) ∧ o1(COURSE) = c(CODE) ∧

o1(STATUS) = 'SCHD' }

=

#{ o2 | o2∈dbs(OFFR) ∧ o2(COURSE) = c(CODE) ∧

o2(STATUS) = 'SCHD' ∧

o2(MAXCAP) < 10 ) } }
```

`SEI`:

```sql
select c.CODE, c.DESC, c.CAT
from CRS c
where not exists (select o.*
                  from OFFR o
                  where o.COURSE = c.CODE
                  and o.STATUS = 'SCHD'
                  and o.MAXCAP >= 10)
```

`SEII`:

```sql
select c.CODE, c.DESC, c.CAT
from CRS c
where c.CODE not in (select o.COURSE
                     from OFFR o
                     where o.STATUS = 'SCHD'
                     and o.MAXCAP >= 10)
```

`SEIII`:

```sql
select c.CODE, c.DESC, c.CAT
from CRS c
where (select count(*)
       from OFFR o1
       where o1.COURSE = c.CODE
       and o1.STATUS = 'SCHD')
      =
      (select count(*)
       from OFFR o2
       where o2.COURSE = c.CODE
       and o2.STATUS = 'SCHD'
       and o2.MAXCAP < 10)
```

规范 `FEa` 紧密遵循了查询的非正式描述。`FEb` 是 `FEa` 的一个重写版本；全称量词被重写为存在量词。非正式地讲，它查询所有不存在最大容量为十或更多的预定授课单元的课程。因为将谓词重写为等价谓词的能力极其重要，我们将在此案例中再次演示其执行过程：

`(∀o∈dbs(OFFR): (o(COURSE)=c(CODE) ∧ o(STATUS)='SCHD') ⇒ o(MAXCAP)<10 )`

```
⇔ /* Add double negation */
¬¬(∀o∈dbs(OFFR): (o(COURSE)=c(CODE) ∧ o(STATUS) = 'SCHD') ⇒ o(MAXCAP)<10 )
⇔ /* Bring one negation into the quantification, quantifier changes */
¬(∃o∈dbs(OFFR): ¬((o(COURSE)=c(CODE) ∧ o(STATUS)='SCHD') ⇒ o(MAXCAP)<10 ))
⇔ /* Transform implication into disjunction */
¬(∃o∈dbs(OFFR): ¬(¬(o(COURSE)=c(CODE) ∧ o(STATUS)='SCHD') ∨ o(MAXCAP)<10 ))
⇔ /* Apply De Morgan, disjunction changes into conjuction */
¬(∃o∈dbs(OFFR): ¬¬(o(COURSE)=c(CODE) ∧ o(STATUS)='SCHD') ∧ ¬o(MAXCAP)<10 )
⇔ /* Get rid of negations */
¬(∃o∈dbs(OFFR): o(COURSE)=c(CODE) ∧ o(STATUS)='SCHD' ∧ o(MAXCAP)≥10 )
```

您经常会需要将全称量化重写为存在量化，原因很快就会明了。



规范 `FEc` 是 `FEb` 的一种变体；“不存在”现在被表达为“不是某个集合的元素”。`FEd` 可能看起来有点牵强；只有那些已安排的课程开设数量等于那些最大容量少于十个学生的已安排课程开设数量的课程，才会被包含在查询结果中。通过施加这个限制，你最终得到的正好是那些每个已安排的开设其最大容量都少于十个学生的课程。

SQL 缺乏直接表达全称量化的能力。

## ■注意

请记住，本书中我们使用的是 `Oracle 版本的 SQL`。SQL 标准提供了一种称为 `<量化比较谓词>` 的语言结构，可用于在 SQL 中指定全称量化。然而，Oracle 的 SQL DBMS 不支持这种语言结构。

因此，你无法将 `FEa` 直接翻译成 SQL；你必须将全称量化重写为存在量化（SQL 支持后者）。一旦你完成了这项工作（这正是备选方案 `FEb` 的核心），你就可以直接为该查询编写等效的 SQL 表达式；参见 `SEI`。备选方案 `SEII` 也紧密遵循 `FEb`，但它使用了 SQL 中可用的“不存在”运算符，而非“不在”运算符。

备选方案 `SEIII` 是 `FEc` 的 SQL 版本；这种查询表述方式可以被视为在 SQL 中模拟全称量化的一种技巧。

你对当前**完全没有**任何已安排开设的课程有何看法？它们是否应该出现在查询 `EXQ6` 的结果集中？

## ■注意

还记得对空集进行全称量化的真值吗？无论被量化的谓词是什么，这样的量化结果总是为真。

没有已安排开设的课程将会出现在此查询的结果集中。每当你遇到全称量化——或一个否定的存在量化时——你应该意识到这个后果（警钟应该响起）。此外，每当被量化的集合可能为空时，你都应该与你的客户（最终用户）再次确认，确保他或她知晓这种“副作用”，并需要查询具备这种行为。

如果完全没有已安排开设的课程不应该出现在此查询的结果集中，那么你可以按如下方式修改 `FEa` 来反映这一点：

7451CH09.qxd 5/7/07 11:28 AM Page 208

**208**

第 9 章 **■** 数 据 检 索

```
{ c↓{CODE,DESCR,CAT} | c∈dbs(CRS) ∧
(∃o∈dbs(OFFR): o(COURSE) = c(CODE) ∧
o(STATUS) = 'SCHD') ∧
(∀o∈dbs(OFFR): (o(COURSE) = c(CODE) ∧
o(STATUS) = 'SCHD') ⇒
o(MAXCAP) < 10 ) }
```

现在添加存在量化可确保没有已安排开设的课程不会出现在结果集中。

在下一个示例中，你将看到 `NVL` 函数和外连接语法 `(+)` 的使用。这两种结构在 Oracle SQL DBMS 中都可用。我们将在列表 9-7 之后直接解释它们的含义。

列表 9-7 列出了每个部门的部门编号、名称、在该部门工作的员工人数，以及这些员工的月工资总额。

#### 列表 9-7. 查询 `EXQ7(dbs)`

**FEa**:

```
{ d↓{DEPTNO,DNAME} ∪
{ (NUM_EMP; #{ e1 | e1∈dbs(EMP) ∧ e1(deptno)=d(deptno) })
,(SUM_MSAL; (SUM e2∈{ e3 | e3∈dbs(EMP) ∧ e3(deptno)=d(deptno) }: e2(MSAL)))
}
| d∈dbs(DEPT) }
```

**FEb**:

```
{ d ∪
{ (NUM_EMP; #{ e1 | e1∈dbs(EMP) ∧ e1(deptno)=d(deptno) })
,(SUM_MSAL; (SUM e2∈{ e3 | e3∈dbs(EMP) ∧ e3(deptno)=d(deptno) }: e2(MSAL)))
}
| d∈dbs(DEPT)⇓{DEPTNO,DNAME} }
```

**SEI**:

```sql
select d.DEPTNO
      ,d.DNAME
      ,(select count(*)
        from EMP e1
        where e1.deptno = d.deptno) as NUM_EMP
      ,(select nvl(sum(MSAL),0)
        from EMP e2
        where e1.deptno = d.deptno) as SUM_MSAL
from DEPT d
```

**SEII**:

```sql
select d.DEPTNO
      ,d.DNAME
      ,count(*) as NUM_EMP
      ,sum(e.MSAL) as SUM_MSAL
from DEPT d
    ,EMP e
where d.DEPTNO = e.DEPTNO
group by d.DEPTNO,d.DNAME
```

**SEIII**: select d.DEPTNO

7451CH09.qxd 5/7/07 11:28 AM Page 209


