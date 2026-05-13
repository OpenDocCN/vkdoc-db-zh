# 如何（以及如何不）开发数据库应用程序

```
SQL> set serverout on
SQL> declare
l_rec  t%rowtype;
begin
l_rec := get_first_unlocked_row;
dbms_output.put_line( 'I got row ' || l_rec.id || ', ' || l_rec.payload );
end;
/
I got row 2, payload 2
PL/SQL procedure successfully completed.
SQL> declare
pragma autonomous_transaction;
l_rec  t%rowtype;
begin
l_rec := get_first_unlocked_row;
dbms_output.put_line( 'I got row ' || l_rec.id || ', ' || l_rec.payload );
commit;
end;
/
I got row 4, payload 4
PL/SQL procedure successfully completed.
```

现在，在 Oracle 11g 及更高版本中，我们可以使用 `SKIP LOCKED` 子句来实现前面的逻辑。在下面的例子中，我们将再次运行两个并发事务，观察它们如何同时找到并锁定不同的记录：

```
SQL> declare
l_rec t%rowtype;
cursor c
is
select *
from t
where decode(processed_flag,'N','N') = 'N'
FOR UPDATE
SKIP LOCKED;
begin
open c;
fetch c into l_rec;
if ( c%found )
then
dbms_output.put_line( 'I got row ' || l_rec.id || ', ' || l_rec.payload );
end if;
close c;
end;
/
I got row 2, payload 2
PL/SQL procedure successfully completed.
SQL> declare
pragma autonomous_transaction;
l_rec t%rowtype;
cursor c
is
select *
from t
where decode(processed_flag,'N','N') = 'N'
FOR UPDATE
SKIP LOCKED;
begin
open c;
fetch c into l_rec;
if ( c%found )
then
dbms_output.put_line( 'I got row ' || l_rec.id || ', ' || l_rec.payload );
end if;
close c;
commit;
end;
/
I got row 4, payload 4
PL/SQL procedure successfully completed.
```

前面这两个“解决方案”都能帮助解决我的客户在处理消息时遇到的第二个序列化问题。但是，如果我的客户当初直接使用高级队列并调用 `DBMS_AQ.DEQUEUE`，解决方案会简单多少？为了解决消息生产者的序列化问题，我们不得不实现一个基于函数的索引。为了解决消费者的序列化问题，我们不得不使用那个基于函数的索引来检索记录并编写代码。所以，我们解决了他们的主要问题——这是由于他们没有完全理解自己所使用的工具，并且由于系统缺乏良好的监测机制，经过大量排查和研究才发现的。但我们尚未解决以下问题：

-   该应用程序的构建完全没有考虑数据库层面的扩展。
-   该应用程序正在执行的功能（队列表），数据库*已经以高度并发和可扩展的方式提供*。我指的是内置于数据库中的高级队列（AQ）软件，这正是他们试图重新发明的功能。
-   经验表明，*所有*调优工作中，80%到 90%（甚至更多！）应该在应用层面完成（通常是读写数据库的接口代码），而不是在数据库层面。
-   开发人员完全不知道这些（业务）组件在数据库中做了什么，也不知道该去哪里寻找潜在问题。

这个项目的问题远不止于此。我们还必须弄清楚以下几点：

-   如何在不改变 SQL 的情况下对其进行调优。一般来说，这非常困难。我们可以在一定程度上通过 SQL 配置文件（此选项需要 Oracle 调优包的许可）、扩展统计信息和自适应查询优化来实现这一“魔法”。但低效的 SQL 终究还是低效的 SQL。
-   如何衡量性能。
-   如何找出瓶颈所在。
-   如何以及应该索引什么。等等。

在这一周结束时，那些原本被隔绝于数据库之外的开发人员，对数据库实际能为他们提供的功能以及获取这些信息的便捷程度感到惊讶。最重要的是，他们看到了利用数据库特性能为应用程序性能带来多大的提升。最终，他们成功了——只是比原计划晚了几周。

我关于数据库功能强大之处的观点，并非是对 Hibernate、EJB 和容器管理持久化等工具或技术的批评。批评的是故意对数据库保持无知，不去了解它如何工作以及如何使用它。在这个案例中，使用的技术本身运行良好——是在开发人员对数据库本身有了一些深入了解之后。

归根结底，数据库通常是应用程序的基石。如果它工作不佳，其他一切都无足轻重。如果你有一个黑匣子，而它不工作，你能做什么？你能做的几乎只有看着它，纳闷为什么它工作得不好。你无法修复它；你无法调优它。很简单，你不理解它的工作原理——而选择陷入这种境地的正是你自己。另一种选择是我所倡导的方法：理解你的数据库，知道它如何工作，知道它能为你做什么，并充分利用它的潜力。

## 理解 Oracle 架构

我曾与许多运行大型生产应用程序的客户合作——这些应用程序是从其他数据库（例如 SQL Server）“移植”到 Oracle 上的。我之所以给“移植”打引号，是因为我所看到的大多数移植都反映了一种“我们要做最小的改动，以使我们的 SQL Server 代码能在 Oracle 上编译和执行”的思维方式。坦率地说，我最常遇到的正是源于这种思路的应用程序，因为它们最需要帮助。然而，我想明确一点，我并非在此抨击 SQL Server——恰恰相反！将一个 Oracle 应用程序尽可能少地改动就直接扔到 SQL Server 上，会产生同样性能糟糕的逆向代码；问题是双向的。

但在一个特定案例中，SQL Server 的架构以及你使用 SQL Server 的方式确实严重影响了 Oracle 的实现。宣称的目标是向上扩展，但这些人并不想真正迁移到另一个数据库。他们希望以尽可能少的工作量进行移植，因此在客户端和数据库层保持了基本相同的架构。这个决定有两个重要的后果：

-   Oracle 中的连接架构与 SQL Server 中的相同。
-   开发人员使用了字面量（非绑定）SQL。

这两个后果导致了一个无法支持所需用户负载的系统（数据库服务器的可用内存耗尽），以及一个性能极其低下的系统。


## 在 Oracle 中使用单一连接

目前，在 SQL Server 中，为每个要执行的并发语句打开一个数据库连接是一种非常普遍的做法。如果你要执行五个查询，在 SQL Server 中你可能会看到五个连接。相反，在 Oracle 中，无论你要执行 5 次还是 500 次查询，你期望打开的最大连接数应该是一个。因此，在 SQL Server 中常见的做法，在 Oracle 中不仅不被鼓励，而且是被积极劝阻的；拥有多个数据库连接（当你能只用一个时）是你根本不想做的事情。

但他们确实这样做了。一个简单的基于 Web 的应用程序会为每个网页打开 5、10、15 个甚至更多的连接，这意味着他们的服务器只能支持原本应该能支持的并发用户数量的 1/5、1/10 或 1/15。我给他们的建议是重新设计应用程序架构，使其能够利用一个连接来生成一个页面，而不是使用 5 到 15 个连接。这是唯一能真正解决问题的方案。你可以想象，这不是一个“好的，我们今天下午就搞定”那种解决方案。这是一个非平凡的解决方案，针对一个本可以在数据库移植阶段、当你还在代码中摸索和修改东西时最容易纠正的问题。此外，在推出到生产环境之前进行一个简单的扩展性测试，本可以在最终用户感受到痛苦之前就发现这类问题。

### 使用绑定变量

如果我写一本关于如何构建**不可扩展**的 Oracle 应用程序的书，“不要使用绑定变量”将是第一章和最后一章。不使用绑定变量是导致性能问题和阻碍可扩展性的主要原因——更不用说这是一个巨大的安全风险。Oracle 共享池（一个非常重要的共享内存数据结构）的运作方式在很大程度上是建立在开发者在大多数情况下使用绑定变量的基础上的。如果你想让一个事务型的 Oracle 实现运行缓慢，甚至完全停滞，只需要拒绝使用它们。

绑定变量是查询中的一个占位符。例如，要检索员工 `123` 的记录，我可以查询

```
SQL> select * from emp where empno = 123;
```

或者，我可以查询

```
SQL> select * from emp where empno = :empno;
```

在一个典型的系统中，你可能会查询员工 `123` 一两次，然后在很长一段时间内再也不会查询。之后，你可能会查询员工 `456`，然后是 `789`，依此类推。或者，放弃 `SELECT` 语句，如果你在插入语句中不使用绑定变量，你的主键值将是硬编码在其中的，而我知道这些插入语句以后肯定无法被重用！！！如果你在查询中使用字面量（常量），那么每个查询都是一个全新的查询，数据库以前从未见过。它必须被解析、限定（名称解析）、安全检查、优化等等。简而言之，你执行的每一个唯一的语句在每次执行时都必须被编译。

第二个查询使用了一个绑定变量 `:empno`，其值在查询执行时提供。这个查询被编译一次，然后查询计划被存储在一个共享池（库缓存）中，从中可以被检索和重用。两者在性能和可扩展性方面的差异是巨大的，甚至是戏剧性的。

从前面的描述来看，应该相当明显，解析带有硬编码变量的唯一语句（称为**硬解析**）会比重用已经解析过的查询计划（称为**软解析**）花费更长的时间并消耗更多的资源。可能不那么明显的是前者会多大程度地减少你的系统所能支持的用户数量。显然，这部分是由于资源消耗的增加，但一个更重要的因素源于库缓存的闩锁机制。当你硬解析一个查询时，数据库会花费更多时间持有某些称为**闩锁**的低级序列化设备（更多细节请参见第 6 章）。这些闩锁保护 Oracle 共享内存中的数据结构免受两个会话的并发修改（否则，Oracle 最终会出现损坏的数据结构），并防止在数据结构被修改时有人读取它。你闩锁这些数据结构的时间越长、频率越高，获取这些闩锁的队列就会变得越长。你将开始独占稀缺资源。你的机器有时可能看起来未被充分利用，然而数据库中的所有事情都运行得非常慢。很可能有人正持有其中一个序列化机制，并且排起了队——你无法以最高速度运行。你的数据库中只需要一个行为不佳的应用程序就会显著影响其他每个应用程序的性能。一个不使用绑定变量的单一小型应用程序，将导致其他经过良好调整的应用程序的相关 SQL 随时间推移从共享池中被丢弃。你只需要一个坏苹果就能毁掉整桶水果。

## 使用绑定变量优化数据库性能

如果你使用绑定变量，那么每个提交相同查询并引用相同对象的人都将使用池中已编译的执行计划。你只需编译一次子程序，就可以反复使用。这种方式非常高效，也是数据库期望你采用的工作方式。这样不仅会减少资源消耗（软解析的资源密集度要低得多），而且持有 latch 的时间会更短、频率会更低。这提升了性能，并极大地增强了可扩展性。

为了让你对性能提升的巨大差异有一个直观的认识，只需要运行一个很小的测试。在这个测试中，我们将向一个表中插入一些行；我们将使用一个简单的表：

```
SQL> drop table t purge;
SQL> create table t ( x int );
Table created.
```

现在我们将创建两个非常简单的存储过程。它们都会向这个表中插入数字 1 到 10,000；然而，第一个过程使用一条带有绑定变量的 SQL 语句：

```
SQL> create or replace procedure proc1
as
begin
for i in 1 .. 10000
loop
execute immediate
'insert into t values ( :x )' using i;
end loop;
end;
/
Procedure created.
```

第二个过程为要插入的每一行都构建一个唯一的 SQL 语句：

```
SQL> create or replace procedure proc2
as
begin
for i in 1 .. 10000
loop
execute immediate
'insert into t values ( '||i||')';
end loop;
end;
/
Procedure created.
```

现在，两者之间的唯一区别在于一个使用了绑定变量，而另一个没有。两者都使用了动态 SQL，其他逻辑完全相同。唯一的区别就是第一个过程中使用了绑定变量。

**注意**

关于 `runstats` 和其他实用程序的详细信息，请参阅本书开头的“设置你的环境”部分。你可能不会观察到 CPU 或任何指标的数值完全相同。差异是由不同的 Oracle 版本、不同的操作系统和不同的硬件平台引起的。核心思想是一样的，但具体的数字无疑会略有不同。

我们准备评估这两种方法，并将使用我开发的一个简单工具 `runstats` 来详细比较两者：

```
SQL> set serverout on
SQL> exec runstats_pkg.rs_start
PL/SQL procedure successfully completed.
SQL> exec proc1
PL/SQL procedure successfully completed.
SQL> exec runstats_pkg.rs_middle
PL/SQL procedure successfully completed.
SQL> exec proc2
PL/SQL procedure successfully completed.
SQL> exec runstats_pkg.rs_stop(9500)
Run1 ran in 20 cpu hsecs
Run2 ran in 709 cpu hsecs
run 1 ran in 2.82% of the time
```

现在，前面的结果清楚地显示，基于 CPU 时间（以百分之一秒为单位），不使用绑定变量插入 10,000 行所花费的时间和资源明显多于使用它们的情况。事实上，不使用绑定变量插入行所花费的 CPU 时间多了一个数量级以上。对于每一个不使用绑定变量的插入操作，我们花费绝大部分时间仅仅是在**解析**语句！但情况还会更糟。当我们查看其他信息时，可以看到两种方法在资源利用方面存在显著差异：

```
Name                                      Run1            Run2            Diff
LATCH.KJCT flow control latch              202           9,909           9,707
LATCH.object stats modificatio               9           9,768           9,759
STAT...session logical reads            10,827          20,671           9,844
STAT...db block gets from cach          10,587          20,442           9,855
STAT...db block gets                    10,587          20,442           9,855
STAT...db block gets from cach          10,448          20,318           9,870
STAT...calls to kcmgcs                     164          10,128           9,964
STAT...enqueue requests                     44          10,017           9,973
STAT...enqueue releases                     42          10,017           9,975
STAT...ASSM gsp:L1 bitmaps exa              21          10,004           9,983
STAT...ASSM gsp:get free block              16          10,000           9,984
STAT...ASSM cbk:blocks examine              16          10,000           9,984
STAT...ASSM gsp:good hint                   15          10,000           9,985
STAT...calls to get snapshot s              47          10,040           9,993
STAT...parse count (hard)                    3          10,003          10,000
STAT...session cursor cache hi          10,030              29         -10,001
LATCH.shardgroup list latch                  3          10,005          10,002
STAT...parse count (total)                  24          10,028          10,004
LATCH.session idle bit                      33          14,963          14,930
LATCH.checkpoint queue latch            19,403           4,230         -15,173
LATCH.gcs resource hash                    327          16,879          16,552
LATCH.name-service namespace b             106          18,200          18,094
LATCH.PDB Hash Table Latch                   8          18,579          18,571
LATCH.parallel query alloc buf               1          18,739          18,738
LATCH.object queue header oper           1,783          22,378          20,595
LATCH.object queue header free              38          21,643          21,605
LATCH.shared pool simulator                 21          21,735          21,714
STAT...file io wait time                32,001           8,369         -23,632
LATCH.ASM map operation hash t             570          27,386          26,816
LATCH.gc element                           501          28,119          27,618
LATCH.post/wait queue                    2,824          32,126          29,302
STAT...recursive calls                  10,113          40,202          30,089
LATCH.JS queue state obj latch           3,672          34,884          31,212
LATCH.gcs partitioned table ha             210          33,281          33,071
STAT...global enqueue releases             454          60,423          59,969
STAT...global enqueue gets syn             455          60,426          59,971
LATCH.active service list                3,698          64,116          60,418
LATCH.pdb domain request queue           4,464          81,550          77,086
LATCH.enqueue hash chains                8,053         116,473         108,420
LATCH.ksim group membership ca              41         155,915         155,874
STAT...cell physical IO interc         655,360         851,968         196,608
STAT...physical read total byt         655,360         851,968         196,608
STAT...physical read bytes             655,360         851,968         196,608
LATCH.cache buffers chains              55,873         272,739         216,866
LATCH.ges group table                    5,065         223,153         218,088
STAT...physical read total byt         630,784         851,968         221,184
LATCH.ges domain table                   6,410         277,980         271,570
LATCH.recovery domain hash buc          15,873         290,222         274,349
LATCH.ges resource hash list             6,829         295,246         288,417
LATCH.process queue reference                1         326,406         326,405
LATCH.ges process parent latch          10,099         435,392         425,293
LATCH.shared pool                          295         517,197         516,902
STAT...session uga memory                    0        -523,840        -523,840
STAT...session pga memory              -65,536        -720,896        -655,360
STAT...logical read bytes from      88,612,864     169,402,368      80,789,504
Run1 latches total versus runs -- difference and pct
Run1               Run2              Diff        Pct
156,002         3,498,539         3,342,537      4.46%
```


