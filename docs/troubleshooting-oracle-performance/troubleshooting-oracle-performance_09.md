# Oracle SQL 跟踪格式与调试事件

## 跟踪输出示例

```
...
...
PARSING IN CURSOR #1 len=142 dep=1 uid=28 oct=3 lid=28 tim=1156387084566620 hv=1624534809 ad='6f8a7620'
SELECT CUST_ID, EXTRACT(YEAR FROM TIME_ID), SUM(AMOUNT_SOLD) FROM SH.SALES
WHERE CHANNEL_ID = :B1 GROUP BY CUST_ID, EXTRACT(YEAR FROM TIME_ID)
END OF STMT
PARSE #1:c=0,e=93,p=0,cr=0,cu=0,mis=0,r=0,dep=1,og=1,tim=1156387084566617
BINDS #1:
kkscoacd
  Bind#0
   oacdty=02 mxl=22(21) mxlc=00 mal=00 scl=00 pre=00
   oacflg=03 fl2=1206001 frm=00 csi=00 siz=24 off=0
   kxsbbbfp=2a9721f070  bln=22  avl=02 flg=05
   value=3
EXEC #1:c=1000,e=217,p=0,cr=0,cu=0,mis=0,r=0,dep=1,og=1,tim=1156387084566889
WAIT #1: nam='db file sequential read' ela= 19333 file#=4 block#=211 blocks=1 obj#=10293 tim=1156387084610301
WAIT #1: nam='db file sequential read' ela= 2962 file#=4 block#=219 blocks=1 obj#=10294 tim=1156387084613517
...
...
WAIT #2: nam='SQL*Net message from client' ela= 978 driver id=1413697536#bytes=1 p3=0 obj#=10320 tim=1156387086475763
STAT #1 id=1 cnt=16348 pid=0 pos=1 obj=0 op='HASH GROUP BY (cr=1720 pr=2588 pw=941 time=1830257 us)'
STAT #1 id=2 cnt=540328 pid=1 pos=1 obj=0 op='PARTITION RANGE ALL PARTITION: 1 28 (cr=1720 pr=1647 pw=0 time=1129471 us)'
STAT #1 id=3 cnt=540328 pid=2 pos=1 obj=10292 op='TABLE ACCESS FULL SALES PARTITION: 1 28 (cr=1720 pr=1647 pw=0 time=635959 us)'
WAIT #0: nam='SQL*Net message to client' ela= 1 driver id=1413697536#bytes=1 p3=0 obj#=10320 tim=1156387086475975
```

## 跟踪文件中的关键标记

在此摘录中，一些描述所提供信息类型的标记以粗体显示：

*   `PARSING IN CURSOR` 和 `END OF STMT` 围绕着 SQL 语句的文本。
*   `PARSE`、`EXEC` 和 `FETCH` 分别代表解析、执行和获取调用。
*   `BINDS` 表示绑定变量的定义和值。
*   `WAIT` 表示处理过程中发生的等待事件。
*   `STAT` 表示发生的执行计划和相关的统计信息。

您可以在 MetaLink 注释 **Interpreting Raw SQL_TRACE and DBMS_SUPPORT.START_TRACE output** (39817.1) 中找到跟踪文件格式的简短描述。如果您对此主题的详细描述和讨论感兴趣，Millsap/Holt 的书 **Optimizing Oracle Performance** (O'Reilly, 2003) 值得一读。

## SQL 跟踪的内部机制与调试事件

在内部，SQL 跟踪基于调试事件 10046。表 3-2 描述了支持的级别，这些级别定义了跟踪文件中提供信息的数量。当 SQL 跟踪在高于 1 的级别使用时，它也被称为 **扩展 SQL 跟踪**。

**表 3-2. 调试事件 10046 的级别**

| 级别 | 描述 |
| --- | --- |
| 0 | 调试事件被禁用。 |
| 1 | 启用调试事件。对于每个处理的数据库调用，提供以下信息：SQL 语句、响应时间、服务时间、处理的行数、逻辑读次数、物理读和写次数、执行计划以及少量额外信息。 |
| 4 | 与级别 1 相同，包含关于绑定变量的附加信息。主要是每个执行使用的数据类型、其精度和值。 |
| 8 | 与级别 1 相同，加上关于等待时间的详细信息。对于处理过程中经历的每个等待，提供以下信息：等待事件的名称、持续时间和一些标识所等待资源的附加参数。 |
| 12 | 同时是级别 4 和级别 8。 |

接下来的章节将描述如何启用和禁用 SQL 跟踪，如何配置环境以获得最佳优势，以及如何找到它生成的跟踪文件。

## 调试事件

调试事件由数字标识，是用于在运行的数据库引擎进程中设置一种标志的方式。其目的是改变其行为，例如，通过启用或禁用某个功能、通过测试或模拟损坏或崩溃，或通过收集跟踪或调试信息。一些调试事件不是简单的标志，可以在多个级别启用。每个级别都有其自己的行为。在某些情况下，级别是块或内存结构的地址。

您应谨慎使用调试事件，并且仅在 Oracle 支持人员指示或您知道并理解调试事件将要更改什么时才设置它。调试事件会启用特定的代码路径。因此，如果在设置调试事件时出现问题，值得检查在不设置调试事件的情况下是否可以重现相同的问题。

Oracle 文档中记录的调试事件很少。如果存在文档，通常通过 MetaLink 注释提供。换句话说，调试事件通常不会在关于数据库引擎的官方 Oracle 文档中描述。您可以在文件 `` `$ORACLE_HOME/rdbms/mesg/oraus.msg` `` 中找到可用调试事件的完整列表。请注意，此文件并非在所有平台上都分发。从 10000 到 10999 的范围保留给调试事件。

## 传统方式启用 SQL 跟踪

直到 Oracle9*i*，文档（特别是 **Database Performance Tuning Guide and Reference** 手册）描述了三种启用 SQL 跟踪的方法：初始化参数 `sql_trace`、包 `dbms_session` 中的过程 `set_sql_trace` 以及包 `dbms_system` 中的过程 `set_sql_trace_in_session`。关于这三种方法需要注意的重要一点是，它们都只能在级别 1 启用 SQL 跟踪。不幸的是，这在实践中是不够的。事实上，在大多数情况下，您需要完全分解响应时间以了解瓶颈在哪里。因此，我不会在这里描述这三种方法。相反，我将介绍其他未文档化⁵的方法，用于在任何级别启用 SQL 跟踪。请注意，从 Oracle Database 10*g* 开始，引入了一种有文档记录且官方支持的启用和禁用 SQL 跟踪的方法。因此，从 Oracle Database 10*g* 开始，不再需要使用本节中描述的方法。但是，如果您使用的是早期版本，您可以放心地利用这些方法，因为它们多年来已被成功使用。

要在任何级别启用和禁用 SQL 跟踪，有两种主要方法。要么通过执行 SQL 语句 `ALTER SESSION` 来设置参数 `events`，要么调用包 `dbms_system` 中的过程 `set_ev`。前者显然只能为执行它的会话启用 SQL 跟踪，而后者能够通过会话 ID 和序列号在任何会话中启用 SQL 跟踪。以下是一些使用示例。

以下 SQL 语句为执行它的会话在级别 12 启用 SQL 跟踪。注意事件号和级别是如何指定的。

```sql
ALTER SESSION SET events '10046 trace name context forever, level 12'
```

以下 SQL 语句为执行它的会话禁用 SQL 跟踪。注意，这是通过指定级别 0 来实现的。

```sql
ALTER SESSION SET events '10046 trace name context off'
```

以下 PL/SQL 调用为 ID 为 127、序列号为 29 的会话在级别 12 启用 SQL 跟踪。没有参数具有默认值。因此，最后一个参数，即使在此情况下不相关，也必须指定。

```sql
dbms_system.set_ev(si => 127,    -- session id
                   se => 29,     -- serial number
                   ev => 10046,  -- event number
                   le => 12,     -- level
                   nm => NULL)
```


以下 PL/SQL 调用为 ID 为 127、序列号为 29 的会话禁用 SQL 跟踪。请注意，与之前的情况相比，仅指定了跟踪级别的参数值发生了变化。

```
dbms_system.set_ev(si => 127,    -- 会话 ID
                   se => 29,     -- 序列号
                   ev => 10046,  -- 事件号
                   le => 0,      -- 级别
                   nm => NULL)
```

你可以通过执行以下查询，列出连接到实例的每个用户的会话 ID 和序列号：

```
SELECT sid, serial#, username, machine
FROM v$session
WHERE type != 'BACKGROUND'
```

你也可以通过执行 SQL 语句 `ALTER SYSTEM` 来设置初始化参数 `events`。其语法与 SQL 语句 `ALTER SESSION` 相同。无论如何，在实例级别启用 SQL 跟踪通常没有意义。另外请注意，它仅对执行后创建的会话生效。

默认情况下，只有用户 `sys` 可以执行包 `dbms_system`。如果需要将该执行权限提供给其他用户，请务必小心，因为该包包含其他过程，而且过程 `set_ev` 本身也可用于设置其他事件。如果你确实需要为其他用户提供在任何会话中启用和禁用 SQL 跟踪的能力，建议使用另一个专门包含用于启用和禁用 SQL 跟踪所需过程的包。Oracle 提供了这样一个包 `dbms_support`，但默认不安装。你应该注意到，它仅在 MetaLink 注释 *The DBMS_SUPPORT Package* (62294.1) 和 *Tracing Sessions in Oracle Using the DBMS_SUPPORT Package* (62160.1) 中有文档说明，并且不受官方支持。以下是一些示例。

要安装包 `dbms_support`，为其创建一个公共同义词，并授予角色 `dba` 执行它的权限。你可以使用以下命令：

```
CONNECT / as sysdba
@?/rdbms/admin/dbmssupp.sql
CREATE PUBLIC SYNONYM dbms_support FOR dbms_support;
GRANT EXECUTE ON dbms_support TO dbma;
```

从 Oracle Database 10*g* 开始，确实不应再使用此包。你应该转而使用下一节中解释的技术。不过，如果你在启用 PL/SQL 警告的情况下安装它（例如，通过将初始化参数 `plsql_warnings` 设置为 `enable:all`），则在创建包主体时会生成以下警告。你可以简单地忽略它。

```
PLW-05005: 函数 CURRENT_SERIAL 在第 29 行返回时没有值
```

以下 PL/SQL 调用为 ID 为 127、序列号为 29 的会话在级别 8 启用 SQL 跟踪。请注意，你无需指定要启用 SQL 跟踪的级别，而是通过第三和第四个参数分别指定是否需要等待事件和绑定变量。虽然前两个参数没有默认值，但参数 `waits` 默认为 `TRUE`，参数 `binds` 默认为 `FALSE`。

```
dbms_support.start_trace_in_session(sid     => 127,
                                    serial  => 29,
                                    waits   => TRUE,
                                    binds   => FALSE)
```

以下 PL/SQL 调用为 ID 为 127、序列号为 29 的会话禁用 SQL 跟踪。两个参数均没有默认值。

```
dbms_support.stop_trace_in_session(sid    => 127,
                                   serial => 29)
```

`启用 SQL 跟踪：现代方法`

从 Oracle Database 10*g* 开始，提供了包 `dbms_monitor` 用于启用和禁用 SQL 跟踪。有了这个包，你不仅终于有了一个官方的方式来充分利用 SQL 跟踪，更重要的是，你可以基于会话属性（参见"数据库调用"部分）来启用和禁用 SQL 跟踪：客户端标识符、服务名、模块名和操作名。这意味着，如果应用程序被正确地插桩，你可以独立于用于执行数据库调用的会话来启用和禁用 SQL 跟踪。如今，这尤其有用，因为在许多情况下会使用连接池，因此用户并不绑定于特定的会话。

以下是使用包 `dbms_monitor` 在会话、客户端、组件和数据库级别启用和禁用 SQL 跟踪的一些示例。请注意，默认情况下，只有启用了 `dba` 角色的用户才允许执行包 `dbms_monitor` 提供的过程。

`会话级别`

要为会话启用和禁用 SQL 跟踪，包 `dbms_monitor` 分别提供了过程 `session_trace_enable` 和 `session_trace_disable`。

以下 PL/SQL 调用为 ID 为 127、序列号为 29 的会话在级别 8 启用 SQL 跟踪。所有参数都有默认值。如果未指定标识会话的两个参数，则将为执行 PL/SQL 调用的会话启用 SQL 跟踪。参数 `waits` 默认为 `TRUE`，参数 `binds` 默认为 `FALSE`。

```
dbms_monitor.session_trace_enable(session_id  => 127,
                                  serial_num  => 29,
                                  waits       => TRUE,
                                  binds       => FALSE)
```

从 Oracle Database 10*g* Release 2 开始，当使用过程 `session_trace_enable` 启用 SQL 跟踪时，视图 `v$session` 的列 `sql_trace`、`sql_trace_waits` 和 `sql_trace_binds` 会相应地设置。警告：这仅在使用过程 `session_trace_enable` 并且被跟踪的会话至少执行了一条 SQL 语句时才会发生。例如，使用前面的 PL/SQL 调用启用 SQL 跟跟踪将导致给出以下信息：

```
SQL> SELECT sql_trace, sql_trace_waits, sql_trace_binds
  2  FROM v$session
  3  WHERE sid = 127;

SQL_TRACE       SQL_TRACE_WAITS SQL_TRACE_BINDS
--------------- --------------- ---------------
ENABLED         TRUE            FALSE
```

以下 PL/SQL 调用为 ID 为 127、序列号为 29 的会话禁用 SQL 跟踪。请注意，两个参数均有默认值。如果未指定，则将为执行 PL/SQL 调用的会话禁用 SQL 跟踪。

```
dbms_monitor.session_trace_disable(session_id => 127,
                                   serial_num => 29)
```

请注意，如果使用了真正应用集群，则必须在会话所在的实例上执行过程 `session_trace_enable` 和 `session_trace_disable`。

`客户端级别`

要为客户端启用和禁用 SQL 跟踪，包 `dbms_monitor` 分别提供了过程 `client_id_trace_enable` 和 `client_id_trace_disable`。当然，这些过程仅在设置了会话属性客户端标识符时才能使用。

以下 PL/SQL 调用为所有具有作为参数指定的客户端标识符的会话在级别 8 启用 SQL 跟踪。参数 `client_id` 没有默认值，而参数 `waits` 默认为 `TRUE`，参数 `binds` 默认为 `FALSE`。请注意，因为参数 `client_id` 是区分大小写的。

```
dbms_monitor.client_id_trace_enable(client_id => 'helicon.antognini.ch',
                                    waits     => TRUE,
                                    binds     => FALSE)
```


