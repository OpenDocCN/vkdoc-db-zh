# Azure SQL 是开发工具吗？

许多开发者可能会认为，管理数据库乃至更广泛的数据，不属于他们的工作范畴。这在 20 年前或更早可能是对的，但自那时起情况已大不相同。

在当今世界，敏捷性以及适应和变化的能力是成功的关键因素，全栈开发者或后端开发者的角色日益普遍。虽然这些角色不需要具备数据库管理员、数据工程师或数据科学家所期望的那种深厚的数据知识，但他们的工作最终也会涉及操作一些数据。

现在，这并不意味着你应该停止学习 C#、Java、Python 或任何你想要或需要的语言，而一头扎进去学习 SQL。需要做的只是理解，Azure SQL 是你工具集中的一个工具，就像你可能拥有的任何其他工具一样，从设计模式到 `WeakReference` 类，仅举两个处于知识谱系完全相反两端的例子。

如同任何其他工具一样，你对它了解得越多，就能运用得越好。



### 保持超级简单

作为开发者，你必须明白，例如，在尚未正确设计数据库以利用索引之前，却去创建一个极其复杂的缓存机制来提升解决方案的性能，这是没有意义的。让我清楚地告诉你：没有什么方法能比一个好的索引策略带来更大的性能提升。如果你试图做一些不同且更“聪明”的事情，你迟早会在自己的应用程序中（很可能以某种形式的**B 树**）重新实现数据库中已有的解决方案——只是成本和复杂度要高得多。从专业角度来看，这毫无道理。像这样的过度工程化故事不仅仅是假设，也不只是用来吓唬初级开发者：其中一位作者就亲身经历过。他曾被要求优化一个接近实时处理的复杂数据解决方案，该方案使用了复杂的多层缓存、最先进的微服务架构和最新的硬件，但性能仍然很差且成本高昂。经过深入分析，提出的方案是拆除一大部分现有架构，并用数据库上的几个索引（更准确地说，是**列存储索引**）来替代。创建该解决方案的开发者并不知道这些索引的存在，因为他们的认知还停留在**SQL Server 2005**的功能上。经过一些争论、阻力以及关于成长型心态的讨论，代码被重构和重新架构，最终审查通过的解决方案从使用七台虚拟机来承载一个复杂的分布式缓存方案，变成了仅需一个**Azure SQL**数据库，成本降至十分之一，性能却大大提升。

通过使用正确的工具，本质上复杂的解决方案可以被分解为更小的片段，每个片段都可以用正确的工具更简单地进行管理。**Azure SQL**就是这样一个工具，在数据处理方面通常很合适。即使你认为它不足以应对挑战，也请试一试，并准备好改变你的看法。这就是你成为更好的开发者的方式。

### 成为通才专家

开发更多的是做出决策，而不仅仅是编写代码。当然，代码是你向世界展示你为解决某个业务问题或实现某个功能而决定做的事情的方式，但开发过程早在你的脑海中就开始了，即你开始评估手头有哪些选项来实现你被要求开发的功能时。

作为开发者，实现目标最重要的步骤之一是能够掌控局面，并决定什么是实现目标的最佳架构和实施策略。通过掌握不止一门编程语言和不止一种数据管理解决方案，人们可以挑选，或建议，为当前正在开发的特定解决方案选择最佳方案。有时它可能是像**Apache Spark**这样复杂灵活的东西，有时则可能是像**Redis**这样简单易用的东西。很多时候，像**Azure SQL**这样的现代数据库可以在一个地方提供数量惊人的特性，在选项、成本和性能之间提供良好的平衡。

为了做出最佳选择，人们需要至少了解一些处于开发空间边缘的工具，因为它们将有助于处理该空间之外的所有事情。在许多情况下，“所有事情”意味着“数据”。一个人对数据了解得越多，解决方案就会越好。使用敏捷方法中同样的定义，一个人需要成为一个 *通才专家* ([`https://aka.ms/swkogs`](https://aka.ms/swkogs))。

精通一些技能，但也要积极学习其他技能。

### 如果想了解更多

本书旨在为你提供关于**Azure SQL**的非常实用的观点，以及它如何在你作为开发者或架构师的日常工作中提供帮助。当然，要理解所描述的内容为何重要或比某些其他替代方案更好，需要一些理论背景。我们希望你通过理解概念来学习，而不仅仅是死记硬背。这是一种基础的心态，因为你将是那个被要求做出某些开发或设计选择的人。虽然我们无法在你身边提供帮助，但本书可以，但前提是你理解所解释概念的细节。我们在书中放入了我们认为足够让你入门并给你一个关于你能实现什么的概念的内容。这足以让你掌控自己的开发和设计决策。然而，你可能想知道更多并深入某些概念；毕竟，你知道得越多，就越想知道。因此，在每一章中，你都会找到一个同名的小节，你可以在其中找到一些书籍、文章或帖子作为参考，以更深入地了解数据库知识。以下是本章的列表：

*   《Database in Depth: Relational Theory for Practitioners》 – [`www.amazon.com/Database-Depth-Relational-Theory-Practitioners/dp/0596100124`](http://www.amazon.com/Database-Depth-Relational-Theory-Practitioners/dp/0596100124)
*   《Practical Issues in Database Management: A Reference for the Thinking Practitioner》 – [`www.amazon.com/Practical-Issues-Database-Management-Practitioner/dp/0201485559`](http://www.amazon.com/Practical-Issues-Database-Management-Practitioner/dp/0201485559)
*   《SQL and Relational Theory: How to Write Accurate SQL Code》 – [`www.amazon.com/SQL-Relational-Theory-Write-Accurate/dp/1449316409`](http://www.amazon.com/SQL-Relational-Theory-Write-Accurate/dp/1449316409)
*   《Data Modeling Essentials, Third Edition》 – [`www.amazon.com/Modeling-Essentials-Third-Graeme-Simsion/dp/0126445516`](http://www.amazon.com/Modeling-Essentials-Third-Graeme-Simsion/dp/0126445516)
*   《Database Design and Relational Theory: Normal Forms and All That Jazz》 – [`www.amazon.com/Database-Design-Relational-Theory-Normal/dp/1484255399`](http://www.amazon.com/Database-Design-Relational-Theory-Normal/dp/1484255399)
*   《Hot Patching SQL Server Engine in Azure SQL Database》 – [`https://techcommunity.microsoft.com/t5/azure-sql-database/hot-patching-sql-server-engine-in-azure-sql-database/ba-p/849700`](https://techcommunity.microsoft.com/t5/azure-sql-database/hot-patching-sql-server-engine-in-azure-sql-database/ba-p/849700)



