# 第 6 章 性能特性

来自 SQL Server 客户顾问团队的朋友 Denzil Ribero 创建了一个很好的 T-SQL 脚本，你可以用它来为我们的 Linux 服务器安装添加多个 `tempdb` 文件。你可以在 [`github.com/denzilribeiro/sqlunattended/blob/master/AddTempdb.sql`](https://github.com/denzilribeiro/sqlunattended/blob/master/AddTempdb.sql) 找到这个脚本。你可以在我们的文档 [`docs.microsoft.com/sql/relational-databases/databases/tempdb-database`](https://docs.microsoft.com/sql/relational-databases/databases/tempdb-database) 和这篇知识库文章 [`support.microsoft.com/help/2154845/recommendations-to-reduce-allocation-contention-in-sql-server-tempdb-d`](https://support.microsoft.com/help/2154845/recommendations-to-reduce-allocation-contention-in-sql-server-tempdb-d) 中阅读更多关于如何确定 `tempdb` 文件大小和数量的内容。

##### 数据库选项

虽然上一节介绍了一些影响 SQL Server 实例的、更有趣的重要性能配置选项，但你也可以在数据库级别进行更改，以帮助提升应用程序的性能。所有这些选项都使用 T-SQL `ALTER DATABASE` 语句进行配置。

-   **PARAMETERIZATION**：我在上一节讨论了如何最小化即席查询的计划缓存占用。即席查询会膨胀计划缓存的一个原因是它们没有被参数化。参数化涉及为那些除了 T-SQL 语句的 `WHERE` 子句中的参数外几乎完全相同的查询缓存一个单一的查询计划。

    SQL Server 默认包含对即席查询执行*简单*参数化的逻辑。简单参数化涵盖的查询范围不广。因此，你可以使用 `ALTER DATABASE` 选项 `SET PARAMETERIZATION FORCED`。使用此数据库选项将导致 SQL Server 扩大可能被参数化的查询范围。那么，你如何知道是否应该考虑此数据库选项呢？请考虑以下要点：

    -   提交未参数化查询的应用程序通常以高查询编译率为特征。你可以在 `object_name = 'SQLServer:SQL Statistics'` 且 `counter_name = 'SQL Compilations/sec'` 时检查 `dm_os_performance_counters` DMV。

    -   什么是“高”？嗯，可能缺少查询参数化的第二个症状是 SQL Server 的高 CPU 利用率。这是因为查询的高编译率会导致更高的 SQL CPU 利用率。

    -   你可以使用 DMV `dm_exec_query_stats` 通过一种称为查询哈希的概念来查找计划缓存中除了字面值外相同的查询。你可以运行如下查询（可以在示例脚本 `queryhash.sql` 中找到）这样的 T-SQL 批处理：

    ```sql
    -- 找出哪些查询哈希值相同但在缓存中是不同的
    --
    SELECT dest.text, deqs.query_hash, count (*) query_count
    FROM sys.dm_exec_query_stats deqs
    CROSS APPLY sys.dm_exec_sql_text(deqs.sql_handle) AS dest
    GROUP BY (query_hash), dest.text
    ORDER BY query_count DESC
    GO
    ```



如果运行此查询后发现最高计数值仅为 2 或 3，那么您可能不存在任何问题。但如果这些查询的计数器值明显更高，则可以通过研究查询参数化来提升性能。

*   前述查询使用起来非常快捷，但仅能查找执行计划已缓存至内存中的查询。若因内存压力导致某些可作为参数化候选方案的查询未缓存，此时查询存储（Query Store）将发挥重要作用。请查阅我们文档中的以下示例，了解如何使用查询存储识别即席工作负载：[`docs.microsoft.com/sql/relational-databases/performance/query-store-usage-scenarios`](https://docs.microsoft.com/sql/relational-databases/performance/query-store-usage-scenarios)。

关于此数据库选项需要说明的是：即便发现需要查询参数化，最佳解决方法仍是修正应用程序。但若无法更改应用程序，启用此选项可能带来收益。

如同任何非默认选项一样，请谨慎调整此设置。虽然启用此选项能减少查询编译次数，但也会迫使许多查询共享相同的编译执行计划。这对某些应用程序而言未必是最佳选择。

## 第六章：性能特性

*   **`READ_COMMITTED_SNAPSHOT`**

SQL Server 应用程序常见的性能问题是*阻塞*。虽然我一直建议遇到阻塞问题的用户尝试排查原因并解决问题，但存在一个数据库选项可缓解甚至彻底解决该问题。首先建议您阅读我们文档中关于锁机制和*行版本控制*基础原理的章节：[`docs.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide`](https://docs.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide)。

常见阻塞场景之一涉及读写操作因访问相同数据而相互阻塞，尤其是在涉及长时间运行的事务时。避免此类场景的选项之一是使用`ALTER DATABASE`语句设置`READ_COMMITTED_SNAPSHOT ON`。启用此数据库选项后，该数据库上下文中的读取操作将看到 T-SQL `SELECT`语句开始时的数据快照/版本，而非使用锁机制。启用此选项会在数据库页和`tempdb`中产生一定开销。此文档链接详细说明了使用行版本控制的前提条件：[`docs.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide#Row_versioning`](https://docs.microsoft.com/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide#Row_versioning)。

应用程序也可使用快照隔离级别，关于快照与行版本控制的更多内容，请参阅我们文档：[`docs.microsoft.com/dotnet/framework/data/adonet/sql/snapshot-isolation-in-sql-server#understanding-snapshot-isolation-and-row-versioning`](https://docs.microsoft.com/dotnet/framework/data/adonet/sql/snapshot-isolation-in-sql-server#understanding-snapshot-isolation-and-row-versioning)。



[快照隔离与行版本控制](https://docs.microsoft.com/dotnet/framework/data/adonet/sql/snapshot-isolation-in-sql-server#understanding-snapshot-isolation-and-row-versioning)。

