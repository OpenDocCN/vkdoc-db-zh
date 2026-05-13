# 工作原理

自动诊断知识库是一个目录结构，即使数据库关闭，您也可以访问它，因为它存储在数据库之外。ADR 的根目录称为 **ADR 基目录**，由 `DIAGNOSTIC_DEST` 初始化参数设置。每个 Oracle 产品或组件在 ADR 基目录下都有自己的 **ADR 主目录**。每个 ADR 主目录的位置遵循路径 `diag/产品类型/产品 ID/实例 ID`。因此，对于一个名为 `orcl1`、实例名称为 `orcl1` 且 ADR 基目录设置为 `/app/oracle` 的数据库，其 ADR 主目录将是 `/app/oracle/diag/rdbms/orcl1/orcl1`。每个 ADR 主目录下都包含该 Oracle 产品实例的诊断数据，例如跟踪和转储文件、警报日志以及其他诊断文件。

ADRCI 实用程序可帮助您管理 ADR 中的诊断数据。ADRCI 允许您执行以下类型的诊断任务：

*   查看 ADR（自动诊断知识库）中的诊断数据：ADR 存储警报日志、转储文件、跟踪文件、健康检查报告等诊断数据。
*   查看健康检查报告：诊断基础结构会自动运行健康检查以捕获有关错误的详细信息，并将其添加到它为该错误收集的其他诊断数据中。您也可以手动调用健康检查。
*   将事件和问题信息打包以便传输给 Oracle 支持人员：**问题** 是指关键数据库错误，**事件** 是指特定问题的单次发生。事件包是您发送给 Oracle 支持人员用于故障排除的诊断数据集合。ADRCI 提供特殊命令，使您能够创建包并生成压缩的诊断包以发送给 Oracle 支持人员。

您可以通过查询 `V$DIAG_INFO` 视图来查看当前数据库实例的所有 ADR 位置，如下所示。

```sql
SQL> select * from v$diag_info;

   INST_ID           NAME                   VALUE
-----------    ------------------     ----------------------------------------
1              Diag Enabled           TRUE
1              ADR Base               c:\app\ora
1              ADR Home               c:\app\ora\diag\rdbms\orcl1\orcl1
1              Diag Trace             c:\app\ora\diag\rdbms\orcl1\orcl1\trace
1              Diag Alert             c:\app\ora\diag\rdbms\orcl1\orcl1\alert
1              Diag Incident          c:\app\ora\diag\rdbms\orcl1\orcl1\incident
1              Diag Cdump             c:\app\ora\diag\rdbms\orcl1\orcl1\cdump
1              Health Monitor         c:\app\ora\diag\rdbms\orcl1\orcl1\hm
1              Default Trace File     c:\app\ora…\trace\orcl1_ora_6272.trc
1              Active Problem Count        2
1              Active Incident Count       3

11 rows selected.

SQL>
```

ADR 主目录是数据库诊断数据的根目录。所有诊断文件，如警报日志和各种跟踪文件，都位于 ADR 主目录下。ADR 主目录直接位于您使用 `DIAGNOSTIC_DEST` 初始化参数指定的 ADR 基目录之下。以下是如何找出 ADR 基目录位置的方法：

```
adrci> show base
ADR base is "c:\app\ora"
adrci>
```

您可以通过执行以下命令查看 ADR 基目录下的所有 ADR 主目录：

```
adrci> show homes
ADR Homes:
diag\clients\user_salapati\host_3975876188_76
diag\clients\user_system\host_3975876188_76
diag\rdbms\orcl1\orcl1
diag\tnslsnr\miropc61\listener
adrci>
```

您可以在 ADR 基目录下拥有多个 ADR 主目录，并且在任何给定时间可以有多个 ADR 主目录是当前的。您的 ADRCI 命令将仅在当前 ADR 主目录中的诊断数据上操作。如何知道在任何时间点哪个 ADR 主目录是当前的？ADRCI 主路径通过指向 ADR 基目录目录层次结构下的 ADR 主目录来帮助确定当前的 ADR 主目录。

![images](img/square.jpg) **注意** 某些 ADRCI 命令只需要一个 ADR 主目录是当前的——如果有多个 ADR 主目录是当前的，这些命令将报错。

您可以使用 `show homes` 或 `show homepath` 命令来查看所有当前的 ADR 主目录：

```
adrci> show homepath
ADR Homes:
diag\clients\user_salapati\host_3975876188_76
diag\clients\user_system\host_3975876188_76
diag\rdbms\orcl1\orcl1
diag\tnslsnr\miropc61\listener
adrci>
```

如果您想处理来自多个数据库实例或组件的诊断数据，必须确保所有相关的 ADR 主目录都是当前的。然而，大多数时候，您只会处理单个数据库实例或单个 Oracle 产品或组件，例如监听器。ADR 主路径总是相对于 ADR 的。例如，如果您将 `/u01/app/oracle/` 指定为 ADR 基目录的值，那么所有 ADR 主目录都将位于 `ADR_Base/diag` 目录下。使用 `set homepath` 命令将 ADR 主目录设置为单个主目录，如下所示：

```
adrci> set homepath diag\rdbms\orcl1\orcl1

adrci> show homepath

ADR Homes:
diag\rdbms\orcl1\orcl1
adrci>
```

![images](img/square.jpg) **注意** 诊断数据包括事件和问题的描述、健康监控报告以及传统的诊断文件，如跟踪文件、转储文件和警报日志。

请注意，在使用 `set homepath` 命令设置主路径之前，`show homepath` 命令会显示所有 ADR 主路径。但是，一旦您将主路径设置为特定主目录，`show homepath` 命令将仅显示该单个主路径。在执行多个 ADRCI 命令之前设置主路径非常重要，因为它们仅适用于单个 ADR 主目录。例如，如果在发出以下命令之前未设置主路径，您将收到错误：

```
adrci> ips create package
DIA-48448: This command does not support multiple ADR homes
adrci>
```

发生错误是因为 `ips create package` 命令在有多个 ADR 主目录时无效。在您使用 `set homepath` 命令将主路径设置为单个 ADR 主目录后，该命令将正常工作。像这里显示的命令这样的命令仅适用于单个当前 ADR 主目录，而其他命令则适用于多个当前 ADR 主目录——也有些命令不需要当前 ADR 主目录。最重要的是，所有 ADRCI 命令都将在单个当前 ADR 主目录上工作。

### 7-10. 从 ADRCI 查看警报日志

#### 问题

您想使用 ADRCI 命令查看警报日志。

#### 解决方案

要使用 ADRCI 查看警报日志，请按照以下步骤操作：

1.  调用 ADRCI。
    `$ adrci`
2.  使用 `set homepath` 命令设置 ADR 主目录。
    `adrci> set homepath diag\rdbms\orcl1\orcl1`
3.  输入以下命令查看警报日志：
    `adrci> show alert`

    ```
    ADR Home = c:\app\ora\diag\rdbms\orcl1\orcl1:
    *************************************************************************
    Output the results to file: c:\temp\alert_10916_7048_orcl1_1.ado
    ```

警报日志将在您的默认编辑器中弹出。一旦您在编辑器中关闭文本文件，ADRCI 提示符将返回。

您也可以查询 `V$DIAG_INFO` 视图以找到与 Diag Trace 条目对应的路径。您可以切换到该目录并使用文本编辑器打开 `alert_<数据库名称>.log` 文件。


### 工作原理

警报日志保存了 Oracle 实例的运行时信息，并提供诸如实例正在使用的初始化参数等信息，以及重做日志文件切换等关键变更的记录，最重要的是，还包含显示 Oracle 错误及其详细信息的消息。警报日志对于故障排查至关重要，通常是问题发生时你首先查看的地方。Oracle 以文本文件和 XML 格式文件两种形式提供警报日志。

`show alert` 命令会显示 XML 格式的警报日志，但不显示 XML 标签。你可以使用 `SET EDITOR` 命令设置默认编辑器，如下所示：

`adrci> set editor notepad.exe`

上一条命令将默认编辑器更改为记事本。`show alert -term` 命令在终端窗口中显示警报日志内容。如果你想只检查警报日志中最新的事件，请执行以下命令：

`adrci>show alert -tail 50`

`tail` 选项在命令窗口中显示警报日志中最 recent 的一组行。在此示例中，它显示警报日志的最后 50 行。如果你没有为 `tail` 参数指定值，默认情况下，它会显示警报日志的最后十行。

以下命令显示一个“实时”警报日志，即它会随着条目被添加到日志中而显示警报日志的变更。

`adrci> show alert -tail -f`

上一条命令显示警报日志的最后十行，并将所有新消息打印到屏幕上，从而提供警报日志持续添加的“实时”显示。按 CTRL+C 组合键将带你返回到 ADRCI 提示符。

在故障排查时，查看数据库是否发出任何 `ORA-600` 错误非常有用。你可以执行以下命令来捕获 `ORA-600` 错误。

`adrci> show alert -p "MESSAGE_TEXT LIKE '%ORA-600%'"`

尽管你可以通过直接访问存储警报日志的文件系统位置来查看它，但通过 ADRCI 工具这样做要容易得多。ADRCI 对于处理实例的跟踪文件尤其有用。`SHOW TRACEFILE` 命令显示实例跟踪目录中的所有跟踪文件。你可以使用各种过滤器来执行 `SHOW TRACEFILE` 命令——以下示例查找引用后台进程 `mmon` 的跟踪文件：

```
$ adrci> show tracefile %mmon%
     diag\rdbms\orcl1\orcl1\trace\orcl1_mmon_1792.trc
     diag\rdbms\orcl1\orcl1\trace\orcl1_mmon_2340.trc
adrci>
```

此命令列出文件名中包含字符串 `mmon` 的所有跟踪文件。你可以应用过滤器将输出限制为仅与特定事件号关联的跟踪文件，如下所示：

```
adrci> show tracefile -I 43417
          diag\rdbms\orcl1\orcl1\incident\incdir_43417\orcl1_ora_4276_i43417.trc
adrci>
```

上一条命令列出了与事件号 43417 相关的跟踪文件。

## 7-11. 使用 ADRCI 查看事件

### 问题

你想使用 ADRCI 查看事件。

### 解决方案

你可以使用 `show incident` 命令查看 ADR 中的所有事件（请务必先设置 homepath）：

```
$ adrci
$ set homepath diag\rdbms\orcl1\orcl1

adrci> show incident

ADR Home = c:\app\ora\diag\rdbms\orcl1\orcl1:
*******************************************************************************/
INCIDENT_ID          PROBLEM_KEY
 CREATE_TIME
-------------------- -----------------------------------------------------------
 43417                ORA 600 [kkqctinvvm(2): Inconsistent state space!]
 2010-12-17 09:26:15.091000 -05:00
43369                ORA 600 [kkqctinvvm(2): Inconsistent state space!]
 2010-12-17 11:08:40.589000 -05:00
79451                ORA 445
 2011-03-04 03:00:39.246000 -05:00
84243                ORA 445
 2011-03-14 19:12:27.434000 -04:00
84244                ORA 445
 2011-03-20 16:55:54.501000 -04:00
 5 rows fetched
```

你可以指定 `detail` 模式来查看特定事件的详细信息，如下所示：

```
adrci> show incident -mode detail -p "incident_id=43369"

ADR Home = c:\app\ora\diag\rdbms\orcl1\orcl1:
*******************************************************************************

INCIDENT INFO RECORD 1
*******************************************************************************/
   INCIDENT_ID                   43369
   STATUS                        ready
   CREATE_TIME                   2010-12-17 11:08:40.589000 -05:00
   PROBLEM_ID                    1
   CLOSE_TIME                    <NULL>
   FLOOD_CONTROLLED              none
   ERROR_FACILITY                ORA
   ERROR_NUMBER                  600
   ERROR_ARG1                    kkqctinvvm(2): Inconsistent state space!
   SIGNALLING_COMPONENT          SQL_Transform
   PROBLEM_KEY                   ORA 600 [kkqctin: Inconsistent state space!]
   FIRST_INCIDENT                43417
   FIRSTINC_TIME                 2010-12-17 09:26:15.091000 -05:00
   LAST_INCIDENT                 43369
   LASTINC_TIME                  2010-12-17 11:08:40.589000 -05:00
   KEY_VALUE                     ORACLE.EXE.3760_3548
   KEY_NAME                      PQ
   KEY_NAME                      SID
   KEY_VALUE                     71.304
   OWNER_ID                      1
   INCIDENT_FILE                 c:\app\ora\diag\rdbms\orcl1\orcl1\trace\orcl1_ora_3548.trc

adrci>
```

### 工作原理

`show incident` 命令报告数据库中所有未解决的事件。对于每个事件，此命令的输出显示问题键、事件 ID 以及事件发生的时间。在此示例中，我们首先设置了 ADRCI homepath，因此该命令只显示来自此 ADR 主目录的事件。如果你未设置 homepath，你将看到所有当前 ADR 主目录的事件。

如前所述，事件是问题的单次发生。问题是指关键错误，例如 `ORA-600`（内部错误）或与操作系统异常相关的 `ORA-07445` 错误。问题键是一个显示问题详细信息的文本字符串。例如，问题键 `ORA 600 [kkqctinvvm(2): Inconsistent state space!]` 表明问题是由于内部错误引起的。

当一个问题发生多次时，数据库会为问题的每次发生创建一个事件，每个事件都有一个唯一的事件 ID。数据库将事件记录在警报日志中，并向 Oracle Enterprise Manager 发送警报，警报会显示在主页上。数据库自动收集事件的诊断数据，称为事件转储，并将它们存储在 ADR 跟踪目录中。

由于一个关键错误可能产生大量相同的事件，因此故障诊断基础结构应用“洪泛控制”机制来限制事件的生成。对于给定的问题键，数据库在一小时内只允许 5 个事件，一天内最多允许 25 个事件。一旦问题触发的事件超过这些阈值，数据库仅在警报日志和 Oracle Enterprise Manager 中记录这些事件，但会停止为其生成新的事件转储。你无法更改事件洪泛控制机制的默认阈值设置。

## 7-12. 为 Oracle 支持打包事件

### 问题

你想将与特定问题相关的诊断文件发送给 Oracle 支持部门。


#### 解决方案

您可以通过 Database Control 或 ADRCI 界面执行的命令，来打包更多事件的诊断信息。在本解决方案中，我们将展示如何通过 ADRCI 打包事件。您可以使用各种 IPS 命令，以压缩格式打包与特定问题相关的所有诊断文件，并将文件发送给 Oracle 支持。以下是创建事件包的步骤。

1.  创建一个空的逻辑包，如下所示：

    ```
    adrci> ips create package
    Created package 1 without any contents, correlation level typical
    adrci>
    ```

    在此示例中，我们创建了一个空包，但您也可以基于事件编号、问题编号、问题键或时间间隔来创建包。在所有这些情况下，包都不会是空的——它将包含您指定的事件或问题的诊断信息。由于我们创建了一个空包，因此需要在下一步中向该包添加诊断信息。

2.  使用 `ips add incident` 命令向逻辑包添加诊断信息：

    ```
    adrci> ips add incident 43369 package 1
    Added incident 43369 to package 1
    adrci>
    ```

    此时，事件 43369 与包 1 相关联，但其中还没有诊断数据。

3.  生成物理包。

    ```
    adrci> ips generate package 1 in \app\ora\diagnostics
    Generated package 1 in file \app\ora\diagnostics\IPSPKG_20110419131046_COM_1.zip, mode complete
    adrci>
    ```

    当您发出 `generate package` 命令时，ADRCI 会收集所有相关的诊断文件，并将它们添加到您指定目录中的一个 zip 文件中。

4.  将生成的 zip 文件发送给 Oracle 支持。

    如果您决定向现有的物理包（zip 文件）添加补充诊断数据，可以通过在 `generate package` 命令中指定 `incremental` 选项来完成：

    ```
    adrci> ips generate package 1 in \app\ora\diagnostics incremental
    ```

    此命令创建的增量 zip 文件将在文件名中包含 `INC` 一词，表示它是一个增量 zip 文件。

#### 工作原理

物理包是您可以发送给 Oracle 支持以诊断数据库问题的 zip 文件。由于事件是问题的单次发生，因此将事件编号添加到逻辑包并生成物理包，会将该问题（事件）的所有诊断数据汇总到一个 zip 文件中。在此示例中，我们展示了如何首先创建一个空的逻辑包，然后将其与事件编号关联。然而，`ips create package` 命令有几个选项：您可以在创建逻辑包时直接指定事件编号或问题编号，从而跳过 `add incident` 命令。您还可以创建一个包含两个时间点之间所有事件的包，如下所示：

```
adrci> ips create package time '2011-04-12 10:00:00.00 -06:00' to '2011-04-12 23:00:00.00 -06:00'
Created package 2 based on time range 2011-04-12 12:00:00.000000 -06:00 to 2011-04-12 23:00:00.000000 -06:00, correlation level typical
adrci>
```

由先前命令生成的包包含 2011 年 4 月 12 日上午 10 点至晚上 11 点之间发生的所有事件。

请注意，您还可以手动将特定的诊断文件添加到现有包中。要添加文件，您需要在 `ips add file` 命令中指定文件名——您仅限于添加位于 ADR 基目录内的那些诊断文件。以下是一个示例：

```
adrci> ips add file <ADR_BASE>/diag/rdbms/orcl1/orcl1/trace/orcl_ora12345.trc package 1
```

默认情况下，`ips generate package` 命令生成一个包含包中所有文件的 zip 文件。`incremental` 选项将限制文件为数据库自您最初为该包生成 zip 文件以来所生成的文件。`ips show files` 命令显示包中的所有文件，`ips show incidents` 命令显示包中的所有事件。您可以发出 `ips remove file` 命令从包中移除诊断文件。

### 7-13. 运行数据库健康检查

#### 问题

您希望对数据库运行全面的诊断健康检查。您想了解是否存在任何数据字典或文件损坏，以及数据库中的任何其他潜在问题。

#### 解决方案

您可以使用数据库健康监控基础结构来运行数据库的健康检查。您可以运行各种完整性检查，例如事务完整性检查和字典完整性检查。您可以通过查询 `V$HM_CHECK` 视图来获取可以运行的所有健康检查的列表：

```
SQL> select name from v$hm_check where internal_check='N';
```

一旦决定了检查类型，请在 `DBMS_HM` 包的 `RUN_CHECK` 过程中指定检查的名称，如下所示：

```
SQL> begin
  2   dbms_hm.run_check('Dictionary Integrity Check','testrun1');
  3  end;
  4  /

PL/SQL procedure successfully completed.

SQL>
```

您也可以从 Enterprise Manager 运行健康检查。转到 Advisor Central -> Checkers，然后从 Checkers 子页面中选择特定的检查器来运行健康检查。

#### 工作原理

Oracle 在遇到关键错误时会自动运行健康检查。您可以使用“解决方案”部分中所示的过程手动运行检查。数据库将所有健康检查结果存储在 ADR 中。

您可以在数据库打开时运行大多数健康检查。只能在数据库关闭时运行重做完整性检查和数据库结构完整性检查——您必须将数据库置于 `NOMOUNT` 状态才能运行这两项检查。

您可以使用 `DBMS_HM` 包或通过 Enterprise Manager 查看健康检查的结果。以下是使用 `DBMS_HM` 包获取健康检查报告的方法：

```
SQL> set long 100000
SQL> set longchunksize 1000
SQL> set pagesize 1000
SQL> set linesize 512
SQL> select dbms_hm.get_run_report('testrun1') from dual;

DBMS_HM.GET_RUN_REPORT('TESTRUN1')
-----------------------------------------------------------------
Basic Run Information
 Run Name                     : testrun1
 Run Id                       : 61
 Check Name                   : Dictionary Integrity Check
 Mode                         : MANUAL
 Status                       : COMPLETED
 Start Time                   : 2011-04-19 15:46:50.313000 -04:00
 End Time                     : 2011-04-19 15:46:54.117000 -04:00
 Error Encountered            : 0
 Source Incident Id           : 0
 Number of Incidents Created  : 0

Input Paramters for the Run
 TABLE_NAME=ALL_CORE_TABLES
 CHECK_MASK=ALL
Run Findings And Recommendations

SQL>
```

在此示例中，幸运的是，没有发现任何问题，因此没有建议，因为字典健康检查没有发现任何问题。您也可以转到 Advisor Central -> Checkers，并为您运行过的任何健康检查从“运行详细信息”页面运行报告。使用 `show hm_run`、`create report` 和 `show report` 命令通过 ADRCI 实用程序查看健康检查报告。您可以使用视图 `V$HM_FINDING` 和 `V$HM_RECOMMENDATION` 来调查健康检查的发现和相应建议。

### 7-14. 创建 SQL 测试用例

#### 问题

您需要创建一个 SQL 测试用例，以便在不同的机器上重现 SQL 故障，无论是为了支持您自己的诊断工作，还是为了让 Oracle 支持能够重现该故障。



#### 解决方案

要创建一个 SQL 测试案例，首先必须导出 SQL 语句以及关于该语句的若干有用信息。以下示例展示了如何捕获抛出错误的 SQL 语句。在此示例中，用户 SH 执行导出操作（你无法以用户 SYS 的身份执行导出）。

首先，以 SYSDBA 身份连接到数据库并创建一个目录来保存测试案例：

```
SQL> conn / as sysdba
Connected.
SQL> create or replace directory TEST_DIR1 as 'c:\myora\diagnsotics\incidents\';

Directory created.
SQL> grant read,write on directory TEST_DIR1 to sh;

Grant succeeded.
SQL>
```

然后，将 DBA 角色授予将用于创建测试案例的用户，并以该用户身份连接：

```
SQL> grant dba to sh;

Grant succeeded.

SQL> conn sh/sh
Connected.
```

执行抛出错误的 SQL 命令：

```
SQL> select * from  my_mv  where max_amount_sold >100000 order by 1;
```

现在，你已准备好导出 SQL 语句和相关信息，以便日后可以将其导入到不同的系统中。使用 `EXPORT_SQL_TESTCASE` 过程来导出数据，如下所示：

```
SQL> set serveroutput on

SQL> declare mycase clob;
  2  begin
  3  dbms_sqldiag.export_sql_testcase
  4  (directory    =>'TEST_DIR1',
  5  sql_text      => 'select * from my_mv where max_amount_sold >100000 order by 1',
  6  user_name     => 'SH',
  7  exportData    =>  TRUE,
  8  testcase      => mycase
  9  );
 10  end;
 11  /

PL/SQL procedure successfully completed.

SQL>
```

一旦导出过程完成，你就可以执行导入操作了，无论是在同一台服务器还是不同的服务器上。以下示例创建一个名为 TEST 的新用户，并将测试案例导入到该用户的模式中。以下是将 SQL 语句和相关信息导入到不同模式的步骤。

```
SQL> conn /as sysdba
Connected.
SQL> create or replace directory TEST_DIR2 as 'c:\myora\diagnsotics\incidents\'; /

Directory created.
SQL> grant read,write on directory TEST_dir2 to test;
```

将 `TEST_DIR1` 目录中的所有文件传输到 `TEST_DIR2` 目录。然后，将 DBA 角色授予用户 TEST，并以该用户身份连接：

```
SQL> grant dba to test;

Grant succeeded.

SQL> conn test/test
Connected.
```

以用户 TEST 的身份执行 SQL 数据的导入，调用 `IMPORT_SQL_TESTCASE` 过程，如下所示：

```
SQL> begin
  2  dbms_sqldiag.import_sql_testcase
  3  (directory=>'TEST_DIR2',
  4  filename=>'oratcb1_008602000001main.xml',
  5  importData=>TRUE
  6  );
  7  end;
  8  /

PL/SQL procedure successfully completed.

SQL>
```

现在，用户 TEST 将拥有执行你想要调查的 SQL 语句所需的所有对象。你可以通过执行原始的 `select` 语句来验证这一点。它应该给出与在 SH 模式下相同的输出。

#### 工作原理

Oracle 提供了 SQL 测试案例构建器 (TCB) 来重现 SQL 故障。你可以通过企业管理器或通过 PL/SQL 包来创建测试案例。本教程的“解决方案”部分展示了如何使用 `DBMS_SQLDIAG` 包的 `EXPORT_SQL_TESTCASE` 过程创建测试案例。该包有多个变体，我们的示例展示了如何使用 SQL 语句作为源来创建 SQL 测试案例。有关创建测试案例的其他选项，请查阅 Oracle PL/SQL Packages 手册中的 `DBMS_SQLDIAG.EXPORT_TESTCASE` 过程的详细信息。

![images](img/square.jpg) **注意** 记住，你应该以任何已被授予 DBA 角色的用户（SYS 除外）的身份运行测试案例构建器。

通常，你会发现自己需要为 Oracle 提供一个测试案例，否则 Oracle Support 人员将无法调查他们正在帮助你解决的特定问题。SQL 测试案例构建器是 Oracle Database 11g 的一部分，其主要目的是帮助你快速获得一个可复现的测试案例。SQL 测试案例构建器帮助你轻松捕获与失败 SQL 语句相关的关键信息，并将其打包成一种格式，供开发人员或 Oracle 支持人员在另一个环境中重现问题。

你可以通过 `DBMS_SQLDIAG` 包访问 SQL 测试案例构建器。要创建测试案例，必须首先导出 SQL 语句，包括该语句涉及的所有对象以及其他相关信息。导出过程与使用 `EXPDP` 命令的 Oracle 导出非常相似，因此也像 `EXPDP` 一样使用一个目录。Oracle 将 SQL 测试案例创建为一个脚本，其中包含将重新创建必要数据库对象的语句，以及相关的运行时信息（如统计信息），这些信息使你能够重现错误。以下是作为测试案例创建过程一部分捕获和导出的各种类型的信息：

*   问题语句的 SQL 文本
*   表数据 —— 这是可选的，你可以导出样本数据或完整数据。
*   执行计划
*   优化器统计信息
*   PL/SQL 函数、过程和包
*   绑定变量
*   用户权限
*   SQL 配置文件
*   作为 SQL 语句一部分的所有对象的元数据
*   动态采样结果
*   运行时信息，例如并行度

在 `DBMS_SQLDIAG` 包中，`EXPORT_SQL_TESTCASE` 过程将 SQL 语句的 SQL 测试案例导出到一个目录。`IMPORT_SQL_TESTCASE` 过程则从一个目录导入测试案例。

在 `EXPORT_SQL_TESTCASE` 过程中，各属性含义如下：

> `DIRECTORY`：你想要存储测试案例文件的目录
> 
> `SQL_TEXT`：实际抛出错误的 SQL 语句
> 
> `TESTCASE`：测试案例的名称
> 
> `EXPORTDATA`：默认情况下，Oracle 不导出数据。你可以将此参数设置为 `TRUE` 以导出数据。你可以通过为 Sampling Percent 属性指定一个值来选择性地限制要导出的数据量。默认值为 100。

测试案例构建器会自动导出 PL/SQL 包规范，但不导出包体。但是，你也可以指定 TCB 同时导出包体。导出过程在你指定的目录中创建多个文件。其中，格式为 `oratcb1_008602000001main.xml` 的文件包含测试案例的元数据。

### 7-15. 生成 AWR 报告

#### 问题

你想要生成一份 AWR 报告来分析数据库中的性能问题。


#### 解决方案

数据库会自动每小时执行一次 AWR 快照，并将统计信息保存在 AWR 中八天。一份 AWR 报告包含两个快照之间捕获的数据，这两个快照不必是连续的。因此，AWR 报告可以让您检查两个时间点之间的实例性能。您可以通过**Oracle Enterprise Manager**生成 AWR 报告。不过，我们会向您展示如何使用 Oracle 提供的脚本创建 AWR 报告。

要为单实例数据库生成 AWR 报告，请执行`awrrpt.sql`脚本，如下所示。

```
SQL> @?/rdbms/admin/awrrpt.sql

Current Instance
~~~~~~~~~~~~~~~~
   DB Id    DB Name      Inst Num Instance
----------- ------------ -------- ------------
1118243965 ORCL1               1 orcl1
Specify the Report Type
~~~~~~~~~~~~~~~~~~~~~~~
Would you like an HTML report, or a plain text report?
Enter 'html' for an HTML report, or 'text' for plain text
Defaults to 'html'
Enter value for report_type: text

Type Specified:  
```

选择基于文本或 HTML 的报告。HTML 报告是默认报告类型，它提供外观美观、格式良好、易于阅读的报告。按 Enter 键选择默认的 HTML 类型报告。

```
Instances in this Workload Repository schema
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

   DB Id     Inst Num DB Name      Instance     Host
------------ -------- ------------ ------------ ------------
* 1118243965        1 ORCL1        orcl1        MIROPC61

Using 1118243965 for database Id
Using          1 for instance number
```

此时您必须指定数据库的 DBID。然而，在我们的示例中，只有一个数据库，因此也只有一个 DBID，所以无需输入 DBID。

```
Specify the number of days of snapshots to choose from
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Entering the number of days (n) will result in the most recent
(n) days of snapshots being listed.  Pressing <return> without
specifying a number lists all completed snapshots.
Enter value for num_days: 1
```

输入您希望数据库列出快照 ID 的天数。在此示例中，我们选择 1，因为我们想生成一个发生在最近一天的时间段内的 AWR 报告。

```
Listing the last day's Completed Snapshots
                                                Snap
Instance     DB Name        Snap Id    Snap Started    Level
------------ ------------ --------- ------------------ -----
orcl1        ORCL1             1877 17 Apr 2011 00:00      1
                               1878 17 Apr 2011 07:47      1
```

为 AWR 报告指定一个开始和结束的快照。

```
Specify the Begin and End Snapshot Ids
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Enter value for begin_snap: 1877
Begin Snapshot Id specified: 1877

Enter value for end_snap: 1878
End Snapshot Id specified: 1878
```

您可以通过按 Enter 键接受 AWR 报告的默认名称，或者为报告输入一个名称。

```
Specify the Report Name
~~~~~~~~~~~~~~~~~~~~~~~
The default report file name is awrrpt_1_1877_1878.txt.  To use this name,
press <return> to continue, otherwise enter an alternative.

Enter value for report_name:
Using the report name awrrpt_1_1877_1878.html
```

数据库会在您调用`awrrpt.sql`脚本的同一目录下生成 AWR 报告。例如，如果您选择基于 HTML 的报告，AWR 报告将采用以下格式：`awrrpt_1_1881_1882.html`。

![images](img/square.jpg) 提示：您可以通过执行`awrsqrpt.sql`脚本来生成 AWR 报告，以分析单个 SQL 语句的性能。

#### 工作原理

您生成的 AWR 报告会显示两个时间点之间捕获的性能统计信息，每个时间点称为一个快照。通过仔细阅读 AWR 报告，您可以深入了解数据库性能。AWR 报告在大多数情况下运行时间不到一分钟，并且包含大量性能信息。该报告由多个部分组成。浏览 AWR 报告通常可以找出数据库未达到最佳性能水平的原因。

要在 Oracle RAC 环境中为所有实例生成 AWR 报告，请改用`awrgrpt.sql`脚本。您可以通过使用`awrrpti.sql`脚本为 RAC 环境中的特定实例生成 AWR 报告。您还可以通过调用`awrsqrpt.sql`脚本并提供 SQL 语句的`SQL_ID`，为单个 SQL 语句生成 AWR 报告。

只要 AWR 拥有覆盖该时间段的快照，您就可以生成跨越任意时长的 AWR 报告。默认情况下，AWR 将其快照保留八天。

您可以随时生成 AWR 快照，无论是通过**Oracle Enterprise Manager**还是使用`DBMS_WORKLOAD_REPOSITORY`包。当您想调查 20 分钟前发生的问题，而下一个快照要在 40 分钟后才产生时，这个能力就很有用了。在这种情况下，您必须等待 40 分钟或创建手动快照。以下示例展示了如何手动创建 AWR 快照：

```
SQL> exec dbms_workload_repository.create_snapshot();

PL/SQL procedure successfully completed.
SQL>
```

创建快照后，您可以运行`awrrpt.sql`脚本。然后选择之前的两个快照来生成最新的 AWR 报告。

当用户响应时间突然增加时，例如在高峰时段从一秒增加到十秒，您可以生成 AWR 报告。如果关键批处理作业突然需要更长时间才能完成，AWR 报告也可能有所帮助。当然，您还必须借助操作系统工具（如`sar`、`vmstat`和`iosat`）检查该时间段内的系统 CPU、I/O 和内存使用情况。

如果系统 CPU 使用率很高，这并不一定意味着 CPU 是罪魁祸首。CPU 使用率百分比并不总是吞吐量的真实度量，CPU 使用率也并不总是作为数据库工作负载指标有用。请确保为观察到性能恶化的精确时间段生成 AWR 报告。跨度为 24 小时的 AWR 报告对于诊断两小时前仅持续了 30 分钟的性能下降几乎没有用处。让您的报告时间范围与性能问题的发生时间段相匹配。

### 7-16. 比较两个时间段的数据库性能

#### 问题

您想要检查并比较数据库在两个不同时段内的性能表现。

## 解决方案

使用`awrddrpt.sql`脚本（位于`$ORACLE_HOME/rdbms/admin`目录）生成一个 AWR 比较时段报告，以比较两个时段之间的性能。步骤如下。

1.  调用`awrddrpt.sql`脚本。`SQL> @$ORACLE_HOME/rdbms/admin/awrddrpt.sql`
2.  选择报告类型（默认为文本）。`输入报告类型的值: html`
    `指定的类型:   html`
3.  指定要从中选择第一个时段起始和结束快照的快照天数。`指定要选择的快照天数`
    `输入 num_days 的值: 4`
4.  选择要分析第一个时段性能的一对快照。`指定第一对起始和结束快照 ID`
    `~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~`
    `输入 begin_snap 的值: 2092`
    `指定的第一开始快照 ID: 2092`

    `输入 end_snap 的值: 2093`
    `指定的第一结束快照 ID: 2093`
5.  选择要从中选择第二个时段快照对的天数。输入值 4，以便从过去 4 天中选择一对快照。`指定要选择的快照天数`
    `输入 num_days 的值: 4`
6.  指定第二个时段的起始和结束快照。`指定第二对起始和结束快照 ID`
    `~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~`
    `输入 begin_snap2 的值: 2134`
    `指定的第二开始快照 ID: 2134`

    `输入 end_snap2 的值: 2135`
    `指定的第二结束快照 ID: 2135`
7.  指定报告名称或接受默认名称。`指定报告名称`
    `~~~~~~~~~~~~~~~~~~~~~~~`
    `默认报告文件名为 awrdiff_1_2092_1_2134.html。要使用此名称，`
    `按 <回车> 键继续，否则请输入替代名称。`

    `输入 report_name 的值:`
    `使用报告名称 awrdiff_1_2092_1_2134.html`
    `报告已写入 awrdiff_1_2092_1_2134.html`
    `SQL>`

## 工作原理

生成 AWR 比较时段报告的过程与生成普通 AWR 报告的过程非常相似。最大的区别在于，该报告不显示两个快照之间发生的情况，而普通 AWR 报告会显示。AWR 比较时段报告比较两个不同时间段的性能，每个时间段涉及不同的一对快照。如果你想比较数据库今天上午 9 点到 10 点与三天前同一时间段的性能，可以使用 AWR 比较时段报告来完成。你可以在一个或所有 RAC 数据库实例上运行 AWR 比较时段报告。

AWR 比较时段报告的组织结构与普通 AWR 报告类似，但它并排显示两个时段的每个性能统计信息，因此你可以快速看到性能上的差异（或相似之处）。以下是报告中的一个部分，展示了如何轻松查看第一个和第二个时段之间性能统计信息的差异。

```
                                     第一时段            第二时段        差异
                              ---------------       ---------------      ------
 每次读取变更的块百分比:                 61.48                 15.44      -46.04
           递归调用百分比:                 98.03                 97.44       -0.59
每事务回滚百分比:                  0.00                  0.00        0.00
              每排序行数:                  2.51                  2.07       -0.44
每次调用平均数据库时间（秒）:                  1.01                  0.03       -0.98
```

### 7-17. 分析 AWR 报告

### 问题

你已经生成了一份覆盖数据库出现性能问题时段的 AWR 报告。你想要分析这份报告。

### 解决方案

AWR 报告在其各个部分下总结了与性能相关的统计信息。以下是对 AWR 报告中最重要的部分的简要总结。

#### 会话信息

你可以从 AWR 报告最顶部的部分了解会话数量，如下所示：

```
快照 ID       快照时间       会话数 游标/会话
            --------- ------------------- -------- ---------
开始快照:      1878 11-4 月-11 07:47:33        38       1.7
  结束快照:      1879 11-4 月-11 09:00:48        34       3.7
   已用时间:               73.25（分钟）
   数据库时间:               33.87（分钟）
```

务必检查“开始快照”和“结束快照”时间，以确认该时段涵盖了性能问题发生的时间。如果你注意到会话数量非常多，可以调查是否正在创建影子进程——例如，如果在你预期两个时间点的会话数应该相同的情况下，会话数在“开始快照”和“结束快照”时间之间增加了 200，最可能的原因是应用程序启动问题，因为它正在生成所有这些会话。

#### 负载概况

“负载概况”部分显示了各项数据库负载指标（如硬解析和事务数）的每秒和每事务统计信息。

```
负载概况              每秒    每事务
~~~~~~~~~~~~          ---------------    ---------------
      数据库时间(秒):                0.5                1.4
       数据库 CPU(秒):                0.1                0.3
       重做量:            6,165.3           19,028.7
   逻辑读:              876.6            2,705.6
   块变更:               99.2              306.0
  物理读:               10.3               31.8
  物理写:               1.9                5.9
      用户调用:                3.4               10.4
          解析:               10.2               31.5
      硬解析:                1.0                3.0
          登录:                0.1                0.2
    事务:                0.3
```

“负载概况”部分是 AWR 报告中最重要的部分之一。物理 I/O 速率和硬解析尤为重要。在高效运行的数据库中，你应该看到主要是软解析，硬解析非常少。高硬解析率通常是未使用绑定变量的结果。如果你看到每秒登录数很高，通常意味着你的应用程序未使用持久连接。登录数过多或异常高的事务数告诉你数据库中正在发生一些异常情况。然而，只有当你定期检查 AWR 报告并了解各种统计信息在数据库正常运行时、一天中不同时间的表现，你才知道这些数字是否异常！

#### 实例效率百分比

实例效率部分显示了几个命中率以及“执行到解析”和“闩锁命中”百分比。

```
实例效率百分比（目标 100%）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
            缓冲区无等待 %:  100.00       重做无等待 %:  100.00
            缓冲区命中 %:   99.10    内存中排序 %:  100.00
            库缓存命中 %:   95.13        软解析 %:   90.35
          执行到解析比 %:   70.71         闩锁命中 %:   99.97
  解析 CPU 与解析耗时比 %: 36.71     % 非解析 CPU:   83.60

共享池统计信息        开始    结束
                              ------  ------
             内存使用率 %:   81.08   88.82
    % 执行次数>1 的 SQL:   70.41   86.92
  % 执行次数>1 的 SQL 内存占用:   69.60   91.98
```

在运行良好的实例中，`执行到解析比`应该非常高。`% 执行次数>1 的 SQL`统计值低意味着数据库未重用共享 SQL 语句，通常是因为 SQL 未使用绑定变量。

