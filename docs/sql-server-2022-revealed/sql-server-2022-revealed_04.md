# 引言

与*《SQL Server 2019 揭秘》*类似，本书的手稿也经历了大量修改。2021 年底和 2022 年，我又踏上了旅途，许多章节是在拉斯维加斯（多次）、奥兰多、伦敦（包括伦敦地铁）、南卡罗来纳州的查尔斯顿（我儿子 Troy 居住的地方）、南卡罗来纳州的格林维尔、达拉斯（我儿子 Ryan 居住的地方）、德克萨斯州的普尔维尔（我妻子家人居住的地方）、奥斯汀、科罗拉多州的杰纳西（Ginger 和我有一个小的山间居所）、佛罗里达州的西耶斯塔岛（感谢 Tom 和 Janet Grubish）、华盛顿州的雷德蒙德（多次）、芝加哥和亚特兰大等地撰写或润色的。这包括许多次在酒店、飞机、火车甚至公路旅行中 Ginger 开车时，我在前排座位上写稿。但我最高效地完成章节或进行编辑的地方，始终是在德克萨斯州北里奇兰山我家中那有限的空间里。

本书是我将从 SQL Server 2022 作为 Dallas 项目开始，直到产品正式发布期间，我所学习、吸收、测试并绞尽脑汁深入研究的所有内容，完整地传授给你们。能做这件事是我的荣幸和喜悦。所有那些深夜和周末的学习、测试、查阅源代码、与产品经理和开发者交谈，以及基于我 29 年经验构思的各种场景，都包含在这本书中。我试图将高层次的“是什么和为什么”与深入的“工作原理”结合起来，以便您能从各个角度学习 SQL Server 2022。仅书的页数就证明了这一目标。我还尝试在几乎每一章都提供了深入的示例。我希望您不仅阅读 SQL Server 2022 为何特别，还能亲自尝试，亲眼看到它的实际效果。我想您也会喜欢几乎每一章中我们工程团队关于他们为何认为 SQL Server 2022 是一个重要版本的观点引述。

如果您喜欢了解 SQL Server 2022 的历史和“幕后故事”，请从第 1 章开始，您还将洞察 SQL Server 2022 的整体故事。如果您想了解安装 SQL Server 2022 与之前版本的差异，包括哪些内容已被移除以及新的 Azure 连接选项，请参阅第 2 章。

所有其他章节可以按顺序阅读，也可以作为独立主题单独阅读。如果您想深入了解 SQL Server 如何与 Azure 连接，第 3 章是一个完整的资源。本章包含大量屏幕截图，向您展示完整的 Azure 体验。即使您没有 Azure 订阅，也可以看看我们如何连接到 Azure 以实现灾难恢复（DR）、分析和安全。

也许您想直接深入引擎内部。内置查询智能非常广泛，我需要用两章才能涵盖全部内容。第 4 章和第 5 章包含了非常有趣的练习，您可以使用评估版或开发人员版在自己的笔记本电脑上独立尝试。



第 6 章专注于“引擎核心”，相信你会喜欢我们在 `SQL Server 2022` 中引入的 25 多项主要功能，包括用于 `SQL Server` 的 Ledger、全自动化的 `tempdb` 以及包含的可用性组。

如果你听说过 `S3` 对象存储但不确定如何使用，那么你会喜欢第 7 章。我将展示如何使用我们的 `Polybase` 技术从 `SQL Server` 访问 `parquet` 文件和 Delta 表。或者，如果你正在寻找存放备份的新位置，我也会演示如何将原生 `SQL` 备份备份到任何兼容 `S3` 的存储提供商，并从那里进行还原。

我认为许多人觉得我们已经遗忘了 `Transact-SQL (T-SQL)` 语言。在第 8 章中，看到我们倾注到 `SQL Server 2022` 中的全新 `T-SQL` 函数和语言增强功能，你可能会感到惊喜。

`SQL Server 2022` 针对 `Linux` 的新功能并不多，但我想确保你了解其基础。在第 9 章，我将展示在 `Linux`、容器和 `Kubernetes` 上部署和使用 `SQL Server` 的基础知识。

我相信 `Azure` 是 `SQL Server` 的最佳云平台，因此第 10 章是一段引导你学习如何在 `Azure` 虚拟机上部署、优化和管理 `SQL Server 2022` 的旅程。

最后，我希望用一种方式来收尾本书，阐述 `SQL Server` 如何在任何你需要的地方成为一股力量。因此，第 11 章讲述了 `SQL Server` 如何以你从未梦想过的方式实现“从边缘到云端”的存在，这是一个关于一致性与灵活性的故事。

如果你以前喜欢读我的书，或者这是第一次接触，我希望你全方位享受这本书。我以非常对话式的风格写作，许多人告诉我，这有助于他们“想象我正在对他们讲话”，就像他们在各种现场和在线活动中看到我的演示一样。我也希望为你提供资源，因此你会在全书中看到大量的在线参考文献，供你深入探究。

每当撰写这样的书籍时，你总希望读者获得最新的信息。因此，请通过 `https://aka.ms/sql2022bookexamples` 获取本书的最新示例和勘误。或者，查看我在 `https://aka.ms/sql2022workshop` 的免费研讨会材料。

我乐于听取读者的意见，所以你可以在亚马逊上为本书评分，或者直接发送邮件至 `bobward@microsoft.com`。我总是想知道你对本书的看法，或者你是否想了解更多关于 `SQL Server 2022` 的信息。

> Bob Ward
>
> 北德克萨斯州，理查德兰山市
>
> 2022 年 9 月

致谢

首先，我要感谢上帝赐下他独生的儿子耶稣，使我的罪得赦免，让我沉浸于圣灵之中，并赐予永生的应许。若无我的信仰，我将一无所有。

我要深深感谢我的妻子 Ginger，感谢她多年来对我的爱与奉献，尤其是在本书写作期间。你耐心地允许我在疯狂的时间和情境下工作，甚至占用了一些假期时间，才让这本书得以问世。我还要感谢我的儿子 Troy 和 Ryan，以及 Ryan 的妻子 Blair，我的儿媳。看到你们所有人成长并在所做的一切中展现恩典与真理，你们给了我未来的希望。

这是我在 `Apress` 出版的第四本书，我要亲自感谢 `Apress` 的 Jonathan Gennick。当许多出版公司拒绝我时，Jonathan 给了我首次著书的机会。谢谢你，Jonathan，感谢你一直以来的支持，让我可以“按自己的方式写作”。我还要感谢 `Apress` 的 Jill Balzano，我从未与她本人谋面，但尽管我们总是面对疯狂的截止日期，她始终如此友善和专业。

我邀请 Erin Stellato 担任我的技术审校，因为她是 `SQL` 领域最资深的专家之一，也是一位了不起的人。事实证明，她在本书写作期间加入 `Microsoft` 是一种福气，因为我们可以讨论机密信息。Erin，感谢你的细致入微和出色态度，我们所有人都在最后阶段给你施加了巨大压力来审阅如此多的章节。

`Microsoft` 有许多人支持我完成本书的工作。但首先，我要感谢 Joe Sack、Pedro Lopes 和 James Rowland-Jones，他们虽然已不在 `Microsoft` 工作，但在帮助我构建本书中呈现的 `SQL Server 2022` 故事方面发挥了巨大作用。

在 `Microsoft`，我要感谢支持我所有努力的领导们，包括 Rohan Kumar、Peter Carlin、Asad Khan 和 Sanjay Mishra。本书真正的英雄是 `Microsoft` 工程团队中的所有人，他们不厌其烦地回答我所有问题，审阅复杂主题，并为我提供了一些精彩的引述。这份名单很长，但我必须感谢每一位提供帮助的人，因为他们每个人都以自己的方式让整本书和产品变得有价值。感谢：Ajay Jagannathan, David Pless, Kevin Farlee, Derek Wilson, Kate Smith, Kendal Van Dyke, Sasha Nosov, Travis Wright, Hanuma Kodavalla, Naveen Prakash, Joachim Hammer, Chuck Heinzelman, Mine Tokus, Hugo Queiroz, Tim Chen, Dani Ljepava, Mladen Andzic, Andreas Wolter, Srdan Bozovic, Vlad Rodriguez, Mirek Sztajno, Dylan Gray, Devin Rider, Robert Dorr, Peter Byrne, Jack Li, Mike Ray, Van To, Lee Woods, Sarah Kaufman, Conor Cunningham, Tejas Shah, Amit Khandelwal, Pam Lahoud, Purvi Shah, Aditya Badramraju, Taryn Pratt, Lani O'Brien, Nancy McGrory, Ebru Ersan, Umachandar Jayachandran (UC), Nicholas Simmons, Sharanya Bhat, Dilan Galindo Reyna, Ryan Stonecipher, Ravinder Vuppula, Thierry Fevrier, Aashna Bafna, Panagiotis Antonopoulos, Pieter Vanhove, Ankit Mahajan, Swaroop Moida, Fang Hou, Balmukund Lakhani, Parag Paul, Jakub Szymaszek, Yuqing Li, Perry Skountrianos, Pratim Dasgupta, Milos Vucic, Nikolas Ogg, David Liao, Chandrashekhar Kadiam, and Brian Carrig.

我还要感谢工程团队中一些不知疲倦地帮助我筹办研讨会和活动的同事。谢谢你们，Marisa Mathews 和 Rie Merritt。你们在幕后辛勤工作，无人知晓你们付出的努力。特别感谢我的同事 Buck Woody。你是一位亲爱的朋友，也是我认识的最有才华的人之一。你总是让出差旅行成为一次有趣的冒险。

同样感谢 `Microsoft Mechanics` 团队帮助我们展示 `SQL` 的卓越技术。谢谢你们，Jeremy Chapman 和 Celine Allee。参加你们的节目总是一种特权和有趣的体验。

我还要感谢我们市场团队的成员，他们在 `SQL Server 2022` 的发布期间是出色的合作伙伴，包括 Matthew Burrows, Miwa Monji, Eric Hudson, Sonya Waitman, Guy Schoonmaker（感谢你的幻灯片制作技巧），以及 Emilija Dufresne。

感谢大家，即使是那些最小的邮件回复，也帮助了本书的完成。

关于作者与技术审校  关于作者  关于技术审校

