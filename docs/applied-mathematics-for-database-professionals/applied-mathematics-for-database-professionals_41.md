# 第 7 章 指定数据库设计

##### 表

###### 外部谓词

员工编号为`EMPNO`的员工，其姓名为`ENAME`，职位为`JOB`，出生于`BORN`，入职于`HIRED`，在`SGRADE`薪资等级内月薪为`MSAL`美元，分配至账户`USERNAME`，并为部门编号`DEPTNO`的部门工作。

**SREP**
员工编号为`EMPNO`的销售代表，其年度销售目标为`TARGET`美元，年度佣金为`COMM`美元。

**MEMP**
员工编号为`EMPNO`的员工，由员工编号为`MGR`的员工管理。

**TERM**
员工编号为`EMPNO`的员工，已于日期`LEFT`离职或被解雇，原因为`COMMENTS`。

DEPT
部门编号为`DEPTNO`的部门，其名称为`DNAME`，位于`LOC`，由员工编号为`MGR`的员工管理。

GRD
ID 为`GRADE`的薪资等级，其月薪下限为`LLIMIT`美元，月薪上限为`ULIMIT`美元，年度最高奖金为`BONUS`美元。

CRS
代码为`CODE`的课程，其描述为`DESCR`，属于课程类别`CAT`，持续时间为`DUR`天。

OFFR
课程代码为`COURSE`、开课日期为`STARTS`的课程开设，其状态为`STATUS`，最大容纳学员数为`MAXCAP`，开设地点为`LOC`，且（除非`TRAINER`等于-1）该开设分配有员工编号为`TRAINER`的员工作为培训师。

REG
员工编号为`STUD`的员工，已注册课程代码为`COURSE`、开课日期为`STARTS`的课程，且（除非`EVAL`等于-1）已为该课程给出评分为`EVAL`。

HIST
在日期`UNTIL`，对于员工编号为`EMPNO`的员工，其部门或月薪（或两者）发生了变更。在日期`UNTIL`之前，该员工的部门为`DEPTNO`，月薪为`MSAL`。

注：你发现这里的两处“取巧”了吗？显然，这里存在两种类型的课程开设：分配了培训师的开设和未分配培训师的开设。对于注册记录也可以做类似评论；有些包含课程开设的评分，有些则没有。在设计合理的数据库中，你应该将开设和注册表结构各自分解为两个表结构。

这些外部谓词让你对数据库骨架引入的所有相关表结构及其属性的含义有了一个非正式的初步了解。随着我们在后续章节中完成数据库 universe 定义的所有正式阶段，这个示例数据库设计的确切含义将变得清晰。

下一节将为骨架中引入的每个表结构提供特征描述。

##### 特征描述

正如你在第 4 章所见，特征描述定义了给定表结构的属性的值集。对于一个给定的表结构，特征描述是一个集合值函数，其定义域是该表结构的属性集合。对于每个属性，特征描述给出该属性的属性值集。特征描述构成了下一节构建元组 universe 的基础。然后你会注意到，这里定义这些特征描述的方式非常方便。看一下代码清单 7-2。

它定义了`EMP`表的特征描述。

注：几点说明：
- 在定义`EMP`表的属性值集时，我们使用了表 2-4 中引入的集合简写名称。
- 我们使用 `chr_<表结构名称>` 作为表结构特征描述的命名约定。
- 在 `chr_EMP` 的定义中（以及其他一些地方），你会看到一个名为 `upper` 的函数。该函数接受一个区分大小写的字符串，并返回该字符串的大写版本。

##### 元组全域

###### 代码清单 7-2. 特征化 chr_EMP

`chr_EMP` :=
{ ( `EMPNO`; `[1000..9999]` )
, ( `ENAME`; `varchar(9)` )
, ( `JOB`; /* 允许五种 JOB 值 */
{'PRESIDENT','MANAGER','SALESREP',
'TRAINER','ADMIN'} )
, ( `BORN`; `date` )
, ( `HIRED`; `date` )
, ( `SGRADE`; `[1..99]` )
, ( `MSAL`; { n | n∈`number(7,2)` ∧ n > 0 } )
, ( `USERNAME`; /* 用户名始终为大写 */
{ s | s∈`varchar(15)` ∧
`upper(USERNAME)` = `USERNAME` } )
, ( `DEPTNO`; `[1..99]` )
}

对于表结构 `EMP` 的每个属性，函数 `chr_EMP` 都会生成该属性的属性值集。现在你可以编写诸如 `chr_EMP(EMPNO)` 的表达式，它代表 `EMP` 表结构的 `EMPNO` 属性的属性值集。该表达式表示集合 `[1000..9999]`。

特征化 `chr_EMP` 的定义告诉我们以下信息：
- `EMPNO` 值是介于 1000 到 9999 之间的正整数。
- `ENAME` 值是最多包含九个字符的可变长度字符串。
- `JOB` 值仅限于以下五个值：'PRESIDENT', 'MANAGER', 'SALESREP', 'TRAINER','ADMIN'。
- `BORN` 和 `HIRED` 值是日期值。
- `SGRADE` 值是 1 到 99 之间的正整数。
- `MSAL` 值是精度为七、小数位数为二的正数。
- `USERNAME` 值是最多 15 个字符的大写可变长度字符串。
- `DEPTNO` 值是 1 到 99 之间的正整数。

在我们数据库设计定义的剩余部分中，有四个集合会频繁出现：员工编号、部门编号、薪资相关金额和课程代码。我们在这里为它们定义简短名称（便于文中引用的符号），并在后续的特征化定义中使用它们。

`EMPNO_TYP` := { n | n∈`number(4,0)` ∧ n > 999 }
`DEPTNO_TYP` := { n | n∈`number(2,0)` ∧ n > 0 }
`SALARY_TYP` := { n | n∈`number(7,2)` ∧ n > 0 }
`CRSCODE_TYP` := { s | s∈`varchar(6)` ∧ s = `upper(s)` }

代码清单 7-3 至 7-11 介绍了剩余表结构的特征化。在查看这些特征化时，你可能想重新审视表 7-1（外部谓词）。嵌入的注释在认为必要时澄清了属性约束。

###### 代码清单 7-3. 特征化 chr_SREP

`chr_SREP` :=
{ ( `EMPNO`; `EMPNO_TYP` )
/* 销售代表的目标是五位数数字 */
, ( `TARGET`; `[10000..99999]` )
, ( `COMM`; `SALARY_TYP` )
}

###### 代码清单 7-4. 特征化 chr_MEMP

`chr_MEMP` :=
{ ( `EMPNO`; `EMPNO_TYP` )
, ( `MGR`; `EMPNO_TYP` )
}

###### 代码清单 7-5. 特征化 chr_TERM

`chr_TERM` :=
{ ( `EMPNO`; `EMPNO_TYP` )
, ( `LEFT`; `date` )
, ( `COMMENTS`; `varchar(60)` )
}

###### 代码清单 7-6. 特征化 chr_DEPT

`chr_DEPT` :=
{ ( `DEPTNO`; `DEPTNO_TYP` )
, ( `DNAME`; { s | s∈`varchar(12)` ∧ `upper(DNAME)` = `DNAME` } )
, ( `LOC`; { s | s∈`varchar(14)` ∧ `upper(LOC)` = `LOC` } )
, ( `MGR`; `EMPNO_TYP` )
}

###### 代码清单 7-7. 特征化 chr_GRD

`chr_GRD` :=
{ ( `GRADE`; { n | n∈`number(2,0)` ∧ n > 0 } )
, ( `LLIMIT`; `SALARY_TYP` )
, ( `ULIMIT`; `SALARY_TYP` )
, ( `BONUS`; `SALARY_TYP` )
}

###### 代码清单 7-8. 特征化 chr_CRS

`chr_CRS` :=
{ ( `CODE`; `CRSCODE_TYP` )
, ( `DESCR`; `varchar(40)` )
/* 课程类别值：设计、生成、构建 */
, ( `CAT`; {'DSG','GEN','BLD'} )
/* 课程时长必须在 1 到 15 天之间 */
, ( `DUR`; `[1..15]` )
}

###### 代码清单 7-9. 特征化 chr_OFFR

`chr_OFFR` :=
{ ( `COURSE`; `CRSCODE_TYP` )
, ( `STARTS`; `date` )
/* 允许三种状态值：已计划、已确认、已取消 */
, ( `STATUS`; {'SCHD','CONF','CANC'} )
/* 最大课程提供容量；最小值 = 6 */
, ( `MAXCAP`; `[6..100]` )
/* TRAINER = -1 表示“未分配培训师” */
, ( `TRAINER`; `EMPNO_TYP` ∪ { -1 } )
, ( `LOC`; `varchar(14)` )
}

###### 代码清单 7-10. 特征化 chr_REG

`chr_REG` :=
{ ( `STUD`; `EMPNO_TYP` )
, ( `COURSE`; `CRSCODE_TYP` )
, ( `STARTS`; `date` )
/* -1：评估为时过早（课程在未来） */
/* 0：参与者未评估 */
/* 1-5：常规评估值（1=差 到 5=优秀） */
, ( `EVAL`; `[-1..5]` )
}

###### 代码清单 7-11. 特征化 chr_HIST

`chr_HIST` :=
{ ( `EMPNO`; `EMPNO_TYP` )
, ( `UNTIL`; `date` )
, ( `DEPTNO`; `DEPTNO_TYP` )
, ( `MSAL`; `SALARY_TYP` )
}

请注意，在代码清单 7-9 中，属性 `TRAINER` 的属性值集除了包含有效的员工编号外，还包含一个特殊值 -1。该值代表尚未分配培训师的事实。在我们正式的数据库设计规范方法中，不存在 SQL 数据库管理系统中常被（误）用作表示缺失值的 `NULL` 这种“值”。元组内部没有缺失值；它们总是为每个属性关联一个值。特征化指定了可以从中选择这些值的属性值集。因此，要表示一个“缺失的培训师”值，你必须在相应的属性值集中显式地为此事实包含一个值。代码清单 7-10 中 `EVAL` 属性的属性值集也指定了类似的内容。

**注意：** 附录 F 将专门处理 `NULL` 现象。第 11 章在解决数据库设计实现问题并提供指导原则时，将重新讨论这些 -1 值。

我们的数据库设计规范始于一个骨架定义以及骨架引入的表结构的外部谓词。在本节中，你了解了示例数据库设计的特征化。通过属性值集，你正在逐步深入地理解这个数据库设计的含义。

下一节将把这一理解推进到下一层：元组全域。

##### 元组全域

元组全域是一个（非空的）元组集合。这是一个非常特殊的元组集合；该集合旨在仅包含对给定表结构可接受的元组。你现在知道元组是用函数表示的。例如，这里有一个示例函数 `tdept1`，它代表 `DEPT` 表结构的一个可能元组：

`tdept1` := `{(DEPTNO;10), (DNAME;'ACCOUNTING'), (LOC;'DALLAS'), (MGR;1240)}`

如你所见，`tdept1` 的定义域代表了数据库骨架 `DB_S` 引入的表结构 `DEPT` 的属性集合。

`dom(tdept1)` = `{DEPTNO, DNAME, LOC, MGR}` = `DB_S(DEPT)`

并且，对于每个属性，`tdept1` 都从 `DEPT` 表结构的特征化引入的相应属性值集中产生一个值：
- `tdept1(DEPTNO)` = 10，它是 `chr_DEPT(DEPTNO)` 的一个元素
- `tdept1(DNAME)` = 'ACCOUNTING'，它是 `chr_DEPT(DNAME)` 的一个元素
- `tdept1(LOCATION)` = 'DALLAS'，它是 `chr_DEPT(LOCATION)` 的一个元素
- `tdept1(MGR)` = 1240，它是 `chr_DEPT(MGR)` 的一个元素

这是 `DEPT` 表结构的另一个可能元组：`tdept2` := `{(DEPTNO;20), (DNAME;'SALES'), (LOC;'HOUSTON'), (MGR;1755)}`

现在考虑集合 `{tdept1, tdept2}`。这是一个包含两个元组的集合。理论上它可能代表 `DEPT` 表结构的元组全域。然而，这是一个相当小的元组全域；它不太可能代表 `DEPT` 表结构的元组全域。对于给定表结构的元组全域应该包含我们为该表结构允许（接受）的*每一个*元组。

**注意：** 元组 `tdept1` 和 `tdept2` 是共享相同定义域的函数。这是对元组全域的要求；元组全域中的所有元组都共享相同的定义域，该定义域又等于给定表结构的标题。


您已经了解了如何利用表结构的表征来生成一个包含该结构所有可能元组的集合（参见第 5 章的“表构建”部分）。如果您将广义积应用于一个表征，最终会得到一个元组集合。这个集合不仅仅是普通的元组集合，它正是基于该表征所定义的属性值集构成的*所有可能*元组的集合。

让我们再用一个简单的例子来说明。假设您正在设计一个名为 `RESULT` 的表结构；它存储了属于特定人群的学生所修课程的平均分数。以下是 `RESULT` 的外部谓词：“属于人群 `POPULATION` 的学生，针对课程 `COURSE`，其四舍五入后的平均分数为 `AVG_SCORE`。”

###### 代码清单 7-12. 表征 `chr_RESULT`

`chr_RESULT` :=
{ ( `POPULATION`; {'DP','NON-DP'} )
/* DP = 数据库专业人士, NON-DP = 非数据库专业人士 */
, ( `COURSE`; {'集合论','逻辑学'} )
, ( `AVG_SCORE`; {'A','B','C','D','E','F'} )
}

这三个属性值集代表了 `RESULT` 表结构的属性约束。如果您将广义积 `∏` 应用于 `chr_RESULT`，您将得到 `RESULT` 表结构的以下可能元组集合：

```
∏(chr_RESULT) =
{ { (POPULATION; 'DP'), (COURSE; '集合论'), (AVG_SCORE; 'A') }
, { (POPULATION; 'DP'), (COURSE; '集合论'), (AVG_SCORE; 'B') }
, { (POPULATION; 'DP'), (COURSE; '集合论'), (AVG_SCORE; 'C') }
, { (POPULATION; 'DP'), (COURSE; '集合论'), (AVG_SCORE; 'D') }
, { (POPULATION; 'DP'), (COURSE; '集合论'), (AVG_SCORE; 'E') }
, { (POPULATION; 'DP'), (COURSE; '集合论'), (AVG_SCORE; 'F') }
, { (POPULATION; 'DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'A') }
, { (POPULATION; 'DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'B') }
, { (POPULATION; 'DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'C') }
, { (POPULATION; 'DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'D') }
, { (POPULATION; 'DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'E') }
, { (POPULATION; 'DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'F') }
, { (POPULATION; 'NON-DP'), (COURSE; '集合论'), (AVG_SCORE; 'A') }
, { (POPULATION; 'NON-DP'), (COURSE; '集合论'), (AVG_SCORE; 'B') }
, { (POPULATION; 'NON-DP'), (COURSE; '集合论'), (AVG_SCORE; 'C') }
, { (POPULATION; 'NON-DP'), (COURSE; '集合论'), (AVG_SCORE; 'D') }
, { (POPULATION; 'NON-DP'), (COURSE; '集合论'), (AVG_SCORE; 'E') }
, { (POPULATION; 'NON-DP'), (COURSE; '集合论'), (AVG_SCORE; 'F') }
, { (POPULATION; 'NON-DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'A') }
, { (POPULATION; 'NON-DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'B') }
, { (POPULATION; 'NON-DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'C') }
, { (POPULATION; 'NON-DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'D') }
, { (POPULATION; 'NON-DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'E') }
, { (POPULATION; 'NON-DP'), (COURSE; '逻辑学'), (AVG_SCORE; 'F') }
}
```

在这个包含 24 个元组的集合中，之前定义的属性约束将成立。然而，该集合对于一个元组内部不同属性的属性值*组合*没有限制。通过指定*属性间*约束——或者更准确地说，*元组约束*——您可以将可能的元组集合限制为给定表的*可接受*元组集合。

假设您不允许数据库专业人士的平均分数为 D、E 或 F，也不允许非数据库专业人士的平均分数为 A 或 B（无论课程是什么）。您可以通过定义元组宇宙 `tup_RESULT` 来指定这一点；它形式化地指定了两个*元组谓词*：

`tup_RESULT` :=
{ r | r∈Π(`chr_RESULT`) ∧
/* ============================ */
/* RESULT 的元组约束 */
/* ============================ */
/* 数据库专业人士的平均分永远不会是 D、E 或 F */



##### 指定数据库设计

r(`POPULATION`)='DP' ⇒ r(`AVG_SCORE`)∉{'D','E','F'} ∧

/* 非数据库专业人员从不会获得平均成绩 A 或 B */

r(`POPULATION`)='NON-DP' ⇒ r(`AVG_SCORE`)∉{'A','B'}

}

由元组宇宙定义引入的元组谓词被称为元组约束。你也可以用枚举方式指定集合 `tup_RESULT`。

**注意** 原始的 24 个可能元组集合现在已缩减为 14 个可允许元组的集合。

有十个元组不满足在 `tup_RESULT` 中指定的元组约束。

```
{ { (POPULATION; 'DP'), (COURSE; 'set theory'), (AVG_SCORE; 'A') }
, { (POPULATION; 'DP'), (COURSE; 'set theory'), (AVG_SCORE; 'B') }
, { (POPULATION; 'DP'), (COURSE; 'set theory'), (AVG_SCORE; 'C') }
, { (POPULATION; 'DP'), (COURSE; 'logic'), (AVG_SCORE; 'A') }
, { (POPULATION; 'DP'), (COURSE; 'logic'), (AVG_SCORE; 'B') }
, { (POPULATION; 'DP'), (COURSE; 'logic'), (AVG_SCORE; 'C') }
, { (POPULATION; 'NON-DP'), (COURSE; 'set theory'), (AVG_SCORE; 'C') }
, { (POPULATION; 'NON-DP'), (COURSE; 'set theory'), (AVG_SCORE; 'D') }
, { (POPULATION; 'NON-DP'), (COURSE; 'set theory'), (AVG_SCORE; 'E') }
, { (POPULATION; 'NON-DP'), (COURSE; 'set theory'), (AVG_SCORE; 'F') }
, { (POPULATION; 'NON-DP'), (COURSE; 'logic'), (AVG_SCORE; 'C') }
, { (POPULATION; 'NON-DP'), (COURSE; 'logic'), (AVG_SCORE; 'D') }
, { (POPULATION; 'NON-DP'), (COURSE; 'logic'), (AVG_SCORE; 'E') }
, { (POPULATION; 'NON-DP'), (COURSE; 'logic'), (AVG_SCORE; 'F') }
}
```

请注意，使用谓词方法来指定集合的 `tup_RESULT` 的先前规范，远优于后者的枚举规范，因为它明确地向我们展示了元组约束是什么（而且它的定义也更短；通常情况下要短得多）。

现在让我们继续我们的示例数据库设计。看一下清单 7-13，它为示例数据库设计的 `EMP` 表结构定义了元组宇宙 `tup_EMP`。

###### 清单 7-13. 元组宇宙 `tup_EMP`

```
tup_EMP :=
{ e | e∈Π(chr_EMP) ∧
/* ========================= */
/* EMP 的元组约束 */
/* ========================= */
/* 我们只雇佣成年员工 */
e(BORN) + 18 ≤ e(HIRED) ∧
/* 总统的薪资高于 120K */
e(JOB) = 'PRESIDENT' ⇒ 12*e(MSAL) > 120000 ∧
/* 管理员薪资低于 5K */
e(JOB) = 'ADMIN' ⇒ e(MSAL) < 5000
}
```

**注意** 在这个定义中，我们假设已为日期类型的值定义了加法（参见表 2-4），使我们能够向这样的值添加年数。

你开始明白这是如何运作的了吗？元组宇宙 `tup_EMP` 是 `Π(chr_EMP)` 的一个子集。

所有不满足在 `tup_EMP` 定义中指定的元组约束（总共有三个）的元组都被排除在外。你可以使用第 1 章表 1-2 中引入的任何逻辑连接词，并结合有效的属性表达式，来形式化地指定元组约束。

请注意，这些形式化规范消除了所有歧义：

-   “成年”指的是 18 岁或以上。`≤` 符号意味着某人年满 18 岁的当天就可以被雇佣。
-   `120K` 和 `5K`（在注释中）的 “K” 代表整数 `1000`，而不是 `1024`。提到的薪资（用户非正式地提及，规范内形式化地提及）对于 `CLERK` 实际上是月薪，对于 `PRESIDENT` 则是年薪。这可能是现实世界中的习惯做法，在形式化规范中反映这一点或许是明智的。当然，你也可以这样指定涉及 `PRESIDENT` 的谓词：`e(JOB) = 'PRESIDENT' ⇒ e(MSAL) > 10000`。

清单 7-14 到 7-22 介绍了我们数据库设计中其他表结构的元组宇宙。你会发现其中嵌入了非正式注释以阐明元组约束。

请注意，仅为表结构 `GRD`、`CRS` 和 `OFFR` 引入了元组约束；其他表结构恰好没有元组约束。

###### 清单 7-14. 元组宇宙 `tup_SREP`

```
tup_SREP :=
{ s | s∈Π(chr_SREP) /* SREP 无元组约束 */ }
```

###### 清单 7-15. 元组宇宙 `tup_MEMP`

```
tup_MEMP :=
{ m | m∈Π(chr_MEMP) }
```

###### 清单 7-16. 元组宇宙 `tup_TERM`

```
tup_TERM :=
{ t | t∈Π(chr_TERM) }
```

###### 清单 7-17. 元组宇宙 `tup_DEPT`

```
tup_DEPT :=
{ d | d∈Π(chr_DEPT) }
```

###### 清单 7-18. 元组宇宙 `tup_GRD`

```
tup_GRD :=
{ g | g∈Π(chr_GRD) ∧
/* 薪资等级的“带宽”至少为 500 美元 */
g(LLIMIT) ≤ g(ULIMIT) - 500 ∧
/* 奖金必须低于下限 */
g(BONUS) < g(LLIMIT)
}
```

###### 清单 7-19. 元组宇宙 `tup_CRS`

```
tup_CRS :=
{ c | c∈Π(chr_CRS) ∧
/* 构建类课程从不超过 5 天 */
c(CAT) = 'BLD' ⇒ c(DUR) ≤ 5
}
```

###### 清单 7-20. 元组宇宙 `tup_OFFR`

```
tup_OFFR :=
{ o | o∈Π(chr_OFFR) ∧
/* 仅对某些 STATUS 值允许未分配的 TRAINER */
o(TRAINER) = -1 ⇒ o(STATUS)∈{'CANC','SCHD'}
}
```

###### 清单 7-21. 元组宇宙 `tup_REG`

```
tup_REG :=
{ r | r∈Π(chr_REG) }
```

###### 清单 7-22. 元组宇宙 `tup_HIST`

```
tup_HIST :=
{ h | h∈Π(chr_HIST) }
```

清单 7-20 定义了何时允许 `TRAINER` 属性使用特殊的 `-1` 值；已确认的课程 (`STATUS = 'CONF'`) 必须分配一个员工作为培训师。

这完成了我们示例数据库设计的元组宇宙层。通过指定元组约束，你对这个数据库设计的含义有了更深入的了解。下一节将继续构建数据库设计的规范，推进到表宇宙层。正如你将看到的，这涉及应用本书第一部分中介绍的更多集合论和逻辑概念。

##### 表宇宙

你可以使用元组宇宙来构建一个可允许表的集合（我们很快会演示这一点）。

这样的集合被称为表宇宙。表宇宙中的每个元素都是对应表结构的一个可允许表。

元组宇宙是一个元组集合，也可以被看作一张表。它是一个相当大的元组集合，因为它包含了使用特征化并考虑元组约束所能构建的每一个元组。我们之前提到过，元组宇宙可以被看作给定表结构的最大表。

元组宇宙的每个子集也是一张表。事实上，如果你构造一个包含元组宇宙的所有子集的集合，那么这个集合将包含许多表；给定表结构的所有可能的表都将包含在这个集合中。你还记得本书第一部分中如何构造给定集合的所有子集的集合吗？幂集算子正是做这个的。元组宇宙的幂集可以被看作给定表结构的所有可能表的集合。

**注意** 你可能需要回顾第 2 章中“幂集与划分”一节，并重温一下关于幂集算子的知识。

与定义元组宇宙的方式类似，你可以限制元组宇宙的幂集以获得可允许表的集合。你可以添加表谓词（约束元组的组合）来丢弃由幂集算子生成但不反映现实世界有效表示的那些可能的表。用于限制元组宇宙幂集的表谓词被称为表约束。



让我们用上一节介绍的 `RESULT` 表结构来阐明这一切。元组宇宙 `tup_RESULT` 的幂集产生了一个包含所有可能 `RESULT` 表的集合。这个集合里有大量的表。确切地说，因为 `tup_RESULT` 的基数是 14，所以恰好有 16384（2 的 14 次方）种可能的表。用枚举的方式列出它们实在太多了。图 7-2 展示了其中一张表（`tup_RESULT` 的一个任意选择的子集）。让我们将这个表命名为 `R1`。

###### 图 7-2. 一个可能的名为 R1 的 RESULT 表

表 `R1` 是 `tup_RESULT` 的一个子集。它包含了 11 个不同的元组。因为这些元组来源于元组宇宙，所以它们都是可接受的元组；它们满足元组约束，并且每个属性都持有一个可接受的值。

现在假设以下（非形式化的）数据完整性约束在 `RESULT` 表结构的表中起作用：

*   属性 `POPULATION` 和 `COURSE` 的组合在 `RESULT` 表中是唯一标识的（约束 `P1`）。
*   一个 `RESULT` 表要么是空的，要么恰好包含四个元组：每个 `POPULATION` 和 `COURSE` 的组合对应一个元组（约束 `P2`）。
*   逻辑课程的平均分 (`AVG_SCORE`) 总是高于集合论课程的平均分；分数 A 最高，F 最低（约束 `P3`）。
*   非数据库专业人员的平均分总是低于数据库专业人员（约束 `P4`）。

**■ 注意** 其中一些完整性约束是相当刻意设计的。它们在本例中的唯一目的，就是将 `RESULT` 表结构的表宇宙中的表数量减少到可以实际枚举所有可接受表的程度。

在代码清单 7-23 中，你可以看到这四个约束被形式化地指定为表谓词，使用了前面引入的名称 `P1` 到 `P4`。为了能够比较这些规范中的平均分，我们引入了一个函数 `f`，其定义如下：`f := { ('A';6), ('B';5), ('C';4), ('D';3), ('E';2), ('F';1) }`

这使得我们可以比较，例如，分数 B 和 E。因为 `f(B)=5` 且 `f(E)=2`（并且 5>2），我们可以说 B 是比 E 更高的分数。

###### 代码清单 7-23. 表谓词 P1, P2, P3 和 P4

`P1(T) := ( ∀r1,r2∈T: r1↓{POPULATION,COURSE} = r2↓{POPULATION,COURSE} ⇒ r1 = r2 )`
`P2(T) := ( #T = 0 ∨ #T = 4 )`
`P3(T) := ( ∀r1,r2∈T: ( r1(POPULATION) = r2(POPULATION) ∧ r1(COURSE) = 'logic' ∧ r2(COURSE) = 'set theory' ) ⇒ f(r1(AVG_SCORE)) > f(r2(AVG_SCORE)) )`
`P4(T) := ¬( ∃r1,r2∈T: r1(POPULATION) = 'NON-DP' ∧ r2(POPULATION) = 'DP' ∧ f(r1(AVG_SCORE)) ≥ f(r2(AVG_SCORE)) )`

表谓词 `P1` 是第 6 章“唯一标识谓词”一节中介绍的常见类型的数据完整性谓词之一。

表谓词 `P2` 相当简单；表的基数应该为零或四。这是另一种指定方式：`#T∈{0,4}`。

如你所见，表谓词 `P3` 指定了逻辑课程的平均分应该总是高于集合论课程的平均分，*在一个人群范围内*。

全称量化中的第一个合取项——`r1(POPULATION) = r2(POPULATION)`——指定了这一点。用户可以理所当然地认为，在*给定的人群内*，逻辑课程的平均分总是高于集合论课程的平均分，但在非形式化传达需求时，可能会忘记明确说明这一点（如前面非形式化规范中所做的那样）。

表谓词 `P4` 明确地指定了非形式化规范中提到的“较低平均分”是不考虑课程的；在存在量词内部没有合取项 `r1(COURSE) = r2(COURSE)`。



你可以使用图 7-2 中介绍的表 `R1` 来实例化谓词 `P1` 到 `P4`。请自行验证表 `R1` 违反了谓词 `P1`、`P2` 和 `P3`，但满足谓词 `P4`。

`P1(R1) = false`
`P2(R1) = false`
`P3(R1) = false`
`P4(R1) = true`

利用这些形式化的表谓词规范，你现在可以为 `RESULT` 表结构定义表宇宙。请看代码清单 7-24，它使用元组宇宙 `tup_RESULT` 和表谓词 `P1`、`P2`、`P3`、`P4` 形式化地定义了表宇宙 `tab_RESULT`。

###### 代码清单 7-24. 表宇宙 `tab_RESULT` 的规范

```
tab_RESULT :=
{ R | R∈℘(tup_RESULT) ∧ P1(R) ∧ P2(R) ∧ P3(R) ∧ P4(R)
}
```

`tab_RESULT` 包含了满足所有四个表谓词的 `tup_RESULT` 的每个子集；显然，表 `R1` 不是 `tab_RESULT` 的一个元素。表谓词限制了元组宇宙的幂集，因此被称为*表约束*。

**注意：** 如果一个唯一标识谓词构成了一个表约束（如上例中的 `P1`），那么唯一标识属性的集合通常被称为给定表结构的*键*。在这种情况下，`{POPULATION, COURSE}` 是 `RESULT` 表结构的一个键。

表约束 `P1`、`P2`、`P3` 和 `P4` 是精心设计的，它们显著地将幂集生成的原始 16384 个可能表的总数大幅降低；事实上，此表宇宙中仅剩 13 个允许的表。代码清单 7-25 展示了表宇宙 `tab_RESULT` 的枚举规范。

###### 代码清单 7-25. 表宇宙 `tab_RESULT` 的枚举规范

```
tab_RESULT :=
{ ∅
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'B') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'D') }
  , { (POPULATION;'NON-DP'), (COURSE;'logic'), (AVG_SCORE;'C') } }
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'B') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'E') }
  , { (POPULATION;'NON-DP'), (COURSE;'logic'), (AVG_SCORE;'C') } }
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'B') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'F') }
  , { (POPULATION;'NON-DP'), (COURSE;'logic'), (AVG_SCORE;'C') } }
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'B') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'E') }
  , { (POPULATION;'NON-DP'), (COURSE;'logic'), (AVG_SCORE;'D') } }
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'B') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'F') }
  , { (POPULATION;'NON-DP'), (COURSE;'logic'), (AVG_SCORE;'D') } }
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'B') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'F') }
  , { (POPULATION;'NON-DP'), (COURSE;'logic'), (AVG_SCORE;'E') } }
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'C') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'E') }
  , { (POPULATION;'NON-DP'), (COURSE;'logic'), (AVG_SCORE;'D') } }
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'C') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'F') }
  , { (POPULATION;'NON-DP'), (COURSE;'logic'), (AVG_SCORE;'D') } }
, { { (POPULATION;'DP'), (COURSE;'set theory'), (AVG_SCORE;'C') }
  , { (POPULATION;'DP'), (COURSE;'logic'), (AVG_SCORE;'A') }
  , { (POPULATION;'NON-DP'), (COURSE;'set theory'), (AVG_SCORE;'F') }
```



```
, { (POPULATION;'非直接路径'), (COURSE;'逻辑学'), (AVG_SCORE;'E') } }

, { { (POPULATION;'直接路径'), (COURSE;'集合论'), (AVG_SCORE;'C') }

, { (POPULATION;'直接路径'), (COURSE;'逻辑学'), (AVG_SCORE;'B') }

, { (POPULATION;'非直接路径'), (COURSE;'集合论'), (AVG_SCORE;'E') }

, { (POPULATION;'非直接路径'), (COURSE;'逻辑学'), (AVG_SCORE;'D') } }

, { { (POPULATION;'直接路径'), (COURSE;'集合论'), (AVG_SCORE;'C') }

, { (POPULATION;'直接路径'), (COURSE;'逻辑学'), (AVG_SCORE;'B') }

, { (POPULATION;'非直接路径'), (COURSE;'集合论'), (AVG_SCORE;'F') }

, { (POPULATION;'非直接路径'), (COURSE;'逻辑学'), (AVG_SCORE;'D') } }

, { { (POPULATION;'直接路径'), (COURSE;'集合论'), (AVG_SCORE;'C') }

, { (POPULATION;'直接路径'), (COURSE;'逻辑学'), (AVG_SCORE;'B') }

, { (POPULATION;'非直接路径'), (COURSE;'集合论'), (AVG_SCORE;'F') }

, { (POPULATION;'非直接路径'), (COURSE;'逻辑学'), (AVG_SCORE;'E') } }
```

7451CH07.qxd 5/15/07 9:43 AM Page 161

