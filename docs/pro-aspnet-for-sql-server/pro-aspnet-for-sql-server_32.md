# 清单 5-27. SqlSiteMapProvider 的 Initialize 方法

```csharp
public override void Initialize(string name, NameValueCollection attributes)
{
    lock (this)
    {
        base.Initialize(name, attributes);
        connStringName = attributes["connectionStringName"].ToString();
        db = DatabaseFactory.CreateDatabase(connStringName);
        siteMapNodes = new List<DictionaryEntry>();
        childParentRelationship = new List<DictionaryEntry>();
        EnsureSiteMapLoaded();
    }
}
```

这里准备了初始基础设施，然后调用了 `EnsureSiteMapLoaded` 方法。实际上，在整个类中，如果 `SiteMap` 数据被更新并需要重新加载，此方法会在多个地方被调用。让我们看一下清单 5-28 中的这个方法。

