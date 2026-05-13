# 第 11 章：在 Oracle 中实现数据库设计

## 过渡效应

过渡效应描述了受 DML 语句影响的实际行，对于`UPDATE`语句而言，还包括这些行是如何被修改的。过渡效应为`语句后触发器`提供了一种便捷方式，可以精确查看哪些行受到了触发它的`DML`语句的影响。

通过在`语句后触发器`中检查过渡效应，你能够解决之前为`EM4`提到的效率低下问题。

你可以将过渡效应实现为一个视图，该视图仅保存在`DML`语句处理后立即存在的行。触发的`语句后触发器`可以访问此视图，以确定究竟哪些行被插入、删除或更新。

在此执行模型中，我们假设每个表结构都有三个可用的过渡效应 (`TE`) 视图：

*   **插入 TE 视图**，名为 `v_[表名]_ite`：此视图将显示触发它的`INSERT`语句刚刚插入的行。如果触发语句不是`INSERT`，则此视图为空（不包含任何行）。
*   **更新 TE 视图**，名为 `v_[表名]_ute`：此视图将显示触发它的`UPDATE`语句刚刚更新的行；此视图会显示被修改列的新旧值。如果触发语句不是`UPDATE`，则此视图为空。
*   **删除 TE 视图**，名为 `v_[表名]_dte`：此视图将显示触发它的`DELETE`语句刚刚删除的行。如果触发语句不是`DELETE`，则此视图为空。

目前，Oracle 的`SQL DBMS`并未为你提供过渡效应（至少还有另外一家`DBMS`供应商提供了此功能）。但是，你可以为每个表结构开发行级和语句级触发器，通过执行必要的簿记工作来提供这三个`TE`视图。

## 实现示例

请参见清单 11-33。它展示了为维护`EMP`表结构的过渡效应所需的`DI`代码。行级触发器使用会话临时表`EMP_TE`来存储过渡效应。在此表之上，定义了三个`TE`视图。

**清单 11-33.** 维护`EMP`表结构过渡效应的 DI 代码

```sql
create global temporary table EMP_TE
(DML char(1) not null check(DML in ('I','U','D'))
,ROW_ID rowid
,EMPNO number(4,0)
,JOB varchar(9)
,HIRED date
,SGRADE number(2,0)
,MSAL number(7,2)
,DEPTNO number(2,0)
,check(DML<>'I' or ROW_ID is not null)
,check(DML<>'U' or ROW_ID is not null)
,check(DML<>'D' or ROW_ID is null)
) on commit delete rows
/

create trigger EMP_BIUDS_TE
before insert or update or delete on EMP
begin
-- 在每个 DML 语句执行前重置过渡效应。
delete from EMP_TE;
--
end;
/

create trigger EMP_AIUDR_TE
after insert or update or delete on EMP
for each row
begin
-- 有条件地维护过渡效应。
if INSERTING
then
-- 仅存储指向受影响行的“指针”。
insert into EMP_TE(DML,ROW_ID) values('I',:new.rowid);
elsif UPDATING
then
-- 存储旧行的快照，加上指向新版本的指针。
insert into EMP_TE(DML,ROW_ID,EMPNO,JOB,HIRED,SGRADE,MSAL,DEPTNO) values ('U',:new.rowid,:old.empno,:old.job,:old.hired
,:old.sgrade,:old.msal,:old.deptno);
elsif DELETING
then
-- 存储旧行的快照。
insert into EMP_TE(DML,ROW_ID,EMPNO,JOB,HIRED,SGRADE,MSAL,DEPTNO) values ('D',null,:old.empno,:old.job,:old.hired
,:old.sgrade,:old.msal,:old.deptno);
end if;
--
end;
/

create view V_EMP_ITE as
select e.*
from EMP_TE te
,EMP e
where DML='I'
and te.ROW_ID = e.ROWID
/

create view V_EMP_UTE as
select e.EMPNO as N_EMPNO ,e.JOB as N_JOB ,e.HIRED as N_HIRED
,e.SGRADE as N_SGRADE ,e.MSAL as N_MSAL ,e.DEPTNO as N_DEPTNO
,te.EMPNO as O_EMPNO ,te.JOB as O_JOB ,te.HIRED as O_HIRED
```



