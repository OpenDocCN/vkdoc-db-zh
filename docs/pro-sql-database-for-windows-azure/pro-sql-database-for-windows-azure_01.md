# 内容概览

前言 xvii

关于作者 xix

关于技术审阅者 xxi

致谢 xxiii

引言 xxv

第 1 章：SQL 数据库入门 1

第 2 章：设计考虑因素 23

第 3 章：安全 45

第 4 章：数据迁移与备份策略 67

第 5 章：使用 SQL 数据库进行编程 99

第 6 章：SQL 报表 125

第 7 章：SQL 数据同步 143

第 8 章：Windows Azure 与 ASP.NET 165

第 9 章：高性能设计 183

第 10 章：联合 207

第 11 章：性能调优 219

第 12 章：Windows Azure 移动服务 241

附录 A：SQL 数据库管理门户 257

附录 B：Windows Azure SQL 数据库快速参考 275

索引 283

v

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 引言

Windows Azure SQL Database，正式名称为 SQL Azure，大约在五年前出现。当时，人们对它知之甚少，但微软已经开始大量谈论 Azure 平台。大多数人认为 SQL Azure 是另一个 NoSQL 产品，而实际上它从来都不是。那时，它能处理的最大数据库是 1GB，并没有多少人认真对待它。从那时起，Windows Azure SQL Database 已经发展成为一个基于久经验证的 SQL Server 技术、企业就绪的 PaaS（平台即服务）产品。



云计算已不再是炒作的概念。如今，基于云的解决方案正逐渐成为常态，而非可有可无或边缘化的选择。Windows Azure 云平台（包括 Windows Azure SQL Database）带来的益处，使企业能够以较低的购置成本快速创建和扩展解决方案，同时提供高可用性和互操作性。SQL 开发人员和数据库管理员可以利用现有技能和知识，扩展其本地解决方案，并加快云开发速度。本书涵盖了核心的 Windows Azure SQL Database 概念、实践与方法，旨在为您的宝贵数据做好准备，迎接向云端及 Windows Azure SQL Database 的迁移之旅。

由于 Windows Azure SQL Database 更新迅速，本书撰写时讨论的部分服务可能仍处于预览版状态，到您阅读时可能会有所变化。不过，我们已尽力为您提供最新的信息。更新信息可查阅我们的博客、Windows Azure 博客 (http://blogs.msdn.com/b/windowsazure/) 以及至关重要的 Windows Azure 主页 (http://www.windowsazure.com/)，您可以在那里找到功能特性、定价、开发者信息等更多内容。

我们希望，在阅读本书后，您能对 Windows Azure SQL Database 有更好的理解和认识。无论您是刚刚起步还是“经验丰富的老手”，每一章都包含了一些场景和信息，我们希望它们在您设计和构建 Windows Azure 项目时能有所帮助和裨益。书中还提供了大量源代码，用于各章节中给出的示例。

## 本书面向的读者

*《Windows Azure 专业 SQL 数据库（第 2 版）》* 面向那些希望即时访问功能完备的 SQL Server 数据库环境，而无需费心处理和管理物理基础设施的开发人员和数据库管理员。

## 本书的组织结构

*《Windows Azure 专业 SQL 数据库》* 旨在引导您从对 SQL Database 几乎一无所知，到能够为其配置和部署以供生产应用程序使用。本书确实假设您对数据库和 SQL 有基本的了解。在此基础之上，本书将带您从入门开始，逐步深入到性能调优以及使用其他 Azure 数据服务。

xxv

[www.it-ebooks.info](http://www.it-ebooks.info/)

### ■ 引言

本书各章内容如下：

第 1 章 *，SQL Database 入门，* 帮助您在云端创建您的第一个数据库。

第 2 章 *，设计注意事项，* 讨论了在创建应用程序以连接云数据库（而非托管在您自己数据中心的数据库）时应该考虑的设计问题。

第 3 章 *，安全性，* 涵盖了在通过公共互联网访问数据的场景下，如何保护数据安全这一至关重要的议题。

第 4 章 *，数据迁移与备份策略，* 帮助您高效地将数据移入和移出 SQL Database。同时还涵盖了备份策略，以在数据库丢失或损坏时保护您。

第 5 章 *，使用 SQL Database 编程，* 涵盖了从本地应用程序与从 Azure 托管应用程序使用 SQL Database 的差异。

第 6 章 *，SQL Reporting，* 展示了如何创建基于云的报表。

第 7 章 *，SQL Data Sync，* 涵盖了多个 SQL Database 实例之间，以及云端 SQL Database 与您数据中心内 SQL Server 之间的复制。

第 8 章 *，Windows Azure 与 ASP.NET，* 提供了构建基于 Windows Azure 和 SQL Database 的 ASP.NET 应用程序的示例和指导。

第 9 章 *，高性能设计，* 涵盖了诸如分片、延迟加载、缓存等主题和特性，这些对于构建高性能应用程序非常重要。



