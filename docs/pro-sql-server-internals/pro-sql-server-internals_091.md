# 第 27 章 ■ 扩展事件

#### 创建事件会话

现在是时候将所有内容整合起来，了解扩展事件会话了。我们将重点介绍 T-SQL 实现；不过，如果你更喜欢通过 UI 操作，也可以使用 SQL Server 2012 及以上版本的 Management Studio，或者使用 Jonathan Kehayias 的 SSMS 插件来操作 SQL Server 2008。

每个扩展事件会话都指定了要收集的事件、收集数据的目标以及几个配置属性。清单 27-11 展示了一个创建扩展事件会话的语句，该会话使用`hash_warning`和`sort_warning`事件收集有关`tempdb`溢出的信息。此代码适用于 SQL Server 2012 及以上版本，因为 SQL Server 2008/2008R2 不支持`hash_warning`或`sort_warning`事件。然而，`CREATE EVENT SESSION`命令的语法在所有版本的 SQL Server 中都是相同的。

**清单 27-11.** 创建事件会话

```sql
create event session [TempDB Spills]
on server
add event sqlserver.hash_warning
(
    action ( sqlserver.session_id, sqlserver.plan_handle, sqlserver.sql_text )
    where ( sqlserver.is_system=0 )
),
add event sqlserver.sort_warning
(
    action ( sqlserver.session_id, sqlserver.plan_handle, sqlserver.sql_text )
    where ( sqlserver.is_system=0 )
)
add target package0.event_file
( set filename='c:\ExtEvents\TempDB_Spiils.xel', max_file_size=25 ),
add target package0.ring_buffer
( set max_memory=4096 )
with -- Extended Events session properties
(
    max_memory=4096KB
    ,event_retention_mode=allow_single_event_loss
    ,max_dispatch_latency=15 seconds
    ,track_causality=off
    ,memory_partition_mode=none
    ,startup_state=off
);
```

如前所述，对于异步目标，SQL Server 将收集的事件存储在一组内存缓冲区中，使用多个缓冲区来分离事件的收集和处理。缓冲区的数量及其大小取决于`max_memory`和`memory_partition_mode`设置。SQL Server 使用以下算法，将缓冲区大小向上舍入到下一个 64 KB 边界：

`memory_partition_mode = none`：SQL Server 使用三个中央缓冲区，大小为`max_memory / 3`向上舍入到下一个 64 KB 边界。例如，无论服务器配置如何，`max_memory`为 4000 KB 将创建三个大小为 1344 KB 的缓冲区。

`memory_partition_mode = per_node`：SQL Server 为每个 NUMA 节点创建一组独立的三个缓冲区。例如，在具有两个 NUMA 节点的服务器上，`max_memory`为 4000 KB 将创建六个缓冲区，每个节点三个，每个缓冲区大小为 704 KB。

`memory_partition_mode = per_cpu`：SQL Server 根据公式`2.5 * (number of CPUs)`创建缓冲区数量，并在每个 CPU 上进行分区。例如，在具有 20 个 CPU 的服务器上，`max_memory`为 4000 KB 将创建 50 个大小为 128 KB 的缓冲区。

按 NUMA 节点或 CPU 分区允许多个 CPU 将事件存储在一组独立的缓冲区中，这有助于减少争用，从而降低收集大量事件的扩展事件会话对性能的影响。然而，有一个注意事项。事件需要能够放入缓冲区才能被收集。你可能已经注意到，缓冲区分区增加了缓冲区的数量，这减小了它们的大小。这通常不是问题，因为大多数事件相对较小。但是，也可能定义一个非常大的事件，以至于无法放入缓冲区。在具有大量 NUMA 节点和/或 CPU 的服务器上对事件进行分区时，请确保增加`max_memory`。

> **注意** 你可以检查`sys.dm_xe_sessions`视图的`largest_event_dropped_size`列，以确认缓冲区是否足够大以容纳事件。

`ALLOW_MULTIPLE_EVENT_LOSS`选项允许一个会话在缓冲区满时丢失多个事件。此选项以潜在丢失大量事件为代价，最小化了对 SQL Server 的性能影响。



