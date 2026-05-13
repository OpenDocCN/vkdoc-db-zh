# 第十章 数据操纵

#### 代码清单 10-2. 事务 ETX1(dbs)

在 SQL 规范中，我们将使用由数据库特征 `DB_CHREX` 引入的表结构名称（参见代码清单 7-38）作为可用于 `INSERT`、`UPDATE` 和 `DELETE` 语句的表名。备选的 SQL 规范将被命名为 `SEr`（`SQL` 表达式 r），其中 r 代表一个罗马数字（I，II，III，等等）。

在以下的形式化表达中，我们将把事务指定为一个类似于上一节中指定 `Tx1a` 和 `Tx2a` 的方式（总是产生一个修改后的数据库状态）的函数。然而，我们打算——如前一节所讨论的——所有静态和动态约束都应保持满足；也就是说，只要结果数据库状态不是 `DB_UEX` 的一个元素，或者表示从起始状态到结束状态的转移的序偶不是 `ST_UEX` 的一个元素，事务就应该回滚。

在上一节中，你看到了两个插入单个元组的事务示例。

看一下代码清单 10-2 中的例子。它指定了一个（可能）插入多个元组的事务。代码清单 10-2 注册了过去一个季度（91 天）内雇用的所有管理员，用于 2007 年 3 月 1 日开始的课程 `AM4DP` 的开设。

**FEa**: `dbs↓{EMP,SREP,MEMP,TERM,DEPT,GRD,CRS,OFFR,HIST}`
∪
`{ (REG; dbs(REG)`
∪
`{ { (COURSE;'AM4DP'`
`)`
`,(STARTS;'01-mar-2007')`
`,(STUD;`
`e(EMPNO)`
`)`
`,(EVAL;`
`) }`
`| e∈dbs(EMP) ∧ e(JOB)='ADMIN' ∧ e(HIRED) ≥ sysdate-91`
`∧ e(HIRED) ≤ sysdate } ) }`

**SEI**:
```sql
insert into REG(STUD,EVAL,COURSE,STARTS)
select e.EMPNO, -1, 'AM4DP', '01-mar-2007'
from EMP e
where e.JOB = 'ADMIN'
and e.HIRED between sysdate - 91 and sysdate
```

此事务仅更改 `REG` 表结构；查询 `EMP` 表结构得到的结果元组被添加到其中。在这些元组中，属性 `COURSE`、`STARTS` 和 `EVAL` 分别被设置为值 `'AM4DP'`、`'01-mar-2007'` 和 `-1`。属性 `STUD` 遍历过去 91 天内雇用的所有管理员的员工编号。

在形式化规范中，此规范查询部分内枚举的属性-值对的顺序无关紧要。然而，在 SQL 表达式中，第二行（以关键字 `SELECT` 开头的那一行）列出的值表达式的顺序必须与第一行（紧接关键字 `INSERT` 之后）列出的属性的顺序相匹配。

**■ 注意** 在 SQL 中，如 `SEI` 中显示的嵌入式查询通常被称为*子查询*。

最后，你应该认识到，因为 `SEI` 中的子查询选择了 `EMPNO`（`EMP` 表结构中的唯一标识属性），所以不需要在 `select` 关键字之后立即包含 `distinct` 关键字。

#### 代码清单 10-3. 事务 ETX2(dbs)

另一个常见的事务是从数据库中删除元组。代码清单 10-3 给出了一个删除注册记录的示例。代码清单 10-3 删除了学生 3124 的所有未来开始日期的已计划或已确认的课程注册记录。

**FEa**: `dbs↓{EMP,SREP,MEMP,TERM,DEPT,GRD,CRS,OFFR,HIST}`
∪
`{ (REG; { r | r∈dbs(REG) ∧`
`¬ ( r(STUD) = 3124 ∧`
`r(STARTS) > sysdate ∧`
`↵{ o(STATUS) | o∈dbs(OFFR) ∧`
`o↓{COURSE,STARTS} = r↓{COURSE,STARTS} }`
`∈{'SCHD','CONF'} )`
`} ) }`

**FEb**: `dbs↓{EMP,SREP,MEMP,TERM,DEPT,GRD,CRS,OFFR,HIST}`
∪
`{ (REG; { r↓{STUD,COURSE,STARTS,EVAL} | r∈dbs(REG)⊗dbs(OFFR) ∧`
`( r(STUD) ≠ 3124 ∨`
`r(STARTS) ≤ sysdate ∨`
`r(STATUS) = 'CANC' )`
`} ) }`

**FEc**: `dbs↓{EMP,SREP,MEMP,TERM,DEPT,GRD,CRS,OFFR,HIST}`
∪
`{ (REG; dbs(REG) -`
`{ r | r∈dbs(REG) ∧ r(STUD) = 3124 ∧ r(STARTS) > sysdate ∧`
`↵{ o(STATUS) | o∈dbs(OFFR) ∧`
`o↓{COURSE,STARTS} = r↓{COURSE,STARTS} }`
`∈{'SCHD','CONF'}`
`} ) }`

**SEI**:
```sql
delete from REG r
where r.STUD = 3124
and r.STARTS > sysdate
and (select o.STATUS
from OFFR o
where o.COURSE = r.COURSE
and o.STARTS = r.STARTS) in ('SCHD','CONF') )
```



