# 图 26-16. Plan_handle 和 sql_handle

图 26-16 展示了相同 SQL 批处理的两个不同计划，这是由 SET 选项的差异导致的。

你可以使用 `sys.dm_exec_query_plan` 函数获取执行计划的 XML 表示形式，该函数接受 `plan_handle` 作为参数。但是，如果 XML 计划的嵌套层级超过 128 层，由于 XML 数据类型的限制，它不会返回查询计划。在这种情况下，你可以使用 `sys.dm_exec_text_query_plan` 函数，它返回 XML 计划的文本表示形式。

你可以使用 `sys.dm_exec_requests` 视图 检索当前正在执行的请求信息。清单 26-29 展示了一个查询，该查询返回来自用户会话的、当前正在运行的请求数据，并按运行时间降序排序。

