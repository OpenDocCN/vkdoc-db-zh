# 在 ORACLE 中实现数据库设计

##### 注意

在过渡效应中，你只需要维护那些涉及与`EMP`表结构相关的任何（多行）约束的列。这就是为什么前面的代码不维护列`ENAME`、`BORN`和`USERNAME`：这三个列不涉及`DB_UEX`的任何约束。

基于前面的代码，你现在可以为约束`EMP_TAB03`创建更高效的 `INSERT` 和 `DELETE` 语句后的触发器。请记住，在`EM4`中，如果插入了一个销售代表或删除了一名培训师，这个约束的`DI`代码将会不必要地运行。

利用过渡效应，你现在可以精确地编写在执行`INSERT`语句或`DELETE`语句时需要验证约束的代码：

-   对于插入操作：只有当语句插入了一名总裁或经理时，你才需要检查（同一部门内）是否存在管理员。
-   对于删除操作：只有当语句删除了一名管理员时，你才需要检查（是否仍然）有另一名管理员，以防该部门雇佣了经理或总裁。

在所有其他`INSERT`或`DELETE`语句的情况下，不需要验证约束`EMP_TAB03`。

在清单 11-34 中，你可以找到修改后的插入和删除触发器。它们现在首先查询过渡效应以验证前述属性之一是否为`TRUE`，如果是，才执行验证约束`EMP_TAB03`的查询。

**清单 11-34.** `EM5`为约束`EMP_TAB03`设计的更高效的插入和删除触发器

```sql
create trigger EMP_AIS_TAB03
after insert on EMP
declare
  pl_dummy varchar(40);
begin
  -- 如果此查询返回无行，则 EMP_TAB03 不可能被违反。
  select 'EMP_TAB03 must be validated' into pl_dummy
  from DUAL
  where exists (select 'A president or manager has just been inserted'
                from v_emp_ite
                where JOB in ('PRESIDENT','MANAGER'));
  --
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
exception when no_data_found then
  -- 无需验证 EMP_TAB03。
  null;
  --
end;
/
```

```sql
create trigger EMP_ADS_TAB03
after delete on EMP
declare
  pl_dummy varchar(40);
begin
  -- 如果此查询返回无行，则 EMP_TAB03 不可能被违反。
  select 'EMP_TAB03 must be validated' into pl_dummy
  from DUAL
  where exists (select 'An administrator has just been deleted'
                from v_emp_dte
                where JOB='ADMIN');
  --
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
exception when no_data_found then
  -- 无需验证 EMP_TAB03。
  null;
  --
end;
/
```

你可能已经注意到，随着过渡效应的可用，你现在也可以编写更高效的更新触发器。例如，将一名培训师更新为销售代表，不需要为`EMP_TAB03`执行`DI`代码。



写成在 `DUAL` 上的查询时，以下是需要查找的属性，它会在执行 `UPDATE` 语句时要求对 `EMP_TAB03` 进行验证：
```sql
select 'EMP_TAB03 is in need of validation'
from DUAL
where exists(select 'Some department just won a president/manager or just lost an administrator'
from v_emp_ute
where (n_job in ('PRESIDENT','MANAGER') and
o_job not in ('PRESIDENT','MANAGER')
or (o_job='ADMIN' and n_job<>'ADMIN')
or (o_deptno<>n_deptno and
(o_job='ADMIN' or n_job in ('PRESIDENT','MANAGER'))))
```

您可以使用上述查询，以与清单 11-34 所示的插入和删除触发器相同的方式，创建一个更高效的更新触发器。

在后文中，我们将这些对过渡效应视图的查询称为*过渡效应查询*（TE 查询）。

`注意` 我们不知道数据库管理系统是否能够计算在此执行模型中使用的 TE 查询（用于查找特定于约束的属性）。对该领域已进行的科学研究进行调查，并未为我们提供关于此问题的明确答案。因此，我们无法断然地说，数据库管理系统是否原则上应该能够通过执行模型 EM5 为我们提供完整的声明式多元组约束支持。

清单 11-35 提供了使用执行模型 EM5 实现第二个示例表约束的触发器。它假定维护`DEPT`表结构的过渡效应视图的代码已经以类似清单 11-33 中为`EMP`设置过渡效应的方式建立起来。请注意，约束`DEPT_TAB01`不需要删除触发器。

**清单 11-35.** *约束 DEPT_TAB01 的 DI 代码的 EM5 实现*
```sql
create trigger DEPT_AIS_TAB01
after insert on DEPT
declare pl_dummy varchar(40);
begin
-- 如果此查询不返回行，则 DEPT_TAB01 不可能被违反。
select 'DEPT_TAB01 must be validated' into pl_dummy
from DUAL
where exists (select 'A row has just been inserted'
from v_dept_ite);
--
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
exception when no_data_found then
-- 无需验证 DEPT_TAB01。
null;
--
end;
/

create trigger DEPT_AUS_TAB01
after update on DEPT
declare pl_dummy varchar(40);
begin
-- 如果此查询不返回行，则 DEPT_TAB01 不可能被违反。
select 'DEPT_TAB01 must be validated' into pl_dummy
from DUAL
where exists (select 'A department manager has just been updated'
from v_dept_ute
where o_mgr<>n_mgr);
--
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
exception when no_data_found then
-- 无需验证 DEPT_TAB01。
null;
--
end;
/
```

请注意，在约束`DEPT_TAB01`的情况下，执行模型 EM4 和 EM5 是等价的。

可能会产生这样一个问题：在过渡效应中测试某个属性是否违背了其初衷。这是因为，如果执行此测试的成本与运行实际的约束验证代码一样高，那么您的收益何在？对于这个问题，我们可以做出以下观察：

*   过渡效应中的行数通常远少于约束所涉及的表结构中的行数。
*   过渡效应总是在缓存中，而涉及的表结构中的所有行则不能这么说。


### 第 11 章 在 Oracle 中实现数据库设计

TE 查询本身通常比为验证约束而条件运行的查询要简单得多。

示例已展示了最后一点观察结果。结合前两点观察，这应该能够回答那个问题：TE 查询不会违背其目的（DI 代码的效率）。

此外，使用 TE 查询来*守护*约束验证还有第二个——更重要的——目的。防止执行不必要的约束验证查询将极大地有利于事务的并发性。我们将在下一小节“DI 代码序列化”中处理这个方面。

还有一种方法可以进一步优化执行模型 `EM5` 的效率。这涉及执行一个更高效的查询来验证约束。到目前为止，你都是直接从约束的形式化规范中推导出这些查询，因此它们验证的是*完整*的约束谓词。

在 `EMP_TAB03` 的情况下，该查询验证是否*每个*部门都满足量化的谓词（如果有总裁/经理，则有管理员）。在 `DEPT_TAB01` 的情况下，该查询验证是否*每个*部门经理管理的部门都不超过两个。通常，给定一个触发的 `DML` 语句，执行一个验证*较弱*谓词的查询就足够了。

例如，如果一条 `INSERT` 语句向 `EMP` 表结构插入一个部门 10 的经理，那么理论上你只需要验证该量化谓词是否仍*仅对部门 10* 满足。同样，如果一条 `UPDATE` 语句更改了某个部门的经理，那么你只需要验证这位新的部门经理管理的部门是否没有超过两个。

最后一个执行模型解决了这个进一步提高 DI 代码效率的机会。

## 执行模型 6：过渡效应属性加上优化查询
这个执行模型 (`EM6`) 与 `EM5` 相似，因为它也需要每个表结构的过渡效应；你现在以稍微更复杂的方式使用这个过渡效应，以便能够确定较弱的谓词。

你现在不仅利用过渡效应中的某个属性来守护约束验证查询的执行，而且还利用过渡效应来提供可用于优化验证查询的值。例如，在 `EMP_TAB03` 的情况下，你可以使用过渡效应来确定需要检查哪些部门的较弱谓词；这应该总是导致执行更高效的验证查询。在 `DEPT_TAB01` 的情况下，你可以使用过渡效应来找出需要检查哪些 `MGR` 值的较弱谓词。

清单 11-36 向你展示了如何做到这一点。它展示了约束 `EMP_TAB03` 的插入、更新和删除触发器的 `EM6` 版本。

**清单 11-36.** 约束 EMP_TAB03 的 DI 代码的 EM6 实现

```sql
create trigger EMP_AIS_TAB03
after insert on EMP
declare
  pl_dummy varchar(40);
begin
  --
  for r in (select distinct deptno
            from v_emp_ite
            where JOB in ('PRESIDENT','MANAGER'));
  loop
    begin
      --
      select 'Constraint EMP_TAB03 is satisfied' into pl_dummy
      from DUAL
      where not exists(select e2.*
                       from EMP e2
                       where e2.DEPTNO = r.DEPTNO
                       and e2.JOB in ('PRESIDENT','MANAGER'))
      or exists(select e3.*
                from EMP e3
                where e3.DEPTNO = r.DEPTNO
                and e3.JOB = 'ADMIN');
      --
    exception when no_data_found then
      --
      raise_application_error(-20999,
        'Constraint EMP_TAB03 is violated for department '||to_char(r.deptno)||'.');
      --
    end;
  end loop;
end;
/
```

```sql
create trigger EMP_ADS_TAB03
after delete on EMP
declare
  pl_dummy varchar(40);
begin
  --
  for r in (select distinct deptno
            from v_emp_dte
            where JOB='ADMIN');
  loop
    begin
      --
      select 'Constraint EMP_TAB03 is satisfied' into pl_dummy
      from DUAL
      where not exists(select e2.*
                       from EMP e2
                       where e2.DEPTNO = r.DEPTNO
                       and e2.JOB in ('PRESIDENT','MANAGER'))
      or exists(select e3.*
                from EMP e3
                where e3.DEPTNO = r.DEPTNO
                and e3.JOB = 'ADMIN');
      --
    exception when no_data_found then
      --
      raise_application_error(-20999,
        'Constraint EMP_TAB03 is violated for department '||to_char(r.deptno)||'.');
      --
    end;
  end loop;
end;
/
```

```sql
create trigger EMP_AUS_TAB03
after update on EMP
declare
  pl_dummy varchar(40);
begin
  --
  for r in (select n_deptno as deptno
            from v_emp_ute
            where (o_job not in ('PRESIDENT','MANAGER') or updated_deptno='TRUE') and n_job in ('PRESIDENT','MANAGER')
            union
            select o_deptno as deptno
            from v_emp_ute
            where (n_job<>'ADMIN' or updated_deptno='TRUE')
            and old_job ='ADMIN')
  loop
    begin
      --
      select 'Constraint EMP_TAB03 is satisfied' into pl_dummy
      from DUAL
      where not exists(select e2.*
                       from EMP e2
                       where e2.DEPTNO = r.DEPTNO
                       and e2.JOB in ('PRESIDENT','MANAGER'))
      or exists(select e3.*
                from EMP e3
                where e3.DEPTNO = r.DEPTNO
                and e3.JOB = 'ADMIN');
      --
    exception when no_data_found then
      --
      raise_application_error(-20999,
        'Constraint EMP_TAB03 is violated for department '||to_char(r.deptno)||'.');
      --
    end;
  end loop;
end;
/
```

##### 注意
前面 `TE` 查询中的 `distinct` 关键字可能防止在 `UPDATE` 语句影响了多行的情况下多次执行相同的约束验证查询。

因为示例约束恰好是全称量词，所以在 `EMP_TAB03` 和 `DEPT_TAB01` 的情况下找到较弱的谓词是显而易见的。然而，情况并非总是如此；要发现较弱的谓词也可能相当复杂。这完全取决于约束的规范。同样，我们不知道是否有可能，不必你自己去发现，而是让 `DBMS` 在给定声明的约束形式规范的情况下自动计算较弱的谓词（包括 `TE` 查询）。

###### DI 代码序列化
当你自己实现 `DI` 代码时，你应该意识到另一个主要问题。无论你是遵循触发式过程策略还是嵌入式过程策略，这个问题都适用。

在 Oracle 的 `SQL` `DBMS` 中，并发执行事务。在任何时候都可能有许多*开放*事务；也就是说，事务已启动但尚未提交。如本章前面所述，一个开放事务无法看到其他并发开放事务所做的更改。这在 `DI` 代码的正确性方面具有严重的影响。正如我们稍后将演示的，该问题的根本原因在于，作为 `DI` 代码一部分执行的约束验证查询，假定它读取的所有数据——或由于数据不存在而未能读取的数据——目前没有被其他事务更改（或创建）。对于给定约束的 `DI` 代码，只有当并发运行的、执行该约束验证查询（嵌入在 `DI` 代码中）的事务被*序列化*时，才能正确实现该约束。

##### 注意
我们实际上将演示在不支持并发执行事务*可串行性*的 `DBMS` 中出现的一些经典并发问题。如果你不熟悉可串行性的概念，请查阅我们在附录 C 中添加的一些参考文献。


让我们通过描述两个并发执行的事务场景来演示这个可串行性问题，这两个事务都可能违反 `EMP_TAB03` 约束。为简单起见，假设 `EMP` 表结构当前包含一行，代表在部门 20 工作的管理员。事务 `TX1` 在 `EMP` 表结构中插入一名在部门 20 工作的经理。事务 `TX2` 从 `EMP` 表结构中删除在部门 20 工作的管理员。现在请查看表 11-2，它描述了这两个事务并发执行的可能场景。

```
表 11-2. TX1 和 TX2 并发执行
时间        TX1       TX2       DI 代码
----------------------------------------------------------
t=0                   插入      找到管理员；因此允许此次插入。
t=1                   删除      未找到经理；因此允许此次删除。
t=2                   提交
t=3                   提交
```

在时间 `t=0`，事务 `TX1` 开始。它插入经理，这导致执行约束 `EMP_TAB03` 的 DI 代码。作为此 DI 代码一部分执行的约束验证查询找到了部门 20 的管理员，这导致 DI 代码允许此次插入。在时间 `t=1`，事务 `TX2` 开始。它删除 `EMP` 表结构中代表管理员的唯一一行。删除管理员也会触发执行约束 `EMP_TAB03` 的相同 DI 代码。作为此 DI 代码一部分执行的约束验证查询没有找到在部门 20 工作的经理（或总裁），这导致 DI 代码也允许此次删除。

**注意：** 在 `t=1`，事务 `TX2` 内执行的查询看不到事务 `TX1` 所做的未提交更改。

在 `t=2`，事务 `TX1` 提交。在 `t=3`，事务 `TX2` 提交。数据库现在违反了约束 `EMP_TAB03`；`EMP` 表结构中持有一名在部门 20 工作的经理，但该部门却没有管理员。

让我们也在约束 `DEPT_TAB01` 的上下文中看一个例子。假设 `DEPT` 表结构当前包含一个由员工 1042 管理的部门。事务 `TX3` 插入一个同样由员工 1042 管理的新部门。事务 `TX4` 也插入一个再次由员工 1042 管理的部门。请看表 11-3，它描述了这两个事务并发执行的可能场景。

```
表 11-3. TX3 和 TX4 并发执行
时间        TX3       TX4       DI 代码
----------------------------------------------------------
t=0                   插入      看到 1042 现在管理两个部门；因此允许此次插入。
t=1                   插入      看到 1042 现在管理两个部门；因此允许此次插入。
t=2                   提交
t=3                   提交
```

同样，在 `t=1` 时于 `TX4` 中执行的 DI 代码看不到事务 `TX3` 在 `t=0` 插入的未提交部门。两次插入都成功执行。在 `t=3` 之后，数据库违反了约束 `DEPT_TAB01`；员工 1042 正在管理三个部门。

对于每一个多元组约束，你都可以设计一个并发执行事务的场景，使得这些事务提交后，数据库处于违反约束的状态。请注意，这与用于实现 DI 代码的执行模型无关。

那么，问题是你能解决这个问题吗？答案是肯定的。然而，这要求你在 DI 代码中开发并嵌入有时相当复杂的串行化代码，以确保永远不会有 *两个涉及相同约束的事务同时执行* （即，运行相同约束的验证查询）。

**注意：** 以下部分假设你熟悉 Oracle 的 `dbms_lock` 包和自治事务的概念。如果不熟悉，我们建议你首先学习 Oracle 关于此包和自治事务的文档。



技巧在于使用 Oracle SQL 数据库管理系统提供的 `dbms_lock` 包。通过使用这个包，你可以获取应用锁，从而有效地序列化涉及相同约束的并发执行事务。

### 列表 11-37. 应用锁服务

`列表 11-37` 展示了构建在 `dbms_lock` 包之上的过程 `p_request_lock` 的代码，通过它你可以请求一个应用锁。这个过程需要调用一个会执行隐式提交的 `dbms_lock` 模块。因为你将从 DI 代码中调用 `p_request_lock`，而 DI 代码又是在触发器中执行的（在此上下文中不允许提交），所以你需要对当前事务隐藏这个隐式提交。你可以通过使用一个自治事务来实现这一点。辅助函数 `f_allocate_unique`（也在 `列表 11-37` 中）实现了这个自治事务。

```sql
create function f_allocate_unique
(p_lockname in varchar) return varchar as
--
pragma autonomous_transaction;
--
pl_lockhandle varchar(128);
--
begin
  -- This does implicit commit.
  dbms_lock.allocate_unique(upper(p_lockname)
                           ,pl_lockhandle
                           ,60*10); -- 将过期时间设为 10 分钟。
--
  return pl_lockhandle;
--
end;
/

create procedure p_request_lock(p_lockname in varchar) as
--
pl_lockhandle varchar(128);
pl_return number;
--
begin
--
  -- Go get a unique lockhandle for this lockname.
--
  pl_lockhandle := f_allocate_unique(p_lockname);
--
  -- Request the named application lock in exclusive mode.
  -- Allow for a blocking situation that lasts no longer than 60 seconds.
--
  pl_return :=
    dbms_lock.request(lockhandle => pl_lockhandle
                     ,lockmode => dbms_lock.x_mode
                     ,timeout => 60
                     ,release_on_commit => true);
--
  if pl_return not in (0,4)
  then
    raise_application_error(-20998,
    'Unable to acquire constraint serialization lock '||p_lockname||'.'); end if;
--
end;
/
```

现在，你可以——在这种情况下相当容易地——确保两个都需要为约束 `DEPT_TAB01` 运行 DI 代码的事务被正确地序列化，以防止在 `表 11-3` 中演示的并发问题。来看一下 `列表 11-38`，它展示了使用执行模型 `EM6` 为约束 `DEPT_TAB01` 编写的 DI 代码，其中包含了必要的对 `p_lock_request` 的调用以确保正确的序列化。

### 列表 11-38. DEPT_TAB01 的 DI 代码的 EM6 实现（包含序列化）

```sql
create trigger DEPT_AIS_TAB01
after insert on DEPT
declare pl_dummy varchar(40);
begin
--
  for r in (select distinct mgr as mgr
            from v_dept_ite)
  loop
    begin
      -- Acquire serialization lock.
      p_request_lock('DEPT_TAB01');
--
      select 'Constraint DEPT_TAB01 is satisfied' into pl_dummy from DUAL
      where 2 >= (select count(*)
                  from DEPT d
                  where d.MGR = r.MGR);
--
    exception when no_data_found then
--
      raise_application_error(-20999,'Constraint DEPT_TAB01 is violated '||
      'for department manager '||to_char(r.MGR)||'.');
--
    end;
  end loop;
--
end;
/

create trigger DEPT_AUS_TAB01
after update on DEPT
declare pl_dummy varchar(40);
begin
--
  for r in (select distinct n_mgr as mgr
            from v_dept_ute)
  loop
    begin
      -- Acquire serialization lock.
      p_request_lock('DEPT_TAB01');
--
      select 'Constraint DEPT_TAB01 is satisfied' into pl_dummy from DUAL
      where 2 >= (select count(*)
                  from DEPT d
                  where d.MGR = r.MGR);
--
    exception when no_data_found then
--
      raise_application_error(-20999,'Constraint DEPT_TAB01 is violated '||
      'for department manager '||to_char(r.MGR)||'.');
--
    end;
  end loop;
--
end;
/
```

在 `表 11-3` 中描述的场景现在按如下方式执行（参见 `表 11-4`）。

### 表 11-4. TX3 和 TX4 的序列化

| **时间** | **TX3** | **TX4** | **DI 代码** |
| :--- | :--- | :--- | :--- |
| t=0 | INSERT | | 看到 1042 目前管理着两个部门；因此允许此次插入。 |



t=1

INSERT

在序列化锁上被阻塞（`DI`代码的执行被暂停）。

t=2

COMMIT

（在 60 秒内。）

t=3

`TX4`解除阻塞，`DI`代码恢复执行。

t=4

`DI`代码看到员工 1042 现在管理着三个部门，插入操作失败。

在 t=1 时，事务`TX4`开始等待事务`TX3`释放应用程序锁。紧接着 t=2 之后，这个等待结束，`DI`代码的执行得以恢复。它现在执行约束验证查询，并发现`TX4`中的插入操作不被允许，因为它会使员工 1042 成为三个部门的经理。

你现在已经解决了序列化问题：两个都需要执行`DEPT_TAB01`验证查询的事务永远不可能同时执行。然而，所实现的锁定方案可能有点过于粗放。例如，如果`TX4`要插入一个由其他人管理的部门（即不是员工 1042），那么`TX4`也将被阻塞，而实际上在这种情况下，为了`DI`代码能正确实施约束，它并不需要被阻塞。

鉴于在代码清单 11-38 中对`EM6`的使用，你可以通过将对`p_request_lock`的调用改为以下形式，为此约束实现一个更细粒度的锁定方案：`p_request_lock('DEPT_TAB01'||to_char(r.mgr))`

你现在不是始终请求名称恒定的应用程序锁（`'DEPT_TAB01'`），而是请求一个名称取决于需要检查的*情况*的应用程序锁。现在，插入由不同员工管理的两个部门的两个事务被允许同时执行。

注意：只有`EM6`允许你实现这种更细粒度的锁定方案。

