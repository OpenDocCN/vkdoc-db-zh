# 第 1 章 集成服务导论

## 新功能

*   **分析功能**可提供元数据以定位对象依赖关系——例如，某个包依赖于哪些表。此新工具为解决性能问题或依赖关系问题，或在主动定位依赖关系时提供了有用信息。我们将在第 12 章演示影响分析和数据沿袭分析。
*   **改进的合并和合并联接转换**：SQL Server 12 SSIS 中的 `Merge` 和 `Merge Join` 转换通过提供更好的内存使用内部控制得到了改进。我们将在第 6 章介绍 `Merge` 和 `Merge Join` 转换的新特性。
*   **数据更正组件**：`Data Correction` 转换提供了一个有助于提高数据质量的工具。我们将在第 7 章讨论这个新的转换。
*   **自定义数据流组件改进**：SQL Server 12 SSIS 包含多项改进，使开发人员能更轻松地创建支持多输入的自定义数据流组件。我们将在第 21 章关于自定义数据流组件的讨论中探索这些新功能。
*   **源和目标助手**：新的 `Source` 和 `Destination` 助手旨在指导您完成创建源或目标的步骤。我们将在第 6 章讨论这些新助手。
*   **简化的数据查看器**：SQL Server 12 SSIS 中的数据查看器已简化，使其更易于使用。我们将在第 8 章演示数据查看器。

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 我们喜爱的人物与资源

自 SSIS 问世以来，已有许多 SSIS 专家和资源。以下是我们推荐的一些资源，堪称网络上 SSIS 的“精选”列表：

Andy Leonard 是一位 SSIS 专家和 SQL Server 最有价值专家（`MVP`），从第一天起就是一位埋头苦干、亲力亲为的 SSIS 开发人员。事实上，Andy 是原版《*Professional SQL Server 2005 Integration Services*》（Wrox，2006 年）——SQL Server 2005 SSIS 的黄金标准——的合著作者之一。Andy 的博客位于 `http://sqlblog.com/blogs/andy_leonard/default.aspx`。

Jamie Thomson 很可能是最活跃的 SSIS 专家之一。他已就数百个 SSIS 主题撰写博客，是一位 SQL Server `MVP` 和最初的 *SSIS Junkie*。您可以在 `http://sqlblog.com/blogs/jamie_thomson/` 阅读他的最新文章，或在 `http://consultingblogs.emc.com/jamiethomson` 上回顾您的 SSIS Junkie 经典读物。

Brian Knight 是 Pragmatic Works 的创始人和 SQL Server `MVP`，是关于所有 BI 和所有 SSIS 事物的知名作家和培训师。在 `www.bidn.com/brians/BrianKnight` 可以找到 Brian。

联机丛书（`BOL`）是所有 SQL Server 相关内容的圣典，其中也包括 SSIS。当您需要针对特定 SQL Server 或 SSIS 问题的答案时，您的搜索极有可能最终指向 `BOL`。考虑到这一点，我们喜欢跳过中间环节，直接从 `http://msdn.microsoft.com/en-us/library/ms130214(v=SQL.110).aspx` 开始搜索。

SQLServerCentral.com（`SSC`）由一群硬核 DBA 创立，其中包括穿着标志性的夏威夷衬衫、戴着牛仔帽的传奇人物 Steve Jones，他是一位 SQL Server `MVP`。Steve 不断更新 `SSC`，提供大量基于社区的内容，涵盖包括 SSIS 在内的广泛主题。当您想查看社区生成的最佳内容时，请访问 `www.sqlservercentral.com`。

SSIS 团队博客由微软自己的 Masson 维护，位于 `http://blogs.msdn.com/b/mattm`。查看此博客可获取 SSIS 更新、补丁和内部技巧与窍门的最新、最棒信息。

Allan Mitchell 和 Darren Green 是来自大洋彼岸的 SSIS 专家和 SQL `MVP`，他们在 `www.sqlis.com` 分享他们的专业知识。

CodePlex 是一个托管开源项目的微软网站。从 `www.codeplex.com` 您可以下载各种开源项目，包括 AdventureWorks 系列示例数据库、开源 SSIS 自定义组件、完整的基于 SSIS 的 ETL 框架、ssisUnit（SSIS...



单元测试框架）和`BIDS Helper`。这是一个值得你去浏览的网站。

SQL Server 专业人士协会（PASS）是一个面向所有 SQL Server 开发人员、数据库管理员和商业智能专业人士的专业组织。会员资格是免费的，其福利包括访问《*SQL Server 标准*》杂志。访问 PASS 网站 `www.sqlpass.org` 以获取更多信息。

[www.it-ebooks.info](http://www.it-ebooks.info/)

