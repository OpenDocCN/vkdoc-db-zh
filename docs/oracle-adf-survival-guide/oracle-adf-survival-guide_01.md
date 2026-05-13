# Oracle ADF 生存指南：精通应用开发框架
*Sten Vesterli*

![A437725_1_En_BookFrontmatter_Figa_HTML.png](img/A437725_1_En_BookFrontmatter_Figa_HTML.png)

本书作者引用的源代码或其他补充材料，读者均可通过本书在 GitHub 上的产品页面获取：[`www.apress.com/9781484228197`](http://www.apress.com/9781484228197)。更详细信息请访问：[`http://www.apress.com/source-code`](http://www.apress.com/source-code)。

ISBN 978-1-4842-2819-7  
e-ISBN 978-1-4842-2820-3  
[`doi.org/10.1007/978-1-4842-2820-3`](https://doi.org/10.1007/978-1-4842-2820-3)  
美国国会图书馆控制号：2017952558  
© Sten Vesterli 2017

本作品受版权保护。出版商保留所有权利，无论是材料的全部还是部分，特别是翻译、转载、插图重用、朗诵、广播、微缩胶片或任何其他物理方式的复制，以及信息存储与检索、电子改编、计算机软件，或目前已知或未来开发的类似或不同方法。

本书中可能出现商标名称、徽标和图片。我们仅在编辑意义上使用这些名称、徽标和图片，旨在使商标所有者受益，并无侵权意图。

本书中使用的商品名称、商标、服务标志和类似术语，即使未特别标识，也不应被视为对其是否受专有权利的表达。

尽管本书中的建议和信息在出版时被认为是真实和准确的，但作者、编辑和出版商均不对可能存在的任何错误或遗漏承担法律责任。出版商对本出版物所含材料不作任何明示或暗示的保证。

本书使用无酸纸印刷。

由 Springer Science+Business Media New York 向全球图书贸易发行。  
地址：233 Spring Street, 6th Floor, New York, NY 10013  
电话：1-800-SPRINGER  
传真：(201) 348-4505  
电子邮件：orders-ny@springer-sbm.com  
网址：www.springeronline.com

Apress Media, LLC 是一家加利福尼亚州有限责任公司，其唯一成员（所有者）是 Springer Science + Business Media Finance Inc (SSBM Finance Inc)。SSBM Finance Inc 是一家特拉华州公司。

## 为解决问题而不仅仅是编写代码的开发者们

本书将向您介绍 Oracle ADF，该工具被众多 Oracle 客户所使用。ADF 是一款高生产力的工具，可高效构建用户友好的企业应用程序。这些 ADF 应用程序中，有些是新建的，有些用于替代 Oracle Forms 等遗留技术，有些则是 Oracle 软件即服务应用程序套件的扩展。

ADF 开发人员需求旺盛，本书包含了成为一名称职的 ADF 开发人员所需的全部信息。

在第 1 章中，您将学习如何使用 `JDeveloper` 中的声明式工具构建 ADF 应用程序。`ADF` 的强大功能让您无需编写任何代码即可交付大量无缺陷的功能。

第 2 章解释了 `ADF` 架构以及如何使用 `ADF` 特性来构建各种规模的、可维护的应用程序。

在第 3 章中，您将学习如何控制应用程序的视觉外观，以及如何实现所需的精确布局。

第 4 章和第 5 章解释了如何向应用程序添加业务逻辑，包括在业务组件层和用户界面层。

在第 6 章中，您将学习如何使用 `ADF` 日志记录和调试，第 7 章作为本书的结尾，介绍了如何建立高效的 `ADF` 开发工作流程。

### 致谢

我要感谢 Oracle 提供了 Oracle ADF，这是快速高效地生产用户友好型全栈应用程序的最佳工具。我欣赏 Oracle 免费为我们提供了 `ADF Essentials` 这项强大的技术，并且 `ADF` 正在持续演进。凭借更新、更酷的可视化功能以及将 `ADF` 业务逻辑发布为 REST Web 服务的能力，`ADF` 继续作为众多 Oracle 客户应用程序架构的核心。

Oracle 还通过其 Oracle Applications User Experience 团队分享了他们使用 `ADF` 构建 Oracle Cloud Applications 的经验。如果您正在构建 `ADF` 应用程序，可以在 [`www.oracle.com/usableapps`](http://www.oracle.com/usableapps) 找到用户体验设计模式和美观、用户友好的示例应用程序及其完整源代码。我鼓励您访问。

我还要感谢 Oracle ACE Michael Rosenblum，在旧金山湾畔晚餐时的讨论促成了这本《ADF 生存指南》；感谢 Apress 的 Jonathan Gennick 对这个想法的信念；感谢 Apress 的 Jill Balzano 引导项目完成；以及 Oracle ACE Director Luc Bors 提供的大量宝贵审阅意见。

最后，感谢我出色的妻子给予的爱与支持，并接受我们日历上又一批标记为“书籍截稿日”的周末。

