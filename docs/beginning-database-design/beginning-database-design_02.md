# 数据库设计入门：从新手到专家

版权所有 © 2012 Clare Churcher

本作品受版权法保护。出版商保留所有权利，无论是材料的全部或部分，特别是翻译、转载、插图重用、朗诵、广播、微缩胶片或其他任何物理方式的复制、信息存储和检索、电子改编、计算机软件改编，或当前已知或未来开发的类似或不同方法。此法律保留的例外情况是：为便于评论或学术分析而摘录的简短片段，或专门为输入和执行于计算机系统而提供的材料，仅供作品购买者专用。只有在遵守出版商所在地现行版本《版权法》规定的情况下，才允许复制本出版物或其部分内容，并且必须始终获得 Springer 的许可。使用许可可通过版权结算中心的 RightsLink 获取。违规行为将根据相应的版权法承担法律责任。

ISBN-13 (平装本): 978-1-4302-4209-3
ISBN-13 (电子版): 978-1-4302-4210-9

本书中可能出现商标名称、标识和图像。我们仅在编辑性质上，并为了商标所有者的利益而使用这些名称、标识和图像，无意侵犯商标权。

在本书中使用商品名称、商标、服务标志和类似术语，即使未明确标识，也不应被视为表达意见，即这些术语是否受专有权利约束。

尽管本书中的建议和信息在出版时被认为是真实和准确的，但作者、编辑或出版商均不对可能出现的任何错误或遗漏承担任何法律责任。出版商对本出版物所含材料不作任何明示或暗示的保证。

总裁兼出版人：Paul Manning
主编：Jonathan Gennick
技术评审：Stéphane Faroult
编辑委员会：Steve Anglin, Ewan Buckingham, Gary Cornell, Louise Corrigan, Morgan Ertel, Jonathan Gennick, Jonathan Hassell, Robert Hutchinson, Michelle Lowman, James Markham, Matthew Moodie, Jeff Olson, Jeffrey Pepper, Douglas Pundick, Ben Renow-Clarke, Dominic Shakeshaft, Gwenan Spearing, Matt Wade, Tom Welsh
协调编辑：Anita Castro
文字编辑：Chandra Clarke
排版：SPi Global
索引编制：SPi Global
美工：SPi Global
封面设计：Anna Ishchenko

本书在全球范围内通过 Springer Science+Business Media New York 发行，地址：233 Spring Street, 6th Floor, New York, NY 10013。电话：1-800-SPRINGER，传真：(201) 348-4505，电子邮件：orders-ny@springer-sbm.com，或访问 [www.springeronline.com](http://www.springeronline.com)。

有关翻译信息，请发送电子邮件至 rights@apress.com，或访问 [www.apress.com](http://www.apress.com)。

Apress 和 friends of ED 的图书可批量购买，用于学术、企业或促销用途。大多数书名也提供电子书版本和许可。更多信息，请参考我们的批量销售–电子书许可网页：[www.apress.com/bulk-sales](http://www.apress.com/bulk-sales)。

作者在文中引用的任何源代码或其他补充材料，读者均可访问 [www.apress.com](http://www.apress.com) 获取。有关如何查找您所购图书源代码的详细信息，请访问 [www.apress.com/source-code](http://www.apress.com/source-code)。

献给内维尔

## 内容概览



## 目录

## 前言
- 前言

### 关于作者
- 关于作者

## 关于技术审阅者
- 关于技术审阅者

### 致谢
- 致谢

### 引言
- 引言

## 第 1 章：可能出现的问题
![image](img/sq1.jpg) 第 1 章：可能出现的问题
- 关键词与类别处理不当
- 信息重复
- 为单一报表而设计
- 小结

## 第 2 章：开发过程导览
![image](img/sq1.jpg) 第 2 章：开发过程导览
- 初始问题陈述
- 分析与简单数据模型
- 类与对象
- 关系
- 深入分析：重新审视用例
- 设计
- 实现
- 输入用例的接口
- 输出用例的报表
- 小结

## 第 3 章：初始需求与用例
![image](img/sq1.jpg) 第 3 章：初始需求与用例
- 问题的真实视图与抽象视图
- 数据挖掘
- 任务自动化
- 用户做什么？
- 涉及哪些数据？
- 系统的目标是什么？
- 满足目标需要哪些数据？
- 输入用例有哪些？
- 第一个数据模型是什么？
- 输出用例有哪些？
- 关于用例的更多信息
- 参与者
- 异常与扩展
- 维护数据的用例
- 报表信息的用例
- 深入了解问题
- 我们推迟了什么？
- 价格变动
- 停售的餐品
- 特定餐品的数量

## 第 4 章：从数据模型中学习
![image](img/sq1.jpg) 第 4 章：从数据模型中学习

## 第 5 章：开发数据模型
![image](img/sq1.jpg) 第 5 章：开发数据模型

## 第 6 章：泛化与特化
![image](img/sq1.jpg) 第 6 章：泛化与特化

## 第 7 章：从数据模型到关系数据库设计
![image](img/sq1.jpg) 第 7 章：从数据模型到关系数据库设计

## 第 8 章：规范化
![image](img/sq1.jpg) 第 8 章：规范化

## 第 9 章：深入探讨键与约束
![image](img/sq1.jpg) 第 9 章：深入探讨键与约束

## 第 10 章：查询基础
![image](img/sq1.jpg) 第 10 章：查询基础

## 第 11 章：用户界面
![image](img/sq1.jpg) 第 11 章：用户界面

## 第 12 章：其他实现方式
![image](img/sq1.jpg) 第 12 章：其他实现方式

## 附录
![image](img/sq1.jpg) 附录

#### 索引
- 索引



