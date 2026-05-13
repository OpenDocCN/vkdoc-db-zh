# 精通 Oracle SQL

## 版权所有 © 2014 托尼·哈斯勒

本作品受版权保护。无论涉及材料的全部或部分，出版者保留所有权利，特别是翻译、转载、图表再利用、朗诵、广播、微缩胶片或其他任何物理形式的复制，以及信息存储与检索、电子改编、计算机软件，或任何现有及未来开发的类似或不同方式的权利。此项法律保留的例外情况是用于评论或学术分析目的的简短摘录，或专门为了在计算机系统上输入和执行而提供的材料，仅供作品的购买者专用。只有在出版者所在地版权法现行版本的规定下，才允许复制本出版物或其部分内容，且必须始终获得施普林格的许可。使用许可可通过版权结算中心的 RightsLink 获得。违规行为将根据相应的版权法承担法律责任。

ISBN-13 (平装): 978-1-4302-5977-0

ISBN-13 (电子): 978-1-4302-5978-7

本书中可能出现商标名称、标识和图片。我们仅在编辑意义上且为商标所有者利益而使用这些名称、标识和图片，并非每次出现都附带商标符号，无侵犯商标权之意。

在本出版物中使用商品名称、商标、服务标识及类似术语，即使未特别标识，也不应被视为表达其是否受专有权利约束的意见。

虽然本书中的建议和信息在出版时被认为是真实和准确的，但作者、编辑或出版商均不对可能出现的任何错误或遗漏承担任何法律责任。出版商对本出版物所含材料不作任何明示或暗示的保证。

## 出版者: 海因茨·温海默
## 主编: 乔纳森·詹尼克
## 开发编辑: 马修·穆迪
## 技术审阅: 弗里茨·霍格兰，兰道夫·盖斯特，多米尼克·德尔莫利诺，卡罗尔·达科
## 编辑委员会: 史蒂夫·安格林，马克·贝克纳，尤安·白金汉，加里·康奈尔，路易丝·科里根，吉姆·德沃尔夫，乔纳森·詹尼克，乔纳森·哈塞尔，罗伯特·哈钦森，米歇尔·洛曼，詹姆斯·马克汉姆，马修·穆迪，杰夫·奥尔森，杰弗里·佩珀，道格拉斯·庞迪克，本·雷诺-克拉克，多米尼克·沙克索夫特，格温南·斯皮林，马特·韦德，史蒂夫·韦斯
## 协调编辑: 吉尔·巴尔扎诺
## 文字编辑: 阿普丽尔·龙多
## 排版: SPi Global
## 索引: SPi Global
## 美术: SPi Global
## 封面设计: 安娜·伊什琴科

本书通过 Springer Science+Business Media New York 面向全球图书贸易发行，地址：233 Spring Street, 6th Floor, New York, NY 10013。电话 1-800-SPRINGER，传真 (201) 348-4505，电子邮件 `orders-ny@springer-sbm.com`，或访问 `www.springeronline.com`。Apress Media, LLC 是一家位于加利福尼亚州的有限责任公司，其唯一成员（所有者）是 Springer Science + Business Media Finance Inc (SSBM Finance Inc)。SSBM Finance Inc 是一家特拉华州公司。

有关翻译信息，请发送电子邮件至 `rights@apress.com`，或访问 `www.apress.com`。

Apress 和 friends of ED 的书籍可批量购买用于学术、企业或推广用途。大多数书名也提供电子书版本和许可。更多信息，请参阅我们的批量销售-电子书许可网页：`www.apress.com/bulk-sales`。

作者在正文中引用的任何源代码或其他补充材料，读者均可访问 `www.apress.com` 获取。有关如何查找本书源代码的详细信息，请访问 `www.apress.com/source-code/`。

献给玛丽安。

## 内容概览

关于作者
关于技术审阅
致谢
前言
引言

![image](img/sq1.jpg) 第一部分：基本概念
![image](img/sq1.jpg) 第 1 章：SQL 特性
![image](img/sq1.jpg) 第 2 章：基于成本的优化器
![image](img/sq1.jpg) 第 3 章：执行计划基础概念
![image](img/sq1.jpg) 第 4 章：运行时引擎
![image](img/sq1.jpg) 第 5 章：优化入门
![image](img/sq1.jpg) 第 6 章：对象统计与部署

![image](img/sq1.jpg) 第二部分：高级概念
![image](img/sq1.jpg) 第 7 章：高级 SQL 概念
![image](img/sq1.jpg) 第 8 章：高级执行计划概念
![image](img/sq1.jpg) 第 9 章：对象统计

![image](img/sq1.jpg) 第三部分：基于成本的优化器
![image](img/sq1.jpg) 第 10 章：访问方法
![image](img/sq1.jpg) 第 11 章：连接
![image](img/sq1.jpg) 第 12 章：最终状态优化
![image](img/sq1.jpg) 第 13 章：优化器转换

![image](img/sq1.jpg) 第四部分：优化
![image](img/sq1.jpg) 第 14 章：问题出在哪里？
![image](img/sq1.jpg) 第 15 章：物理数据库设计
![image](img/sq1.jpg) 第 16 章：重写查询
![image](img/sq1.jpg) 第 17 章：优化排序
![image](img/sq1.jpg) 第 18 章：使用提示
![image](img/sq1.jpg) 第 19 章：高级调优技术

![image](img/sq1.jpg) 第五部分：使用 TSTATS 管理统计信息
![image](img/sq1.jpg) 第 20 章：使用 TSTATS 管理统计信息

索引

![image](img/FM-02.jpg)

