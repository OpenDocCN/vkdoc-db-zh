# SQL 跟踪技术

## 概述与工作原理

### 如何工作
在追踪任何 SQL 会话之前，请确保已将 `timed_statistics` 初始化参数设置为 `true`。如果此参数的值为 `false`，SQL 跟踪将被禁用。将 `timed_statistics` 参数设置为 `true` 可使数据库收集 CPU 和耗时等统计信息，并将其存储在各种动态性能表中。从 Oracle 11.1.0.7.0 版本开始，此参数的默认值取决于初始化参数 `statistics_level` 的值。如果将 `statistics_level` 参数设置为 `basic`，则 `timed_statistics` 参数的默认值为 `false`。如果将 `statistics_level` 设置为 `typical` 或 `all`，则 `timed_statistics` 参数的默认值为 `true`。`timed_statistics` 参数是动态的，这意味着您无需重启数据库即可将其打开——您可以在整个数据库上开启此参数而不会产生显著开销。您也可以仅为单个会话开启此参数。

当您追踪一个 SQL 会话时，Oracle 会生成一个跟踪文件，其中包含对排查 SQL 性能问题非常有用的诊断数据。从 Oracle Database 11g 开始，数据库将所有诊断文件存储在一个专用的诊断目录中，该目录通过 `diagnostic_dest` 初始化参数指定。诊断目录的结构如下：

```
<diagnostic_dest>/diag/rdbms/<dbname>/<instance>
```

该诊断目录称为 ADR 主目录。如果您的数据库名称是 `prod1`，实例名称也是 `prod1`，那么 ADR 主目录将是：

```
<diagnostic_dest>/diag/rdbms/prod1/prod1
```

ADR 主目录在 `<ADR Home>/trace` 子目录中包含跟踪文件。跟踪文件通常以 `.trc` 为扩展名。您会注意到，多个跟踪文件有一个对应的跟踪映射文件，扩展名为 `.trm`。`.trm` 文件包含有关跟踪文件的结构信息，数据库使用这些信息进行搜索和导航。您可以使用以下命令查看数据库的诊断目录设置：

```
SQL> sho parameter diagnostic_dest

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
diagnostic_dest                      string      C:\APP\ORA
SQL>
```

`V$DIAG_INFO` 视图显示了各个诊断目录的位置，包括跟踪目录，该目录在视图中以 Diag Trace 名称列出。尽管 Oracle Database 11g 中的新数据库可诊断性基础设施忽略了 `user_dump_dest` 初始化参数，但该参数仍然存在，并且指向与 `$ADR_BASE\diag\rdbms\<database>\<instance>\trace` 目录相同的目录，如以下命令所示：

```
SQL> show parameter user_dump_dest

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
user_dump_dest                       string      c:\app\ora\diag\rdbms\orcl1\or
                                                 cl1\trace
SQL>
```

在 Oracle Database 11g 中，您无需设置 `max_dump_file_size` 参数来指定跟踪文件的最大大小。

## 追踪特定 SQL 语句

### 问题
您想要追踪特定的 SQL 语句，以查明数据库在执行该语句期间将时间花费在了哪里。

### 解决方案
在 Oracle 11.1 或更高版本中，您可以使用增强的 SQL 跟踪接口来追踪一条或多条 SQL 语句。以下是对一组 SQL 语句进行追踪的步骤。

1.  执行 `alter session set events` 语句以设置追踪，如下所示：
    ```
    SQL> alter session set events 'sql_trace level 12';
    Session altered.
    SQL>
    ```
2.  执行 SQL 语句：
    ```
    SQL> select count(*) from sales;
    ```
3.  关闭追踪：
    ```
    SQL> alter session set events 'sql_trace off';
    Session altered.
    SQL>
    ```

您可以通过在 `alter session set events` 语句中指定 SQL 语句的 SQL ID 来选择追踪特定的 SQL 语句。步骤如下：

1.  通过执行以下语句找到 SQL 语句的 SQL ID：
    ```
    SQL> select sql_id,sql_text
      2  from v$sql
      3  where sql_text='select sum(quantity_sold) from sales';

    SQL_ID                       SQL_TEXT
---------------------------- ------------------------------------
    fb2yu0p1kgvhr                select sum(quantity_sold) from sales

    SQL>
    ```
2.  为您检索到的 SQL ID 对应的特定 SQL 语句开启追踪：
    ```
    SQL> alter session set events 'sql_trace [sql:fb2yu0p1kgvhr] level 12';
    Session altered.
    SQL>
    ```
3.  执行 SQL 语句：
    ```
    SQL> select sum(quantity_sold) from sales;

    SUM(QUANTITY_SOLD)
    ------------------
                918843
    ```
4.  关闭追踪：
    ```
    SQL> alter session set events 'sql_trace[sql:fb2yu0p1kgvhr] off';
    Session altered.
    SQL>
    ```

您可以通过用管道符 (`|`) 分隔 SQL ID 来追踪多条 SQL 语句，如下所示：

```
SQL> alter session set events ‘sql_trace [sql: fb2yu0p1kgvhr|4v433su9vvzsw]‘;
```

您可以通过执行 `alter system set events` 语句来追踪在另一个会话中运行的特定 SQL 语句：

```
SQL> alter system set events 'sql_trace[sql:fb2yu0p1kgvhr] level 12';
System altered.
SQL>
```

您可以通过查询 `V$SQL` 视图（如本配方前面所示）来获取该语句的 SQL ID，也可以通过 Oracle Enterprise Manager 获取。一旦另一个会话中的用户完成执行 SQL 语句，请使用以下命令关闭追踪：

```
SQL> alter system set events 'sql_trace[sql:fb2yu0p1kgvhr] off';
System altered.
SQL>
```

### 如何工作
在 Oracle Database 11g 中，您可以设置 Oracle 事件 `SQL_TRACE` 来追踪一条或多条 SQL 语句的执行。您可以执行 `alter session` 或 `alter system` 语句来追踪特定的 SQL 语句。命令语法如下：

```
alter session/system set events ‘sql_trace [sql:<sql_id>|<sql_id>] … event specification‘;
```

即使您在关闭追踪之前执行了多条 SQL 语句，跟踪文件也只会显示与您指定的 SQL_ID 或 SQL_ID 相关的信息。

## 在自己的会话中启用追踪

### 问题
您想要追踪您自己的会话。

### 解决方案
普通用户可以使用 `DBMS_SESSION` 包来追踪其会话，如下例所示：

```
SQL>execute dbms_session.session_trace_enable(waits=>true, binds=> false);
```

要禁用追踪，用户必须执行 `session_trace_disable` 过程，如下所示：

```
SQL> execute dbms_session.session_trace_disable();
```

### 如何工作
Oracle 推荐用于所有追踪的 `DBMS_MONITOR` 包只能由具有 DBA 角色的用户执行。如果您没有 DBA 角色，可以使用 `dbms_session.session_trace_enable` 过程来追踪您自己的会话。

## 查找跟踪文件

### 问题
您希望找到一种轻松识别您的跟踪文件的方法。


#### 解决方案

在开始生成跟踪之前，请执行以下语句为跟踪文件设置标识符：

`SQL> alter session set tracefile_identifier='MyTune1';`

要查看数据库创建的最新跟踪文件（适用于 Oracle Database 11.1 及更新版本），可以通过执行以下命令查询自动诊断存储库（有关`adrci`工具的详细信息，请参见第 5 章）：

```
adrci> show tracefile -t
08-MAY-11 19:01:48  diag\rdbms\orcl1\orcl1\trace\orcl1_p000_8652_MyTune1.trc
08-MAY-11 19:01:48  diag\rdbms\orcl1\orcl1\trace\orcl1_p001_6424_MyTune1.trc
08-MAY-11 19:01:48  diag\rdbms\orcl1\orcl1\trace\orcl1_p002_5980_MyTune1.trc
adrci>
```

要查找当前会话的跟踪文件路径，请执行以下命令：

```
SQL>  select value from v$diag_info
      where name = 'Default Trace File';

VALUE
-----------------------------------------------------------------------

c:\app\ora\diag\rdbms\orcl1\orcl1\trace\orcl1_ora_11248_My_Tune1.trc

SQL>
```

要查找当前实例的所有跟踪文件，请执行以下查询：

`SQL> select value from v$diag_info where name = 'Diag Trace'`

#### 工作原理

通常，很难找到你正在寻找的确切跟踪文件，因为跟踪目录中可能有一堆其他跟踪文件，且文件名看起来都很相似。在进行 SQL 跟踪时，一个最佳实践是将你的跟踪文件与一个唯一标识符关联起来。为即将生成的跟踪文件设置一个标识符，可以轻松地从数据库在跟踪目录中生成的众多跟踪文件中识别出 SQL 跟踪文件。

你可以使用以下命令确认跟踪标识符的值：

```
SQL> sho parameter tracefile_identifier
NAME                                 TYPE        VALUE
------------------------------------ ----------- ---------------
tracefile_identifier                 string      MyTune1
SQL>
```

`V$PROCESS`视图中的`TRACEID`列也显示了`tracefile_identifier`参数的当前值。你设置的跟踪文件标识符会成为跟踪文件名的一部分，使得从跟踪目录中大量跟踪文件里轻松挑选出正确的文件名成为可能。你可以为一个会话多次修改`tracefile_identifier`参数的值。一个进程的跟踪文件名将包含表明它们都属于同一进程的信息。

一旦你设置了`tracefile_identifier`参数，跟踪文件将具有以下格式，其中`sid`是 Oracle SID，`pid`是进程 ID，`traceid`是你为`tracefile_identifier`初始化参数设置的值。

`sid_ora_pid_traceid.trc`

### 10-5. 检查原始 SQL 跟踪文件

#### 问题

你想检查一个原始 SQL 跟踪文件。

#### 解决方案

在文本编辑器中打开跟踪文件以检查跟踪信息。以下是通过执行`dbms_monitor.session_trace_enable`过程生成的原始 SQL 跟踪的部分内容：

```
PARSING IN CURSOR #3 len=490 dep=1 uid=85 oct=3 lid=85 tim=269523043683 hv=672110367 ad='7ff18986250' sqlid='bqasjasn0z5sz'
PARSE #3:c=0,e=647,p=0,cr=0,cu=0,mis=1,r=0,dep=1,og=1,plh=0,tim=269523043680
EXEC #3:c=0,e=1749,p=0,cr=0,cu=0,mis=1,r=0,dep=1,og=1,plh=3969568374,tim=269523045613
WAIT #3: nam='Disk file operations I/O' ela= 15833 FileOperation=2 fileno=4 filetype=2 obj#=-1 tim=269523061555
FETCH #3:c=0,e=19196,p=0,cr=46,cu=0,mis=0,r=1,dep=1,og=1,plh=3969568374,tim=269523064866
STAT #3 id=3 cnt=12 pid=2 pos=1 obj=0 op='HASH GROUP BY (cr=46 pr=0 pw=0 time=11 us cost=4 size=5317 card=409)'
STAT #3 id=4 cnt=3424 pid=3 pos=1 obj=89079 op='TABLE ACCESS  FULL DEPT (cr=16 pr=0 pw=0 time=246 us cost=3 size=4251 card=327)'
```

从这个原始跟踪文件的摘录中可以看出，你可以收集到有用的信息，例如解析未命中（parse misses）、等待事件以及 SQL 语句的执行计划。

#### 工作原理

获取会话跟踪文件后的常规做法是使用`TKPROF`等工具进行分析。但是，你也可以通过目视阅读跟踪输出内容来检查跟踪文件。原始跟踪文件捕获了 SQL 语句处理以下三个步骤的信息：

`解析`：在此阶段，数据库将 SQL 语句转换为执行计划，并检查授权以及表和其他对象是否存在。

`执行`：数据库在此阶段执行 SQL 语句。对于`SELECT`语句，执行阶段确定数据库必须检索的行。对于诸如插入、更新和删除之类的 DML 语句，数据库会修改数据。

`获取`：此步骤仅适用于`SELECT`语句。在此阶段，数据库检索选定的行。

除了等待事件信息外，SQL 跟踪文件还将包含这三个执行阶段的详细统计信息。你通常使用`TKPROF`等实用程序来格式化原始跟踪文件。然而，有时通过简单地滚动浏览文件，原始跟踪文件可以非常快速地向你显示有用的信息。锁定情况就是一个很好的例子，在这种情况下你可以目视检查原始跟踪文件。`TKPROF`不会为你提供有关 latches 和 locks（enqueues）的详细信息。如果你怀疑查询正在等待锁，深入研究原始跟踪文件会准确地向你显示查询在何处以及为何等待。在`WAIT`行中，经过时间（`ela`）显示了等待的时间量（以微秒为单位）。在我们的示例中，“Disk file operations I/O”的等待经过时间是 15,833 微秒。因为 1 秒=1,000,000 微秒，所以这不是一个显著的等待时间。原始跟踪文件清楚地显示了是 I/O 等待事件（如此例所示）还是其他类型的等待事件阻碍了查询。如果查询正在等待锁，你将看到类似以下内容：`WAIT #2: nam='enqueue ela-300…. `

我们在此方法中特意将讨论保持简短，因为诸如`TKPROF`和 Oracle Trace Analyzer 之类的工具通过分析原始跟踪文件为你提供复杂的诊断信息。

### 10-6. 分析 Oracle 跟踪文件

#### 问题

你想知道如何分析 Oracle 跟踪文件。

#### 解决方案

有多种方法可以解释 SQL 跟踪文件。以下是不同的方法：

*   在文本编辑器中阅读原始 SQL 跟踪文件。
*   使用 Oracle 提供的`TKPROF`（跟踪内核分析器）实用程序。
*   使用 Oracle Trace Analyzer，这是一个可从 Oracle 支持免费下载的产品。
*   使用第三方工具。


## 工作原理

获取 SQL 跟踪通常比较容易，而分析它当然比收集跟踪更具挑战性。有时，如果你特别擅长，可以直接查看源跟踪文件本身，但在大多数情况下，你需要一个工具来解释和分析跟踪文件可能包含的大量数据。请注意，`TKPROF` 或其他分析工具显示的是查询执行各个阶段的耗时，但不包含锁和闩锁的信息。如果你试图找出是否有任何锁在减慢查询速度，请查看原始跟踪文件，检查原始文件的 `WAIT` 行中是否有任何入队等待事件。

你可以轻松阅读某些跟踪文件，例如事件 10053 的跟踪文件，因为该文件不包含任何 SQL 执行统计信息（如解析、执行和获取统计信息），也没有等待事件分析——它主要包含基于成本的优化器（CBO）使用的执行路径跟踪。然而，对于任何 SQL 执行跟踪文件，例如使用事件 10046 生成的文件，虽然从技术上讲可以目视检查，但这不仅耗时，而且原始数据不是摘要形式，关键事件通常以晦涩的方式描述。因此，使用像 `TKPROF` 实用程序这样的分析器确实是你的最佳选择。

`TKPROF` 实用程序是 Oracle 提供的分析工具，大多数 Oracle DBA 会例行使用。配方 10-7 和 10-8 展示了如何使用 `TKPROF`。

Oracle 的 Trace Analyzer 是免费的（你需要从 Oracle Support 下载），易于安装和使用，并能生成包含大量有用诊断信息的清晰报告。你确实需要先安装该工具，但安装只需几分钟即可完成。之后，你只需将跟踪文件的名称传递给一个脚本以生成格式化输出。配方 10-9 展示了如何安装和使用 Oracle Trace Analyzer。

还有一些第三方分析工具提供 `TKPROF` 实用程序所不具备的功能。其中一些工具能生成漂亮的 HTML 跟踪报告，有些还包括图表，以帮助你目视检查你跟踪的 SQL 语句执行的详细信息。请注意，为了使用其中一些产品，你需要上传你的跟踪文件进行分析。如果你的跟踪文件包含敏感数据或安全信息，这可能不适用。

### 10-7. 使用 TKPROF 格式化跟踪文件

#### 问题

你已经跟踪了一个会话，并希望使用 `TKPROF` 来格式化跟踪文件。

#### 解决方案

你从命令行运行 `TKPROF` 实用程序。以下是一个典型的用于格式化跟踪文件的 `tkprof` 命令示例。

```
$ tkprof user_sql_001.trc user1.prf explain=hr/hr table=hr.temp_plan_table_a sys=no
  sort=exeela,prsela,fchela
```

在此示例中，`tkprof` 命令接受 `user_sql_001.trc` 跟踪文件作为输入，并生成一个名为 `user1.prf` 的输出文件。本配方的“工作原理”部分解释了 `TKPROF` 实用程序的关键可选参数。

#### 工作原理

`TKPROF` 是一个实用程序，可让你格式化使用事件 10046 或通过 `DBMS_MONITOR` 包生成的任何扩展跟踪文件。你可以使用此工具生成报告，用于分析本章解释的各种 SQL 跟踪结果。你可以在单个跟踪文件或使用 `trcsess` 实用程序连接的一组跟踪文件上运行 `TKPROF`。`TKPROF` 显示 SQL 语句执行各个方面的详细信息，例如以下内容：

*   SQL 语句文本
*   SQL 跟踪统计信息
*   解析和执行阶段期间库缓存未命中的次数
*   所有 SQL 语句的执行计划
*   递归 SQL 调用

你可以通过不带任何参数地发出 `tkprof` 命令来查看可以指定的所有参数列表，如下所示：

```
$ tkprof
Usage: tkprof tracefile outputfile [explain= ] [table= ]
              [print= ] [insert= ] [sys= ] [sort= ]
...
```

> 以下是你可以使用 `tkprof` 命令指定的重要参数的简要说明：
>
> `filename1`：指定跟踪文件的名称
>
> `filename2`：指定格式化的输出文件
>
> `waits`：指定输出文件是否应记录等待事件的摘要；默认是`yes`。
>
> `sort`：默认情况下，`TKPROF` 按执行顺序列出跟踪文件中的 SQL 语句。你可以使用 `sort` 参数指定各种选项来控制 `TKPROF` 列出各个 SQL 语句的顺序。
>
> *   `prscpu`：解析花费的 CPU 时间
> *   `prsela`：解析花费的耗时
> *   `execpu`：执行花费的 CPU 时间
> *   `exeela`：执行花费的耗时
> *   `fchela`：获取花费的耗时
>
> `print`：默认情况下 `TKPROF` 会列出所有跟踪到的 SQL 语句。通过为 `print` 选项指定一个值，你可以限制输出文件中列出的 SQL 语句数量。
>
> `sys`：默认情况下 `TKPROF` 列出用户 SYS 发出的所有 SQL 语句以及递归语句。为 `sys` 参数指定值 `no` 可使 `TKPROF` 省略这些语句。
>
> `explain`：将执行计划写入输出文件；`TKPROF` 连接到数据库并使用你在此参数提供的用户名和密码发出 explain plan 语句。
>
> `table`：默认情况下，`TKPROF` 使用由 explain 参数指定的用户模式中名为 `PLAN_TABLE` 的表来存储执行计划。你可以使用 table 参数指定一个替代表。
>
> `width`：这是一个整数，决定了某些类型输出（如 explain plan 信息）的输出行宽。

### 10-8. 分析 TKPROF 输出

#### 问题

你已经使用 `TKPROF` 格式化了一个跟踪文件，现在希望分析 `TKPROF` 输出文件。

#### 解决方案

使用 `tkprof` 命令调用 `TKPROF` 实用程序，如下所示：

```
c:\>tkprof orcl1_ora_6448_mytrace1.trc ora6448.prf explain=hr/hr sys=no sort=prsela,exeela,fchela

TKPROF: Release 11.2.0.1.0 - Development on Sat May 14 11:36:35 2011

Copyright (c) 1982, 2009, Oracle and/or its affiliates.  All rights reserved.

c:\app\ora\diag\rdbms\orcl1\orcl1\trace>
```

在此示例中，`orcl1_ora_6448_mytrace1.trc` 是你想要格式化的跟踪文件。`ora6448.prf` 文件是 `TKPROF` 输出文件。随后的“工作原理”部分展示了如何解释 `TKPROF` 输出文件。

#### 工作原理

在我们的示例中，只有一个 SQL 语句。因此，排序参数（`prsecla`, `exeela`, `fchela`）并不重要，因为它们仅在 `TKPROF` 需要列出多个 SQL 语句时才会起作用。以下是 `TKPROF` 输出文件中关键部分的简要说明。

##### 标题

标题部分显示了跟踪文件名、排序选项以及输出文件中所用术语的描述。

```
Trace file: orcl1_ora_6448_mytrace1.trc
Sort options: prsela  exeela  fchela  
********************************************************************************
count    = number of times OCI procedure was executed
cpu      = cpu time in seconds executing
elapsed  = elapsed time in seconds executing
disk     = number of physical reads of buffers from disk
query    = number of buffers gotten for consistent read
current  = number of buffers gotten in current mode (usually for update)
rows     = number of rows processed by the fetch or execute call
*************************************************************************
```



## 执行统计

`TKPROF` 会列出跟踪文件中每条 SQL 语句的执行统计信息。`TKPROF` 列出了 SQL 语句处理过程中三个步骤的执行统计：`解析`、`执行` 和 `获取`。

```
调用   次数      CPU 时间   耗时      磁盘读    查询(一致模式)  查询(当前模式)  行数
------- ------  -------- -------- ---------- ---------- ---------- ----------
Parse        1      0.01       0.03          0         64          0          0
Execute      1      0.00       0.00          0          0          0          0
Fetch     5461      0.29       0.40          0       1299          0      81901
------- ------  -------- -------- ---------- ---------- ---------- ----------
total     5463      0.31       0.43          0       1363          0      81901
```

> 下表解释了 SQL 执行统计表中各项的含义：
>
> `count`：数据库解析、执行或获取此语句的次数
>
> `cpu`：解析/执行/获取阶段所使用的 CPU 时间
>
> `elapsed`：解析/执行/获取阶段的总耗时（以秒为单位）
>
> `disk`：解析/执行/获取阶段的物理数据块读取次数
>
> `query`：在一致模式下，通过逻辑读从缓冲区缓存中读取的数据块数量（针对 `select` 语句的解析/获取/执行阶段）
>
> `current`：在当前模式下，通过逻辑读从缓冲区缓存中读取和检索的数据块数量（针对 `insert`、`update`、`delete` 和 `merge` 语句）
>
> `rows`：对于 `select` 语句，表示获取的行数；对于 `insert`、`delete`、`update` 或 `merge` 语句，则分别表示插入、删除或更新的行数

## 行源操作

报告的下一部分显示了库缓存中的未命中次数、当前的优化器模式以及查询的行源操作。行源操作显示了数据库为每个操作（如连接或全表扫描）处理的行数。

```
解析期间库缓存未命中次数：1
优化器模式：ALL_ROWS
解析用户 ID：85  (HR)

行数   行源操作
-------  ---------------------------------------------------
  81901  HASH JOIN  (cr=1299 pr=0 pw=0 time=3682295 us cost=22 size=41029632 card=217088)
   1728   TABLE ACCESS FULL DEPT (cr=16 pr=0 pw=0 time=246 us cost=6 size=96768 card=1728)
   1291   TABLE ACCESS FULL EMP (cr=1283 pr=0 pw=0 time=51213 us cost=14 size=455392 card=3424)
```

`解析期间库缓存未命中次数` 表示在解析和执行数据库调用期间的硬解析次数。在 `行源操作` 列中，输出包含了每个行源操作的若干统计信息。这些统计信息量化了行源操作期间执行的各种类型的工作。以下是不同统计信息的含义（并非每个查询都会显示所有信息）：

> `cr`：通过一致模式逻辑读取的数据块数
>
> `pr`：通过物理磁盘读取的数据块数
>
> `pw`：通过物理写入磁盘的数据块数
>
> `time`：处理该操作花费的总时间（以毫秒为单位）
>
> `cost`：该操作的估计成本
>
> `size`：该操作返回的数据量估计值（字节）
>
> `card`：该操作返回的行数估计值

## 执行计划

如果您在发出 `tkprof` 命令时指定了 `explain` 参数，您将看到一个执行表，显示每条 SQL 语句的执行计划。

```
行数   执行计划
-------  ---------------------------------------
      0  SELECT STATEMENT   MODE: ALL_ROWS
  81901   HASH JOIN
   1728    TABLE ACCESS (FULL) OF 'DEPT' (TABLE)
   1291    TABLE ACCESS (FULL) OF 'EMP' (TABLE)
```

在我们的示例中，执行计划显示有两次全表扫描，随后进行了一次哈希连接。

## 等待事件

只有在跟踪命令中指定了 `waits=>true` 时，您才会看到等待事件部分。等待事件表总结了跟踪期间的等待情况：

```
耗时包括等待以下事件：
  等待的事件                          等待次数  最大等待时间  总等待时间
  ----------------------------------------   -------  ----------  ------------
    SQL*Net message from client                5461     112.95        462.81
    db file sequential read                      1       0.05          0.05
```

```
********************************************************************************
```

在此示例中，`SQL*Net message to client` 等待占据了大部分等待，但这些属于空闲等待。如果您看到诸如 `db file sequential read` 或 `db file scattered read` 之类的等待事件，并且等待次数（和/或总等待时间）显著，您需要进一步调查这些等待事件。

请注意，`TKPROF` 输出不会显示任何绑定变量的信息。它也不显示因队列锁而导致的任何等待。

### 10-9. 使用 Oracle Trace Analyzer 分析跟踪文件

#### 问题

您想使用 Oracle Trace Analyzer 来分析跟踪文件。


