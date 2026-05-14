# 第 5 章 SQL Server 工具

## 注意
这是一个未记录在案的功能，并且未来可能轻易改变。如果启用跟踪标志`2588`，它将使`DBCC HELP`能够显示代码中的所有`DBCC`命令，包括那些不受支持的命令。再次强调，对于任何未记录的`DBCC`命令，您都应该格外小心。它们并非为生产环境而设计，除非得到 Microsoft 的指导，否则使用它们甚至可能导致您的`SQL Server`出现问题。

##### 跟踪标志
跟踪标志可用于在`SQL Server`中启用特定功能，或获取可用于调试或诊断的技术细节的洞察。

可以将跟踪标志视为`SQL Server`引擎代码中的**动态**决策点，可用于打开或关闭某项功能或行为。

跟踪标志在`SQL Server`中有着悠久的历史，最初是作为`SQL Server`引擎开发人员的调试辅助工具。开发人员希望有一种方法可以在不重新构建代码的情况下打开或关闭某些行为。如今，在`SQL Server`源代码中存在数百个可能的跟踪标志。然而，唯一受支持的官方跟踪标志要么列在此文档页面[`docs.microsoft.com/sql/t-sql/database-console-commands/dbcc-traceon-trace-flags-transact-sql`](https://docs.microsoft.com/sql/t-sql/database-console-commands/dbcc-traceon-trace-flags-transact-sql)上，要么在官方的`Microsoft`知识库文章中。

跟踪标志是针对单个会话启用的，还是对所有会话全局启用。有些跟踪标志只能全局启用，必须在`SQL Server`启动时打开才能被代码识别。

要为会话启用跟踪标志，您需要使用`T-SQL`命令`DBCC TRACEON`。要关闭跟踪标志，则使用命令`DBCC TRACEOFF`。如果您在使用`DBCC TRACEON`开启或关闭跟踪标志时使用特殊参数`-1`，则从那时起，跟踪标志将全局应用于所有会话。跟踪标志可以在`SQL Server`启动时打开，以通过`mssql-conf`脚本全局启用，如文档[`docs.microsoft.com/sql/linux/sql-server-linux-configure-mssql-conf#traceflags`](https://docs.microsoft.com/sql/linux/sql-server-linux-configure-mssql-conf#traceflags)所述。

### 注意
与`Windows`版`SQL Server`一样，`Linux`版`SQL Server`支持使用`sqlservr`命令行参数`-T`设置启动跟踪标志。对于`Linux`上的`SQL Server`，我建议您始终使用`mssql-conf`脚本来设置或取消设置启动跟踪标志。

您可以使用命令`DBCC TRACESTATUS`查看所有已启用跟踪标志的状态。

许多有文档记录的跟踪标志最终会保留在产品中，以启用特定的性能优化或修复用户遇到的问题。这些更改通常需要跟踪标志，以避免对不需要此行为的客户造成问题。在某些情况下，我们尝试使用其他方法来启用此类增强功能，例如`ALTER DATABASE`选项或通过`sp_configure`等命令。

以下是我经常使用的一些常见跟踪标志，您可以在文档中找到：

**1222**
以`XML`格式在`SQL Server ERRORLOG`中显示死锁信息的详细信息。您可以在我们的文档中找到关于锁定和死锁主题的讨论，包括此跟踪标志产生的`XML`死锁信息的详细信息。

```
DBCC HELP('?')
GO
```

```
DBCC HELP ('CHECKDB')
GO
```



[`docs.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide#Lock_Engine`](https://docs.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide#Lock_Engine)

