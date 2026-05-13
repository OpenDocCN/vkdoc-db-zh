# 第 15 章 其他云提供商审计选项

我的建议是在审计数据仍处于数据库实例中、尚未传输到 S3 时进行查询。如果你按照清单 15-2 的方式设置服务器审计，并像清单 15-4 那样添加适当的过滤器，那么你不会产生大量的审计数据。你可能可以每小时收集一次，而不会丢失任何审计数据。确保不丢失审计数据的最佳方法是密切关注其收集速度。然后，你可以根据这个速度来设置查询计划。

**注意** 有关 RDS 中 SQL Server 审计的更多信息，请访问 https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.SQLServer.Options.Audit.html

值得高兴的是，AWS RDS 允许你使用 SQL Server 代理和链接服务器。你可以像第 11 章“集中审计数据”中那样进行集中化设置。这将使查询和报告多个 RDS 实例变得更加容易。

#### AWS RDS 扩展事件

在 AWS RDS 中使用扩展事件相当简单。主要需要注意的是文件名。你必须将扩展事件放在路径 `D:\rdsdbdata\Log\` 中，就像我在清单 15-6 中所做的那样。

***清单 15-6.*** 在 RDS 中设置扩展事件

```sql
CREATE EVENT SESSION [auditxel] ON SERVER
ADD EVENT sqlserver.rpc_completed(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_hostname,
        sqlserver.database_name,
        sqlserver.sql_text,
        sqlserver.username
    )
    WHERE ([sqlserver].[username]=N'josephine')
),
ADD EVENT sqlserver.sql_batch_completed(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_hostname,
        sqlserver.database_name,
        sqlserver.sql_text,
        sqlserver.username
    )
    WHERE ([sqlserver].[username]=N'josephine')
)
ADD TARGET package0.event_file
(
    SET filename=N'D:\rdsdbdata\Log\auditxel',
    max_file_size=(10),
    max_rollover_files=(5)
)
WITH (
    MAX_MEMORY=4096 KB,
    EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY=30 SECONDS,
    MAX_EVENT_SIZE=0 KB,
    MEMORY_PARTITION_MODE=NONE,
    TRACK_CAUSALITY=OFF,
    STARTUP_STATE=ON
);

ALTER EVENT SESSION [auditxel] ON SERVER STATE=START;
```

**注意** 扩展事件仅在企业版和标准版上有效。

要查询你的扩展事件，请使用清单 15-7 中的脚本。

***清单 15-7.*** 在 RDS 中查询扩展事件数据

```sql
SELECT
    n.value('(@timestamp)[1]', 'datetime') as timestamp,
    n.value('(action[@name="sql_text"]/value)[1]', 'nvarchar(max)') as [sql],
    n.value('(action[@name="client_hostname"]/value)[1]', 'nvarchar(50)') as [client_hostname],
    n.value('(action[@name="username"]/value)[1]', 'nvarchar(50)') as [user],
    n.value('(action[@name="database_name"]/value)[1]', 'nvarchar(50)') as [database_name],
    n.value('(action[@name="client_app_name"]/value)[1]', 'nvarchar(50)') as [client_app_name]
FROM
    (SELECT CAST(event_data as XML) as event_data
     FROM sys.fn_xe_file_target_read_file('D:\rdsdbdata\log\auditxel*.xel', NULL, NULL, NULL)) as ed
CROSS APPLY ed.event_data.nodes('event') as q(n)
WHERE n.value('(@timestamp)[1]', 'datetime') >= DATEADD(HOUR, -4, GETDATE())
ORDER BY timestamp DESC;
```

**提示** AWS 在 Xevents 方面做得很好，尽可能地模仿了 SQL Server。



## 15 其他云提供商审计选项

在 Azure SQL 数据库和托管实例中查询 Xevent 数据更为困难，因为你必须知道文件的确切名称。无法使用 `*.xel` 来一次性查询所有文件。

来自清单 15-7 w 的结果将类似于图 15-23. 中的结果。

**图 15-23.** 扩展事件查询结果示例

**注意** 有关 RDS 中扩展事件的更多信息，请访问 [`aws.amazon.com/blogs/database/set-up-extended-events-in-amazon-rds-for-sql-server/`](https://aws.amazon.com/blogs/database/set-up-extended-events-in-amazon-rds-for-sql-server/)

### 审计 Google Cloud SQL 数据库

Google Cloud 提供了一项名为 Cloud SQL 的服务，允许你设置完全托管的 SQL Server。此服务不支持 SQL Server 审计或扩展事件。

**注意** 有关 Cloud SQL 的更多信息，请访问 [`cloud.google.com/sql/docs/features#sqlserver`](https://cloud.google.com/sql/docs/features#sqlserver)

可以通过 Google Cloud 门户审计 Cloud SQL，但这无法模拟 `SQL Server Audit` 或 `Extended events` 的功能。它只会让你访问 SQL Server 日志。

因此，如果你希望拥有任何审计功能，我建议在 Google Cloud 中使用虚拟机。

请参阅附录 [A](https://doi.org/10.1007/978-1-4842-8634-0_16) 以获取审计选项的比较和概述。它将提供用例、优点和缺点，并会链接回各章节以获取更多信息。

## 附录 A

### 数据库审计选项比较

本章快速回顾了审计选项，并为每个审计选项提供了用例、优点和缺点，并引用了具体章节以获取更多信息。

#### 审计选项

*   **SQL Server 审计** – 这项内置的 SQL Server 审计功能可以通过 `SQL Server Management Studio` 或 SQL 脚本来设置审计。此功能便于查看 SQL Server 上发生的变化。要使 SQL Server 审计正常工作，根据你要审计的内容，你需要两到三个组件。服务器审计规范通常用于审计服务器级更改和/或所有数据库。数据库审计规范则适用于审计一个数据库或一个数据库中的部分活动。
*   **扩展事件** – 这项内置的 SQL Server 审计功能可以通过 `SQL Server Management Studio` 或 SQL 脚本来设置审计。它的审计功能不如 `SQL Server Audit` 细致；因此，如果你希望审计用户或数据库的部分活动，那么最好使用 `SQL Server Audit`。要使扩展事件正常工作，你需要设置一个会话。这是唯一必需的部分，不像 `SQL Server Audit` 需要两到三个部分。
*   **变更数据捕获** – 此功能使用 `SQL Server Agent` 跟踪对表所做的 DML 更改。这使你能够查看对数据所做的更改，其细节以易于使用的格式呈现。这对于提取、转换和加载（ETL）应用程序或过程特别有用。
*   **变更跟踪** – 此功能是一种跟踪 DML 更改的轻量级方法。这通常被应用程序用于查询数据库更改。
*   **C2 和通用标准合规性** – 此功能是

© Josephine Bush 2022

J. Bush, *Practical Database Auditing for Microsoft SQL Server and Azure SQL*, [`doi.org/10.1007/978-1-4842-8634-0_16`](https://doi.org/10.1007/978-1-4842-8634-0_16#DOI)



### 附录 A：数据库审计选项比较

国际公认在审计时遵循特定安全准则。此类审计是对数据库服务器上所有活动进行全面的日志记录。如果没有审计员要求你开启此功能，请保持关闭。它可能对服务器性能产生重大影响。

*   **成功和失败登录审计** – `SQL Server 审计`和扩展事件都允许你审计成功和失败的登录。你将需要一个审计来存储审计数据，以及一个服务器审计规范来收集登录信息。

*   **触发器**
    *   可以设置**服务器触发器**来阻止人员在特定时间更改数据库或登录。
    *   可以设置 **DDL 触发器**来阻止某些操作的发生，例如 `CREATE`、`ALTER`、`DROP`、`GRANT`、`DENY` 或 `REVOKE`。它们可以响应这些操作执行另一个操作，或者记录这些操作。
    *   可以设置 **DML 触发器**来阻止或响应 `INSERT`、`UPDATE` 或 `DELETE` 语句执行另一个操作。

*   **云审计**
    *   **Azure SQL 数据库** – 通过 Azure 门户使用 `SQL Server 审计`功能或使用扩展事件。
    *   **Azure 托管实例** – 通过 Azure 门户的诊断设置、`SQL Server 审计`或扩展事件。
    *   **AWS RDS** – 使用 `SQL Server 审计`和扩展事件。
    *   **Google Cloud** – 仅在 VM 上。

**注意：** 任何审计技术都可能使你的服务器过载，具体取决于你用它审计的内容量。最好的情况是你无法筛选所有数据，最坏的情况是你的生产服务器过载。始终牢记根据你的具体情况执行最少量的必要审计。

#### 审计选择的优缺点

**优缺点**

| **技术** | **优点** | **缺点** |
| :--- | :--- | :--- |
| **SQL Server 审计** | 易于捕获非常具体的审计事件。<br>无需解析 XML。 | 比 XEvents 设置更复杂。<br>没有模板指导。 |
| **扩展事件 (XEvents)** | 通过模板易于上手。<br>如果你用过 SQL 跟踪或 Profiler 会感觉熟悉。 | 需要解析 XML 来查询结果。 |
| **变更数据捕获 (CDC)** | 对 ETL 流程有用。<br>允许你查看数据变更前后的状态。 | 在大型、繁忙的表上，性能和存储可能成为问题。 |
| **变更跟踪** | 对应用程序跟踪数据库表上的变更很有用。 | 在大型、繁忙的表上，性能可能成为问题。<br>数据库需要快照隔离级别以保持一致性。<br>表需要有主键。<br>无法查看变更发生的次数或每次变更的值。 |
| **C2 和通用准则合规性** | 国际公认在审计时遵循特定安全准则。 | 它可能对服务器性能产生**非常重大**的影响。 |
| **Azure SQL 数据库审计** | 在 Azure 门户中易于设置。<br>通过 PowerShell 易于集中到 Log Analytics。 | 默认审计策略审计所有内容，但你可以修改它。<br>需要在 Kusto 中查询审计数据。 |
| **Azure SQL 数据库扩展事件** | 工作方式与 SQL Server 扩展事件非常相似。 | 需要使用存储账户。<br>难以查询多个 `.xel` 文件。 |
| **Azure SQL 托管实例诊断设置** | 在 Azure 门户中易于设置。<br>易于集中到 Log Analytics。 | 需要在 Kusto 中查询审计数据。 |
| **Azure SQL 托管实例扩展事件** | 工作方式与 SQL Server 扩展事件完全相同。 | 与 SQL Server XEvents 相同的缺点。<br>难以查询多个 `.xel` 文件。 |
| **Azure SQL 托管实例 SQL Server 审计** | 工作方式与 SQL Server 审计完全相同。 | 难以查询多个 `.sqlaudit` 文件。 |
| **Google Cloud SQL Server 审计** | 只能在安装了 SQL Server 的 VM 上进行审计。 |  |

## SQL Server 审计 vs. 扩展事件

下表对比了 `SQL Server 审计`与扩展事件，以帮助你决定哪种最适合你的用例。

| **功能** | **扩展事件** | **SQL Server 审计** |
| :--- | :--- | :--- |
| **通过 GUI 或脚本设置** | 是 | 是 |
| **通过 GUI 或脚本查询** | 是 | 是 |
| **在 GUI 或脚本中删除，它** | 否，XEL 文件会保留在磁盘上，如果... | |



