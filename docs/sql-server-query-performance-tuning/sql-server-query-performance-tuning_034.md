# 第 2 章 ■ 内存性能分析

在拥有多实例的 SQL Server 服务器上，你需要调整这些内存设置，以确保每个实例都有足够的可用值。务必确保为操作系统和外部进程预留了足够的内存。

SQL Server 中的内存大致可分为缓冲池内存（代表数据页和空闲页）与非缓冲内存（包含线程、DLL、链接服务器等）。SQL Server 使用的大部分内存都分配给了缓冲池。但你也可能遇到缓冲池之外的分配，即所谓的“私有字节”，它们可能引发内存压力，而这在常规监控缓冲池的过程中并不明显。如果你怀疑系统存在此类问题，请对比检查 `Process: sqlservr: Private Bytes` 与 `SQL Server: Buffer Manager: Total pages`。

你也可以使用 `sp_configure` 系统存储过程来管理 `min server memory` 和 `max server memory` 的配置值。要查看这些参数的配置值，请按如下方式执行 `sp_configure` 存储过程：

```sql
EXEC sp_configure 'show advanced options', 1;
GO
RECONFIGURE;
GO
EXEC sp_configure 'min server memory';
EXEC sp_configure 'max server memory';
```

图 2-4 展示了运行这些命令的结果。

## 图 2-4. SQL Server 内存配置属性

请注意，`min server memory` 设置的默认值为 0MB，`max server memory` 设置的默认值为 2147483647MB。

你同样可以使用 `sp_configure` 存储过程来修改这些配置值。例如，要将 `max server memory` 设置为 10GB，并将 `min server memory` 设置为 5GB，请执行以下语句集（对应下载包中的 `setmemory.sql`）：

```sql
USE master;
EXEC sp_configure 'show advanced option', 1;
RECONFIGURE;
exec sp_configure 'min server memory (MB)', 5120;
exec sp_configure 'max server memory (MB)', 10240;
RECONFIGURE WITH OVERRIDE;
```

`min server memory` 和 `max server memory` 配置被归类为高级选项。默认情况下，`sp_configure` 存储过程不会显示/影响高级选项。如前所述，将 `show advanced option` 设置为 1，即可使 `sp_configure` 存储过程能够影响/显示高级选项。

`RECONFIGURE` 语句会更新由 `sp_configure` 设置的内存配置值。由于不建议对包含内存配置值的系统目录进行即席更新，因此 `RECONFIGURE` 语句搭配使用 `OVERRIDE` 标志来强制应用内存配置。如果你通过 Management Studio 进行内存配置，Management Studio 会在配置设置后自动执行 `RECONFIGURE WITH OVERRIDE` 语句。

[www.it-ebooks.info](http://www.it-ebooks.info/)

另一种查看设置（但不进行操作）的方法是使用 `sys.configurations` 系统视图。你可以使用标准的 T-SQL 语句从 `sys.configurations` 中进行选择，而无需执行命令。

你可能需要考虑到 SQL Server 与系统共享内存的情况。具体来说，设想一台同时运行 SQL Server 和 SharePoint 的计算机。这两台服务器都是内存消耗大户，因此会不断相互竞争内存。SQL Server 的动态内存行为使其能够在某个实例中向 SharePoint 释放内存，并在 SharePoint 释放内存时将其收回。你可以通过将 SQL Server 配置为固定内存大小来避免这种动态内存管理开销。不过，请记住，由于 SQL Server 是一个资源消耗极大的进程，强烈建议你使用专用的 SQL Server 生产机器。

现在你已从宏观层面了解了 SQL Server 的内存管理，接下来让我们看看可以用来分析内存压力的性能计数器，如表 2-1 所示。

## 表 2-1. 用于分析内存压力的性能监视器计数器

| **对象(实例[,实例 N])** | **计数器** | **描述** | **值** |
| --- | --- | --- | --- |
| Memory | Available Bytes | 空闲物理内存 | |


