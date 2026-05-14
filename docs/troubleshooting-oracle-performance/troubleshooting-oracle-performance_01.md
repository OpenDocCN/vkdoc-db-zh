# **Oracle 性能故障排查**

**版权所有 © 2008 Christian Antognini**

保留所有权利。未经版权所有者和出版商的书面许可，本作品的任何部分不得以任何形式或任何方式（电子或机械，包括影印、录音或任何信息存储和检索系统）进行复制或传播。

ISBN-13: 978-1-59059-917-4
ISBN-10: 1-59059-917-9
ISBN-13 (电子版): 978-1-4302-0498-5

美国印刷装订 9 8 7 6 5 4 3 2 1

本书中可能出现商标名称。我们并未在每次提及商标名时使用商标符号，而是以编辑方式仅使用这些名称，旨在使商标所有者受益，并无商标侵权之意。

主编：Jonathan Gennick
策划编辑：Curtis Gautschi
技术审校：Alberto Dell'Era, Francesco Renne, Jože Senegacnik, Urs Meier
编辑委员会：Clay Andres, Steve Anglin, Ewan Buckingham, Tony Campbell, Gary Cornell, Jonathan Gennick, Matthew Moodie, Joseph Ottinger, Jeffrey Pepper, Frank Pohlmann, Ben Renow-Clarke, Dominic Shakeshaft, Matt Wade, Tom Welsh
项目经理：Sofia Marchant
文字编辑：Kim Wimpsett
制作副总监：Kari Brooks-Copony
制作编辑：Laura Esterman
排版：Susan Glinert Stevens
校对：Lisa Hamilton
索引：Brenda Miller
插图：April Milne
封面设计：Kurt Krames
制作总监：Tom Debolski

本书由 Springer-Verlag New York, Inc. 向全球书商发行，地址：233 Spring Street, 6th Floor, New York, NY 10013。电话 1-800-SPRINGER，传真 201-348-4505，电子邮件 orders-ny@springer-sbm.com，或访问 [`www.springeronline.com`](http://www.springeronline.com)。

有关翻译事宜，请直接联系 Apress，地址：2855 Telegraph Avenue, Suite 600, Berkeley, CA 94705。电话 510-549-5930，传真 510-549-5939，电子邮件 info@apress.com，或访问 [`www.apress.com`](http://www.apress.com)。

Apress 和 friends of ED 的图书可批量购买用于学术、企业或推广用途。大多数书目也提供电子书版本和许可。欲了解更多信息，请参考我们的批量销售-电子书许可网页：[`www.apress.com/info/bulksales`](http://www.apress.com/info/bulksales)。

本书中的信息“按原样”提供，不作任何担保。尽管在本书编写过程中已采取一切预防措施，但作者和 Apress 对因本书所含信息直接或间接引起或据称引起的任何损失或损害，不向任何人或实体承担任何责任。

将此书献给那些本应与我一同书写它，而我却花了太多时间独力完成的人……

献给 Michelle、Sofia 和 Elia。

## 内容概览

前言
关于作者
关于技术审校
致谢
引言
关于 OakTable 网络

## 第一部分 基础

### 第 1 章 性能问题

### 第 2 章 关键概念

## 第二部分 识别

### 第 3 章 识别性能问题

## 第三部分 查询优化器

## 第 4 章 系统与对象统计信息

## 第 5 章 配置查询优化器

## 第 6 章 执行计划

## 第 7 章 SQL 调优技术

## 第四部分 优化

### 第 8 章 解析

### 第 9 章 优化数据访问

### 第 10 章 优化连接

### 第 11 章 超越数据访问与连接优化

### 第 12 章 优化物理设计

## 第五部分 附录

### 附录 A 可下载文件

### 附录 B 参考文献

### 索引



