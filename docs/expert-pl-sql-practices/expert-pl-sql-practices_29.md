# PL/Scope 代码分析与命名标准验证

### 查询结构与说明

```sql
SELECT code_info( ai.owner,
                  prior_object_name,
                  CASE
                  WHEN ai.object_type IN ('PROCEDURE', 'FUNCTION')
                  THEN NULL
                  ELSE
                    ai.name
                  END,
                  ai.object_type,
                  'CALL',
                  cnctby_vw.line,
                  cnctby_vw.p_col ) code_value
        FROM all_identifiers ai, (
   SELECT usage, level, usage_id, usage_context_id, PRIOR usage_id p_usage_id,
          PRIOR usage_context_id prior_usagectx_id,
          object_name, name, object_type, type, line, PRIOR col p_col,
          PRIOR object_name prior_object_name, PRIOR name p_name,
          PRIOR object_type prior_object_type
      FROM all_identifiers
      WHERE OWNER = p_owner
        AND OBJECT_TYPE IN ('PACKAGE BODY', 'PROCEDURE', 'FUNCTION', 'TRIGGER', 'TYPE BODY')
        AND USAGE NOT IN ('DECLARATION', 'ASSIGNMENT', 'DEFINITION')
        AND type IN ('PACKAGE BODY', 'FUNCTION', 'PROCEDURE', 'TYPE', 'TYPE BODY')
      CONNECT BY PRIOR usage_id = usage_context_id
         AND PRIOR name = p_object
         AND name = p_unit
         AND prior usage_context_id != 0 )  cnctby_vw
  WHERE ai.usage_id = cnctby_vw.prior_usagectx_id
    AND ai.object_name = cnctby_vw.prior_object_name
    AND ai.object_type = cnctby_vw.prior_object_type  )
```

内部查询 (`cnctby_vw`) 是一个层次查询，用于获取已更改的程序单元信息。此查询专门查找包调用。它很容易修改为查找任何类型的调用；这个练习留给您完成。将其修改为查找过程或函数调用比查找包调用或对象方法更容易。

外部查询将已更改的单元（被调用的程序单元）与子单元（包中的过程或函数）联系起来，或者如果它是独立的，则与独立单元联系起来。

在使用此查询作为基础创建您自己的影响分析查询之前，我建议您从内部查询开始并试用一段时间。该基本查询可以修改为返回 PL/Scope 提供的几乎任何两个相关项。

##### 验证命名标准合规性

与 `USER_PROCEDURES`、`USER_ARGUMENTS` 和 `USER_TYPE` 系列视图类似，PL/Scope 可以提供有关命名标准合规性的信息。不过，PL/Scope 更胜一筹，因为您可以验证包内部变量的命名以及过程和函数主体中的命名，而不仅仅是参数和程序单元的命名。

以下查询将查找所有不以 'V_' 开头的变量和不以 'P_' 开头的参数。它可以使用代码分析部分中的层次查询进行扩展，以验证全局变量和子类型。

```sql
SELECT  ui_var.name, type, usage, line, col
  FROM user_identifiers ui_var
  WHERE ui_var.object_name = 'JOBS_API'
    AND ui_var.object_type = 'PACKAGE BODY'
    AND (
          (ui_var.type = 'VARIABLE' AND name NOT LIKE 'V_%')
      OR
          (ui_var.type IN ('FORMAL IN', 'FORMAL OUT', 'FORMAL IN OUT')
              AND name NOT LIKE 'P_%')
        )
    AND ui_var.usage = 'DECLARATION'
  ORDER BY line, col;
```

```
NAME                TYPE               USAGE       LINE       COL
------------------- ------------------ ----------- ---------- -------
JOB_TITLE           FORMAL IN          DECLARATION 4          5
```

如果您查看 `JOBS_API` 中从第 4 行开始的代码，您将找到以下代码：

```sql
  FUNCTION job_exists(
    job_title  IN jobs.job_title%TYPE )
  RETURN jobs.job_id%TYPE AS
    v_job_id jobs.job_id%TYPE;
  BEGIN
    SELECT job_id
      INTO v_job_id
      FROM jobs
      WHERE UPPER(job_title) = upper(job_title);
```

从这段代码中，您可以看到它确实有一个名为 `JOB_TITLE` 的参数。查看代码中的查询，您会发现该查询有一个问题。问题出在 `WHERE` 子句中，其中引用了相同的列名。编写该函数的人本意是将表列与参数进行比较，但实际发生的比较并非如此。以下是那个有问题的比较：

```sql
WHERE UPPER(job_title) = UPPER(job_title);
```

如果有人运行该 `SELECT`，表中的每一条记录都会匹配（SQL 解析器将使用表中的 `job_title` 而不是 `job_title` 参数）。修复方法是按如下方式重写函数规范，向参数名添加一个 `p_`：

```sql
FUNCTION job_exists(
    p_job_title  IN jobs.job_title%TYPE )
  RETURN jobs.job_id%TYPE;
```

修改 `WHERE` 子句中的过滤条件，添加到参数上的 `p_`，像这样：

```sql
WHERE UPPER(job_title) = upper(p_job_title);
```

该查询无法成功运行的原因是，就 Oracle 而言，过滤条件是在比较 `job_title` 列中的值与 `job_title` 列中的值。这等同于条件 `WHERE 1=1`。结果是表中的每一行都会被返回——这显然不是预期的结果。通过向参数及其使用添加 `p_` 前缀，过滤器将只返回表中职位名称与传入函数的 `job_title` 匹配的那一行。


## 查找未使用的标识符

许多组织对变量使用有标准，例如“禁止全局变量”和/或“所有变量必须被使用”。我个人并不完全赞同禁止全局变量的规则（尽管全局变量的使用应该被仔细考量）。然而，我确实同意，程序中存在未使用的变量会使维护变得更加复杂（主要是杂乱）。

使用 PL/Scope，很容易查看在任何作用域中，哪些变量没有被使用。这意味着你有一个声明但没有引用。接下来的几个查询将逐步构建，最终找到一个存储过程，该过程定义了一个未使用的全局受保护变量、一个未使用的参数和一个未使用的局部变量。

首先，列出包体中所有已声明的变量（注意，这不会获取规范中定义的全局变量，只会获取在体中定义的受保护全局变量），如下所示：

```sql
SELECT  ui_var.name, type, usage, line, col
  FROM user_identifiers ui_var
  WHERE ui_var.object_name = 'VARIABLES_PKG'
    AND ui_var.object_type = 'PACKAGE BODY'
    AND ui_var.type IN ('VARIABLE', 'FORMAL IN', 'FORMAL OUT', 'FORMAL IN OUT')
    AND ui_var.usage = 'DECLARATION'
  ORDER BY line, col;
```

查询结果如下：
```
NAME                         TYPE           USAGE       LINE     COL
---------------------------- -------------- ----------- -------- -----
G_PROTECTED_VARIABLE_USED    VARIABLE       DECLARATION 3        3
G_PROTECTED_VARIABLE_UNUSED  VARIABLE       DECLARATION 4        3
P_KEYNUM                     FORMAL IN      DECLARATION 7        5
P_DATE                       FORMAL IN OUT  DECLARATION 8        5
P_LOB                        FORMAL OUT     DECLARATION 9        5
V_USED_VARIABLE              VARIABLE       DECLARATION 11       5
V_UNUSED_VARIABLE            VARIABLE       DECLARATION 12       5
```

验证所有这些参数/变量是否被使用的最简单方法是检查它们是否被引用。然而，可能存在只进行赋值却从未引用变量的情况（这有什么用呢？），也存在变量在声明的同一步骤中被赋值的情况。对我来说，我想要确保所有变量都被使用（这意味着无论是否赋值，都必须有引用）。

要查看被引用的变量，只需将使用情况从 `DECLARATION` 更改为 `REFERENCE`，如下所示的查询：

```sql
SELECT  ui_var.name, type, usage, line, col
  FROM user_identifiers ui_var
  WHERE ui_var.object_name = 'VARIABLES_PKG'
    AND ui_var.object_type = 'PACKAGE BODY'
    AND ui_var.type IN ('VARIABLE', 'FORMAL IN', 'FORMAL OUT', 'FORMAL IN OUT')
    AND ui_var.usage = 'REFERENCE'
  ORDER BY line, col;
```

查询结果如下：
```
NAME                         TYPE        USAGE       LINE      COL
---------------------------- ----------- ----------- --------- -----
P_KEYNUM                     FORMAL IN   REFERENCE   14        24
V_USED_VARIABLE              VARIABLE    REFERENCE   16        22
G_PROTECTED_VARIABLE_USED    VARIABLE    REFERENCE   16        40
P_LOB                        FORMAL OUT  REFERENCE   17        39
```

这显示有三个变量缺失。使用简单的 SQL，很容易找到它们。这个最终查询返回那些已声明但未被使用的变量：

```sql
SELECT  name, type, usage, line, col
  FROM user_identifiers
  WHERE object_type = 'PACKAGE BODY'
    AND usage = 'DECLARATION'
    AND (name, type) IN (
SELECT  ui_var.name, type
  FROM user_identifiers ui_var
  WHERE ui_var.object_name = 'VARIABLES_PKG'
    AND ui_var.object_type = 'PACKAGE BODY'
    AND ui_var.type IN ('VARIABLE', 'FORMAL IN', 'FORMAL OUT', 'FORMAL IN OUT')
    AND ui_var.usage = 'DECLARATION'
MINUS
  SELECT  ui_var.name, type
  FROM user_identifiers ui_var
  WHERE ui_var.object_name = 'VARIABLES_PKG'
    AND ui_var.object_type = 'PACKAGE BODY'
    AND ui_var.type IN ('VARIABLE', 'FORMAL IN', 'FORMAL OUT', 'FORMAL IN OUT')
    AND ui_var.usage = 'REFERENCE')
  ORDER BY line, col;
```

查询结果如下：
```
NAME                         TYPE               USAGE       LINE     COL
---------------------------- ------------------ ----------- -------- -------
G_PROTECTED_VARIABLE_UNUSED  VARIABLE           DECLARATION 4        3
P_DATE                       FORMAL IN OUT      DECLARATION 8        5
V_UNUSED_VARIABLE            VARIABLE           DECLARATION 12       5
```

如果你移除外部查询中的条件（上面查询的第四行），即 `AND usage = 'DECLARATION'`，你可以添加额外的记录，表明那些被声明并赋值但从未被引用的变量（修改后的结果如下）。

```
NAME                         TYPE               USAGE       LINE     COL
---------------------------- ------------------ ----------- -------- --------
G_PROTECTED_VARIABLE_UNUSED  VARIABLE           ASSIGNMENT  4        3
G_PROTECTED_VARIABLE_UNUSED  VARIABLE           DECLARATION 4        3
P_DATE                       FORMAL IN OUT      DECLARATION 8        5
V_UNUSED_VARIABLE            VARIABLE           DECLARATION 12       5
```

## 查找作用域问题

另一个不良的编码实践是在程序的不同作用域中创建同名变量。如果你在引用时总是包含作用域，那么你可以确保访问的是正确的项。然而，这会导致痛苦的维护，通常是个坏主意。

为了举例说明，我使用了提供的 `SCOPE_PKG` 示例包。包规范和体如下，便于参考：

```sql
create or replace PACKAGE SCOPE_PKG AS
  i CONSTANT NUMBER := 0;

  g_global_variable NUMBER := 10;

  PROCEDURE proc1 ;
END SCOPE_PKG;
/

create or replace PACKAGE BODY SCOPE_PKG AS

  PROCEDURE proc1
  AS
    i PLS_INTEGER := 30;
  BEGIN
    FOR i IN 1..10
    LOOP
      DBMS_OUTPUT.PUT_LINE(i);
    END LOOP;

    DBMS_OUTPUT.PUT_LINE(i);
  END;

END SCOPE_PKG;
/
```

下一个查询显示在同一个程序（本例中是一个包）的不同作用域中声明的所有变量标识符（`VARIABLES`、`CONSTANTS` 和 `ITERATORS`）。包可以在规范（spec）和体（body）中声明变量。是否认为这种作用域划分是一个错误，由你决定。

```sql
WITH source_ids AS (
  SELECT    *
    FROM user_identifiers
      WHERE object_name = 'SCOPE_PKG'
        AND object_type IN ('PACKAGE BODY', 'PACKAGE')
)
SELECT name, type, usage, prior name, object_type, line, col
  FROM source_ids si
  WHERE type IN ('VARIABLE', 'ITERATOR', 'CONSTANT')
    AND usage = 'DECLARATION'
    AND EXISTS (
      SELECT count(*), name
        FROM user_identifiers
        WHERE object_Name = si.object_name
          AND name = si.name
          AND usage = si.usage
        GROUP BY name
        HAVING count(*) > 1 )
  START WITH usage_context_id = 0
  CONNECT BY PRIOR usage_id = usage_context_id
    AND PRIOR object_type = object_type;
```

查询结果如下：
```
NAME     TYPE        USAGE       PRIORNAME    OBJECT_TYPE   LINE     COL
-------- ----------- ----------- ------------ ------------- -------- ------
I        CONSTANT    DECLARATION SCOPE_PKG    PACKAGE       3        3
I        VARIABLE    DECLARATION PROC1        PACKAGE BODY  5        5
I        ITERATOR    DECLARATION PROC1        PACKAGE BODY  8        9
```

变量 `I` 在包规范中、体内的一个过程中、以及同一过程的循环迭代器中被声明。

你可以向查询中添加额外的条件，只返回那些明确冲突的值。你可以通过包含层级关系来实现——如果一个声明包含在一个 `USAGE_ID` 中，而这个 `USAGE_ID` 又被一个更高的 `USAGE_CONTEXT_ID` 所覆盖，例如当这个使用（usage）是前一个使用的子项时。


##### 数据类型检查

在 USER_ARGUMENTS 部分，我展示了如何验证数据类型，但也指出了其局限性——仅限于参数。通过 PL/Scope，我可以将验证扩展到程序中定义的任何变量。使用相同的过程示例 `BAD_DATA_TYPES_PRC`，以下查询展示了如何找到这些变量（和参数）：

```sql
SELECT * FROM (
WITH source_ids AS (
  SELECT    line,
            col,
            name,
            type,
            usage,
            usage_id,
            usage_context_id,
            object_name
    FROM user_identifiers
      WHERE object_name = 'BAD_DATA_TYPES_PRC'
        AND object_type = 'PROCEDURE')
SELECT  prior name "标识符",
        CASE
        WHEN prior type IN ('FORMAL IN', 'FORMAL OUT', 'FORMAL IN OUT')
        THEN '参数'
        WHEN prior type IN ('VARIABLE', 'CONSTANT', 'ITERATOR')
        THEN '变量'
        ELSE prior name
        END "标识符类型",
        Name "数据类型",
        Usage
  FROM source_ids
  START WITH usage_context_id = 0
  CONNECT BY PRIOR usage_id = usage_context_id
  ORDER SIBLINGS BY line, col
)
WHERE usage =  'REFERENCE';
```

查询结果如下：

```
标识符            标识符类型        数据类型        用法
--------------- -------------------- ------------ -----------
P_DATA_IN       参数                 CHAR           REFERENCE
P_DATA_OUT      参数                 LONG           REFERENCE
V_CHAR          变量                 CHAR           REFERENCE
V_VARCHAR       变量                 VARCHAR        REFERENCE
V_LONG          变量                 LONG           REFERENCE
```

目前仍然无法验证数据类型是否是锚定的。如果 PL/Scope 能将此作为标识符声明的子项提供，那就太好了。你可以推断（使用 `user_arguments`）数据长度或数据精度意味着参数是锚定的，但 `null` 并不意味着未锚定。例如，日期数据类型没有精度。

### 执行动态分析

从静态分析转向动态分析，让我们看看 Oracle 数据库（11g 之前版本）中可用的工具（`DBMS_PROFILER` 和 `DBMS_TRACE`），以及 11g 之后版本中可用的工具（上述工具加上 `DBMS_HPROF`）。尽管 `DBMS_HPROF` 是用于性能指标的更好的工具（在我看来好得多），但 `DBMS_PROFILER` 和 `DBMS_TRACE` 在代码覆盖测试方面仍然提供了一些可用性。

本节并非关于如何使用这些内置包的教程，但会涵盖一些基础知识以帮助你入门。本节将指出这些内置包中哪些部分对于了解你的代码是有用的。

使用这些 Oracle 提供的工具进行动态分析有两个主要好处。首先是洞察应用程序的性能特征。通过分析，你可以在代码投入生产前识别瓶颈。其次，算是一个无意的附带好处，就是能够测试代码覆盖率（稍后会详细介绍）。

请注意，第 13 章 专门从性能角度介绍了性能剖析和插桩，但没有涵盖本章讨论的 Oracle 提供的包。关于这些包的完整介绍，请参阅 Oracle 文档，其中概述了所有参数等信息。

#### DBMS_PROFILER 和 DBMS_TRACE

`DBMS_PROFILER` 是一个平面剖析器。它提供 PL/SQL 程序中两个过程调用之间的时间信息。你“打开”剖析器来启动计时器，然后在想停止计时时“关闭”它。“开启”和“关闭”之间的时间就是运行所需的时间（加上来自 `DBMS_PROFILER` 代码的一些最小开销）。

从性能角度看，`DBMS_PROFILER` 与 `DBMS_TRACE` 结合使用时最为强大。`DBMS_TRACE` 包为 `DBMS_PROFILER` 收集的计时提供了上下文。它提供了 PL/Scope 提供的静态层次结构的动态版本。PL/Scope 按可能发生的顺序向你显示标识符，并由此推断代码。PL/Scope 向你展示了每一种可能性；`DBMS_TRACE` 向你展示的是*实际*执行的层次结构。

`DBMS_TRACE` 要求所有要追踪的代码都必须使用 `DEBUG=TRUE` 编译。不建议在生产环境中运行为调试而编译的过程，因此 `DBMS_TRACE` 包作为插桩工具是无用的。然而，作为一种了解代码的方式，该包仍然有价值。

`DBMS_TRACE` 会话的输出显示哪个过程调用了哪个过程，按顺序排列。`DBMS_TRACE` 的输出与 `DBMS_PROFILER` 的逐行剖析相结合，是我找到的进行半自动化代码覆盖的最简单方法。即使在 11g 之后的世界里，`DBMS_TRACE` 和 `DBMS_PROFILER` 仍然具有一些无法以其他方式获得的价值。`DBMS_TRACE` 会为执行的每一行创建一条记录。当与 PL/Scope（仅限 11g+）结合使用时，你就得到了一个用于面向行的代码覆盖的强大工具。

##### 配置 DBMS_PROFILER 和 DBMS_TRACE

设置这两个工具是一个相当简单的过程。你需要对 `DBMS_PROFILER` 和 `DBMS_TRACE` 都拥有 `EXECUTE` 权限。这些包应该已经存在于数据库中。

在使用这些工具之前，你（或 DBA）需要运行几个脚本来创建底层表。这些脚本位于 `$ORACLE_HOME/RDBMS/ADMIN` 目录中。`PROFTAB.sql` 将为 `DBMS_PROFILER` 创建所需的表，`TRACETAB.sql` 为 `DBMS_TRACE` 创建表。注意，`TRACETAB.sql` 必须以 `SYS` 身份运行。这些脚本中创建的表将在后面描述。

如果你在单个服务器上运行多个应用程序，你可能想创建一个通用的 `PROFILER` 用户，并以该用户身份运行 `PROFTAB.sql`。这样，你可以授予对表的访问权限并为它们创建同义词，所有应用程序都可以共享相同的表。当你运行关于程序调用之间关系的报告时，可以跨越模式。如果你想保持每次运行分离，或者你只有一个应用程序模式，你可以在脚本所在的模式下运行它。

运行 `DBMS_PROFILER` 和 `DBMS_TRACE` 时，你必须具有特定的权限和编译器选项设置。对于任何要追踪或剖析的代码，你需要对被剖析的模式授予 `EXECUTE` 权限。此外，对于 `DBMS_PROFILER`，它需要对要剖析的任何程序单元拥有 `CREATE` 访问权限。最后，`DBMS_TRACE` 要求程序单元必须使用 `DEBUG=TRUE` 编译。

你需要对你连接的用户账户授予 `SELECT` 权限，以便查询 `DBMS_TRACE` 表。如果你连接的数据库用户是 `DBMS_PROFILER` 表的所有者，则不需要额外的授权。如果一个通用的剖析器用户拥有这些表，则该模式需要向你的模式授予 `SELECT` 权限。

配置就到此为止。完成这些设置步骤后，你就可以开始收集一些信息了。

