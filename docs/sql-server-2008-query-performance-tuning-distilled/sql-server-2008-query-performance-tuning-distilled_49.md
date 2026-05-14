# 死锁日志条目说明

#### 死锁信息开始
`<deadlock>` `<victim-list>` 标签标志着死锁信息的开始。它首先列出受害进程。

#### 受害者进程
`<victimProcess id="process179b3f3b468" />` 表示被选为死锁牺牲品的进程的物理内存地址。

#### 进程列表
`<process-list>` 定义了死锁牺牲品的进程。可能不止一个。

`<process id="process179b3f3b468" taskpriority="0" logused="400" waitresource="KEY: 6:72057594050904064 (4ab5f0d47ad5)" waittime="3703" ownerId="179351993" transactionname="user_transaction" lasttranstarted="2018-03-25T11:28:18.140" XDES="0x179b4bdc490" lockMode="U" schedulerid="1" kpid="2168" status="suspended" spid="53" sbid="0" ecid="0" priority="0" trancount="2" lastbatchstarted="2018-03-25T11:29:05.377" lastbatchcompleted="2018-03-25T11:29:05.363" lastattention="1900-01-01T00:00:00.363" clientapp="Microsoft SQL Server Management Studio - Query" hostname="WIN-8A2LQANSO51" hostpid="7028" loginname="WIN-8A2LQANSO51\Administrator"` `isolationlevel="已提交读 (2)"` `xactid="179351993" currentdb="6" lockTimeout="4294967295" clientoption1="671090784" clientoption2="390200">`

这是关于被选为死锁牺牲品的会话的所有信息。注意高亮显示的 `isolationlevel`，它是帮助识别死锁根本原因的关键。

```
<executionStack>
    <frame procname="adhoc" line="1" stmtend="240" sqlhandle="0x02000000d0c7f31a30fb1ad425c34357fe8ef6326793e7aa0000000000000000000000000000000000000000">unknown</frame>
    <frame procname="adhoc" line="1" stmtend="240" sqlhandle="0x02000000e7794d32ae3080d4a3217fdd3d1499f2e322d46e0000000000000000000000000000000000000000">unknown</frame>
</executionStack>
<inputbuf>
UPDATE  Purchasing.PurchaseOrderDetail
SET     OrderQty = 2
WHERE   ProductID = 448
    AND PurchaseOrderID = 1255;
</inputbuf>
```

`<process id="process179b7b63468" taskpriority="0" logused="9800" waitresource="KEY: 6:72057594050969600 (4bc08edebc6b)" waittime="44833" ownerId="179352664" transactionname="user_transaction" lasttranstarted="2018-03-25T11:28:24.163" XDES="0x179bc2a8490" lockMode="U" schedulerid="1" kpid="3784" status="suspended" spid="63" sbid="0" ecid="0" priority="0" trancount="2" lastbatchstarted="2018-03-25T11:28:23.960" lastbatchcompleted="2018-03-25T11:28:23.920" lastattention="1900-01-01T00:00:00.920" clientapp="Microsoft SQL Server Management Studio - Query" hostname="WIN-8A2LQANSO51" hostpid="7028" loginname="WIN-8A2LQANSO51\Administrator"` `isolationlevel="已提交读 (2)"` `xactid="179352664" currentdb="6" lockTimeout="4294967295" clientoption1="673319008" clientoption2="390200">`

这是定义的第二个进程。

`<frame procname="AdventureWorks2017.Purchasing.uPurchaseOrderDetail" line="39" stmtstart="2732" stmtend="3830"` `sqlhandle="0x0300060025999f1142d8ef0019a8000000000000000000000000000000000000000000000000000000000000"` `UPDATE [Purchasing].[PurchaseOrderHeader]`            `SET [Purchasing].[PurchaseOrderHeader].[SubTotal] =`                `(SELECT SUM([Purchasing].[PurchaseOrderDetail].[LineTotal])`                    `FROM [Purchasing].[PurchaseOrderDetail]`                    `WHERE [Purchasing].[PurchaseOrderHeader].[PurchaseOrderID]`                        `= [Purchasing].[PurchaseOrderDetail].[PurchaseOrderID])`            `WHERE [Purchasing].[PurchaseOrderHeader].[PurchaseOrderID]`                `IN (SELECT inserted.[PurchaseOrderID] FROM inserted</frame>`

你可以看到这是一个触发器，被称为 `procname`，`uPurchaseOrderDetail`。它具有高亮的 `sqlhandle`，这样你就可以从缓存或查询存储中检索它。它还显示了触发器的代码。

`<frame procname="adhoc" line="2" stmtstart="38" stmtend="278" sqlhandle="0x02000000352f5b347ab7d87fc940e4f04e534f1c825a28b40000000000000000000000000000000000000000">unknown</frame>`

```
BEGIN TRANSACTION
UPDATE  Purchasing.PurchaseOrderDetail
SET     OrderQty = 4
WHERE   ProductID = 448
    AND PurchaseOrderID = 1255;
```

这是批处理中的下一条语句以及被调用的代码。

#### 资源列表
`<resource-list>`
  `<keylock hobtid="72057594050904064" dbid="6" objectname="AdventureWorks2017.Purchasing.PurchaseOrderDetail" indexname="PK_PurchaseOrderDetail_PurchaseOrderID_PurchaseOrderDetailID" id="lock17992a41a00" mode="X" associatedObjectId="72057594050904064">`
    `<owner-list>`
      `<owner id="process179b7b63468" mode="X" />`
    `</owner-list>`
    `<waiter-list>`
      `<waiter id="process179b3f3b468" mode="U" requestType="wait" />`
    `</waiter-list>`
  `</keylock>`
  `<keylock hobtid="72057594050969600" dbid="6" objectname="AdventureWorks2017.Purchasing.PurchaseOrderHeader" indexname="PK_PurchaseOrderHeader_PurchaseOrderID" id="lock179b7a1a880" mode="X" associatedObjectId="72057594050969600">`
    `<owner-list>`
      `<owner id="process179b3f3b468" mode="X" />`
    `</owner-list>`
    `<waiter-list>`
      `<waiter id="process179b7b63468" mode="U" requestType="wait" />`
    `</waiter-list>`
  `</keylock>`
`</resource-list>`

这些是导致冲突的对象。其中包含了 `Purchasing.PurchaseOrderDetail` 表中主键的定义。你可以看到之前的代码中哪个进程拥有哪个资源。你还可以看到定义等待进程的信息。这就是你判断问题所在所需的一切。


这些信息比图形化死锁图提供的那套简洁数据要稍微难读一些。然而，它包含的信息类别是相似的，只是更为详细。你可以看到，在靠近底部的位置以粗体突出显示了一个与死锁关联的键的定义。你还可以看到，在它之前，执行计划的文本可以通过扩展事件工具的 XML 输出获得，就像死锁图一样。无论使用哪种方式，你都能获得隔离死锁原因所需的一切信息。

跟踪标志 1222 收集的信息几乎在所有方面都与 XML 数据完全相同。主要区别在于格式和位置。1222 的输出位于 SQL Server 错误日志中，并且是文本格式，而非整洁的 XML 格式。而跟踪标志 1204 收集的信息则与前两者完全不同，并且提供的细节远没有那么丰富。跟踪标志 1204 也更难以解读。基于所有这些原因，我建议你尽可能坚持使用扩展事件——如果不行，则使用跟踪标志 1222——来捕获死锁数据。你还有一个默认情况下捕获多种事件（包括死锁）的 `system_health` 会话。如果你尚未准备好捕获此类信息，这是一个很好的资源。只需记住，它只保留四个 5MB 的文件在线。当这些文件填满时，最旧文件中的数据将会丢失。根据系统中的事务数量以及可能填满这些文件的死锁或其他事件的数量，你可能只能访问到近期的数据。此外，如前所述，由于 `system_health` 会话使用环形缓冲区来捕获事件，你可能会遇到一些事件丢失的情况，因此你的死锁事件也可能丢失。

这个示例演示了一个经典的循环引用。虽然不那么显而易见，但死锁是由 `Purchasing.PurchaseOrderDetail` 表上的一个触发器引起的。当 `Purchasing.PurchaseOrderDetail` 表上的 `Quantity` 被更新时，它会尝试更新 `Purchasing.PurchaseOrderHeader` 表。当前两个查询分别在开放事务中运行时，这只是一个阻塞情况。第二个查询正在等待第一个查询完成，以便它也能更新 `Purchasing.PurchaseOrderHeader` 表。但是当引入第三个查询（即第一个事务中的第二个查询）时，就形成了循环引用。解决它的唯一方法是终止其中一个进程。

在继续之前，请确保回滚所有未完成的事务。

现阶段有个显而易见的问题：你能避免这个死锁吗？如果答案是“能”，那么如何避免？

## 避免死锁

避免死锁场景的方法取决于死锁的性质。以下是一些可用于避免死锁的技术：

*   按相同的物理顺序访问资源。
*   减少访问的资源数量。
*   最小化锁争用。
*   优化查询。

### 按相同的物理顺序访问资源

避免死锁最常用的技术之一是确保每个事务都以相同的物理顺序访问资源。例如，假设两个事务需要访问两个资源。如果每个事务都以相同的物理顺序访问资源，那么第一个事务将成功获取资源上的锁，而不会被第二个事务阻塞。第二个事务在尝试获取第一个资源上的锁时将被第一个事务阻塞。这将导致典型的阻塞场景，而不会导致循环阻塞和死锁。

如果资源没有按相同的物理顺序访问（如前面的死锁分析示例所示），则可能导致两个事务之间的循环阻塞。

*   事务 1：
    *   访问资源 1
    *   访问资源 2
*   事务 2：
    *   访问资源 2
    *   访问资源 1

在当前的死锁场景中，以下资源涉及死锁：

*   资源 1，`hobtid=72057594046578688`：这是 `Purchasing.PurchaseOrderDetail` 表上索引 `PK_PurchaseOrderDetail_PurchaseOrderId_PurchaseOrderDetailId` 中的索引行。
*   资源 2，`hobtid=72057594046644224`：这是 `Purchasing.PurchaseOrderHeader` 表上聚集索引 `PK_PurchaseOrderHeader_PurchaseOrderId` 中的行。

两个会话都试图访问该资源；不幸的是，它们访问键的顺序是不同的。

在使用诸如 nHibernate 和 Entity Framework 等工具生成的代码时，经常会在不同的查询中看到对象以不同的顺序被引用。你必须与开发团队合作，消除生成代码中的这类问题。

### 减少访问的资源数量

死锁至少涉及两个资源。一个会话持有第一个资源，然后请求第二个资源。另一个会话持有第二个资源，然后请求第一个资源。如果你能阻止会话（或至少其中一个）访问死锁涉及的资源之一，就可以防止死锁。你可以通过重新设计应用程序来实现这一点，而这在项目后期是开发人员强烈抵制的解决方案。然而，你可以考虑使用 SQL Server 的以下功能，而无需更改应用程序设计：

*   将非聚集索引转换为聚集索引。
*   为 `SELECT` 语句使用覆盖索引。

#### 将非聚集索引转换为聚集索引

如你所知，非聚集索引的叶级页与堆或聚集索引的数据页是分开的。因此，非聚集索引需要两个锁：一个用于基表（聚集索引或堆），一个用于非聚集索引本身。然而，在聚集索引的情况下，索引的叶级页与表的数据页是相同的；它只需要一个锁，这一个锁同时保护聚集索引和表，因为叶级页和数据页是相同的。与访问非聚集索引相比，这减少了同一查询需要访问的资源数量。但是，这完全取决于它是否是一个合适的聚集索引。聚集索引并非万能，将其应用于任何列并不一定有帮助。你仍然需要评估其适用性。

#### 为 SELECT 语句使用覆盖索引

你也可以使用覆盖索引来减少 `SELECT` 语句访问的资源数量。由于 `SELECT` 语句可以直接从覆盖索引本身获取所有内容，因此它不需要访问基表。否则，`SELECT` 语句需要同时访问索引和基表以检索所有需要的列值。使用覆盖索引可以阻止 `SELECT` 语句访问基表，使基表可以被另一个会话自由锁定。

### 最小化锁争用

你也可以通过避免对某个争用资源的锁请求来解决死锁。当资源仅用于读取数据时，可以这样做。修改资源总是会在资源上获取排他锁（X 锁）以维护资源的一致性；因此，在死锁情况下，识别那些只读访问的资源，并尽可能通过使用脏读功能来避免它们的锁请求。你可以使用以下技术来避免对争用资源的锁请求：

*   实现行版本控制。
*   降低隔离级别。
*   使用锁提示。

#### 实现行版本控制

与其尝试使用更严格的锁定方案来阻止对资源的访问，不如通过 `READ_COMMITTED_SNAPSHOT` 隔离级别或 `SNAPSHOT` 隔离级别来实现行版本控制。行版本控制隔离级别用于减少阻塞，如第 21 章所述。由于它们减少了阻塞（这是死锁的根本原因），因此也有助于解决死锁问题。通过以下 T-SQL 引入 `READ_COMMITTED_SNAPSHOT`，你可以在 tempdb 中获取行的一个版本，从而可能消除前面死锁场景中由锁引起的争用：

```sql
ALTER DATABASE AdventureWorks2017
SET READ_COMMITTED_SNAPSHOT ON;
```

这将允许任何必要的读取操作，而不会引起锁争用，因为读取的是数据的不同版本。行版本控制存在开销，特别是在 tempdb 中以及当需要从多个资源（而不仅仅是查询中使用的表或索引）中整合数据时。但是，增加 tempdb 开销与减少死锁和提高并发性所带来的好处之间的权衡可能是值得的。

#### 降低隔离级别

有时，`SELECT` 语句请求的（S）锁会导致循环阻塞的形成。你可以通过将包含 `SELECT` 语句的事务的隔离级别降低到 `READ COMMITTED SNAPSHOT` 来避免这种类型的循环阻塞。这将允许 `SELECT` 语句在不请求（S）锁的情况下读取数据，从而避免循环阻塞。你可能还会在游标周围看到这类问题，因为它们倾向于使用悲观并发。

同时检查连接是否将自己设置为 `SERIALIZABLE`。有时，在线连接字符串生成器会包含此选项，开发人员可能会无意中使用它。MSDTC 默认使用可序列化，但可以更改。

#### 使用锁定提示

我绝对不推荐这种方法。但是，你可以使用以下锁定提示来解决前面技术中展示的死锁：

*   `NOLOCK`
*   `READUNCOMMITTED`

与 `READ UNCOMMITTED` 隔离级别类似，`NOLOCK` 或 `READUNCOMMITTED` 锁定提示将避免给定会话请求的（S）锁，从而防止循环阻塞的形成。

锁定提示的作用是在查询级别上，并且仅限于应用它的表（及其索引）。`NOLOCK` 和 `READUNCOMMITTED` 锁定提示仅允许在 `SELECT` 语句以及 `INSERT`、`DELETE` 和 `UPDATE` 语句的数据选择部分中使用。

最小化锁争用的解决技术会引入脏读的副作用，这在每个事务中可能都是不可接受的。脏读可能涉及由于页面拆分和重新排列页面而导致的行丢失或行多余。因此，仅在数据质量要求不高的情况下使用这些解决技术。

#### 优化查询

从根本上讲，死锁是关于性能的。如果所有查询在资源争用可能发生之前就完成了执行，那么你就可以完全避免这个问题。

## 总结

正如你在本章中学到的，死锁是进程之间冲突阻塞的结果，并以错误号 1205 报告给应用程序。你可以通过使用各种资源收集死锁信息来分析死锁的原因，但扩展事件 `xml:deadlock_report` 可能是最好的。

你可以使用多种技术来避免死锁；哪种技术适用取决于参与会话执行的查询类型、所涉及资源上保持和请求的锁，以及管理所需隔离级别的业务规则。通常，你可以通过重新配置索引和事务隔离级别来解决死锁。然而，有时你可能需要重新设计应用程序或在发生死锁时自动重新执行事务。请记住，死锁的核心是一个性能问题，任何能让查询运行更快的方法都将有助于缓解（即使不能消除）查询中的死锁。

在下一章中，我将介绍游标的性能方面以及如何优化使用游标的成本开销。

