# 清单 5-28. SqlSiteMapProvider 中的 EnsureSiteMapLoaded 方法

```csharp
private void EnsureSiteMapLoaded()
{
    if (rootNode == null)
    {
        // 在内存中构建站点地图。
        LoadSiteMapFromDatabase();
    }
}
```

`SiteMap` 的工作需要 `rootNode`，因此将其设置为 `null` 会导致在下次调用此方法时重新加载数据。我创建了一个机制，在数据更新时自动将 `rootNode` 重置回 `null`。

`SiteMap` 数据是通过单个查询从单个表中加载的。最终，这些数据从数据库加载，并以一小时的生存时间窗口放置在缓存机制中，并附加了 `CachedItemRemovedCallback`。当它从缓存中移除时，会清除 `rootNode`，强制在下次访问时获取新数据。有很多种方法可以操作缓存以使缓存项失效。目前，这里使用的是一个非常简化的解决方案。

重置 `SiteMap` 数据的第一种方式是通过简单的一小时超时期限，这在缓存项未被使用时有助于清理内存。我还假设在对照片相册数据库进行更改时，我能够控制 `SiteMap`。我为 `SqlSiteMapProvider` 创建了一个辅助类，用于从缓存中移除项。

8601Ch05CMP3 8/27/07 8:05 AM Page 145

## 第 5 章 ■ SQL 提供程序

**145**

这会触发 `CachedItemRemoved` 事件并清除 `rootNode`。在网站的代码中，每次添加或移除相册时，代码也会调用此辅助类来移除缓存项，并用新数据重新填充 `SiteMap`。

这不是一个非常优雅的方法，但它是可行的。在下一章中，你将学习到管理缓存数据的技术，这些方法要高效得多，无需通过手动调用来清除数据。

## COMMON 文件夹的补充

自定义的 `SiteMap` 提供程序对许多网站都很有用，因为标准的 XML 提供程序通常不够用。此示例可以放置在你的 `Templates` 文件夹下的 `Common` 文件夹中（`D:\Projects\Common\Templates\Provider Model`）。提供程序模型也是你希望在未来项目中实现的，因此你可能还想包含相册提供程序，该提供程序可从本书随附的示例代码下载中获取（在 Apress 网站的 Source Code 区域，http://www.apress.com）。

#### 总结

本章涵盖了三个主要的 ASP.NET SQL 提供程序，以及如何调整它们的行为以满足你的需求。你从头开始实现了一个提供程序，并将其与标准 SQL 提供程序的自定义实现集成在一起。本章涵盖的主题展示了你的应用程序如何将提供程序用作可插拔组件。这些组件在设计上是完全可互换的，为你提供了必要的灵活性，以重构你确定需要重构的任何应用程序部分。本章还向你展示了如果你只是调整配置，接口层将如何自动与多个提供程序实现协同工作。

8601Ch06CMP2 8/24/07 12:13 PM Page 147

