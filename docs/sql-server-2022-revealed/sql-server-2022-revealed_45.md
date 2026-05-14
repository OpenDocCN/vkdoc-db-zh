# SQL Server 2022 新特性概览

## 内存 OLTP 内存管理

我们在 SQL Server 2022 中进行了增强，以改进针对内存 OLTP 操作的内存管理。这些改进大多是为了避免内存不足（OOM）场景的具体优化，不需要您进行任何操作；它们已内置在引擎中。

我们还在 SQL Server 2019 中引入了一个新的系统存储过程，该过程在 SQL Server 2022 中同样可用，名为`sys.sp_xtp_force_gc`。此过程以数据库名称作为输入，用于按需释放内存优化表数据的未使用内存，而无需内存压力。

## 自动删除统计信息

这里有一个有趣的增强功能，用于解决一个长期存在的问题。假设 SQL Server 基于某个查询自动在列上创建了统计信息，或者第三方软件程序在列上创建了统计信息。如果您随后尝试删除这些统计信息所基于的列（或进行其他影响该列的架构更改），可能会遇到如下错误：

```
Msg 5074, Level 16, State 1, Line 20
The statistics ' is dependent on column ''.
```

您需要先去删除统计信息。现在，您可以使用`WITH AUTO_DROP = ON`选项创建统计信息。在上述场景中，统计信息将被自动删除，而不会阻止架构更改。默认情况下，自动创建的统计信息未启用自动删除。

## 可恢复的添加表约束

在 SQL Server 2017 和 2019 中，我们增加了使用*可恢复*选项创建和重建索引的能力。这意味着如果在索引构建过程中发生错误（或您暂停它），我们可以从“暂停的地方”恢复，而不必从头开始。

这对于需要很长时间的索引构建来说是一个相当不错的功能。您可以在 [`https://docs.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations?#resumable-index-considerations`](https://docs.microsoft.com/sql/relational-databases/indexes/guidelines-for-online-index-operations?#resumable-index-considerations) 阅读更多关于可恢复索引的内容。

然而，我们意识到许多索引是作为约束添加的，因此我们增强了`ALTER TABLE ADD CONSTRAINT`的语法以支持可恢复操作。您可以在 [`https://docs.microsoft.com/sql/relational-databases/security/resumable-add-table-constraints`](https://docs.microsoft.com/sql/relational-databases/security/resumable-add-table-constraints) 阅读确切的语法。

## 新的等待类型

随着 SQL Server 每个新版本的发布，我们都会添加新的等待类型作为构建新功能的一部分。在 SQL Server 2022 与 SQL Server 2019 之间，我发现了大约 300 个新的等待类型。

在审查这些新类型时，我的一个观察是，SQL Server 2022 中新增了`PREEMPTIVE_*`等待类型。具有此名称的等待类型指示我们代码中的某些位置，工作线程被切换到“抢占”模式，不再属于 SQLOS 调度的一部分。这通常是因为我们正在调用 API 或运行无法控制线程调度的代码。两个指示抢占调度的新等待类型是`PREEMPTIVE_SYNAPSESTREAMING_HTTP_EVENT_WAIT`和`PREEMPTIVE_AAD_HTTP_EVENT_WAIT`，它们可能在使用 Synapse Link 或 Azure Active Directory (AAD) 认证时出现。

## 新的扩展事件

几乎每个新功能都有一组新的扩展事件，您已经在第 4 章和第 5 章中看到了一些与 IQP 功能相关的事件，本章中也有几个。

我杰出的同事兼老朋友 Robert Dorr 告诉我一个有趣的新事件，它与某个功能无关，而是他添加的，用于帮助进行查询性能故障排除和调试。该事件的名称是`query_abort`（它是一个调试通道事件）。此事件包括会话、输入缓冲区，甚至调用堆栈，并在查询因任何原因被中止时触发。如果您看到查询被取消、中止或超时，并且不确定原因，这个新事件可能会有所帮助。

## 行业验证的引擎

#sqllegend David Campbell 曾经告诉我，*“Bob，如果我们不在引擎中创新，我们就没有产品。”* 我想从本章中您可以看到，我们在安全性、可扩展性和可用性方面提供了大量的创新。回想一下您在本章中看到的内容：仅举几例，如 SQL Server 的分类账、“免触碰”的 tempdb 和包含的可用性组。

我的同事 David Pless 可能比我们团队中任何人都更多地参与了新的 SQL Server 2022 引擎功能。我请他分享了他的看法：

*我一直在思考围绕我们在本版本中所完成工作的主题，特别是回顾之前的版本。在 SQL Server 2016 中，它是“It's just faster”，我们在 SQL Server 2017 中说它是“all about choice”。在本版本中，它是“return to performance”，但不仅如此——我们所做的是对可扩展性做出坚定承诺。对可扩展性的承诺不仅仅是单一功能，而是多个功能，使 SQL Server 2022 能够充分利用我们最大的系统来运行最关键的工作负载。我们通过缓冲池并行扫描实现了这一点，使客户能够扩展到可用的大型内存服务器，并获得他们对 SQL Server 所期望的完整性能。我们通过并发 GAM 和 SGAM 更新实现了这一点，将 tempdb 性能提升到一个因子，以至于我们甚至质疑是否仍然需要近四分之一世纪以来一直有效的 tempdb 最佳实践。而且这两项重大改进都是默认开启的——您只需将数据放在 SQL Server 2022 上，让可扩展的 SQL Server 引擎完成剩下的工作。还有更多。加速数据库恢复改进使我们能够比以往更快地恢复数据库。此外，我们与 Intel 的合作引入了一个全新的功能，不仅帮助 SQL Server 提高备份容量和吞吐量——而且通过 Intel QuickAssist Technology 硬件，我们可以通过将备份过程的计算开销从主机 SQL Server 系统卸载下来，来保护主机机器的 CPU。我为这个团队感到骄傲；我为我们在这个版本中取得的成就感到骄傲。*



