# 资源控制

默认情况下，SQL Server 容器（如同所有容器）可以访问主机服务器上的所有 CPU 和内存资源。Docker 提供了一种方法来控制和管理任何容器对这些资源的访问。例如，`docker run` 命令提供了以下选项：

`-m` – 控制容器可访问的内存量。

`-cpuset-cpus` – 控制容器内的线程可以在哪些 CPU 上运行。使用此选项时请务必小心。SQL Server 不会基于此选项限制调度器的数量。但是，Docker（使用 cgroups）将强制规定所有 SQL Server 线程在哪些 CPU 上运行。如果您使用此选项，我建议结合使用 SQL Server 亲和性配置 `ALTER SERVER CONFIGURATION`。

您可以在 [`https://docs.docker.com/config/containers/resource_constraints/`](https://docs.docker.com/config/containers/resource_constraints/) 阅读更多关于 Docker 容器资源使用的内容。

虽然这些选项确实有效，但对于 SQL Server，我建议您使用 SQL Server 配置的内置功能来控制内存和 CPU 资源。

例如，考虑以下您可用的选项：

`memorylimitmb` – 此选项控制在 Linux 上暴露给 SQL Server 使用的物理内存量。您可以在 [`https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-mssql-conf#memorylimit`](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-mssql-conf#memorylimit) 阅读更多关于此选项的内容。

`"max server memory"` – 此 `sp_configure` 选项对于 SQL Server 用户非常熟悉，用于控制 SQL Server 引擎在 `memorylimitmb` 空间内使用的内存量。

`ALTER SERVER CONFIGURATION` – 此 T-SQL 命令允许您设置 SQL Server 将运行于哪些 NUMA 节点和/或 CPU 的亲和性。您可以在 [`https://docs.microsoft.com/en-us/sql/t-sql/statements/alter-server-configuration-transact-sql`](https://docs.microsoft.com/en-us/sql/t-sql/statements/alter-server-configuration-transact-sql) 阅读更多关于此选项的内容。

`Resource Governor` – 资源调控器有助于控制 CPU、内存和 I/O 资源，特别是在应用程序或工作负载级别。您可以在 [`https://docs.microsoft.com/en-us/sql/relational-databases/resource-governor/resource-governor`](https://docs.microsoft.com/en-us/sql/relational-databases/resource-governor/resource-governor) 阅读更多关于资源调控器的内容。

公平地说，并非在 Linux 上运行于 `sqlservr` 中（使用 SQLPAL）的所有内容都受这些 T-SQL 选项控制。由 SQL Agent、DTC、Polybase 或运行在 SQLPAL 内的、位于 SQL 引擎之外的其他代码进行的处理，在作为容器运行时可能需要一些资源控制，如果您有需要，本节列出的 Docker 选项可能会有用。然而，绝大多数的 CPU 和内存消耗来自数据库引擎，而 SQL 提供了相应的选项让您能实现所需的控制。

