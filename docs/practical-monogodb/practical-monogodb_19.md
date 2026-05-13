# 10.2.2 操作

这些模式针对读取性能进行了优化。

#### 10.2.2.1 查看帖子

由于 `social.posts` 和 `user.wall` 集合针对在瞬间渲染新闻流或墙贴进行了优化，因此查询相当直接。这两个集合具有类似的模式，因此获取操作可以由相同的代码支持。以下是相同的伪代码。该函数接受以下参数作为参数：

*   需要查询的集合。
*   需要查看其数据的用户。
*   月份是一个可选参数；如果指定，它应列出日期小于或等于指定月份的所有帖子。

```
Function Fetch_Post_Details (Parameters: CollectionName, View_User_ID, Month)
SET QueryDocument to {"User_id": View_User_ID}
IF Month IS NOT NULL
APPEND Month Filter ["Month":{"$lte":Month}] to QueryDocument
Set O_Cursor = (resultset of the collection after applying the QueryDocument filter)
Set Cur = (sort O_Cursor by "month" in reverse order)
while records are present in Cur
Print record
End while
End Function
```

上述函数以倒序时间顺序检索给定用户墙或新闻流上的所有帖子。

在渲染帖子时，需要应用某些检查。以下是其中一些。

首先，当用户查看自己页面时，在渲染墙上的帖子时，你需要检查该帖子是否可以在其自己的墙上显示。用户墙包含他发布的帖子或他关注的用户的帖子。以下函数接受两个参数：墙所属的用户和正在渲染的帖子：

```
function Check_VisibleOnOwnWall (Parameters: user, post)
While Loop_User IN user.Circles List
If post by = Loop_User
return true
else
return false
end while
end function
```

上述循环遍历在 `user.profile` 集合中指定的好友圈，如果提到的帖子是由列表中的用户发布的，则返回 true。

此外，你还需要处理用户屏蔽列表中的用户：

```
function ReturnBlockedOrNot(user, post)
if post by user id not in user blocked list
return true
else
return false
endfunction
```

当用户查看另一个用户的墙时，你还需要处理权限检查：

```
Function visibleposts(parameter user, post)
if post circles is public
return true
If post circles is public to all followed users
Return true
set listofcircles = followers circle whose user_id is the post's by id.
if listofcircles in post's circles
return true
return false
end function
```

此函数首先检查帖子的圈组是否为公开。如果是公开的，该帖子将对所有用户显示。如果帖子的圈组未设置为公开，但如果他/她正在关注该用户，则该帖子将显示给该用户。如果两种情况都不成立，则它会检查所有关注登录用户的用户的圈组。如果圈组列表在帖子的圈组列表中，这意味着用户位于接收该帖子的圈组中，因此该帖子将可见。如果两个条件都不满足，该帖子将对该用户不可见。

为了获得更好的性能，你需要在 `social.posts` 和 `user.wall` 集合中对 `user_id` 和 `month` 建立索引。



#### 10.2.2.2 创建评论

要为给定帖子创建一条包含给定文本的用户评论，你需要执行类似下面的代码：

```
Function postcomment(
    Parameters: commentedby, commentedonpostid, commenttext)
    Set commentedon to current datetime
    Set month to month of commentedon
    Set comment document as {"by": {id: commentedby[id], "Name": commentedby["name"]}, "ts": commentedon, "text": commenttext}
    Update user.posts collection. Push comment document.
    Update user.walls collection. Push the comment document.
    Increment the comments_shown in user.walls collection by 1.
    Update social.posts collection. Push the comment document.
    Increment the comments_shown counter in social.posts collection by 1.
End function
```

由于你在两个依赖集合（`user.wall` 和 `social.posts` 集合）中最多只显示三条评论，你需要定期运行以下更新语句：

```
Function MaintainComments
    SET MaximumComments = 3
    Loop through social.posts
        If posts.comments_shown > MaximumComments
            Pop the comment which was inserted first
            Decrement comments_shown by 1
        End if
    Loop through user.wall
        If posts.comments_shown > MaximumComments
            Pop the comment which was inserted first
            Decrement comments_shown by 1
        End if
    End loop
End Function
```

为了快速执行这些更新，你需要为 `posts.id` 和 `posts.comments_shown` 创建索引。

#### 创建新帖

此代码中的基本操作序列如下：

帖子首先被保存到“记录系统”，即 `user.posts` 集合。接下来，帖子被更新到 `user.wall` 集合。最后，根据帖子所属的圈子，更新所有圈子内用户的 `social.posts` 集合。

```
Function createnewpost
    (parameter createdby, posttype, postdetail, circles)
    Set ts = current timestamp.
    Set month = month of ts
    Set post_document = {"ts": ts, "by":{id:createdby[id], name: createdby[name]}, "circles":circles, "type":posttype, "details":postdetails}
    Insert post_document into users.post collection
    Append post_document into user.walls collection
    Set userlist = all users who’s circled in the post based on the posts circle and the posted user id
    While users in userlist
        Append post_document to users social.posts collection
    End while
End function
```

### 10.2.3 分片

可以通过对上述四个集合进行分片来实现扩展。由于 `user.profile`、`user.wall` 和 `social.posts` 包含特定于用户的文档，`user_id` 是这些集合的完美分片键。`_id` 是 `users.post` 集合的最佳分片键。

## 10.3 本章小结

在本章中，你通过两个用例探讨了如何使用 MongoDB 来解决特定问题。下一章，我们将列出 MongoDB 的局限性以及它不太适用的用例。

## 11. MongoDB 的局限性

> “开始使用一个新数据库时，你也应该了解其局限性，以便更好地使用它。”

在本章中，我们将列出 MongoDB 的局限性以及它不太适用的用例。

### 11.1 MongoDB 占用空间过大（适用于 MMAPv1）

让我们从磁盘空间问题开始。MongoDB（使用 MMAPv1 存储引擎）占用空间太大；换句话说，数据目录文件比数据库的实际数据大。

这是由于预分配的数据文件造成的。这是为了防止文件系统碎片化而设计的。

数据目录中的文件被命名为 `<dbname>.0`、`<dbname>.1` 等。`mongod` 分配的第一个文件大小为 64MB；之后所有文件的大小都以 2 的倍数增长，因此第二个文件为 128MB，第三个文件为 256MB，依此类推，直到达到 2GB，之后所有文件大小都为 2GB。虽然空间在创建时已分配给数据文件，但有些文件可能有 90% 是空的。这种未使用的分配空间在较大的数据库中通常占比较小。

*   可以使用 `-- noprealloc` 选项禁用此选项。但是，不建议在生产环境中使用，它应仅用于测试以及频繁调用 drop database 的小型数据集。
*   Oplog：如果 `mongod` 是副本集成员，那么数据目录中将有一个名为 `oplog.rs` 的文件。此文件存在于 `local` 数据库中，是一个预分配的固定集合。在 64 位安装中，此文件的分配默认为磁盘空间的约 5%。
*   日志：日志文件也包含在数据目录中，用于在 MongoDB 将写入操作应用到数据库之前，先将这些写操作存储在磁盘上。
*   MongoDB 为日志预分配 3GB 的数据，这超出了实际数据库的大小，使其不适合小型安装。解决方法是在你的命令行标志或 `/etc/mongod.conf` 文件中使用 `–smallflags`，直到你运行在有足够磁盘空间的环境中。但这个特性使其不适合小型安装。
*   空记录：当文档或集合被删除时，空间永远不会返回给操作系统；相反，MongoDB 会维护一个这些空记录的列表，以便重用。

要回收这些已删除的空间，可以使用 `compact` 或 `repairDatabase` 选项，但请注意，这两个选项都需要额外的磁盘空间才能运行。

注意

WiredTiger 存储引擎不存在此限制。相反，由于数据文件的压缩，存储大小减少了 50%。此外，一旦集合被删除，磁盘空间会自动回收，这与上述 MMAPv1 存储引擎不同。

## 11.2 内存问题（适用于 MMAPv1 存储引擎）

在 MongoDB 中，内存是通过内存映射整个数据集来管理的。它允许操作系统控制内存映射并分配最大量的 RAM。结果是性能并非最优，并且无法有效推断内存使用情况。

索引是内存密集型的；换句话说，索引会占用大量 RAM。由于这些是 B 树索引，定义许多索引可能导致系统资源消耗过快。其后果是内存会在需要时自动分配。在共享环境中运行数据库更为棘手。总的来说，与所有数据库服务器一样，最好在专用服务器上运行 MongoDB。

### 11.3 32 位与 64 位

MongoDB 提供两个版本：32 位和 64 位。

由于 MongoDB 使用内存映射文件，32 位版本仅限存储约 2GB 的数据。如果你需要存储更多数据，应使用 64 位版本。

从 3.0 版本开始，MongoDB 不再为 32 位版本提供商业支持。此外，32 位版本的 MongoDB 不支持 WiredTiger 存储引擎。

### 11.4 BSON 文档

本节介绍 BSON 文档的限制。

*   大小限制：与其他数据库一样，文档中可存储的内容有限制。当前版本支持最大 16MB 大小的文档。此最大值确保单个文档在传输过程中不会占用过多的 RAM 或过多的带宽。
*   嵌套深度限制：在 MongoDB 中，BSON 文档的嵌套层级不超过 100 层。
*   字段名：如果你存储 1000 个键为 “col1” 的文档，那么该键会在数据集中存储那么多次。虽然 MongoDB 支持任意文档，但实际上大多数字段名是相同的。保持简短的字段名被认为是优化空间使用的良好实践。



## 11.5 命名空间限制

请注意从命名空间角度出发的以下限制。

*   **命名空间长度**：每个命名空间的长度（包括集合和数据库名称）必须小于 123 字节。
*   **命名空间文件大小**（适用于 MMAPv1 存储引擎）：命名空间文件大小不能大于 2047MB。默认大小为 16MB；但是，这可以通过 `nssize` 选项进行配置。
*   **命名空间数量**（适用于 MMAPv1 存储引擎）：命名空间数量 = （命名空间文件大小 / 628）。一个 16MB 的命名空间文件将支持大约 24,000 个命名空间。

> **注意**
>
> WiredTiger 存储引擎不存在此类限制。

## 11.6 索引限制

本节介绍 MongoDB 中索引的限制。

*   **索引大小**：索引项不能大于 1024 字节。
*   **每个集合的索引数量**：每个集合最多允许 64 个索引。
*   **索引名称长度**：默认情况下，索引名称由字段名称和索引方向组成。包括命名空间（即数据库和集合名称）在内的索引名称不能大于 128 字节。如果默认索引名称变得过长，您可以向 `ensureIndex()` 辅助方法显式指定一个索引名称。
*   **分片集合中的唯一索引**：只有当完整分片键作为唯一索引的前缀包含时，才在分片间支持该唯一索引；否则，该唯一索引不在分片间支持。在这种情况下，唯一性是在整个键上强制执行的，而不是在单个字段上。
*   **复合索引中的索引字段数量**：不能超过 31 个字段。

## 11.7 固定集合限制 - 固定集合中的最大文档数

如果使用 `max` 参数来指定固定集合中的最大文档数，则该值不能超过 2³² 个文档。但是，如果不使用此参数，则对文档数量没有限制。

## 11.8 分片限制

分片是在分片间拆分数据的机制。以下部分介绍了在处理分片时需要注意的限制。

### 11.8.1 尽早分片以避免问题

使用分片键，数据被拆分成块，然后自动分布到各个分片上。但是，如果分片实施较晚，可能会导致服务器变慢，因为块的拆分和迁移需要时间和资源。

一个简单的解决方案是使用诸如 MongoDB Cloud Manager 之类的工具监控您的 MongoDB 实例容量（刷新时间、锁百分比、队列长度和故障是很好的度量指标），并在达到估计容量的 80% 之前进行分片。

### 11.8.2 分片键无法更新

一旦文档插入到集合中，分片键就无法更新，因为 MongoDB 使用分片键来确定文档应路由到哪个分片。如果要更改文档的分片键，建议的解决方案是在更改完成后删除该文档并重新插入。

### 11.8.3 集合分片限制

集合应在达到 256GB 之前进行分片。

### 11.8.4 选择正确的分片键

选择正确的分片键非常重要，因为一旦选定就很难更正。

> **注意**
>
> 什么是错误的分片键完全取决于应用程序。假设应用程序是一个新闻源；选择时间戳字段作为分片键将是错误的分片键，因为这将最终只向一个分片插入、查询和迁移数据，而不是整个集群。如果需要更正分片键，通常使用的过程是转储并恢复集合。

## 11.9 安全限制

安全对于数据库来说是一个重要问题。让我们从安全角度来看 MongoDB 的限制。

### 11.9.1 默认无身份验证

虽然默认情况下未启用身份验证，但它得到完全支持，并且可以轻松启用。

### 11.9.2 与 MongoDB 的通信流量未加密

默认情况下，与 MongoDB 的通信流量是未加密的。在公共网络上运行时，请考虑对通信进行加密；否则可能对您的数据构成威胁。公共网络上的通信可以使用 MongoDB 的 SSL 支持版本进行加密，该版本仅在 64 位版本中可用。

## 11.10 读写限制

以下部分涵盖了重要限制。

### 11.10.1 大小写敏感查询

默认情况下，MongoDB 是大小写敏感的。

例如，以下两个命令将返回不同的结果：`db.books.find({name: 'PracticalMongoDB'})` 和 `db.books.find({name: 'practicalmongodb'})`。您应确保知道数据以何种大小写存储。虽然可以使用正则表达式搜索，如 `db.books.find({name: /practicalmongodb/i})`，但这并不理想，因为它们相对较慢。

### 11.10.2 类型敏感字段

由于 MongoDB 中的文档没有强制的模式，它无法知道您是否犯了错误。您必须确保对数据使用正确的类型。

### 11.10.3 无 JOIN

MongoDB 中不支持 JOIN。如果需要从多个集合中检索数据，必须进行多次查询。但是，您可以重新设计模式，将相关数据保存在一起，以便在单个查询中检索信息。

### 11.10.4 事务

MongoDB 仅支持单文档原子性。由于写操作可以修改多个文档，因此该操作不是原子的。但是，您可以使用隔离操作符来隔离影响多个文档的写操作。

#### 11.10.4.1 副本集限制 - 副本集成员数量

副本集用于确保 MongoDB 中的数据冗余。一个成员充当主成员，其余成员充当次成员。由于 MongoDB 投票的工作方式，您必须使用奇数个成员。

这是因为一个节点需要获得多数票才能成为主节点。如果您使用偶数个节点，最终会陷入僵局，因为没有一个成员能获得多数票，因此无法选出主节点。在这种情况下，副本集将变为只读。

您可以使用仲裁者来打破这种僵局。它们可以帮助支持故障转移并节省成本。要了解更多关于副本集的功能，请参考第 7 章。

## 11.11 MongoDB 不适用的范围

MongoDB 不适用于以下场景：

*   高度事务性的系统，如会计或银行系统。传统的关系型数据库管理系统仍然更适用于需要大量原子性复杂事务的应用程序。
*   传统的商业智能应用程序，其中针对特定问题的 BI 数据库会生成高度优化的查询。对于此类应用程序，数据仓库可能是更合适的选择。
*   需要复杂 SQL 查询的应用程序。
*   MongoDB 不支持事务操作，因此银行系统肯定不能使用它。

## 11.12 总结

在本章中，您了解了 MongoDB 的限制以及它不太适合的使用场景。

在下一章中，我们将介绍 MongoDB 的操作指南。



