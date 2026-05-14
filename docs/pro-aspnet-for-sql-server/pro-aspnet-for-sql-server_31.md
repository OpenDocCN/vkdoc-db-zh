# 第 5 章 ■ SQL 提供程序

## 清单 5-25. sm_InsertSiteMapNode.sql

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'sm_RepopulateSiteMapNodes')
BEGIN
DROP Procedure sm_RepopulateSiteMapNodes
END
GO

CREATE Procedure dbo.sm_RepopulateSiteMapNodes
AS
SET NOCOUNT ON

DECLARE @RootNodeID bigint
DECLARE @AlbumsNodeID bigint

-- 重置表
--TRUNCATE TABLE sm_SiteMapNodes
--DBCC CHECKIDENT (sm_SiteMapNodes, RESEED, 0)
DELETE FROM sm_SiteMapNodes

EXEC sm_InsertSiteMapNode -1, 'Default.aspx', 'Home', 0, @RootNodeID OUTPUT
EXEC sm_InsertSiteMapNode @RootNodeID, 'Albums/Default.aspx', 'Albums', 1, @AlbumsNodeID OUTPUT

DECLARE @Albums TABLE
(
ID int IDENTITY,
AlbumID bigint,
[Name] varchar(50),
UserName nvarchar(256)
)

INSERT INTO @Albums (AlbumID, [Name], UserName)
SELECT ID, [Name], UserName
FROM pap_Albums
WHERE IsActive = 1

DECLARE @CurID int
DECLARE @MaxID int
DECLARE @AlbumID bigint
DECLARE @AlbumNodeID bigint
DECLARE @Name varchar(50)
DECLARE @UserName nvarchar(256)
DECLARE @Url nvarchar(150)

SET @MaxID = ( SELECT MAX(ID) FROM @Albums )
SET @CurID = 1

WHILE (@CurID <= @MaxID)
BEGIN
SET @AlbumID = ( SELECT AlbumID FROM @Albums WHERE ID = @CurID )
SET @Name = ( SELECT Name FROM @Albums WHERE ID = @CurID )
SET @UserName = ( SELECT UserName FROM @Albums WHERE ID = @CurID )
SET @Url = ( 'Albums/Album.aspx?AlbumID=' + CONVERT(varchar(10), @AlbumID) + '&UserName=' + @UserName )

-- PRINT 'Name = ' + @Name
-- PRINT 'UserName = ' + @UserName
-- PRINT 'Url = ' + @Url

EXEC sm_InsertSiteMapNode @AlbumsNodeID, @Url, @Name, 2, @AlbumNodeID OUTPUT

SET @CurID = @CurID + 1
END

SET NOCOUNT OFF
GO

GRANT EXEC ON sm_RepopulateSiteMapNodes TO PUBLIC
GO
```

我有几行被注释掉了，这样在正常使用存储过程时它们不会被包含在内，但当我需要对其进行更新时，我发现重新启用这些行很有帮助。至于被注释掉的 `TRUNCATE` 命令，它需要更高的权限，而这在生产系统中可能是不允许的，所以我设置为使用 `DELETE` 命令代替。`truncate` 方案很有用，因为当它运行时，不会生成审计日志，这可以节省你的时间和磁盘空间。

由于 `@CurID` 用于循环遍历表变量，因此组装了 `@Url`，并通过使用 `sm_InsertSiteMapNode` 将新节点插入到 `sm_SiteMapNodes` 表中。创建此存储过程的脚本如清单 5-25 所示。

## 清单 5-25. sm_InsertSiteMapNode.sql

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'sm_InsertSiteMapNode')
BEGIN
DROP Procedure sm_InsertSiteMapNode
END
GO

CREATE Procedure dbo.sm_InsertSiteMapNode
(
@ParentID bigint,
@Url nvarchar(150),
@Title nvarchar(50),
@Depth int,
@ID bigint OUTPUT
)
AS
IF NOT EXISTS (SELECT * FROM sm_SiteMapNodes WHERE Url = @Url)
BEGIN
INSERT INTO sm_SiteMapNodes (ParentID, Url, Title, Depth, Creation, Modified)
VALUES (
@ParentID,
@Url,
@Title,
@Depth,
GETDATE(),
GETDATE()
)

SET @ID = @@IDENTITY
END
ELSE
BEGIN
SET @ID = ( SELECT ID FROM sm_SiteMapNodes WHERE Url = @Url )
END
GO

GRANT EXEC ON sm_InsertSiteMapNode TO PUBLIC
GO
```

实现 `SiteMapProvider` 本身就是一个重写四个抽象方法的问题。

在 `SiteMapProvider` 抽象基类上还有其他几个方法，它们用于协调这四个方法，但其中一些方法也可以被单独重写，如清单 5-26 所示。

## 清单 5-26. SiteMapProvider 上的抽象方法

```csharp
public abstract SiteMapNode FindSiteMapNode(string rawUrl);
public abstract SiteMapNodeCollection GetChildNodes(SiteMapNode node);
public abstract SiteMapNode GetParentNode(SiteMapNode node);
protected abstract SiteMapNode GetRootNodeCore();
```

实现这些方法并没有什么特别困难的。真正的工作在于从数据库加载数据并保持其最新。这项工作始于清单 5-27 中的 `Initialize` 方法。



