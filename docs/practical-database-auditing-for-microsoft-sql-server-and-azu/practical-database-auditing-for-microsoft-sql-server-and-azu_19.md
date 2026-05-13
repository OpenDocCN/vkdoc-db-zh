# 第 5 章 通过 SQL 脚本实现 SQL Server 审计

要让 `SQL Server` `审计` 功能运作起来，你需要两个或三个组件，具体取决于你想要 `审计` 的内容。第 3 章“什么是 SQL Server 审计？”介绍了这些组件各自是什么，以及与它们相关的 `审计` 类别和组。



在本章中，你将学习如何使用 SQL 脚本设置审计。第 [4](https://doi.org/10.1007/978-1-4842-8634-0_4) 章“通过 GUI 实现 SQL Server 审计”介绍了如何在 SSMS 中通过 GUI 设置 SQL Server 审计。在本章中，你将学习如何通过脚本设置 SQL Server 审计。

## 脚本化现有规范

学习如何编写审计脚本的一个简单方法是将现有审计脚本化。你可以右键单击在第 [4 章“通过 GUI 实现 SQL Server 审计,”](https://doi.org/10.1007/978-1-4842-8634-0_4) 中创建的任何规范，选择 **脚本化审计为**，然后选择 **CREATE 到**，再选择 **新建查询编辑器窗口**，如图 5-1 所示。

***图 5-1.** 脚本化现有审计*

© Josephine Bush 2022
J. Bush, *Practical Database Auditing for Microsoft SQL Server and Azure SQL*, [`doi.org/10.1007/978-1-4842-8634-0_5`](https://doi.org/10.1007/978-1-4842-8634-0_5#DOI)

第 5 章 通过 SQL 脚本实现 SQL Server 审计

这将把你的审计的脚本版本放入 SSMS 的一个选项卡中。本章将介绍这些脚本，以便你能更好地理解如何创建、修改和删除它们。

#### 设置审计

快速回顾一下，要使 SQL Server 审计正常工作，根据你想要审计的内容，你需要两样或三样东西：

*   `你必须创建一个审计。` 这将决定审计数据的存储位置、保留多少审计数据以及与审计相关的其他几个设置。
*   `每个数据库还需要一个服务器和/或一个数据库审计规范来收集审计数据。` 服务器和数据库审计规范必须关联到一个审计。每个审计可以有一个服务器审计规范和一个数据库审计规范。这些服务器和数据库审计规范彼此不依赖。服务器审计规范通常适用于审计服务器级更改和/或同时审计所有数据库。数据库审计规范则适用于审计单个数据库或单个数据库中的部分活动。

清单 5-1 展示了如何通过脚本创建审计。

***清单 5-1.*** 创建审计

```sql
USE [master];

CREATE SERVER AUDIT [AuditSpecification]
TO FILE
(   FILEPATH = N'e:\audits\'
    ,MAXSIZE = 50 MB
    ,MAX_FILES = 4
    ,RESERVE_DISK_SPACE = OFF
) WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

ALTER SERVER AUDIT [AuditSpecification] WITH (STATE = ON);
```

第 5 章 通过 SQL 脚本实现 SQL Server 审计

让我们看一下清单 5-1 脚本的各个部分：

*   `数据库上下文` – `USE [master];`
    所有审计都在 master 数据库中创建。
*   `审计名称` – `CREATE SERVER AUDIT [AuditSpecification]`
    我倾向于将其命名为 `AuditSpecification` 或 `AuditSpecification_servername`。这完全取决于你希望名称有多具描述性。不过，你可以创建多个审计，因此如果你计划添加更多审计，使用更具描述性的名称可能更有意义。
*   `审计目标`
    *   `TO FILE` – 我总是选择写入文件，因为这对我来说最简单。我没有很多审计限制。是的，审计员想知道发生了什么，但他们不认为我们会偷偷进去删除审计文件而不报告审计数据。
    *   `TO APPLICATION_LOG` – 如果你有像 Splunk 这样读取所有日志并将它们集中存储的应用程序，我可以考虑将审计数据存储在应用程序日志中。与 `TO FILE` 相比，这个选项可配置的选项更少。下一节将提供一个示例。
    *   `TO SECURITY_LOG` – 这比写入应用程序日志有更多限制，但概念相同。这是存储审计数据最安全的选择，因为它最不可能被他人篡改。与 `TO FILE` 相比，这个选项可配置的选项更少。下一节将提供一个示例。
*   `队列延迟` – `QUEUE_DELAY = 1000`



