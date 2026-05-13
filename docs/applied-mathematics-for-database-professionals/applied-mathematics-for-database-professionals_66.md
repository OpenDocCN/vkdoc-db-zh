# 第 7 章详解

第 7 章解释了以下内容：

-   一个`属性值集`作为对应属性的数据类型。
-   一个`元组全域`作为当前表的`元组变量`的数据类型。
-   一个`表全域`作为对应的`表变量`的数据类型。

你会注意到，属性值集总是首先通过从当前 SQL 数据库管理系统中可用的基础数据类型（即`NUMBER`、`VARCHAR`和`DATE`）中取值来定义，然后通过指定属性约束来缩小这些集合的范围。在此过程中，将使用 SQL `CHECK`子句实现的属性约束表达式变得明确。

在认为必要的地方，你会发现嵌入的注释（`/* ... */`）用于解释形式化定义。

所有的元组、表、数据库和动态约束都按顺序编号，以便于参考。

## 一些便利的集合

在我们的数据库定义中，有四个集合经常出现：员工编号、部门编号、与薪资相关的金额和课程代码。因此，我们在此定义它们，以便可以通过名称引用它们。你可以将它们视为用户定义的数据类型：

```
EMPNO_TYP = { n | n∈number(4,0) ∧ n > 999 }
DEPTNO_TYP = { n | n∈number(2,0) ∧ n > 0 }
SALARY_TYP = { n | n∈number(7,2) ∧ n > 0 }
CRSCODE_TYP = { s | s∈varchar(6) ∧ s = upper(s) }
```

## EMP 的表全域

**外部谓词：** 员工编号为`EMPNO`的员工，姓名为`ENAME`，职位为`JOB`，出生于`BORN`，受雇于`HIRED`，月薪为`MSAL`美元，属于`SGRADE`薪资等级，分配到的账户为`USERNAME`，并且在部门编号为`DEPTNO`的部门工作。

以下三个列表（A-2、A-3 和 A-4）分别展示了`EMP`的属性值集、元组全域和表全域。

**列表 A-2.** `特征化 chr_EMP`

```
chr_EMP :=
{ ( EMPNO; EMPNO_TYP )
, ( ENAME; varchar(9) )
, ( JOB; { s | s∈varchar(8) ∧
s∈{'PRESIDENT','MANAGER'
,'SALESREP','TRAINER','ADMIN'} )
, ( BORN; date )
, ( HIRED; date )
, ( SGRADE; { n | n∈number(2,0) ∧ n > 0 } )
, ( MSAL; SALARY_TYP )
, ( USERNAME; { s | s∈varchar(15) ∧
upper(USERNAME) = USERNAME } )
, ( DEPTNO; DEPTNO_TYP )
}
```

**列表 A-3.** `元组全域 tup_EMP`

```
tup_EMP :=
{ e | e∈Π(chr_EMP) ∧
/* 我们只雇佣成年员工 r1 */
e(BORN) + 18 ≤ e(HIRED)
∧
/* 总裁月薪超过 1 万 r2 */
e(JOB) = 'PRESIDENT' ⇒ e(MSAL) > 10000
∧
/* 管理员月薪低于 5 千 r3 */
e(JOB) = 'ADMIN' ⇒ e(MSAL) < 5000
}
```

**列表 A-4.** `表全域 tab_EMP`

```
tab_EMP :=
{ E | E∈℘(tup_EMP) ∧
/* EMPNO 唯一标识一个员工元组 r4 */
( ∀e1,e2∈E: e1(EMPNO) = e2(EMPNO) ⇒ e1 = e2 )
∧
/* USERNAME 唯一标识一个员工元组 r4 */
( ∀e1,e2∈E: e1(USERNAME) = e2(USERNAME) ⇒ e1 = e2 )
∧
/* 最多只允许一名总裁 r5 */
#{ e | e∈E ∧ e(JOB) = 'PRESIDENT' } ≤ 1
∧
/* 雇佣了总裁或经理的部门 r6 */
/* 也应至少雇佣一名管理员 */
( ∀d∈{ e1(DEPTNO) | e1∈E }:
( ∃e2∈E: e2(DEPTNO) = d ∧ e2(JOB) ∈ {'PRESIDENT','MANAGER'} )
⇒
( ∃e2∈E: e2(DEPTNO) = d ∧ e2(JOB) = 'ADMIN' )
)
}
```

## SREP 的表全域

**外部谓词：** 员工编号为`EMPNO`的销售代表，其年度销售目标为`TARGET`美元，年度佣金为`COMM`美元。

以下三个列表（A-5、A-6 和 A-7）分别展示了`SREP`的属性值集、元组全域和表全域。

**列表 A-5.** `特征化 chr_SREP`

```
chr_SREP :=
{ ( EMPNO; EMPNO_TYP )
, ( TARGET; { n | n∈number(5,0) ∧ n > 9999 } )
, ( COMM; SALARY_TYP )
}
```

**列表 A-6.** `元组全域 tup_SREP`

```
tup_SREP :=
{ s | s∈Π(chr_SREP) }
```

**列表 A-7.** `表全域 tab_SREP`

```
tab_SREP :=
{ S | S∈℘(tup_SREP) ∧
/* EMPNO 唯一标识一个元组 r7 */
( ∀s1,s2∈S: s1(EMPNO) = s2(EMPNO) ⇒ s1 = s2 )
}
```

## MEMP 的表全域

**外部谓词：** 员工编号为`EMPNO`的员工由员工编号为`MGR`的员工管理。

以下三个列表（A-8、A-9 和 A-10）分别展示了`MEMP`的属性值集、元组全域和表全域。

**列表 A-8.** `特征化 chr_MEMP`

```
chr_MEMP :=
{ ( EMPNO; EMPNO_TYP )
, ( MGR; EMPNO_TYP )
}
```

**列表 A-9.** `元组全域 tup_MEMP`

```
tup_MEMP :=
{ m | m∈Π(chr_MEMP) ∧
/* 你不能管理自己 r8 */
m(EMPNO) ≠ m(MGR)
}
```

**列表 A-10.** `表全域 tab_MEMP`

```
tab_MEMP :=
{ M | M∈℘(tup_MEMP) ∧
/* EMPNO 唯一标识一个元组 r9 */
( ∀m1,m2∈M: m1(EMPNO) = m2(EMPNO) ⇒ m1 = m2 )
}
```

## TERM 的表全域

**外部谓词：** 编号为`EMPNO`的员工在日期`LEFT`因`COMMENTS`所述原因辞职或被解雇。

以下三个列表（A-11、A-12 和 A-13）分别展示了`TERM`的属性值集、元组全域和表全域。

**列表 A-11.** `特征化 chr_TERM`

```
chr_TERM :=
{ ( EMPNO; EMPNO_TYP )
, ( LEFT; date )
, ( COMMENTS; varchar(60) )
}
```

**列表 A-12.** `元组全域 tup_TERM`

```
tup_TERM :=
{ t | t∈Π(chr_TERM) }
```

**列表 A-13.** `表全域 tab_TERM`

```
tab_TERM :=
{ T | T∈℘(tup_TERM) ∧
/* EMPNO 唯一标识一个元组 r10 */
( ∀t1,t2∈T: t1(EMPNO) = t2(EMPNO) ⇒ t1 = t2 )
}
```

## DEPT 的表全域

**外部谓词：** 部门编号为`DEPTNO`的部门，名称为`DNAME`，位于`LOC`，由员工编号为`MGR`的员工管理。

以下三个列表（A-14、A-15 和 A-16）分别展示了`DEPT`的属性值集、元组全域和表全域。

**列表 A-14.** `特征化 chr_DEPT`

```
chr_DEPT :=
{ ( DEPTNO; DEPTNO_TYP )
, ( DNAME; { s | s∈varchar(12) ∧ upper(DNAME) = DNAME } )
, ( LOC; { s | s∈varchar(14) ∧ upper(LOC) = LOC } )
, ( MGR; EMPNO_TYP )
}
```

**列表 A-15.** `元组全域 tup_DEPT`

```
tup_DEPT :=
{ d | d∈Π(chr_DEPT) }
```

**列表 A-16.** `表全域 tab_DEPT`

```
tab_DEPT :=
{ D | D∈℘(tab_DEPT) ∧
/* 部门编号唯一标识一个元组 r11 */
( ∀d1,d2∈D: d1(DEPTNO) = d2(DEPTNO) ⇒ d1 = d2 )
∧
/* 部门名称和位置唯一标识一个元组 r12 */
( ∀d1,d2∈D:
d1↓{DNAME,LOC} = d2↓{DNAME,LOC} ⇒ d1 = d2 )
∧
/* 你不能管理超过两个部门 r13 */
( ∀m∈{ d(MGR) | d∈D }: #{ d | d∈D ∧ d(MGR) = m } ≤ 2 )
}
```

## GRD 的表全域

**外部谓词：** ID 为`GRADE`的薪资等级，月薪下限为`LLIMIT`美元，月薪上限为`ULIMIT`美元，最高净月奖金为`BONUS`美元。

以下三个列表（A-17、A-18 和 A-19）分别展示了`GRD`的属性值集、元组全域和表全域。

**列表 A-17.** `特征化 chr_GRD`

```
chr_GRD :=
{ ( GRADE; { n | n∈number(2,0) ∧ n > 0 } )
, ( LLIMIT; SALARY_TYP )
, ( ULIMIT; SALARY_TYP )
, ( BONUS; SALARY_TYP )
}
```

**列表 A-18.** `元组全域 tup_GRD`

```
tup_GRD :=
{ g | g∈Π(chr_GRD) ∧
/* 薪资等级的带宽至少为 500 美元 r14 */
g(LLIMIT) ≤ g(ULIMIT) - 500
∧
/* 奖金必须低于下限 r15 */
g(BONUS) < g(LLIMIT)
}
```

**列表 A-19.** `表全域 tab_GRD`

```
tab_GRD :=
{ G | G∈℘(tup_GRD) ∧
/* 薪资等级代码唯一标识一个元组 r16 */
( ∀g1,g2∈G: g1(GRADE) = g2(GRADE) ⇒ g1 = g2 )
∧
/* 薪资等级下限唯一标识一个元组 r17 */
( ∀g1,g2∈G: g1(LLIMIT) = g2(LLIMIT) ⇒ g1 = g2 )
∧
/* 薪资等级上限唯一标识一个元组 r18 */
( ∀g1,g2∈G: g1(ULIMIT) = g2(ULIMIT) ⇒ g1 = g2 )
∧
/* 一个薪资等级最多与一个（较低）等级重叠 r20 */
( ∀g1∈G:
( ∃g2∈G: g2(LLIMIT) < g1(LLIMIT) )
⇒
#{ g3 | g3∈G ∧ g3(LLIMIT) < g1(LLIMIT) ∧
g3(ULIMIT) ≥ g1(LLIMIT) ∧
g3(ULIMIT) < g1(ULIMIT) } = 1
)
}
```

## CRS 的表全域

**外部谓词：** 代码为`CODE`的课程，描述为`DESCR`，属于课程类别`CAT`，持续时间为`DUR`天。

以下三个列表（A-20、A-21 和 A-22）分别展示了`CRS`的属性值集、元组全域和表全域。

**列表 A-20.** `特征化 chr_CRS`

```
chr_CRS :=
{ ( CODE; CRSCODE_TYP )
, ( DESCR; varchar(40) )
/* 课程类别值：设计、生成、构建 */
, ( CAT; { s | s∈varchar(3) ∧
s∈{'DSG','GEN','BLD'} } )
/* 课程持续时间必须在 1 到 15 天之间 */
, ( DUR; { n | n∈number(2,0) ∧ 1 ≤ n ≤ 15 } )
}
```

**列表 A-21.** `元组全域 tup_CRS`

```
tup_CRS :=
{ c | c∈Π(chr_CRS) ∧
/* 构建类课程从不超过 5 天 r21 */
c(CAT) = 'BLD' ⇒ t(DUR) ≤ 5
}
```

**列表 A-22.** `表全域 tab_CRS`

```
tab_CRS :=
{ C | C∈℘(tup_CRS) ∧
/* 课程代码唯一标识一个元组 r22 */
( ∀c1,c2∈C: c1(CODE) = c2(CODE) ⇒ c1 = c2 )
}
```

## OFFR 的表全域

**外部谓词：** 课程代码为`COURSE`的课程，开课日期为`STARTS`，状态为`STATUS`，最大容纳`MAXCAP`名学员，由员工编号为`TRAINER`的员工教授，授课地点为`LOC`。

以下三个列表（A-23、A-24 和 A-25）分别展示了`OFFR`的属性值集、元组全域和表全域。

**列表 A-23.** `特征化 chr_OFFR`

```
chr_OFFR :=
{ ( COURSE; CRSCODE_TYP )
, ( STARTS; date )
, ( STATUS; { s | s∈varchar(4) ∧
/* 允许三种状态值： */
s∈{'SCHD','CONF','CANC'} } )
/* 最大课程容量；最小值=6 */
, ( MAXCAP; { n | n∈number(2,0) ∧ n ≥ 6 } )
/* TRAINER = -1 表示“未分配培训师”（参见 r23） */
, ( TRAINER; EMPNO_TYP ∪ { -1 } )
, ( LOC; varchar(14) )
}
```

**列表 A-24.** `元组全域 tup_OFFR`

```
tup_OFFR :=
{ o | o∈Π(chr_OFFR) ∧
/* 未分配的 TRAINER 仅允许用于某些状态值 r23 */
o(TRAINER) = -1 ⇒ o(STATUS)∈{'CANC','SCHD'}
}
```

**列表 A-25.** `表全域 tab_OFFR`

```
tab_OFFR :=
{ O | O∈℘(tup_OFFR) ∧
/* 课程代码和开始日期唯一标识一个元组 r24 */
( ∀o1,o2∈O:
o1↓{COURSE,STARTS} = o2↓{COURSE,STARTS} ⇒ o1 = o2 )
∧
/* 开始日期和（已知的）培训师唯一标识一个元组 r25 */
( ∀o1,o2∈{ o | o∈O ∧ o(TRAINER) ≠ -1 }:
o1↓{STARTS,TRAINER} = o2↓{STARTS,TRAINER} ⇒ o1 = o2 )
}
```

## REG 的表全域

**外部谓词：** 员工编号为`STUD`的员工已注册参加课程代码为`COURSE`、开课日期为`STARTS`的课程，并且该课程的评分为`EVAL`。

以下三个列表（A-26、A-27 和 A-28）分别展示了`REG`的属性值集、元组全域和表全域。

**列表 A-26.** `特征化 chr_REG`

```
chr_REG :=
{ ( STUD; EMPNO_TYP )
, ( COURSE; CRSCODE_TYP )
, ( STARTS; date )
/* -1：为时过早，无法评估；0：尚未评估； */
/* 1-5：常规评估值（从 1=差到 5=优秀） */
, ( EVAL; { n | n∈number(1,0)
∧ -1 ≤ n ≤ 5 } )
}
```

**列表 A-27.** `元组全域 tup_REG`

```
tup_REG :=
{ r | r∈Π(chr_REG) }
```

**列表 A-28.** `表全域 tab_REG`

```
tab_REG :=
{ R | R∈℘(tup_REG) ∧
/* 参与者和开始日期(!)唯一标识一个元组 r26 */
( ∀r1,r2∈R:
r1↓{STUD,STARTS} = r2↓{STUD,STARTS} ⇒ r1 = r2 )
∧
/* 课程已被评估， */
/* 或评估该课程为时过早 r27 */
( ∀r1,r2∈R:
( r1↓{COURSE,STARTS} = r2↓{COURSE,STARTS} )
⇒
( ( r1(EVAL) = -1 ∧ r2(EVAL) = -1 ) ∨
( r1(EVAL) ≠ -1 ∧ r2(EVAL) ≠ -1 )
) )
}
```

## HIST 的表全域

**外部谓词：** 在日期`UNTIL`，对于员工编号为`EMPNO`的员工，其部门或月薪（或两者）发生了变更。在`UNTIL`日期之前，该员工的部门为`DEPTNO`，月薪为`MSAL`。

以下三个列表（A-29、A-30 和 A-31）分别展示了`HIST`的属性值集、元组全域和表全域。

**列表 A-29.** `特征化 chr_HIST`

```
chr_HIST :=
{( EMPNO; EMPNO_TYP )
,( UNTIL; date )
,( DEPTNO; DEPTNO_TYP )
,( MSAL; SALARY_TYP )
}
```

**列表 A-30.** `元组全域 tup_HIST`

```
tup_HIST :=
{ h | h∈Π(chr_HIST) }
```

**列表 A-31.** `表全域 tab_HIST`

```
tab_HIST :=
{ H | H∈℘(tup_HIST) ∧
/* 员工编号和结束日期唯一标识一个元组 r28 */
( ∀h1,h2∈H: h1↓{EMPNO,UNTIL} = h2↓{EMPNO,UNTIL} ⇒ h1 = h2 )
∧
/* 部门编号或月薪（或两者） r29 */
/* 必须在两个连续的历史记录之间发生变更 */
( ∀h1,h2∈H:
( h1(EMPNO) = h2(EMPNO) ∧
h1(UNTIL) < h2(UNTIL) ∧
¬ ( ∃h3∈H: h3(EMPNO) = h1(EMPNO) ∧
h3(UNTIL) > h1(UNTIL) ∧
h3(UNTIL) < h2(UNTIL) )
) ⇒
( h1(MSAL) ≠ h2(MSAL) ∨ h1(DEPTNO) ≠ h2(DEPTNO) )
)
}
```

## 数据库特征化 DBCH

数据库特征化`DBCH`将十个表全域（如本附录前一节所定义）附加到它们对应的表别名上；参见列表 A-32。这样，数据库特征化“重新审视”了数据库骨架，并提供了更多细节。

**列表 A-32.** `数据库特征化 DBCH`

```
DBCH :=
{ ( EMP; tab_EMP )
, ( SREP; tab_SREP )
, ( MEMP; tab_MEMP )
, ( TERM; tab_TERM )
, ( DEPT; tab_DEPT )
, ( GRD; tab_GRD )
, ( CRS; tab_CRS )
, ( OFFR; tab_OFFR )
, ( REG; tab_REG )
, ( HIST; tab_HIST )
}
```

## 数据库全域 DB_UEX

如列表 A-33 所示，数据库全域`DB_UEX`构建在数据库特征化`DBCH`之上。`DB_UEX`的规范包含了所有静态数据库约束。

**列表 A-33.** `数据库全域 DB_UEX`

```
DB_UEX :=
{ v | v∈Π(DBCH) ∧
/* ==================================================================== */
/* 子集要求开始 */
/* ==================================================================== */
/* 员工为已知部门工作 r30 */
{ e(DEPTNO) | e∈v(EMP) } ⊆ { d(DEPTNO) | d∈v(DEPT) }
∧
/* 部门经理是已知员工，不包括管理员和总裁 r31 */
{ d(MGR) | d∈v(DEPT) } ⊆
{ e(EMPNO) | e∈v(EMP) ∧ e(JOB) ∉ {'ADMIN','PRESIDENT'} }
∧
/* 员工只能向总裁或经理汇报 r32 */
{ m(MGR) | m∈v(MEMP) } ⊆
{ e(EMPNO) | e∈v(EMP) ∧ e(JOB) ∈ {'PRESIDENT','MANAGER' } }
∧
/* 终止记录对应于已知员工；并非所有人都已离职 r33 */
{ t(EMPNO) | t∈v(TERM) } ⊂ { e(EMPNO) | e∈v(EMP) }
∧
/* 员工拥有已知的薪资等级 r34 */
{ e(SGRADE) | e∈v(EMP) } ⊆ { g(GRADE) | g∈v(GRD) }
∧
/* 课程开设对应于已知课程 r35 */
{ o(COURSE) | o∈v(OFFR) } ⊆ { c(CODE) | c∈v(CRS) }
∧
/* 课程在我们设有部门的地点进行 r36 */
{ o(LOC) | o∈v(OFFR) } ⊆ { d(LOC) | d∈v(DEPT) }
∧
/* 课程的培训师是已知的培训师 r37 */
{ o(TRAINER) | o∈v(OFFR) ∧ o(TRAINER) ≠ -1 } ⊆
{ e(EMPNO) | e∈v(EMP) ∧ e(JOB) = 'TRAINER' }
∧
/* 课程注册对应于已知员工 r38 */
{ r(STUD) | r∈v(REG) } ⊆ { e(EMPNO) | e∈v(EMP) }
∧
/* 课程注册对应于已知的课程开设 r39 */
{ r↓{COURSE,STARTS} | r∈v(REG) } ⊆
{ o↓{COURSE,STARTS} | o∈v(OFFR) }
∧
/* 历史记录对应于已知员工 r40 */
{ h(EMPNO) | h∈v(HIST) } ⊆ { e(EMPNO) | e∈v(EMP) }
∧
/* 历史记录对应于已知部门 r41 */
{ h(DEPTNO) | h∈v(HIST) } ⊆ { d(DEPTNO) | d∈v(DEPT) }
∧
/* ==================================================================== */
/* 子集要求结束；特化规则开始 */
/* ==================================================================== */
/* 销售代表有目标和佣金 r42 */
{ e(EMPNO) | e∈v(EMP) ∧ e(JOB) = 'SALESREP' } =
{ s(EMPNO) | s∈v(SREP) }
∧
/* 除总裁外，所有人都是被管理的员工 r43 */
{ e(EMPNO) | e∈v(EMP) ∧ e(JOB) ≠ 'PRESIDENT' } =
{ m(EMPNO) | m∈v(MEMP) }
∧
/* ==================================================================== */
/* 特化结束；元组连接规则开始 */
/* ==================================================================== */
/* 月薪必须在分配的薪资等级范围内 r44 */
( ∀e∈v(EMP), g∈v(GRD):
e(SGRADE) = g(GRADE) ⇒ g(LLIMIT) ≤ e(MSAL) ≤ g(ULIMIT)
) ∧
/* 离职日期必须在雇佣日期之后 r45 */
( ∀e∈v(EMP), t∈v(TERM):
e(EMPNO) = t(EMPNO) ⇒ e(HIRED) < t(LEFT)
) ∧
/* 销售代表不能比其汇报对象的收入高 r46 */
( ∀s∈v(SREP), es∈v(EMP), em∈v(EMP), m∈v(MEMP):
( s(EMPNO)=es(EMPNO) ∧ es(EMPNO)=m(EMPNO) ∧ m(MGR) = em(EMPNO) )
⇒
( es(MSAL) + s(COMM)/12 < em(MSAL) )
) ∧
/* 非销售代表不能比其汇报对象的收入高 r47 */
( ∀e∈v(EMP), em∈v(EMP), m∈v(MEMP):
(e(EMPNO)=m(EMPNO) ∧ m(MGR) = em(EMPNO) ∧ e(JOB) ≠ 'SALESREP')
⇒
( e(MSAL) < em(MSAL) )
) ∧
/* 雇佣日期之前不允许有历史记录 r48 */
( ∀e∈v(EMP), h∈v(HIST):
e(EMPNO) = h(EMPNO) ⇒ e(HIRED) < h(UNTIL)
) ∧
/* 离职日期之后不允许有历史记录 r49 */
( ∀t∈v(TERM), h∈v(HIST):
t(EMPNO) = h(EMPNO) ⇒ t(LEFT) > h(UNTIL)
) ∧
/* 入职头四周内不能注册课程 r50 */
( ∀e∈v(EMP), r∈v(REG):
e(EMPNO) = r(STUD) ⇒ e(HIRED) + 28 ≤ r(STARTS)
) ∧
/* 不能注册在离职日期当天或之后开始的课程 r51 */
( ∀t∈v(TERM), r∈v(REG), c∈v(CRS):
( t(EMPNO) = r(STUD) ∧ r(COURSE) = c(CODE) )
⇒
( t(LEFT) ≥ r(STARTS) + c(DUR) )
) ∧
/* 不能注册时间重叠的课程 r52 */
( ∀e∈v(EMP), r1∈v(REG), r2∈v(REG), o1∈v(OFFR), o2∈v(OFFR), c1∈v(CRS), c2∈v(CRS):
( e(EMPNO) = r1(STUD) ∧
r1↓{COURSE,STARTS} = o1↓{COURSE,STARTS} ∧
o1(COURSE) = c1(CODE) ∧
e(EMPNO) = r2(STUD) ∧
r2↓{COURSE,STARTS} = o2↓{COURSE,STARTS} ∧
o2(COURSE) = c2(CODE)
) ⇒
( o1↓{COURSE,STARTS} = o2↓{COURSE,STARTS} ∨
o1(STARTS) ≥ o2(STARTS) + c2(DUR) ∨
o2(STARTS) ≥ o1(STARTS) + c1(DUR)
) ) ∧
/* 培训师不能在雇佣日期之前授课 r53 */
( ∀e∈v(EMP), o∈v(OFFR):
e(EMPNO) = o(TRAINER) ⇒ e(HIRED) ≤ o(STARTS)
) ∧
/* 培训师不能在离职日期当天或之后授课 r54 */
( ∀t∈v(TERM), o∈v(OFFR), c∈v(CRS):
( t(EMPNO) = o(TRAINER) ∧ o(COURSE) = c(CODE) )
⇒
( t(LEFT) ≥ o(STARTS) + c(DUR) )
) ∧
/* 培训师不能注册自己讲授的课程 r55 */
( ∀r∈v(REG), o∈v(OFFR):
r↓{COURSE,STARTS} = o↓{COURSE,STARTS} ⇒
r(STUD) ≠ o(TRAINER)
) ∧
/* 培训师不能同时教授不同的课程 r56 */
( ∀o1∈v(OFFR), o2∈v(OFFR), c1∈v(CRS), c2∈v(CRS):
( o1(TRAINER) = o2(TRAINER) ∧
o1(COURSE) = c1(CODE) ∧
o2(COURSE) = c2(CODE)
) ⇒
( o1↓{COURSE,STARTS} = o2↓{COURSE,STARTS} ∨
o1(STARTS) ≥ o2(STARTS) + c2(DUR) ∨
o2(STARTS) ≥ o1(STARTS) + c1(DUR)
) ) ∧
/* 员工不能注册与其作为培训师讲授的课程 r57 */
/* 时间重叠的课程 */
( ∀e∈v(EMP), r∈v(REG), o1∈v(OFFR), o2∈v(OFFR), c1∈v(CRS), c2∈v(CRS):
( e(EMPNO) = r(STUD) ∧
r↓{COURSE,STARTS} = o1↓{COURSE,STARTS} ∧
o1(COURSE) = c1(CODE) ∧
e(EMPNO) = o2(TRAINER) ∧
o2(COURSE) = c2(CODE)
) ⇒
( o1↓{COURSE,STARTS} = o2↓{COURSE,STARTS} ∨
o1(STARTS) ≥ o2(STARTS) + c2(DUR) ∨
o2(STARTS) ≥ o1(STARTS) + c1(DUR)
) ) ∧
/* ==================================================================== */
/* 元组连接规则结束；其他数据库规则开始 */
/* ==================================================================== */
/* 部门经理必须在其管理的部门工作 r58 */
( ∀d1∈v(DEPT): { e(DEPTNO)| e∈v(EMP) ∧ e(EMPNO)=d1(MGR)} ⊆
{ d2(DEPTNO)| d2∈v(DEPT) ∧ d2(MGR)=d1(MGR) } )
∧
/* 在职员工不能由已终止的员工管理 r59 */
{ t1(EMPNO) | t1∈v(TERM) } ∩
{ m(MGR) | m∈v(MEMP) ∧
¬ ( ∃t2∈v(TERM): t2(EMPNO) = m(EMPNO) ) } = ∅
∧
/* 部门不能由已终止的员工管理 r60 */
{ t(EMPNO) | t∈v(TERM) } ∩ { d(MGR) | d∈v(DEPT) } = ∅
∧
/* 培训师教授的课程中，至少一半（按持续时间计） r61 */
/* 必须在“基地”进行 */
( ∀e∈{ o(TRAINER) | o∈v(OFFR) ∧ o(STATUS) ≠ 'CANC' }: 
( ∑ t∈{ o2.DUR | d2∈v(DEPT), e2∈v(EMP), o2∈v(OFFR), c2∈v(CRS) |
e2(EMPNO) = e ∧
e2(EMPNO) = o2(TRAINER) ∧
e2(DEPTNO) = d2(DEPTNO) ∧
o2(COURSE) = c2(CODE) ∧
o2(STATUS) ≠ 'CANC' ∧
c2(LOC) = d2(LOC)
} : t
) ≥
( ∑ t∈{ o3.DUR | d3∈v(DEPT), e3∈v(EMP), o3∈v(OFFR), c3∈v(CRS) |
e3(EMPNO) = e ∧
e3(EMPNO) = o3(TRAINER) ∧
e3(DEPTNO) = d3(DEPTNO) ∧
o3(COURSE) = c3(CODE) ∧
o3(STATUS) ≠ 'CANC' ∧
c3(LOC) ≠ d3(LOC)
} : t
) ) ∧
/* 注册人数超过 6 人的课程必须状态为已确认 r62 */
( ∀o∈v(OFFR):
#{ r | r∈v(REG) ∧
r↓{COURSE,STARTS} = o↓{COURSE,STARTS} } ≥ 6
⇒
o(STATUS) = 'CONF'
) ∧
/* 注册人数不能超过课程的最大容量 r63 */
( ∀o∈v(OFFR):
#{ r | r∈v(REG) ∧
r↓{COURSE,STARTS} = o↓{COURSE,STARTS} } ≤ o(MAXCAP)
) ∧
/* 已取消的课程不能有注册 r64 */
( ∀o∈v(OFFR): o(STATUS) = 'CANC'
⇒
¬( ∃r∈v(REG): r↓{COURSE,STARTS} = o↓{COURSE,STARTS} )
) ∧
/* 你只能在满足以下条件时讲授某门课程： r65 */
/* 1. 你已受雇至少一年，或 */
/* 2. 你已参加过该课程，并且该课程的培训师作为参与者参加你的首次授课 */
( ∀o1∈v(OFFR):
/* 如果这是该培训师第一次讲授此课程 ... */
( ¬∃o2∈v(OFFR):
o1↓{COURSE,TRAINER} = o2↓{COURSE,TRAINER} ∧
o2(STARTS) < o1(STARTS)
) ⇒
( /* 那么教室里应该有一位学员 ... */
( ∃r1∈v(REG):
r1↓{COURSE,STARTS} = o1↓{COURSE,STARTS} ∧
/* 他/她在更早的日期讲授过该课程 ... */
( ∃o3∈v(OFFR):
o3(TRAINER) = r1(STUD) ∧
o3(COURSE) = o1(COURSE) ∧
o3(STARTS) < o1(STARTS) ∧
/* 而*该*课程由当前的培训师参加过 */
( ∃r2∈v(REG):
o3↓{COURSE,STARTS} = r2↓{COURSE,STARTS} ∧
r2(STUD) = o1(TRAINER)
) ) ) ∨
/* 或者，该培训师已受雇至少一年 */
( ↵{ e(HIRED) | e∈v(EMP) ∧ e(EMPNO) = o1(TRAINER) } <
o1(STARTS) - 365
) ) )
/* ==================================================================== */
/* 其他数据库规则结束 */
/* ==================================================================== */
}
```

## 状态转换全域 TX_UEX

`状态转换全域` `TX_UEX` 的定义是通过首先将数据库全域 `DB_UEX`（如本附录“数据库全域 `DB_UEX`”一节所定义）与自身进行笛卡尔积运算。你可以将这个笛卡尔积的结果视为所有可能事务的集合。在这个状态转换全域 `TX_UEX` 中，每个事务被描绘为一个有序对 `(b;e)`，其中 `b` 代表事务开始时的数据库状态，`e` 代表事务结束时的数据库状态。如列表 A-34 所示，`TX_UEX` 的定义通过指定*动态*（也称为*状态转换*或*事务*）约束，将这个集合限制为仅包含*有效*事务。

**注意：** 在本节中，我们使用 `sysdate` 表示移动的“现在”时间点。

**列表 A-34.** `事务全域 TX_UEX`

```
TX_UEX :=
{ (b;e) | b∈DB_UEX ∧ e∈DB_UEX ∧
/* 月薪只能增加 r66 */
( ∀e1∈b(EMP), e2∈e(EMP):
e1(EMPNO) = e2(EMPNO) ⇒ e1(MSAL) ≤ e2(MSAL)
) ∧
/* 新课程必须以 SCHED 状态开始 r67 */
( ∀o1∈e(OFFR)⇓{COURSE,STARTS} − b(OFFR)⇓{COURSE,STARTS}:
↵{ o2(STATUS) | o2∈e(OFFR) ∧ o2↓{COURSE,STARTS} = o1↓{COURSE,STARTS} }
= 'SCHD'
) ∧
/* 有效的课程状态转换是： r68 */
/* SCHED -> CONF, SCHED -> CANC, CONF -> CANC */
( ∀o1∈b(OFFR), o2∈e(OFFR):
o1↓{COURSE,STARTS} = o2↓{COURSE,STARTS}
⇒
( o1(STATUS) = o2(STATUS) ∨
( o1(STATUS) = 'SCHD' ∧ o2(STATUS) = 'CONF' ) ∨
( o1(STATUS) = 'SCHD' ∧ o2(STATUS) = 'CANC' ) ∨
( o1(STATUS) = 'CONF' ∧ o2(STATUS) = 'CANC' )
) ) ∧
/* 不允许更新历史记录 r69 */
( ∀h1∈b(HIST), h2∈e(HIST):
h1↓{EMPNO,UNTIL} = h2↓{EMPNO,UNTIL}
⇒
( h1(DEPTNO) = h2(DEPTNO) ∧ h1(MSAL) = h2(MSAL) )
) ∧
/* 新的历史记录必须准确反映员工更新 r70 */
( ∀h1∈e(HIST)⇓{EMPNO,UNTIL} − b(HIST)⇓{EMPNO,UNTIL}:
( ∃h2∈e(HIST):
h2↓{EMPNO,UNTIL} = h1↓{EMPNO,UNTIL} ∧
h2(UNTIL) = sysdate ∧
( ∃e1∈b(EMP), e2∈e(EMP):
e1↓{EMPNO,MSAL,DEPTNO} = h2↓{EMPNO,MSAL,DEPTNO} ∧
e2(EMPNO) = h2(EMPNO) ∧
( e2(MSAL) ≠ e1(MSAL) ∨ e2(DEPTNO) ≠ e1(DEPTNO) )
) ) ) ∧
/* 新的注册元组必须以 EVAL = -1 开始 r71 */
( ∀r1∈e(REG)⇓{STUD,STARTS} − b(REG)⇓{STUD,STARTS}:
( ∃r2∈e(REG):
r2↓{STUD,STARTS} = r1↓{STUD,STARTS} ∧ r2(EVAL) = -1
) ) ∧
/* 评估的转换必须有效 r72 */
/* 且不能发生在课程开始日期之前 */
( ∀r1∈b(REG), r2∈e(REG):
r1↓{STUD,STARTS} = r2↓{STUD,STARTS} ∧ r1(EVAL) ≠ r2(EVAL)
⇒
( ( r1(EVAL) = -1 ∧ r2(EVAL) = 0 ∧ r2(STARTS) ≥ sysdate ) ∨
( r1(EVAL) = 0 ∧ r2(EVAL) ∈ {1,2,3,4,5} )
) )
}
```

