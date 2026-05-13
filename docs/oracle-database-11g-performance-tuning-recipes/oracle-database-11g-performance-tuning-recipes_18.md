# 工作原理

当您收到 `ORA-01652` 错误时，通常的第一反应是检查临时表空间。您会查看 `DBA_TEMP_FREE_SPACE` 视图，发现默认临时表空间 `TEMP` 中有大量空闲空间。但是，如果您仔细查看错误信息，它会告诉您数据库无法在 `INDX_01` 表空间中扩展临时段。当您像在此例中一样创建索引时，您需要提供数据库创建新索引时必须使用的永久表空间的名称。Oracle 开始创建新索引时，会将新的索引结构放入您为索引指定的表空间（我们的例子中是 `INDX_01`）的一个临时段中。原因是，如果您的索引创建过程失败，Oracle（更具体地说，是 SMON 进程）会从您指定用于创建新索引的表空间中删除该临时段。一旦索引成功创建（或重建），Oracle 会将临时段转换为 `INDX_01` 表空间中的一个永久段。然而，只要 Oracle 仍在创建索引，数据库就会将其视为临时段，因此当索引创建失败时，数据库会发出 `ORA-01652` 错误，这也是临时表空间“空间不足”错误的错误代码。错误中提到的 `TEMP` 段，是指在构建过程中保存新索引的那段空间。一旦您增大了 `INDX_01` 表空间的大小，错误就会消失。

![images](img/square.jpg) `提示` `ORA-1652` 错误消息中的临时段可能并非指临时表空间中的临时段。

解决 `ORA-01652` 错误的关键在于理解，Oracle 在临时表空间之外的地方也会使用临时段。临时表空间中的临时段用于排序等活动，而永久表空间在执行创建表（`CTAS`）或索引所需的临时操作时，也可以使用临时段。

![images](img/square.jpg) `提示` 当您创建索引时，创建过程会使用两个不同的临时段。一个位于 `TEMP` 表空间的临时段用于对索引数据进行排序。另一个位于永久表空间的临时段则在索引创建期间保存索引。索引创建完成后，Oracle 会将索引表空间中的临时段更改为永久段。使用 `CREATE TABLE…AS SELECT`（`CTAS`）选项创建表时，情况也是如此。

如“解决方案”部分所述，`ORA-01652` 错误指的是您正在重建索引的表空间。如果您正在创建新索引，Oracle 会使用临时表空间对索引数据进行排序。在创建大型索引时，明智的做法可能是创建一个大型临时表空间并将其分配给正在创建索引的用户。索引创建完成后，您可以为该用户重新分配原始临时表空间，并删除那个大型临时表空间。此策略有助于避免为容纳大型索引的创建而将默认临时表空间扩展得非常大。

如果您为临时表空间指定了 `autoextend`，临时文件可能会变得非常大，这取决于数据库中一两次大型排序操作。当您尝试回收 `TEMP` 表空间的空间时，可能会遇到以下错误。

```sql
SQL> alter database tempfile '/u01/app/oracle/oradata/prod1/temp01.dbf' resize 500M;
alter database tempfile '/u01/app/oracle/oradata/prod1/temp01.dbf' resize 500M
*ERROR at line 1:
ORA-03297: file contains used data beyond requested RESIZE value
```

一种解决方案是创建一个新的临时表空间，将其设为默认临时表空间，然后删除那个较大的临时表空间。在 Oracle Database 11g 中，您可以通过使用以下 `alter tablespace` 命令来简化临时表空间的收缩操作：

```sql
SQL> alter tablespace temp shrink space;

Tablespace altered.

SQL>
```

在此示例中，我们收缩了整个临时表空间，但您可以通过执行命令 `alter tablespace temp shrink tempfile <file_name>` 来收缩特定的临时文件。该命令会将临时文件收缩到尽可能小的大小。

## 7-7. 解决 Open Cursor 错误

### 问题

您频繁遇到 `Maximum Open Cursors exceeded error`，并希望解决此错误。

### 解决方案

当您收到 `ORA-01000`：“maximum open cursors exceeded”错误时，首先需要做的一件事是检查初始化参数 `open_cursors` 的值。您可以通过执行以下命令查看当前打开的游标数限制：

```sql
SQL> sho parameter open_cursors

NAME                                 TYPE        VALUE
------------------------------------ ----------- ---------
open_cursors                         integer     300
SQL>
```

参数 `OPEN_CURSORS` 设置一个会话可以同时打开的最大游标数。您指定此参数以控制打开游标数。将该参数的值设置得太低将导致会话收到 `ORA-01000` 错误。为 `OPEN_CURSORS` 参数指定一个非常大的值通常没有害处（除非您预计所有会话会同时达到其游标上限，而这不太可能），因此您通常只需将参数值提高到一个较大的数字，即可解决与游标相关的错误。然而，有时您可能会发现提高 `open_cursors` 参数的值并不能“修复”问题。在这种情况下，请通过执行以下查询来调查哪些进程正在使用打开的游标：

```sql
SQL> select a.value, s.username,s.sid,s.serial#,s.program,s.inst_id
     from gv$sesstat a,gv$statname b,gv$session s
     where a.statistic# = b.statistic# and s.sid=a.sid
     and b.name='opened cursors current'
```

`GV$OPEN_CURSOR`（或 `V$OPEN_CURSOR`）视图显示每个用户会话当前已打开并解析或缓存的所有游标。您可以执行以下查询来识别具有大量已打开、已解析或已缓存游标的会话。

```sql
SQL> select saddr, sid, user_name, address,hash_value,sql_id, sql_text
     from gv$open_cursor
     where sid in
     (select sid from v$open_cursor
     group by sid having count(*)  > &threshold);
```

该查询列出所有打开游标数大于您指定阈值的会话。这样，您可以限制查询的输出，并专注于那些已打开、解析或缓存了大量游标的会话。

您可以通过执行以下查询获取特定会话的实际 SQL 代码和打开的游标数：

```sql
SQl> select sql_id,substr(sql_text,1,50) sql_text, count(*)
     from gv$open_cursor where sid=81
     group by sql_id,substr(sql_text,1,50)
     order by sql_id;
```

输出显示了 SID 为 81 的会话中所有打开游标的 SQL 代码。您可以检查所有具有高打开游标数的 SQL 语句，以了解该会话为何保持大量游标打开。


#### 工作原理

如果你的应用没有关闭打开的游标，那么将 `OPEN_CURSORS` 参数设置为更高的值对你帮助不大。你可能暂时解决了问题，但不久后很可能又会遇到同样的问题。如果应用层从不关闭由 PL/SQL 代码创建的 ref 游标，数据库将仅仅为已使用的游标持有服务器资源。你必须修复应用逻辑，使其关闭游标——问题其实不在于数据库。

如果你使用的是部署在应用服务器（如 Oracle WebLogic Server）上的 Java 应用，WebLogic Server 的 JDBC 连接池会为应用程序提供打开的数据库连接。这些连接中的每个预处理语句都将使用一个游标。多个应用服务器实例和多个 JDBC 连接池意味着数据库需要支持所有的游标。如果多个请求共享相同的会话 ID，打开的游标问题可能是由隐式游标引起的。此时唯一的解决方案是在每个请求后关闭连接。

`游标泄漏` 是指数据库打开游标但没有关闭它们。你可以为一个会话运行 10046 跟踪，以查明它是否关闭了其游标：

```
SQL> alter session set events '10046 trace name context forever, level 12';
```

如果你注意到同一条 SQL 语句与不同的游标相关联，那就意味着应用没有关闭其游标。如果应用在打开游标后不关闭它们，Oracle 会为其执行的下一条 SQL 语句分配不同的游标号。反之，如果游标被关闭，Oracle 将为分配的下一个游标重用相同的游标号。因此，如果你在 10046 跟踪的输出中看到 `PARSING IN CURSOR #nnnn` 项逐渐增加，则表明应用没有关闭游标。请注意，游标保持打开状态可能是由于有缺陷的应用设计，但开发者也可能有意让游标保持打开以减少软解析，或者当他们使用会话游标缓存时。

你可以使用 `SESSION_CACHED_CURSORS` 初始化参数来设置每个会话缓存的已关闭游标的最大数量。默认设置是 50。你可以使用此参数来防止一个会话打开过多游标，从而填满库缓存或强制进行过多的硬解析。对一条 SQL 语句重复进行解析调用会导致 Oracle 将该语句的会话游标移入会话游标缓存。数据库通过使用缓存的游标来满足后续的解析调用，而不是重新打开游标。

当你重新执行一条 SQL 语句时，Oracle 会首先尝试在共享池中查找该语句的已解析版本——如果在共享池中找到了已解析版本，则发生软解析。如果未在共享池中找到该语句的已解析版本，Oracle 将被迫执行开销大得多的硬解析。虽然软解析的开销远小于硬解析，但大量的软解析会影响性能，因为它们确实需要 CPU 使用和库缓存闩。为了减少软解析的数量，Oracle 将每个会话最近关闭的游标缓存在该会话的本地会话缓存中——Oracle 存储那些至少进行了三次解析调用的游标，从而避免了缓存每一个会话游标（这会填满游标缓存）。

`SESSION_CACHED_CURSORS` 初始化参数的默认值 50 对许多数据库来说可能太低了。你可以通过执行以下语句来检查数据库是否达到了会话缓存游标的上限：

```
SQL> select max(value) from v$sesstat
  2  where statistic# in (select statistic# from v$statname
  3  where name = 'session cursor cache count');

MAX(VALUE)
----------
        49

SQL>
```

该查询显示了过去已被缓存的会话游标的最大数量。由于这个数字（49）几乎与 `SESSION_CACHED_CURSORS` 参数的默认值（或你设置的值）相同，你必须将该参数的值设置为更大的数字。会话游标缓存使用共享池。如果你使用的是自动内存管理，在重置 `SESSION_CACHED_CURSORS` 参数后无需做其他事情——数据库会在必要时增加共享池大小。你可以通过执行以下查询来了解每个会话在其会话游标缓存中有多少游标：

```
SQL> select a.value,s.username,s.sid,s.serial#
  2  from v$sesstat a, v$statname b,v$session s
  3  where a.statistic#=b.statistic# and s.sid=a.sid
  4  and b.name='session cursor cache count';
```

### 7-8. 解决数据库挂起问题

#### 问题

你的数据库挂起了。用户无法登录，现有用户也无法完成其事务。具有 SYSDBA 权限的数据库管理员也可能无法登录到数据库。你需要找出导致数据库挂起的原因，并修复该问题。

## 解决方案

当遇到似乎已挂起的数据库时，请遵循以下通用步骤：

1.  检查你的`告警日志`，查看数据库是否报告了任何错误，这些错误可能指示了数据库挂起的原因。
2.  尝试获取`AWR`或`ASH`报告，或查询一些`ASH`视图，如第 5 章所述。你可能会在`AWR`报告的`负载概要`部分顶部注意到诸如`硬解析`之类的事件，这表明正是它导致了数据库变慢。
3.  单个`即席查询`确实有可能使整个数据库陷入瘫痪。试着找出一个或多个性能极差的`SQL`语句，它们可能导致了数据库挂起（或性能极差）。
4.  检查数据库是否存在阻塞性的`锁`以及`门`争用。
5.  检查服务器的内存使用情况以及`CPU`使用率。确保会话没有因为`PGA`大小设置过低而停滞，如第 3 章所述。
6.  不要忽视这一点：看起来很吓人的数据库挂起，可能仅仅是由`归档日志`目标空间填满这样简单的原因引起的。如果`归档目标`已满，数据库将挂起，新的用户连接将会失败。不过，你仍然可以以`SYS`用户身份连接，并且一旦通过移动一些已归档的重做日志文件在`归档目标`中腾出空间，数据库就能重新被用户访问。
7.  检查`快速恢复区`。当数据库无法将`闪回数据库`日志写入恢复区时，也会发生挂起。当`快速恢复区`填满时，数据库将不会处理新工作，也不会建立新的数据库连接。你可以通过`alter system set db_recovery_file_dest_size`命令增大恢复区来解决此问题。

如果你仍然无法解决数据库挂起的原因，那么你很可能遇到了一个真正挂起的数据库。在调查这样的数据库时，你有时可能会发现自己无法连接和登录。在这种情况下，请使用`prelim`选项登录到数据库。`prelim`选项不需要真正的数据库连接。以下是一个示例，展示了如何使用`prelim`选项登录数据库：

```
C:\app\ora\product\11.2.0\dbhome_1\bin>sqlplus /nolog

SQL*Plus: Release 11.2.0.1.0 Production on Sun Mar 27 10:43:31 2011

Copyright (c) 1982, 2010, Oracle.  All rights reserved.

SQL> set _prelim on
SQL> connect / as sysdba
Prelim connection established
SQL>
```

或者，你可以使用命令 `sqlplus -prelim "/ as sysdba"` 来使用 `-prelim` 选项登录。请注意，你使用 `nolog` 选项来打开一个 `SQL*Plus` 会话。如果你已经连接到数据库，则无法执行 `set _prelim on` 命令。一旦你如上所示建立了 `prelim` 连接，就可以执行 `oradebug hanganalyze` 命令来分析挂起的数据库，例如：

```
SQL> oradebug hanganalyze 3
Statement processed.
SQL>
```

在 Oracle `RAC` 环境中，指定带有附加选项的 `oradebug hanganalyze` 命令，如下所示：

```
SQL> oradebug setinst all
SQL> oradebug -g def hanganalyze 3
```

你可以重复执行几次 `oradebug hanganalyze` 命令，以生成不同进程状态的转储文件。

除了 `hanganalyze` 命令生成的转储文件外，Oracle 支持团队通常还会请求一个进程状态转储（也称为 `systemstate` 转储）来分析数据库挂起状况。`systemstate` 转储将报告进程正在做什么以及它们当前持有的资源。你可以通过执行以下命令集从非 `RAC` 系统获取 `systemstate` 转储。

```
SQL> oradebug setmypid
Statement processed.
SQL> oradebug dump systemstate 266
Statement processed.
SQL>
```

在 `RAC` 环境中，执行以下命令来获取 `systemstate` 转储：

```
SQL> oradebug setmypid
SQL> oradebug unlimit
SQL> oradebug -g all dump systemstate 266
```

请注意，与 `oradebug hanganalyze` 命令不同，你必须连接到一个进程。`setmypid` 选项指定了进程，在本例中是你自己的进程。你也可以指定一个非你自己的进程 ID，在这种情况下，你需要在执行 `dump systemstate` 命令之前，先执行 `oradebug setmypid <pid>` 命令。如果你尝试在未设置 `PID` 的情况下执行 `dump systemstate` 命令，将会收到错误：

```
SQL> oradebug dump systemstate 10
ORA-00074: no process has been specified
SQL>
```

你必须执行几次 `systemstate` 转储，每次转储之间间隔大约一分钟左右。Oracle 支持团队通常会请求多个 `systemstate` 转储以及 `hanganalyze` 命令生成的跟踪文件。


#### 工作原理

当你遇到一个“挂起”的数据库时，关键是要确定它是真的挂起了，还是只是运行缓慢。如果一两个用户抱怨查询运行缓慢，你需要使用第 5 章中描述的技术来分析他们的会话，以查看缓慢是由于阻塞会话还是 Oracle 等待事件引起的。如果有多个用户报告他们的工作进行缓慢，这可能是由于多种原因造成的，包括 CPU、内存（SGA 或 PGA）或其他系统资源问题。

在排查挂起数据库的故障时，首先应检查服务器的 CPU 使用率。如果你的服务器显示 100%的 CPU 利用率，或者正在交换或分页，那么问题可能根本不在数据库上。至于内存，如果服务器没有足够的空闲内存，新会话就无法连接到数据库。

![images](img/square.jpg) `Tip` “解决方案”部分中展示的`prelim`选项允许你在不打开会话的情况下连接到 SGA。因此，即使正常的 SQL*Plus 登录不起作用，你也可以“登录”到挂起的数据库。当你连接到 SGA 后启动的`oradebug`会话实际上会分析 SGA 中的内容，并将其转储到跟踪文件中。

真正的数据库挂起可能由多种原因引起，包括耗尽了 CPU 或内存等资源的系统，或者几个会话卡在等待某个资源（如锁）。虽然数据库可以自动解决会话之间的死锁（通过终止持有所需锁的其中一个会话），但当涉及内部内核级资源的栓锁或引脚时，Oracle 有时无法自动检测和解决内部死锁——这导致了 Oracle 支持团队所称的“真正的数据库挂起”。因此，真正的数据库挂起是一种内部死锁或多个进程之间的循环依赖。Oracle 支持团队通常会要求你提供`hanganalyze`跟踪文件和多个系统状态转储，以便他们诊断挂起的根本原因。在这种情况下，你甚至可能无法登录数据库。当你发现自己甚至无法登录数据库时，你的第一反应通常是尝试关闭并重启，这通常被称为“弹跳”数据库。不幸的是，虽然关闭并重启数据库可能“解决”问题，但它也会断开所有用户的连接——而且你仍然不知道到底是什么原因导致的问题。如果你决定要弹跳数据库，请先快速生成一些`hanganalyze`和`systemstate`转储。

![images](img/square.jpg) `Tip` 尽管有时可能令人不快，但如果你发现自己根本无法连接到挂起的数据库，那么请收集任何可能需要的跟踪转储，然后快速弹跳数据库，以便用户可以访问他们的应用程序。特别是当你处理的是由于内存问题而挂起的数据库时，弹跳实例可能会让事情迅速重新运转起来。

如果你发现数据库完全没有响应，并且你甚至无法使用`SYSDBA`特权登录数据库，你可以使用`prelim`选项登录数据库。`prelim`选项代表初步连接，它会启动一个 Oracle 进程并将该进程附加到 SGA 共享内存。然而，这不是一个完整或完全的连接，而是一个有限的连接，其中用于查询执行的结构没有被建立——因此，你甚至无法查询`V$`视图。不过，`prelim`选项允许你运行`oradebug`命令来获取用于诊断目的的错误转储堆栈。`hanganalyze`命令的输出可以告诉 Oracle 支持工程师你的数据库是否真的因为会话等待某个资源而挂起。该命令进行内部内核调用，以找出所有正在等待资源的会话，并显示阻塞会话和等待会话之间的关系。你可以通过`oradebug`命令或`alter session`语句指定的`hanganalyze`选项会生成有关挂起会话的详细信息。一旦你获得转储文件，Oracle 支持人员就可以分析它，并告知你数据库挂起的原因。

你可以在 1 到 10 之间的各个级别调用`hanganalyze`命令。级别 3 转储处于挂起（`IN_HANG`）状态的进程。通常不需要指定高于 3 的级别，因为更高的级别会产生冗长的报告，包含过多的进程细节。

![images](img/square.jpg) `Note` 你使用`hanganalyze`和`systemstate`命令创建的转储文件位于 ADR 的 trace 目录中。

注意，我们发出`oradebug`命令是为了获取级别为 266 的`systemstate`转储。级别 266（结合了产生短栈信息的级别 256 和级别 10）适用于 Oracle 9.2.0.6 及更高版本的发行版（早期的发行版使用`systemstate level 10`）。级别 266 允许你转储每个进程的短栈，这些是 Oracle 函数调用，可帮助 Oracle 开发团队确定是哪个 Oracle 函数导致了问题。短栈信息也有助于匹配代码中的已知错误。在 Solaris 和 Linux 系统上，你可以安全地指定级别 266，但在其他系统上，转储短栈可能需要很长时间。因此，对于其他操作系统，你可能希望坚持使用级别 10。

如果你能找出阻塞会话，你也可以只为该会话获取转储，使用命令`oradebug setospid nnnn`，其中`nnnn`是阻塞会话的 PID，然后调用`oradebug`命令，如下所示：

```
SQL> oradebug setospid  9999
SQL> oradebug unlimit
SQL> oradebug dump errorstack  3
```

请注意，你可以在正常会话（以及在`prelim`会话）中生成`hanganalyze`和`systemstate`转储，而无需使用`oradebug`命令。你可以通过`alter session`命令调用`hanganalyze`命令，如下所示。

```
SQL> alter session set  events 'immediate trace name hanganalyze level 3';
```

同样，你可以使用以下命令获取`systemstate`转储：

```
SQL> alter session set events 'immediate trace name SYSTEMSTATE level 10';
Session altered.
SQL>
```

`oradebug`和`systemstate`转储只是你可以收集的众多转储中的两种。使用`oradebug dumplist`命令查看你可以收集的各种错误转储。

```
SQL> oradebug dumplist
TRACE_BUFFER_ON
TRACE_BUFFER_OFF
LATCHES
PROCESSSTATE
SYSTEMSTATE
INSTANTIATIONSTATE
REFRESH_OS_STATS
CROSSIC
CONTEXTAREA
HANGDIAG_HEADER
HEAPDUMP
…
```

请注意，虽然你可以在编辑器中阅读一些转储文件，但这些文件主要是为了帮助 Oracle 支持专业人员排查数据库挂起情况。你对这些转储文件能做的有限，特别是当数据库挂起是由于 Oracle 错误或内核级锁引起时，除了将它们发送给 Oracle 支持团队进行分析之外。

### 7-9. 调用自动诊断存储库命令解释程序

#### Problem

你想调用自动诊断存储库命令解释程序（ADRCI）并使用自动诊断存储库（ADR）的各种组件。


### 解决方案

ADRCI 是一个帮助你管理 Oracle 诊断数据的工具。你可以在交互模式和批处理模式下使用 ADRCI 命令。

要在交互模式下启动 ADRCI，请在命令行输入 `adrci`，如下所示：

```
$ adrci
ADRCI: Release 11.2.0.1.0 - Production on Mon Mar 14 11:41:41 2011
Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.
ADR base = "c:\app\ora"
adrci>
```

只要 `PATH` 环境变量包含了 `ORACLE_HOME/bin/`，你就可以从任何目录发出 `adrci` 命令。你可以在 `adrci>` 提示符下输入每条命令，使用完该实用程序后，你可以键入 `EXIT` 或 `QUIT` 来退出。你可以通过在 ADRCI 命令行键入 `HELP` 来查看所有可用的 ADRCI 命令，如下所示：

```
adrci> HELP
  HELP [topic]
   Available Topics:
        CREATE REPORT
        ECHO
        EXIT
        HELP
        HOST
        IPS
…
        SHOW HOMES | HOME | HOMEPATH
        SHOW INCDIR
        SHOW INCIDENT
        SHOW PROBLEM
        SHOW REPORT
        SHOW TRACEFILE
        SPOOL

 There are other commands intended to be used directly by Oracle, type
 "HELP EXTENDED" to see the list
adrci>
```

你可以通过将命令名作为属性添加到 `HELP` 命令中，来获取单个 ADRCI 命令的详细信息。例如，以下是获取 `show tracefile` 命令语法的方法：

```
adrci> help show tracefile
  Usage: SHOW TRACEFILE [file1 file2 ...] [-rt | -t]
                         [-i inc1 inc2 ...] [-path path1 path2 ...]

  Purpose: List the qualified trace filenames.
…
  Options:
  Examples:
…
adrci>
```

你也可以通过将命令集成到脚本或批处理文件中，以批处理模式执行 ADRCI 命令。例如，如果你想从操作系统脚本中运行 ADRCI 命令 `SET HOMEPATH` 和 `SHOW ALERT`，请在 shell 脚本中包含以下内容：

```
SET HOMEPATH diag/rdbms/orcl/orcl; SHOW ALERT -term
```

假设你的脚本名是 `myscript.txt`。然后，你可以在操作系统 shell 脚本或批处理文件中执行以下命令来运行此脚本：

```
$ adrci script=myscript.txt
```

请注意，参数 `SCRIPT` 告诉 ADRCI 它必须执行文本文件 `myscript.txt` 中的命令。如果文本文件不在运行 shell 脚本或批处理文件的同一目录中，你必须提供保存文本文件的目录的路径。

要直接在命令行执行 ADRCI 命令，而不是先调用 ADRCI 并与 ADR 接口进行交互，请使用参数 `EXEC` 指定 ADRCI 命令，如下所示：

```
$ adrci EXEC="SHOW HOMES; SHOW INCIDENT"
```

此示例展示了如何通过在命令行执行 ADRCI 命令来包含两个 ADRCI 命令——`SHOW HOMES` 和 `SHOW INCIDENT`。


