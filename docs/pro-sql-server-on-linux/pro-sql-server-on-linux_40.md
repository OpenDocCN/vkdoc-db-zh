# 第 6 章 性能能力

性能的多少能有所帮助，并且此功能已弃用，因此可能在未来的版本中被移除。请参阅我们的文档了解如何执行此操作：[`docs.microsoft.com/sql/database-engine/configure-windows/default-trace-enabled-server-configuration-option`](https://docs.microsoft.com/sql/database-engine/configure-windows/default-trace-enabled-server-configuration-option)。

## 线程

我已经提到了工作线程的概念，以及 SQL Server 拥有这些线程池来满足诸如登录和查询等任务需求的事实。SQL Server 在启动时初始化此池，并根据需要增加或减少线程数。

为了避免消耗过多资源，SQL Server 对池中的线程数设置了最大值。默认情况下，SQL Server 根据在 Linux 服务器上检测到的 CPU 数量来计算此最大值。你可以在 `sys.dm_os_sys_info.max_workers_count` 列中看到此计算值。此值的计算公式可在我们的文档中找到：[`docs.microsoft.com/sql/database-engine/configure-windows/configure-the-max-worker-threads-server-configuration-option`](https://docs.microsoft.com/sql/database-engine/configure-windows/configure-the-max-worker-threads-server-configuration-option)。

在大多数情况下，如果此工作线程池计算不足以服务应用程序的工作负载，问题通常是并发性问题，例如许多线程在某个资源上被阻塞。然而，在某些情况下，增加默认值可能会提升高并发、多用户工作负载的性能。对于这些情况，请使用 `sp_configure` 系统过程来更改 `max worker threads` 配置值。你需要重新启动 SQL Server 才能使此更改生效。请使用本节中的文档资源获取完整语法。

## 计划缓存

计划缓存可能是一个宝贵的资源，用于确保缓存最大数量的查询和对象，以避免频繁的查询编译。某些应用程序工作负载不可避免地使用许多 `single-user ad hoc queries`。这些是 T-SQL 语句，它们不以存储过程对象的形式存在，没有 `parameterized`，并且通常只执行一次。为了最小化此类查询的内存占用，你可以通过 `sp_configure` 启用 `optimize for ad hoc workloads` 配置选项。这将最小化这些类型的临时查询对内存的影响。

你可以在我们的文档中确定你的应用程序工作负载是否符合此选项的需求以及如何配置它：[`docs.microsoft.com/sql/database-engine/configure-windows/optimize-for-ad-hoc-workloads-server-configuration-option`](https://docs.microsoft.com/sql/database-engine/configure-windows/optimize-for-ad-hoc-workloads-server-configuration-option)。

## Tempdb 文件

我在第 4 章讨论临时表的使用时，谈到了 tempdb 的用途。由于 tempdb 的独特用途涉及数据库页面的频繁分配和释放，使用单个 tempdb 数据库文件可能会导致用于跟踪分配信息的系统数据库页面出现并发问题，并且


任何合理的多用户工作负载。

为了使这些并发场景更具可扩展性，你可以通过为 `tempdb` 数据库创建多个数据库文件来改进对这些系统数据库页面的访问。Linux 上的 SQL Server 在安装时默认仅提供单个 `tempdb` 数据库文件。因此，在任何拥有一个以上 CPU 的 Linux 服务器上，你都应该计划创建额外的 `tempdb` 数据库文件。

