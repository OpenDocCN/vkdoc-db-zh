# SQL Server 扩展事件

## 事件会话配置

SQL Server 会在缓冲区满时，和/或根据 `max_dispatch_latency` 设置指定的时间间隔（默认 30 秒），将事件会话数据刷新到异步目标。

`startup_state` 选项控制事件会话是否应在 SQL Server 启动时自动启动。

最后，`track_causality` 选项允许你跟踪事件的序列，并查看不同事件如何相互引发。一个这样的场景示例是：一个 SQL 语句触发了文件读取事件，该事件又触发了带有 `PAGELATCHIO` 等待的等待事件，等等。启用此选项后，SQL Server 会添加一个唯一的活动 ID，该 ID 是 GUID 值（对于任务保持不变）和事件序列号的组合。

## 管理会话

创建事件会话后，可以使用 `ALTER EVENT SESSION` 命令启动或停止它，或使用 `DROP EVENT SESSION` 命令删除它，如清单 27-12 所示。

清单 27-12. 操作事件会话

```sql
-- 启动事件会话
alter event session [TempDB Spills] on server state=start;

-- 停止事件会话
alter event session [TempDB Spills] on server state=stop;

-- 删除事件会话
drop event session [TempDB Spills] on server;
```

#### 处理事件数据

Management Studio 2012 及以上版本提供了一个 UI，用于监控事件的实时流或检查已在目标中收集的数据。这个 UI 非常方便灵活，它允许你自定义显示事件的网格布局，让你能够对事件数据进行分组、聚合，并将其导出到数据库表、事件或 CSV 文件。但是，在连接到实时事件流时应小心，因为事件会话生成事件的速度可能快于 Management Studio 消费它们的速度。发生这种情况时，Management Studio 会断开与实时数据流的连接，以避免对服务器性能产生负面影响。

在本节中，我将不讨论如何使用 Management Studio UI，而是专注于 T-SQL 实现。不过，我鼓励你尝试使用 Management Studio。尽管扩展事件管理 UI 存在一些局限性，但在大量情况下它已足够。

### 关键数据管理视图

可用于检查事件会话和数据的关键扩展事件数据管理视图包括：

- `sys.dm_xe_sessions` 视图提供有关活动事件会话的信息。它显示会话的配置参数以及执行统计信息，例如，如果使用了 `NO_EVENT_LOSS` 选项，则显示丢弃的事件数量或事件收集导致阻塞的时间量。
- `sys.dm_xe_session_targets` 视图返回有关目标的信息。该视图的关键列之一是 `event_data`。某些目标（例如 `ring_buffer` 或 `histogram`）通过此列公开收集的事件数据。对于其他目标（如 `event_file`），`event_data` 列包含元数据信息，例如文件名和会话统计信息。
- `sys.dm_xe_sessions_object_columns` 视图公开绑定到会话的对象的配置值。你可以使用此视图获取目标的配置属性；例如，事件文件路径。

> **注意**：你可以在 [`technet.microsoft.com/en-us/library/bb677293.aspx`](http://technet.microsoft.com/en-us/library/bb677293.aspx) 找到有关扩展事件 DMV 的更多信息。

现在，让我们看看如何访问存储在不同目标中的数据。

## 使用 ring_buffer 目标

`ring_buffer` 事件数据通过 `sys.dm_xe_session_targets` 视图中的 `event_data` 列公开。清单 27-13 展示了如何解析由我们定义的 `TempDB Spill` 事件会话（见清单 27-11）收集的数据。

清单 27-13. 检查 ring_buffer 目标数据

```sql
;with TargetData(Data)
as
(
```



