# 表 19-2. DBMS_XPLAN 中可用的函数

## 过程 目的

`DISPLAY`
显示 `PLAN_TABLE` 内容对应的执行计划。

`DISPLAY_AWR`
显示存储在 AWR 存储库中的 SQL 语句的执行计划。

`DISPLAY_CURSOR`
显示游标的执行计划。

`DISPLAY_SQL_PLAN_BASELINE`
显示由 SQL 句柄标识的 SQL 的执行计划。

`DISPLAY_SQL_SET`
显示 SQL 调优集中包含的 SQL 的执行计划。

`DBMS_XPLAN` 也很方便，可用于使用 `DISPLAY_CURSOR` 显示查询的 `V$SQL_PLAN` 内容。

例如，要显示当前查询的解释计划，首先运行查询：

```sql
select * from parties where party_id = 1;
```

现在为会话中最近执行的查询生成解释计划：

```sql
select * from table(dbms_xplan.display_cursor(null, null));
```

以下是输出的部分列表（为适应页面进行了截断）：

```
| Id | Operation | Name | Rows | Bytes | Cost (%CPU)| Time |
| 0 | SELECT STATEMENT | | | | 1 (100)| |
| 1 | TABLE ACCESS BY | PARTIES | 1 | 47 | 1 (0)| 00:00:01 |
|* 2 | INDEX UNIQUE SC| SYS_C0047742 | 1 | | 1 (0)| 00:00:01 |
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 19 章 ■ SQL 监控与调优

##### 19-9. 追踪会话的所有 SQL 语句

### 问题
一个用户运行的应用程序调用了数百条 SQL 语句，出现了性能问题。你想具体确定哪些语句消耗的资源最多。

### 解决方案
你可以指示 Oracle 捕获一个会话运行的所有 SQL 语句所消耗的资源和执行计划。一种启用 SQL 追踪的技术是使用 `DBMS_SESSION` 包。在执行此操作之前，你需要拥有授予你模式的 `ALTER SESSION` 权限：

```sql
SQL> exec dbms_session.set_sql_trace(sql_trace=>true);
```

**注意：** 必须将 `TIMED_STATISTICS` 初始化参数设置为 `TRUE`（默认设置）才能生成 SQL 追踪统计信息。

现在，你运行的任何 SQL 都将被追踪。为演示目的，我们将追踪以下 SQL 语句：

```sql
select 'my_trace' from dual;
```

要关闭会话的 SQL 追踪，你可以退出 SQL*Plus 或使用 `DBMS_SESSION` 禁用追踪：

```sql
SQL> exec dbms_session.set_sql_trace(sql_trace=>false);
```

现在检查你的 `USER_DUMP_DEST` 初始化参数的值：

```sql
SQL> show parameter user_dump_dest

NAME TYPE VALUE
-------------------------- ----------- --------------------------------------
user_dump_dest string /oracle/diag/rdbms/rmdb11/RMDB11/trace
```

从操作系统导航到用户转储目标目录：

```bash
$ cd /oracle/diag/rdbms/rmdb11/RMDB11/trace
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

第 19 章 ■ SQL 监控与调优

**提示：** 通过在启用追踪前设置 `TRACEFILE_IDENTIFIER` 初始化参数，你可以为追踪文件分配一个易于识别的名称。例如，`ALTER SESSION SET TRACEFILE_IDENTIFIER=FOO` 将生成一个名为 `<SID>_ora_<PID>_FOO.trc` 的追踪文件。

你的用户转储目标目录中可能有数百个追踪文件。查找最新的文件。在此示例中，我们知道我们的追踪文件将包含字符串 `my_trace`（来自上面运行的 SQL）。在 Unix 环境中，你可以使用 `grep` 命令搜索该字符串：

```bash
$ grep -i my_trace *.trc

RMDB11_ora_21846.trc: select 'my_trace' from dual
```

Oracle 提供了 `tkprof` 命令行实用程序，用于将追踪文件的内容转换为人类可读的格式。此实用程序位于 `ORACLE_HOME/bin` 目录中（与其他所有标准 Oracle 实用程序如 `sqlplus`、`expdp` 等一起）。向 `tkprof` 提供用于生成追踪文件的模式的用户名和密码（在此示例中为 `heera/foo`）：

```bash
$ tkprof RMDB11_ora_21846.trc readable.txt explain=heera/foo sys=no
```

**提示：** 在命令行不带任何参数运行 `tkprof` 以查看此实用程序的所有选项。默认情况下，SQL 语句按运行顺序报告。你可以使用 `SORT` 参数更改排序顺序，以按 CPU、磁盘读取、耗时等对输出进行排序。

现在用文本编辑器检查 `readable.txt` 文件。以下是文本文件中的一些示例输出：

```
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.01       0.01          0          0          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch        2      0.00       0.00          2          3          0           1
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        4      0.01       0.01          2          3          0           1

Misses in library cache during parse: 1
Optimizer mode: ALL_ROWS
Parsing user id: 28 (HEERA)

Rows     Row Source Operation
-------  ---------------------------------------------------
      1  SORT AGGREGATE (cr=3 pr=2 pw=2 time=0 us)
    103   TABLE ACCESS FULL EMP (cr=3 pr=2 pw=2 time=4 us cost=2 size=0 card=103)
```

```
Rows     Execution Plan
-------  ---------------------------------------------------
      0  SELECT STATEMENT   MODE: ALL_ROWS
      1   SORT (AGGREGATE)
    103    TABLE ACCESS   MODE: ANALYZED (FULL) OF 'EMP' (TABLE)
```

`tkprof` 输出提供了在追踪期间发出的每条 SQL 语句所消耗资源的详细信息。使用此实用程序识别哪些 SQL 语句是潜在的瓶颈，并相应地进行调优。

**提示：** 默认情况下，追踪文件的权限仅允许 Oracle 软件操作系统所有者查看文件。要启用对追踪文件的公共读取权限，请将初始化参数 `_TRACE_FILES_PUBLIC` 设置为 `TRUE`。但是，你只应在测试或开发系统上执行此操作，开发人员需要访问追踪文件用于调优目的。

### 工作原理



