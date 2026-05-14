# 优化索引重建与整理

在 SQL Server 2005 及更新版本中，重建索引也会压缩大型对象（LOB）页。您可以通过设置 `LOB_COMPACTION = OFF` 来选择不执行此操作。如果您不担心存储空间，但关心索引重组所需的时间，那么关闭此选项可能是可取的。

在创建索引时使用 `PAD_INDEX` 设置，它决定了索引中间页上要保留多少可用空间，这有助于处理页拆分。在索引重建过程中会考虑这一点，除非您另有指定，否则新页面将被设置回您在索引创建时确定的原始值。我几乎从未见过这对大多数系统产生重大影响。您需要在您的系统上进行测试，以确定它是否能提供帮助。

如果您没有另行指定，默认行为是跨所有分区对所有索引进行碎片整理。如果想控制该过程，您只需指定要重建哪个分区。

如前所述，`ALTER INDEX REBUILD` 技术能有效减少碎片。您也可以使用它在一条语句中重建表的所有索引。
```sql
ALTER INDEX ALL ON dbo.Test1 REBUILD;
```
虽然这是最有效的碎片整理技术，但它确实有一些开销和限制。

*   **阻塞**：与之前的两种索引重建技术类似，`ALTER INDEX REBUILD` 会在系统中引入阻塞。它会阻止其他所有尝试访问该表（或该表的任何索引）的查询。它也可能被那些查询阻塞。您可以使用 `ONLINE INDEX REBUILD` 来减少这种情况。
*   **事务回滚**：由于 `ALTER INDEX REBUILD` 在操作中是完全原子性的，如果它在完成前停止，那么到那时为止执行的所有碎片整理操作都将丢失。您可以使用 `ONLINE` 关键字运行 `ALTER INDEX REBUILD`，这将减少锁定机制，但会增加重建索引所需的时间。

在 Azure SQL Database 中引入并在 SQL Server 2017 中可用，您现在有能力重启索引重建操作。您可以重启失败的索引重建，或者可以暂停重建操作，稍后再重启。为此，您还必须使用 `ONLINE` 重建选项。`ONLINE` 选项显著减少了与重建操作相关的阻塞。要重建一个既是 `ONLINE` 又是 `RESUMABLE` 的索引，您必须在命令中指定所有这些内容。
```sql
ALTER INDEX i1 ON dbo.Test1 REBUILD WITH (ONLINE=ON, RESUMABLE=ON);
```
这将运行索引重建操作，直到完成或直到您发出以下命令：
```sql
ALTER INDEX i1 ON dbo.Test1 PAUSE;
```
这将暂停 `ONLINE` 重建操作，并且相关的表和索引将保持可访问，没有任何阻塞。要重启操作，请使用：
```sql
ALTER INDEX i1 ON dbo.Test1 RESUME;
```

### 执行 ALTER INDEX REORGANIZE 语句

对于行存储索引，`ALTER INDEX REORGANIZE` 无需重建索引即可减少索引的碎片。它通过按索引键的逻辑顺序重新排列索引的现有叶级页来减少外部碎片。它压缩页面内的行，减少内部碎片，并丢弃产生的空页。此技术不使用任何新页面进行碎片整理。

对于列存储索引，`ALTER INDEX REORGANIZE` 将确保列存储索引内的增量存储（deltastore）被清理，并且所有逻辑删除都得到处理。它在保持索引在线且可访问的同时执行此操作。这将确保索引被碎片整理。此外，您可以选择强制压缩所有行组。其功能类似于运行 `ALTER INDEX REBUILD`，但与 `REBUILD` 过程不同，它在操作期间继续保持索引在线。因此，对于列存储索引，更推荐使用 `ALTER INDEX REORGANIZE`。

为了避免与 `ALTER INDEX REBUILD` 相关的阻塞开销，此技术使用非原子的在线方法。它在执行步骤时，会请求少量锁并持有很短时间。一旦每个步骤完成，它就会释放锁并进入下一步骤。当尝试访问一个页面时，如果发现该页面正在被使用，它会跳过该页面并不再返回。这允许其他查询与 `ALTER INDEX REORGANIZE` 操作一起在表上运行。此外，如果此操作中途停止，则到那时为止执行的所有碎片整理步骤都将被保留。

对于行存储索引，由于 `ALTER INDEX REORGANIZE` 不使用任何新页面来重新排序索引，并且它会跳过被锁定的页面，因此此方法提供的碎片整理量通常少于 `ALTER INDEX REBUILD`。要观察 `ALTER INDEX REORGANIZE` 与 `ALTER INDEX REBUILD` 相比的相对有效性，请使用上一节关于 `ALTER INDEX REBUILD` 中的测试表进行重建。

使用之前的脚本重建碎片化的行存储表。要减少聚集行存储索引的碎片，请按如下所示使用 `ALTER INDEX REORGANIZE`：
```sql
ALTER INDEX i1 ON dbo.Test1 REORGANIZE;
```
图 14-19 显示了 `sys.dm_db_index_physical_stats` 的输出结果。

![../images/323849_5_En_14_Chapter/323849_5_En_14_Fig19_HTML.jpg](img/323849_5_En_14_Fig19_HTML.jpg)

**图 14-19** ALTER INDEX REORGANIZE 的结果

从输出中可以看到，正如前面章节所示，`ALTER INDEX REORGANIZE` 减少碎片的效果不如 `ALTER INDEX REBUILD`。对于高度碎片化的索引，`ALTER INDEX REORGANIZE` 操作可能比重建索引花费更长的时间。此外，如果一个索引跨越多个文件，`ALTER INDEX REORGANIZE` 不会在文件之间迁移页面。然而，使用 `ALTER INDEX REORGANIZE` 的主要优点是它允许其他查询同时访问表（或索引）。

为了查看列存储索引碎片整理的结果，让我们使用本章前面“Columnstore Overhead”部分中已经碎片化的列存储索引。
```sql
ALTER INDEX ClusteredColumnStoreTest ON dbo.bigTransactionHistory REORGANIZE;
```
`REORGANIZE` 语句的结果如图 14-20 所示。

![../images/323849_5_En_14_Chapter/323849_5_En_14_Fig20_HTML.jpg](img/323849_5_En_14_Fig20_HTML.jpg)

**图 14-20** 在列存储索引上不带压缩执行 REORGANIZE 的结果



