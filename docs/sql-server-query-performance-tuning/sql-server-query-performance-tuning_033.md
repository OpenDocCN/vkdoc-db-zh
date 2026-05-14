# 第 2 章 ■ 内存性能分析

增加容量意味着添加更多磁盘或升级到更快的磁盘。降低到达率则意味着识别磁盘子系统高 I/O 请求的原因，并采取措施减少其数量。

例如，您或许可以通过在表上添加合适的索引以限制访问的数据量，或者通过编写包含更多或更好过滤条件的 `T-SQL` 语句（在 `WHERE` 子句中），来减少 I/O 请求。

## 内存瓶颈分析

内存可能是一个麻烦的瓶颈，因为内存瓶颈也会在其他资源上显现出来。对于运行 `SQL Server` 的系统来说尤其如此。当 `SQL Server` 的缓存（或内存）不足时，`SQL Server` 内部的一个进程（称为 `lazy writer`）必须大量工作，以在 `SQL Server` 内维持足够的空闲内部内存页。这会消耗额外的 CPU 周期并执行额外的物理磁盘 I/O 操作，将内存页写回磁盘。

### SQL Server 内存管理

`SQL Server` 在一个称为 `buffer pool` 的大型内存池中管理数据库的内存，包括数据和查询执行计划的内存需求。该内存池过去由一组 8KB 缓冲区来管理内存。现在则有用于数据页和计划缓存页、空闲页等的多种页面分配。`buffer pool` 通常是 `SQL Server` 内存中最大的部分。`SQL Server` 通过动态增长或收缩其内存池大小来管理内存。

您可以在 `SQL Server Management Studio (SSMS)` 中为 `SQL Server` 配置动态内存管理。

转到“服务器属性”对话框的“内存”文件夹，如图 2-3.所示。

[`www.it-ebooks.info`](http://www.it-ebooks.info/)

![](img/`index-29_1.jpg`)

***图 2-3.** `SQL Server` 内存配置*

动态内存范围通过两个配置属性控制：`Minimum(MB)` 和 `Maximum(MB)`。

- `Minimum(MB)`，也称为 `min server memory`，作为内存池的下限值。一旦内存池达到与下限值相同的大小，`SQL Server` 可以继续提交内存池中的页面，但不能将其收缩到小于该下限值。请注意，`SQL Server` 并非以 `min server memory` 配置值启动，而是根据需要动态提交内存。
- `Maximum(MB)`，也称为 `max server memory`，作为上限值以限制内存池的最大增长。这些配置设置立即生效，无需重启。在 `SQL Server 2014` 中，32 位系统的最大内存下限为 64MB，64 位系统为 128MB。

Microsoft 建议您对 `SQL Server` 使用动态内存配置，其中 `min server memory` 设为 0，`max server memory` 设为允许操作系统使用一些内存的值（假设机器上只有单个实例）。

分配给操作系统的内存量取决于系统本身。对于大多数具有 8 GB –16GB 内存的系统，您应该为操作系统保留大约 2GB – 4GB。随着服务器内存量的增加，您需要为操作系统分配更多内存。一个常见的建议是，对于总系统内存在 32GB 以上的部分，每 16GB 额外分配 4GB。您需要根据自己系统的需求和内存分配情况进行调整。您不应将其他内存密集型应用程序与 `SQL Server` 运行在同一服务器上，但如果必须这样做，我建议您首先估算其他应用程序所需的内存量，然后配置 `SQL Server` 的 `max server memory` 值

[`www.it-ebooks.info`](http://www.it-ebooks.info/)

![](img/`index-30_1.jpg`)

第 2 章 ■ 内存性能分析

*设置此值以防止其他应用程序耗尽 `SQL Server` 的内存。*在 `SQL Server` 独立运行的系统上，我倾向于将 `minimum server memory` 设置为与 `max` 值相等，并直接禁用动态管理。


