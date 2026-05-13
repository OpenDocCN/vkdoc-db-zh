# 第 7 章 指定数据库设计

###### 代码清单 7-26. *表全域 `tab_EMP`*

```
tab_EMP :=

{ E | E∈℘(tup_EMP) ∧

/* EMPNO 唯一地标识一个员工元组 */

( ∀e1,e2∈E: e1(EMPNO) = e2(EMPNO) ⇒ e1 = e2 )

∧

/* USERNAME 唯一地标识一个员工元组 */

( ∀e1,e2∈E: e1(USERNAME) = e2(USERNAME) ⇒ e1 = e2 )

∧

/* 最多只允许一位总裁 */

#{ e | e∈E ∧ e(JOB) = 'PRESIDENT' } ≤ 1

∧

/* 雇佣了总裁或经理的部门 */

/* 也应至少雇佣一名管理员 */

( ∀d∈E⇓{DEPTNO}:

( ∃e2∈E: e2(DEPTNO) = d(DEPTNO) ∧ e2(JOB) ∈ {'PRESIDENT','MANAGER'} )

⇒

( ∃e3∈E: e3(DEPTNO) = d(DEPTNO) ∧ e3(JOB) = 'ADMIN' )

)
}
```

前两个约束是唯一标识谓词；`EMPNO` 属性在 `EMP` 表中是唯一标识符，`USERNAME` 属性也是如此。

第三个约束的指定构造了 `JOB` 属性值为 `PRESIDENT` 的元组子集，并将此集合的基数限制为最多 1。这反映了现实世界中不能有超过一位总裁的要求。

最后一个表约束指出，如果在一个*给定*的部门中存在一个 `PRESIDENT` 或 `MANAGER`，那么在*同一个*部门中就存在一个 `ADMIN`。外部的全称量化提供了所有需要执行此检查的部门。请注意，此约束的非正式嵌入式说明可能暗示这是一个多表约束，不仅涉及 `EMP` 表结构，还涉及 `DEPT` 表结构。看看下面这个名为 `P` 的数据库谓词，它接受两个参数：一个部门表（参数 `D`）和一个员工表（参数 `E`）。

```
P(D,E) :=

( ∀d∈D:

( ∃e2∈E: e2(DEPTNO) = d(DEPTNO) ∧ e2(JOB) ∈ {'PRESIDENT','MANAGER'} )

⇒

( ∃e2∈E: e3(DEPTNO) = d(DEPTNO) ∧ e3(JOB) = 'ADMIN' )

)
```

这个谓词严格遵循非正式说明：它量化了所有部门，并指出如果部门雇佣了 `PRESIDENT` 或 `MANAGER`，那么该部门就应该雇佣一名 `ADMIN`。请注意，只使用了参数 `D` 的 `DEPTNO` 属性。在 `tab_EMP` 定义内部的相应谓词量化了 `EMP` 表中所有可用的 `DEPTNO` 值，然后陈述了同样的事情。因为 `EMP` 表结构中有一个 `DEPTNO` 属性（并且，我们稍后将看到，它总是标识一个有效的部门），所以这个约束可以被指定为一个表约束。

7451CH07.qxd 5/15/07 9:43 AM Page 163

