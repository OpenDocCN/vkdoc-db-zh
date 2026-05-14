# 索引



## 关于作者

![Image](img/9781430238102_FM-01.jpg) ![Image](img/square.jpg) Brian K. McDonald，拥有 MCDBA、MCSD 认证，是一位商业智能和数据仓库极客，为美国顶尖的销售与营销公司 Acosta 工作。Brian 在俄亥俄州辛辛那提出生并长大，但目前与美丽的妻子以及两个小忍者——Bailey 和 Kylie——居住在佛罗里达州的杰克逊维尔。作为 Business Enterprise Solution Technologies 的前任所有者和独立顾问，Brian 在架构数据仓库、数据建模、数据库管理、`SQL Server Reporting Services`、`Integration Services`、`Analysis Services`和`PowerPivot`方面拥有丰富经验。在 IT 行业工作的 13 年中，他还曾担任应用程序开发人员、网络管理员和数据库管理员。在美国海军陆战队为国服役期间培养的一个特质，便是 Brian 对他所有项目所怀有的自豪感，他不仅勤奋工作以满足期望，而且力求大幅超越。他持续致力于在专业和个人方面提升自己的技能组合。除了拥有商业管理和计算机信息系统学位外，他还是微软认证解决方案开发专家（`MCSD`）和微软认证数据库管理员（`MCDBA`）。Brian 是`SQL Server`社区的活跃成员。他乐于通过培训、虚拟和公开演讲，以及为许多行业领先网站（如`SQLBIGeek`、`SQLServerCentral`和`SQLServerPedia`）撰写文章和博客来回馈`SQL Server`社区。您可能曾在佛罗里达州奥兰多举办的首届`SQL Rally`上，或在多年来他参与过的众多`SQL Saturdays`、`Code Camps`或`User Groups`中的某次活动中见过 Brian 演讲。在不回馈社区或工作时，Brian 喜欢与妻子 Sherri 一起“极客”一下，并让他的孩子们在他身上练习他们的`UFC`风格动作。

![Image](img/9781430238102_FM-02.jpg) ![Image](img/square.jpg) Shawn McGehee 是美国一家大型保险公司的`DBA`和经理。自 7.0 版本以来，他就一直以某种方式与`SQL Server`打交道，并从 1998 年起就开始在`IT`领域工作。从开发者起步，他逐渐转向全职管理岗位，但仍然乐于在能运用其开发背景解决挑战时接受挑战。他活跃于`SQL Server`社区，喜欢在用户组、`SQL Saturdays`和偶尔的`Code Camp`上演讲。他目前负责运营奥兰多`SQL Server`用户组`OPASS`，并在过去几年里一直参与`SQL Server`社区。您通常可以在东南地区的任何地方找到他在`SQL Saturday`上漫步。

## 关于技术审校

![Image](img/9781430238102_FM-03.jpg) ![Image](img/square.jpg) Rodney Landrum 与`SQL Server`打交道的时间久到自己都记不清了。他定期撰写关于各种技术的文章，包括`Integration Services`、`Analysis Services`和`Reporting Services`。（Rodney 是本书的原作者）。他著有*`SQL Server Tacklebox`*以及三本关于`Reporting Services`的书籍。他定期为`SQLServerCentral`、`SQL Server Magazine`和`Simple–Talk`供稿。他的日常工作涉及在奥兰多监督一个大型的`SQL Server`基础设施。他声称自己拥有“日复一日与数据库打交道”这句话的所有权。任何不同意的人，都渴望在一场掰手腕比赛中输掉。

![Image](img/9781430238102_FM-04.jpg) ![Image](img/square.jpg) Sherri McDonald 是一位`BI`开发人员，专门从事`SQL Server Reporting Services`，担任`Reporting Services`和`TSQL`基础的培训师。她还因与`SSWUG`共同制作的`Reporting Services` `DVD`而闻名。她担任过许多角色，在服务行业有超过 14 年的经验，而最近则专注于微软`BI`技术栈。Sherri 是一位活跃的博主，是她当地`SQL Server`用户组的成员，并在佛罗里达州各地的`SQL Saturday`和`Code Camp`活动上发表过演讲。Sherri 在俄亥俄州辛辛那提出生并长大，目前与丈夫 Brian 和两个孩子居住在佛罗里达州的杰克逊维尔。当不学习最新的技术时，Sherri 喜欢看电影、乘游轮旅行以及与朋友和家人一起进行其他旅行。

## 引言

从核心来看，设计报表的过程在过去 20 年里并没有发生实质性变化。报表设计者在诸如`Reporting Services`、`Business Objects Reports`或`Microsoft Access`等设计应用程序中，布局包含来自已知数据源的数据的报表对象。然后，他或她测试报表执行情况，验证结果的准确性，并将报表分发给目标受众。

当然，不同的设计应用程序之间存在足够的差异，这意味着设计者必须熟悉每个特定的环境。然而，有足够的交叉功能使得学习曲线变得平缓。例如，`SUM`函数在`Business Objects Reports`中与在`Microsoft Access`中，以及在`结构化查询语言`（`SQL`）中是相同的。

使用`Microsoft SQL Server 2012 Reporting Services`（本书中统称为`SSRS`），不同图形化报表设计应用程序之间的报表设计方式差异再次变得微小。因此，如果您有以前的报表设计经验，您对`SSRS`的学习曲线应该相对平缓。如果您来自`.NET`环境，这一点尤其正确，因为`SSRS 2012`的报表设计器应用程序是`Visual Studio 2010`或随`SQL Server 2012`一起提供的应用程序`SQL Server Data Tools`（`SSDT`），以前称为`Business Intelligence Development Studio`（`BIDS`）。我们在本书中交替使用`BIDS`和`SSDT`，大多数引用使用`BIDS`。我们这样做主要是因为`Reporting Services`在`SQL Server`的`Business Intelligence`产品栈中所扮演的角色，同时也考虑到可能正在使用早期版本`Reporting Services`（如`SSRS 2008 R2`）的读者。

尽管如此，有几点不同将`SSRS`与其他报表解决方案区分开来：

*   它提供了一个基于`报表定义语言`（`RDL`）的标准报表平台，RDL 是规定所有`SSRS`报表通用结构的`XML`架构。这允许从任何支持`RDL`架构的第三方应用程序创建报表。
*   `SSRS`是`SQL Server 2012`版本不可分割的一部分。
*   `SSRS`开箱即用地提供了在其他产品中需要昂贵附加组件才能获得的功能。这些功能包括订阅服务、报表缓存、报表历史记录和报表执行计划安排。
*   `SSRS`可以通过第三方插件、自定义代码和已编译的`DLL`进行扩展。
*   `SSRS`作为一种基于 Web 的解决方案，可以部署在各种平台上。
*   `SSRS`还能轻松与微软的企业协作软件`SharePoint 2010`集成。

本书是与一个真实的医疗保健应用程序`SSRS`部署并行编写的，因此它涵盖了`SSRS`几乎所有的设计和部署考虑因素，始终从如何有效完成任务的角度出发。您将找到分步指南、实用技巧和最佳实践，以及您可以修改并在自己的`SSRS`应用程序中使用的代码示例。

### 本书面向的读者

我们合著本书的意图是从多个角度演示如何使用`SSRS`。作为报表架构师和报表开发人员，我们使用`SSRS`标准工具（如`BIDS`中的`Report Designer`和`Report Manager`）来完成报表设计和部署过程。我们还展示了开发人员如何通过创建自定义`Windows`和`Web Forms`应用程序来扩展`SSRS`。



## 前提条件

本书所有示例中使用的核心软件是：

*   Microsoft SQL Server 2012
*   Microsoft Visual Studio 2010 – 在第 7、8、9 和 10 章中使用
*   Microsoft SharePoint 2010 – 在第 12 章中与 SSRS 集成时使用
*   Microsoft SQL Server 2008 R2 – 在第 13 章中用于通过报表模型进行临时报告

如果您，作为读者，希望跟随本书中的示例进行操作，则需要使用上述每项软件。大多数示例是使用 SQL Server 2012 构建的，但除了第 7、8 和 9 章外，它们也可以在 SQL Server 2008 R2 上执行。

## 下载代码

在本书中，我们使用了为某个医疗保健应用程序设计的真实数据库的子集，这是我们中一些人多年来开发的。您可以在 Apress 网站（[`www.apress.com`](http://www.apress.com)）的“源代码/下载”部分找到所有支持材料（数据库、第 12 章中使用的数据仓库数据库和多维数据集文件、完成的 RDL 文件、查询、存储过程以及 .NET 应用程序项目）和完整的安装说明。由于多年来已有许多类似标题的书籍存在，使用本书的 ISBN 号码可能更容易找到它。本书的 13 位行业标准 ISBN 号码是 978-1-4302-3810-2。

## 联系作者

如果您对本书的任何部分有任何疑问，请随时通过我们的电子邮件或 Twitter 账户联系我们。我们很乐意听到您购买了我们的书，所以请随时给我们发推文。我们衷心希望您能从阅读本书中获得与我们为您撰写本书时所拥有的同等乐趣。

> Brian K. McDonald
> [`bmcdonald@sqlbigeek.com`](http://bmcdonald@sqlbigeek.com)
> @briankmcdonald
>
> Shawn McGehee
> [`shawnnwf@gmail.com`](http://shawnnwf@gmail.com)
> @SQLShawn
>
> Rodney Landrum
> [`rodneylandrum@hotmail.com`](http://rodneylandrum@hotmail.com)
> @SQLBeat

