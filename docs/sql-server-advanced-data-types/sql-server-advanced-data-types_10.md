# 第 10 章：理解空间数据 ������������������������������������������������279

## 理解空间数据 ������������������������������������������������������������������������������279

## 空间数据标准 �������������������������������������������������������������������������������������286

## 知名文本格式 ������������������������������������������������������������������������������������������287



## 第十一章：处理空间数据

## 已知二进制

## 空间参考系统

## SSMS 与空间数据

## 小结

viii

目录

## 构造空间数据

## 查询空间数据

## 空间数据的索引

### 理解空间索引

### 创建空间索引

## 小结

ix

![](img/index-10_1.jpg)

## 关于作者

**彼得·A·卡特** 是一位 SQL Server 专家，在数据库开发、管理和平台工程方面拥有超过 15 年的经验。他目前是常驻伦敦的顾问。彼得撰写了多本涵盖各种 SQL Server 主题的书籍，包括安全性、高可用性和自动化。

xi

![](img/index-11_1.jpg)

## 关于技术评审

**伊恩·斯蒂尔克** 是一位常驻伦敦的自由 SQL Server 顾问。除了日常工作，他还是一位作者、软件工具创建者和技术评审员，并定期为 [www.i-programmer.info](http://www.i-programmer.info/) 撰写书评。他涵盖 SQL Server 的方方面面，并对性能和可扩展性有专业兴趣。如果您需要 SQL Server 系统方面的帮助，请随时通过 ian_stirk@yahoo.com 或 [www.linkedin.com/in/ian-stirk-bb9a31](http://www.linkedin.com/in/ian-stirk-bb9a31) 联系他。

伊恩要感谢彼得·卡特、乔纳森·詹尼克和吉尔·巴尔扎诺让这次的书籍编写经历对他而言更加轻松。我们没有人是孤军奋战的，考虑到这一点，伊恩要感谢这些特别的人：帕特·理查兹、巴加瓦·甘蒂、乔恩·麦凯布、

## 第十二章：处理分层数据和 HierarchyID

## 分层数据用例

## 传统层次结构建模

## 使用 HierarchyID 对层次结构建模

## HierarchyID 方法

## 使用 HierarchyID 方法

## 为 HierarchyID 列创建索引

## 小结

### 索引



Nick Fairway, Aida Samuel, Paul Fuller, Vikki Singini, Rob Lee, Gerald Hemming, and Jordy Mumba.

Ian's fee for his work on this book has been donated to the [全球驱虫行动](http://www.givingwhatwecan.org/report/deworm/).

xiii

## 引言

*SQL Server 高级数据类型*旨在揭开现代版本 SQL Server 中可供开发人员使用的复杂数据类型的神秘面纱。

在过去的几年里，我注意到许多 SQL 开发人员虽然听说过 SQL Server 中每种复杂数据类型，但常常避免使用它们，因为他们不确定如何最好地利用这些数据类型。这导致开发出了一些次优的解决方案，例如我最近遇到的一个事件：一位非常优秀且经验丰富的 SQL 开发人员使用自连接实现了复杂的层次逻辑，因为他对自己实现 `HierarchyID` 数据类型没有信心。

这激励我撰写本书——帮助 SQL 以及其他负责在应用程序中编写 `T-SQL` 的开发人员，更好地理解 SQL Server 中可用的复杂数据类型，并让他们有信心恰当地使用这些复杂结构。

本书首先探讨 SQL Server 中可用的简单、传统数据类型，并提醒读者为什么为数据类型做出正确的选择如此重要。接着，本书深入讨论了 SQL Server 中的复杂数据类型，即 `XML`、`JSON`、`HierarchyID`、`GEOGRAPHY` 和 `GEOMETRY`。书中的许多代码示例基于我在伦敦担任 SQL Server 顾问期间遇到的实际问题和解决方案。

本书中的许多代码示例使用了 `WideWorldImporters` 示例数据库。该数据库的 GitHub 仓库可在 `github.com/Microsoft/sql-server-samples/tree/master/samples/databases/wide-world-importers` 找到，`.bak` 文件可从 `github.com/Microsoft/sql-server-samples/releases/download/wide-world-importers-v1.0/WideWorldImporters-Standard.bak` 下载。

xv

