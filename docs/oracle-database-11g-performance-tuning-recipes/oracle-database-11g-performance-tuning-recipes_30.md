# 10-10. 追踪并行查询

## 解决方案

Oracle Trace Analyzer，也称为 TRCANLZR 或 TRCA，是一个 SQL 跟踪分析工具，可作为 `TKPROF` 工具的替代方案。您必须从 Oracle Support 下载 TRCA。下载后，解压文件并通过执行 `/trca/install/trcreate.sql` 脚本来安装 TRCA。

安装 TRCA 后，您必须以具有 SYSDBA 权限的用户登录来执行 `tacreate.sql` 脚本。`tacreate.sql` 会为任何已生成的跟踪文件生成格式化输出文件。该脚本会询问您有关跟踪文件位置、输出文件以及希望 TRCA 存储其数据的表空间等信息。

以下是安装和运行 TRCA 的步骤。

1.  安装 TRCA 非常简单，这里我们只展示安装摘要：
    ```
    SQL> @tacreate.sql
    Uninstalling TRCA, please wait
    TADOBJ completed.
    SQL>
    SQL> WHENEVER SQLERROR EXIT SQL.SQLCODE;
    SQL> REM If this DROP USER command fails that means a session is connected with this user.
    SQL> DROP USER trcanlzr CASCADE;
    SQL> WHENEVER SQLERROR CONTINUE;
    SQL>
    SQL> SET ECHO OFF;
    TADUSR completed.
    TADROP completed.

    Creating TRCA$ INPUT/BDUMP/STAGE Server Directories
    ...

    TACREATE completed. Installation completed successfully.
    SQL>
    ```
2.  设置跟踪。
    ```
    SQL> alter session set events '10046 trace name context forever, level 12';

    System altered.

    SQL>
    ```
3.  执行要跟踪的 SQL 语句。
    ```
    SQL> select ...
    ```
4.  关闭跟踪。
    ```
    SQL> alter session set events '10046 trace name context off';
    System altered.

    SQL>
    ```
5.  运行 `/trca/run/trcanlzr` 脚本（`START trcanlzr.sql`）来分析刚刚生成的跟踪文件。您必须将跟踪文件名作为输入传递给此脚本：
    ```
    c:\trace\trca\trca\run>sqlplus hr/hr
    SQL> START trcanlzr.sql orcl1_ora_7460_mytrace7.trc
    Parameter 1:
    Trace Filename or control_file.txt (required)
    Value passed to trcanlzr.sql:
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    TRACE_FILENAME: orcl1_ora_7460_mytrace7.trc
    Analyzing orcl1_ora_7460_mytrace7.trc
    ... analyzing trace(s) ...
    Trace Analyzer completed.
    Review first trcanlzr_error.log file for possible fatal errors.
    TKPROF: Release 11.2.0.1.0 - Development on Sat May 14 15:59:13 2011
      …
       233387  05/14/2011 15:59   trca_e21106.html
       115885  05/14/2011 15:59   trca_e21106.txt
    File trca_e21106.zip has been created
    TRCANLZR completed.
    SQL>
    c:\trace\trca\trca\run>
    ```

现在，您可以在文本或 HTML 格式中查看分析后的跟踪数据——TRCA 在完成跟踪文件分析后创建的 ZIP 文件中提供了这两种格式。TRCA 将 ZIP 文件放在运行 `/trca/run/trcanlzr.sql` 脚本的目录中。

## 工作原理

Oracle Support Center of Expertise (CoE) 提供了 TRCA 诊断工具。虽然许多 DBA 知道 TRCA，但很少有人定期使用它。我们中的一些人只是在 Oracle Support 要求时才使用它。正如您在“解决方案”部分学到的，TRCA 工具接受您生成的 SQL 跟踪，并以文本和 HTML 格式输出诊断报告。TRCA 工具还为您提供 `TKPROF` 报告（它在诊断数据收集中执行 `tkprof` 命令）。由于 TRCA 提供了丰富的诊断信息，请考虑使用它来代替 `TKPROF`。

除了 `TKPROF` 工具通常收集的数据外，TRCA 还会识别昂贵的 SQL 语句并收集它们的执行计划。它还显示优化器统计信息和配置参数，这些参数对跟踪中 SQL 语句的性能有影响。

![images](img/square.jpg) **提示** 使用 TRCA 而不是 `TKPROF` 来分析您的跟踪文件——它不仅为您提供丰富的诊断信息，还附带赠送一个 `TKPROF` 输出文件。

在我们的示例中，我们在生成跟踪的同一系统上使用 TRCA 来格式化跟踪文件。但是，如果您无法在生产系统上安装 TRCA，不用担心。TRCA 也可以使用不同的系统分析生产环境中的跟踪。详细信息在 `trca_instructions` HTML 文档中，该文档是 TRCA 下载 ZIP 文件的一部分。

以下是 TRCA 报告的主要部分，正如您已经看到的，该报告提供了比 `TKPROF` 更丰富的诊断信息。

> *摘要*：提供耗时、分解为 CPU 和非空闲等待时间的响应时间以及其他与响应时间相关的信息的细分。
>
> *非递归时间和总计*：提供解析、执行和获取步骤期间的响应时间和耗时的细分；报告还包含一个表，提供每个空闲和非空闲等待事件的总计和平均等待时间。
>
> *Top SQL*：提供有关占用最多响应时间、耗时和 CPU 时间的 SQL 语句的详细信息，如下方报告摘录所示：
> ```
> There are 2 SQL statements with "Response Time Accounted-for" larger than threshold of 10.0% of the "Total Response Time Accounted-for"
> These combined 2 SQL statements are responsible for a total of 99.3% of the "Total Response Time Accounted-for".
>
> There are 3 SQL statements with "Elapsed Time" larger than threshold of 10.0% of the "Total Elapsed Time".
> These combined 3 SQL statements are responsible for a total of 75.5% of the "Total Elapsed Time".
>
> There is only one SQL statement with "CPU Time" larger than threshold of 10.0% of the "Total CPU Time".
> ```
> *单个 SQL*：这是一个非常有用的部分，它列出了所有 SQL 语句并显示了它们的耗时、响应时间和 CPU 时间。它提供了每条语句的哈希值和 SQL ID。
>
> *SQL 自身时间、总计、等待、绑定和行源计划*：显示每条语句的解析、执行和获取统计信息，类似于 `TKPROF` 工具；它还显示了每条语句的等待事件细分（平均和总时间）。每条语句还有一个非常好的执行计划，显示了每个执行步骤的时间和成本。
>
> *表和索引*：显示行数、分区状态、样本大小以及对象上次分析的时间；对于索引，它还额外显示聚簇因子和键数。
>
> *摘要*：显示与 I/O 相关的等待（例如 db file sequential read 事件）信息，包括表的平均和总等待时间。
>
> *热 I/O 块*：显示等待时间最长或等待次数最多的块列表。
>
> *非默认初始化参数*：列出所有非默认初始化参数。

正如对 TRCA 的简要回顾所示，它是一个比 `TKPROF` 优越得多的工具。此外，如果您碰巧喜欢 `TKPROF` 报告，它的 ZIP 文件中也包含了这些报告。那么，您还在等什么？下载 TRCA 并利用它对问题 SQL 语句的丰富诊断分析能力吧。

## 问题

您希望追踪一个并行查询。


#### 解决方案

获取并行查询的 10046 跟踪文件的方式与获取任何其他查询的方式相同。唯一的区别是，10046 事件将生成与并行查询服务器数量一样多的跟踪文件。下面是一个示例：

```sql
SQL> alter session set tracefile_identifier='MyTrace1';

SQL> alter session set events '10046 trace name context forever, level 12';

Session altered.

SQL> select /*+ full(sales) parallel (sales 6) */ count(quantity_sold) from
     sales;

COUNT(QUANTITY_SOLD)
--------------------
              918843

SQL> alter session set events '10046 trace name context off';

Session altered.

SQL>
```

现在，你将在跟踪目录中看到总共七个带有跟踪文件标识符 `MyTrace1` 的跟踪文件。根据你要查找的内容，你可以分别分析每个跟踪文件，或者在用 `TKPROF` 或 Oracle Trace Analyzer 等其他分析器进行分析之前，使用 `trcsess` 工具将它们合并成一个大的跟踪文件。你还会在跟踪目录中找到一些后缀为 `.trm` 的文件——可以忽略这些文件，因为它们是供数据库使用的。

#### 工作原理

获取单个查询的扩展跟踪与获取并行查询的扩展跟踪之间，唯一真正的区别在于你会得到多个跟踪文件，每个并行查询服务器对应一个。当用户执行并行查询时，Oracle 会创建多个并行查询进程来处理该查询，每个进程都有自己的会话。这就是 Oracle 为并行查询创建多个跟踪文件的原因。

关闭跟踪后，转到跟踪目录并执行以下命令以查找并行查询的所有跟踪文件：

```bash
$ find . -name '*MyTrace1*'
```

`find` 命令会列出你并行查询的所有跟踪文件（忽略跟踪目录中以 `.trm` 结尾的文件）。你可以将跟踪文件移动到另一个目录，并使用 `trcsess` 工具合并这些文件，如下所示：

```bash
$ trcsess output=MyTrace1.trc clientid='px_test1' orcl1_ora_8432_mytrace1.trc orcl1_ora_8432_mytrace2.trc
```

现在，你已准备好使用 `TKPROF` 工具来分析并行查询。

当你发出并行查询时，并行执行协调器/查询协调器 (`QC`) 控制查询的执行。并行执行服务器/从属进程 (`QS`) 执行实际的工作。并行执行服务器*集*是执行某个操作的所有查询服务器的集合。查询协调器和每个执行服务器都会生成自己的跟踪文件。例如，如果某个从属进程在等待资源，数据库会将由此产生的等待事件记录在该从属进程的跟踪文件中，而不是查询协调器的跟踪文件中。

请注意，如果你使用的是 10.2 或更早的版本，用户进程的跟踪文件将创建在用户转储目录中，而后台进程（从属进程）将在后台转储目录中生成跟踪文件。在 11.1 及更新版本的数据库中，后台和用户进程的跟踪文件都在同一个目录（trace）中。你可以在调用 `ADRCI` 后执行 `show tracefile -t` 命令，或者通过 SQL*Plus 查询 `V$DIAG_INFO` 视图来查找跟踪文件名。

### 10-11. 跟踪特定的并行查询进程

#### 问题

你想要跟踪一个或多个特定的并行查询进程。

#### 解决方案

使用以下命令识别你想要跟踪的并行查询进程。

```sql
SQL> select inst_id,p.server_name,
     p.status as p_status,
     p.pid as p_pid,
     p.sid as p_sid
     from  gv$px_process p
     order by p.server_name;
```

假设你决定跟踪进程 `p002` 和 `p003`。发出以下 `alter system set events` 命令以仅跟踪这两个并行进程。

```sql
SQL> alter system set events 'sql_trace  {process: pname = p002 | p003}';
```

跟踪完成后，发出以下命令关闭跟踪：

```sql
SQL> alter system set events 'sql_trace  {process: pname = p002 | p003} off';
```

#### 工作原理

跟踪并行进程总是很棘手。Oracle Database 11g 版本对跟踪基础设施的一项改进是能够跟踪特定语句或一组语句。当某个会话正在执行大量 SQL 语句，并且你确切知道要跟踪其执行的 SQL 语句的身份时，这一功能就派上用场了。

### 10-12. 在 RAC 系统中跟踪并行查询

#### 问题

你正在 RAC 环境中跟踪一个并行查询，但不确定跟踪文件位于哪个实例中。

#### 解决方案

在 RAC 环境中查找服务器（或线程或从属）进程的跟踪文件有时很困难，因为你不确定数据库在哪个或哪些节点上创建了跟踪文件。请按照以下步骤操作，以便更轻松地在不同节点上查找跟踪文件。

> 1.  使用 `alter session` 命令设置 `px_trace`，以帮助识别跟踪文件，如下所示：
>     ```sql
>     SQL> alter session set tracefile_identifier='10046';
>     SQL> alter session set  "_px_trace" = low , messaging;
>     SQL> alter session set events '10046 trace name context forever,level 12';
>     ```
> 2.  执行你的并行查询。
>     ```sql
>     SQL> alter table bigsales (parallel 4);
>     SQL> select count(*) from bigsales;
>     ```
> 3.  关闭所有跟踪。
>     ```sql
>     SQL> alter session set events '10046 trace name context off';
>     SQL> alter session set "_px_trace" = none;
>     ```

指定 `px_trace` 将导致查询协调器的跟踪文件包含有关作为查询一部分的从属进程以及每个从属进程所属实例的信息。然后，你可以从查询协调器跟踪文件中列出的实例中检索跟踪文件。

#### 工作原理

`_px_trace` (px trace) 参数是一个未文档化的内部 Oracle 参数，自 9.2 版本起就已存在。一旦你运行了本“解决方案”部分中所示的跟踪命令，查询协调器 (`QC`) 进程的跟踪文件将显示其中运行的每个从属进程的名称以及这些进程运行所在的实例——例如：

```
Acquired 4 slaves on 1 instances avg height=4 in 1 set q serial:2049
        P000 inst 1 spid 7512
        P001 inst 1 spid 4088
        P002 inst 1 spid 7340
        P003 inst 1 spid 9256
```

在这种情况下，你知道需要在实例 1 中查找从属进程 `P000`、`P001`、`P002` 和 `P003` 的跟踪文件。在实例 1 上，在 ADR 跟踪子目录中，查找文件名中包含 `P000`（或 `P001`/`P002`/`P003`）字样的文件，以识别正确的跟踪文件。

### 10-13. 合并多个跟踪文件

#### 问题

你已经为某个会话生成了多个跟踪文件以调整性能，并希望将这些文件合并成一个跟踪文件。

#### 解决方案

使用 `trcsess` 命令将多个跟踪文件合并成一个跟踪文件。下面是一个简单的例子：

```bash
c:\trace> trcsess output=combined.trc session=196.614 orcl1_ora_8432_mytrace1.trc orcl1_ora_8432_mytrace2.trc
C:\trace>
```

此处显示的 `trcsess` 命令将为某个会话生成的两个跟踪文件合并为一个跟踪文件。`session` 参数使用会话标识符（由会话索引和会话序列号组成）来标识会话，你可以从 `V$SESSION` 视图中获取该标识符。



