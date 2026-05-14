# 抽象提供程序

抽象提供程序是直接继承 `ProviderBase` 类的类，所有提供程序都构建于该基类之上。这个基类为所有提供程序定义了基本的支持规范，包括 `Name` 和 `Description` 属性，以及 `Initialize` 方法（该方法在提供程序首次加载到应用程序中时被调用）。这些属性和方法都使用 `virtual` 特性声明，因此你可以重写它们，但通常你只需重写 `Initialize` 方法。

在抽象提供程序中，你可以保持 `Initialize` 方法不变，留给具体的提供程序实现去处理，但你可能希望为提供程序的所有实现添加一个或多个通用属性，例如一个 `Enabled` 属性。你应该在抽象提供程序中完成这个操作。

现在，你要开始添加抽象属性和方法声明，以定义你的服务契约，所有实现都必须履行这个契约。对于 `PhotoAlbumProvider`，这些声明包括插入、更改和删除与相册和照片相关的数据的方法。

## 清单 5-20. `PhotoAlbumProvider.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Configuration.Provider;

namespace Chapter04.PhotoAlbumProvider
{
    /// <summary>
    /// Photo Album Provider
    /// </summary>
    public abstract class PhotoAlbumProvider : ProviderBase
    {
        #region " Abstract Methods "

        /// <summary>
        /// Gets albums for a user
        /// </summary>
        public abstract List<Album> GetAlbums(string userName);

        /// <summary>
        /// Gets photos for an album
        /// </summary>
        public abstract List<Photo> GetPhotosByAlbum(Album album);

        /// <summary>
        /// Creates an album
        /// </summary>
        public abstract Album AlbumInsert(string userName, string albumName, bool active, bool shared);

        /// <summary>
        /// Creates a photo
        /// </summary>
        public abstract Photo PhotoInsert(Album album, string photoName, DateTime photoDate, String regularUrl, int regularWidth, int regularHeight, String thumbnailUrl, int thumbnailWidth, int thumbnailHeight, bool active, bool shared);

        /// <summary>
        /// Updates an album
        /// </summary>
        public abstract void AlbumUpdate(Album album);

        /// <summary>
        /// Updates a photo
        /// </summary>
        public abstract void PhotoUpdate(Photo photo);

        /// <summary>
        /// Deletes an album
        /// </summary>
        public abstract void AlbumDeletePermanent(Album album);

        /// <summary>
        /// Deletes album permanently
        /// </summary>
        public abstract void PhotoDeletePermanent(Photo photo);

        /// <summary>
        /// Moves an album
        /// </summary>
        public abstract void AlbumMove(Album album, string sourceUserName, string destinationUserName);

        /// <summary>
        /// Moves a photo
        /// </summary>
        public abstract void PhotoMove(Photo photo, Album sourceAlbum, Album destinationAlbum);

        #endregion
    }
}
```

`PhotoAlbumProvider` 中的抽象方法定义了提供程序将支持的完整范围。

##### 提供程序实现

对于提供程序的具体实现，你可能需要读取更多的配置设置。一个 SQL 实现肯定会使用一个名为 `connectionStringName` 的配置属性，就像 `SqlMembershipProvider` 和 `SqlProfileProvider` 那样。当你准备初始化提供程序时，你需要验证将要使用的数据。首先检查 `config` 参数是否为 null，如果是则抛出 `ArgumentNullException` 异常。然后确保 `Name` 和 `Description` 这些核心属性已经设置，以便 `ProviderBase` 可以使用它们。描述可能未定义，因此你可以在未定义时为其赋予一个默认值。然后，你需要调用基类上的 `Initialize` 方法，在本例中基类是 `PhotoAlbumProvider`。这最终会调用 `ProviderBase` 中的 `Initialize` 方法，该方法将为 `Name` 和 `Description` 属性加载这些配置值。



接下来，你将仅处理与此实现相关的配置值，即 `connectionStringName` 属性。同样，你需要确保该属性已定义，并且它与连接字符串配置中的某个连接字符串相关联。如果一切就绪，你可以用它来初始化在整个类中使用的数据库连接。最后，你需要检查是否已处理所有配置属性，并且没有意外包含额外的属性。当你使用每个配置值时，将其从集合中移除，这样最终应该一个不剩。如果确实发现仍有属性残留，你可以抛出一个异常，提示提供了无法识别的属性。这看起来可能有点严格，但它能帮助你发现配置中拼写错误的属性，否则可能被忽略，从而导致提供程序出现无法解释的行为。清单 5-21 展示了 SQL 实现是如何初始化的。

### 清单 5-21. `SqlPhotoAlbumProvider.cs` 的初始化方法

```
public override void Initialize(string name, NameValueCollection config)
{
    if (config == null)
    {
        throw new ArgumentNullException("config");
    }
    if (String.IsNullOrEmpty(name))
    {
        name = "SqlPhotoAlbumProvider";
    }
    if (String.IsNullOrEmpty(config["description"]))
    {
        config.Remove("description");
        config.Add("description", "SQL Photo Album Provider");
    }
    base.Initialize(name, config);
    if (config["connectionStringName"] == null)
    {
        throw new ProviderException("Required attribute missing: connectionStringName");
    }
    connStringName = config["connectionStringName"].ToString();
    config.Remove("connectionStringName");
    if (WebConfigurationManager.ConnectionStrings[connStringName] == null)
    {
        throw new ProviderException("Missing connection string");
    }
    db = DatabaseFactory.CreateDatabase(connStringName);
    if (config.Count > 0)
    {
        string attr = config.GetKey(0);
        if (!String.IsNullOrEmpty(attr))
        {
            throw new ProviderException("Unrecognized attribute: " + attr);
        }
    }
}
```

有关此代码的完整清单，请参考附录 A。

##### 提供程序服务类

现在，你可以将所有这些部分组合在一起了。提供程序服务读取配置节，加载提供程序，并设置默认提供程序。正是在这里，提供程序与简单的程序集版本控制区分开来。该类最初是配置中的一个简单字符串，然后被实例化为一个类的实例。在你处理实现和配置问题以及梳理细节时，这可能也是你会花时间设置断点的地方。

这里需要记住的一个重要细节是，提供程序应该只初始化一次；否则，它将抛出异常。并且你需要以一种线程安全的方式来完成此操作。为了满足这些要求，你将使 `PhotoAlbumService` 类成为一个单例——这意味着默认构造函数是私有的，并且一个名为 `Instance` 的静态属性返回唯一的实例。

在这个属性中，提供程序将被加载，同时使用 `lock` 语句来确保线程安全（参见清单 5-22）。

### 清单 5-22. `PhotoAlbumService.cs`

```
using System.Configuration.Provider;
using System.Web.Configuration;

namespace Chapter04.PhotoAlbumProvider
{
    public class PhotoAlbumService
    {
        private static PhotoAlbumProvider _defaultProvider = null;
        private static PhotoAlbumProviderCollection _providers = null;
        private static object _lock = new object();

        private PhotoAlbumService() {}

        public PhotoAlbumProvider DefaultProvider
        {
            get { return _defaultProvider; }
        }

        public PhotoAlbumProvider GetProvider(string name)
        {
            return _providers[name];
        }

        public static PhotoAlbumProvider Instance
        {
            get
            {
                LoadProviders();
                return _defaultProvider;
            }
        }

        private static void LoadProviders()
        {
            // 加载提供程序的代码，但文本在此处结束。
        }
    }
}
```



// 如果提供程序已加载，避免获取锁

```csharp
if (_defaultProvider == null)
{
    lock (_lock)
    {
        // 再次检查，确保 _defaultProvider 仍为 null
        if (_defaultProvider == null)
        {
            // 获取 <PhotoAlbumSection> 配置节的引用
            PhotoAlbumSection section = (PhotoAlbumSection)
                WebConfigurationManager.GetSection("photoAlbumService");

            // 此处只需一个提供程序
            //_defaultProvider = (PhotoAlbumProvider)
            //    ProvidersHelper.InstantiateProvider
            //        (section.Providers[0], typeof(PhotoAlbumProvider));

            _providers = new PhotoAlbumProviderCollection();
            ProvidersHelper.InstantiateProviders(
                section.Providers, _providers,
                typeof(PhotoAlbumProvider));
            _defaultProvider = _providers[section.DefaultProvider];

            if (_defaultProvider == null)
                throw new ProviderException
                    ("无法加载默认的 PhotoAlbumProvider");
        }
    }
}
```

你可以看到，大部分工作是在 `LoadProviders` 方法中完成的。`photoAlbumService` 配置节被加载到 `PhotoAlbumSection` 对象中。然后，准备 `PhotoAlbumProviderCollection` 来容纳所有的提供程序实现。接着，你就会看到那行有趣的、使用了 `ProvidersHelper` 类的 `InstantiateProviders` 方法的代码。如果你曾大量使用反射进行开发，你会欣赏这行代码的简洁性。但它具体做了什么？如果你深入探究其内部机制，会发现它最终使用了 `System.Activator` 类的 `CreateInstance` 方法。该方法会根据配置设置来实例化指定类的一个实例。

这就是从头开始创建一个自定义提供程序的全部内容。只需搭建好一点基础架构，你就可以开始着手处理手头任务的“真正代码”了。

##### 单元测试

在构建你自己的自定义提供程序时，为了增强和加速开发过程，使用单元测试是极其有益的。提供程序是一种服务契约，而单元测试旨在验证预期的功能。这两者本就相辅相成。

对于 `PhotoAlbumProvider`，我从一开始就使用单元测试来检查每个新方法和实现的功能是否正确。运行测试能立即识别出破坏性变更，无论是由于 C# 代码中的逻辑错误还是存储过程的功能异常。当我选择完全重写实现时，通过针对提供程序接口运行单元测试，我能够在片刻之间彻底测试我的更改。如果我有多个提供程序实现，我还可以对所有实现运行相同的测试，以确保兼容性。

有关 `PhotoAlbumProvider` 的完整源代码清单，包括单元测试类以及创建表和存储过程的所有 SQL 脚本，请参考附录 A。

## 最终成品

随着 `SqlPhotoAlbumProvider` 的完成，你现在可以将一个网站配置为使用它，并开始展示那些相册和照片了。你只需将程序集放入网站的 `bin` 文件夹中，并配置 `Web.config` 文件以包含配置信息。清单 5-23 展示了一个示例配置。

**清单 5-23.** `PhotoAlbumProvider` 的 `Web.config`

```xml
<?xml version="1.0"?>
<configuration>
  <configSections>
    <section name="photoAlbumService"
             type="Chapter04.PhotoAlbumProvider.PhotoAlbumSection, Chapter04.PhotoAlbumProvider"/>
  </configSections>

  <!-- 其他配置 -->

  <photoAlbumService defaultProvider="SqlPhotoAlbumProvider">
    <providers>
      <clear/>
      <add name="SqlPhotoAlbumProvider"
           connectionStringName="chpt4"
           type="Chapter04.PhotoAlbumProvider.SqlPhotoAlbumProvider, Chapter04.PhotoAlbumProvider"/>
    </providers>
  </photoAlbumService>
</configuration>
```



### 第 5 章：SQL 提供程序

当几个 Web 窗体与`PhotoAlbumProvider`连接起来后，照片库很快就搭建完成了，如`图 5-3`所示。为了填充一些示例照片，我还创建了一个导入例程，该例程通过 Flickr 的标签拉取照片，以预填充几个示例相册。（所有源代码请参见附录 A。）

![Image 20](img/index-146_1.jpg)

`图 5-3.` 最终成品

#### 构建 SQL 站点地图提供程序

如果你仔细观察`图 5-3`中`PhotoAlbumProvider`的最终成品，你会在顶部看到一个熟悉的元素。它是一个面包屑导航轨迹，这是 ASP.NET 的一项标准功能。

遗憾的是，面包屑导航的唯一实现是基于 XML 的。为了使其与新的照片库协同工作，你需要一个 SQL 实现，该实现能够即时适应照片库的变化。自然地，你希望能够浏览不同的照片库，因此利用 ASP.NET 中与`SiteMapProvider`协同工作的导航控件是有意义的。

这一次，事情会稍微容易一些，因为有几个细节对你有利。首先，`SiteMapProvider`已经为你处理了配置加载。其次，你可以选择扩展现有的`SiteMapProvider`实现之一，或者从抽象的`SiteMapProvider`类开始实现。由于 MSDN 上有一个很好的示例展示了如何从基类构建实现，我以此为基础开始，并根据我的需求进行了调整。

> **注意** MSDN 上有大量关于示例代码和架构解释的内容。随着微软识别出需要更多关注的领域，这些内容一直在持续改进。在.NET 2.0 准备发布期间，我见证了文档的不断扩充。我发现每次重返 MSDN 时，都能发现以前没有的新内容。它现在还包含一个 Wiki，这样像我们这样的开发者就可以为 MSDN 贡献内容，以更好地完善整个文档和平台。如果你现在去看，你会看到大多数页面都允许你切换到不同版本的框架，如.NET 1.1、2.0、3.0 和 3.5。

站点地图是一种层次结构，它必须有一个唯一的父节点，并且每个节点必须有一个唯一的 URL。重要的是，在生成用于存储网站层次结构数据的数据库时，必须遵循这些规则；否则，站点地图将无法工作。

### SiteMap 要求

`SiteMap`像一棵树一样工作，并符合以下要求。首先，必须有一个根节点。从那里开始，每个节点必须有一个最终指向根节点的父节点。最后，每个节点必须有一个唯一的 URL。唯一 URL 的要求可能有些棘手，因为你可能允许一个叶子节点出现在网站的多个区域中，例如一个皮革椅子的产品。显示皮革家具的页面会指向这把椅子，显示椅子的页面同样会指向它。面包屑导航只能指向一个父节点，因此 URL 必须以某种方式是唯一的，即使它在概念上可能存在于两个不同的父节点下。

### 实现 SiteMapProvider

有了一个可工作的示例后，我开始调整它以适应我的数据库要求。这些要求非常简单，可以通过单个表轻松处理，如`表 5-4`所示。

`表 5-4.` `sm_SiteMapNodes`表

- `ID`: `Bigint` (主键)
- `ParentID`: `Bigint`
- `Url`: `Nvarchar(150)`
- `Title`: `Nvarchar(50)`
- `Depth`: `Int`
- `Creation`: `Datetime`
- `Modified`: `Datetime`

为了加载`SiteMap`，我硬编码了主页和主相册页面，然后读取了`PhotoAlbumProvider`使用的表。用于填充`SiteMap`的脚本如`清单 5-24`所示。

`清单 5-24.` `sm_RepopulateSiteMapNodes.sql`



