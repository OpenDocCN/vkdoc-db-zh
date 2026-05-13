# 关于技术审阅者

**Frits Hoogland** 是一名 IT 专业人士，专门从事 Oracle 数据库性能和内核研究。Frits 经常在全球各地的会议上就 Oracle 技术主题发表演讲。2009 年，他获得了 Oracle 技术网络颁发的 Oracle ACE 奖，一年后成为 Oracle ACE Director。2010 年，他加入了 OakTable Network。除了发展他的 Oracle 专业知识外，Frits 还从事 MySQL、PostgreSQL 和现代操作系统方面的工作。Frits 目前在 Enkitec LP 工作。

**Randolf Geist** 从事 Oracle 软件工作已有 19 年，自 2000 年起担任自由数据库顾问。他主要关注性能相关问题，特别是帮助人们理解并释放 Oracle 成本基优化器（CBO）的强大功能——随时准备接受短期、临时的 Oracle 性能故障排除任务以及内部研讨会和咨询。

他在 `oracle-randolf.blogspot.com` 上撰写关于 CBO 相关问题的博客，定期为官方 OTN 数据库相关论坛做出贡献，并且也是《*Expert Oracle Practices*》一书的合著者，该书于 2010 年由 OakTable Press 出版，他负责撰写其中与性能相关的章节。

Randolf 是全球所有主要用户会议的常客，并且作为 Oracle University “名人研讨会”计划的一部分担任讲师，讲授“Oracle 高级性能故障排除”研讨会。Randolf 是 OakTable Network 和 Oracle ACE Director 计划的成员。

**Dominic Delmolino** 是 Agilex 的首席数据库技术专家。他专门从事 Oracle 和敏捷数据库部署方法的研究，是 OakTable Network 的成员。他喜欢研究和介绍高级数据库主题。

## 致谢

作者以感谢配偶开始一份致谢清单，这似乎是一种政治正确的做法。除非你是一位作者——或者是作者的配偶——否则你才会理解。写这本书花了我大约 1000 到 1500 个小时。考虑到我同时还保持着一份全职工作，你可能会开始理解这个项目对家人和朋友的生活造成的影响。除了生活中常见的压力之外，我们几个月前搬了家，Marianne 不得不承担大部分相关的工作！如果没有 Marianne 的支持，写这本书将是完全不可能的，我的感谢是发自内心的。

尽管我绝不愿将自己比作艾萨克·牛顿或其他使用过这个隐喻的伟大思想家，但我只是站在巨人肩膀上的一个侏儒。我的两个主要信息和灵感来源是 Jonathan Lewis 所著的《*Cost Based Oracle*》和 Christian Antognini 所著的《*Troubleshooting Oracle Performance*》。Jonathan 的书由 Apress 于 2008 年出版，Christian 书的第二版也于 2014 年由 Apress 出版。我很幸运与两位作者都私交甚笃。Jonathan 通过电子邮件往来，帮助澄清了本书中一两个主题上的我个人误解，Christian 则向我提供了他关于优化器转换的章节草稿，从而为我节省了无数的工作时间。

然而，写这本书的想法并非来自 Jonathan 或 Christian。事实上，它来自 Dan Tow，他是《*SQL Tuning*》的作者，该书由 O'Reilly 于 2003 年出版，但我在 2011 年才发现这本书。我将在引言部分再次提到 Dan 的书。



## 致谢

我得承认，我并不倾向于在业余时间大量浏览博客，但如果我说我的知识和灵感完全来自纸质书籍，那也是错误的。互联网是填补知识空白的绝佳资源，其中一个网站——`www.hellodba.com`（黄伟（Wei Huang）的中文网站）——为我提供了许多关于 Oracle 数据库中未公开提示的细节。

Gordon Mitchell 是我的一位客户，他第一个创造了“TSTATS”这个术语，意为“Tony 的统计数据”。但实际上，我并不是第一个或唯一一个构思出本书第 6 章和第 20 章所讨论方案的人。我必须特别提到 Adrian Billington (`http://www.oracle-developer.net/`)，他提出了一个关键思路：删除列统计信息中的最大值和最小值（但请参阅引言中关于在 12.1.0.1 版本上运行示例的说明）。

当然，我必须感谢 Tanel Põder 撰写了如此精彩的前言，感谢我的技术审阅者 Chris Dunscombe，他从读者角度为我一些更具争议性的章节提供了意见，还要感谢 Apress 的编辑们。要创作如此规模的作品而又完全不存在技术或语法错误是不可能的，但他们的努力极大地提高了本书的质量。谢谢你们所有人。

最后，当然，我必须感谢你们，我的读者，购买本书并赋予它生命力。对于仍然存在的任何错误，我负全部责任，但我真诚地希望你们阅读 *Expert Oracle SQL* 的体验既富有教育意义又令人愉快。

