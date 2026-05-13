# 第 7 章 指定数据库设计

###### 代码清单 7-29. *表全域 `tab_TERM`*

```
tab_TERM :=

{ T | T∈℘(tup_TERM) ∧

/* EMPNO 唯一地标识一个元组 */

( ∀t1,t2∈T: t1(EMPNO) = t2(EMPNO) ⇒ t1 = t2 )

}
```

`DEPT` 的表全域（如代码清单 7-30 所示）引入了两个唯一标识约束。除了 `{DEPTNO}`，属性集 `{DNAME,LOC}` 在 `DEPT` 表中也是唯一标识符。

###### 代码清单 7-30. *表全域 `tab_DEPT`*

```
tab_DEPT :=

{ D | D∈℘(tab_DEPT) ∧

/* 部门编号唯一地标识一个元组 */

( ∀d1,d2∈D: d1(DEPTNO) = d2(DEPTNO) ⇒ d1 = d2 )

∧

/* 部门名称和位置唯一地标识一个元组 */

( ∀d1,d2∈D:

d1↓{DNAME,LOC} = d2↓{DNAME,LOC} ⇒ d1 = d2 )

∧

/* 你不能管理超过两个部门 */

( ∀m∈D⇓{MGR}: #{ d | d∈D ∧ d(MGR) = m(MGR) } ≤ 2 )

}
```


## GRD 表的表域

`Listing 7-31`展示了`GRD`表结构的表域定义。

**Listing 7-31.** `表域 tab_GRD`

```
tab_GRD :=

{ G | G∈℘(tup_GRD) ∧

/* 工资等级代码唯一标识一个元组 */

( ∀g1,g2∈G: g1(GRADE) = g2(GRADE) ⇒ g1 = g2 )

∧

/* 工资等级下限唯一标识一个元组 */

( ∀g1,g2∈G: g1(LLIMIT) = g2(LLIMIT) ⇒ g1 = g2 )

∧

/* 工资等级上限唯一标识一个元组 */

( ∀g1,g2∈G: g1(ULIMIT) = g2(ULIMIT) ⇒ g1 = g2 )

∧

/* 一个工资等级最多与一个（较低的）等级重叠 */

( ∀g1∈G:

( ∃g2∈G: g2(LLIMIT) < g1(LLIMIT) )

⇒

#{ g3 | g3∈G ∧ g3(LLIMIT) < g1(LLIMIT) ∧

g3(ULIMIT) ≥ g1(LLIMIT) ∧

g3(ULIMIT) < g1(ULIMIT) } = 1

)
}
```

如你所见，这个表域为`GRD`指定了三个唯一标识约束。第四个表约束规定了两个连续工资等级的`ULIMIT`工资和`LLIMIT`工资之间不能存在工资缺口。这是通过要求对于每一个元组（例如`g1`），如果存在一个对应较低工资等级的元组（例如`g2`），那么必须存在一个元组（例如`g3`），它对应一个较低的工资等级并且与元组`g1`的工资范围重叠。实际上，应该恰好存在一个这样的`g3`，从而确保每个工资最多被两个工资等级覆盖。形式化规范精确地定义了其含义，没有留下任何歧义。

`Listing 7-32`和`Listing 7-33`定义了`CRS`和`OFFR`表结构的表域。如你所见，其中只涉及唯一标识约束。

**Listing 7-32.** `表域 tab_CRS`

```
tab_CRS :=

{ C | C∈℘(tup_CRS) ∧

/* 课程代码唯一标识一个元组 */

( ∀c1,c2∈C: c1(CODE) = c2(CODE) ⇒ c1 = c2 )

}
```

**Listing 7-33.** `表域 tab_OFFR`

```
tab_OFFR :=

{ O | O∈℘(tup_OFFR) ∧

/* 课程代码和开始日期唯一标识一个元组 */

( ∀o1,o2∈O:

o1↓{COURSE,STARTS} = o2↓{COURSE,STARTS} ⇒ o1 = o2 )

∧

/* 开始日期和（已知的）培训师唯一标识一个元组 */

( ∀o1,o2∈{ o | o∈O ∧ o(TRAINER) ≠ -1 }:

o1↓{STARTS,TRAINER} = o2↓{STARTS,TRAINER} ⇒ o1 = o2 )

}
```

第二个表约束在某种程度上是唯一标识约束的一个特例。集合`{STARTS,TRAINER}`在*OFFR 表的一个子集*内是唯一标识符。此约束规定一名培训师在一个开始日期不能讲授多门课程。当然，可能存在多门在同一天开始但尚未分配培训师的课程；因此，在全称量化中排除了那些未分配培训师的课程。

> **注意** 这里显然存在一个更广泛的限制，它涉及一门课程的完整持续时间：培训师在已分配课程期间不能再讲授另一门课程。这涉及相应课程的持续时间（`CRS`表结构中的`DUR`属性）。因此，这是一个多表约束，将在下一阶段（数据库域）中指定。

`Listing 7-34`定义了`REG`表结构的表域`tab_REG`。

**Listing 7-34.** `表域 tab_REG`

```
tab_REG :=

{ R | R∈℘(tup_REG) ∧

/* 参与者和开始日期唯一标识一个元组 */

( ∀r1,r2∈R:

r1↓{STARTS,STUD} = r2↓{STARTS,STUD} ⇒ r1 = r2 )

∧

/* 课程已被所有参与者评价，或者现在评价该课程为时过早 */

( ∀r1,r2∈R:
```


##### 数据库设计的形式化规范（续）

( r1↓{COURSE,STARTS} = r2↓{COURSE,STARTS} )

⇒

( ( r1(EVAL) = -1 ∧ r2(EVAL) = -1 ) ∨

( r1(EVAL) ≠ -1 ∧ r2(EVAL) ≠ -1 )

) )

}

你注意到第一个表约束并没有声明 {`COURSE`,`STARTS`,`STUD`} 是 `REG` 表的唯一标识吗？考虑到 `REG` 是 `CRS` 的“子表”，通常预期 `COURSE` 会是标识属性集的一部分。然而，在现实世界中，我们不允许学生注册两个开课日期相同的课程提供，无论这些提供涉及什么课程。因此，在 `REG` 表中，缩减的属性集 {`STARTS`,`STUD`} 是唯一标识。这里同样有一个更广泛的限制，涉及课程持续时间，这将在稍后的数据库宇宙定义中予以说明。

第二个表约束指出，在一个课程提供的注册中，所有的评估值要么（仍然）都保持为 -1，要么所有值都不同于 -1。请注意（引用嵌入的注释）“由所有参与者评估”包括了特殊值 0。这里的想法是，在课程提供接近尾声时，培训师会在一个单一的事务中，将所有评估值从 -1（为时过早，无法评估）更改为 0（未评估）。

然后，学生们在各自的事务中单独评估该提供。正如你将在下一章看到的，你可以通过一个动态（状态转换）约束来形式化地指定这种预期行为。

清单 7-35 定义了关于 `HIST` 表结构的最后一个表宇宙（`tab_HIST`）。

```
( r1↓{COURSE,STARTS} = r2↓{COURSE,STARTS} )
⇒
( ( r1(EVAL) = -1 ∧ r2(EVAL) = -1 )
  ∨
  ( r1(EVAL) ≠ -1 ∧ r2(EVAL) ≠ -1 )
) )
```

**清单 7-35.** 表宇宙 `tab_HIST`

```
tab_HIST :=
{ H | H∈℘(tup_HIST) ∧
  /* 员工编号和历史结束日期唯一标识一个元组 */
  ( ∀h1,h2∈H: h1↓{EMPNO,UNTIL} = h2↓{EMPNO,UNTIL} ⇒ h1 = h2 )
  ∧
  /* 部门编号或月薪（或两者）必须在两个连续的历史记录之间发生变化 */
  ( ∀h1,h2∈H:
    ( h1(EMPNO) = h2(EMPNO) ∧
      h1(UNTIL) < h2(UNTIL) ∧
      ¬ ( ∃h3∈T: h3(EMPNO) = h1(EMPNO) ∧
          h3(UNTIL) > h1(UNTIL) ∧
          h3(UNTIL) < h2(UNTIL) )
    ) ⇒
    ( h1(MSAL) ≠ h2(MSAL) ∨ h1(DEPTNO) ≠ h2(DEPTNO) )
  )
}
```

第一个（唯一标识）表约束指出，每个员工在每个日期最多只能有一条历史记录。第二个表约束指出，两个连续的历史记录（针对同一员工）不能同时具有不变的月薪和不变的部门编号；这些属性中至少有一个必须具有不同的值。表谓词指定此约束的方式是，对所有 `HIST` 元组对（`h1` 和 `h2`）进行全称量化；如果这样一对元组涉及同一员工的两个连续 `HIST` 元组，那么 `MSAL` 或 `DEPTNO` 属性值必须不同。两个 `HIST` 元组是连续的这一事实是通过存在量化来指定的：不存在另一个（第三个）针对同一员工的 `HIST` 元组（`h3`）“介于”其他两个元组之间。

再次强调，此约束的形式化规范没有留下任何模糊空间。

至此，关于示例数据库设计的表宇宙的讨论就结束了。下一节将转向数据库宇宙的定义。此定义将建立在表宇宙的基础上，并将形式化地指定数据库约束。

###### 数据库宇宙

你现在已经到达本书形式化方法论中指定数据库设计的最后一步（阶段）。与其他阶段一样，它继续建立在前一阶段指定的集合之上。

你可以使用表宇宙来构建一个*可接受的数据库状态集合*（我们很快将演示这一点）。这样的集合被称为*数据库宇宙*。数据库宇宙中的每个元素都是该数据库设计的一个可接受的数据库状态。

在本章开头，我们说“每个数据库状态都是一个可接受的由 n 个表（每个表结构一个）组成的集合。”当然，这是粗略的说法。形式化表示数据库状态的方式在第 5 章介绍过；它是一个函数，其中每个对的第一个元素是表结构名称，每个对的第二个元素是表。正如你将看到的，这种表示数据库状态的方式很方便；我们可以使用已定义的表宇宙轻松构建这样的函数。

让我们用前面各节中使用的 `RESULT` 表结构来说明这一点。

因为本节是关于数据库约束的，所以我们首先引入第二个表结构与 `RESULT` 表结构一起使用。假设存在一个 `LIMIT` 表结构。以下是 `LIMIT` 的外部谓词：“平均分数 `SCORE` 不允许用于总体 `POPULATION`。”其思想是 `LIMIT` 中的元组限制了 `RESULT` 的可接受表。我们将通过数据库约束来精确指定它们是如何限制的。

清单 7-36 指定了特征 `chr_LIMIT`、元组宇宙 `tup_LIMIT` 和表宇宙 `tab_LIMIT`。

**清单 7-36.** `chr_LIMIT`、`tup_LIMIT` 和 `tab_LIMIT` 的规范

```
chr_LIMIT :=
{ ( POPULATION; {'DP','NON-DP'} )
, ( SCORE; {'A','F'} )
}

tup_LIMIT :=
{ l | l∈Π(chr_LIMIT) ∧
  l(POPULATION) = 'DP' ⇒ l(SCORE) = 'A' ∧
  l(POPULATION) = 'NON-DP' ⇒ l(SCORE) = 'F'
}

tab_LIMIT :=
{ L | L∈℘(tup_LIMIT) ∧
  ( ∀l1,l2∈L: l1(POPULATION) = l2(POPULATION) ⇒ l1 = l2 )
}
```

`LIMIT` 的元组宇宙和表宇宙都是相当小的集合。以下是它们两者的枚举式规范：

```
tup_LIMIT = { { (POPULATION;'DP') , (SCORE;'A') }
            , { (POPULATION;'NON-DP'), (SCORE;'F') } }

tab_LIMIT = { ∅
            , { { (POPULATION;'DP') , (SCORE;'A') } }
            , { { (POPULATION;'NON-DP'), (SCORE;'F') } }
            , { { (POPULATION;'DP') , (SCORE;'A') }
              , { (POPULATION;'NON-DP'), (SCORE;'F') } }
            }
```

如你所见，只有四个可接受的 `LIMIT` 表：空表、两个包含一个元组的表和一个包含两个元组的表。

**注意：** `tab_LIMIT` 规范中的唯一标识表约束实际上是冗余的。它是由元组约束和特征隐含的。

图 7-4 使用表的简写符号显示了 `tab_LIMIT`。

**图 7-4.** 使用表简写符号对 `tab_LIMIT` 的枚举式规范

给定这两个表结构，我们现在可以看一下一个数据库状态。图 7-5 显示了一个名为 `DBS1` 的数据库状态，它涉及 `RESULT` 和 `LIMIT` 表结构。

**图 7-5.** 数据库状态 `DBS1`

如你所见，它是一个包含两个对的函数。第一个有序对的第二个坐标是一个可接受的 `RESULT` 表（即 `tab_RESULT` 的一个元素）。同样，第二个有序对的第二个坐标是一个可接受的 `LIMIT` 表（即 `tab_LIMIT` 的一个元素）。

这些表可以通过它们的表结构名称访问：作为第一个坐标出现的值 `RESULT` 和 `LIMIT`。使用 `DBS1` 的定义，你可以编写诸如 `DBS1(RESULT)` 和 `DBS1(LIMIT)` 的表达式，它们分别产生显示的 `RESULT` 和 `LIMIT` 表。

现在看一下以下名为 `DBCHR` 的*集合*函数：
`DBCHR := { (RESULT; tab_RESULT), (LIMIT; tab_LIMIT) }`

集合函数 `DBCHR` 被称为*数据库特征*。它通过引入数据库设计中涉及的表结构的名称，并将相关的表宇宙（为每个表结构保存可接受表的集合）附加到它们上面，从而对数据库进行特征描述。



**■注意** 数据库状态 `DBS1` 是 `∏(DBCHR)` 的一个元素。`DBS1` 与函数 `DBCHR` 具有相同的定义域，并且 `DBS1` 的每个第二坐标都是从 `DBCHR` 中相应第二坐标出现的集合（`tab_RESULT` 和 `tab_LIMIT`）中选出的一个元素。

你可以通过取集合函数 `DBCHR` 的广义乘积，来构造一个包含结果/限制数据库设计所有可能数据库状态的集合：
`DB_U1 := { dbs | dbs∈∏(DBCHR) }`

在集合 `DB_U1` 中，每个元素 `dbs` 都是在表全集 `tab_RESULT` 和 `tab_LIMIT` 下的一个可能的数据库状态。广义乘积算子生成了一个 RESULT 表和一个 LIMIT 表的所有可能组合；总共有 52 种组合（`#tab_RESULT` 乘以 `#tab_LIMIT`）。

现在，数据库约束终于登场了。你可以通过在规范中添加*数据库谓词*来限制集合 `DB_U1`。看一下列表 7-37，它引入了集合 `DB_U2`。

**列表 7-37.** *数据库全集 DB_U2*

```
DB_U2 :=
{ dbs | dbs∈∏(DBCHR) ∧
( ∀r∈db(RESULT),l∈db(LIMIT):
r(POPULATION) = l(POPULATION) ⇒ r(AVG_SCORE) ≠ l(SCORE) ) }
```

`DB_U2` 的规范包含一个数据库谓词。它表明，如果存在针对数据库专业人员的限制，则不能存在针对数据库专业人员的平均成绩为 A 的结果；并且如果存在针对非数据库专业人员的限制，则不能存在针对非数据库专业人员的平均成绩为 F 的结果。

这个数据库谓词丢弃了由广义乘积算子生成、但不符合我们现实世界可接受表示的可能数据库状态。

约束数据库全集中元素的数据库谓词被称为*数据库约束*。

**■注意** 给定数据库全集 `DB_U2` 的约束，图 7-5 所示的状态 `DBS1` 是一个不可接受的状态（即 `DB_U2∉DBS1`）；`RESULT` 表包含一个针对数据库专业人员的平均成绩为 A 的元组。根据数据库状态 `DBS1` 中 `LIMIT` 表的内容，这是不允许的。在全集 `DB_U2` 中，数据库约束将总共丢弃 26 个（可能的）数据库状态。你可能想自己验证一下。

```
7451CH07.qxd 5/15/07 9:43 AM Page 171

第 7 章 ■ 规范数据库设计

171
```

我们经常会以稍有不同的方式指定数据库约束。我们不会指定一个以数据库状态类型为单参数的数据库谓词，而是会指定一个以两个或多个表类型为参数的谓词。为了说明这一点，这里是定义数据库全集 `DB_U2` 的另一种方式：

`DB_U2 := { dbs | dbs∈∏(DBCHR) ∧ PDC1(dbs(RESULT),dbs(LIMIT)) }`

在这个定义中，我们实例化了一个名为 `PDC1` 的谓词，它接受两个表作为其参数。其定义如下（`R` 和 `L` 代表表类型的参数）：
```
PDC1(R,L) := ( ∀r∈R,l∈L:
r(POPULATION) = l(POPULATION) ⇒ r(AVG_SCORE) ≠ l(SCORE)
)
```

这两个定义共同构成了与列表 7-37 中给出的规范等价的规范。这种指定方式立即显示了每个数据库约束中涉及多少张表，并且在数据库谓词内部，你可以直接使用表参数，而不必调用具有特定表结构名称的数据库状态。

现在让我们继续示例数据库设计，定义它的数据库全集，从而正式指定所有的数据库约束。我们从定义数据库特征化开始。看一下列表 7-38，它包含了 `DB_CHREX` 的定义。

**列表 7-38.** *数据库特征化 DB_CHREX*

```
DB_CHREX :=
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


## 第 7 章 指定数据库设计

`Set`函数`DB_CHREX`引入了十个表别名，并将每个别名与前一阶段指定的相关表宇宙关联起来。`DB_CHREX`的广义乘积将产生一个包含许多可能数据库状态的集合。我们的数据库宇宙`DB_UEX`（在代码清单 7-39 中定义）建立了 36 个数据库约束。它们在此定义中仅通过名称指定。在接下来的章节中，您将找到每个命名约束的正式规范。

###### 代码清单 7-39. 数据库宇宙 `DB_UEX`

```
DB_UEX :=

{ dbs | dbs∈Π(DB_CHREX) ∧

/* ======================================= */

/* Start of Subset Requirement Constraints */

/* ======================================= */

PSSR1(dbs(EMP),dbs(DEPT)) ∧ PSSR2(dbs(DEPT),dbs(EMP)) ∧

PSSR3(dbs(MEMP),dbs(EMP)) ∧ PSSR4(dbs(TERM),dbs(EMP)) ∧

PSSR5(dbs(EMP),dbs(GRD)) ∧ PSSR6(dbs(OFFR),dbs(CRS)) ∧

PSSR7(dbs(OFFR),dbs(DEPT)) ∧ PSSR8(dbs(OFFR),dbs(EMP)) ∧

PSSR9(dbs(REG),dbs(EMP)) ∧ PSSR10(dbs(REG),dbs(OFFR)) ∧

PSSR11(dbs(HIST),dbs(EMP)) ∧ PSSR12(dbs(HIST),dbs(DEPT)) ∧

/* =================================== */

/* Start of Specialization Constraints */

/* =================================== */

PSPEC1(dbs(EMP),dbs(SREP)) ∧ PSPEC2(dbs(EMP),dbs(MEMP)) ∧

/* ================================== */

/* Start of Tuple-in-Join Constraints */

/* ================================== */

PTIJ1(dbs(EMP),dbs(GRD)) ∧ PTIJ2(dbs(EMP),dbs(TERM)) ∧

PTIJ3(dbs(SREP),dbs(EMP),dbs(MEMP)) ∧ PTIJ4(dbs(EMP),dbs(MEMP)) ∧

PTIJ5(dbs(EMP),dbs(DEPT)) ∧ PTIJ6(dbs(EMP),dbs(HIST)) ∧

PTIJ7(dbs(TERM),dbs(HIST)) ∧ PTIJ8(dbs(EMP),dbs(REG)) ∧

PTIJ9(dbs(TERM),dbs(REG),dbs(CRS)) ∧

PTIJ10(dbs(EMP),dbs(REG),dbs(OFFR),dbs(CRS)) ∧

PTIJ11(dbs(EMP),dbs(OFFR)) ∧ PTIJ12(dbs(TERM),dbs(OFFR),dbs(CRS)) ∧

PTIJ13(dbs(REG),dbs(OFFR)) ∧ PTIJ14(dbs(OFFR),dbs(CRS)) ∧

PTIJ15(dbs(EMP),dbs(REG),dbs(OFFR),dbs(CRS)) ∧

/* =================================== */

/* Start of Other Database Constraints */

/* =================================== */

PODC1(dbs(TERM),dbs(MEMP)) ∧ PODC2(dbs(TERM),dbs(DEPT)) ∧

PODC3(dbs(OFFR),dbs(DEPT),dbs(EMP),dbs(CRS)) ∧ PODC4(dbs(OFFR),dbs(REG)) ∧

PODC5(dbs(OFFR),dbs(REG)) ∧ PODC6(dbs(OFFR),dbs(REG)) ∧

PODC7(dbs(OFFR),dbs(REG),dbs(EMP))

}
```

如您所见，大多数数据库约束都基于第 6 章“表和数据库谓词的常见模式”一节中引入的常见谓词类型之一；在此数据库宇宙定义中有 12 个子集需求约束、两个特化约束和 15 个元组-连接约束。最后，还有七个“其他”（即非常见类型）数据库约束。大多数数据库约束仅涉及一对表结构；然而，一些元组-连接和“其他”数据库约束涉及两个以上的表结构。

在本节的剩余部分，您将找到所有数据库约束的正式规范，其名称在`DB_UEX`的定义中已介绍。为方便起见，每个这些谓词中表参数的名称与谓词所涉及的相关表结构的名称相同。正如您在`DB_UEX`规范中看到的，数据库谓词通过提供相关表作为参数来实例化。

代码清单 7-40 显示了数据库谓词`PSSR1`到`PSSR12`，它们代表了示例数据库设计中的子集需求。

###### 代码清单 7-40. `DB_UEX`的子集需求

```
PSSR1(EMP,DEPT) :=

/* Employee works for a known department */

{ e(DEPTNO) | e∈EMP } ⊆ { d(DEPTNO) | d∈DEPT }

PSSR2(DEPT,EMP) :=

/* Dept mgr is a known employee, excluding admins and president */

{ d(MGR) | d∈DEPT } ⊆

{ e(EMPNO) | e∈EMP ∧ e(JOB) ∉ {'ADMIN','PRESIDENT'} }
```

