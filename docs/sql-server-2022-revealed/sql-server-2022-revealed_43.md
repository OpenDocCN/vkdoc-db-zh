# 性能与可伸缩性

我总是乐于看到任何 SQL Server 版本在性能和可伸缩性方面的创新和增强。原因之一是，这个领域的几乎所有改进都源于调优实践、真实客户案例或我们为基准测试所做的研究。SQL Server 2022 在整个工作负载和引擎组件的各个方面都进行了各种性能与可伸缩性投资。

## 列存储与批处理模式改进

SQL Server 2022 包含增强功能，以提升列存储索引和批处理模式操作（在 SQL Server 2019 及更高版本中，批处理模式操作不再强制要求列存储）的性能。

### 有序聚集列存储索引

Azure Synapse Analytics 和 SQL Server 都支持列存储索引。关于列存储索引以及它们为何对应用程序有益，可以写一整本书。我仍然认为这是 SQL Server 中一个未被充分利用的功能。我遇到的几乎所有不了解列存储的客户，一旦理解了它们的工作原理，立即就能看出如何利用它们。如果你不了解列存储，请直接查阅文档开始学习：[`https://docs.microsoft.com/sql/relational-databases/indexes/columnstore-indexes-overview`](https://docs.microsoft.com/sql/relational-databases/indexes/columnstore-indexes-overview)。

Azure Synapse Analytics 增加了一个选项，可以创建 *有序* 的聚集列存储索引。虽然构建有序聚集列存储索引可能需要更长时间，但对某些查询的性能提升可能非常显著。SQL Server 2022 现在支持有序聚集列存储索引。了解更多：[`https://docs.microsoft.com/azure/synapse-analytics/sql-data-warehouse/performance-tuning-ordered-cci`](https://docs.microsoft.com/azure/synapse-analytics/sql-data-warehouse/performance-tuning-ordered-cci)。

### 列存储字符串处理改进

我们没有关于此的特定文档，但当我与资深 SQL 工程师 Ryan Stonecipher 聊到这个话题时，他对我说的原话是：“只需重建你的索引，我们就会为深层数据（例如 *char 和 *binary + guid）维护最小/最大值。此外，我们现在支持在列存储上进行 LIKE 下推操作，以及一种快速的字符串相等实现。”



#### 向量扩展以改进批处理模式

在 SQL Server 2016 中，我们添加了代码以利用称为基于`向量`的硬件能力的特殊处理器指令。基于向量的硬件能力允许程序因为处理器中的向量寄存器而更简单、更快速且以更并行的方式处理数据。如果您希望阅读一些关于此主题的非常深入的内容，这里有一篇不错的文章：[`www.techspot.com/article/2166-mmx-sse-avx-explained`](http://www.techspot.com/article/2166-mmx-sse-avx-explained)。

SQL Server 2016 能够识别两种不同类型的向量能力，即`SSE`和`AVX`，并利用它们来加速列存储操作，尤其是在使用批处理模式时。Robert Dorr 在这篇博文中概述了我们是如何做到这一点的：[`https://docs.microsoft.com/archive/blogs/bobsql/sql-2016-it-just-runs-faster-column-store-uses-vector-instructions-sseavx`](https://docs.microsoft.com/archive/blogs/bobsql/sql-2016-it-just-runs-faster-column-store-uses-vector-instructions-sseavx)。

最新的向量硬件技术是`AVX-512`，而 SQL Server 2022 可以识别这种新的硬件能力，从而为列存储和行存储的批处理模式操作符获得更显著的性能提升。我很好奇是什么促使我们研究`AVX-512`，并发现了 Conor Cunningham 的这段有趣引述：

在疫情期间，我读到了一篇学术论文，其中解释了如何使用`AVX-512`指令实现快速排序。这些指令本质上是一次执行 N 个操作而不是一个，这意味着它有可能大幅提升性能关键代码的速度。我实现了论文中的排序算法，并开始对其进行改进。后来，我想开始尝试在 SQL 的代码中实现一些东西。我们有一些逻辑使用了上一代的向量指令（`AVX2`），我研究了那段代码，并想通了如何编写一个等效版本，使用更新、更宽的向量指令，这算是一个副业项目。当我完成后，我们在关键的分析基准测试（`TPC-H`）上获得了良好的性能提升。有趣的是，直到有一天我出现并将这份礼物交给团队时，我都没有告诉任何人我在做这件事——这并不是一个为 SQL Server 2022 正式资助的项目，但当我们意识到它能为客户带来的价值后，我们优先考虑了它。

在撰写本书时，我们将此增强功能置于一个跟踪标志之下，并且有可能我们在发布产品时仍然需要一个跟踪标志——主要是因为使用此功能是一个高级概念，并非适用于所有人。请访问 [`https://aka.ms/sqlserver2022docs`](https://aka.ms/sqlserver2022docs) 查看最新信息。

这项工作的一个方面是您可能会看到一个新的`ERRORLOG`条目，它看起来像这样：

```
CPU vectorization level(s) detected:  SSE SSE2 SSE3 SSSE3 SSE41 SSE42 AVX AVX2 POPCNT BMI1 BMI2
```

这些名称是我们检测到的处理器中各种向量 CPU 能力。如果您使用的是支持这种新向量模型的芯片组，则会出现`AVX-512`。

#### 列存储段消除增强

列存储索引的列被组织成段。列存储包含一个称为段消除的概念，以便可以根据查询消除或不准备某些段。在 SQL Server 2022 之前，只有数据类型为数值、日期和时间以及比例小于或等于二的`datetimeoffset`的列才是段消除的候选对象。现在在 SQL Server 2022 中，我们将段消除扩展到了新的数据类型，包括字符串、二进制和`guid`数据类型，以及比例大于二的`datetimeoffset`数据类型。

### 可扩展性改进

有几项改进旨在帮助提高您的查询、工作负载和 SQL Server 使用的可扩展性。您最不希望看到的就是投入精力修改应用程序设计、购买新服务器或基础设施后，却被 SQL Server 限制了扩展能力。此领域的所有工作都是客户案例或我们内部及与合作伙伴进行的基准和测试的直接结果。

#### 缓冲池并行扫描

如果您访问我的网站查看我所有的演讲（我是一位开源演讲者），[`https://aka.ms/bobwardms`](https://aka.ms/bobwardms)，您会找到一个名为`SQL PASS 2013`的文件夹。在 2013 年的`PASS Community Summit`上，我做了一个关于内存的半天演讲。如果您查看这份演示文稿，会看到一些关于 SQL Server 缓冲池的内部信息。在这些幻灯片中，我谈到了`BUF`哈希表和`BUF`结构。`BUF`结构指向页面，而`BUF`哈希表是一种组织所有`BUF`结构以便轻松找到页面的方法。因此，使用`BUF`哈希表查找来定位特定页面并不那么困难。

不幸的是，SQL Server 中的某些操作必须扫描所有的`BUF`结构。可以将其想象为扫描聚集索引与使用索引查找。就像聚集索引扫描一样，如果您只需要几行数据，但聚集索引很大，那么获取结果仍可能需要很长时间。

对于缓冲池的某些操作也可能发生同样的问题，我们曾在客户那里看到过这种情况。他们的服务器可能有，比如说，1TB 的 RAM，并允许缓冲池消耗大部分内存。然后他们会执行一些维护操作，即使针对一个非常小的数据库，也会花费异常长的时间。这是因为我们会扫描整个大约 1TB 的`BUF`结构来完成操作。我们甚至添加了`ERRORLOG`消息来警告用户正在发生这种情况。更多信息请参阅文章 [`https://docs.microsoft.com/troubleshoot/sql/performance/buffer-pool-scan-runs-slowly-large-memory-machines`](https://docs.microsoft.com/troubleshoot/sql/performance/buffer-pool-scan-runs-slowly-large-memory-machines)。这是一个经典问题：某个架构多年来没有引起问题，但随着硬件技术变得极其庞大，这种设计需要更新。

我们研究了这个问题，并提出了类似这样的想法：“如果查询可以通过并行运行来获得更快的结果，为什么我们在扫描缓冲池时不能这样做呢？”（这不是原话；这是我在阅读此功能的设计文档时的转述。）

这项工作的成果是缓冲池并行扫描，该功能在 SQL Server 2022 中默认为内存容量大的系统启用。我在 Microsoft 的同事 David Pless 参与了此功能的开发，并在 [`https://cloudblogs.microsoft.com/sqlserver/2022/07/07/improve-scalability-with-buffer-pool-parallel-scan-in-sql-server-2022`](https://cloudblogs.microsoft.com/sqlserver/2022/07/07/improve-scalability-with-buffer-pool-parallel-scan-in-sql-server-2022) 中解释了其工作原理。

David 展示了我们在 SQL Server 2022 中看到的一些显著的改进数据。此外，请查看他的 Data Exposed 节目，他在其中通过一个巧妙的演示展示了更多细节：[`https://youtu.be/4GvU106Xiag`](https://youtu.be/4GvU106Xiag)。我在一个 T-SQL 笔记本中包含了 David 发现的结果示例，您可以使用 Azure Data Studio 查看该示例：`ch06_meatandpotatoes\scalability\Buffer_pool_parallel_scan_BPP_notebook_quick.ipynb`。



## 普尔维的清单

多年来，我在微软遇到了许多才华横溢、善良而出色的人。他们默默无闻地工作，从不大肆宣扬。他们只是做好自己的工作，做得非常出色，并且极其友善和友好。普尔维·沙阿就是其中之一。普尔维是 SQL 团队的高级软件工程师，但她的角色远超过这个头衔。普尔维在我们的“性能”团队工作，她的时间都花在寻找让 SQL Server 和 Azure SQL 变得更快的方法上——快到我们大多数人都无法理解的程度。她和团队的工作在很多方面都能被感受到，但你可能永远看不到，因为它们通常是对引擎的增强，而不是一个“功能”。

> **注意**
>
> 她为 SQL Server 2019 工作的一个显著例子是针对 `tempdb` 的 `PFS` 页面并发性改进，这非常棒！

在这个版本发布之际，我联系了普尔维，请她分享我希望读者了解的可扩展性增强列表。以下是她的列表：

*   **减少的缓冲池 I/O 提升** – 过去存在一些情况，我们会将单页读取提升为八页 I/O，从而导致系统问题。事实证明这并没有真正帮助性能，反而可能使 I/O 系统过载，因此我们调整了这个算法。这只是让我们的引擎在执行 I/O 操作时变得更智能。

*   **增强了我们的一些核心自旋锁算法** – 自旋锁是引擎内部多线程一致性的关键部分。我们一直在寻找调整这些算法的方法，以确保引擎在各种工作负载下都能以最佳状态运行。我们进行了一些内部调整，使自旋锁更高效，这些改进深入引擎内部，很难量化其影响（普尔维和团队在非常深的层次上做这件事，以确保他们的工作真的让一切变得更好，而不是更糟）。

*   **改进了虚拟日志文件算法** – 虚拟日志文件是物理事务日志的抽象。基于日志增长而产生大量小型虚拟日志文件会影响恢复等操作的性能。我们更改了在某些日志增长场景下创建多少虚拟日志文件的算法，以减少此类问题。你可以在我们的文档 [`https://docs.microsoft.com/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?#virtual-log-files-vlfs`](https://docs.microsoft.com/sql/relational-databases/sql-server-transaction-log-architecture-and-management-guide?#virtual-log-files-vlfs) 中阅读有关我们如何更改此算法的信息。

*   **日志文件增长的即时文件初始化** – 直到 SQL Server 2022，我们都不能对事务日志使用即时文件初始化，因为我们认为无法正确检测“日志末尾”。在 SQL Server 2022 中，我们已经确定，当我们需要增长事务日志时，如果增长大小被认为是“中等”的（最高 64Mb），我们**可以**使用 `IFI`。对于在增长操作期间将日志文件清零会导致阻塞问题的场景，这可能是一个重大的性能胜利。事实证明，64Mb 正好是创建新数据库时 SQL Server 数据和日志文件的默认自动增长大小。

普尔维和她的团队做了如此出色的工作，使 SQL Server 引擎保持调整和快速。我问普尔维更多关于她为什么喜欢在微软工作的原因，她说：“保持 SQL Server 和 Azure SQL 以峰值性能运行是我工作的一部分。我竭尽全力做到最好。当然，我热爱我的工作！希望我能做出积极的贡献，继续我们的使命。”

### “免动手”的 tempdb

尽管 SQL Server 是一个伟大的产品，但在与 SQL Server 打交道 29 年后，我仍然遇到一些至今让我抓狂的问题。其中之一就是 `tempdb` 的 `latch` 争用。

#### 挑战是什么？

SQL Server 中的所有数据库页面都需要使用 `latch` 进行物理保护。一些数据库页面被称为系统页面，因为它们包含分配信息；这些就是 `PFS`、`GAM` 和 `SGAM` 页面。任何需要分配或释放的查询或操作都可能需要读写这些页面。对这些页面的读写需要一种 *latch 保护方案*。需要 `latch` 保护是因为两个线程可能同时尝试修改或读取一个页面（这是对 `latch` 工作原理的一个非常基本的描述）。

这些页面在数据库文件中位于固定位置。此外，每个表都有用于存储数据和索引的页面。任何影响这些页面的查询或操作也需要 `latch` 保护。因为系统表和其他表一样有页面，SQL 中的特定 DDL 语句最终会导致系统表页面的 `latch` 保护和争用。

现在考虑 `tempdb`。涉及 `tempdb` 的 SQL Server 工作负载通常需要大量的分配和系统表页面操作。临时表、表变量和溢出只是引发这些操作的几个例子。所有这些通常都需要快速且频繁的页面分配和系统表修改。因此，当许多用户并发运行导致在 `tempdb` 内进行这些操作的查询时，自然会对用于分配和系统表的页面的 `latch` 产生争用或*压力*。这个问题的主要症状是看到你的请求等待在 `PAGELATCH_XX` 等待类型上。

最初，当 `latch` 在 SQL Server 7.0 中首次引入时，我看到过 `tempdb` 在 `PFS`、`GAM` 和 `SGAM` 页面上的 `latch` 争用，因为当时数据库页面的默认行为是单个文件。通常会发生的情况是，一个应用程序会被多个并发用户使用，这会导致频繁的 `CREATE` 和 `DROP` 操作，从而需要分配和释放操作。这自然会对单个文件中的 `PFS`、`GAM` 和 `SGAM` 页面造成压力。




#### 历年解决方案

当我们最初在 SQL Server 7.0 乃至 SQL Server 2000 中真正开始遇到这个问题时，CSS（客户服务与支持）团队的许多人都会建议客户为 tempdb 数据创建多个文件。为什么？因为这是在自然地为这些分配页的**闩锁并发**进行 `分区`。这是因为 SQL Server 会以轮询方式分配页，并在多个文件之间利用比例填充算法来处理数据页。这个方案确实帮助解决了许多这类闩锁争用问题。但随后出现了一个新问题：需要创建多少个文件才能解决问题？不幸的是，许多人矫枉过正，建议文件数量应该匹配 SQL Server 的 CPU 数量。我以及社区中的其他人，比如 Paul Randal，则有不同的看法。例如，我们知道对于一台 64 核 CPU 的机器，你可能并不需要 64 个文件。常识告诉我们，这似乎弊大于利。我最终对此课题进行了一些详细研究，由此我建议的最佳方法是：对于 8 个 CPU 使用 8 个文件，然后每次增加 4 个文件，直到你的工作负载达到性能平衡点。关于这种方法的更多细节，你可以查看我在 2011 年 PASS Summit 大会上做的演示：[`www.youtube.com/watch?v=SvseGMobe2w`](http://www.youtube.com/watch?v=SvseGMobe2w)。

我们做的另一件事是引入了 `跟踪标志 1118`，以便所有表的分配都使用统一区，从而减轻文件的 `SGAM` 页的压力。我们还发现，使用多个文件来缓解系统页闩锁争用，只有在所有文件大小完全相同时才有效。因此，我们引入了 `跟踪标志 1117`，以便任何自动增长活动都会增长所有文件以保持它们大小相同。

任何真正的瓶颈都需要像剥洋葱一样层层深入。即使采用了这种多文件方法，一个系统如果用户数量极大，`PFS`、`GAM` 和 `SGAM` 闩锁争用仍然可能发生，但与新的主要问题——系统表页闩锁争用——相比，它就相形见绌了。我们起初很难认识到这一点。我们已经习惯于运行脚本来识别特定页的闩锁争用：`PFS`、`GAM` 和 `SGAM`。

一旦我们发现热点是系统表页，老实说，我们别无他法，只能告诉客户减少他们的 tempdb 工作负载争用。我们确实创建了一些有帮助的解决方案，称为 `tempdb 缓存`。我们发现可以保存临时表的元数据，包括一些针对重复模式的页。你可以在 [`https://docs.microsoft.com/sql/relational-databases/databases/tempdb-database?#performance-improvements-in-tempdb-for-sql-server`](https://docs.microsoft.com/sql/relational-databases/databases/tempdb-database?#performance-improvements-in-tempdb-for-sql-server) 查看更多关于这个概念的信息。即使有了 `tempdb 缓存`，它也没有消除问题，而且并非所有 tempdb 场景都能使用缓存。

在 SQL Server 2016 中，我们通过包含一个默认配置（在安装时创建与检测到的 CPU 数量相匹配的文件，最多 8 个）来帮助客户确保为 tempdb 创建多个文件。我们还通过将 `跟踪标志 1117 和 1118` 转变为 `ALTER DATABASE` 选项并将其设为 tempdb 的默认设置，从而消除了对它们的需要。

工作负载持续对 tempdb 施加压力，我们仍然看到问题存在。我们在 SQL Server 2019 中引入了两项重大改进来帮助解决闩锁问题：

-   `PFS` 页现在将不再有闩锁问题。我们找到了即使在共享闩锁下也能更新 `PFS` 页的方法。
-   我们引入了 tempdb 元数据优化作为一个新的服务器选项。启用此选项后（需要重启服务器），我们会将 tempdb 中的关键系统表转换为内存优化表。内存优化表是“无锁的”，因此对这些表的任何读写操作都不会遇到闩锁争用。

改进是巨大的。有了这两项增强功能，我看到许多工作负载不再因闩锁争用而出现显著问题。但核心问题偶尔仍能看到，即 `SGAM` 和 `GAM` 页上的闩锁争用。当我第一次看到 `SGAM` 争用继续发生时，我感到困惑，因为我们已经将 tempdb 改为始终使用统一区。然而，有一个页仍然必须为临时表或索引分配，而且它不在统一区内，那就是 `IAM` 页，它跟踪对象的分配情况。

#### 为何现在可以“免维护”？

当我看到项目 Dallas 的最初一套引擎改进时，我非常高兴地了解到 `GAM` 和 `SGAM` 的并发性得到了增强，其方式与 SQL Server 2019 中 `PFS` 页的增强相同。

现在，tempdb 在闩锁问题方面可以真正做到“免维护”了。

请考虑以下几点：

-   `PFS`、`GAM` 和 `SGAM` 闩锁争用完全消失。
-   如果你执行此语句 `ALTER SERVER CONFIGURATION SET MEMORY_OPTIMIZED TEMPDB_METADATA = ON` 并重新启动 SQL Server，关键系统表将是内存优化的，不会遇到闩锁争用。

注意：tempdb 闩锁问题从未与缓慢的 I/O 有关。但是，你应该始终通过将 tempdb 数据和日志文件放在与用户数据不同的驱动器上，并且最好放在 SSD 上，来确保消除 tempdb 的任何 I/O 问题。如果你的 tempdb 存在 I/O 问题，通常表现为 `PAGELATCH_IO`、`IO_COMPLETION` 和 `WRITELOG`（tempdb 日志）等待类型。



