# SQL 监控与调优

有时用户可能运行一个包含数百个查询的应用程序。在这种情况下，可能很难准确定位到底是哪条 SQL 语句导致了性能问题（特别是当问题在于累计成本高，而非单次执行成本高时）。遇到这类情况，你可以开启 SQL 跟踪来捕获某个用户运行的所有 SQL 语句的统计信息。

此外，当 SQL 是通过应用程序动态生成时，你无法从源代码中判断哪条 SQL 可能导致了性能问题。你可能遇到的情况是，应用程序生成的 SQL 在每次运行时都不同。在这种情况下，你必须为某个会话开启 SQL 跟踪，以记录该会话发出的所有 SQL 语句消耗的资源。此外，跟踪输出允许你查看实际的执行计划。

以下是跟踪会话的一般步骤：

1. 启用跟踪。
2. 运行你想要跟踪的 SQL 语句。
3. 禁用跟踪。
4. 使用像 `tkprof`、`trcsess` 或 Oracle Trace Analyzer 这样的工具，将跟踪文件转换为人类可读的格式。

Oracle 提供了多种方法来生成 SQL 资源使用跟踪文件（老实说，多到令人困惑），包括：

- `DBMS_SESSION`
- `DBMS_MONITOR`
- `DBMS_SYSTEM`
- `DBMS_SUPPORT`
- `ALTER SESSION`
- `ALTER SYSTEM`
- `oradebug`

你使用的方法取决于个人偏好和环境的各个方面，例如数据库版本和安装的 PL/SQL 包。以下小节将简要描述这些跟踪方法。

## 使用 DBMS_SESSION

本解决方案章节已介绍过如何使用 `DBMS_SESSION` 包。以下是使用 `DBMS_SESSION` 启用和禁用跟踪的简要总结：

```sql
SQL> exec dbms_session.set_sql_trace(sql_trace=>true);
SQL> -- 运行你想要跟踪的 sql 语句...
SQL> exec dbms_session.set_sql_trace(sql_trace=>false);
```

## 使用 DBMS_MONITOR

如果你使用的是 Oracle Database 10 *g* 或更高版本，我们推荐使用 `DBMS_MONITOR` 包，它提供了高度的灵活性，便于跟踪。要在当前会话中启用和禁用跟踪，请使用以下语句：

```sql
SQL> exec dbms_monitor.session_trace_enable;
SQL> -- 运行你想要跟踪的 sql 语句...
SQL> exec dbms_monitor.session_trace_disable;
```

使用 `WAIT` 和 `BINDS` 参数可启用包含等待和绑定变量信息的跟踪：

```sql
SQL> exec dbms_monitor.session_trace_enable(waits=>TRUE, binds=>TRUE);
```

> **注意**：等待事件跟踪等待资源所花费的时间。绑定变量是用于替代字面变量的替换变量。

使用 `SESSION_ID` 和 `SERIAL_NUM` 参数可为已连接的会话启用和禁用跟踪。首先运行此 SQL 查询以确定目标会话的 `SESSION_ID` 和 `SERIAL_NUM`：

```sql
select username,sid,serial# from v$session;
```

现在，在调用 `DBMS_SESSION` 时使用适当的值：

```sql
SQL> exec dbms_monitor.session_trace_enable(session_id=>1234, serial_num=>12345);
SQL> -- 运行你想要跟踪的 sql 语句...
SQL> exec dbms_monitor.session_trace_disable(session_id=>1234, serial_num=>12345);
```

你也可以如下启用等待和绑定变量信息的跟踪：

```sql
SQL> exec dbms_monitor.session_trace_enable(session_id=>1234, -
> serial_num=>12345, waits=>TRUE, binds=>TRUE);
```

## 使用 DBMS_SYSTEM

要在另一个会话中启用 SQL 跟踪，你可以使用 `DBMS_SYSTEM` 包。你必须首先识别出你想要跟踪的会话：

```sql
select username,sid,serial# from v$session;
```

将适当的值传递给以下代码行：

```sql
SQL> exec dbms_system.set_sql_trace_in_session(sid=>200,serial#=>5,-
> sql_trace=>true);
```

运行以下命令以禁用该会话的跟踪：

```sql
SQL> exec dbms_system.set_sql_trace_in_session(sid=>200,serial#=>5,-
> sql_trace=>false);
```

你也可以使用 `DBMS_SYSTEM` 来捕获等待事件：

```sql
SQL> exec dbms_system.set_ev(si=>123, se=>1234, ev=>10046, le=>8, nm=>' ');
SQL> exec dbms_system.set_ev(si=>123, se=>1234, ev=>10046, le=>0, nm=>' ');
```

## 使用 DBMS_SUPPORT

此技术要求你首先加载 `DBMS_SUPPORT` 包（默认不创建）：

```sql
SQL> @?/rdbms/admin/dbmssupp.sql
```

使用以下语法在你当前的会话中启用和禁用跟踪：

```sql
SQL> exec dbms_support.start_trace(waits=>TRUE, binds=>TRUE);
SQL> exec dbms_support.stop_trace;
```

使用此语法在你自身以外的会话中启用和禁用跟踪：

```sql
SQL> exec dbms_support.start_trace_in_session(sid=>123, serial=>1234,-
> waits=>TRUE, binds=>TRUE);
SQL> exec dbms_support.stop_trace_in_session(sid=>123, serial=>1234);
```

## 修改你的会话

你可以使用 `ALTER SESSION` 来开启和关闭跟踪：

```sql
SQL> alter session set sql_trace=true;
SQL> -- 运行 sql 命令...
SQL> alter session set sql_trace=false;
```

> **注意**：`SQL_TRACE` 参数已弃用。Oracle 建议你使用 `DBMS_MONITOR` 或 `DBMS_SESSION` 包来启用跟踪。

有时，查看 SQL 语句在数据库内等待资源的位置很有用。

要指示 Oracle 将等待信息写入跟踪文件，请使用以下语法：

```sql
SQL> alter session set events '10046 trace name context forever, level 8';
SQL> -- 运行 sql 命令...
SQL> alter session set events '10046 trace name context off';
```

`LEVEL 8` 指示 Oracle 将等待事件写入跟踪文件。如果你想同时查看等待事件和绑定变量，请指定 `LEVEL 12`。

> **提示**：在 Unix 系统上，可以检查 `$ORACLE_HOME/rdbms/mesg/oraus.msg` 文件以获取 Oracle 事件的描述。

Oracle 提供了一个名为 Trace Analyzer 的工具，它将解析 SQL 跟踪文件并生成全面的调优报告。详情请参阅 My Oracle Support（以前称为 MetaLink）说明 224270.1。

有时在跟踪应用程序时，跟踪文件可能跨越多个文件。在这种情况下，使用 `trcsess` 实用程序将多个跟踪文件转换为一个可读的输出文件。

## 修改系统

你可以使用以下 `ALTER SYSTEM` 语句为数据库中的所有会话开启跟踪：

```sql
SQL> alter system set sql_trace=true;
```

使用以下 SQL 禁用系统范围的跟踪：

```sql
SQL> alter system set sql_trace=false;
```

> **注意**：我们建议不要在系统级别设置 SQL 跟踪，因为它会严重降低系统性能。

## 使用 oradebug

`oradebug` 实用程序可用于为会话启用和禁用跟踪。你需要 `SYSDBA` 权限才能运行此实用程序。

你可以通过其操作系统进程 ID 或其 Oracle 进程 ID 来标识要跟踪的进程。

要确定这些 ID，请为你想要跟踪的用户（本例中用户为 `HEERA`）运行以下 SQL 查询：

```sql
select spid os_pid, pid ora_pid from v$process
where addr=(select paddr from v$session where username='HEERA');
```

以下是此示例的输出：

```
OS_PID   ORA_PID
-------- ---------
31064    23
```

接下来，使用带有 `SETOSPID` 或 `SETORAPID` 选项的 `oradebug` 将 `oradebug` 附加到会话。此示例使用 `SETOSPID` 选项：

```sql
SQL> oradebug setospid 31064;
```

现在你可以为该会话开启跟踪：

```sql
SQL> oradebug EVENT 10046 TRACE NAME CONTEXT FOREVER, LEVEL 8;
```

你可以通过查看跟踪文件名来验证跟踪是否已启用：

```sql
SQL> oradebug TRACEFILE_NAME;
```

使用以下语法禁用跟踪：

```sql
SQL> oradebug EVENT 10046 TRACE NAME CONTEXT OFF;
```

使用 `HELP` 选项可以查看 `oradebug` 实用程序中所有可用的功能：

```sql
SQL> oradebug help
```


`DBMS_TRACE`包并非一个 SQL 跟踪工具，而是一个 PL/SQL 调试工具。要准备使用`DBMS_TRACE`的环境，首先以 SYS 身份运行以下 SQL 脚本：
```sql
@?/rdbms/admin/tracetab.sql
@?/rdbms/admin/dbmspbt.sql
```

现在，你可以通过两种方式之一启用 PL/SQL 跟踪。第一种方法是使用`ALTER SESSION`命令：
```sql
alter session set plsql_debug=true;
```

第二种方法是使用`ALTER`命令并以`DEBUG`选项进行编译：
```sql
alter <procedure | function | package body> compile debug;
```

现在，你可以通过如下调用`DBMS_TRACE`包来跟踪 PL/SQL 代码：
```sql
exec dbms_trace.set_plsql_trace(dbms_trace.trace_all_lines);
```

此后你运行的任何 PL/SQL 代码都将被跟踪。执行以下命令以关闭跟踪：
```sql
exec dbms_trace.clear_plsql_trace();
```

要查看 PL/SQL 跟踪信息，请查询 SYS 拥有的`PLSQL_TRACE_EVENTS`表：
```sql
select
  event_seq, stack_depth, event_kind,
  event_unit, event_line, event_comment
from sys.plsql_trace_events;
```

`DBMS_TRACE`包为你提供了一种调试 PL/SQL 代码的实用技术。

## 19-10\. 解释执行计划

### 问题

你正在查看来自 AUTOTRACE 等实用程序的格式化执行计划输出（关于 AUTOTRACE 的详细信息，请参见配方 19-7）。你想知道如何解释该输出。

### 解决方案

一个*执行计划*是 Oracle 将如何为一条 SQL 语句检索和处理结果集的逐行描述。执行计划中的每一行描述了 Oracle 将如何从数据库物理检索行，或处理在先前步骤中已检索到的行。以下是一些确定格式化执行计划（例如 AUTOTRACE 的输出）中步骤运行顺序的指导原则：

*   如果两个步骤处于相同的缩进级别，则最上面的步骤最先执行。
*   对于给定步骤，缩进最深的子步骤最先执行。
*   当操作完成时，它会将其结果传递给上一层级。

一个简单的例子将有助于说明这些概念。使用`SET AUTOTRACE`语句为以下查询生成一个解释计划：
```sql
SET AUTOTRACE TRACE EXPLAIN
select
  p.first_name
  ,a.address1
  ,i.invoice_id
from parties p
  ,addresses a
  ,invoice_transactions i
where a.party_id = p.party_id
  and p.first_name = 'HEERA'
  and p.party_id = i.party_id;
```

这是相应计划的局部列表：

| Id | 操作                  | 名称                   | 行数 | 字节数 | 成本（%CPU） |
|----|-----------------------|------------------------|------|-------|-------------|
| 0  | SELECT STATEMENT      |                        | 39   | 1716  | 375 (1)     |
| 1  | HASH JOIN             |                        | 39   | 1716  | 375 (1)     |
| 2  | HASH JOIN             |                        | 2    | 70    | 32 (4)      |
| 3  | TABLE ACCESS FULL     | PARTIES                | 2    | 22    | 22 (0)      |
| 4  | TABLE ACCESS FULL     | ADDRESSES              | 1664 | 39936 | 9 (0)       |
| 5  | TABLE ACCESS FULL     | INVOICE_TRANSACTIONS   | 34072| 299K  | 342 (1)     |

从这个输出中，最右边缩进的操作是对`PARTIES`和`ADDRESSES`表的全表扫描。由于它们处于相同的缩进级别，因此最先执行最上面的对`PARTIES`的全表扫描（ID 3）。接下来扫描`ADDRESSES`表（ID 4）。这两个操作的输出通过一个哈希连接（ID 2）进行组合。另一个哈希连接（ID 1）将前一个哈希连接（ID 2）的输出与`INVOICE_TRANSACTIONS`表的全表访问（ID 5）相结合。结果返回给调用的 SQL SELECT 语句。

解释计划输出表明，步骤 ID 5 中的全表扫描操作成本为 342，并对`INVOICE_TRANSACTIONS`表进行了全表扫描。`INVOICE_TRANSACTIONS`表通过`PARTY_ID`列与`PARTIES`表连接。为了尝试降低成本，在`INVOICE_TRANSACTIONS`的`PARTY_ID`列上添加了一个索引：
```sql
create index inv_trans_idx1 on invoice_transactions(party_id);
```

通过再次运行原始查询来重新生成解释计划：
```sql
select
  p.first_name
  ,a.address1
  ,i.invoice_id
from parties p
```

```sql
,addresses a
,invoice_transactions i
where a.party_id = p.party_id
and p.first_name = 'HEERA'
and p.party_id = i.party_id;
```

以下是部分输出，显示成本已显著降低：484

[IT 电子书官网](http://www.it-ebooks.info/)

