# 第 1 章 为什么要在 Linux 上使用 SQL Server？

与任何版本发布一样，我们会艰难地决定是否包含某些功能或增强改进——如果时间不是问题，我们很乐意实现它们。因此，以下功能未能包含在 SQL Server 2017 on Linux 的普遍可用版本中：

- 事务复制和合并复制
- 分布式事务（链接服务器查询或通过 `MSDTC` 的应用程序分布式事务）
- `Stretch Database`
- `Polybase`
- `Machine Learning Services`（但支持 `Native Scoring`）
- 系统扩展存储过程，如 `xp_cmdshell`

我只列出了未能进入 SQL Server 2017 普遍可用版本的主要功能。还有一些其他功能，您可以在我们的发布说明中阅读完整列表：[`https://docs.microsoft.com/sql/linux/sql-server-linux-release-notes#Unsupported`](https://docs.microsoft.com/sql/linux/sql-server-linux-release-notes#Unsupported)。

关于功能，我再补充一点说明。当前 `Windows Server` 上的 `SQL Server` 产品附带其他服务，如 `SQL Server Analysis Services (SSAS)` 和 `SQL Server Reporting Services (SSRS)`。这些通常被称为我们的 `Business Intelligence (BI) Services`（这还包括 `Master Data Services (MDS)` 和 `Data Quality Services (DQS)`）。这些服务尚未为 `SQL Server on Linux` 实现。请注意，这些服务能够连接并查询 `SQL Server on Linux`。

##### 我应该使用 Windows 还是 Linux？

本节最后，我来回答一个常被问到的问题：“我应该在 `Linux` 还是 `Windows Server` 上运行 `SQL Server`？” 答案是 `是`（希望此刻读者们能会心一笑）。关键在于，我们构建 `SQL Server on Linux` 是为了提供一种选择，而不一定是因为 `SQL Server` 在 `Windows Server` 上比在 `Linux` 上运行得更快或更好。如果当前 `SQL Server on Linux` 不支持的功能不影响您，请根据哪个操作系统平台最适合您、您的应用程序或您的公司来做出决定。我与之交谈的一些客户正在其组织内制定 `Linux` 标准，因此他们将为了保持一致性而做出这个选择，而现在 `SQL Server` 给了他们这个机会。对于其他人来说，他们习惯于 `Windows Server` 并喜欢那个平台，因此将继续在该操作系统上使用 `SQL Server`。对于一些人来说，`Linux` 是新开发者中非常流行的操作系统，因此他们将在开发阶段享受在 `SQL Server on Linux` 上构建应用程序的乐趣，然后在生产环境中可以选择使用 `Linux` 或 `Windows` 上的 `SQL Server`。

我在微软支持部门结识多年的好友 Robert Dorr（您可以在我们的联合博客 [`http://aka.ms/bobsql`](http://aka.ms/bobsql) 上找到我们俩）于 2016 年初加入了赫尔辛基开发团队。当我和他谈到这本书时，他回忆道，这总结了团队的经验和项目背后的原因。

“我发现我登上了一艘火箭飞船。整个生态系统中有很多新事物，也有很多事情要做。正如 Scott Konersmann 所描述的那样，我们就像在飞行中更换喷气式飞机的引擎，有时确实感觉如此。我们将 `Windows` 上数十年经过考验和调优的技术及经验带到 `Linux` 平台，我们不想移除功能。只是想给客户选择运行哪种‘引擎’的权利。”

#### 容器是新的虚拟机

我们构建 `SQL Server on Linux` 的另一个主要动机是提供支持



