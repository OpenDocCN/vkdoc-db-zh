# DBMS_PROFILER 与 DBMS_TRACE 示例会话

本节提供了一个同时使用 `DBMS_PROFILER` 和 `DBMS_TRACE` 进行性能分析和跟踪的示例会话。本节中展示的所有跟踪和性能分析信息均通过此处提供的代码生成：

```sql
DECLARE
  v_employee_id employees.employee_id%TYPE;
BEGIN
  -- 启动性能分析会话
  dbms_profiler.start_profiler;
  dbms_trace.set_plsql_trace(
                   DBMS_TRACE.trace_enabled_calls
                   +
                   DBMS_TRACE.trace_enabled_exceptions
                   +
                   DBMS_TRACE.trace_all_sql
                   +
                   DBMS_TRACE.trace_all_lines);

  -- 调用要进行性能分析的顶级过程
  v_employee_id := employee_api.create_employee(
        p_first_name => 'Lewis',
        p_last_name => 'Cunningham',
        p_email => 'lewisc@yahoo.com',
        p_phone_number => '888-555-1212',
        p_job_title => 'Programmer',
        p_department_name => 'IT',
        p_salary => 99999.99,
        p_commission_pct => 0 );
  -- 停止性能分析会话
  dbms_trace.clear_plsql_trace;
  dbms_profiler.stop_profiler;

END;
```

`DBMS_PROFILER.start_profiler` 开始性能分析会话，`DBMS_TRACE.set_plsql_trace` 启动事件跟踪。这两个过程都接受可选参数作为输入。`DBMS_TRACE` 的参数 `TRACE_LEVEL` 决定了收集信息的多少。在本例中，我需要为所有启用的调用（对使用 `DEBUG` 编译的程序单元的调用）、所有启用的异常、所有 SQL 调用以及所有行创建记录。这在大多数情况下确实有点过度了。输出如下所示。

## DBMS_PROFILER 与 DBMS_TRACE 输出

运行时，`DBMS_TRACE` 将数据编译到两个表中。这些表如下：

*   `PLSQL_TRACE_RUNS` – `DBMS_TRACE`：标识每个独立运行的顶级表。
*   `PLSQL_TRACE_EVENTS` – `DBMS_TRACE`：一个非常有用的表，可以记录跟踪会话中执行的每一行，可能迅速变得非常大，需要监控。

运行时，`DBMS_TRACE` 将数据编译到以下表中：

*   `PLSQL_PROFILER_RUNS` – `DBMS_PROFILER`：标识每个独立性能分析会话的顶级表。
*   `PLSQL_PROFILER_UNITS` – `DBMS_PROFILER`：描述在性能分析会话期间执行的程序单元。
*   `PLSQL_PROFILER_DATA` – `DBMS_PROFILER`：会话期间每个单元运行的计时和执行统计信息。

你可以通过 `RUN_DATE` 和 `RUN_COMMENT`（可选；默认为会话启动时的 `SYSDATE`）来标识一个 `DBMS_PROFILER` 运行。以下查询显示了两个性能分析会话后存储在 `PLSQL_PROFILER_RUNS` 表中的数据：

```sql
select runid, run_date, run_comment, run_total_time
from plsql_profiler_runs;
```

```
RUNID     RUN_DATE               RUN_COMMENT      RUN_TOTAL_TIME
---------- --------------------- ---------------- ----------------------
2          13-MAR-2011 11:56:54  13-MAR-11        133000000
3          13-MAR-2011 12:30:18  13-MAR-11        80000000
```

从该输出中，你可以识别出你正在寻找的运行。在分析性能时，你通常会对同一程序代码的多次运行取平均值。为了本示例的需要，我将使用一个特定的程序运行。

标识跟踪会话与标识性能分析会话非常相似。会话通过 `RUNID` 和 `RUN_DATE` 来标识。以下查询显示了跟踪会话后存储在 `PLSQL_TRACE_RUNS` 表中的数据：

```sql
SELECT runid, run_date
FROM plsql_trace_runs;
```

```
RUNID                  RUN_DATE
---------------------- -------------------------
10                     26-MAR-2011 11:04:35
```

`DBMS_PROFILER` 提供关于运行中可执行行的计时信息。你收集这些信息，然后分析它以确定代码中是否存在瓶颈。运行时间长的代码可能是没问题的；但也可能不是。这个判断来自你的分析。性能分析器无法告诉你哪里有性能问题。性能分析器只能给你一个用于分析的性能剖析。一旦你分析了输出并进行了任何更改，你就需要运行另一个性能分析会话来验证你的更改。

以下查询逐行列出了性能分析器收集的计时信息。我删除了多行输出以控制篇幅。我忽略了 `ANONYMOUS BLOCK` 条目，因为它们是运行本身的噪声。

```sql
SELECT unit_name, unit_type, line#, total_occur, ppd.total_time, max_time
  FROM plsql_profiler_units ppu
  JOIN plsql_profiler_data ppd
    ON (ppu.runid = ppd.runid
       AND
       ppu.unit_number = ppd.unit_number)
WHERE ppu.runid = :RUNID
  AND unit_type != 'ANONYMOUS BLOCK'  ;
```

```
UNIT_NAME           UNIT_TYPE      LINE#    TOTAL_OCCUR   TOTAL_TIME    MAX_TIME
------------------- -------------- -------- ------------- ------------- -------------
EMPLOYEE_API        PACKAGE BODY   3        1             11982         9985
EMPLOYEE_API        PACKAGE BODY   20       1             1997          1997
EMPLOYEE_API        PACKAGE BODY   21       1             2995          2995
DEPARTMENTS_API     PACKAGE BODY   3        1             7988          6989
DEPARTMENTS_API     PACKAGE BODY   8        1             0             0
JOBS_API            PACKAGE BODY   3        1             7988          6989
JOBS_API            PACKAGE BODY   7        1             0             0
.
.
.
80 行 已选中
```


此列表显示了每个被分析程序单元中执行的代码行。它提供了该行的执行次数以及在该行花费的总时间（包括对被调用程序单元的调用）。`max time` 是在任何特定调用上花费的最长时间。利用这些信息，你应该能够很好地了解从何处开始调查任何性能问题。

## 使用 DBMS_TRACE 查看调用层次

虽然性能指标很棒，但如果你能知道这些时间数据的上下文或调用层次结构，那将会更好。`DBMS_TRACE` 并不能完全提供这个，但它确实提供了调用结构。要查看已执行代码的层次结构（请记住，这是实际的执行，而非预期的或可能的执行），你可以运行以下这个非常简单的查询：

```sql
SELECT event_unit, event_proc_name,
       CASE
         WHEN proc_unit IS NOT NULL and proc_name IS NOT NULL
           THEN proc_unit || '.' || proc_name || '(' || proc_line || ')'
         WHEN proc_unit IS NULL and proc_name IS NOT NULL
           THEN proc_name || '(' || proc_line || ')'
         ELSE
           substr(event_comment,1,30)
       END executed_code
FROM plsql_trace_events pse
WHERE runid = :RUNID
  AND event_unit NOT IN ('<anonymous>', 'DBMS_TRACE')
  AND event_kind NOT IN (50, 51)
ORDER BY event_seq;
```

```
EVENT_UNIT        EVENT_PROC_NAME                 EXECUTED_CODE
---------------- ------------------------------- ------------------------------------------
EMPLOYEE_API      CREATE_EMPLOYEE                 DEPARTMENTS_API.DEPARTMENT_EXISTS(3)
DEPARTMENTS_API   DEPARTMENT_EXISTS               SELECT DEPARTMENT_ID FROM DEPA
EMPLOYEE_API      CREATE_EMPLOYEE                 JOBS_API.JOB_EXISTS(3)
JOBS_API          JOB_EXISTS                      SELECT JOB_ID FROM JOBS WHERE
EMPLOYEE_API      CREATE_EMPLOYEE                 JOBS_API.VALIDATE_JOB(21)
EMPLOYEE_API      CREATE_EMPLOYEE                 DEPARTMENTS_API.DEPARTMENT_MANAGER_ID(29)
DEPARTMENTS_API   DEPARTMENT_MANAGER_ID           SELECT MANAGER_ID FROM DEPARTM
EMPLOYEE_API      CREATE_EMPLOYEE                 Select EMPLOYEES_SEQ.NEXTVAL f
EMPLOYEE_API      CREATE_EMPLOYEE                 INSERT INTO EMPLOYEES (EMPLOYE
EMPLOYEE_API      CREATE_EMPLOYEE                 EMPLOYEE_API.ASSIGN_SALARY(74)
EMPLOYEE_API      ASSIGN_SALARY                   UPDATE EMPLOYEES SET SALARY =

 11 rows selected
```

此列表（以及查询）提供了从本次运行中得到的执行顺序。`EMLOYEE_API.CREATE_EMPLOYEE` 调用了 `DEPARTMENTS_API.DEPARTMENT_EXISTS`（它在 `DEPARTMENTS_API` 包体中定义于第 3 行）。被调用的程序单元在返回前运行了一个查询。

## 可扩展性考量：DBMS_TRACE 与 DBMS_PROFILER

`DBMS_TRACE` 可能会产生非常冗长的输出。本节中我使用的示例相当简单，属于后台办公类应用。当你拥有非常庞大的应用程序，或者需要运行较长时间并反复调用相同程序单元的应用程序时，随着复杂性的增加，`DBMS_TRACE` 的价值就开始下降。

例如，本章提供了一个名为 `TEST_CALLS_PRC` 的演示过程。它是一个简单的 `for` 循环（100 次迭代），每次迭代调用两个过程。使用与之前相同的选项运行 `DBMS_TRACE` 和 `DBMS_PROFILER`，比较两者输出行数的差异（较小的 `RUNID` 是之前两者计数的追踪记录）。以下查询比较了由 `DBMS_PROFILER` 产生的两次运行之间的输出行数：

```sql
SELECT ppd.runid, count(*)
  FROM plsql_profiler_units ppu
  JOIN plsql_profiler_data ppd
    ON(ppu.runid = ppd.runid
    AND ppu.unit_number = ppd.unit_number)
  WHERE ppu.runid IN (9,10)
    AND ppd.total_time != 0
    AND unit_type != 'ANONYMOUS BLOCK'
  GROUP BY ppd.runid;
```

```
RUNID                  COUNT(*)
---------------------- ----------------------
9                      29
10                     15
```

下一个查询对两次 `DBMS_TRACE` 会话产生相同的行数统计：

```sql
  SELECT runid, count(*)
    FROM plsql_trace_events
    WHERE runid IN (12,13)
    GROUP BY runid;
```

```
RUNID                  COUNT(*)
---------------------- ----------------------
12                     100
13                     5545
```

对于不太复杂的代码，`DBMS_PROFILER` 的行数较少，但 `DBMS_TRACE` 的行数却多得多。在针对 `DBMS_TRACE` 表编写查询时，你需要考虑到这一点。使用选项只收集你真正想要的数据，然后计划创造性地过滤掉那些你不想要但无法通过选项排除的内容。



### 代码覆盖测试与性能指标

代码覆盖测试是一种测试方法，用于验证程序中每一条`可执行`的代码行是否确实被执行了。这种测试属于**回归测试**，而非功能测试。某段代码被执行过，并不意味着它被正确地执行了。然而，如果它至少被执行过一次，那么你就知道它实际上在特定情况下能够无错误地运行。

代码覆盖测试本身意义不大。但作为自动化构建流程的一部分，或作为一个完整测试计划的一个组成部分，它确实有其用处。我喜欢使用代码覆盖测试，理由和我喜欢编译器错误一样——它能发现大量小错误。不过，如果没有合适的工具，进行这种测试会非常困难。幸运的是，Oracle 已为我们提供了所需的大部分工具。

我在这里并不特别关注性能指标（尽管没有理由不将其包括在内）。我希望看到的是，所有可执行的代码都被执行了。

可以将 `DBMS_TRACE` 和 `DBMS_PROFILER` 的输出结合起来，以获取每行已执行代码及其上下文的一个清晰跟踪。要做到这一点，需要创建一个巧妙的查询，并了解哪些运行是相关的（因为它们是一起运行或在非常接近的时间运行的）。如果将不同版本代码的分析器运行和跟踪运行混在一起，你会得到一团非常混乱的输出。但如果你知道这些运行是匹配的，那么这些信息就非常有用。

下面的查询连接了 `DBMS_PROFILER` 的输出和 `DBMS_TRACE` 的输出。为了简化查询，我省略了 `PLSQL_TRACE_RUNS` 和 `PLSQL_PROFILER_RUNS` 表——这些才是你实际应该开始的地方。我只是手动从两者中各选择了一次运行。我过滤掉了特定的事件类型，具体是 SQL 调用 (54) 和过程调用返回 (50)。你可以从文档或 `tracetab.sql` 安装代码中获取所有事件类型的列表。在查看 `total_time` 时，我喜欢了解谁被调用、从哪里调用以及代码具体做了什么。"谁"和"哪里"的信息来自 `DBMS_TRACE` 数据，而"什么"则来自 `USER_SOURCE`。查询如下：

```sql
BREAK ON event_unit ON event_proc_name
SELECT event_unit, event_proc_name, line# line,
       total_time time, pse.proc_unit, pse.proc_name,
       substr(ltrim(rtrim(text)),1,30) source_text
FROM (
  (SELECT ppu.runid, ppu.unit_type, ppu.unit_name,
          ppd.line#, ppd.total_time
     FROM plsql_profiler_units ppu
     JOIN plsql_profiler_data ppd
       ON(ppu.runid = ppd.runid
       AND ppu.unit_number = ppd.unit_number)
     WHERE ppu.runid = :PROF_RUNID
       AND ppd.total_time != 0)) prof_data
JOIN (
    SELECT event_unit, event_proc_name, event_line,
           proc_unit, proc_name, event_unit_kind,
           event_kind, min(event_seq) event_seq
    FROM plsql_trace_events trace_data
    WHERE runid = :TRACE_RUNID
      AND event_kind NOT IN (50, 54)
    GROUP BY  event_unit, event_proc_name, event_line,
           proc_unit, proc_name, event_unit_kind,
           event_kind) pse
  ON (pse.event_unit = prof_data.unit_name
  AND pse.event_line = prof_data.line#)
JOIN user_source us
  ON (us.name = pse.event_unit
  AND us.type = pse.event_unit_kind
  AND us.line = pse.event_line)
ORDER BY event_seq;
```

```
EVENT_UNIT         EVENT_PROC_NAME          LINE    TIME PROC_UNIT
------------------ ----------------------- ----- ------- -----------------
PROC_NAME               SOURCE_TEXT
----------------------- ------------------------------
EMPLOYEE_API       CREATE_EMPLOYEE             3   12933
                        FUNCTION create_employee(

                                             20    2984
                        v_exception BOOLEAN := FALSE;

                                             21    3979
                        v_exception_message CHAR(2048)

                                             24  247720
                        v_department_id := departments

                                             24  247720 DEPARTMENTS_API
DEPARTMENT_EXISTS       v_department_id := departments

DEPARTMENTS_API    DEPARTMENT_EXISTS           3    7958
                        FUNCTION department_exists(

                                               9  185044
                        SELECT department_id

                                              14    1989
                        RETURN v_department_id;

EMPLOYEE_API       CREATE_EMPLOYEE            24  247720
                        v_department_id := departments
.
.
.
38 rows selected.
```

将这个输出格式化以适应页面相当困难，但希望你能看出其中的端倪。输出从包体 `EMPLOYEE_API.CREATE_EMPLOYEE` 的第 3 行开始。该行的执行时间是 12,933，实际执行的代码（至少其前 30 个字节）是 `FUNCTION create_employee`。

输出的第二行对应于包体的第 20 行，是一个赋值操作，耗时 2,984。你可以将此输出中的代码与实际源代码进行比较，可以看到第 20 行是第一个可执行行。它之前的所有内容都是参数和声明。这个赋值操作（虽然是在声明部分）使其成为可执行行。第 20 行之后是第 21 行，这是另一个赋值操作。

最后，在第 24 行，通过调用 `DEPARTMENTS_API.DEPARTMENT_EXISTS` 进行了一次赋值。这使得 `EVENT_UNIT` 从 `EMPLOYEE_API` 变成了 `DEPARTMENTS_API`。接下来的几行是该程序单元中的可执行行，直到到达 `DEPARTMENTS_API` 包体的第 14 行的返回语句。显示的最后一行将控制权交还给 `EMPLOYEE_API`。为了节省空间，其余行被省略了。

在一份报告/查询中，你就得到了一份代码覆盖报告、一份代码分析报告（因为包含了可执行代码、行号、计时信息和实际源代码），以及性能指标。如果将此报告保存到一个表中（你自建的用户表），你可以在代码的任何更改之后重新运行此查询并比较输出。你只需要注意如何比较单个行，因为它们会随着时间的推移而改变。



