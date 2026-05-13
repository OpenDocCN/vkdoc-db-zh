# 盖伊·哈里森 下一代数据库 NoSQL、NewSQL 与大数据

![](img/A978-1-4842-1329-2_BookFrontmatter_Figa_HTML.png)

作者在本书中引用的任何源代码或其他补充材料，读者均可访问 [`www.apress.com`](http://www.apress.com/) 获取。有关如何查找本书源代码的详细信息，请访问 [`www.apress.com/source-code/`](http://www.apress.com/source-code/)。

ISBN 978-1-4842-1330-8
e-ISBN 978-1-4842-1329-2
DOI 10.1007/978-1-4842-1329-2
© Apress 2015

## 下一代数据库

总编辑：Welmoed Spahr
主编：Jonathan Gennick
开发编辑：Douglas Pundick
技术审校：Stephane Faroult
编辑委员会：Steve Anglin, Pramila Balen, Louise Corrigan, Jim DeWolf, Jonathan Gennick, Robert Hutchinson, Celestin Suresh John, Michelle Lowman, James Markham, Susan McDermott, Matthew Moodie, Jeffrey Pepper, Douglas Pundick, Ben Renow-Clarke, Gwenan Spearing
协调编辑：Jill Balzano
文字编辑：Carole Berglie
排版：SPi Global
索引：SPi Global
美工：SPi Global
封面设计：Anna Ishchenko

有关翻译事宜，请发送电子邮件至 `rights@apress.com`，或访问 [`www.apress.com`](http://www.apress.com/)。Apress 和 friends of ED 的图书可批量购买用于学术、企业或推广用途。大部分图书也提供电子书版本和许可。更多信息，请参考我们的批量销售-电子书许可网页 [`www.apress.com/bulk-sales`](http://www.apress.com/bulk-sales)。

本作品受版权保护。出版者保留所有权利，无论涉及材料的全部或部分，具体包括翻译权、转载权、插图的重用权、朗诵权、广播权、以微缩胶片或其他物理方式的复制权，以及信息存储与检索、电子改编、计算机软件方面的传播权，或任何目前已知或今后开发的类似或不同方法的使用权。此法律保留的例外情况是用于评论或学术分析而摘录的简短片段，或专门为输入和执行到计算机系统中而提供的材料，仅供作品购买者专用。仅在出版者所在地版权法现行条款允许下，才可复制本出版物或其部分内容。使用许可必须始终从 Springer 获得。可通过版权许可中心的 RightsLink 获得使用许可。侵权行为将依据相应的版权法追究责任。本书中可能出现商标名称、标识和图像。我们并非在每次出现商标名称、标识和图像时都使用商标符号，而仅以编辑方式并为了商标所有者的利益使用这些名称、标识和图像，无侵犯商标之意。在本出版物中使用商品名称、商标、服务标志和类似术语，即使未特别标识，也不应被视为表达关于它们是否受专有权利约束的意见。尽管本书中的建议和信息在出版时被认为是真实准确的，但作者、编辑和出版商对可能出现的任何错误或遗漏不承担任何法律责任。出版商对本出版物包含的材料不作任何明示或暗示的保证。本书通过 Springer Science+Business Media New York 在全球图书贸易中发行，地址：233 Spring Street, 6th Floor, New York, NY 10013。电话：1-800-SPRINGER，传真：(201) 348-4505，电子邮件：orders-ny@springer-sbm.com，或访问 www.springer.com。Apress Media, LLC 是一家加利福尼亚州的有限责任公司，其唯一成员（所有者）是 Springer Science + Business Media Finance Inc (SSBM Finance Inc)。SSBM Finance Inc 是一家特拉华州的公司。

谨以此书纪念凯瑟琳·玛蕾·阿诺德（1981-2010）

## 致谢

我要感谢 Apress 所有为本书出版提供帮助的人，特别是主编 Jonathan Gennick、协调编辑 Jill Balzano 和开发编辑 Douglas Pundick。我尤其要感谢 Stéphane Faroult，他提供了出色的技术审校反馈。我很少能与 Stéphane 这样高水平的审校者合作，他的评论非常宝贵。

一如既往，感谢我的家人——Jenni, Chris, Kate, Mike 和 Willie——他们提供了承担这个项目所需的情感支持与理解。

本书谨以此纪念我们挚爱的侄女凯瑟琳·玛蕾·阿诺德，她于 2010 年去世。我遇见凯瑟琳时她五岁；第一次见到她时，她向我展示了罗尔德·达尔《蠢特夫妇》的魔力。我最后一次见到她时，她向我解释了现代 DNA 测序技术，并告诉我她拯救物种免于灭绝的抱负。她是那种你能期望遇到的最聪明、最有趣、最善良的人之一。1987 年我的第一本书出版时，她开玩笑地坚持要求我将书献给她，因此，将此书献给她的记忆是再合适不过的了。

盖伊·哈里森
澳大利亚墨尔本
2015 年 12 月


