# SQL 数据库登录与用户管理入门

#### 创建新登录名

要创建新的登录名，必须连接到 `master` 数据库。连接后，使用 `CREATE LOGIN` 命令创建登录名。

连接到 `master` 数据库（使用管理员账户或任何被授予 `loginmanager` 角色的账户），并运行以下命令：

```sql
CREATE LOGIN test WITH PASSWORD = 'T3stPwd001'
```

此时，应该有一个名为 `test` 的新登录名可用。但直到创建用户后才能实际登录。要验证登录名是否已创建，请运行以下命令：

```sql
select * from sys.sql_logins
```

*图 1-12. 从 `master` 数据库查看 SQL 登录名*

如果尝试在用户数据库中创建登录账户，会收到错误。登录名必须在 `master` 数据库中创建。

*图 1-13. 在用户数据库中创建登录名时的错误*

如果密码复杂度不够，会收到类似这样的错误信息。

*图 1-14. 密码复杂度不足时的错误*

`Note`：在云环境中运行时，选择强密码至关重要，即使你的数据库仅用于开发或测试。强密码和防火墙规则是抵御数据库攻击的重要安全防线。第 3 章将深入探讨安全性。

#### 创建新用户

现在可以为测试登录名创建用户账户。为此，使用管理员账户连接到一个用户数据库（如果此登录名需要能够连接到 `master` 数据库，也可以在那里创建用户），并运行以下命令：

```sql
CREATE USER test FROM LOGIN test
```

如果未先创建登录账户就尝试创建用户，会收到类似这样的消息。

*图 1-15. 未先创建登录账户就创建用户时的错误*

`Note`：不能创建与管理员登录名同名的用户。这是因为管理员登录名已映射到用户 `dbo`。你可以在图 1-8 的属性窗格中找到管理员登录名。

### 分配访问权限

至此，已在 `master` 数据库中创建了登录账户，在用户数据库中创建了用户账户。但此用户账户尚未被分配任何访问权限。

要允许 `test` 账户对所选的用户数据库拥有无限制访问权限，需要将该用户添加到 `db_owner` 角色组：

```sql
EXEC sp_addrolemember 'db_owner', 'test'
```

现在，你已准备好使用 `test` 账户来创建表、视图、存储过程等。

`Note`：在 SQL Server 中，用户账户会自动分配到 `public` 角色。然而，在 SQL Database 中，出于增强安全性的考虑，`public` 角色不能分配给用户账户。因此，必须授予特定的访问权限才能使用用户账户。

## SQL Database 计费理解

SQL Database 采用按需付费模式，费用包括基于每日消耗的数据库数量和大小的月度费用，以及基于实际带宽使用量的费用。使用 SQL Database，你只需为实际使用量付费；因此，一个 7GB 的数据库实例会比 8GB 的实例便宜。并且正如你所料，每 GB 空间的使用成本随着数据库规模的增大而降低。因此，一个 100GB 的数据库实例比两个 50GB 的实例更便宜。此外，在撰写本文时，当 SQL Database 实例的消费应用程序部署为 Windows Azure 应用程序或服务，并且与数据库属于同一地理区域时，带宽费用将被免除。

要从计费角度查看当前的带宽消耗和已配置的数据库，可以运行以下命令：

```sql
SELECT * FROM sys.database_usage -- databases defined
SELECT * FROM sys.bandwidth_usage -- bandwidth
```

第一条语句返回特定类型（Web 或 Business 版）每日可用的数据库数量。此信息用于计算月度费用。第二条语句显示每个数据库的每小时消耗明细。

请注意，此数据库中存储的信息可保留一段时间，但最终会被微软清除。你应该能在此表中查看最近三个月的数据。

*图 1-16. 每小时带宽消耗*

图 1-16 显示了返回带宽消耗的语句的示例输出。该语句返回以下信息：
*   `time`：带宽适用的小时。例如，查看 2011 年 12 月 22 日凌晨 1 点到 2 点之间的汇总。
*   `database_name`：提供汇总信息的数据库。
*   `direction`：数据移动方向。`Egress` 表示出站数据，`Ingress` 表示入站数据。
*   `class`：如果数据是从 Windows Azure 外部的应用程序（例如 SQL Server Management Studio 应用程序）传输的，则为 `External`。如果数据是从 Windows Azure 内部传输的，则此列包含 `Internal`。
*   `time_period`：数据传输的时间窗口。
*   `quantity`：传输的数据量，以千字节（KB）为单位。

请访问 [`www.microsoft.com/windowsazure`](http://www.microsoft.com/windowsazure) 获取最新的定价信息。

## SQL Database 的限制

正如你所见，创建数据库和用户需要手动编写脚本和切换数据库连接。

SQL Server 和 SQL Database 之间的根本差异在于云计算的基本设计原则，必须仔细权衡性能、易用性和可扩展性。用户数据库可能位于不同物理服务器上的事实带来了自然的限制。此外，针对 SQL Database 设计应用程序和服务需要你对这些限制有深刻的理解。

### 安全性

第 3 章将深入介绍安全性，但以下列表总结了在部署 SQL Database 实例之前需要考虑的重要安全事项。从安全角度来看，你需要考虑以下约束：

*   `Encryption`：尽管 SQL Database 使用 SSL 进行数据传输，但它不支持 SQL Server 中可用的数据加密功能。不过，SQL Database 提供了对哈希函数的支持。
*   `SSPI authentication`：SQL Database 仅支持数据库登录。因此，不支持使用安全支持提供程序接口（SSPI）的网络登录。
*   `Connection constraints`：在某些情况下，数据库连接会因以下原因之一而关闭：
    *   资源使用过度
    *   长时间运行的查询


