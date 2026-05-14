# SQL Server 中的内存配置

32 位版本中的内存配置；如今绝对没有理由再使用 32 位操作系统和 SQL Server。64 位版本提供了更好的性能，升级是有益的。需要再次提及的是，SQL Server 2016 甚至不提供 32 位版本。

## 内存分配

SQL Server 中的所有内存分配都通过 SQLOS 完成。在内部，SQLOS 根据服务器的 NUMA 配置将内存分区为内存节点。例如，具有四个 NUMA 节点的服务器将有四个内存节点。没有 NUMA 硬件的服务器将只有一个内存节点。

每个内存节点都有一个 `memory allocator` 组件，负责内存分配，通过调用各种 Windows API 方法执行分配。在 SQL Server 2012 之前，内存节点为单页和多页分配使用不同的内存分配器，分别称为 `single-page allocator` 和 `multi-page allocator`。从 SQL Server 2012 开始，只有一个称为 `any size page allocator` 的内存分配器，它处理两种类型的分配。您可以使用 `sys.dm_os_memory_nodes` 视图按内存节点跟踪内存使用情况和分配。

SQL Server 内存架构中还有另一个关键元素，称为 `memory clerks`。SQL Server 的每个主要组件都有自己的内存 clerk，它充当组件和内存分配器之间的代理。当组件需要内存时，它向相应的内存 clerk 请求，然后 clerk 从内存分配器获取内存。每个内存 clerk 跟踪分配统计信息，这允许您确定各个组件的内存使用情况。

列表 28-15 显示了返回服务器上十大内存消耗者的代码。图 28-12 显示了其中一台生产服务器的输出。在 2012 之前的 SQL Server 中，由于 SQL Server 使用不同的内存分配器，您应该将 `pages_kb` 列替换为 `single_page_kb` 和 `multi_pages_kb` 列的总和。

***列表 28-15.*** 分析内存 clerks (SQL Server 2012 及以上)

```sql
select top 10
    [type] as [Memory Clerk]
    ,convert(decimal(16,3),sum(pages_kb) / 1024.0) as [Memory Usage(MB)]
from sys.dm_os_memory_clerks with (nolock)
group by [type]
order by sum(pages_kb) desc
```

***图 28-12.** 锁存统计信息*

第 28 章 ■ 系统故障排除

## 一些常见的内存 clerks 如下：

### `MEMORYCLERCK_SQLBUFFERPOOL` clerk 显示缓冲池的内存使用情况。
该 clerk 内存使用率高是正常现象。

### `CACHESTORE_SQLCP` clerk 显示即席查询、自动参数化和预执行计划的内存使用情况。
该 clerk 内存使用率高通常表示系统中存在大量即席查询。检查是否启用了 `Optimize for Ad-hoc Workload` 设置，然后对查询进行参数化。

### `CACHESTORE_OBJCP` clerk 负责存储过程、函数和触发器的已编译执行计划的内存使用情况。

### `CACHESTORE_PHDR` clerk 指示绑定树（由查询优化器创建的结构）的内存使用情况。

### `USERSTORE_TOKENPERM` clerk 显示安全令牌存储的内存使用情况。
某些 SQL Server 版本存在与 `USERSTORE_TOKENPERM` 增长相关的已知错误。如果遇到该 clerk 内存使用量过大，请应用最新的服务包。作为临时解决方案，可以使用 `DBCC FREESYSTEMCACHE('TokenAndPermUserStore')` 命令清除令牌存储。

### `OBJECTSTORE_LOCK_MANAGER` clerk 显示锁管理器的内存使用情况。
锁管理器消耗大量内存可能表示系统中的事务策略不理想；例如，在长时间运行的事务中使用大批量更新。

### `MEMORYCLERK_SQLQERESERVATIONS` clerk 负责查询内存授予的预留。
该 clerk 消耗大量内存表示过度的内存授予，这会减小缓冲池的大小。有益的做法是



