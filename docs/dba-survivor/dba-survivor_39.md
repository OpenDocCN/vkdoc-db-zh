# 第六章：基础故障排除

## 图 6-4. 导航至动态管理视图

## 磁盘 I/O

以下是一些有助于你确定磁盘 I/O 问题的动态管理视图（DMV）：
-   `sys.dm_db_file_space_usage`
-   `sys.dm_db_index_operational_stats`
-   `sys.dm_db_index_usage_stats`
-   `sys.dm_exec_query_stats`
-   `sys.dm_exec_query_plan`
-   `sys.dm_io_virtual_file_stats`
-   `sys.dm_io_pending_io_requests`
-   `sys.dm_os_wait_stats`
-   `sys.dm_os_waiting_tasks`

[www.it-ebooks.info](http://www.it-ebooks.info)

![图片 113](img/index-125_1.png)

这里有一个 T-SQL 脚本示例，它将帮助你使用 `sys.dm_io_virtual_file_stats` DMV 来查找每次 I/O 平均毫秒数最高的数据文件：

```sql
SELECT io_stall / (num_of_reads + num_of_writes) AS avg_ms_per_io,
io_stall AS io_stall_ms,
(num_of_reads + num_of_writes) AS num_io,
DB_NAME (database_id) AS database_name,
file_id
FROM sys.dm_io_virtual_file_stats (null, null)
WHERE (num_of_reads + num_of_writes) > 500
ORDER BY 1 DESC
```

### 内存

以下是一些有助于你确定内存问题的 DMV：
-   `sys.dm_os_cache_counters`
-   `sys.dm_os_memory_clerks`
-   `sys.dm_os_memory_cache_clock_hands`
-   `sys.dm_os_process_memory`
-   `sys.dm_os_ring_buffers`
-   `sys.dm_os_sys_memory`
-   `sys.dm_os_virtual_address_dump`
-   `sys.dm_os_wait_stats`
-   `sys.dm_os_waiting_tasks`

这里有一个 T-SQL 脚本示例，它将帮助你使用 `sys.dm_os_buffer_descriptors` DMV 来查找当前驻留在缓冲池中数据页最多的数据库：

```sql
SELECT COUNT(*) AS cached_pages,
CASE database_id WHEN 32767 THEN 'ResourceDb'
ELSE DB_NAME(database_id)
END AS [database]
FROM sys.dm_os_buffer_descriptors
GROUP BY DB_NAME(database_id), database_id
ORDER BY cached_pages DESC
```

### CPU

以下是一些有助于你确定 CPU 问题的 DMV：
-   `sys.dm_exec_cached_plans`
-   `sys.dm_exec_query_stats`
-   `sys.dm_exec_query_optimizer_info`
-   `sys.dm_exec_requests`
-   `sys.dm_exec_sessions`
-   `sys.dm_os_schedulers`
-   `sys.dm_os_wait_stats`
-   `sys.dm_os_waiting_tasks`

[www.it-ebooks.info](http://www.it-ebooks.info)

![图片 114](img/index-126_1.png)

第六章：基础故障排除 108

这里有一个 T-SQL 脚本示例，它将帮助你使用 `sys.dm_os_schedulers` DMV 来定位潜在的 CPU 问题，方法是识别当前正在 CPU 上等待运行的任务数量：

```sql
SELECT scheduler_id, current_tasks_count, runnable_tasks_count
FROM sys.dm_os_schedulers
WHERE scheduler_id < 255
```

你有没有注意到我把这两个 DMV 列了两次？
-   `sys.dm_os_wait_stats`
-   `sys.dm_os_waiting_tasks`

这些 DMV 有助于识别自服务器上次重启以来活动的等待统计和等待类型。等待统计信息可能是你在排查性能问题时最强大的工具之一。即使是下面这个简单的查询：

```sql
select top 10 *
from sys.dm_os_wait_stats
order by wait_time_ms desc
```

也能在短时间内提供大量信息。当其他数据库管理员还在摸索着启动 SQL Profiler 时，你可能已经隔离出问题查询、确定了它在等待什么，并提出了可能的解决方案。

## 等待统计

大多数人对商店里卖的室内/室外温度计都很熟悉。这些设备让你可以同时看到两个地方的温度。与传统的在窗外放置温度计的方法相比，它们并没有真正优势。事实上，除非探头放置得当，否则你在室外得到的读数可能会偏差很大。最终，我们都是通过打开门窗，感受皮肤上的空气来检查室外的真实温度。

现在想象一下，你的电话响了，另一端的声音说“服务器很慢”。你手头有一些工具可以深入数据库引擎内部，看看发生了什么。简而言之，你在外部观察一个“温度”，同时你也可以深入内部查看另一个“温度”。



等待统计是你**切身感受**系统状况的一种方式。它们能让你立即发现问题所在，并且通常能直接提供解决方案。在故障排查方面，它们堪称一块**隐藏的瑰宝**，能够让你比使用其他工具快得多地诊断出性能问题。

[www.it-ebooks.info](http://www.it-ebooks.info)

![Image 115](img/index-127_1.png)

