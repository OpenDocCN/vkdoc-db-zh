# Oracle Exadata 入门指南

![image](img/FM-00.jpg)

David Fitzjarrell

Mary Mikell Spence

![image](img/FM-01.jpg)

**Oracle Exadata 入门指南**

版权所有 © 2013，作者：David Fitzjarrell, Mary Mikell Spence

本作品受版权法保护。出版者保留所有权利，无论其涉及材料的全部或部分，特别是翻译、重印、插图再利用、朗诵、广播、缩微胶片或其他任何物理形式的复制，以及信息存储与检索、电子改编、计算机软件或目前已知或未来开发的类似或相异方法进行的传播。不适用此法律保留条款的情形包括：为撰写评论或进行学术分析而摘录的简短片段，或专门为输入和执行到计算机系统中而提供的材料，且仅供该作品的购买者专用。复制本出版物或其部分内容，仅可依据出版者所在地的版权法现行版本的规定进行，且必须始终获得施普林格的许可。使用许可可通过 `版权结算中心` 的 `RightsLink` 服务获取。侵权行为将依据相应的版权法追究法律责任。

`ISBN-13` (平装本): `978-1-4302-6010-3`
`ISBN-13` (电子版): `978-1-4302-6011-0`

本书中可能出现商标名称、标识和图像。我们并非在每次出现商标名称、标识和图像时都使用商标符号，而仅以编辑方式并为商标所有者利益使用这些名称、标识和图像，并无侵犯商标之意图。

本书中对商品名称、商标、服务标志及类似术语的使用，即使未特别标识，亦不应被视为表达意见认为其不受专有权约束。

尽管本书中的建议和信息在出版时被认为是真实准确的，但作者、编辑或出版商均不对可能出现的任何错误或遗漏承担任何法律责任。出版商对本出版物所含材料不作任何明示或暗示的担保。

总裁兼出版人：Paul Manning
主编：Jonathan Gennick
策划编辑：Tom Welsh
技术评审：Arup Nanda

编辑委员会：Steve Anglin, Mark Beckner, Ewan Buckingham, Gary Cornell, Louise Corrigan, James DeWolf, Jonathan Gennick, Jonathan Hassell, Robert Hutchinson, Michelle Lowman, James Markham, Matthew Moodie, Jeff Olson, Jeffrey Pepper, Douglas Pundick, Ben Renow-Clarke, Dominic Shakeshaft, Gwenan Spearing, Matt Wade, Steve Weiss, Tom Welsh

协调编辑：Anamika Panchoo
文字编辑：Michael G. Laraque
排版：`SPi Global`
索引：`SPi Global`
美工：`SPi Global`
封面设计：Anna Ishchenko

本书通过 `Springer Science+Business Media New York`（地址：`233 Spring Street, 6th Floor, New York, NY 10013`，电话：1-800-SPRINGER，传真：`(201) 348-4505`，邮箱：`orders-ny@springer-sbm.com`，或访问网站 `www.springeronline.com`）面向全球图书贸易发行。Apress Media, LLC 是一家位于加利福尼亚州的有限责任公司，其唯一成员（所有者）是 `Springer Science + Business Media Finance Inc (SSBM Finance Inc)`。`SSBM Finance Inc` 是一家特拉华州公司。

有关翻译信息，请发送电子邮件至 `rights@apress.com`，或访问 `www.apress.com`。

`Apress` 和 friends of ED 图书可批量购买用于学术、企业或促销用途。大多数图书也提供电子书版本和许可。欲了解更多信息，请参考我们的批量销售-电子书许可专页 `www.apress.com/bulk-sales`。

作者在文中引用的任何源代码或其他补充材料，读者均可从 `www.apress.com` 获取。有关如何查找本书源代码的详细信息，请访问 `www.apress.com/source-code/`。




## 目录概览

关于作者

关于技术评审

致谢

前言

![image](img/sq1.jpg) 第 1 章：Exadata 基础知识

![image](img/sq1.jpg) 第 2 章：智能扫描与下推处理

![image](img/sq1.jpg) 第 3 章：存储索引

![image](img/sq1.jpg) 第 4 章：智能闪存缓存

![image](img/sq1.jpg) 第 5 章：并行查询

![image](img/sq1.jpg) 第 6 章：压缩

![image](img/sq1.jpg) 第 7 章：Exadata 单元等待事件

![image](img/sq1.jpg) 第 8 章：性能衡量

![image](img/sq1.jpg) 第 9 章：存储单元监控

![image](img/sq1.jpg) 第 10 章：监控 Exadata

![image](img/sq1.jpg) 第 11 章：存储重新配置

![image](img/sq1.jpg) 第 12 章：将数据库迁移至 Exadata

![image](img/sq1.jpg) 第 13 章：ERP 设置——实用指南

![image](img/sq1.jpg) 第 14 章：最终思考

索引

## 目录

关于作者

关于技术评审

致谢

前言

![image](img/sq1.jpg) 第 1 章：Exadata 基础知识

`什么是` Exadata？

可用配置

存储

智能闪存缓存

更多存储

须知事项

![image](img/sq1.jpg) 第 2 章：智能扫描与下推处理

智能扫描

执行计划与指标

智能扫描优化

列投影

谓词过滤

基础连接

函数下推

虚拟列

须知事项

![image](img/sq1.jpg) 第 3 章：存储索引

并非索引的“索引”

不要看这里

我使用了它

或者也许我没有

执行计划不知道

多多益善

NULL 值

需要“重建”

须知事项

![image](img/sq1.jpg) 第 4 章：智能闪存缓存

闪存一下

工作原理

让我们记录日志

它是一块磁盘

监控

存储服务器工具

数据库服务器工具

性能表现

须知事项

![image](img/sq1.jpg) 第 5 章：并行查询

进入队列



