# SQL 监控

还有一个工具对于确定 SQL 语句是否被卸载非常有用，这对于所有性能调查来说都相当酷。`REPORT_SQL_MONITOR` 过程是 11g 版新增的实时 SQL 监控功能的一部分。它内置于 `DBMS_SQLTUNE` 包中，并且（如果你有相应的许可）能提供大量信息。它不仅提供语句是否被卸载的信息，还能显示执行计划中的哪些步骤被卸载。以下是一个已卸载语句的示例。遗憾的是，输出内容太宽了——虽然已经稍微压缩了一点，但仍然包含了必要信息：

```
SQL> select /*+ gather_plan_statistics monitor sqlmonexample001 */
2  count(*) from bigtab where id between 1000 and 50000;

COUNT(*)
----------
1568032

Elapsed: 00:00:00.66

SQL> @report_sql_monitor
Enter value for sid:
Enter value for sql_id:
Enter value for sql_exec_id:

REPORT
------------------------------------------------------------------------
SQL Monitoring Report

SQL Text
------------------------------------------------------------------------
select /*+ gather_plan_statistics monitor sqlmonexample002 */ count(*) from bigtab where id between 1000 and 50000

Global Information
------------------------------
Status              :  DONE (ALL ROWS)
Instance ID         :  1
Session             :  MARTIN (1108:55150)
SQL ID              :  0kytf1zmdt5f1
SQL Execution ID    :  16777216
Execution Started   :  01/22/2015 05:59:26
First Refresh Time  :  01/22/2015 05:59:26
Last Refresh Time   :  01/22/2015 05:59:36
Duration            :  10s
Module/Action       :  SQL*Plus/-
Service             :  SYS$USERS
Program             :  sqlplus@enkdb03.enkitec.com (TNS V1-V3)
Fetch Calls         :  1

Global Stats
=======================================================================================================================
| Elapsed |   Cpu   |    IO    | Application | Fetch | Buffer | Read  | Read  |  Cell   |
| Time(s) | Time(s) | Waits(s) |  Waits(s)   | Calls |  Gets  | Reqs  | Bytes | Offload |
=======================================================================================================================
|      11 |    4.03 |     7.25 |        0.00 |     1 |    10M | 80083 |  78GB |  99.96% |
=======================================================================================================================

SQL Plan Monitoring Details (Plan Hash Value=2140185107)
================================================================================================================================
| Id |      Operation             |  Name  | Cost |   Time    | Activity | Activity Detail |
|    |                            |        |      | Active(s) |   (%)    |     (# samples) |
================================================================================================================================
|  0 |SELECT STATEMENT            |        |      |         9 |          |                 |
|  1 | SORT AGGREGATE             |        |      |         9 |          |                 |
|  2 |  TABLE ACCESS STORAGE FULL | BIGTAB |   3M |         9 |   100.00 | Cpu (3)         |
|    |                            |        |      |           |          | cell smart table|
|    |                            |        |      |           |          | scan (6)        |
================================================================================================================================
```

你可以看到，在全局部分，报告显示了整个语句的存储单元卸载百分比。在详细信息部分，它还基于 ASH 样本显示了哪些步骤被卸载以及它们做了什么（活动详情）。它也显示了语句的时间花费在哪里（活动百分比）。对于那些有多个步骤适合卸载的更复杂语句来说，这极其有用。并行执行的语句会为每个查询服务器进程列出这些信息，这就引出了下一个值得提及的要点：对于更复杂的语句，SQL 监控报告的文本版本可能变得难以阅读。你能获得的最有用的输出格式是向 `REPORT_LEVEL` 传递 `ALL`，并为 `TYPE` 参数传递 `ACTIVE`。生成的输出是一个 HTML 文件，你可以在浏览器中打开并查看。Oracle 企业管理器也提供图形界面来访问 SQL 监控输出。你可以在第 12 章中了解更多关于 SQL 监控各个方面的信息。

请注意，监控会自动发生在并行化的语句以及优化器预期会运行很长时间的语句上。如果 Oracle 没有自动选择监控你感兴趣的语句，你可以使用 `MONITOR` 提示来告诉 Oracle 监控该语句，如示例所示。你可以检查 `V$SQL_MONITOR` 来查看是否能根据你的 `SQL_ID` 创建报告。

### 参数

有几个参数适用于卸载。主要的是 `CELL_OFFLOAD_PROCESSING`，它控制卸载的开关。还有其他几个重要性稍低的参数。表 2-2 列出了影响卸载的非隐藏参数列表（截至 Oracle 数据库版本 12.1.0.2）。请注意，我们还包含了控制这个非常重要的功能的隐藏参数 `_SERIAL_DIRECT_READ`。

表 2-2.
控制卸载的重要数据库参数

| 参数 | 默认值 | 描述 |
| --- | --- | --- |
| `cell_offload_decryption` | `TRUE` | 控制是否卸载解密。请注意，当此参数设置为 `FALSE` 时，在加密数据上将完全禁用智能扫描。 |
| `cell_offload_plan_display` | `AUTO` | 控制在 `DBMS_XPLAN.DISPLAY%` 函数的执行计划输出中是否使用 Exadata 操作名称。 |
| `cell_offload_processing` | `TRUE` | 打开或关闭卸载。 |
| `_serial_direct_read` | `AUTO` | 控制串行直接路径读取机制。有效值为 `ALWAYS`、`AUTO`、`TRUE`、`FALSE` 和 `NEVER`。 |

除了正常的 Oracle 批准的参数外，还有一些所谓的隐藏参数影响卸载的各个方面。你可以使用在线代码仓库中提供的 `parms.sql` 脚本来查看它们，以 SYSDBA 身份连接并同时指定 `kcfis`（用于内核文件智能存储）和 `cell`（用于所有 cellsrv 相关的参数）。和以往一样，请注意，在未经 Oracle 支持事先讨论和同意的情况下，不应在 Oracle 系统中使用隐藏参数，但它们确实为一些 Exadata 功能的工作方式和控制方式提供了有价值的线索。

