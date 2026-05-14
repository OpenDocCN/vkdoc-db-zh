# 第 1 章 为何在 Linux 上运行 SQL Server？

话说回来，在超过 20 年的时间里，我已成为 SQL Server 在 Windows Server 上运行的专家，自以为永远不会在其它平台上见到它。

直到 2015 年的某一天，我和同事鲍勃·多尔收到了 SQL Server 首席开发工程师斯拉瓦·奥克斯的邮件，问我们觉得如果把 SQL Server 做到在 Linux 上运行会怎么样。当然，我不得不把邮件读了好几遍才明白他的提议。接下来的几个月我没怎么想这件事，直到 2015 年底我参加了一个微软内部活动。在这个活动上，令我惊讶的是，该项目最初的程序经理之一托比亚斯

© 鲍勃·沃德 2018

B. Ward, *Pro SQL Server on Linux*, `doi.org/10.1007/978-1-4842-4128-8_1`

特恩斯特罗姆向观众展示了在 Ubuntu 上使用`apt-get`部署 SQL Server，并在几分钟内连接运行查询。

坐回椅子后，我满脑子疑问。我们是怎么这么快做到的？将一个领先的数据库引擎移植到 Linux 上并在如此短的时间内让它运行起来，背后的架构是怎样的？但我也想知道为什么。SQL Server 产品当时在业界已取得巨大成功，客户都在 Windows Server 上运行它。

而这正是我们本书的起点。我将解释我们将 SQL Server 引入 Linux 平台的动机、我们实现这一目标的基本原理以及其核心能力。我在现有的 SQL Server 社区中以讲解 SQL Server 内部机制而闻名，因此在本章及全书中，我有时会情不自禁地带你“窥探幕后”，了解这项技术如何在 Linux 平台上运作。

#### 平台选择

2016 年 5 月，我们发布了 Microsoft SQL Server 2016，这是自 1993 年推出 Windows NT 版 4.2 以来，SQL Server 的第 11 个主要版本。这个版本功能强大，受到了 SQL Server 社区的积极评价。

在大家为发布此版本而努力的同时，工程团队已经在着手开发 SQL Server 2017，其关键核心功能将是 SQL Server 最终能在 Linux 和容器上获得支持。业界和世界已经知道我们将实现这一目标，因为在当年早些时候，微软云与企业事业部执行副总裁斯科特·格思里已在客户活动和我们的官方微软博客上宣布了此消息（参见[`blogs.microsoft.com/blog/2016/03/07/announcing-sql-server-on-linux`](https://blogs.microsoft.com/blog/2016/03/07/announcing-sql-server-on-linux)）。在这篇博客中，斯科特阐明了将 SQL Server 引入 Linux 的真正原因：“……将 SQL Server 引入 Linux 是我们让产品和创新成果惠及更广泛用户、并在他们所处之地满足其需求的另一种方式。” 我们将 SQL Server 引入 Linux 的决定，并非要背离 Windows Server。而是为了在 Windows 和 Linux 上都构建一个出色的数据平台。是为了给我们的客户提供平台的选择权。而我们并非在毫无数据和客户证据的情况下得出这一结论的。

首先，我们了解到，业界多年来出现的一个趋势是 Linux 正变得非常流行。现今的研究表明，大约 30%的企业服务器正在使用某种 Linux 发行版。Gartner 等独立公司的研究显示，Linux 服务器是增长最快的 OS 细分市场（www.gartner.com/doc/3731017/market-share-analysis-server-operating）。

其次，在微软内部，我们亲眼看到了证据。对于 Microsoft Azure 虚拟



