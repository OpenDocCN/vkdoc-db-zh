# TM 锁配置

关于 TM 锁的一个有趣补充：系统中允许的 TM 锁总数是可由您配置的（详情请参阅 *Oracle Database Reference* 手册中的 `DML_LOCKS` 参数定义）。实际上，它可以被设置为零。这并不意味着您的数据库变成了只读数据库（无锁），而是意味着不允许执行 DDL。这在非常专业的应用场景中很有用，例如 RAC 实现，可以减少原本会发生的实例内协调量。您也可以使用 `ALTER TABLE <TABLENAME> DISABLE TABLE LOCK` 命令，以逐对象为基础移除获取 TM 锁的能力。这是一种快速降低意外删除表风险的方法，因为您在删除表之前必须重新启用表锁。它也可用于检测因我们之前讨论过的未索引外键而导致的表级锁。

## DDL 锁

在 DDL 操作期间，系统会自动对对象施加 DDL 锁，以防止其他会话对其进行更改。例如，如果我执行 DDL 操作 `ALTER TABLE T`，那么表 `T` *通常*会被放置一个排他 DDL 锁，阻止其他会话获取该表的 DDL 锁和 TM 锁。

> **注意**
>
> 在旧版本的 Oracle 中，`ALTER TABLE T` 会对其施加排他 DDL 锁。在此示例中，表 `T` 阻止其他会话执行 DDL 和获取 TM 锁（用于修改表内容）。现在，许多 `ALTER` 命令可以在线执行——不会阻止修改。

DDL 锁在 DDL 语句执行期间持有，并在执行完成后立即释放。这实际上是通过始终将 DDL 语句包裹在隐式提交（或提交/回滚对）中来完成的。因此，在 Oracle 中，DDL 总是会提交。每个 `CREATE`、`ALTER` 等语句的执行过程都类似于以下伪代码：

```
Begin
Commit;
DDL-STATEMENT
Commit;
Exception
When others then rollback;
End;
```

所以，即使 DDL 执行不成功，它也总是会提交。请注意，DDL 以提交开始。它首先提交，这样如果需要回滚，它不会回滚您的事务。如果您执行了 DDL，它将使您执行的所有未完成工作永久生效，即使 DDL 未成功。如果您需要执行 DDL，但又不希望它提交现有事务，您可以使用自治事务。

DDL 锁有三种类型：

*   *排他 DDL 锁*：这些锁阻止其他会话获取 DDL 锁或 `TM` (DML) 锁。这意味着您可以在 DDL 操作期间查询表，但不能以任何方式修改它。

*   *共享 DDL 锁*：这些锁保护被引用对象的结构不被其他会话修改，但允许修改数据。

*   *可中断的解析锁*：这些锁允许对象（例如共享池中缓存的查询计划）注册其对其他某个对象的依赖。如果您对该对象执行 DDL，Oracle 将检查已注册依赖关系的对象列表，并使它们无效。因此，这些锁是可中断的——它们不会阻止 DDL 发生。

大多数 DDL 会获取排他 DDL 锁。如果您发出类似这样的语句：

```
SQL> alter table t move;
```

在该语句执行期间，表 `T` 将不可用于修改。在此期间可以使用 `SELECT` 查询该表，但大多数其他操作将被阻止，包括所有其他 DDL 语句。在 Oracle 中，现在某些 DDL 操作可以在不获取 DDL 锁的情况下进行。例如，我可以执行以下操作：

```
SQL> create index t_idx on t(x) ONLINE;
```

`ONLINE` 关键字修改了实际构建索引的方法。Oracle 不再获取阻止数据修改的排他 DDL 锁，而只是尝试在表上获取一个低级别（模式 2）的 `TM` 锁。这将有效地阻止其他 DDL 发生，但会允许 DML 正常执行。Oracle 通过记录 DDL 语句执行期间对表所做的修改，并在 `CREATE` 操作完成时将这些更改应用到新索引来实现这一特性。这大大提高了数据的可用性。为了亲自验证这一点，您可以创建一个具有一定规模的表：

```
SQL> create table t as select * from all_objects;
表已创建。
SQL> select object_id from user_objects where object_name = 'T';
OBJECT_ID
```

然后针对该表运行创建索引：

```
SQL> create index t_idx on t(owner,object_type,object_name) ONLINE;
```

同时在另一个会话中运行此查询，以查看针对该新创建表所获取的锁（请记住，`ID1=244277` 是我示例中特有的，您需要使用*您自己的*对象 ID）：

```
SQL> select (select username
from v$session
where sid = v$lock.sid) username,
sid,
id1,
id2,
lmode,
request, block, v$lock.type
from v$lock
where id1 = 244277;
USERNAME            SID        ID1      ID2    LMODE    REQUEST    BLOCK TY
------------ ---------- ---------- -------- -------- -------- -------- --
EODA                 22     244277        0        3          0        0 DL
EODA                 22     244277        0        3          0        0 DL
EODA                 22     244277        0        2          0        0 TM
EODA                 22     244277        0        4          0        0 OD
```

因此，这里我们看到对我们的对象获取了四个锁。两个 `DL` 锁是*直接路径加载*锁。它们用于在索引创建期间防止对基础表进行直接路径加载（这当然意味着您不能同时进行直接路径加载*和*创建索引）。`OD` 锁是允许真正在线 DDL 的锁。在过去（旧版本的 Oracle），在线 DDL 如 `CREATE INDEX ONLINE` 并非 100% 在线。它会在 `CREATE` 语句的开始和结束时获取一个锁——阻止其他并发活动（对基础表数据的修改）。它*大部分在线*但并非*完全在线*。从 11g 开始，`CREATE INDEX ONLINE` 命令是完全在线的；它不需要在命令开始/结束时获取排他锁。实现这一特性的一部分工作就是引入了 `OD`（在线 DDL）锁；它在内部用于实现真正的在线 DDL 操作。

其他类型的 DDL 会获取共享 DDL 锁。当您创建存储的编译对象（如过程和视图）时，会对依赖对象获取这些锁。例如，如果您执行以下命令，在处理 `CREATE VIEW` 命令期间，将对 `EMP` 和 `DEPT` 都放置共享 DDL 锁：

```
Create view MyView
as
select emp.empno, emp.ename, dept.deptno, dept.dname
from emp, dept
where emp.deptno = dept.deptno;
```

您可以修改这些表的内容，但不能修改它们的结构。

最后一种类型的 DDL 锁是可中断的解析锁。当您的会话解析语句时，会针对该语句引用的每个对象获取一个解析锁。获取这些锁是为了允许在共享池中，如果被引用的对象以某种方式被删除或更改，那么已解析的、缓存的语句可以被作废（刷新）。

用于查看此信息的宝贵视图是 `DBA_DDL_LOCKS`。没有对应的 `V$` 视图。`DBA_DDL_LOCKS` 视图构建在更神秘的 `X$` 表之上，并且默认情况下可能未安装在您的数据库中。您可以通过运行位于目录 `[ORACLE_HOME]/rdbms/admin` 中的 `catblock.sql` 脚本来安装此视图和其他锁定视图。此脚本必须以用户 `SYS` 身份执行才能成功。一旦执行了此脚本，您就可以针对该视图运行查询。例如，在新连接的会话中，我可能会看到以下内容：


$ `sqlplus eoda/foo@PDB1`
`SQL> select session_id sid, owner, name, type,`
`mode_held held, mode_requested request`
`from dba_ddl_locks`
`where session_id = (select sid from v$mystat where rownum=1);`
`SID OWNER    NAME                  TYPE                 HELD      REQUEST`
`------ -------- --------------------- -------------------- --------- -------`
`286 SYS      DICTIONARY_OBJ_OWNER  Table/Procedure/Type Null      None`
`286 SYS      DBMS_APPLICATION_INFO Body                 Null      None`
`286 SYS      DICTIONARY_OBJ_TYPE   Table/Procedure/Type Null      None`
`286 SYS      DBMS_STANDARD         Table/Procedure/Type Null      None`
`286 SYS      DATABASE              18                   Null      None`
`286 SYS      DICTIONARY_OBJ_NAME   Table/Procedure/Type Null      None`
`286 SYS      IS_VPD_ENABLED        Table/Procedure/Type Null      None`
`286 SYS      IDGEN1$               Table/Procedure/Type Null      None`
`286          EODA                  73                   Share     None`
`286 EODA     EODA                  18                   Null      None`

这些都是我的会话正在锁定的对象。我对几个 `DBMS_*` 包持有可破解析锁。这是使用 SQL*Plus 的副作用；例如，当你初始登录时，它可能会调用 `DBMS_APPLICATION_INFO`（通过 `SET SERVEROUTPUT` 命令启用/禁用 `DBMS_OUTPUT`）。我在这里可能会看到同一个对象的多个副本；这很正常，它仅仅意味着我在共享池中使用了多个引用这些对象的东西。请注意，在此视图中，`OWNER` 列不是锁的所有者；相反，它是被锁定对象的所有者。这就是为什么你会看到许多 `SYS` 行。`SYS` 拥有这些包，但它们都“属于”我的会话。

要查看一个可破解析锁的实际效果，我们首先创建并运行一个存储过程 `P`：

```
SQL> create or replace procedure p
as
begin
null;
end;
/
Procedure created.
SQL> exec p
PL/SQL procedure successfully completed.
```

现在，过程 `P` 将出现在 `DBA_DDL_LOCKS` 视图中。我们对它持有一个解析锁：

```
SQL> select session_id sid, owner, name, type,
mode_held held, mode_requested request
from dba_ddl_locks
where session_id = (select sid from v$mystat where rownum=1)
/
SID  OWNER    NAME                  TYPE                 HELD     REQUEST
------ -------- --------------------- -------------------- -------- -------
22 EODA     P                     Table/Procedure/Type Null     None
...
22 SYS      DATABASE              18                   Null     None
9 rows selected.
```

然后我们重新编译我们的过程并再次查询该视图：

```
SQL> alter procedure p compile;
Procedure altered.
SQL> select session_id sid, owner, name, type,
mode_held held, mode_requested request
from dba_ddl_locks
where session_id = (select sid from v$mystat where rownum=1);
SID OWNER  NAME                   TYPE                  HELD     REQUEST
------ ------ --------------------- --------------------- -------- -------
22 SYS    DBMS_OUTPUT            Body                  Null        None
22 SYS    DBMS_OUTPUT            Table/Procedure/Type  Null        None
22 EODA   EODA                   18                    Null        None
22 SYS    DBMS_APPLICATION_INFO  Body                  Null        None
22 SYS    PLITBLM                Table/Procedure/Type  Null        None
22 SYS    DBMS_APPLICATION_INFO  Table/Procedure/Type  Null        None
22        EODA                   73                    Share       None
22 SYS    DATABASE               18                    Null        None
```

我们发现 `P` 现在从视图中消失了。我们的解析锁已被打破。

当你作为一名开发人员，发现测试或开发系统中的某些代码无法编译时——它会挂起并最终超时，这个视图就很有用。这表明其他人正在使用它（实际上正在运行它），你可以使用此视图来查看那个人可能是谁。对于对象的 `GRANT` 语句和其他类型的 DDL，也会发生同样的情况。例如，你无法对正在运行的过程授予 `EXECUTE` 权限。你可以使用相同的方法来发现潜在的阻塞者和等待者。

注意

Oracle 11*g* Release 2 及以上版本引入了 *特性* 基于版本的重定义 (EBR)。通过 EBR，你实际上可以授予 `EXECUTE` 权限和/或重编译数据库中的代码，而不会干扰当前执行代码的用户。EBR 允许你在同一个模式中同时拥有同一个存储过程的多个版本。这使你可以在一个新版本（edition）中处理该过程的副本，而不会与其他用户当前使用的过程版本产生冲突。然而，本书不会涵盖 EBR，只是在它改变规则时提及一下。

## Latches

`Latches`（闩）是轻量级的序列化设备，用于协调多用户对共享数据结构、对象和文件的访问。

闩是设计为持有时间极短的锁——例如，修改内存中数据结构所需的时间。它们用于保护某些内存结构，例如数据库块缓冲区高速缓存或共享池中的库高速缓存。闩通常在内部以 `willing to wait`（愿意等待）模式请求。这意味着如果闩不可用，请求会话将休眠一小段时间，稍后重试该操作。其他闩可能以 `immediate`（立即）模式请求，这在概念上类似于 `SELECT FOR UPDATE NOWAIT`，意味着该进程将去做其他事情，例如尝试获取可能空闲的等效同级闩，而不是坐着等待此闩变为可用。由于许多请求者可能同时等待一个闩，你可能会看到一些进程比其他进程等待更长时间。闩的分配相当随机，可以说基于运气的好坏。哪个会话在闩刚被释放后请求它，它就会获得它。没有闩等待者队列——只有一群等待者不断重试。

Oracle 使用原子指令如“test and set”（测试并设置）和“compare and swap”（比较并交换）来操作闩。由于设置和释放闩的指令是原子的，操作系统本身保证只有一个进程能够测试并设置闩，即使许多进程可能同时去获取它。由于该指令只有一条指令，因此可以非常快（但整个闩定算法本身是许多 CPU 指令）。闩的持有时间很短，并提供了一种在闩持有者异常死亡时进行清理的机制。这个清理过程将由 `PMON` 执行。

`Enqueues`（队列锁），我们之前讨论过，是另一种更复杂的序列化设备，在更新数据库表中的行等情况时使用。它们与闩的不同之处在于，它们允许请求者排队等待资源。对于闩请求，请求会话会立即被告知是否获得了闩。而对于队列锁，请求会话将被阻塞，直到它实际获得该锁。

注意

使用 `SELECT FOR UPDATE NOWAIT` 或 `WAIT [n]`，你可以选择不等待队列锁，如果你的会话将被阻塞的话，但如果你确实阻塞并等待了，你将在队列中等待。

因此，队列锁不如闩快，但它提供了闩无法提供的功能。队列锁可以在不同级别获取，因此你可以拥有许多共享锁和具有不同程度可共享性的锁。



#### 锁存“自旋”

关于锁存（Latch），我想强调的一点是：锁存是一种锁，锁是序列化设备，而序列化设备会抑制可扩展性。如果你的目标是构建一个在 Oracle 环境中扩展性良好的应用程序，就必须寻找并采用那些能将所需执行的锁存操作降至最低的方法与解决方案。

即便是看似简单的活动，例如解析一条 SQL 语句，也会在库缓存（Library Cache）以及共享池（Shared Pool）的相关结构上获取并释放成百上千次锁存。如果我们持有一个锁存，那么可能就有其他进程正在等待它。而当我们去获取一个锁存时，我们自己也很可能不得不等待。

等待锁存可能是一项代价高昂的操作。如果锁存无法立即获得，并且我们愿意等待它（大多数情况下我们确实会等待），那么在多 CPU 机器上，我们的会话将会进入“自旋”状态，在一个循环中反复尝试获取锁存。这样做的理由是上下文切换（即被踢出 CPU 然后必须重新回到 CPU）代价很高。因此，如果进程无法立即获得锁存，我们会保持在 CPU 上并立即重试，而不是直接休眠、放弃 CPU，等到以后需要重新被调度到 CPU 上时再尝试。我们希望锁存的持有者正在另一个 CPU 上忙于处理（并且因为锁存被设计为持有时间极短，这很有可能），并且会很快释放它。如果经过自旋并不断尝试后，我们仍然未能获得锁存，此时进程才会进入休眠，即脱离 CPU，让其他工作得以执行。这种休眠行为通常是由于多个会话同时请求同一个锁存所致；并非单个会话长时间持有它，而是因为太多会话在同一时刻需要它，而每个会话持有它的持续时间都很短。如果你频繁地执行某些短暂（快速）的操作，其累积效应是巨大的！获取一个锁存的伪代码可能如下所示：

```
Loop
for i in 1 .. 2000
loop
try to get latch
if got latch, return
if i = 1 then misses=misses+1
end loop
INCREMENT WAIT COUNT
sleep
Add WAIT TIME
End loop;
```

其逻辑是尝试获取锁存，如果失败，则增加未命中计数（miss count），这是一个我们可以在 Statspack 报告中或通过直接查询 `V$LATCH` 视图看到的统计信息。一旦进程发生未命中，它将循环尝试一定次数（一个未公开的参数控制着尝试次数，通常设为 2000 次），反复尝试获取锁存。如果在这期间某次尝试成功，则返回并继续处理。如果全部失败，进程将在增加该锁存的睡眠计数后，进入短暂休眠。醒来后，进程将重新开始整个过程。这意味着获取锁存的成本不仅仅是发生的“测试并设置”类型的操作，还包括我们在尝试获取锁存过程中消耗的大量 CPU。我们的系统会显得非常繁忙（消耗大量 CPU），但实际完成的工作却不多。

#### 衡量锁存共享资源的成本

作为一个例子，我们将研究锁存共享池的成本。我们会对比一个编写良好的程序（使用绑定变量）和一个编写得不那么好的程序（使用字面量 SQL 或每个语句使用唯一的 SQL）。为此，我们将使用一个非常小的 PL/SQL 程序，该程序只是登录到 Oracle 并在一个循环中执行 25,000 条不同的 `INSERT` 语句。我们将进行两组测试：第一组测试中，我们的 PL/SQL 程序不使用绑定变量；第二组测试中则使用。

为了评估这些程序在多用户环境中的行为，我选择使用 Statspack 来收集指标，步骤如下：

1.  执行一次 Statspack 快照，收集系统当前状态。
2.  运行 N 份程序副本，让每份程序 `INSERT` 到其自己的数据库表中，以避免因所有程序都尝试向单个表插入而产生的争用。
3.  在最后一个程序副本完成后，立即再获取一次快照。

然后，只需打印出 Statspack 报告，就能了解 N 份程序副本完成所需的时间、消耗了多少 CPU、发生了哪些主要的等待事件等等。

注意

为什么不使用 AWR（自动工作负载存储库）进行此分析？答案是因为每个人都能访问 Statspack，每个人都可以。它可能需要由你的 DBA 安装，但每个 Oracle 客户都可以使用它。我希望呈现的结果是每个人都能复现的。

这些测试是在启用了超线程的双 CPU 机器上进行的（使其看起来像是有四个 CPU）。既然有两个物理 CPU，你可能期望这里会有非常好的线性扩展性——也就是说，如果一个用户消耗一个单位的 CPU 来处理其插入操作，那么你可能期望两个用户将需要两个单位的 CPU。你会发现，这个前提听起来合理，但很可能并不准确（具体有多不准确取决于你的编程技术，你将看到）。如果我们的处理过程不需要任何共享资源，那么这个前提是正确的，但我们的过程将使用一个共享资源，即共享池。我们需要锁存共享池来解析 SQL 语句，并且我们需要锁存共享池，因为它是一个共享数据结构，我们不能在他人读取它时修改它，也不能在它被修改时读取它。

注意

我已经使用 Java、PL/SQL、Pro*C 和其他语言执行过这些测试。最终结果每次都基本相同。这个演示和讨论适用于所有语言和所有与数据库的接口。

##### 测试准备

为了进行测试，我们需要一个模式（一组表）来工作。我们将使用多个用户进行测试，并且最主要的是要测量由于锁存引起的争用，这意味着我们对测量由于多个会话向同一个数据库表插入而产生的争用不感兴趣。因此，我们将为每个用户创建一个表，并将这些表命名为 `T1` ... `T10`。例如：

```
$ sqlplus eoda/foo@PDB1
SQL> begin
for i in 1 .. 10
loop
for x in (select * from user_tables where table_name = 'T'||i )
loop
execute immediate 'drop table ' || x.table_name;
end loop;
execute immediate 'create table t' || i || ' ( x int )';
end loop;
end;
/
PL/SQL procedure successfully completed.
```

注意

必须安装 Statspack 才能运行本节中的示例。请确保你阅读了本书“引言”中“设置环境”部分的材料。其中包含安装 Statspack 的说明。

我们将在接下来的每次测试迭代之前运行此脚本，以重置我们的模式，并在多次运行同一测试时强制进行硬解析。在我们的测试中，将遵循以下步骤：

1.  运行 `statspack.snap`。
2.  立即启动 N 份我们的 PL/SQL 代码，N 的值从 1 到 10 不等，代表 1 到 10 个并发用户。
3.  等待所有 N 份副本完成。
4.  再次运行 `statspack.snap`。
5.  为最后两个 Statspack ID 生成 Statspack 报告。

以下测试运行所呈现的数字就是使用此技术收集的。


## 不使用绑定变量

首先，我们的 PL/SQL 代码将不使用绑定变量，而是使用字符串拼接来插入数据：

```
declare
begin
for i in 1 .. 25000 loop
begin
execute immediate
'insert into t' || &1 || ' values(' || i || ')';
exception
when no_data_found then null;
end;
end loop;
end;
/
```

为了实现自动化，前面的代码被放置在一个名为 `nb.sql` 的文本文件中。然后，下一段 SQL 会自动生成一个 shell 脚本，用于调用前面的 PL/SQL 以及调用 Statspack：

```
define NumUsers=&1
-- 创建用于运行并行会话的临时 shell 脚本 temp.sh
set echo off
set verify off
set feedback off
set serverout on
spool temp.sh
begin
dbms_output.put_line( 'echo exec statspack.snap | sqlplus / as sysdba' );
for i in 1 .. &NumUsers
loop
dbms_output.put_line( 'sqlplus eoda/foo@PDB1 @nb.sql ' || i ||' ' || chr(38) );
end loop;
dbms_output.put_line( 'wait' );
dbms_output.put_line( 'echo exec statspack.snap | sqlplus / as sysdba' );
end;
/
spool off
```

生成的 shell 脚本内容包含以下内容（根据你传递给前面代码的变量数量而变化）：

```
echo exec statspack.snap | sqlplus / as sysdba
sqlplus eoda/foo@PDB1 @nb.sql 1 &
wait
echo exec statspack.snap | sqlplus / as sysdba
```

然后，从生成该脚本的同一个 SQL*Plus 会话中执行 shell 脚本：

```
host /bin/bash temp.sh
```

你可以从本书的 GitHub 网站下载封装了此示例逻辑和代码的脚本（查看 `ch06` 文件夹下的 `nobinds` 子文件夹）。包含此代码的脚本名为 `nobind.sql`，执行方式如下：

```
$ sqlplus eoda/foo@PDB1
SQL> @nobind.sql 1
```

我首先在单用户模式下运行了测试，只有一个会话（即单独运行，没有其他活动的数据库会话）。Statspack 报告返回了以下信息：

```
Elapsed:       0.23 (mins) Av Act Sess:       0.7
DB time:       0.16 (mins)      DB CPU:       0.16 (mins)
Cache Sizes            Begin        End
~~~~~~~~~~~       ---------- ----------
Buffer Cache:     5,920M              Std Block Size:         8K
Shared Pool:     1,168M                  Log Buffer:     7,344K
Load Profile             Per Second    Per Transaction    Per Exec    Per Call
~~~~~~~~~~~~     ------------------  ----------------- ----------- -----------
...
Parses:            1,825.2           12,776.5
Hard parses:            1,796.9           12,578.0
...
Top 5 Timed Events                                                Avg %Total
~~~~~~~~~~~~~~~~~~                                               wait   Call
Event                                        Waits    Time (s)   (ms)   Time
------------------------------------- ------------ ----------- ------ ------
pman timer                                       5          15   3000   60.6
CPU time                                                     9          38.3
log file parallel write                         45           0      2     .3
direct path sync                                 8           0      7     .2
buffer busy waits                                1           0     38     .2
```

我包含了 SGA 配置以供参考，但相关的统计信息如下：

*   消耗时间（DB 时间）约为 10 秒（0.16 分钟）
*   每秒 1796.9 次硬解析
*   使用了 9 秒 CPU 时间

现在，如果我们同时运行两个这样的程序，我们可能会预期硬解析次数跳升至每秒约 3600 次（毕竟我们有多个 CPU 可用），CPU 时间翻倍至大约 22 秒。让我们看一下：

```
$ sqlplus eoda/foo@PDB1
SQL> @nobind.sql 2
Elapsed:       0.20 (mins) Av Act Sess:       1.8
DB time:       0.36 (mins)      DB CPU:       0.35 (mins)
...
Load Profile             Per Second    Per Transaction    Per Exec    Per Call
~~~~~~~~~~~~     ------------------  ----------------- ----------- -----------
...
Parses:            4,178.3           16,713.0
Hard parses:            4,167.7           16,670.7
...
Top 5 Timed Events                                               Avg %Total
~~~~~~~~~~~~~~~~~~                                              wait   Call
Event                                       Waits    Time (s)   (ms)   Time
------------------------------------ ------------ ----------- ------ ------
CPU time                                                   21          61.8
pman timer                                      4          12   3000   35.6
latch: shared pool                         17,586           1      0    2.0
log file parallel write                        19           0      4     .2
library cache: dependency mutex X               2           0     19     .1
```

我们发现硬解析次数上升了一点，但 CPU 时间增加了一倍多。这怎么可能呢？答案在于 Oracle 对闩锁的实现方式。在这台多 CPU 机器上，当我们无法立即获得一个闩锁时，我们会自旋（spin）。自旋这个动作本身就会消耗 CPU。进程 1 多次尝试获取共享池的闩锁，却发现进程 2 持有该闩锁，因此进程 1 必须自旋并等待（消耗 CPU）。反过来对进程 2 也是如此；它会多次发现进程 1 持有它所需资源的闩锁。因此，我们的大部分处理时间并非用于实际工作，而是等待资源变得可用。如果我们向下翻阅 Statspack 报告到“闩锁睡眠细分”（Latch Sleep Breakdown）报告，我们会发现以下信息：

```
Latch Name                 Requests      Misses     Sleeps      Gets
----------- ------------------------ ----------- ---------- ---------
shared pool               2,151,856     131,050     17,789   113,513
```

注意这里的 `SLEEPS` 列中出现的数字 17,789 吗？这个数字与前面的“前 5 大计时事件”报告中报告的 `WAITS` 数量非常接近。

**注意**

睡眠次数与等待次数`紧密`对应；这可能会让人感到疑惑。为什么不是`完全`对应？原因是获取快照的操作不是原子的；在 Statspack 快照期间，会执行一系列查询将统计信息收集到表中，而每个查询都是`在`一个稍微不同的时间点执行的。因此，等待事件指标是在获取闩锁详细信息之前稍早的时间收集的。

我们的“闩锁睡眠细分”报告向我们显示了尝试获取闩锁但在自旋循环中失败的次数。这意味着前 5 大报告只向我们展示了冰山一角——在前 5 大报告中，并未显示那 131,050 次未命中（这意味着我们曾自旋尝试获取闩锁）。检查完前 5 大报告后，我们可能不会想到这里存在严重的硬解析问题，尽管实际上问题非常严重。为了执行两个单位的工作，我们不得不使用超过两个单位的 CPU。这完全是由于我们需要那个共享资源——共享池。这就是闩锁的本质。

你可以看到，除非你了解其实现机制，否则诊断与闩锁相关的问题可能非常困难。仅通过使用前 5 大部分快速浏览 Statspack 报告，可能会导致我们忽视这样一个事实：我们面临一个相当糟糕的扩展性问题。只有通过深入研究 Statspack 报告的闩锁部分，我们才能看到手头的问题。


此外，通常无法确定系统使用的 CPU 时间中有多少是由于这种自旋（spinning）造成的——在查看双用户测试时，我们只知道使用了 21 秒的 CPU 时间，并且有 131,050 次未能获取共享池的闩锁（latch）。我们不知道每次获取闩锁失败时尝试自旋了多少次，因此我们无法真正衡量有多少 CPU 时间花在了自旋上，又有多少花在了处理上。我们需要多个数据点来推导这些信息。

在我们的测试中，由于有单用户示例可供比较，我们可以得出结论：大约有 3 秒左右的 CPU 时间花在了自旋等待闩锁、等待该资源上。我们能得出这个结论是因为我们知道单用户只需要 9 秒的 CPU 时间，那么两个单用户就需要 18 秒，而 21（总 CPU 秒数）减去 18 等于 3。

### 使用绑定变量

现在，我想探讨与上一节相同的情况，但这次使用一个在处理过程中显著减少闩锁使用的程序。我们将采用那个 PL/SQL 程序，并使用绑定变量对其进行编码。为了实现这一点，我们只需将硬编码的变量替换为绑定变量：

```sql
declare
begin
for i in 1 .. 25000 loop
begin
execute immediate
'insert into t1 values(:i)' using i;
exception
when no_data_found then null;
end;
end loop;
end;
/
```

此测试的所有代码都放在一个名为 `bind.sql` 的文件中（您可以从 GitHub 网站下载此代码），并按如下方式运行：

```bash
$ sqlplus eoda/foo@PDB1
SQL> @bind.sql 1
```

让我们像处理无绑定变量示例那样，查看单用户和双用户的 Statspack 报告。我们将在这里看到显著的差异。以下是单用户报告：

```
Elapsed:       0.03 (mins) Av Act Sess:       0.5
DB time:       0.02 (mins)      DB CPU:       0.02 (mins)
Load Profile         Per Second    Per Transaction    Per Exec    Per Call
~~~~~~~~~~~~  -----------------  ----------------- ----------- -----------
...
Parses:               46.0               46.0
Hard parses:                1.5                1.5
...
Top 5 Timed Events                                               Avg %Total
~~~~~~~~~~~~~~~~~~                                              wait   Call
Event                                       Waits    Time (s)   (ms)   Time
------------------------------------ ------------ ----------- ------ ------
pman timer                                      1           3   3000   74.2
CPU time                                                    1          24.0
log file parallel write                        14           0      3    1.0
LGWR worker group ordering                      3           0      5     .4
log file sync                                   4           0      3     .3
```

不使用绑定变量和使用绑定变量之间的差异相当显著！我们从无绑定变量示例中的 11 个 CPU 秒减少到了这里的 1 个 CPU 秒。从每秒 1796 次硬解析减少到大约每秒 1.5 次（根据我对 Statspack 工作原理的了解，其中大部分来自运行 Statspack）。甚至经过时间也从大约 0.23 分钟（14 秒）大幅减少到 0.03 分钟（2 秒）。当不使用绑定变量时，我们将大量 CPU 时间用于解析 SQL。这并不完全与闩锁相关，因为在不使用绑定变量时产生的大量 CPU 时间花在了解析和优化 SQL 上。解析 SQL 是非常耗费 CPU 的，但将大部分 CPU 用于某些事情（解析）——这些工作对我们来说并非真正有用，甚至是我们不需要执行的——代价是相当高昂的。

当我们进行双用户测试时，使用绑定变量的测试结果继续看起来好得多：

```
sqlplus eoda/foo@PDB1
SQL> @bind.sql 2
Elapsed:       0.03 (mins) Av Act Sess:       0.7
DB time:       0.02 (mins)      DB CPU:       0.02 (mins)
...
Load Profile         Per Second    Per Transaction    Per Exec    Per Call
~~~~~~~~~~~~ ------------------  ----------------- ----------- -----------
...
Parses:               53.0               35.3
Hard parses:                   1.0                0.7
...
Top 5 Timed Events                                                Avg %Total
~~~~~~~~~~~~~~~~~~                                               wait   Call
Event                                        Waits    Time (s)   (ms)   Time
----------------------------------------- -------- ----------- ------ ------
pman timer                                       1           3   3000   66.7
CPU time                                                     1          29.3
log file parallel write                         18           0      4    1.6
buffer busy waits                            2,101           0      0     .6
library cache lock                               2           0     10     .5
```

CPU 时间量与单用户测试用例报告的大致相同。

注意

由于四舍五入，1 个 CPU 秒实际上可能在 0 到 2 秒之间，而 3 秒实际上可能在 2 到 4 秒之间。

此外，两个使用绑定变量的用户所使用的 CPU 量，*远*低于单个不使用绑定变量的用户所需 CPU 量的一半！当我查看此 Statspack 报告中的闩锁报告时，我发现共享池和库缓存的竞争非常少，甚至不值得报告。事实上，深入挖掘发现，共享池闩锁甚至没有记录共享池请求，而不使用绑定变量的双用户测试则记录了超过 220 万次请求。

### 性能/可扩展性比较

表 6-1 总结了每种实现的 CPU 使用情况，以及当我们将用户数增加到两个以上时的闩锁结果。如您所见，使用较少闩锁（绑定）的解决方案将随着用户负载的增加而具有更好的可扩展性。

表 6-1

使用和不使用绑定变量时的 CPU 使用情况比较

| Users | CPU (Sec)/DB Time (Min) | Shared Pool Latch Requests |
| --- | --- | --- |
|   | No Binds | Binds | No Binds | Binds |
| --- | --- | --- | --- | --- |
| 1 | 11/0.22 | 1.0/0.02 | 0 | 0 |
| 2 | 21/0.21 | 1.0/0.02 | 2.2 million | 0 |
| 5 | 60/2.9 | 3.0/0.05 | 5.3 million | 0 |
| 10 | 130/9.52 | 4.0/0.19 | 10 million | 1179 |

十用户不使用绑定变量的测试花费了 130 个 CPU 秒，而使用绑定变量仅花费了 4 个 CPU 秒！有趣的观察是，十个使用绑定变量的用户（因此产生的闩锁请求非常少）所使用的硬件资源（CPU）量，与一个不使用绑定变量的用户（即过度使用闩锁或执行了超过需要的处理）大致相当。当您检查十个用户的结果时，情况相当惊人。您会看到，与使用绑定变量相比，不使用绑定变量所使用的硬件资源高出几个数量级。随着时间的推移，添加的用户越多，每个用户等待这些闩锁的时间就越长。然而，使用绑定变量的实现避免了过度使用闩锁，因此在扩展时没有受到不良影响。

作为一名开发人员，此测试的结果真正凸显了使用绑定变量的影响。在代码中不使用绑定变量会严重削弱应用程序的性能。作为一名 DBA，您需要了解如何识别未使用绑定变量的情况，并与应用程序团队合作以确保始终使用绑定变量。


### 互斥体

一个*互斥体*是一种序列化设备，非常类似于闩；事实上，mutex 这个名字代表*互斥*。它是数据库使用的另一种序列化工具。在服务器的许多地方，它被用来代替传统的闩。

互斥体与闩的不同之处在于，它的实现甚至更加轻量级。它需要更少的代码来实现，大约五分之一的指令（这通常导致更少的 CPU 请求），并且需要更少的内存来实现，大约七分之一的大小。互斥体除了更轻量级之外，在某些功能方面也稍弱。就像一个队列锁比一个闩重得多一样，一个闩也比一个互斥体重。但是，就像队列与闩的比较一样，闩在某些情况下可以比互斥体做更多的事情（就像队列在某些情况下可以比闩做更多的事情一样）。这意味着，并非每个闩都将被或应该被互斥体取代，正如并非每个队列锁都将被或应该被闩取代。

在阅读各种报告中的互斥体时，请记住它们是更轻量级的序列化设备。它们可能实现比闩更高的可扩展性（就像闩比队列更具可扩展性一样），但它们仍然是*一种序列化设备*。一般来说，如果你可以避免做需要互斥体的事情，你应该避免，原因与你尽可能避免请求闩相同。

### 手动锁定和用户定义锁

到目前为止，我们主要研究了 Oracle 为我们透明放置的锁。当我们更新一个表时，Oracle 会在其上放置一个 TM 锁，以防止其他会话删除该表（或者实际上执行大多数 DDL）。我们在修改的各种块上留有 TX 锁，以便其他人可以判断我们拥有哪些数据。数据库使用 DDL 锁来保护对象在我们自己更改它们时免受更改。它在内部使用闩和锁来保护其自身结构。

接下来，让我们看看如何参与其中一些锁定操作。我们的选择如下：

*   通过 SQL 语句手动锁定数据。
*   通过 `DBMS_LOCK` 包创建我们自己的锁。

以下各节简要讨论了为什么你可能想要做其中的每一项。

#### 手动锁定

事实上，我们已经看到几个我们可能想使用手动锁定的情况。`SELECT...FOR UPDATE` 语句是手动锁定数据的主要方法。我们在前面的例子中使用它来避免丢失更新问题，即一个会话会覆盖另一个会话的更改。我们看到它被用作一种序列化对详细记录访问的方法，以执行业务规则（例如，第 1 章中的资源调度器示例）。

我们也可以使用 `LOCK TABLE` 语句手动锁定数据。该语句很少使用，因为锁的粒度太粗。它只是锁定表，而不是表中的行。如果你开始修改行，它们将被正常锁定。因此，这不是一种节省资源的方法（正如在其他 RDBMS 中可能的那样）。如果你正在编写一个将影响给定表中大多数行的大型批处理更新，并且你希望确保没有人会阻塞你，你可能会使用 `LOCK TABLE IN EXCLUSIVE MODE` 语句。通过这种方式锁定表，你可以确保你的更新能够完成其所有工作而不会被其他事务阻塞。然而，包含 `LOCK TABLE` 语句的应用程序将是罕见的。

#### 创建你自己的锁

Oracle 实际上通过 `DBMS_LOCK` 包向开发人员公开了其内部使用的队列锁机制。你可能想知道为什么你想创建自己的锁。答案通常是特定于应用程序的。例如，你可以使用这个包来序列化对 Oracle 外部某些资源的访问。假设你正在使用 `UTL_FILE` 例程，该例程允许你写入服务器文件系统上的文件。你可能已经开发了一个公共消息例程，每个应用程序都调用它来记录消息。由于文件是外部的，Oracle 不会协调许多同时尝试修改它的用户。这时就需要 `DBMS_LOCK` 包。现在，在打开、写入和关闭文件之前，你将以独占模式请求一个以文件命名的锁，并在关闭文件后手动释放该锁。通过这种方式，一次只有一个人能够向此文件写入消息。其他人将排队等待。`DBMS_LOCK` 包允许你在使用完锁后手动释放它，或者在提交时自动放弃它，甚至只要你登录就保持它。

### 小结

本章涵盖了很多内容，有时可能令人挠头。虽然锁定相当直接，但其一些副作用并非如此。然而，理解这些问题至关重要。例如，如果你不知道 Oracle 在外键未编入索引时用于强制外键关系的表锁，那么你的应用程序将遭受性能不佳之苦。如果你不知道如何查看数据字典以查看谁锁定了谁，你可能永远无法找出原因。你只会假设数据库有时会挂起。有时我希望每次我能够通过简单地运行查询来检测未编入索引的外键并建议索引导致问题的那个键，从而解决无法解决的挂起问题时，我都能得到一美元。那我就会非常富有。

