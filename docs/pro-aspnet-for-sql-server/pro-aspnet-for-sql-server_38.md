# 第七章：手动数据访问层

## 保存数据

接下来，用于通过插入或更新来保存链接的存储过程提出了一个更具挑战性的要求。该过程不仅要保存链接的核心详细信息，还必须获取 `Title` 和 `Url` 值，将它们存储到辅助表中，同时避免为相同的值创建重复记录。表的设计允许一条 `Favorite Link` 记录指向其他 `Favorite Link` 记录已在使用的同一个 `Url` 记录。

所有协调此操作的工作都将在 `chpt07_SaveFavoriteLink` 存储过程中进行管理。首先，该存储过程将接收保存数据所需的所有参数，如代码清单 7-13 所示。

**代码清单 7-13.** `chpt07_SaveFavoriteLink` 的参数

```sql
CREATE Procedure dbo.chpt07_SaveFavoriteLink
(
    @ProfileID bigint,
    @Url nvarchar(250),
    @Title nvarchar(150),
    @Keeper bit,
    @Rating smallint,
    @Note nvarchar(500),
    @Created datetime,
    @Modified datetime,
    @OldFavoriteLinkID bigint,
    @FavoriteLinkID bigint OUTPUT
)
```

处理保存过程的第一步是将 `Url` 和 `Title` 值保存到辅助表，并取回 `ID` 值以用于外键引用。代码清单 7-14 展示了如何开始。

**代码清单 7-14.** 保存 Title 和 Url 值

```sql
DECLARE @LinkUrlID int
DECLARE @LinkTitleID int
EXEC chpt07_SaveLinkUrl @Url, @LinkUrlID OUTPUT
EXEC chpt07_SaveLinkTitle @Title, @LinkTitleID OUTPUT
```

在 `LinkUrlID` 和 `LinkTitleID` 的值设置好之后，就可以保存主记录了。因为这是一个保存例程，该过程可能是插入一个新记录，也可能是更新一个记录。在决定是插入还是更新之前，必须检查 `@OldFavoriteLinkID` 的值。如果该值为正数，则表示保存操作意在更新一个已知的现有记录。此外，如果 `@OldFavoriteLinkID` 值为负数（意味着未定义），则应检查是否已存在具有相同 `Url` 和 `Title` 值的 `Favorite Link` 记录。代码清单 7-15 中的代码处理这项工作。

**代码清单 7-15.** 检查现有记录

```sql
IF (@OldFavoriteLinkID < 0 AND EXISTS (
    SELECT * FROM chpt07_FavoriteLinks
    WHERE LinkUrlID = @LinkUrlID AND ProfileID = @ProfileID
))
BEGIN
    SET @OldFavoriteLinkID =
        (SELECT FavoriteLinkID FROM chpt07_FavoriteLinks
         WHERE LinkUrlID = @LinkUrlID AND ProfileID = @ProfileID)
END
```

最后，我们应该能够根据 `@OldFavoriteLinkID` 的值是否指向指定配置文件的一个现有记录来决定执行插入还是更新。如果记录确实存在，则执行插入。否则，执行更新，如代码清单 7-16 所示。

**代码清单 7-16.** 通过插入或更新进行保存

```sql
IF EXISTS (SELECT * FROM chpt07_FavoriteLinks
    WHERE FavoriteLinkID = @OldFavoriteLinkID
    AND ProfileID = @ProfileID)
BEGIN
    UPDATE chpt07_FavoriteLinks
    SET
        LinkUrlID = @LinkUrlID,
        LinkTitleID = @LinkTitleID,
        Keeper = @Keeper,
        Rating = @Rating,
        Note = @Note,
        Modified = GETDATE()
    WHERE
        FavoriteLinkID = @OldFavoriteLinkID AND
        ProfileID = @ProfileID
    SET @FavoriteLinkID = @OldFavoriteLinkID
END
ELSE
BEGIN
    INSERT INTO chpt07_FavoriteLinks
        (ProfileID, LinkUrlID, LinkTitleID, Keeper,
        Rating, Note, Created, Modified)
    VALUES (
        @ProfileID,
        @LinkUrlID,
        @LinkTitleID,
        @Keeper,
        @Rating,
        @Note,
        @Created,
        @Modified
    )
    SELECT @FavoriteLinkID = @@IDENTITY
END
GO
```

在任何一种情况下，`@FavoriteLinkID` 的值都会被设置为受影响记录的标识，该值作为一个输出参数提供给存储过程的调用者。

## 删除数据



## 删除数据

存储过程处理的最后一项主要工作是删除数据。对于 `FavoriteLinks` 数据库而言，移除对 `TagToken` 记录的引用（即从 `Favorite Link` 记录中删除标签关联）很容易，但这并非一个简单的 `DELETE` 语句就能搞定。要删除所有不再被引用的 `TagToken` 记录，必须完成几个步骤，从代码清单 7-17 开始。

### 代码清单 7-17. RemoveLinkTag 存储过程的第一步

```sql
CREATE Procedure dbo.chpt07_RemoveLinkTag
(
@FavoriteLinkID bigint,
@Token nvarchar(30)
)
AS
DECLARE @LinkTagID bigint
SET @LinkTagID = (
SELECT TOP 1 lt.LinkTagID
FROM chpt07_LinkTags AS lt
JOIN chpt07_TagTokens AS tt ON tt.TagTokenID = lt.TagTokenID
WHERE lt.FavoriteLinkID = @FavoriteLinkID AND tt.Token = @Token
)
DELETE FROM chpt07_LinkTags WHERE LinkTagID = @LinkTagID
```

这第一步接收 `@FavoriteLinkID` 和 `@Token` 参数，并获取 `@LinkTagID` 的值。一旦获知该 ID，就可以删除该记录。现在，我们需要检查 `@Token` 值是否被任何其他记录引用，如代码清单 7-18 所示。

### 代码清单 7-18. 统计令牌引用次数

```sql
DECLARE @TokenCount int
SET @TokenCount = (
SELECT COUNT(*)
FROM chpt07_LinkTags AS lt
JOIN chpt07_TagTokens AS tt ON tt.TagTokenID = lt.TagTokenID
WHERE tt.Token = @Token
)
```

8601Ch07CMP2 8/24/07 12:23 PM Page 214 **214** 第 7 章 ■ 手动数据访问层

如果计数为零，说明没有记录引用它，那么该记录就可以在最后一步安全删除，如代码清单 7-19 所示。

### 代码清单 7-19. 删除令牌记录

```sql
IF (@TokenCount = 0)
BEGIN
DELETE FROM chpt07_TagTokens WHERE Token = @Token
END
```

这个过程将保持数据库尽可能小，这将有助于维持使用此数据的查询的性能。一个更复杂的场景是彻底清除某个配置文件的所有收藏链接，这涉及到按正确顺序删除 `Tags`、`Tokens`、`Titles` 和 `Urls` 的记录，以遵守外键约束。对于这个序列，我们将把过程分解为多个存储过程以使其更易于管理。表 7-3 列出了这些存储过程。

### 表 7-3. 清除存储过程

| 存储过程 | 描述 |
|------------------|-------------|
| chpt07_PurgeProfile | 清除配置文件 |
| chpt07_PurgeFavoriteLinksByProfileID | 清除收藏链接，包括标题和 URL |
| chpt07_PurgeLinkTagsByProfileID | 清除链接标签和令牌 |

从高层次来看，脚本从配置文件开始这个过程，执行存储过程来清除收藏链接，该存储过程进而执行存储过程来清除标签。它像级联一样工作，所有子引用在父记录被删除之前移除，因此不会违反外键约束。这可能是一个棘手的过程，因为如果父记录指向某个子记录，你就无法删除该子记录。你必须在删除父记录之前创建一个子记录列表，以便之后可以删除子记录。

我们将从底层开始，深入分析由 `chpt07_PurgeLinkTagsByProfileID` 存储过程处理的链接标签，从代码清单 7-20 开始。

### 代码清单 7-20. chpt07_PurgeLinkTagsByProfileID 的第一步

```sql
CREATE Procedure dbo.chpt07_PurgeLinkTagsByProfileID
(
@ProfileID int
)
AS
IF EXISTS (
SELECT * FROM chpt07_LinkTags AS lt
JOIN chpt07_FavoriteLinks AS fl ON fl.FavoriteLinkID = lt.FavoriteLinkID
WHERE fl.ProfileID = @ProfileID
)
BEGIN
```

8601Ch07CMP2 8/24/07 12:23 PM Page 215 第 7 章 ■ 手动数据访问层 **215**

这第一步在开始序列之前检查是否存在任何链接标签。因为清除链接标签的工作需要一点设置，这一步有助于避免额外的工作。接下来，会组装要清除的记录列表。代码清单 7-21 中的代码展示了清除例程。

### 代码清单 7-21. 组装令牌清除列表

```sql
DECLARE @TagTokensToPurge TABLE
(
ID int IDENTITY,
TagTokenID bigint,
[Count] int
)

INSERT INTO @TagTokensToPurge (TagTokenID, [Count])
SELECT tt.TagTokenID, COUNT(*) AS [Count]
FROM chpt07_TagTokens AS tt
```





```sql
JOIN chpt07_LinkTags AS lt ON lt.TagTokenID = tt.TagTokenID
JOIN chpt07_FavoriteLinks AS fl ON fl.FavoriteLinkID = lt.FavoriteLinkID
WHERE tt.TagTokenID in (
    SELECT DISTINCT tt2.TagTokenID
    FROM chpt07_LinkTags AS lt2
    JOIN chpt07_TagTokens AS tt2 ON tt2.TagTokenID = lt2.TagTokenID
    JOIN chpt07_FavoriteLinks AS fl2 ON fl2.FavoriteLinkID = lt2.FavoriteLinkID
    WHERE fl2.ProfileID = @ProfileID
)
GROUP BY tt.TagTokenID
HAVING COUNT(*) = 1
```

为了保存列表，需要声明一个表变量并用查询填充它。这个表变量保存`TagTokenID`及其被引用次数的计数。在清单 7-22 中，链接标签最终被删除。

### 清单 7-22. 删除链接标签记录

```sql
DELETE FROM chpt07_LinkTags WHERE LinkTagID IN
(
    SELECT lt.LinkTagID FROM chpt07_LinkTags AS lt
    JOIN chpt07_FavoriteLinks AS fl ON fl.FavoriteLinkID = lt.FavoriteLinkID
    WHERE fl.ProfileID = @ProfileID
)
```

随着`chpt07_LinkTags`中的父记录被移除，现在可以删除`chpt07_TagTokens`表中的子记录了，如清单 7-23 所示。

### 清单 7-23. 删除令牌记录

```sql
DELETE FROM chpt07_TagTokens WHERE TagTokenID IN
(SELECT TagTokenID FROM @TagTokensToPurge)
```

一半的依赖关系现已妥善清除。现在必须清除标题和 URL。这项工作由名为`chpt07_PurgeFavoriteLinksByProfileID`的存储过程完成，其起始部分如清单 7-24 所示。

### 清单 7-24. chpt07_PurgeFavoriteLinksByProfileID 中的第一步

```sql
CREATE Procedure dbo.chpt07_PurgeFavoriteLinksByProfileID
(
    @ProfileID bigint
)
AS
SET NOCOUNT ON
IF EXISTS
    (SELECT * FROM chpt07_FavoriteLinks WHERE ProfileID = @ProfileID) BEGIN
    EXEC chpt07_PurgeLinkTagsByProfileID @ProfileID
    ...
```

第一步是检查是否有工作要做，因此使用`@ProfileID`查找任何现有记录。如果有，则执行先前用于清除链接标签的存储过程。下一步是组装要清除的 URL 列表，如清单 7-25 所示。

### 清单 7-25. 组装 URL 清除列表

```sql
DECLARE @LinkUrlsToPurge TABLE
(
    ID int IDENTITY,
    LinkUrlID bigint,
    [Count] int
)
INSERT INTO @LinkUrlsToPurge (LinkUrlID, [Count])
SELECT lu.LinkUrlID, COUNT(*) AS [Count]
FROM chpt07_LinkUrls AS lu
JOIN chpt07_FavoriteLinks AS fl ON fl.LinkUrlID = lu.LinkUrlID
WHERE lu.LinkUrlID in (
    SELECT DISTINCT LinkUrlID FROM chpt07_FavoriteLinks
    WHERE ProfileID = @ProfileID
)
GROUP BY lu.LinkUrlID
HAVING COUNT(*) = 1
```

同样，使用一个表变量来保存记录列表。这次它保存的是`LinkUrlID`。

此步骤在清单 7-26 中针对标题重复进行。

### 清单 7-26. 组装标题清除列表

```sql
DECLARE @LinkTitlesToPurge TABLE
(
    ID int IDENTITY,
    LinkTitleID bigint,
    [Count] int
)
INSERT INTO @LinkTitlesToPurge (LinkTitleID, [Count])
SELECT lt.LinkTitleID, COUNT(*) AS [Count]
FROM chpt07_LinkTitles AS lt
JOIN chpt07_FavoriteLinks AS fl ON fl.LinkTitleID = lt.LinkTitleID
WHERE lt.LinkTitleID in (
    SELECT DISTINCT LinkTitleID FROM chpt07_FavoriteLinks
    WHERE ProfileID = @ProfileID
)
GROUP BY lt.LinkTitleID
HAVING COUNT(*) = 1
```

URL 和标题的清除列表只保存具有单一引用的记录。因为即将被删除的记录正是持有引用的记录，所以最后一步就是简单地删除所有父记录和清除列表中的所有项目，如清单 7-27 所示。

### 清单 7-27. 清除收藏链接的最后一步

```sql
DELETE FROM chpt07_FavoriteLinks WHERE ProfileID = @ProfileID
DELETE FROM chpt07_LinkUrls WHERE LinkUrlID IN
(SELECT LinkUrlID FROM @LinkUrlsToPurge)
DELETE FROM chpt07_LinkTitles WHERE LinkTitleID IN
(SELECT LinkTitleID FROM @LinkTitlesToPurge)
```

所有用于获取、保存和删除数据的存储过程就位后，我们就可以开始构建数据访问层了，它将使用所有这些存储过程。

#### 数据访问层

虽然存储过程是数据访问层的关键元素，但许多人认为真正的数据访问层始于 C# 代码开始的地方。在许多情况下，这是正确的。当使用类型化数据集（Typed DataSets）自动将表适配器绑定到数据库中的表时，情况就是如此。但本章所做的工作直接将 C# 代码与存储过程耦合在一起，以旨在提升性能的方式分配工作。

为了创建一个能与数据绑定良好协作的数据访问层，我们将创建一个具有默认构造函数的域对象，以便可以用`ObjectDataSource`声明式地实例化它。我们还将创建一个表示收藏链接的业务对象。这个对象将封装数据访问层的内部工作原理，仅向应用层暴露数据。通过隐藏实现细节，使得更新未来的版本更加容易。它使得可以重新设计实现，而无需应用程序的其余部分进行更新。

### 常用文件夹补充

获取、保存和删除的存储过程将随不同的值被经常重用。你可以从可下载的代码示例中取一组这样的存储过程，为自己创建模板，并将它们添加到你的模板文件夹（`D:\Projects\Common\Templates\ Data Access Layer`）中的 Common 文件夹中。从这些模板开始，应该可以在构建数据访问层时节省你的时间。

### 创建类库

数据访问层应该是可复用的，实现这一点的第一步是将其放在一个类库中。另一种方法是直接在你的应用程序中包含数据访问层。在网站项目的案例中，你会将类放在`App_Code`文件夹中。这样做不允许其他项目将这些类用作依赖项。我们的目标是将数据访问层视为一个内部发布版本，供多个应用程序使用，即使它最初仅由单个网站使用。这样做不需要太多额外的努力，并有助于保持应用程序和数据访问层的优先级分离。

该项目的组织结构应该已经包括数据库项目作为解决方案的一部分。现在可以向解决方案添加一个类库来保存数据访问库。

我倾向于给项目起通用的名称，因此数据库项目命名为`Database`，而类库就简单地称为`ClassLibrary`。这简化了将在第 8 章中介绍的脚本编写，因为几乎相同的 MSBuild 脚本可以用在多个解决方案中，只需进行最小的调整。自然，随着`ClassLibrary`项目承担大量的功能，它可以被拆分为多个类库。通过正确使用命名空间，应该可以很容易地将命名空间重新分配到多个类库中，而对使用这些库的应用程序影响最小。实际上，在类库的属性面板中，`Default Namespace`的值应设置为应用程序的名称，而类应添加到项目中的文件夹中。对于本章，`Default Namespace`的值是`Chapter07`，而数据访问层的所有代码将放入一个名为`Domain`的文件夹中。创建新类时自动设置的命名空间将是`Chapter07.Domain`。

### 关注点分离



为了保持数据访问层的模块化特性，组建一个团队专门负责数据层开发，而另一个团队负责应用程序开发，可能会很有帮助。应用程序开发者可以处理业务逻辑方面的需求，同时向数据团队请求必要的功能。这种清晰的分离有助于确保应用程序特有的细节不会被固化到数据访问层中。当应用程序需要一个新功能时，数据团队可以考虑如何在不损害数据访问层完整性的情况下，最好地提供该功能。之后，当包含数据访问层的类库被多个应用程序使用时，该库不应包含应用程序特有的细节，否则下一个应用程序将无法提供这些细节，从而使其更难以被复用。

## 创建数据访问方法

使用企业库中的**数据访问应用程序块**，结合第 2 章介绍的代码片段，可以轻松创建数据访问方法。我们将从类库和一个名为 `Domain` 的文件夹开始。在该文件夹中，我们将创建一个名为 `FavoriteLinkDomain` 的类。在准备好第 2 章的代码片段后，我们将使用第一个数据访问代码片段（快捷方式为 `datypes`）来创建一个新变量。你可以按 Ctrl+K, Ctrl+X 键序列打开代码片段选择器并导航到第一个片段，也可以使用快捷方式，输入 `datypes` 然后按两次 Tab 键来立即插入该片段。第一个代码片段只是声明一个对 `Database` 对象的引用，该对象将在类的其余部分中使用。

接下来，我们需要在默认构造函数中实例化 `Database` 对象。第二个代码片段完成这项工作，其快捷方式定义为 `dacreation`。为域类创建默认构造函数，并将此代码片段插入其中。此片段包含一个连接字符串名称的占位符。目前，我们将它设置为 `FavoriteLinks`，使其如清单 7-28 所示。

**清单 7-28.** `FavoriteLinkDomain` 的默认构造函数
```csharp
public FavoriteLinkDomain()
{
    db = DatabaseFactory.CreateDatabase("FavoriteLinks");
}
```

### 数据库配置

一个简单的应用程序可能只有一个数据库连接。在此类应用程序中，可以将数据访问层配置为使用默认数据库连接，作为企业库配置的一部分。在第 8 章，自定义配置将比在数据访问层中硬编码连接字符串名称提供更大的灵活性。

下一步是创建从数据库获取数据的方法。**获取 DataSet 方法**片段将创建一个方法，该方法调用存储过程并用结果构造一个 `DataSet`。此片段的快捷方式是 `dagetds`。清单 7-29 展示了其中一个获取方法。

**清单 7-29.** 获取最近收藏链接的方法
```csharp
public DataSet GetRecentFavoriteLinksByProfileID(
    long profileId, int startDaysBack, int endDaysBack)
{
    DataSet ds;
    try
    {
        using (DbCommand dbCmd = db.GetStoredProcCommand(
            "chpt07_GetRecentFavoriteLinksByProfileID"))
        {
            db.AddInParameter(dbCmd, "@ProfileID",
                DbType.Int64, profileId);
            db.AddInParameter(dbCmd, "@StartDaysBack",
                DbType.Int32, startDaysBack);
            db.AddInParameter(dbCmd, "@EndDaysBack",
                DbType.Int32, endDaysBack);
            ds = db.ExecuteDataSet(dbCmd);
        }
    }
    catch (Exception ex)
    {
        throw GetException(
            "Error in GetRecentFavoriteLinksByProfileID", ex);
    }
    //返回结果
    return ds;
}
```

此方法接收三个参数，并用结果填充 `DataSet`。对于此示例项目，还有其他几个获取方法，你可以在本章的可下载代码中找到。

作为使用存储过程结果填充 `DataSet` 的替代方法，你也可以使用返回 `IDataReader` 的片段。此片段的快捷方式是 `dagetdr`。它看起来与 `DataSet` 方法相同，但返回的是 `IDataReader` 而不是 `DataSet`。

保存方法的工作方式与获取方法不同。它们不是填充 `DataSet` 或 `IDataReader`，而是简单地执行一个非查询并返回一个可选的输出参数。用于非查询方法的代码片段快捷方式是 `danonquery`。清单 7-30 展示了一个保存方法。

**清单 7-30.** 保存收藏链接的方法
```csharp
public long SaveFavoriteLink(
    long profileId, string url, string title,
    bool keeper, int rating, string note, string tags,
    DateTime created, DateTime modified, long oldFavoriteLinkId)
{
    long favoriteLinkId;
    try
    {
        using (DbCommand dbCmd =
            db.GetStoredProcCommand("chpt07_SaveFavoriteLink"))
        {
            db.AddInParameter(dbCmd, "@ProfileID",
                DbType.Int64, profileId);
            db.AddInParameter(dbCmd, "@Url",
                DbType.String, url);
            db.AddInParameter(dbCmd, "@Title",
                DbType.String, title);
            db.AddInParameter(dbCmd, "@Keeper",
                DbType.Boolean, keeper);
            db.AddInParameter(dbCmd, "@Rating",
                DbType.Int16, rating);
            db.AddInParameter(dbCmd, "@Note",
                DbType.String, note);
            db.AddInParameter(dbCmd, "@Created",
                DbType.DateTime, created);
            db.AddInParameter(dbCmd, "@Modified",
                DbType.DateTime, modified);
            db.AddInParameter(dbCmd, "@OldFavoriteLinkID", DbType.Int64, oldFavoriteLinkId);
            db.AddOutParameter(dbCmd, "@FavoriteLinkID",
                DbType.Int64, 0);
            db.ExecuteNonQuery(dbCmd);
            object obj = db.GetParameterValue(dbCmd, "@FavoriteLinkID");
            favoriteLinkId = (long)obj;
        }
    }
    catch (Exception ex)
    {
        throw GetException("Error in SaveFavoriteLink", ex);
    }
    SaveLinkTags(favoriteLinkId, tags);
    return favoriteLinkId;
}
```

用于删除或清除数据的方法也使用非查询片段来调用存储过程，以执行删除数据所需的必要工作。

这些获取、保存和移除数据的方法都处理的是原始数据。这些原始数据直接绑定到存储过程，存储过程充当数据访问层与数据库中基础表之间的代理。在这些方法之上，用户界面层使用的对象应该以对应用程序所有者和长期维护此软件的开发人员有意义的方式来表示数据。

## 处理异常

在这些数据访问方法中，对数据库的调用总是包裹在 `try...catch` 块中。我加入这个额外保护层有两个主要原因。首先，我希望能够在我知道如何处理问题的数据访问层内部处理该问题。其次，我想在数据访问层中捕获任何可能向用户暴露数据库信息的异常。这样做可以阻止可能导致安全漏洞的问题。然而，添加 `try...catch` 块可能会降低性能。做出这个选择时已经考虑到了这一点。性能损失应该是微小的。

为了避免进入此 `try...catch` 块的性能影响，我仍然可以使用缓存。如果数据库查询要请求的数据已经在缓存中，则可以避免调用数据库以及异常处理。如果你选择完全去掉 `try...catch` 块，可以将其剥离，只保留用于 `DbCommand` 资源管理的 `using` 子句。你可能会发现，为性能所做的权衡并不显著到足以放弃紧密异常处理所提供的稳定性和安全性。

## 创建业务对象



在开发应用程序时，您会希望传递符合业务逻辑的对象。像 `Customer` 这样的业务对象具有诸如 `FirstName`、`LastName`、`CurrentOrder` 和 `LastOrder` 等属性。这个业务对象完整实现了对象的三个部分：数据、行为和关系。`Customer` 拥有姓氏和名字，并与他的当前订单及上次订单存在关联。`Customer` 也可以完成他的 `CurrentOrder`。这些都是应用程序所有者理解并通过精心设计的业务对象传达给开发团队的流程。您无需将数据分散地存储在直接映射关系型数据库结构的 `DataSet` 中，而是有机会真正地为应用程序表示现实世界的对象。除了恰当地表示现实世界的对象之外，您还可以将这些业务对象背后的数据访问层完全封装起来，这为您提供了使用延迟加载等技术的机会，从而在保持统一公共接口的同时提升性能。

“收藏链接”网站的核心业务对象是 `FavoriteLink` 对象。作为一个业务对象，它持有链接的各种属性，如 `Title`、`Url`、`Rating`、`Note` 和 `Tags`。这些属性中的每一个都由调用存储过程返回的列填充。映射这些值可能是一项相当繁重的工作，尤其是在应用程序的早期阶段，当您仍在调整列和属性时。更新映射可能非常耗时。

## 加载数据

反对手动创建自有数据访问层的最有力论据之一，可能是将数据从数据库映射到对象所需的工作量。自对象与关系型数据库结合使用以来，对象/关系映射就一直是一个普遍问题。这些通常不兼容的系统之间的阻抗失衡要求您理解两端是如何构建的，以便利用这种理解将它们映射到一起。

实现 O/R 映射的一种方法是创建配置文件，显式设置每个数据列与对象属性之间的映射。流行的 O/R 映射工具如 `NHibernate` 使用了大量定义所有映射的 XML 配置。许多人认为这是满足 O/R 映射需求的强大解决方案。但我认为这是一个额外且不必要的层。

为了简化 O/R 映射工作，我创建了一个基类，该基类通过匹配可写属性的名称和类型，自动将数据列映射到属性。

由数据访问层填充的每个业务对象都继承此基类。一个加载方法接受 `DataRow`，而另一个接受 `IDataReader` 对象来设置属性。通过反射，基类将来自查询的列与业务对象中的属性进行比较。我称此基类为 `DomainObject`。它始终提供的核心属性是 `ID`、`Created` 和 `Modified`。当我向存储过程添加新列，并向业务对象添加新属性时，这些值会自动加载到对象中，无需任何额外工作。（请参阅代码下载中的 `DomainObject` 类。）我确保 `DomainObject` 可以与 `DataSet` 或 `IDataReader` 协同工作，因为两者在不同场景下都有其价值。我始终希望保持在它们之间切换的灵活性。例如，我可以在数据访问层深处缓存 `DataSet`，因为它持有数据，而 `IDataReader` 只是一个数据枚举器，只能使用一次。但 `IDataReader` 更快，因此如果我不缓存数据，就可以利用 `IDataReader` 的速度优势。

当从数据库提取数据并且代码循环处理结果时，我们会将 `DataRow` 或 `IDataReader` 传递给业务对象的加载方法，以准备将其发送回应用程序层。这些对象被保存在一个 `FavoriteLinkCollection` 对象中，如代码清单 7-31 所示。

### 代码清单 7-31. FavoriteLinkCollection 对象
```csharp
using System.Collections.Generic;

namespace Chapter07.Domain
{
    /// <summary>
    /// FavoriteLink 对象的集合
    /// </summary>
    public class FavoriteLinkCollection : List<FavoriteLink>
    {
    }
}
```

请注意，`FavoriteLinkCollection` 对象只是继承自泛型集合 `List<FavoriteLink>`，这为我们提供了一个简洁的对象，便于后续扩展。它也是使用 `FavoriteLink` 业务对象进行强类型化的。可以使用 `DataSet` 来填充 `FavoriteLinkCollection`，如代码清单 7-32 所示。

### 代码清单 7-32. 使用 DataRow 加载
```csharp
private void AddToFavoriteLinkCollection(
    FavoriteLinkCollection collection, DataSet ds)
{
    if (ds.Tables.Count > 0)
    {
        foreach (DataRow row in ds.Tables[0].Rows)
        {
            FavoriteLink fl = new FavoriteLink(row);
            collection.Add(fl);
        }
    }
}
```

`FavoriteLink` 对象的一个构造函数简单地接受一个 `DataRow`，并在内部调用 `DomainObject` 定义的加载方法。或者，也可以使用 `IDataReader` 以获得可能更好的性能，如代码清单 7-33 所示。

### 代码清单 7-33. 使用 IDataReader 加载
```csharp
private void AddToFavoriteLinkCollection(
    FavoriteLinkCollection collection, IDataReader dr)
{
    while (dr.Read())
    {
        FavoriteLink fl = new FavoriteLink(dr);
        collection.Add(fl);
    }
}
```

`FavoriteLink` 对象还包含一个接受 `IDataReader` 的构造函数，该构造函数被传递给加载方法。在 `DataSet` 和 `IDataReader` 的情况下，用于填充 `FavoriteLink` 的列应该是相同的，尤其是在它们调用相同的存储过程时。

## 调整数据加载

由于使用反射会产生性能损耗，在用反射加载数据与直接将数据转换到业务对象属性之间存在明显的性能差异。如果您知道一个名为 `Title` 的数据列是 `String` 类型，您可以简单地通过直接转换对象来设置属性，如代码清单 7-34 所示。

### 代码清单 7-34. 将数据值直接转换到属性
```csharp
this.Rating = (int)row["Rating"];
```

如果该列是 `int` 类型且值不为 null，这将会奏效——但这从来都不是一个保证。

由于数据库是断开连接的，并且可以独立于应用程序进行更新，数据库的更改可能是一个潜在问题。使用反射的方法创建了匹配名称和类型的映射。每次使用映射设置属性值时，都会检查数据列是否为 null 值，这种检查消除了类型转换错误的风险。这种保护确实是有代价的。幸运的是，这种代价已经被最小化了。

`FavoriteLink` 对象通过设置一个静态枚举值，能够在反射和直接转换模式之间切换。此设置允许测试两种模式的性能。此数据访问层的单元测试包括加载测试，用于测量加载大量 `FavoriteLink` 对象所需的时间，然后显示两种模式的平均时间。这些加载测试包含在本章的代码下载中。

这些加载测试在运行大量项目时显示，反射的开销对将数据加载到 `FavoriteLink` 对象中的时间没有显著影响。



性能已经过优化，旨在最小化反射的开销。通过反射创建的映射在首个对象加载时即已生成。这些映射会在每个新实例中复用，从而在获取收益的同时，几乎消除了性能损耗。

但是，如果您希望利用直接将数据列转换为属性（而不使用反射）的优势，可以将加载方法标记为 `virtual`，以便您能够重写它们。

#### 构建网站

数据访问层准备就绪后，我们现在可以开始组装网站。数据将通过与用户控件中持有的数据绑定控件相关联的 `ObjectDataSources` 引入网站。用户控件可以放置在任何页面上，按需布置。页面也将为网站使用一个统一的母版页。

为网站创建的用户控件将成为网站的构建块，这些构建块将直接与数据访问层交互。用户控件随后可以向父容器公开属性、方法和事件，以提供丰富的用户界面层功能。

您可能会选择从大型页面开始构建网站，但很快您就会遇到包含 2000 行代码隐藏文件的页面，这些页面难以组织和维护。从单个页面开始是原型化新界面的好方法，但最终您会希望将页面的各个部分拆分到用户控件中，以利于维护，并利用将页面不同部分解耦到定义明确的容器中的优势。

最终，您可以选择将用户控件转换为服务器控件，后者可以纯粹作为一个程序集来部署。这种转换使得该控件能够作为依赖项在众多网站中轻松使用。创建和维护服务器控件需要更多工作，因此从一开始创建一个通常不是最佳的第一步。随着用户控件的成熟，它可能最终会成为一个很好的转换候选。目前，如果您想在多个网站中使用一个用户控件，您必须将 `.ascx` 标记文件连同其代码隐藏文件或代码隐藏的编译程序集一起复制到每个网站中。如果一个用户控件将在许多网站中有用，那么采取额外步骤将其转换为服务器控件将是可取的。

随着时间的推移，您可能会拥有一套用户控件和服务器控件，就像 Visual Studio 工具箱中的所有标准控件一样。这可能会催生一组类似于登录控件的数据绑定控件，这些登录控件直接与成员资格提供程序系统集成。

以下示例页面和用户控件正是基于这样的目标而设计。

##### 连接数据访问层

由于数据访问层是同一解决方案内的一个类库，可以通过使用项目引用将其关联到网站。因为 ASP.NET 网站没有项目文件，所以引用由解决方案存储。连接字符串名称设置为 `FavoriteLinks`，因此 `Web.config` 文件应包含一个同名的连接字符串。

对于简单的网站，使用硬编码的字符串作为连接字符串名称可能就足够了，但随着网站的发展和创建的组件越来越多，协调所有硬编码的字符串将变得困难。

随着网站变得越来越复杂，您可以实现自定义配置节，为每个组件定义设置，例如连接字符串名称。

然后，您可以通过使用自定义配置来调整组件的连接字符串名称。创建自定义配置将在第 9 章介绍。本章的示例项目使用了一个自定义节来定义数据库连接。

##### 创建用户控件



用户控件可用于封装至少一个数据绑定控件及其关联的 `ObjectDataSource`。在此独立容器中处理数据，使得处理数据及数据绑定控件所需的功能更为容易。此容器可公开属性和事件，以便与父容器进行通信。

### 属性

为了显示链接，我们将创建 `LinksControl` 用户控件，该控件公开了一些属性，父控件应设置这些属性以协助绑定数据。`ObjectDataSource` 使用的属性是 `StartDaysBack` 和 `EndDaysBack`，它们存储 `int` 值。这些属性如清单 7-35 所示。

**清单 7-35.** `LinksControl` 属性

```csharp
private int _startDaysBack = 7;

[Browsable(true), DefaultValue(7), Category("Links")]
public int StartDaysBack
{
    get { return _startDaysBack; }
    set { _startDaysBack = value; }
}

private int _endDaysBack = 0;

[Browsable(true), DefaultValue(0), Category("Links")]
public int EndDaysBack
{
    get { return _endDaysBack; }
    set { _endDaysBack = value; }
}
```

这些是常规属性，其成员变量用于存储 `int` 值；然而，装饰这些属性的特性是用户控件特有的。`Browsable` 特性使得在设计界面上选中用户控件时，该属性会显示在属性窗口中。`Category` 特性将这两个属性归入“链接”类别，使它们一起显示在属性窗口中。最后，`DefaultValue` 特性指定了默认值。当属性值与默认值不匹配时，此特性会使属性窗口中的值以粗体显示。这些额外特性有助于使用该用户控件的开发人员，并在将数据引入应用程序时进一步封装数据。

设置这些属性后，`ObjectDataSource` 会使用它们的值作为清单 7-36 中由 `ObjectDataSource` 定义的参数。

**清单 7-36.** `LinksControls` ObjectDataSource1

```xml
<asp:ObjectDataSource ID="ObjectDataSource1" runat="server"
    OldValuesParameterFormatString="original_{0}"
    TypeName="Chapter07.Domain.FavoriteLinkDomain"
    SelectMethod="GetRecentFavoriteLinkCollection"
    OnSelecting="ObjectDataSource1_Selecting">
    <SelectParameters>
        <asp:Parameter DefaultValue="0"
            Name="profileId" Type="Int32" />
        <asp:Parameter DefaultValue="7"
            Name="startDaysBack" Type="Int32" />
        <asp:Parameter DefaultValue="0"
            Name="endDaysBack" Type="Int32" />
    </SelectParameters>
</asp:ObjectDataSource>
```

可以看到 `startDaysBack` 和 `endDaysBack` 参数，它们是通过清单 7-37 所示的 `ObjectDataSource1_Selecting` 事件处理程序设置的。

**清单 7-37.** `ObjectDataSource1_Selecting`

```csharp
protected void ObjectDataSource1_Selecting(
    object sender, ObjectDataSourceSelectingEventArgs e)
{
    e.InputParameters["profileId"] = Utility.GetProfile().ProfileID;
    e.InputParameters["startDaysBack"] = StartDaysBack;
    e.InputParameters["endDaysBack"] = EndDaysBack;
}
```

剩下要做的就是将数据绑定到数据绑定控件，即清单 7-38 所示的 `Repeater` 控件。

**清单 7-38.** `Repeater` 控件

```xml
<asp:Repeater ID="rptLinks" runat="server"
    DataSourceID="ObjectDataSource1"
    OnItemDataBound="rptLinks_ItemDataBound">
    <HeaderTemplate>
        <div id="divLinks">
            <h3 class="Title" id="h3Title" runat="server">
                <asp:Literal ID="ltTitle" runat="server"
                    Text="Title"></asp:Literal>
            </h3>
            <ul class="Links">
    </HeaderTemplate>
    <ItemTemplate>
        <li>
            <asp:HyperLink ID="hl1" runat="server"
                Text='<%# Bind("Title") %>'
                NavigateUrl='<%# Bind("Url") %>'></asp:HyperLink>
            <span class="Rating">
                (<asp:Literal ID="lt1" runat="server"
                    Text='<%# Bind("Rating") %>'>1</asp:Literal>)
            </span>
        </li>
    </ItemTemplate>
    <FooterTemplate>
            </ul>
            <br style="clear: left;" />
        </div>
    </FooterTemplate>
</asp:Repeater>
```

第 7 章 ■ 手动数据访问层
**228**



通过将网站内容拆分为更小的用户控件，并通过属性来控制它们，你可以获得极大的灵活性。这些属性的值可以传递到数据访问层，从而允许你重复使用同一个用户控件，同时在显示时获得不同的结果。以这种方式使用用户控件，实质上是将其功能参数化，并以更易于管理的方式将其与数据访问层对齐。

## 事件

当订阅的事件发生时，用户控件也可以使用事件将信息传回父控件。另一个名为 `TagCloudControl` 的用户控件会显示一个带有一组链接的标签云。当用户点击 `TagCloudControl` 中的一个链接时，它会引发一个事件，父页面使用该事件来更改另一个名为 `TaggedLinksControl` 的用户控件所显示的内容。以这种方式使用事件进一步封装了用户控件的内部结构，使其更具可重用性。

`TagCloudControl` 声明了一个名为 `TagSelected` 的事件，每当标签云中的链接被点击时就会引发该事件。该事件及引发事件的方法如清单 7-39 所示。

`清单 7-39.` `TagSelected 事件`

```csharp
public event EventHandler<TagCloudEventArgs> TagSelected;

[Category("Tags")]
protected virtual void OnTagSelected(TagCloudEventArgs e)
{
    if (TagSelected != null)
    {
        TagSelected(this, e);
    }
}
```

`EventHandler` 使用了一个名为 `TagCloudEventArgs` 的控件 `EventArgs` 实现，该实现包含一个名为 `Token` 的属性，该属性对事件订阅者可用。当事件引发时，此值可以设置为 `TaggedLinksControl` 中的 `Token` 属性。`TaggedLinksControl` 的工作方式与 `LinksControl` 类似，但它使用 `Token` 属性通过 `Token` 属性获取链接列表。

**注意** 用户控件可以在每个使用它们的页面中声明，但也可以在 `Web.config` 文件的 `pages/controls` 部分声明，以便在整个网站中使用一致的标签前缀来引用用户控件。通常，当你将用户控件拖到设计界面时，生成的声明会将标签前缀设置为 `uc1` 或 `uc2` 等。之后，如果将某个控件从用户控件转换为服务器控件，网站中只需在一个地方调整引用。

##### 创建页面

现在，将用户控件组合到页面上并连接它们的属性和事件可能是最简单的步骤。由于母版页已经创建，此页面只需保存用户控件以列出链接和标签云。我们可以创建一个名为 `Home.aspx` 的新页面，并将用户控件拖到设计界面上。最终结果将如清单 7-40 所示。

`清单 7-40.` `Home.aspx`

```aspx
<%@ Page Language="C#" MasterPageFile="~/FavoriteLink.master"
    AutoEventWireup="true"
    CodeFile="Home.aspx.cs"
    Inherits="Home"
    Title="Favorite Link: Home" %>

<asp:Content ID="Content1" runat="server"
    ContentPlaceHolderID="MainContentPlaceHolder">
    <div id="HomeContent">
        <div class="tagCloud" style="float: right; width: 150px;">
            <h3 class="Title">标签</h3>
            <fl:TagCloudControl ID="tagCloud" runat="server"
                OnTagSelected="tagCloud_OnTagSelected" />
        </div>
        <div id="Links">
            <fl:TaggedLinksControl
                ID="taggedLinks" runat="server"
                Visible="false" />
            <fl:LinksControl
                ID="lcToday" runat="server"
                Title="今天"
                EndDaysBack="0"
                StartDaysBack="1"
                TitleVisible="true" />
            <fl:LinksControl
                ID="lcThisWeek" runat="server"
                Title="本周"
                EndDaysBack="1"
                StartDaysBack="7"
                TitleVisible="true" />
            <fl:LinksControl
                ID="lcThisMonth" runat="server"
                Title="本月"
                EndDaysBack="8"
                StartDaysBack="31"
                TitleVisible="true" />
        </div>
    </div>
</asp:Content>
```

每个用户控件都没有在页面顶部声明用户控件引用，因为它们已经在 `Web.config` 文件的 `pages` 部分预先声明了，如清单 7-41 所示。

`清单 7-41.` `用户控件配置`

```xml
<pages>
    <controls>
        <add tagPrefix="fl" tagName="HeaderNavigation"
            src="~/Controls/HeaderNavigation.ascx"/>
        <add tagPrefix="fl" tagName="LinksControl"
            src="~/Controls/LinksControl.ascx"/>
        <add tagPrefix="fl" tagName="LoginControl"
            src="~/Controls/LoginControl.ascx"/>
        <add tagPrefix="fl" tagName="TagCloudControl"
            src="~/Controls/TagCloudControl.ascx"/>
        <add tagPrefix="fl" tagName="TaggedLinksControl"
            src="~/Controls/TaggedLinksControl.ascx"/>
    </controls>
</pages>
```

每个用户控件都被赋予 `fl` 作为其 `tagPrefix`，你可以在 `Home.aspx` 中看到。这个细节稍微清理了页面，并为声明每个用户控件创建了一致的方式。

这个主页面将 `LinksControl` 列出了三次，分别显示今天、本周和本月添加的链接。`TagCloudControl` 列出了与用户添加的链接相关的标签。当链接被点击时，`Token` 属性由 `tagCloud_OnTagSelected` 事件处理程序捕获，如清单 7-42 所示。

`清单 7-42.` `tagCloud_OnTagSelected`

```csharp
protected void tagCloud_OnTagSelected(object sender, TagCloudEventArgs e)
{
    lcToday.Visible = false;
    lcThisWeek.Visible = false;
    lcThisMonth.Visible = false;
    taggedLinks.Visible = true;
    taggedLinks.Title = "标记为 " + e.Token;
    taggedLinks.Token = e.Token;
}
```

这几行代码就是调整页面上所有用户控件所需的全部内容。它们所有的独立细节都已完全封装并在内部管理。现在可以在架构的各个层面提升此网站的性能。一些调整可以在表上进行，并与对存储过程的必要修改相协调。另一些调整可以通过在数据访问层引入缓存来完成。在应用层，用户控件和页面可以使用额外的技术，例如数据或输出缓存。每一层都是完全独立和封装的，因此在实施优化时，下一层不需要修改。

#### 总结

本章介绍了手动创建数据访问层时需要考虑的步骤和选择。你了解了手动构建此层需要额外的工作，以便利用它所提供的性能增强。最后，你通过使用 `ObjectDataSource` 和用户控件将数据访问层连接到一个功能网站，这种方式在最大限度地减少维护的同时，也留下了大量无需对应用程序进行重大返工即可优化性能的空间。

