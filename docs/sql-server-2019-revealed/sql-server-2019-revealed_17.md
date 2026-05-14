# 服务器或数据库配置

在本节中，我列出了几个您可能需要考虑的 SQL Server 配置选项。此外，`mssql-conf` 脚本、T-SQL `sp_configure` 和 `ALTER SERVER CONFIGURATION` 还提供了许多其他选项。此外，数据库在创建时或通过 `ALTER DATABASE` 还有许多其他配置选项。

虽然可以使用 T-SQL 语句修改正在运行的 SQL Server 容器，但在使用容器时，您应仔细考虑这是否是正确的策略。例如，如果您对需要重启的容器应用了 `sp_configure` 更改，您将必须重启容器才能使更改生效。此外，您需要确保为系统数据库使用持久化卷，这样您的更改才不会丢失。

另一个选项是构建一个自定义镜像（就像本章中为复制示例展示的容器那样），在 SQL Server 启动后运行您所需的任何配置脚本。

例如，假设您希望确保您的 SQL Server 对环境中的任何 SQL Server 容器强制执行特定的 `max degree of parallelism` 值。一种方法是构建一个自定义容器镜像，其中包含一个设置所需 `maxdop` 值的脚本。您甚至可以为此打标签和命名，以便您知道为某个 SQL 容器使用的选项。现在，这些用于构建容器的脚本以及 T-SQL 脚本可以成为您的变更控制和 CI/CD 生命周期的一部分。

这种方法唯一的缺点是针对那些不需要重启 SQL Server 实例（或数据库实例）的配置更改。对于这些情况，您仍然可以构建一个包含新所需配置更改的特定容器镜像，但也可以直接将配置更改应用到正在运行的 SQL Server。

另一个有趣的问题是应用需要重启的配置更改（这种情况比您想象的要少）。然而，如果您确实遇到这种情况，您将必须在应用配置更改并启动容器后立即重启它。

对于任何 `mssql-conf` 更改，您应使用与您想要设置匹配的环境变量，正如本章中 `docker run` 示例所示。一个使用此选项进行 DTC 配置的绝佳示例可以在 [`https://github.com/microsoft/sql-server-samples/tree/master/samples/containers/dtc`](https://github.com/microsoft/sql-server-samples/tree/master/samples/containers/dtc) 找到。如果由于某种原因，某个 `mssql-conf` 设置没有对应的等效环境变量设置，您可以创建一个包含预先创建的 `mssql.conf` 文件的自定义镜像。您可以使用如下的 Dockerfile：

```
FROM microsoft/mssql-server-linux:latest
COPY ./mssql.conf /
RUN mkdir /var/opt/mssql
RUN mv ./mssql.conf /var/opt/mssql
CMD ["/opt/mssql/bin/sqlservr"]
```

其中您的 `mssql.conf` 文件包含了所有需要的配置值。您可以在 [`https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-mssql-conf#mssql-conf-format`](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-configure-mssql-conf#mssql-conf-format) 了解此文件的协议格式。

