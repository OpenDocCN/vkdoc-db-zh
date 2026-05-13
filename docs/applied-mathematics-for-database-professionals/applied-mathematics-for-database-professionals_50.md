# 第 9 章 ■ 数据检索

209

```sql
,d.DNAME
,count(e.EMPNO) as NUM_EMP
,nvl(sum(e.MSAL),0) as SUM_MSA
from DEPT d
,EMP e
where d.DEPTNO = e.DEPTNO (+)
group by d.DEPTNO,d.DNAME
```

你看到了吗，查询结果是如何构建出额外的属性 `NUM_EMP` 和 `SUM_MSAL` 的？查询 `EXQ5` 返回了一个基于 `{DEPTNO,DNAME,NUM_EMP,SUM_MSAL}` 的表。对于 `dbs(DEPT)` 中的每一个元组，结果集中都出现一个对应的元组。前两个属性直接取自 `dbs(DEPT)` 的一个元组。你另外指定了后两个属性（`NUM_EMP` 和 `SUM_MSAL`），并通过元组联合将它们添加到前两个属性中。你将 `NUM_EMP` 的值指定为包含当前部门所有员工的集合的基数。类似地，你通过使用 `SUM` 聚合运算符来指定属性 `SUM_MSAL` 的值。

`FEa` 和 `FEb` 之间的区别在于将部门元组限制为属性 `DEPTNO` 和 `DNAME` 的“时机”。规范 `FEa` 在竖线（`|`）之前的表达式中执行元组限制；`FEb` 则在变量 `d` 所绑定的表上执行表投影（在竖线之后）。

你应该注意以下几点：
- 此查询的形式化规范说明，空部门（即不雇佣任何员工的部门）将被返回到结果集中。
- 对于空部门，`NUM_EMP` 和 `SUM_MSAL` 属性都有明确定义的值。空集的基数是明确定义的（它是零），并且我们已经定义了求和运算符（参见定义 2-12），使得任何在空集上的表达式求和结果都等于零。

现在让我们来看看 SQL 表达式 `SEI`、`SEII` 和 `SEIII`。这是一个很好的例子，展示了（书中引言提到的）“麻烦之盒”是如何被打开的。

规范 `SEI` 遵循了此查询的形式化规范；使用了子查询来计算员工数量和薪资总和。但是请注意，SQL 的 `count` 运算符被定义为对空表返回零，而 SQL 的 `sum` 运算符被定义为在对空表执行时返回 `NULL`。为了确保在部门没有雇佣员工的情况下，SQL 查询将薪资总和返回为 0（零），我们使用了二元 `NVL` 函数。该函数的语义如下：如果第一个参数代表一个非 `NULL` 的值，则 `NVL` 返回该第一个参数，否则 `NVL` 返回第二个参数。

规范 `SEII` 通常是针对此类问题生成的。你执行一个从 `DEPT` 到 `EMP` 的连接，并通过对行进行分组（`GROUP BY`），可以计算出两个聚合属性。但是，请注意这个 SQL 查询不会返回空部门，因此不代表我们原始查询的正确 SQL 版本。顺便说一句，由于此查询只返回至少雇佣了一名员工的部门，`SUM` 聚合永远不会产生 `NULL`，因此你可以省略 `NVL`。

现在，如果你是一名熟练的 SQL 程序员，你可能知道如何修复这个问题，对吧？

规范 `SEIII` 代表了对 `SEII` 的修复版本；它采用了外连接来确保空部门也会被返回到结果集中。在 `e.DEPTNO` 后面添加 `(+)` 将原始连接的语义更改为 *从表 `DEPT` 到表 `EMP`* 的外连接。外连接也会在“目标”表中找不到匹配行时，为“来源”表返回一行；系统将生成一行全 `NULL` 值的行，并与“来源”表的行进行组合。

7451CH09.qxd 5/7/07 11:28 AM Page 210

210

不过，你应该意识到，在修复后的查询 `SEIII` 中，你现在必须将 `count(*)` 改为——例如——`count(e.EMPNO)`，才能使 SQL 正确地返回 `NUM_EMP` 属性为零员工的值（`count(*)` 会为空部门返回一）。是的，现在“麻烦之盒”已经完全打开了。


### 第 9 章 数据检索

#### 清单 9-8. 查询 `EXQ8(dbs)`

**FEa**:
```
{ d↓{DEPTNO,DNAME} ∪
{ (RICHEMP; { e1↓{EMPNO,ENAME,MSAL} | e1∈dbs(EMP) ∧ e1(DEPTNO)=d(DEPTNO) ∧
¬(∃e2∈dbs(EMP): e2(DEPTNO)=e1(DEPTNO) ∧
e2(MSAL)>e1(MSAL)) }
)
}
| d∈dbs(DEPT) }
```

**FEb**:
```
dbs(DEPT)⇓{DEPTNO,DNAME}⊗
{ e1↓{EMPNO,ENAME,MSAL,DEPTNO} | e1∈dbs(EMP) ∧
¬(∃e2∈dbs(EMP): e2(DEPTNO)=e1(DEPTNO) ∧
e2(MSAL)>e1(MSAL)) }
```

**FEc**:
```
dbs(DEPT)⇓{DEPTNO,DNAME}⊗
{ e1↓{EMPNO,ENAME,MSAL,DEPTNO} | e1∈dbs(EMP) ∧
(∀e2∈dbs(EMP): e2(DEPTNO)=e1(DEPTNO) ⇒
e2(MSAL)≤e1(MSAL)) }
```

**SEI**:
```
select d.DEPTNO, d.DNAME
,cursor(select e1.EMPNO, e1.ENAME, e1.MSAL
from EMP e1
where e1.DEPTNO = d.DEPTNO
and not exists(select e2.*
from EMP e2
where e2.DEPTNO = e1.DEPTNO
and e2.MSAL > e1.MSAL)) as RICHEMP
from DEPT d
```

**SEII**:
```
select d.DEPTNO, d.DNAME, e.EMPNO, e.ENAME, e.MSAL
from DEPT d
,(select e1.EMPNO,e1.ENAME,e1.MSAL,e1.DEPTNO
from EMP e1
where not exists(select e2.*
from EMP e2
where e2.DEPTNO = e1.DEPTNO
and e2.MSAL > e1.MSAL)) e
where d.DEPTNO = e.DEPTNO
```

下一个查询为每个部门检索该部门内具有最高薪资的员工。注意，有可能两名或多名员工拥有相同的最高薪资。清单 9-8 给出了每个部门（`DEPTNO` 和 `DNAME`）及其部门内拥有最高薪资的一名或多名员工（`EMPNO`、`ENAME` 和 `MSAL`）。

请注意 `FEa` 和 `FEb` 指定的结果集在结构上的差异。查询 `FEa` 返回一个基于 `{DEPTNO, DNAME, RICHEMP}` 的表；每个部门返回一个元组（空部门也会返回）。在属性 `RICHEMP` 下，返回一个基于 `{EMPNO, ENAME, MSAL}` 的嵌套表。属性 `RICHEMP` 保存代表该部门最高薪资员工的员工元组集合（仅限三个属性）。查询 `FEb` 返回一个基于 `{DEPTNO, DNAME, EMPNO, ENAME, MSAL}` 的表。此查询不返回空部门。此外，如果一个部门内有多名员工都赚取相同的最高薪资，则该部门的数据会为这些员工重复显示。

让我们通过给出 `dbs(DEPT)` 和 `dbs(EMP)` 的示例表来进一步澄清这一点。图 9-1 展示了 `DB_UEX` 在某个给定状态 `dbs` 下的部门表和员工表的投影版本。

#### 图 9-1. 部门和员工表示例

给定这两张表，查询 `EXQ8` 的 `FEa` 版本将返回以下集合：
```
{ {(DEPTNO;10), (DNAME;'RESEARCH'),
(RICHEMP; { {(EMPNO;1002), (ENAME;'SUSAN'), (MSAL;5500)}
,{(EMPNO;1003), (ENAME;'BRITNEY'), (MSAL;5500)} })}
}
,{(DEPTNO;11), (DNAME;'SALES'), (RICHEMP;∅)}
,{(DEPTNO;12), (DNAME;'MARKETING'),
(RICHEMP; { {(EMPNO;1005), (ENAME;'DEBBY'), (MSAL;6200)} })}
}
```

`FEb` 版本将返回此集合：
```
{ {(DEPTNO;10), (DNAME;'RESEARCH'), (EMPNO;1002), (ENAME;'SUSAN'), (MSAL;5500)}
,{(DEPTNO;10), (DNAME;'RESEARCH'), (EMPNO;1003), (ENAME;'BRITNEY'),(MSAL;5500)}
,{(DEPTNO;12), (DNAME;'MARKETING'),(EMPNO;1005), (ENAME;'DEBBY'), (MSAL;6200)}
}
```

规范 `FEc` 再次展示了量词的重写；它与 `FEb` 的区别在于将存在量词重写为全称量词。

SQL 表达式 `SEI` 表明像 `FEa` 这样的规范可以直接翻译成 SQL。通过使用关键字 `cursor`，你指明了在 `SELECT` 子句中使用的子查询有可能返回多行。SQL 表达式 `SEII` 是 `FEb` 的直接翻译；它在 `FROM` 子句中使用了子查询。

**注意** 在接下来的清单 9-9 中，我们在表达式 `FEa` 和 `FEb` 中使用了一个函数 `to_year`。该函数被定义为返回给定日期值的年份部分。在 Oracle SQL 中，你可以通过 `to_char` 函数实现相同的功能，如 `SEI` 和 `SEII` 中所演示的那样。

清单 9-9 给出了参加了培训师 “De Haan”（员工编号 1206）在 2003 年开设的每一门课程至少一次的员工（`EMPNO` 和 `ENAME`）。


