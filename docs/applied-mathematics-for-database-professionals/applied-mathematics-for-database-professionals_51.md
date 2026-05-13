# 第 9 章 ■ 数据检索

#### 代码清单 9-9. 查询 EXQ9(dbs)

### FEa:

```
{ e↓{EMPNO,ENAME} | e∈dbs(EMP) ∧

{ o(COURSE) | o∈dbs(OFFR) ∧ o(TRAINER)=1206 ∧

o(STATUS)='CONF' ∧ to_year(o(STARTS))=2003 }

⊆

{ r(COURSE) | r∈dbs(REG)⊗dbs(OFFR) ∧

r(STUD)=e(EMPNO) ∧ r(TRAINER)=1206 ∧

r(STATUS)='CONF' ∧ to_year(r(STARTS))=2003 }

}
```

### FEb:

```
{ e↓{EMPNO,ENAME} | e∈dbs(EMP) ∧

(∀o1∈{ o2↓{COURSE} | o2∈dbs(OFFR) ∧ o2(TRAINER)=1206 ∧

o2(STATUS)='CONF' ∧ to_year(o2(STARTS))=2003 }:

(∃r∈dbs(REG)⊗dbs(OFFR): r(COURSE)=o1(COURSE) ∧

r(STUD)=e(EMPNO) ∧ r(TRAINER)=1206 ∧

r(STATUS)='CONF' ∧ to_year(r(STARTS))=2003))

}
```

### SEI:

```sql
select e.EMPNO,e.ENAME
from EMP e
where 0 = (select count(*)
from (select o1.COURSE
from OFFR o1
where o1.TRAINER = 1206
and o1.STATUS = 'CONF'
and to_char(o1.STARTS,'YYYY') = '2003'
minus
select o2.COURSE
from OFFR o2
,REG r
where o2.COURSE = r.COURSE
and o2.STARTS = r.STARTS
and r.STUD = e.EMPNO
and o2.TRAINER = 1206
and o2.STATUS = 'CONF'
and to_char(o2.STARTS,'YYYY') = '2003'))
```

### SEII:

```sql
select e.EMPNO,e.ENAME
from EMP e
where not exists(select o1.*
from (select o.COURSE
from OFFR o
where o.TRAINER = 1206
and o.STATUS = 'CONF'
and to_char(o.STARTS,'YYYY') = '2003') o1
where not exists(select r.*
from REG r
,OFFR o2
where r.COURSE = o2.COURSE
and r.STARTS = o2.STARTS
and r.STUD = e.EMPNO
and o2.STATUS = 'CONF'
and to_char(r.STARTS,'YYYY') = '2003'
and r.TRAINER = 1206
and r.COURSE = o1.COURSE))
```

形式化规范 `FEa` 查询所有员工，对于这些员工，De Haan 在 2003 年讲授的不同课程集合是该员工在 2003 年参加过的（由 De Haan 讲授的）课程的一个子集。请注意，要使员工参加了某门课程，必须满足以下条件：

1.  他或她必须注册了该课程。
2.  该课程的状态必须是已确认。

规范 `FEb` 是将 `FEa` 中的子集操作符重写为根据子集操作符定义推导出的量词。集合 `A` 是 `B` 的子集，当且仅当 `A` 的每个元素都是 `B` 的元素。

```
(A ⊆ B) ⇔ (∀a∈A: ∃b∈B: b=a)
```

我们应用的重写规则稍微复杂一些；其结构如下：

```
{ a | a∈A ∧ P(a) } ⊆ { b | b∈B ∧ Q(b) }

⇔

(∀a1∈{ a2 | a2∈A ∧ P(a2) }: (∃b∈B: b=a1 ∧ Q(b)))
```

SQL 并不支持所有的集合操作符：支持 `union`（并集）、`intersection`（交集）和 `difference`（差集）；不支持 `symmetric difference`（对称差集），也不支持 `subset of`（子集）操作符。幸运的是，你可以使用其他可用的集合操作符来重写子集操作符：

```
(A ⊆ B) ⇔ ((A − B) = ∅)
```

集合 `A` 是 `B` 的子集，当且仅当 `A` 减 `B` 的差集代表空集。要在 SQL 中检查空集，你还需要应用一条重写规则：`((A − B) = ∅) ⇔ (#(A − B) = 0)`

你现在检查的是 `A` 减 `B` 的差集的基数是否为零，而不是检查其是否代表空集。前面的重写规则已应用于 `FEa`，最终得到 SQL 表达式 `SEI`。

备选方案 `SEII` 代表了将子集操作符重写为量词 (`FEb`) 后的形式化版本的 SQL 实现。注意（再次）全称量词必须被进一步重写为存在量词。

代码清单 9-10 给出了课程 (`CODE` 和 `DESCR`)，这些课程在 2006 年的所有开课（且至少必须有一次）只由一位讲师讲授，并且这些开课中没有任何一次被其他讲师参加。

#### 代码清单 9-10. 查询 EXQ10(dbs)

### FEa:

```
{ c↓{CODE,DESCR}
| c∈dbs(CRS) ∧ /* 该课程的所有开课只有一位讲师 */

(∃e∈dbs(EMP):

(∀o∈dbs(OFFR): (o(COURSE)=c(CODE) ∧ o(STATUS)='CONF' ∧

to_year(o(STARTS))=2006) ⇒

o(TRAINER)=e(EMPNO) )

∧ /* 该课程在 2006 年有开课 */

(∃o∈dbs(OFFR): o(COURSE)=c(CODE) ∧ o(STATUS)='CONF' ∧

to_year(o(STARTS))=2006)

∧ /* 这些开课中没有任何一次被其他讲师参加 */

¬(∃r∈dbs(REG)⊗

(dbs(EMP)◊◊{(STUD;EMPNO),(JOB;JOB)})⊗dbs(OFFR):

r(COURSE)=c(CODE) ∧ r(STATUS)='CONF' ∧
```


#### 查询示例：EXQ9

`FEa`:

```
{ c↓{CODE,DESCR} |
  c∈dbs(CRS) ∧ /* 该课程的所有班次均由一名讲师负责 */
  1 = #{ o(TRAINER) | o∈dbs(OFFR) ∧ o(COURSE)=c(CODE) ∧
                   o(STATUS)='CONF' ∧ to_year(o(STARTS))=2006 }
  ∧ /* 没有其他讲师参加这些班次 */
  (∀r∈dbs(REG)⊗
     (dbs(EMP)◊◊{(STUD;EMPNO),(JOB;JOB)})⊗dbs(OFFR):
       (r(COURSE)=c(CODE) ∧ r(STATUS)='CONF' ∧
        to_year(r(STARTS))=2006) ⇒ r(JOB)≠'TRAINER')
}
```

`FEb`:

```
{ c↓{CODE,DESCR} |
  c∈dbs(CRS) ∧ /* 该课程的所有班次均由一名讲师负责 */
  1 = #{ o(TRAINER) | o∈dbs(OFFR) ∧ o(COURSE)=c(CODE) ∧
                   o(STATUS)='CONF' ∧ to_year(o(STARTS))=2006 }
  ∧ /* 没有其他讲师参加这些班次 */
  (∀r∈dbs(REG)⊗
     (dbs(EMP)◊◊{(STUD;EMPNO),(JOB;JOB)})⊗dbs(OFFR):
       (r(COURSE)=c(CODE) ∧ r(STATUS)='CONF' ∧
        to_year(r(STARTS))=2006) ⇒ r(JOB)≠'TRAINER')
}
```

`SEI`:

```sql
select c.CODE
       ,c.DESCR
from   CRS c
where  exists(select e.*
              from   EMP e
              where  not exists(select o2.*
                                from   OFFR o2
                                where  o2.COURSE = c.CODE
                                and    o2.STATUS = 'CONF'
                                and    to_char(o2.STARTS,'YYYY') = '2006'
                                and    o2.TRAINER <> e.EMPNO))
and    exists(select o1.*
              from   OFFR o1
              where  o1.COURSE = c.CODE
              and    o1.STATUS = 'CONF'
              and    to_char(o1.STARTS,'YYYY') = '2006')
and    not exists(select r.*
                  from   REG r
                        ,EMP e
                        ,OFFR o3
                  where  r.STUD = e.EMPNO
                  and    r.COURSE = o3.COURSE
                  and    r.STARTS = o3.STARTS
                  and    r.COURSE = c.CODE
                  and    o3.STATUS = 'CONF'
                  and    to_char(o3.STARTS,'YYYY') = '2006'
                  and    e.JOB = 'TRAINER')
```

`SEII`:

```sql
select c.CODE, c.DESCR
from   COURSE c
where  1 = (select count(distinct o1.TRAINER)
            from   OFFR o1
            where  o1.COURSE = c.CODE
            and    o1.STATUS = 'CONF'
            and    to_char(o1.STARTS,'YYYY') = 2006)
and    not exists(select r.*
                  from   REG r
                        ,EMP e
                        ,OFFR o2
                  where  r.STUD = e.EMPNO
                  and    r.COURSE = o2.COURSE
                  and    r.STARTS = o2.STARTS
                  and    r.COURSE = c.CODE
                  and    o2.STATUS = 'CONF'
                  and    to_char(o2.STARTS,'YYYY') = '2006'
                  and    e.JOB = 'TRAINER')
```

注意`FEa`规范内部额外的合取项（带有注释`/* 课程于 2006 年开设 */`），用于确保第一个合取项内的全称量化不涉及空集。

`FEb`通过声明在 2006 年为该课程授课的所有不同讲师的基数至少为一来处理此问题。通过声明此基数应恰好为一，您指定了该课程由不超过一名讲师讲授。

您可以将这两个形式化规范翻译成 SQL；`FEa`需要重写全称量词。

#### 查询示例：EXQ11

**清单 9-11.** `查询 EXQ11(dbs)`

`FEa`:

```
{ d | d∈dbs(DEPT) ∧
  (∃e1∈dbs(EMP): e1(DEPTNO)=d(DEPTNO) ∧
   e1(JOB)∈{'PRESIDENT','MANAGER'}) ∧
  ¬(∃e2∈E: e2(DEPTNO)=d ∧ e2(JOB) = 'ADMIN' ) }
```

`FEb`:

```
{ d | d∈dbs(DEPT) ∧
  (∃e1∈dbs(EMP): e1(DEPTNO)=d(DEPTNO) ∧
   e1(JOB)∈{'PRESIDENT','MANAGER'}) ∧
  (∀e2∈dbs(EMP): e2(JOB)='ADMIN' ⇒ e2(DEPTNO)≠d(DEPTNO)) }
```

`SEI`:

```sql
select d.*
from   DEPT d
where  exists(select e1.*
              from   EMP e1
              where  e1.DEPTNO = d.DEPTNO
              and    e1.JOB in ('PRESIDENT','MANAGER'))
and    not exists(select e2.*
                  from   EMP e2
                  where  e2.DEPTNO = d.DEPTNO
                  and    e1.JOB = 'ADMIN')
```

清单 7-26 介绍了此表约束的形式化规范。使用此规范，您可以推导出查询 `EXQ11` 的形式化表达式。因为您想查找违反此约束的部门，所以您只需要否定约束规范；只需在规范前加上 `¬`。

如下所示（请注意，在此规范中，自由变量 `E` 代表一个员工表）：

```
¬(∀d∈{ e1(DEPTNO) | e1∈E }:
    (∃e2∈E: e2(DEPTNO)=d ∧ e2(JOB)∈{'PRESIDENT','MANAGER'} )
    ⇒
    (∃e3∈E: e3(DEPTNO)=d ∧ e3(JOB)='ADMIN' ) )
```

如果您然后通过将否定引入量化来重写此规范，您最终会得到以下规范（请自行验证）：

```
(∃d∈{ e1(DEPTNO) | e1∈E }:
    (∃e2∈E: e2(DEPTNO)=d ∧ e2(JOB)∈{'PRESIDENT','MANAGER'} ) ∧
    ¬(∃e3∈E: e3(DEPTNO)=d ∧ e3(JOB)='ADMIN' ) )
```

形式化表达式 `FEa` 现在可以直接从前面重写的约束规范中得出。`FEb` 是 `FEa` 的重写；用全称量词替换了否定的存在量词。

您可以从规范 `FEa` 轻松推导出 SQL 文本 (`SEI`)。

#### 关于否定的说明

您可能已经注意到，由于 SQL 缺乏全称量化和蕴含，某些 SQL 表达式最终比其形式化对应物包含更多的否定。`EXQ9` 的 `SEII` 就是一个很好的例子；它包含一个嵌套的否定存在量词。

理解此类查询的含义很困难。采用全称量词的形式化对应物更容易理解。

通常，我们（人类）难以理解包含否定的表达式；否定越多，对我们来说就越难理解。当您在形式化规范中遇到否定时，您应该研究将此类规范重写为包含更少否定的表达式的机会。您可以使用本书第 1 部分介绍的重写规则来实现这一点。

> **注意** 通过重写形式化规范以使其包含更少的否定，您不仅帮助了自己，还使您的客户（用户）能够获得更清晰的非形式化规范。

因为它们非常重要，我们在此再次列出涉及否定的重写规则。表 9-1 包含了德摩根定律、蕴含和量词重写规则的各种替代形式。

**表 9-1.** `包含否定的重写规则`

**表达式** | **等价于**
---|---
`¬¬A` | ⇔ `A`
`¬A ⇒ ¬B` | ⇔ `B ⇒ A`
`¬A ⇒ B` | ⇔ `¬B ⇒ A`
`A ⇒ ¬B` | ⇔ `B ⇒ ¬A`
`A ⇒ ¬B` | ⇔ `(A ∧ B) ⇒ FALSE`
`¬A ∧ ¬B` | ⇔ `¬( A ∨ B )`
`A ∧ ¬B` | ⇔ `¬( ¬A ∨ B )`
`¬A ∧ B` | ⇔ `¬( A ∨ ¬B )`
`¬A ∨ ¬B` | ⇔ `¬( A ∧ B )`
`A ∨ ¬B` | ⇔ `¬( ¬A ∧ B )`
`¬A ∨ B` | ⇔ `¬( A ∧ ¬B )`
`¬(∃x∈X: ¬P(x))` | ⇔ `(∀x∈X: P(x))`
`(∃x∈X: ¬P(x))` | ⇔ `¬(∀x∈X: P(x))`
`¬(∃x∈X: P(x))` | ⇔ `(∀x∈X: ¬P(x))`
`(∃x∈X: P(x))` | ⇔ `¬(∀x∈X: ¬P(x))`

有时，您可以利用有关数据库设计约束的知识来消除否定。例如，如果您需要查询所有未取消的课程，您可以等效地查询所有已安排或已确认的课程。如果 `o` 代表一个课程元组，则以下重写规则将适用：

```
o(STATUS) ≠ 'CANC' ⇔ o(STATUS)∈{'SCHD','CONF'}
```

这里您使用了关于属性 `STATUS` 的属性值集的知识。

如果——在重写表达式的过程中——您最终需要查询薪资不低于 5000 的员工，那么查询薪资等于或大于 5000 的员工可能更好。以下重写规则适用（`e` 代表一个员工元组）：

```
¬(e(MSAL) < 5000) ⇔ e(MSAL) ≥ 5000
```

我们的大脑不喜欢否定。尝试通过将表达式重写为包含更少否定的表达式来避免否定。这不仅适用于查询规范，也适用于约束规范（前几章）和数据操作规范（下一章）。


#### 章节总结

本节以项目符号列表的形式提供了本章的总结。你可以在继续下一节的练习之前，使用它来检查自己对本章介绍的各种概念的理解。

• 查询构成了数据库的一项关键应用；它们使你能够从数据库中提取对你相关的信息。

• 你可以将查询形式化地表示为一个定义在数据库全域上的函数。对于你提供的每一个数据库状态作为参数，该函数都会返回查询结果。

• 因为你提供了数据库状态，该函数可以引用该数据库状态中所有可用的表来确定查询结果。

• 要指定一个查询，你可以使用本书第一部分介绍的所有形式化概念。

你通常使用混合方法将查询的结果集指定为一个元组集合（一张表）。你开始进行这种指定的给定集合通常是一个表，或者是数据库状态中某些可用表的连接；你也可以使用本书中介绍的任何集合运算符。查询规范内部的谓词通常是复合谓词（它们使用各种逻辑连接词），并且经常使用全称量词和存在量词。

• 在实践中（当然，当你使用 `SQL` 时），查询的结果总是一张表。
然而，我们的形式化模型并不强制要求这一点。

• 当你使用 `SQL` 作为查询语言时，你应该意识到其各种限制。
以下是最值得注意的几点：

• `SQL` 并非面向集合的；你有时必须使用 `distinct` 关键字。

• `SQL` 缺少蕴涵运算符；你必须将其重写为析取式。

• 甲骨文版本的 `SQL` 缺少全称量化；你必须将其重写为存在量化。

• `SQL` 缺少子集运算符；你必须使用其他可用的集合运算符（`union`、`intersect` 和 `minus`）来重写。

##### 练习

为这些查询开发形式化表达式和 `SQL` 表达式。

**注意**：在以下练习中，可能存在无法用给定数据库回答的查询。
在这些情况下，你应该给出无法给出答案的理由。

### 1. 给出属于 10 号部门的员工的编号和姓名。

### 2. 给出不属于 10、11、12 或 15 号部门的员工的编号和姓名。

### 3. 给出同时属于 10 号和 11 号部门的员工的编号和姓名。

### 4. 给出属于 10 号部门的子部门的员工的编号和姓名。

### 5. 确保最多只有一位总裁的约束未被违反。

### 6. 给出年龄超过 35 岁且月薪超过 2000 的行政人员的编号和姓名。

### 7. 给出赚取其工资等级最高薪资的经理的编号和姓名。

### 8. 给出在 2004 年至少一门课程中实际担任过培训师的培训师的编号和姓名。

### 9. 给出在 2004 年未在任何课程中担任过培训师的培训师的编号和姓名。

### 10. 给出每位经理的编号、姓名及其薪资，但仅限于该经理的薪资低于其任何下属员工的情况。

### 11. 查找一个查询，如果 `COURSE` 在 `OFFR` 中能唯一标识则答案为“是”，否则为“否”。答案“是”是否意味着 `COURSE` 是 `OFFR` 的一个键？

### 12. 给出每位经理的编号、姓名，以及其每一位下属员工的薪资。

### 13. 对于每位经理的经理是另一位员工（超级经理）的员工，给出该员工及其超级经理的编号和姓名。

### 14. 给出每位经理及其每一位身为经理的直属下属的编号、姓名和雇佣日期。

### 15. 给出每位经理及其每一位身为经理的下属的编号、姓名以及他们获得经理职位的日期。

### 16. 给出在 2006 年离职并在同一年返回的员工的编号和姓名。

### 17. 给出在 2006 年离职且未返回的员工的编号和姓名。

### 18. 给出 2006 年离开 10 号部门的人数以及进入该部门的人数。

### 19. 给出 2006 年底时 10 号部门的人数。

### 20. 列出 2006 年 3 月 4 日开始的“数据库设计”课程的以下信息：每位注册学生的用户名、姓名、职位和评价。

### 21. 对于代码（`CODE`）为 `DB1`、`DB2`、`DB3` 的每门课程，给出其时长，以及 2006 年每一次开课（带有开始日期）的开始日期、状态和注册学生人数。

### 22. 按部门（`DEPTNO` 和 `DNAME`）分别给出行政人员、培训师、销售代表和经理的人数。

### 23. 给出所有满足以下条件的工资等级数据：该等级下的每位员工（即被分配到此等级的员工）的薪资都高于该工资等级上限减去 500 美元。

### 24. 给出所有被分配到高于其上司工资等级的（被管理）员工。

### 25. 给出满足以下条件的课程：2006 年每一次被取消的开课都至少有一名学生注册了该次开课。

### 26. 给出在 2006 年参加了所有设计类课程（课程类别等于 'DSG'）的（至少一次）开课的员工（`EMPNO` 和 `ENAME`）。

### 27. 列出在 2006 年获得超过 20% 加薪的所有员工。

## 数据操作

通过执行事务，你可以操作数据库中的数据。在本章中，我们将演示如何以形式化的方式指定事务。我们不会形式化地引入任何新概念；我们已经介绍了指定事务所需的所有要素。

“形式化指定事务”一节逐步阐述了如何指定事务。你将了解到，可以将事务指定为当前数据库全域上的一个函数。对于每一个数据库状态，该函数返回另一个反映了该事务预期更改的数据库状态。在本节中，我们还将给出两个示例，以帮助你熟悉事务的这种形式化概念。

“在 `DB_UEX` 上的事务示例”一节为第 7 章引入的数据库全域提供了示例事务。这是你第二次在本书中看到 `SQL` 的使用。每个示例事务都将附带一个或多个等效的 `SQL` 数据操作语言（DML）语句；即 `INSERT`、`UPDATE` 和 `DELETE` 语句。你将了解到，用 `SQL` 表达某些事务需要执行多个 DML 语句。

**注意**：我们假设你具备 `SQL` DML 语句的实用知识。本书的目标不是教授 `SQL`。

为下一章（第 11 章）做准备，在那一章你将了解到使用 `SQL` DBMS 实现数据完整性约束所面临的挑战，我们也将（在一定程度上）讨论给定事务可能违反哪些数据完整性约束。由 DBMS 负责为给定的事务验证这些涉及的数据完整性约束。如果由此产生的数据库状态违反了任何涉及的约束，DBMS 应拒绝该事务的预期更改。

和往常一样，你将在本章末尾找到“章节总结”，随后是“练习”部分。


