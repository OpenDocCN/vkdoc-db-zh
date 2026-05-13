# 附录 C

![image](img/frontdot.jpg)

## 工具配置参数

本附录包含所有工具配置参数及其描述。

| 参数 | 描述 |
| --- | --- |
| `automatic_workload_repository` | 访问自动工作量存储库 (AWR) 需要 Oracle Diagnostic Pack 的许可。如果您没有此许可，可以将此参数设置为 `N`。 |
| `bde_chk_cbo` | 在 EBS 应用程序中，SQLT 会自动执行注释 174605.1 中的 `bde_chk_cbo.sql`。 |
| `c_cbo_stats_vers_days` | 要收集的 CBO 统计信息版本的天数。如果设置为 `0`，则不收集任何统计信息版本。如果设置的值大于实际存储的天数，则 SQLT 会收集全部历史记录。值为 7 表示收集与给定 SQL 相关的模式对象的过去 7 天的 CBO 统计信息版本。这包括表、索引、分区、列和直方图。 |
| `c_dba_hist_parameter` | 从 `DBA_HIST_PARAMETER` 中收集相关条目。如果 `automatic_workload_repository` 和 `c_dba_hist_parameter` 都设置为 `Y`，那么 SQLT 会从视图 `DBA_HIST_PARAMETER` 中收集相关行。 |
| `c_gran_cols` | 列的收集粒度。默认值 `SUBPARTITION` 允许 SQLT 将列的 CBO 统计信息（所有级别：表、分区和子分区）收集到其存储库中。这些都与正在分析的一个 SQL 相关。 |
| `c_gran_hgrm` | 直方图的收集粒度。默认值 `SUBPARTITION` 允许 SQLT 将直方图的 CBO 统计信息（所有级别：表、分区和子分区）收集到其存储库中。这些都与正在分析的一个 SQL 相关。 |
| `c_gran_segm` | 段（表和索引）的收集粒度。默认值 `SUBPARTITION` 允许 SQLT 将表、索引、分区和子分区的 CBO 统计信息收集到其存储库中。这些都与正在分析的一个 SQL 相关。 |
| `collect_perf_stats` | 在 XECUTE 方法上收集性能统计信息。 |
| `connect_identifier` | 可选的连接标识符（根据 Oracle Net）。这在导出 SQLT 存储库时使用。包含“@”符号，例如 `@PROD`。您也可以将此参数设置为 `NULL`。 |
| `count_star_threshold` | 在 SQL 所访问的一组表上执行 `SELECT COUNT(*)` 时，限制要计数的行数。如果要禁用此功能，请将此参数设置为 `0`。 |
| `custom_sql_profile` | 控制是否在每次执行 SQLT 主方法时都生成带有自定义 SQL Profile 的脚本。 |
| `distributed_queries` | SQLT 可以使用被分析 SQL 所引用的数据库链接。它连接到那些远程系统以获取被分发 SQL 的 10053 和 10046 跟踪。 |
| `domain_index_metadata` | 此参数控制域索引元数据是否包含在主报告和元数据脚本中。如果您遇到 ORA-07445 错误，并且 `alert.log` 显示错误是由 `CTXSYS.CTX_REPORT.CREATE_INDEX_SCRIPT` 引起的，那么您需要将此参数设置为 `N`。 |
| `event_10046_level` | SQLT XECUTE 默认开启事件 10046 级别 12。您可以使用此参数设置不同的级别或关闭此事件 10046。它只影响传递给 SQLT XECUTE 的脚本的执行。级别 0 表示无跟踪，级别 1 是标准 SQL 跟踪，级别 8 包含等待，级别 12 同时包含绑定变量和等待。 |
| `event_10053_level` | SQLT XECUTE、XTRACT 和 XPLAIN 默认开启事件 10053 级别 1。您可以使用此参数关闭此事件 10053。它只影响传递给 SQLT 的 SQL。级别 0 表示无跟踪，级别 1 跟踪 CBO。 |
| `event_10507_level` | SQLT XECUTE 在 11g 上使用此事件来跟踪基数反馈 (CFB)。您可以使用此参数关闭此事件 10507。它只影响传递给 SQLT 的 SQL。级别 0 表示无跟踪，其他级别的含义请参阅 MOS 文档 ID 740052.1。 |
| `event_others` | 此参数控制事件 10241、10032、10033、10104、10730、46049 的使用，但前提是 10046 已开启（除 0 外的任何级别）。它只影响传递给 SQLT XECUTE 的脚本的执行。 |



### SQLT 参数参考

以下参数用于配置和控制 SQLT 工具的行为。

#### 导出控制

`export_repository`
方法 `XTRACT`, `XECUTE` 和 `XPLAIN` 会自动导出 SQLT 存储库中的相应条目。此参数控制此自动导出功能。

`export_utility`
SQLT 存储库可以使用两种可用工具之一进行自动导出：传统的 `exp` 或数据泵 `expdp`。使用此参数可以指定 SQLT 应使用哪一个工具。

`keep_trace_10046_open`
如果您需要跟踪 SQLT `XECUTE`, `XTRACT` 或 `XPLAIN` 的执行，此参数允许您在自定义 `SCRIPT` 完成后保持 `10046` 跟踪处于活动状态。当设置为其默认值 `N` 时，事件 `10046` 会在自定义脚本执行后或 `10053` 被关闭后立即关闭。

`mask_for_values`
表列的端点值是 CBO 统计信息的一部分。如果出于隐私原因需要从 SQLT 报告中删除这些端点值，可以将此参数设置为 `SECURE` 或 `COMPLETE`。`SECURE` 仅显示日期的年份，以及字符串和数字的一个字符。`COMPLETE` 完全阻止端点值的显示，并且还会禁用 SQLT 存储库的自动导出。默认值为 `CLEAR`，显示端点值。如果考虑更改为非默认值，请记住，选择性和基数验证需要对这些列端点的值有所了解。还要注意，`10053` 跟踪也包含一些不受此参数影响的高低值。

`refresh_directories`
控制每次执行 SQLT 时是否应检查并刷新用于 UDUMP/BDUMP 的 SQLT 和 TRCA 目录。

`sqlt_max_file_size_mb`
单个 SQLT 文件的最大大小（以 MB 为单位）。

`upload_trace_size_mb`
SQLT 将其存储库上传事件 `10046` 和 `10053` 生成的跟踪文件。此参数控制每个跟踪文件上传的最大兆字节数。

`xecute_script_output`
SQLT `XECUTE` 生成一个假脱机文件，其中包含被分析 SQL 的输出（在输入脚本中传递）。此文件可以保留在本地目录中，包含在 zip 文件中，或直接删除。

#### 健康检查

`healthcheck_blevel`
计算索引/分区/子分区的 `blevel`，并检查它们在一次统计信息收集到下一次之间变化是否超过 10%。

`healthcheck_endpoints`
计算直方图端点计数，并检查它们在一次统计信息收集到下一次之间变化是否超过 10%。

`healthcheck_ndv`
检查列的不同值数量在一次统计信息收集到下一次之间变化是否超过 10%。

`healthcheck_num_rows`
检查表/分区/子分区的行数，并检查它们在一次统计信息收集到下一次之间变化是否超过 10%。

#### 报告与统计信息

`generate_10053_xtract`
在 `XTRACT` 中使用 `DBMS_SQLDIAG.DUMP_TRACE` 生成 `10053` 可以作为一种解决断开连接 `ORA-07445` 在 `SYS.DBMS_SQLTUNE_INTERNAL` 中的问题的变通方法。SQLT 检测到 `ORA-07445` 并在下次执行时禁用对 `DBMS_SQLDIAG.DUMP_TRACE`（以及 `SYS.DBMS_SQLTUNE_INTERNAL`）的调用。如果此参数的值为 `E` 或 `N`，那么您的系统中可能存在一个低影响的错误。

`plan_stats`
来自 `GV$SQL_PLAN` 的执行计划可能包含游标最后一次执行以及所有执行的统计信息（如果参数 `statistics_level` 在游标被硬解析时设置为 `ALL`）。此参数控制这两种统计信息（最后一次执行以及所有执行）的显示。

`predicates_in_plan`
计划中的谓词可以作为解决错误 6356566 的变通方法被消除。SQLT 检测到 `ORA-07445` 并在下次执行中禁用谓词。如果此参数的值为 `E` 或 `N`，那么您的系统中可能存在错误 6356566。您可能需要应用错误 6356566 的修复，然后将此参数重置为其默认值。

`r_gran_cols`
列的报告粒度。默认值 `"PARTITION"` 报告表分区列。所有内容都与正在分析的一个 SQL 相关。

`r_gran_hgrm`
表直方图的报告粒度。默认值 `"PARTITION"` 报告表和分区直方图。所有内容都与正在分析的一个 SQL 相关。

`r_gran_segm`
段（表和索引）的报告粒度。默认值 `"PARTITION"` 报告表、索引和分区。所有内容都与正在分析的一个 SQL 相关。

`r_gran_vers`
表的 CBO 统计信息版本粒度。默认值 `"COLUMN"` 报告段及其列的统计信息版本。所有内容都与正在分析的一个 SQL 相关。

`r_rows_table_l`
限制大型 HTML 表或列表中的元素数量。

`r_rows_table_m`
限制中型 HTML 表或列表中的元素数量。

`r_rows_table_s`
限制小型 HTML 表或列表中的元素数量。

`r_rows_table_xs`
限制超小型 HTML 表或列表中的元素数量。

#### SQL 监控与调优

`sql_monitoring`
请注意，使用 SQL 监控（`V$SQL_MONITOR` 和 `V$SQL_PLAN_MONITOR`）需要 Oracle 调优包的许可。如果您没有，可以将此参数设置为 `N`。

`sql_tuning_advisor`
请注意，使用 SQL 调优顾问（STA）`DBMS_SQLTUNE` 需要 Oracle 调优包的许可。如果您没有，可以将此参数设置为 `N`。

`sql_tuning_set`
在使用 `XTRACT` 时为每个计划生成一个 SQL 调优集。

`sta_time_limit_secs`
STA 时间限制（以秒为单位）。参见 `sql_tuning_advisor`。请注意，使用 SQL 调优顾问（STA）`DBMS_SQLTUNE` 需要 Oracle 调优包的许可。

`tcb_time_limit_secs`
TCB（测试用例构建器）时间限制（以秒为单位）。参见 `test_case_builder`。

#### 其他工具与功能

`search_sql_by_sqltext`
`XPLAIN` 方法使用 SQL 文本来在内存和 AWR 中搜索被分析 SQL 的已知执行记录。如果找到了此 SQL 文本的先前执行，则会提取并报告相应的计划。

`skip_metadata_for_object`
此区分大小写的参数允许您指定要跳过元数据提取的对象名称。它用于 `DBMS_METADATA` 出现 `ORA-7445` 错误的情况。您可以指定要跳过的完整或部分对象名称（例如：“CUSTOMERS” 或 “CUSTOMER%” 或 “CUST%” 或 “%”）。要查找元数据出错的对象名称，您可以使用：`SELECT * FROM sqlt$_log WHERE statement_id = 99999 ORDER BY line_id;` 您必须将 `99999` 替换为正确的 `statement_id`。要实际修复 `ORA-7445` 背后的错误，您可以使用 `alert.log` 及其引用的跟踪文件。

`sql_tuning_set`
在使用 `XTRACT` 时为每个计划生成一个 SQL 调优集。

`test_case_builder`
11g 提供了为 SQL 构建测试用例的功能。TCB 使用 API `DBMS_SQLDIAG.EXPORT_SQL_TESTCASE` 实现。SQLT 在可能的情况下调用此 API。当 SQLT 调用 TCB 时，参数 `exportData` 被传递值 `FALSE`，因此不会将任何应用程序数据导出到 TCB 创建的测试用例中。

`trace_analyzer`
SQLT `XECUTE` 调用跟踪分析器 - TRCA（注：224270.1）。TRCA 分析 SQLT 创建的 `10046_10053` 跟踪。它还将跟踪拆分为两个独立的文件 `10046` 和 `10053`。

`validate_user`
验证主方法的用户是否已被授予 `SQLT_USER_ROLE` 或 `DBA` 角色；或者用户是否是 `SQLTXPLAIN` 或 `SYS`。



