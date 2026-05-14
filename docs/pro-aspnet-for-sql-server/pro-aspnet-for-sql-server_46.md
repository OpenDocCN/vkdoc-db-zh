# 附录 A 照片相册

## 清单 A-3. `SqlPhotoAlbumProvider.cs`

```csharp
public abstract void PhotoMove(Photo photo, Album sourceAlbum, Album destinationAlbum);
```

```csharp
using System;
using System.Collections.Generic;
using System.Collections.Specialized;
using System.Configuration.Provider;
using System.Data;
using System.Data.Common;
using System.Web;
using System.Web.Configuration;
using Microsoft.Practices.EnterpriseLibrary.Data;
```

## `public class SqlPhotoAlbumProvider : PhotoAlbumProvider`

```csharp
#region " Variables "
string connStringName = String.Empty;
private Database db;
#endregion
```

### `Initialize` 方法实现

```csharp
#region " Implementation Methods "
/// <summary>
/// SQL 实现
/// </summary>
public override void Initialize(string name,
    NameValueCollection config)
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
        throw new ProviderException(
            "缺少必需属性：connectionStringName");
    }

    connStringName = config["connectionStringName"].ToString();
    config.Remove("connectionStringName");

    if (WebConfigurationManager.ConnectionStrings[connStringName] == null)
    {
        throw new ProviderException("缺少连接字符串");
    }

    db = DatabaseFactory.CreateDatabase(connStringName);

    if (config.Count > 0)
    {
        string attr = config.GetKey(0);
        if (!String.IsNullOrEmpty(attr))
        {
            throw new ProviderException("无法识别的属性：" + attr);
        }
    }
}
```

### `GetAlbums` 方法实现

```csharp
/// <summary>
/// SQL 实现
/// </summary>
public override List<Album> GetAlbums(string userName)
{
    // 使用缓存，窗口期为 5 秒
    String cacheKey = "PhotoAlbum::" + userName;
    object obj = HttpRuntime.Cache.Get(cacheKey);
    if (obj != null)
    {
        return (List<Album>)obj;
    }

    List<Album> albums = new List<Album>();
    try
    {
        using (DbCommand dbCmd =
            db.GetStoredProcCommand("pap_GetAlbumsByUserName"))
        {
            db.AddInParameter(dbCmd, "@UserName", DbType.String, userName);
            DataSet ds = db.ExecuteDataSet(dbCmd);

            // 填充相册集合
            if (ds.Tables.Count > 0)
            {
                foreach (DataRow row in ds.Tables[0].Rows)
                {
                    Album album = new Album();
                    album.LoadDataRow(row);
                    albums.Add(album);
                }
            }
        }
    }
    catch (Exception ex)
    {
        HandleError("pap_GetAlbumsByUserName 执行异常", ex);
    }

    // 缓存 5 秒
    HttpRuntime.Cache.Insert(cacheKey, albums, null,
        DateTime.Now.AddSeconds(5), TimeSpan.Zero);

    //返回结果
    return albums;
}
```



```
db.AddInParameter(dbCmd, "@UserName", DbType.String, userName); db.AddInParameter(dbCmd, "@Name", DbType.String, albumName); db.AddInParameter(dbCmd, "@IsActive", DbType.Boolean, active); db.AddInParameter(dbCmd, "@IsShared", DbType.Boolean, shared); db.AddOutParameter(dbCmd, "@AlbumID", DbType.Int64, 0); db.ExecuteNonQuery(dbCmd);

long albumId = (long)db.GetParameterValue(dbCmd, "@AlbumID"); ClearAlbumCache(userName);

List<Album> albums = GetAlbums(userName);

foreach (Album album in albums)
{
    if (album.ID == albumId)
    {
        return album;
    }
}
}
}
catch (Exception ex)
{
    HandleError("Exception with pap_InsertAlbum", ex);
}

throw new ApplicationException("New album not found");
}
```

```csharp
/// <summary>
/// SQL 实现
/// </summary>
public override Photo PhotoInsert(Album album, string photoName, DateTime photoDate,String regularUrl, int regularWidth, int regularHeight, String thumbnailUrl, int thumbnailWidth, int thumbnailHeight, bool active, bool shared)
{
    try
    {
        using (DbCommand dbCmd = db.GetStoredProcCommand("pap_InsertPhoto"))
        {
            if (photoName.Length > 40)
            {
                photoName = photoName.Substring(0, 40);
            }
            db.AddInParameter(dbCmd, "@AlbumID", DbType.Int64, album.ID);
            db.AddInParameter(dbCmd, "@Name", DbType.String, photoName);
            db.AddInParameter(dbCmd, "@PhotoDate", DbType.DateTime, photoDate);
            db.AddInParameter(dbCmd, "@RegularUrl", DbType.String, regularUrl);
            db.AddInParameter(dbCmd, "@RegularWidth", DbType.Int32, regularWidth);
            db.AddInParameter(dbCmd, "@RegularHeight", DbType.Int32, regularHeight);
            db.AddInParameter(dbCmd, "@ThumbnailUrl", DbType.String, thumbnailUrl);
            db.AddInParameter(dbCmd, "@ThumbnailWidth", DbType.Int32, thumbnailWidth);
            db.AddInParameter(dbCmd, "@ThumbnailHeight", DbType.Int32, thumbnailHeight);
            db.AddInParameter(dbCmd, "@IsActive", DbType.Boolean, active);
            db.AddInParameter(dbCmd, "@IsShared", DbType.Boolean, shared);
            db.AddOutParameter(dbCmd, "@PhotoID", DbType.Int64, 0); db.ExecuteNonQuery(dbCmd);

            long photoId = (long)db.GetParameterValue(dbCmd, "@PhotoID");

            List<Photo> photos = GetPhotosByAlbum(album);
            foreach (Photo photo in photos)
            {
                if (photo.ID == photoId)
                {
                    return photo;
                }
            }
        }
    }
    catch (Exception ex)
    {
        HandleError("Exception with pap_InsertPhoto", ex);
    }

    throw new ApplicationException("New photo not found");
}
```

```csharp
/// <summary>
/// SQL 实现
/// </summary>
public override void AlbumUpdate(Album album)
{
    try
    {
        using (DbCommand dbCmd = db.GetStoredProcCommand("pap_UpdateAlbum"))
        {
            db.AddInParameter(dbCmd, "@AlbumID", DbType.Int64, album.ID);
            db.AddInParameter(dbCmd, "@Name", DbType.String, album.Name);
            db.AddInParameter(dbCmd, "@IsActive", DbType.String, album.IsActive);
            db.AddInParameter(dbCmd, "@IsShared", DbType.String, album.IsShared);
            db.ExecuteNonQuery(dbCmd);
        }
    }
    catch (Exception ex)
    {
        HandleError("Exception with pap_UpdateAlbum", ex);
    }
}
```

```csharp
/// <summary>
/// SQL 实现
/// </summary>
public override void PhotoUpdate(Photo photo)
{
    try
    {
        using (DbCommand dbCmd = db.GetStoredProcCommand("pap_UpdatePhoto"))
        {
            db.AddInParameter(dbCmd, "@PhotoID", DbType.Int64, photo.ID);
            db.AddInParameter(dbCmd, "@Name", DbType.String,photo.Name);
            db.AddInParameter(dbCmd, "@PhotoDate", DbType.DateTime, photo.PhotoDate);
            db.AddInParameter(dbCmd, "@RegularUrl", DbType.String, photo.RegularUrl);
            db.AddInParameter(dbCmd, "@RegularWidth", DbType.Int32, photo.RegularWidth);
            db.AddInParameter(dbCmd, "@RegularHeight", DbType.Int32, photo.RegularHeight);
            db.AddInParameter(dbCmd, "@ThumbnailUrl", DbType.String, photo.ThumbnailUrl);
            db.AddInParameter(dbCmd, "@ThumbnailWidth", DbType.Int32, photo.ThumbnailWidth);
            db.AddInParameter(dbCmd, "@ThumbnailHeight", DbType.Int32, photo.ThumbnailHeight);
            db.AddInParameter(dbCmd, "@IsActive", DbType.Boolean, photo.IsActive);
            db.AddInParameter(dbCmd, "@IsShared", DbType.Boolean, photo.IsShared);
            db.ExecuteNonQuery(dbCmd);
            ClearAlbumCache(photo.Album.UserName);
        }
    }
}
```



## 清单 A-4. DataObject.cs

```csharp
catch (Exception ex)
{
    HandleError("执行 pap_UpdatePhoto 时发生异常", ex);
}
```

/// <summary>
/// SQL 实现
/// </summary>
/// <param name="album"></param>
public override void AlbumDeletePermanent(Album album)

8601Ch11AppCMP1 8/24/07 8:36 AM 第 353 页

**附录 ■ 照片相册**

**353**

{
    try
    {
        using (DbCommand dbCmd = db.GetStoredProcCommand("pap_DeleteAlbum"))
        {
            db.AddInParameter(dbCmd, "@AlbumID", DbType.Int64, album.ID);
            db.ExecuteNonQuery(dbCmd);
            ClearAlbumCache(album.UserName);
        }
    }
    catch (Exception ex)
    {
        HandleError("执行 pap_DeleteAlbum 时发生异常", ex);
    }
}

/// <summary>
/// SQL 实现
/// </summary>
public override void PhotoDeletePermanent(Photo photo)
{
    try
    {
        using (DbCommand dbCmd = db.GetStoredProcCommand("pap_DeletePhoto"))
        {
            db.AddInParameter(dbCmd, "@PhotoID", DbType.Int64, photo.ID);
            db.ExecuteNonQuery(dbCmd);
            ClearAlbumCache(photo.Album.UserName);
        }
    }
    catch (Exception ex)
    {
        HandleError("执行 pap_DeletePhoto 时发生异常", ex);
    }
}

/// <summary>
/// SQL 实现
/// </summary>
public override void AlbumMove(Album album, string sourceUserName, string destinationUserName)
{
    try

8601Ch11AppCMP1 8/24/07 8:36 AM 第 354 页

**354**

**附录 ■ 照片相册**

    {
        using (DbCommand dbCmd = db.GetStoredProcCommand("pap_MoveAlbum"))
        {
            db.AddInParameter(dbCmd, "@AlbumID", DbType.Int64, album.ID);
            db.AddInParameter(dbCmd, "@SourceUserName", DbType.String, sourceUserName);
            db.AddInParameter(dbCmd, "@DestinationUserName", DbType.String, destinationUserName);
            db.ExecuteNonQuery(dbCmd);
            ClearAlbumCache(sourceUserName);
            ClearAlbumCache(destinationUserName);
        }
    }
    catch (Exception ex)
    {
        HandleError("执行 pap_MoveAlbum 时发生异常", ex);
    }
}

/// <summary>
/// SQL 实现
/// </summary>
public override void PhotoMove(Photo photo, Album sourceAlbum, Album destinationAlbum)
{
    try
    {
        using (DbCommand dbCmd = db.GetStoredProcCommand("pap_MovePhoto"))
        {
            db.AddInParameter(dbCmd, "@PhotoID", DbType.Int64, photo.ID);
            db.AddInParameter(dbCmd, "@SourceAlbumID", DbType.Int64, sourceAlbum.ID);
            db.AddInParameter(dbCmd, "@DestinationAlbumID", DbType.Int64, destinationAlbum.ID);
            db.ExecuteNonQuery(dbCmd);
            sourceAlbum.ClearPhotos();
            ClearAlbumCache(sourceAlbum.UserName);
            destinationAlbum.ClearPhotos();
            ClearAlbumCache(destinationAlbum.UserName);
        }
    }
    catch (Exception ex)
    {

8601Ch11AppCMP1 8/24/07 8:36 AM 第 355 页

**附录 ■ 照片相册**

**355**

        HandleError("执行 pap_MoveAlbum 时发生异常", ex);
    }
}

#endregion

#region " 实用工具方法 "

private void HandleError(string message, Exception ex)
{
    //TODO 记录错误
    throw new ApplicationException(message, ex);
}

/// <summary>
/// SQL 实现
/// </summary>
public void ClearAlbumCache(String userName)
{
    String cacheKey = "PhotoAlbum::" + userName;
    if (HttpRuntime.Cache.Get(cacheKey) != null)
    {
        HttpRuntime.Cache.Remove(cacheKey);
    }
}

#endregion
}
}
```

using System;
using System.Data;
using System.Reflection;

namespace Chapter05.PhotoAlbumProvider
{
    /// <summary>
    /// 数据对象
    /// </summary>
    public abstract class DataObject : IComparable
    {
        /// <summary>
        /// 默认日期时间
        /// </summary>

8601Ch11AppCMP1 8/24/07 8:36 AM 第 356 页

**356**

**附录 ■ 照片相册**

        public readonly static DateTime DefaultDatetime = DateTime.Parse("01/01/1754");

        /// <summary>
        /// 对象 ID
        /// </summary>
        public abstract long ID { get; set; }

        /// <summary>
        /// 从 DataRow 加载值，并尝试将列名与属性名进行匹配。
        /// 例如 `row["Name"] = this.Name` 和 `row["IsActive"] = this.IsActive`，
        /// 其中字符串类型的属性被视为字符串，其他类型也得到适当处理。
        /// 支持的类型包括 String、Boolean、Float、Int 和 DateTime。
        /// 属性还必须设置为 public（而非 protected）且可写。
        /// </summary>
        /// <param name="row"></param>
        protected internal void LoadDataRow(DataRow row)
        {
            Type type = GetType();
            foreach (PropertyInfo pi in type.GetProperties())
            {
                if (pi.CanWrite)
                {
                    if (pi.PropertyType.Equals(typeof(DateTime)))
                    {
                        if (row[pi.Name] != null)
                        {
                            pi.SetValue(this,


## 清单 A-5. Album.cs

```csharp
GetNotNullDateTime(row, pi.Name), null);
}
}
else if (pi.PropertyType.Equals(typeof(Boolean)))
{
    if (row[pi.Name] != null)
    {
        if ("1".Equals(row[pi.Name]))
        {
            pi.SetValue(this, true, null);
        }
        else
        {
            pi.SetValue(this, false, null);
        }
    }
}
else if (pi.PropertyType.Equals(typeof(float)))
{
    if (row[pi.Name] != null)
    {
        pi.SetValue(this, GetNotNullFloat(row, pi.Name), null);
    }
}
else if (pi.PropertyType.Equals(typeof(String)))
{
    if (row[pi.Name] != null)
    {
        pi.SetValue(this, GetNotNullString(row, pi.Name), null);
    }
}
else if (pi.PropertyType.Equals(typeof(int)))
{
    if (row[pi.Name] != null)
    {
        pi.SetValue(this, GetNotNullInteger(row, pi.Name), null);
    }
}
else if (pi.PropertyType.Equals(typeof(Int64)))
{
    if (row[pi.Name] != null)
    {
        pi.SetValue(this, GetNotNullLong(row, pi.Name), null);
    }
}
}
}
}
```

### 摘要

工具方法

```csharp
protected internal DateTime GetNotNullDateTime(DataRow row, String name)
{
    Object obj = row[name];
    if (!DBNull.Value.Equals(obj))
    {
        return (DateTime)obj;
    }
    else
    {
        return DefaultDatetime;
    }
}
```

### 摘要

工具方法

```csharp
protected internal String GetNotNullString(DataRow row, String name)
{
    Object obj = row[name];
    if (!DBNull.Value.Equals(obj))
    {
        return (String)obj;
    }
    else
    {
        return String.Empty;
    }
}
```

### 摘要

工具方法

```csharp
protected internal int GetNotNullInteger(DataRow row, String name)
{
    Object obj = row[name];
    if (!DBNull.Value.Equals(obj) && obj is Int32)
    {
        return (int)obj;
    }
    else
    {
        return 0;
    }
}
```

### 摘要

工具方法

```csharp
protected internal long GetNotNullLong(DataRow row, String name)
{
    Object obj = row[name];
    if (!DBNull.Value.Equals(obj) && obj is Int64)
    {
        return (long)obj;
    }
    else
    {
        return 0;
    }
}
```

### 摘要

工具方法

```csharp
protected internal float GetNotNullFloat(DataRow row, String name)
{
    Object obj = row[name];
    if (!DBNull.Value.Equals(obj) && obj is float)
    {
        return (float)obj;
    }
    else
    {
        return 0;
    }
}

private DateTime _created = DefaultDatetime;
```

### 摘要

对象创建时间

```csharp
public DateTime Created
{
    get
    {
        return _created;
    }
    set
    {
        _created = value;
    }
}

private DateTime _modified = DefaultDatetime;
```

### 摘要

对象修改时间

```csharp
public DateTime Modified
{
    get
    {
        return _modified;
    }
    set
    {
        _modified = value;
    }
}
```

### 摘要

基类重写

```csharp
public int CompareTo(Object obj)
{
    int result = 0;
    if (obj is DataObject)
    {
        DataObject other = (DataObject) obj;
        result = Created.CompareTo(other.Created) * (-1);
    }
    return result;
}
```

### 摘要

基类重写

```csharp
public override int GetHashCode()
{
    return ID.GetHashCode();
}
```

### 摘要

基类重写

```csharp
public override bool Equals(object obj)
{
    if (obj is DataObject) {
        if (ID.Equals(((DataObject)obj).ID))
        {
            return true;
        }
    }
    return false;
}
}
}
```

### 命名空间与类声明

```csharp
using System;
using System.Collections.Generic;

namespace Chapter05.PhotoAlbumProvider
{
```

### 摘要

相册

```csharp
public class Album : DataObject
{
```

### 属性区域

```csharp
#region " 属性 "
private long _id = 0;
```

### 摘要

对象 ID

```csharp
public override long ID
{
    get
    {
        return _id;
    }
    set
    {
        _id = value;
    }
}

private string _username;
```

### 摘要

所有者的用户名

```csharp
public string UserName
{
    get
    {
        return _username;
    }
    set
    {
        _username = value;
    }
}

private string _name;
```

### 摘要

相册名称

```csharp
public string Name
{
    get
    {
        return _name;
    }
    set
    {
        _name = value;
    }
}

private bool _active = true;
```

### 摘要

指示相册是否处于活动状态

```csharp
public bool IsActive
{
    get
    {
        return _active;
    }
    set
    {
        _active = value;
    }
}

private bool _shared = true;
```

### 摘要

指示相册是否被共享

```csharp
public bool IsShared
{
    get
    {
```


# 附录 A: 相册提供程序

## Photo.cs

```csharp
using System;

namespace Chapter05.PhotoAlbumProvider
{
    /// <summary>
    /// 照片
    /// </summary>
    public class Photo : DataObject
    {
        #region " 属性 "

        private long _id = 0;

        /// <summary>
        /// 对象 ID
        /// </summary>
        public override long ID
        {
            get
            {
                return _id;
            }
            set
            {
                _id = value;
            }
        }

        private string _name;

        /// <summary>
        /// 照片名称
        /// </summary>
        public string Name
        {
            get
            {
                return _name;
            }
            set
            {
                _name = value;
            }
        }

        private DateTime _photoDate = DefaultDatetime;

        /// <summary>
        /// 照片拍摄日期
        /// </summary>
        public DateTime PhotoDate
        {
            get
            {
                return _photoDate;
            }
            set
            {
                _photoDate = value;
            }
        }

        private bool _active = true;

        /// <summary>
        /// 指示照片是否处于活动状态
        /// </summary>
        public bool IsActive
        {
            get
            {
                return _active;
            }
            set
            {
                _active = value;
            }
        }

        private bool _shared = true;

        /// <summary>
        /// 指示照片是否已共享
        /// </summary>
        public bool IsShared
        {
            get
            {
                return _shared;
            }
            set
            {
                _shared = value;
            }
        }

        private string _regularUrl = String.Empty;

        /// <summary>
        /// 照片的 URL
        /// </summary>
        public string RegularUrl
        {
            get
            {
                return _regularUrl;
            }
            set
            {
                _regularUrl = value;
            }
        }

        private int _regularWidth = 0;

        /// <summary>
        /// 照片的宽度
        /// </summary>
        public int RegularWidth
        {
            get
            {
                return _regularWidth;
            }
            set
            {
                _regularWidth = value;
            }
        }

        private int _regularHeight = 0;

        /// <summary>
        /// 照片的高度
        /// </summary>
        public int RegularHeight
        {
            get
            {
                return _regularHeight;
            }
            set
            {
                _regularHeight = value;
            }
        }

        private string _thumbnailUrl = String.Empty;

        /// <summary>
        /// 照片的缩略图 URL
        /// </summary>
        public string ThumbnailUrl
        {
            get
            {
                return _thumbnailUrl;
            }
            set
            {
                _thumbnailUrl = value;
            }
        }

        private int _thumbnailWidth = 0;

        /// <summary>
        /// 照片缩略图的宽度
        /// </summary>
        public int ThumbnailWidth
        {
            get
            {
                return _thumbnailWidth;
            }
            set
            {
                _thumbnailWidth = value;
            }
        }

        private int _thumbnailHeight = 0;

        /// <summary>
        /// 照片缩略图的高度
        /// </summary>
        public int ThumbnailHeight
        {
            get
            {
                return _thumbnailHeight;
            }
            set
            {
                _thumbnailHeight = value;
            }
        }

        private Album _album = null;

        /// <summary>
        /// 包含此照片的相册
        /// </summary>
        public Album Album
        {
            get
            {
                return _album;
            }
            set
            {
                // 应在由相册获取时设置
                _album = value;
            }
        }

        #endregion
    }
}
```

`代码清单 A-6. Photo.cs`

## PhotoAlbumService.cs

```csharp
using System.Configuration.Provider;
using System.Web.Configuration;

namespace Chapter05.PhotoAlbumProvider
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
            // 如果提供程序已加载，则避免获取锁
            if (_defaultProvider == null)
            {
                lock (_lock)
                {
                    // 再次执行此操作，以确保在 `_defaultProvider` 仍为 null 时执行
                    if (_defaultProvider == null)
                    {
                        // ... 提供程序加载逻辑 ...
                    }
                }
            }
        }
    }
}
```

`代码清单 A-7. PhotoAlbumService.cs`



### 附录 A. 照片库提供程序

#### 清单 A-8. `PhotoAlbumSection.cs`

```csharp
PhotoAlbumSection section = (PhotoAlbumSection) WebConfigurationManager.GetSection("photoAlbumService");

// 此处只需要一个提供程序
//_defaultProvider = (PhotoAlbumProvider) ProvidersHelper.InstantiateProvider(section.Providers[0], typeof(PhotoAlbumProvider));
_providers = new PhotoAlbumProviderCollection();
ProvidersHelper.InstantiateProviders(section.Providers, _providers, typeof(PhotoAlbumProvider));
_defaultProvider = _providers[section.DefaultProvider];
if (_defaultProvider == null)
    throw new ProviderException("无法加载默认的 PhotoAlbumProvider");
}
}
}
}
}

using System.Configuration;
namespace Chapter05.PhotoAlbumProvider
{
    public class PhotoAlbumSection : ConfigurationSection
    {
        [ConfigurationProperty("providers")]
        public ProviderSettingsCollection Providers
        {
            get { return (ProviderSettingsCollection)base["providers"]; }
        }

        [StringValidator(MinLength = 1)]
        [ConfigurationProperty("defaultProvider", DefaultValue = "SqlPhotoAlbumProvider")]
        public string DefaultProvider
        {
            get { return (string)base["defaultProvider"]; }
            set { base["defaultProvider"] = value; }
        }
    }
}
```

#### 清单 A-9. `PhotoAlbumProviderCollection.cs`

```csharp
using System;
using System.Configuration.Provider;
namespace Chapter05.PhotoAlbumProvider
{
    public class PhotoAlbumProviderCollection : ProviderCollection
    {
        public new PhotoAlbumProvider this[string name]
        {
            get { return (PhotoAlbumProvider)base[name]; }
        }

        public override void Add(ProviderBase provider)
        {
            if (provider == null)
                throw new ArgumentNullException("provider");
            if (!(provider is PhotoAlbumProvider))
                throw new ArgumentException("提供程序类型无效", "provider");
            base.Add(provider);
        }
    }
}
```

##### 表脚本

#### 清单 A-10. `pap_Albums.sql`

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND name = 'pap_Albums')
BEGIN
    PRINT '正在删除表 pap_Albums'
    DROP Table pap_Albums
END
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].pap_Albums NOT NULL,
    [UserName] nvarchar COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [Name] varchar COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [IsActive] char COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [IsShared] char COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [Modified] [datetime] NOT NULL,
    [Created] [datetime] NOT NULL,
    CONSTRAINT [PK_pap_Albums] PRIMARY KEY CLUSTERED
    (
        [ID] ASC
    )WITH (IGNORE_DUP_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY]
GO
SET ANSI_PADDING OFF
```

#### 清单 A-11. `pap_Photos.sql`

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND name = 'pap_Photos')
BEGIN
    PRINT '正在删除表 pap_Photos'
    DROP Table pap_Photos
END
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].pap_Photos NOT NULL,
    [AlbumID] [bigint] NOT NULL,
    [Name] nvarchar COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [PhotoDate] [datetime] NULL,
    [RegularUrl] nvarchar COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [RegularWidth] [int] NOT NULL,
    [RegularHeight] [int] NOT NULL,
    [ThumbnailUrl] nvarchar COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [ThumbnailWidth] [int] NOT NULL,
    [ThumbnailHeight] [int] NOT NULL,
    [IsActive] char COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [IsShared] char COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [Modified] [datetime] NOT NULL,
    [Created] [datetime] NOT NULL,
    CONSTRAINT [PK_pap_Photos] PRIMARY KEY CLUSTERED
    (
        [ID] ASC
    )WITH (IGNORE_DUP_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY]
GO
SET ANSI_PADDING OFF
GO
```

###### 约束脚本

#### 清单 A-12. `AddConstraints.sql`

```sql
IF EXISTS (select * from dbo.sysobjects where id =
object_id(N'[dbo].[FK_pap_Photos_pap_Albums]') and
OBJECTPROPERTY(id, N'IsForeignKey') = 1)
```



##### 存储过程脚本

### Listing A-13. DropConstaints.sql

`ALTER TABLE [dbo].[pap_Photos] DROP CONSTRAINT FK_pap_Photos_pap_Albums GO`

`ALTER TABLE [dbo].[pap_Photos] WITH CHECK ADD`

`CONSTRAINT [FK_pap_Photos_pap_Albums] FOREIGN KEY([AlbumID]) REFERENCES [dbo].[pap_Albums] ([id])`

`GO`

如果存在名为 `FK_pap_Photos_pap_Albums` 的外键约束，则删除它。
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'F'
AND name = 'FK_pap_Photos_pap_Albums')
BEGIN
    print 'Dropping FK_pap_Photos_pap_Albums'
    ALTER TABLE dbo.pap_Photos
    DROP CONSTRAINT FK_pap_Photos_pap_Albums
END
GO
```

### Listing A-14. pap_DeleteAlbum.sql

如果存储过程 `pap_DeleteAlbum` 已存在，则先删除它。
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'pap_DeleteAlbum')
BEGIN
    DROP Procedure pap_DeleteAlbum
END
GO
```

创建存储过程 `dbo.pap_DeleteAlbum`，它接受一个参数 `@AlbumID` 并删除对应的相册。
```sql
CREATE Procedure dbo.pap_DeleteAlbum
(
    @AlbumID bigint
)
AS
/* Assume child dependencies are deleted by provider */
DELETE FROM pap_Albums WHERE ID = @AlbumID
GO
```

授予 `PUBLIC` 角色执行该存储过程的权限。
`GRANT EXEC ON pap_DeleteAlbum TO PUBLIC GO`

### Listing A-15. pap_DeletePhoto.sql

如果存储过程 `pap_DeletePhoto` 已存在，则先删除它。
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'pap_DeletePhoto')
BEGIN
    DROP Procedure pap_DeletePhoto
END
GO
```

创建存储过程 `dbo.pap_DeletePhoto`，它接受一个参数 `@PhotoID` 并删除对应的照片。
```sql
CREATE Procedure dbo.pap_DeletePhoto
(
    @PhotoID bigint
)
AS
/* assume child dependencies have been deleted by provider */
DELETE FROM pap_Photos WHERE ID = @PhotoID
GO
```

授予 `PUBLIC` 角色执行该存储过程的权限。
`GRANT EXEC ON pap_DeletePhoto TO PUBLIC GO`

### Listing A-16. pap_GetAlbumsByUserName.sql

如果存储过程 `pap_GetAlbumsByUsername` 已存在，则先删除它。
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'pap_GetAlbumsByUsername')
BEGIN
    DROP Procedure pap_GetAlbumsByUsername
END
GO
```

创建存储过程 `dbo.pap_GetAlbumsByUserName`，它根据用户名 `@UserName` 查询并返回该用户的所有相册。
```sql
CREATE Procedure dbo.pap_GetAlbumsByUserName
(
    @UserName nvarchar
)
AS
SELECT * FROM pap_Albums
WHERE UserName = @UserName
ORDER BY Created DESC
GO
```

授予 `PUBLIC` 角色执行该存储过程的权限。
`GRANT EXEC ON pap_GetAlbumsByUsername TO PUBLIC GO`

### Listing A-17. pap_GetPhotosByAlbum.sql

如果存储过程 `pap_GetPhotosByAlbum` 已存在，则先删除它。
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'pap_GetPhotosByAlbum')
BEGIN
    DROP Procedure pap_GetPhotosByAlbum
END
GO
```

创建存储过程 `dbo.pap_GetPhotosByAlbum`，它根据相册 ID `@AlbumID` 查询并返回该相册中的所有照片。
```sql
CREATE Procedure dbo.pap_GetPhotosByAlbum
(
    @AlbumID bigint
)
AS
SELECT *
FROM pap_Photos
wHERE AlbumID = @AlbumID
ORDER BY Created DESC
GO
```

授予 `PUBLIC` 角色执行该存储过程的权限。
`GRANT EXEC ON pap_GetPhotosByAlbum TO PUBLIC GO`

### Listing A-18. pap_InsertAlbum.sql

如果存储过程 `pap_InsertAlbum` 已存在，则先删除它。
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'pap_InsertAlbum')
BEGIN
    DROP Procedure pap_InsertAlbum
END
GO
```

创建存储过程 `dbo.pap_InsertAlbum`，用于向 `pap_Albums` 表插入新相册。
```sql
CREATE Procedure dbo.pap_InsertAlbum
(
    @AlbumID bigint OUTPUT,
    @UserName nvarchar,
    @Name varchar,
    @IsActive char,
    @IsShared char
)
AS
INSERT INTO pap_Albums (UserName, [Name], IsActive, IsShared, Modified, Created)
VALUES (@UserName, @Name, @IsActive, @IsShared, GETDATE(), GETDATE())
SELECT @AlbumID = @@IDENTITY
GO
```

授予 `PUBLIC` 角色执行该存储过程的权限。
`GRANT EXEC ON pap_InsertAlbum TO PUBLIC GO`

### Listing A-19. pap_InsertPhoto.sql

如果存储过程 `pap_InsertPhoto` 已存在，则先删除它。
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'pap_InsertPhoto')
BEGIN
    DROP Procedure pap_InsertPhoto
END
GO
```

创建存储过程 `dbo.pap_InsertPhoto`，用于向 `pap_Photos` 表插入新照片。
```sql
CREATE Procedure dbo.pap_InsertPhoto
(
    @PhotoID bigint OUTPUT,
    @AlbumID bigint,
    @Name nvarchar,
    @PhotoDate [datetime],
    @RegularUrl nvarchar,
    @RegularWidth [int],
    @RegularHeight [int],
    @ThumbnailUrl nvarchar,
    @ThumbnailWidth [int],
    @ThumbnailHeight [int],
    @IsActive char,
    @IsShared char
)
AS
INSERT INTO pap_Photos (
    AlbumID, [Name], PhotoDate,
    RegularUrl, RegularWidth, RegularHeight,
    ThumbnailUrl, ThumbnailWidth, ThumbnailHeight,
    IsActive, IsShared, Modified, Created
) VALUES (
    @AlbumID, @Name, @PhotoDate,
    @RegularUrl, @RegularWidth, @RegularHeight,
    @ThumbnailUrl, @ThumbnailWidth, @ThumbnailHeight,
    @IsActive, @IsShared, GETDATE(), GETDATE()
)
SELECT @PhotoID = @@IDENTITY
GO
```

授予 `PUBLIC` 角色执行该存储过程的权限。
`GRANT EXEC ON pap_InsertPhoto TO PUBLIC GO`

### Listing A-20. pap_MoveAlbum.sql

如果存储过程 `pap_MoveAlbum` 已存在，则先删除它。
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
AND name = 'pap_MoveAlbum')
BEGIN
    DROP Procedure pap_MoveAlbum
END
GO
```

创建存储过程 `dbo.pap_MoveAlbum`，它接受源用户名 `@SourceUserName` 和目标用户名 `@DestinationUserName` 作为参数，用于移动指定相册 `@AlbumID` 的所有权。
```sql
CREATE Procedure dbo.pap_MoveAlbum
(
    @AlbumID bigint,
    @SourceUserName nvarchar,
    @DestinationUserName nvarchar
    -- (存储过程定义未完整提供)
)
```



# 附录：相册

## 列表 A-21. pap_MoveAlbum.sql

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
           AND name = 'pap_MoveAlbum')
BEGIN
    DROP Procedure pap_MoveAlbum
END
GO

CREATE Procedure dbo.pap_MoveAlbum
(
    @DestinationUserName nvarchar(256),
    @SourceUserName nvarchar(256),
    @AlbumID bigint
)
AS
    UPDATE pap_Albums SET UserName = @DestinationUserName
    WHERE UserName = @SourceUserName AND ID = @AlbumID
GO

GRANT EXEC ON pap_MoveAlbum TO PUBLIC
GO
```

## 列表 A-22. pap_MovePhoto.sql

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
           AND name = 'pap_MovePhoto')
BEGIN
    DROP Procedure pap_MovePhoto
END
GO

CREATE Procedure dbo.pap_MovePhoto
(
    @PhotoID bigint,
    @SourceAlbumID bigint,
    @DestinationAlbumID bigint
)
AS
    UPDATE pap_Photos SET AlbumID = @DestinationAlbumID
    WHERE AlbumID = @SourceAlbumID AND ID = @PhotoID
GO

GRANT EXEC ON pap_MovePhoto TO PUBLIC
GO
```

## 列表 A-23. pap_UpdateAlbum.sql

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
           AND name = 'pap_UpdateAlbum')
BEGIN
    DROP Procedure pap_UpdateAlbum
END
GO

CREATE Procedure dbo.pap_UpdateAlbum
(
    @AlbumID bigint,
    @Name varchar,
    @IsActive char,
    @IsShared char
)
AS
    UPDATE pap_Albums SET
        [Name] = @Name,
        IsActive = @IsActive,
        IsShared = @IsShared,
        Modified = GETDATE()
    WHERE ID = @AlbumID
GO

GRANT EXEC ON pap_UpdateAlbum TO PUBLIC
GO
```

## 列表 A-24. pap_UpdatePhoto.sql

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P'
           AND name = 'pap_UpdatePhoto')
BEGIN
    DROP Procedure pap_UpdatePhoto
END
GO

CREATE Procedure dbo.pap_UpdatePhoto
(
    @PhotoID bigint,
    @Name varchar,
    @PhotoDate [datetime],
    @RegularUrl nvarchar,
    @RegularWidth [int],
    @RegularHeight [int],
    @ThumbnailUrl nvarchar,
    @ThumbnailWidth [int],
    @ThumbnailHeight [int],
    @IsActive char,
    @IsShared char
)
AS
    UPDATE pap_Photos SET
        [Name] = @Name,
        PhotoDate = @PhotoDate,
        RegularUrl = @RegularUrl,
        RegularWidth = @RegularWidth,
        RegularHeight = @RegularHeight,
        ThumbnailUrl = @ThumbnailUrl,
        ThumbnailWidth = @ThumbnailWidth,
        ThumbnailHeight = @ThumbnailHeight,
        IsActive = @IsActive,
        IsShared = @IsShared,
        Modified = GETDATE()
    WHERE ID = @PhotoID
GO

GRANT EXEC ON pap_UpdatePhoto TO PUBLIC
GO
```

###### 网站类

### 列表 A-25. FlickrHelper.cs

```csharp
using System;
using System.Collections.Generic;
using System.Configuration;
using System.IO;
using System.Net;
using System.Web.Configuration;
using System.Xml;
using Chapter05.CustomSiteMapProvider;
using Chapter05.PhotoAlbumProvider;

/// <summary>
/// FlickrHelper 的摘要描述
/// </summary>
public class FlickrHelper
{
    public static void ImportFlickrPhotosToAlbum(Album album, string tag)
    {
        WebClient client = new WebClient();
        string url = String.Format(
            SiteConfiguration.FlickFeedUrlFormat, tag, "rss2");
        byte[] data = client.DownloadData(url);
        MemoryStream stream = new MemoryStream(data);
        XmlDocument document = new XmlDocument();
        XmlNamespaceManager nsmgr = new XmlNamespaceManager(document.NameTable);
        nsmgr.AddNamespace("media", "http://search.yahoo.com/mrss/");
        nsmgr.AddNamespace("dc", "http://purl.org/dc/elements/1.1/");
        document.Load(stream);
        document.Normalize();

        int max = 10;
        int count = 1;
        XmlNode channelNode = document.SelectSingleNode("/rss/channel");
        if (channelNode != null)
        {
            XmlNodeList itemNodes = channelNode.SelectNodes("item");
            foreach (XmlNode itemNode in itemNodes)
            {
                XmlNode titleNode = itemNode.SelectSingleNode("title");
                XmlNode dateTakenNode = itemNode.SelectSingleNode(
                    "dc:date.Taken", nsmgr);
                XmlNode regularUrlNode = itemNode.SelectSingleNode(
                    "media:content/@url", nsmgr);
                XmlNode regularWidthNode = itemNode.SelectSingleNode(
                    "media:content/@width", nsmgr);
                XmlNode regularHeightNode = itemNode.SelectSingleNode(
                    "media:content/@height", nsmgr);
                XmlNode thumbnailUrlNode = itemNode.SelectSingleNode(
                    "media:thumbnail/@url", nsmgr);
                XmlNode thumbnailWidthNode = itemNode.SelectSingleNode(
                    "media:thumbnail/@width", nsmgr);
                XmlNode thumbnailHeightNode = itemNode.SelectSingleNode(
                    "media:thumbnail/@height", nsmgr);

                try
                {
                    string title = "No Title";
                    DateTime dateTaken;
                    string regularUrl;
                    string thumbnailUrl;
                    int regularWidth, regularHeight,
```


```csharp
thumbnailWidth, thumbnailHeight;

if (titleNode != null && titleNode.FirstChild != null)
{
    title = titleNode.FirstChild.Value;
}

DateTime.TryParse(dateTakenNode.FirstChild.Value,
    out dateTaken);

regularUrl = regularUrlNode.Value;

int.TryParse(regularWidthNode.Value,
    out regularWidth);

int.TryParse(regularHeightNode.Value,
    out regularHeight);

thumbnailUrl = thumbnailUrlNode.Value;

int.TryParse(thumbnailWidthNode.Value,
    out thumbnailWidth);

int.TryParse(thumbnailHeightNode.Value,
    out thumbnailHeight);

PhotoAlbumProvider provider =
    PhotoAlbumService.Instance;

provider.PhotoInsert(album, title,
    dateTaken, regularUrl, regularWidth,
    regularHeight, thumbnailUrl,
    thumbnailWidth, thumbnailHeight,
    true, true);
}
catch (Exception ex)
{
    Utility.LogError("Error reading RSS item", ex);
}

count++;

if (count > max) {
    break;
}
}
}
}

public static void ExportFlickrPhotos(string tag)
{
    WebClient client = new WebClient();
    string feedUrlFormat = "http://api.flickr.com/services/feeds/" +
        "photos_public.gne?tags={0}&amp;format={1}";
    string url = String.Format(feedUrlFormat, tag, "rss2");
    byte[] data = client.DownloadData(url);

    MemoryStream stream = new MemoryStream(data);
    XmlDocument document = new XmlDocument();
    XmlNamespaceManager nsmgr = new XmlNamespaceManager(document.NameTable);
    nsmgr.AddNamespace("media", "http://search.yahoo.com/mrss/");
    nsmgr.AddNamespace("dc", "http://purl.org/dc/elements/1.1/");
    document.Load(stream);
    document.Normalize();

    XmlNode channelNode = document.SelectSingleNode("/rss/channel");
    if (channelNode != null)
    {
        XmlNodeList itemNodes = channelNode.SelectNodes("item");
        foreach (XmlNode itemNode in itemNodes)
        {
            XmlNode titleNode =
                itemNode.SelectSingleNode("title");
            XmlNode dateTakenNode =
                itemNode.SelectSingleNode("dc:date.Taken", nsmgr);
            XmlNode urlNode = itemNode.SelectSingleNode(
                "media:content/@url", nsmgr);
            XmlNode heightNode = itemNode.SelectSingleNode(
                "media:content/@height", nsmgr);
            XmlNode widthNode = itemNode.SelectSingleNode(
                "media:content/@width", nsmgr);
            XmlNode tagsNode = itemNode.SelectSingleNode(
                "media:category", nsmgr);

            string title = titleNode.FirstChild.Value;
            DateTime dateTaken;
            DateTime.TryParse(dateTakenNode.FirstChild.Value, out dateTaken);
            int width;
            int height;
            string photoUrl = urlNode.Value;
            string tags = tagsNode.InnerText;

            int.TryParse(widthNode.Value, out width);
            int.TryParse(heightNode.Value, out height);

            Console.WriteLine("title: " + title);
            Console.WriteLine("dateTaken: " + dateTaken);
            Console.WriteLine("photoUrl: " + photoUrl);
            Console.WriteLine("tags: " + tags);
            Console.WriteLine("width: " + width);
            Console.WriteLine("height: " + height);
            Console.WriteLine("title: " + title);
            //TODO use these values to download and save the file
        }
    }
}

public static void CreateFlickrAlbums(string username)
{
    PhotoAlbumProvider provider = PhotoAlbumService.Instance;
    Dictionary<String, String> defaultAlbums =
        new Dictionary<string, string>();

    defaultAlbums.Add("Forest Album", "forest");
    defaultAlbums.Add("Summer Album", "summer");
    defaultAlbums.Add("Water Album", "water");

    List<Album> albums = provider.GetAlbums(username);
    foreach (Album album in albums)
    {
        // start from scratch
        if (defaultAlbums.ContainsKey(album.Name))
        {
            // delete photos first
            foreach (Photo photo in album.Photos)
            {
                provider.PhotoDeletePermanent(photo);
            }
            provider.AlbumDeletePermanent(album);
        }
    }

    foreach (string albumName in defaultAlbums.Keys)
    {
        CreateFlickrAlbums(username, albumName, defaultAlbums[albumName]);
    }

    RepopulateSiteMap();
}

private static void CreateFlickrAlbums(string username, string albumName, string flickrTag)
{
    PhotoAlbumProvider provider = PhotoAlbumService.Instance;
    Album album = provider.AlbumInsert(username, albumName, true, true);
    ImportFlickrPhotosToAlbum(album, flickrTag);
}

private static void DeleteFlickAlbum(string username,
    string albumName)
{
```


# 附录：相册

#### SQL 站点地图提供程序

##### 类

#### 清单 A-25. `SqlSiteMapProvider.cs`

```csharp
PhotoAlbumProvider provider = PhotoAlbumService.Instance;
List<Album> albums = provider.GetAlbums(username);
foreach (Album album in albums)
{
    // 从零开始
    if (album.Name.Equals(albumName))
    {
        // 先删除照片
        foreach (Photo photo in album.Photos)
        {
            provider.PhotoDeletePermanent(photo);
        }
        provider.AlbumDeletePermanent(album);
    }
}

public static void RemoveFlickAlbum(string username,
    string albumName)
{
    DeleteFlickAlbum(username, albumName);
    RepopulateSiteMap();
}

public static void AddFlickrAlbum(string username,
    string albumName, string flickrTag)
{
    CreateFlickrAlbums(username, albumName, flickrTag);
    RepopulateSiteMap();
}

private static void RepopulateSiteMap()
{
    string connStringName = GetSqlSiteMapConnectionString();
    SqlSiteMapHelper helper = new SqlSiteMapHelper(connStringName);
    helper.RepopulateSiteMapNodes();
}

public static string GetSqlSiteMapConnectionString()
{
    string connStringName = String.Empty;
    SiteMapSection siteMapSection =
        ConfigurationManager.GetSection("system.web/siteMap") as SiteMapSection;
    if (siteMapSection != null)
    {
        string defaultProvider = siteMapSection.DefaultProvider;
        connStringName =
            siteMapSection.Providers[defaultProvider].
            Parameters["connectionStringName"];
    }
    return connStringName;
}
```

```csharp
// 源自 MSDN 上的 PlainTextSiteMapProvider 示例
// http://msdn2.microsoft.com/en-us/library/system.web.sitemapprovider.aspx
using System;
using System.Collections;
using System.Collections.Generic;
using System.Collections.Specialized;
using System.Data;
using System.Data.Common;
using System.Security.Permissions;
using System.Web;
using System.Web.Caching;
using Microsoft.Practices.EnterpriseLibrary.Data;

namespace Chapter05.CustomSiteMapProvider
{
    /// <summary>
    /// SqlSiteMapProvider 的摘要说明
    /// </summary>
    [AspNetHostingPermission(SecurityAction.Demand, Level =
        AspNetHostingPermissionLevel.Minimal)]
    public class SqlSiteMapProvider : SiteMapProvider
    {
        #region " 变量 "
        private string connStringName;
        private Database db;
        private SiteMapProvider _parentSiteMapProvider = null;
        private SiteMapNode rootNode = null;
        private List<DictionaryEntry> siteMapNodes = null;
        private List<DictionaryEntry> childParentRelationship = null;
        #endregion

        #region " 实现方法 "
        public override SiteMapNode CurrentNode
        {
            get
            {
                EnsureSiteMapLoaded();
                string currentUrl = FindCurrentUrl();
                // 查找代表当前页面的 SiteMapNode。
                SiteMapNode currentNode = FindSiteMapNode(currentUrl);
                return currentNode;
            }
        }

        public override SiteMapNode RootNode
        {
            get
            {
                EnsureSiteMapLoaded();
                return rootNode;
            }
        }

        public override SiteMapProvider ParentProvider
        {
            get
            {
                return _parentSiteMapProvider;
            }
            set
            {
                _parentSiteMapProvider = value;
            }
        }

        public override SiteMapProvider RootProvider
        {
            get
            {
                // 如果当前实例属于提供程序层次结构，则它不能是 RootProvider。需依赖 ParentProvider。
                if (ParentProvider != null)
                {
                    return ParentProvider.RootProvider;
                }
                // 如果当前实例没有 ParentProvider，
                // 则它不是层次结构中的子节点，可以是 RootProvider。
                else
                {
                    return this;
                }
            }
        }

        public override SiteMapNode FindSiteMapNode(string rawUrl)
        {
            EnsureSiteMapLoaded();
            // 根节点是否与 URL 匹配？
            if (RootNode.Url == rawUrl)
            {
                return RootNode;
            }
            else
            {
                SiteMapNode candidate;
                // 检索与 URL 匹配的 SiteMapNode。
                lock (this)
                {
                    candidate = GetNode(siteMapNodes, rawUrl);
                }
                return candidate;
            }
        }

        public override SiteMapNodeCollection GetChildNodes(SiteMapNode node)
        {
            EnsureSiteMapLoaded();
            SiteMapNodeCollection children = new SiteMapNodeCollection();
            // 遍历 ArrayList 并查找所有以指定节点为父节点的节点。
            lock (this)
            {
```


# 附录 ■ 照片集

## 私有辅助方法

```csharp
for (int i = 0; i < childParentRelationship.Count; i++)
{
    string nodeUrl = childParentRelationship[i].Key as string;
    SiteMapNode parent = GetNode(childParentRelationship, nodeUrl);
    if (parent != null && node.Url == parent.Url)
    {
        // Url 与 nodeUrl 对应的 SiteMapNode 是指定节点的子节点。
        // 获取 nodeUrl 的 SiteMapNode。
        SiteMapNode child = FindSiteMapNode(nodeUrl);
        if (child != null)
        {
            children.Add(child);
        }
        else
        {
            throw new Exception("ArrayLists 未同步。");
        }
    }
}
return children;
}

protected override SiteMapNode GetRootNodeCore()
{
    EnsureSiteMapLoaded();
    return RootNode;
}

public override SiteMapNode GetParentNode(SiteMapNode node)
{
    // 检查 childParentRelationship 表并找到当前节点的父节点。
    // 如果没有父节点，则当前节点是 RootNode。
    SiteMapNode parent;
    lock (this)
    {
        // 获取节点在 childParentRelationship 中的值
        EnsureSiteMapLoaded();
        parent = GetNode(childParentRelationship, node.Url);
    }
    return parent;
}

public override void Initialize(string name, NameValueCollection attributes)
{
    lock (this)
    {
        base.Initialize(name, attributes);
        connStringName = attributes["connectionStringName"].ToString();
        //SqlCacheDependencyAdmin.EnableNotifications(connString);
        db = DatabaseFactory.CreateDatabase(connStringName);
        siteMapNodes = new List<DictionaryEntry>();
        childParentRelationship = new List<DictionaryEntry>();
        EnsureSiteMapLoaded();
    }
}

#endregion

#region " 私有辅助方法 "

private SiteMapNode GetNode(List<DictionaryEntry> list, string url)
{
    for (int i = 0; i < list.Count; i++)
    {
        DictionaryEntry item = list[i];
        if (((string)item.Key).ToLower().Equals(url.ToLower()))
            return item.Value as SiteMapNode;
    }
    return null;
}

private string FindCurrentUrl()
{
    try
    {
        // 当前的 HttpContext。
        HttpContext currentContext = HttpContext.Current;
        if (currentContext != null)
        {
            return currentContext.Request.RawUrl;
        }
        else
        {
            throw new Exception("HttpContext.Current 无效");
        }
    }
    catch (Exception e)
    {
        throw new NotSupportedException(
            "此提供程序需要有效的上下文。", e);
    }
}

private void EnsureSiteMapLoaded()
{
    if (rootNode == null)
    {
        // 在内存中构建站点地图。
        LoadSiteMapFromDatabase();
    }
}

protected virtual void LoadSiteMapFromDatabase()
{
    lock (this)
    {
        // 如果根节点已存在，则 LoadSiteMapFromDatabase 已被调用过，
        // 该方法可直接返回。
        if (rootNode != null)
        {
            return;
        }
        else
        {
            // 清除集合和 rootNode 的状态
            Clear();
            SiteMapNode temp = null;
            DataSet nodes = LoadSiteMapNodes();
            if (nodes != null && nodes.Tables.Count > 0)
            {
                string baseUrl = HttpRuntime.AppDomainAppVirtualPath + "/";
                foreach (DataRow node in nodes.Tables[0].Rows)
                {
                    long parentNodeId = node["ParentID"] is long ?
                        (long)node["ParentID"] : 0L;
                    String url = node["Url"] as String;
                    String parentUrl = node["ParentUrl"] as String;
                    String title = node["Title"] as String;
                    temp = new SiteMapNode(this, baseUrl + url,
                        baseUrl + url, title);
                    // 这是否已是根节点？
                    if (null == rootNode && parentNodeId < 0)
                    {
                        rootNode = temp;
                    }
                    // 如果不是根节点，则将节点添加到各种集合中。
                    else if (parentUrl != null)
                    {
                        siteMapNodes.Add(
                            new DictionaryEntry(temp.Url, temp));
                        // 父节点已添加到集合中。
                        SiteMapNode parentNode = FindSiteMapNode(
                            baseUrl + parentUrl);
                        if (parentNode != null)
                        {
                            childParentRelationship.Add(
                                new DictionaryEntry(temp.Url, parentNode));
                        }
                        else
                        {
                            throw new Exception(
                                "未找到当前节点的父节点。");
                        }
                    }
                }
            }
        }
    }
    return;
}

private void Clear()
{
    rootNode = null;
    siteMapNodes.Clear();
    childParentRelationship.Clear();
}

/// <summary>
/// 从数据库获取站点地图节点
/// </summary>
/// <returns></returns>
internal DataSet LoadSiteMapNodes()
{
    String cacheKey = SqlSiteMapHelper.CACHE_KEY;
```


# 附录：照片集

## C# 代码：SqlSiteMapHelper 类

### 清单 A-26. `SqlSiteMapHelper.cs`

```csharp
object obj = HttpRuntime.Cache.Get(cacheKey);

if (obj != null)
{
    return obj as DataSet;
}

DataSet ds = null;
try
{
    using (DbCommand dbCmd = db.GetStoredProcCommand("sm_GetSiteMapNodes"))
    {
        ds = db.ExecuteDataSet(dbCmd);
    }
}
catch (Exception ex)
{
    HandleError("Exception with LoadSiteMapNodes", ex);
}

//SqlCacheDependency tableDependency =
// new SqlCacheDependency(connStringName, "sm_SiteMapNodes");
HttpRuntime.Cache.Insert(cacheKey, ds, null, DateTime.Now.AddHours(1), TimeSpan.Zero, CacheItemPriority.NotRemovable, OnRemoveCallback);

//return the results
return ds;
}

private void OnRemoveCallback(string key, object value, CacheItemRemovedReason reason)
{
    if (CacheItemRemovedReason.DependencyChanged == reason ||
        CacheItemRemovedReason.Removed == reason)
    {
        Clear();
        LoadSiteMapFromDatabase();
    }
    else
    {
        Clear();
    }
}

private void HandleError(string message, Exception ex)
{
    //TODO log error
    throw new ApplicationException(message, ex);
}
#endregion
}
}
```

```csharp
using System;
using System.Data.Common;
using System.Web;
using Microsoft.Practices.EnterpriseLibrary.Data;

namespace Chapter05.CustomSiteMapProvider
{
public class SqlSiteMapHelper
{
    public const string CACHE_KEY = "SqlSiteMapNodes";
    private Database db;

    public SqlSiteMapHelper(string connStringName)
    {
        db = DatabaseFactory.CreateDatabase(connStringName);
    }

    public void RepopulateSiteMapNodes()
    {
        try
        {
            using (DbCommand dbCmd = db.GetStoredProcCommand("sm_RepopulateSiteMapNodes"))
            {
                db.ExecuteNonQuery(dbCmd);
                InvalidateSiteMapCache();
            }
        }
        catch (Exception ex)
        {
            HandleError("Exception with RepopulateSiteMapNodes", ex);
        }
    }

    internal void InvalidateSiteMapCache()
    {
        HttpRuntime.Cache.Remove(CACHE_KEY);
    }

    private void HandleError(string message, Exception ex)
    {
        //TODO log error
        throw new ApplicationException(message, ex);
    }
}
}
```

##### 表脚本

### 清单 A-27. `sm_SiteMapNodes.sql`

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND name = 'sm_SiteMapNodes')
BEGIN
    DROP Table sm_SiteMapNodes
END
GO

CREATE TABLE [dbo].sm_SiteMapNodes NOT NULL,
    [ParentID] [bigint] NULL,
    [Url] nvarchar COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [Title] nvarchar COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL,
    [Depth] [int] NOT NULL CONSTRAINT [DF_sm_SiteMapNodes_Depth] DEFAULT ((0)),
    [Creation] [datetime] NOT NULL,
    [Modified] [datetime] NOT NULL,
    CONSTRAINT [PK_sm_SiteMapNodes] PRIMARY KEY CLUSTERED
    (
        [ID] ASC
    )WITH (PAD_INDEX = OFF, IGNORE_DUP_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY]
GO

GRANT SELECT ON sm_SiteMapNodes TO PUBLIC
GO
```

##### 存储过程脚本

### 清单 A-28. `sm_GetSiteMapNodes.sql`

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P' AND name = 'sm_GetSiteMapNodes')
BEGIN
    DROP Procedure sm_GetSiteMapNodes
END
GO

CREATE Procedure dbo.sm_GetSiteMapNodes
AS
SELECT c.ID, c.ParentID, c.Url, c.Title, c.Depth, p.Url AS ParentUrl
FROM sm_SiteMapNodes AS c
LEFT OUTER JOIN sm_SiteMapNodes AS p ON p.ID = c.ParentID
ORDER BY c.Depth, c.ParentID
GO

GRANT EXEC ON sm_GetSiteMapNodes TO PUBLIC
GO
```

### 清单 A-29. `sm_InsertSiteMapNode.sql`

```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P' AND name = 'sm_InsertSiteMapNode')
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
    SET @ID = (SELECT ID FROM sm_SiteMapNodes WHERE Url = @Url)
END
GO

GRANT EXEC ON sm_InsertSiteMapNode TO PUBLIC
GO
```



### **代码清单 A-30.** `sm_RepopulateSiteMapNodes.sql`
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

EXEC sm_InsertSiteMapNode -1, 'Default.aspx', '首页', 0,
@RootNodeID OUTPUT
EXEC sm_InsertSiteMapNode @RootNodeID, 'Albums/Default.aspx',
'相册', 1, @AlbumsNodeID OUTPUT

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
SET @Url = ( 'Albums/Album.aspx?AlbumID=' +
CONVERT(varchar(10), @AlbumID) +
'&UserName=' + @UserName )

-- PRINT 'Name = ' + @Name
-- PRINT 'UserName = ' + @UserName
-- PRINT 'Url = ' + @Url

EXEC sm_InsertSiteMapNode @AlbumsNodeID, @Url, @Name, 2,
@AlbumNodeID OUTPUT

SET @CurID = @CurID + 1
END

SET NOCOUNT OFF
GO

GRANT EXEC ON sm_RepopulateSiteMapNodes TO PUBLIC
GO
```

### **代码清单 A-31.** `sm_UpdateSiteMapNode.sql`
```sql
IF EXISTS (SELECT * FROM sysobjects WHERE type = 'P' AND
name = 'sm_UpdateSiteMapNode')
BEGIN
DROP Procedure sm_UpdateSiteMapNode
END
GO

CREATE Procedure dbo.sm_UpdateSiteMapNode
(
@ID bigint,
@ParentID bigint,
@Url nvarchar(150),
@Title nvarchar(50),
@Depth int
)
AS
IF EXISTS (SELECT * FROM sm_SiteMapNodes WHERE ID = @ID)
BEGIN
UPDATE sm_SiteMapNodes SET
ParentID = @ParentID,
Url = @Url,
Title = @Title,
Depth = @Depth,
Creation = GETDATE(),
Modified = GETDATE()
END
GO

GRANT EXEC ON sm_UpdateSiteMapNode TO PUBLIC
GO
```

### 索引

#### **数字和符号**
`@maximumRows` 输入参数, 67
`@OldFavoriteLinkID`, 值检查, 211–212
`@sortExpression` 输入参数, 67
`@startRowIndex` 输入参数, 67

#### **A**
-A 开关, 8
.abc 文件, 为其添加生成提供程序, 237
`AbcBuildProvider`, 用于 .abc 文件, 237–241
绝对过期, 为缓存数据设置, 149–150
`abstract 属性`, 317
抽象属性与方法声明, 132–133
Active Directory, 网站与之集成, 203
`Add 方法`, 160
Add Provider Services.cmd 脚本, 8
Add SQL Cache Dependencies.cmd, 170–171
`AddConstraints.sql`, 65
`AddIndexing.sql`, 63–64
Admin 角色, 用以限制 Admin 部分的访问, 117–118
Admin 部分, 保护, 34–35
Admin 用户, 创建, 35–36
`AdminHyperlink`, 设置其 `Visibility`, 117
应用程序状态, 作为缓存的替代方案, 149–150
`Application_Start` 方法, 284–285
ASP.NET, 其中的缓存, 147–148
ASP.NET 2.0 应用程序
代码与数据库分离, 7
常用文件夹, 4–5
数据源配置, 5–7
其 `DATA_DIRECTORY`, 12
管理提供程序服务, 7–11
准备环境, 1–7
随其发布的 `SqlMembershipProvider`, 112
ASP.NET 2.0 数据模型。*参见* 数据模型
ASP.NET 控件, 使用提供程序驱动的, 126–128
抽象提供程序类, 132–133
ASP.NET machineKey 生成器, 129
ASP.NET 成员资格用户帐户, 随 ASP.NET 2.0 发布, 112
ASP.NET 提供程序
配置, 11–18
在 ASP.NET 2.0 中引入, 7
管理服务, 7–11
混合与匹配, 11
照片库提供程序, 372
移除服务, 8–10
ASP.NET 运行时, 通过配置文件访问值, 125
`aspnet_regsql.exe` 工具, 8
`AspNet_SqlCacheRegisterTableStored-Procedure`, 已更新, 168–170
`AssemblyHelper 类`, 312–313



Akamai，托管于，294

断言语句，检查返回值

变更脚本，手动编写语句

with，71

with，272–273

AttachedProperty 方法，239–240

变更语句，更新数据库

attributeMapEmail 属性，用于 Active

结构与，272–273

Directory 实现，113

Always Show Solution 选项，设置，3

认证配置文件，从迁移

匿名方法，使用委托，166–167

匿名到，121–122

匿名配置文件

认证系统，其安全性于

与认证配置文件对比，121

ASP.NET，126–128

配置，119–120

认证令牌，126

使用 T-SQL 删除不活动的，120

AutoScaffold 页面，SubSonic 代码生成器，

启用，17, 119

248–249

迁移，121–122

■

Apache Web 服务器 vs. Linux 到 IIS 于

#### B

后端，分发至，296–297

App_Code 文件夹，`Utility` 类于，17

银行网站，保护访问，126–128

App_Data 文件夹，`DATA_DIRECTORY` 于，12

`BeforeBuild` 和 `AfterBuild` 目标，258

**399**

8601Ch12IndexCMP2 8/27/07 7:09 PM Page 400

## 400

■索引

绑定

模拟，163

输入参数，81–84

超时，对可伸缩性的影响，290

用户控件，83–84

`CaptureSettings` 方法，103–104

BirthDate 列，使用 `COALESCE`

`Category` 属性，226

函数于，207

变更脚本，生成，272–273

Blinq 代码生成器，用于 ASP.NET，249–252

变更脚本文件夹，用于更新脚本，273

Blinq 项目，ASP.NET 网站上的原型，

`Chapter09Configuration` 类，使用加载

249–250

配置，287–288

BLOB 存储机制，被

`Chapter09Section` 类，286–287

`SqlProfileProvider` 使用，125

`Chapter09SectionGroup` 类，构建，

BLOBs，动态配置文件与配置文件作为，

285–286

124–125

`chpt02_GetAllPeople.sql` 存储过程，

瓶颈，对可伸缩性的影响，291

`BoundField`，用 `TemplateField` 替换，78

`chpt03_GetPeopleRowCount.sql` 存储

面包屑导航，实现，139

过程，68–69

`Browsable` 属性，226

`chpt03_GetPeopleSubSetSorted.sql` 存储

`Build` 操作，设置为嵌入式资源，

过程，66–67

`chpt03_SaveLocation.sql` 脚本，60–61

构建提供程序，234–236

`chpt03_SavePerson.sql` 脚本，59

构建脚本，在解决方案级别创建，

`chpt03_SavePersonWithLocation.sql` 脚本，

258–260

`build.proj` 脚本

`chpt07_GetFavoriteLinksByProfileID.sql`

被 `Build` 目标调用，269–270

存储过程，209

在解决方案级别创建，258–260

`chpt07_GetFavoriteLinksByTag.sql` 存储

`BuildProvider` 类，于

过程，209–210

`System.Web.Compilation`

`chpt07_PurgeFavoriteLinksByProfileID`

命名空间，235–236

存储过程，216–217

`buildProviders` 配置元素，234

`chpt07_PurgeLinkTagsByProfileID` 存储

业务逻辑，封装于存储过程，

过程，215–216

198–199

`chpt07_SaveFavoriteLink` 存储过程，

业务对象，创建，222

参数，211

`chpt09_GetSchemaVersion.sql` 脚本，

#### C

278–279

缓存

`chpt09_SchemaVersions.sql` 脚本，277

通过索引添加和获取项，

`chpt09_SetSchemaVersion.sql` 脚本，

160–161

277–278

添加依赖项，168

`CityListingControl.ascx`，85–86

从中移除项，165

类库，结构，280

`Cache` 对象，147

`ClassLibrary`，创建，218

缓存的数据，164–165

`ClearCachedItem` 方法，247

`CacheItemPriority` 参数，值，161

客户端缓存，294–295

`CacheItemRemovedCallBack` 参数，

`COALESCE` 函数，207–208

162–163

代码生成，233–234

`CacheItemRemovedReason` 参数，

代码片段管理器，45

值，162

代码片段选择菜单，46

缓存

代码片段。*参见* 数据访问代码片段

添加以启用轮询，160

`CodeAssignStatement` 类，240

替代方案，148

`CodeCompileUnit`，237–238

枚举缓存，161

`CodeDom` 命名空间，237–242

使用缓存提高性能，147

`CodeFieldReferenceExpression` 类，240

方法，160

`CodeMemberProperty` 类型，240

选项，153–163

`CodeMethodReturnStatement` 类，240

参数，161–163

`CodeNamespace` 类型，237–238

性能策略，187–189

`CodeNamespaceImport` 类型，237–238

问题，186

CodePlex

代理，294–295

SubSonic 部分托管于，242–243

按类型清除缓存项，161

网站，242

8601Ch12IndexCMP2 8/27/07 7:09 PM Page 401

## ■索引

**401**



`CodeSnippetCompileUnit` 类，236

`自定义配置节`，*参见*

`CodeTypeDeclaration` 类型，237–238

`配置`
- `命令行`，添加提供程序服务
- `自定义节定义`，288
- `来源`，8–10

`Common 文件夹`
- ■**D**
- 添加 `SubSonic` 和 `Blinq` 工具至，250
- `dacreation` 快捷方式，219
- 添加自定义数据绑定控件至，109
- `dagetdr` 快捷方式，220
- 为 SQL 通知添加内容，175
- `dagetds` 快捷方式，219
- `公共文件夹`。*参见* `文件夹`
- `danonquery` 快捷方式，220

`逗号分隔值` (`CSV`) 导出文件，

`数据`
- 生成，125
- 从数据库删除，213–217

`CompositeDataBoundControl`,
- 加载，222–224
- 为其调用 `CreateChildControls`，92
- 单元测试，68–71

`configSource` 属性选项，`Web`

`数据访问应用程序块`，37–39

`部署项目`，264

`数据访问层` (`DAL`)，191–231
- `配置`
- 创建以配合数据绑定工作，
- `匿名配置文件`，119–120
- 217–225
- 创建自定义节，285–288
- `生成的`，233–253
- `自定义的`，314–321
- `关注点分离`，218–219

`更快查找` a
- 为 Chapter09Group 自定义，288
- `建议的默认值`，207
- `数据源的`，5–7

`数据访问方法`，创建，219–221

`层次结构的总体结构`，315

`数据访问提供程序`，创建，303–310

`分组`，315–317

`数据访问类型`，代码片段，40–41

`照片相册提供程序的节类，`
`数据绑定`
- t
- 130–131
- `代码`，102–103
- `http://superinde`
- `成员资格提供程序设置`，14
- `关于递归绑定的警告`，86
- `配置文件提供程序设置`，16

`数据缓存`，159–163。*另请参见* `缓存`

`Configuration` 属性，在 `MSBuild` 中设置
`数据契约`，201–203
- `脚本`，260

`数据示例`
- `ConfigurationSectionGroup` 类，315
- `非平凡的`，49

`连接字符串`，用于 `AP` 数据库，297
`平凡的`，47–49

`x.apr`

`connectionStringName` 属性，113

`数据加载`，调优，224–225

在 `SqlMembershipProvider` 配置中，

`数据操作方法`，`获取`、`保存` 和
- `ess.com/`
- `删除`，300–303

`connectionStrings`，在 `Machine.config` 文件中，12

`数据模型`，选择，37–54

`控制台应用程序`，用于托管服务，
`数据序列化`，配置文件属性，124
- 335–336

`数据传输对象` (`DTO`)，`DataSets` 用作，

`约束与索引`，管理，62–65

`内容分发网络`，`Akamai`，294

`数据值`，转换为属性，224

`数据仓库`，作为性能策略，
- `content-disposition 头`，125
- 187–188

`持续集成`，工具，71

`数据库`
- `控件集合`，创建，104
- `自动化更新`，285–288
- `ControlState`，89–91
- `构建`，203–225
- `CountryListingControl.ascx`，84–85
- `配置`，219

`CreateChildControls` 方法
- `从其中删除数据`，213
- 用于 `CompositeDataBoundControl`，92
- `部署`，271–288
- 获取传入的数据，94–97
- `初始化文件`，51
- 实现，93–94

`Created` 值，添加至表，205–206
`管理`，55–71

`CreatedUser` 事件处理程序，124
`按日期分区`，299

`CreatePagerControls` 方法，98–99
`数据模型示例`，46–47

`CreateUserWizard`，122–124
`更新`，279–280

`CRUD`，233
`数据库创建`，41
- `方法`，生成，47

`Database` 对象，38
- `存储过程`，59

`数据库项目`，55–57

`CruiseControl.NET`，71
- `使用其构建数据库`，208–217

`当前上下文`，在其中持有数据，150–153

`8601Ch12IndexCMP2 8/27/07 7:09 PM Page 402`

**402**

■`索引`

`数据库服务器`
- `分发服务`，295–296
- `适用于单一服务器的小型公司配置，`
`DLINQ`，查询表，322
- 296–297

`无操作按钮`，107

- `更新后的成长型公司配置，`
`DomainKey` 类型，307–309

`DomainObject` 类
- `与其进行 Web 服务器通信`，148
- 比较方法，309–310

`DatabaseManager` 类，加载并运行
- 修订版，307–310, 341
- `脚本`，280–285

`dotnetUserGroup` 元素，315

`DataBinder`，用其加载数据值，
- `Dropconstraints.sql`，照片相册提供程序，
- 102–103

`数据绑定控件`
`DTO`。*参见* `数据传输对象` (`DTO`)
- `创建`，91–110

`DugConfiguration` 类，315–317
- `嵌入`，84

`DugDataContext` 类，创建，325–327
`DataContracts`，定义，336

`dug_DeleteEvent.sql` 存储过程，
- `DataFirstNameField` 属性，103
- 302–303

`DataObject`，53–54
`dug_GetEvent.sql` 存储过程，300

`DataObjectMethod` 属性，52
`dug_GetEventsByDate.sql` 存储过程，
- `DataObjectMethodType` 属性，199–200
- 300–301

`DataObjects`，199–203


#### E

`inline SQL`和存储过程

`EditTemplate`和`DateEditor`放置于此

嵌入式脚本的使用

使用`FavoriteLinkCollection`对象进行填充

`EndDaysBack`属性

创建 Service Broker 端点

重构

`Enterprise Library`

数据访问应用程序块作为其一部分

数据源配置

`DataSourceSelectArguments`方法调整

`DataSourceViewSelectCallback`，回调委托

`DateEditor`用户控件

`DateTime`类型，转换其值

数据类型快捷方式

`db.proj`脚本，用于数据库更新

`DBNull.Value`

调试

在`DataSets`中设置断点支持

不使用`ViewState`进行调试

`defaultProvider`属性

`DefaultValue`属性

`deploy.proj`脚本

部署

管理部署文件的创建

设计契约

`DetailsView`控件

创建差异虚拟硬盘

`Digg`

分布式后端。*参见* 后端

索引

#### F

`/f`开关，`Binq`代码生成器

`Favorite Link`记录

检查现有记录

使用`INSERT`或`UPDATE`保存

`Favorite Links`网站

中央业务对象

用于获取数据的存储过程

`FavoriteLink`对象

测试其性能

在调试器中查看

`FavoriteLinkCollection`对象

`FavoriteLinkDomain`类，创建

`FavoriteLinks`数据库

从数据库中删除数据

管理脚本

数据库中的`MembershipUser`记录

关系

防止空值

`FetchByID`方法，带缓存

`FetchPersonsByLocationID`方法

编辑和验证字段

`FirstName`数据字段属性

构建使用`Flickr`照片网站的提供程序

`FlickrHelper.cs`，`Website`类

创建公共文件夹

外键约束

处理

移除和添加

`FormatException`，保存日期值时发生

`FormsAuthentication`令牌的分配

`GetClassName`方法

`GetContents`方法

`GetData`方法

`GetFieldName`方法

`GetGeneratedCode`方法

`GetLocationConsumers`方法

`GetLoginTimeout`方法

`GetManifestResourceStream`方法

`GetNamespace`方法

`GetPeopleRowCount`方法

`GetPeopleSubSetSortedDataSet`方法

`GetPerson`方法，来自`Person`类

`GetPersonsByLocation`方法

`GetProduct`方法，在`Utility`类中

`GetRecentFavoriteLinks`方法

`GetSqlCommands`方法

`GetTotalRowCount`

`GridView`控件

数据绑定

使用`TemplateField`

`Guidance Automation Extensions for VS`

#### H

`Hao Kung`

`Home.aspx`页面，创建

由`Akamai`托管

`FirstName`数据字段属性

#### I

`IDataItemContainer`成员

`IDataReader`，加载数据

`IEventService`接口，WCF 提供程序

`ILocationConsumer`接口

索引

向表添加索引

索引和约束管理

碎片整理

通过索引提高性能

`inline SQL`和存储过程

#### G

*   `生成变更脚本` 按钮，62, 272
*   `GenerateCode` 方法，`BuildProvider` 类，260
*   GeoIP 服务，128
*   `GetAllPeopleDataSet` 方法，51–52
*   `GetAllPeopleReader`，添加到 `PersonDomain`，53
*   `GetAllPeopleReader`，添加到 `PersonDomain`，53

#### H

*   无条目

#### I

*   索引创建脚本，273
*   `init.sql` 脚本，作为嵌入资源，279
*   `InitializeDatabase` 方法，281
*   `InitializePagerControls` 方法，99
*   内联 SQL
    *   片段缓存，155
    *   维护注意事项，196
    *   用户控件输出的，153
    *   安全注意事项，196–198
    *   与后缓存替换一起使用，156–159
*   输入参数，绑定，81–84
*   `InputParameterExample.aspx`，81
*   `InputParameterExample2.aspx`，82
*   `Insert` 方法，160
*   IntelliSense 支持，针对生成的类，234
*   `Interactive` 属性，在 MSBuild 脚本中设置，260
*   `InvalidCastException`，192
*   IP 地址，根据其确定物理位置，128
*   `IsLocationInUse` 方法，被调用的方法，311–312
*   `IsRemoteAddressMatched` 方法，127–128
*   `IsUsingLocation` 方法，311–312
*   `ItemGroup` 元素，MSBuild 脚本，257

#### J

*   JetBrains，179

#### K

*   Kung, Hao，125

#### L

*   懒加载，188–189
*   `LinksControl` 用户控件，创建，226–228
*   LINQ 提供程序，实现，321–332
*   `LinqEventProvider` 类，创建，328–332
*   Linux 上的 IIS 对比 Apache Web 服务器在 Windows NT 上，291
*   `LiteralControls`，被 `PersonRow` 使用，104–105
*   `Load` 事件，对数据加载的影响，152
*   加载时间和页面大小，87
*   负载均衡硬件，用于 Web 场，293–294
*   `LoadControlState` 方法，90, 108
*   `LoadViewState` 方法，106
*   `Location` 属性，其访问器，310–311
*   `LocationManager` 类，311
*   位置
    *   保存，60–61
    *   使用，311–313
*   日志记录，使用存储过程处理，196
*   `Login Authenticate` 事件处理程序，126–127
*   `Login` 控件，112
*   登录页面，其配置，116–117

#### M

*   Machine.config 文件，11–12
*   Management Studio。*参见* SQL Server Management Studio
*   多对多关系，205
*   `MarkDirty` 方法，244–245
*   标记代码，创建高效的，104
*   主密钥，定义，173
*   `MaxIndex` 属性，98
*   `MembersControl.ascx`，其中包含的控件，18
*   `Membership` API，使用其创建用户，122–124
*   成员资格提供程序，其配置，13–14。*另请参阅* ASP.NET 提供程序
*   成员资格系统，使用表单身份验证实现，118
*   `MembershipProvider` 类，112–113
*   Microsoft Patterns & Practices 小组
    *   VS 的指导自动化扩展，37
*   Microsoft Virtual PC 环境，准备，1–2
*   `Modified` 值，添加到表中，205–206
*   MSBuild，使用其进行自动化，256–260
*   MSBuild Community Tasks，257
*   MSBuild 脚本
    *   `BeforeBuild` 和 `AfterBuild` 目标，258
    *   Common 文件夹的补充，271
    *   用于数据库更新，274–276
    *   其参数，260
    *   其主要元素，257
    *   使用规则，256
    *   基本骨架，257
*   MSDN，其上的可用内容，140

#### N

*   `Name` 属性，分解，201–202
*   `/namespace` 开关，Blinq 代码生成器，95
*   `NamesUpdate-01.sql` 脚本，280
*   .NET 3.0 with WCF，由 Microsoft 发布，332
*   .NET 用户组网站
    *   为其创建数据提供程序，303–310
    *   为其创建 `EventProvider` 对象，303–306
    *   示例应用程序，298–303
*   `netTcpBinding`，333
*   网络设备，对网络瓶颈的影响，291
*   NHibernate，O/R 映射工具，222
*   `NonQuery` 方法，44–45
*   非平凡数据示例，49
*   `非类型化数据集`，51–53, 192–193
*   通知系统，授予使用权限，174–175
*   空值，防止，207–208
*   空值
    *   将默认日期转换为空值，207–208
    *   处理，206–207
*   NUnit，179
*   .NET 的单元测试框架，70

#### O

*   对象/关系 (O/R) 映射，222–224
*   `ObjectDataSource`，199–203
    *   向 Web 窗体添加第二个，52
    *   其提供的信息，67–70
*   `ObjectDataSource` 声明，针对 `Person` 表，250–252
*   `ObjectDataSource1`
    *   供数据绑定控件使用，69–70
*   `ObjectDataSource1_Selecting` 事件处理程序，69–70
*   `OnChange` 处理程序方法，166
*   一对多关系，205

#### P

*   个性化提供程序。*参见* ASP.NET 个性化提供程序


## 一对一关系
205

## 提供程序

### OnPreRender 方法
设置文本属性

### PersonID 参数
设置，59

位于，104–105

### PersonListing.aspx.cs 代码隐藏文件
83–84

### OnSelecting 事件处理程序
82

### PersonListingControl.ascx
83

##### 输出缓存
153

### PersonRow
创建，101–105

已启用，154

## 相册提供程序
343–398

相关问题，155

构建，129–138

以编程方式为其设置页面，154–155

##### 类
344–371

###### 配置
343

涉及语言时的设置，186

###### 约束脚本
372–373

实现，134–135

### 要求
130

##### 存储过程脚本
373–379

##### 表脚本
371–372

##### 单元测试
137–138

### PhotoAlbumProvider
其 Web.config，138

### PhotoAlbumProvider.cs 类
132–133, 344

### PhotoAlbumProviderCollection.cs 类
131

### PhotoAlbumSection.cs 类
130–311, 369

### PhotoAlbumService.cs
135–137, 368

###### 轮询

为 `SqlCacheDependency` 进行配置，171–172

为表启用，168–171

相关问题，172

对比 查询通知，172–173

## 缓存后替换
155–159

## PreBuildEvent 和 PostBuildEvent 属性
258

## PreferredCustomer 角色
将用户添加到，117

## PreLoad 事件
99–100

## ProcessFieldNodes 方法
238–239

## 配置文件数据
获取的技巧，124–125

## Profile_MigrateAnonymous 事件
121

## Profile_onMigrateAnonymous 脚本
17–18

## 配置文件提供程序
配置，15–18。另请参阅 ASP.NET 提供程序

#### 分部类
由 SubSonic 生成器生成，245–247

## 分区
206

按日期对数据库分区，299

## PropertyGroup 元素
MSBuild 脚本，257

## 密码策略
为 `SqlMembershipProvider` 创建，114–115

## 密码
正则表达式，114–115

## 性能与可伸缩性
理解，289–297

## 性能调整
152–153

## PerformSelect 方法
95–97

## 权限
授予使用通知系统的权限，174–175

## Person 和 Location 表
46

## Person 类
使用生成的类的对象浏览器，46

## 人员记录
插入新记录，59

## Person.abc 文件
237

## 页面
创建，229–231

## PageSize 属性
计算 `MaxIndex` 属性，98

## PageStatePersister 属性
87–88

## Page_Load 方法
151

#### 分页
通过分页减少 `ViewState` 大小，88

## 分页器事件
连接，98–101

## PagerButton_Click 事件处理程序
100

## p
pap_Albums.sql，相册表脚本，371

pap_DeleteAlbum.sql，373

pap_DeletePhoto.sql，373

pap_GetAlbumsByUserName.sql，374

pap_GetPhotosByAlbum.sql，374

pap_InsertAlbum.sql，375

pap_InsertPhoto.sql，375

pap_MoveAlbum.sql，376

pap_MovePhoto.sql，377

pap_Photos.sql，相册表脚本，371

pap_UpdateAlbum.sql，377

pap_UpdatePhoto.sql，378

## 页面缓存
154–155

涉及语言时的设置，186

## 页面大小和加载时间
87

## Providers 属性
在其中定义，315

## ProviderSections 属性
315–316

#### Q
### Query 类
由 SubSonic 查询工具使用，95

###### 查询通知
不允许使用的函数、运算符和表达式，175–176

对比 轮询，172–173

使用查询通知从缓存中移除项，172–176

要求，175–176

故障排除，176–186

#### 查询工具
247–248

#### R
### 重构
193

### 关系
管理，204–205, 310–313

### 发布分支
256

### 远程地址
插入，126–128

### 构建相册提供程序的要求
130

### 保存数据
211–213

#### 脚手架
248–249

### 可伸缩性
并发请求的影响，290

流量高峰的影响，291–293

规划，298

### 可伸缩性与性能。请参阅 性能与可伸缩性

### 脚本模板
使用，273

### 脚本
为脚本创建公共文件夹，4–5

使用嵌入式脚本，276–285

### ScriptsPrefix Constant 脚本
280–281

### 安全注意事项
内联 SQL 中的安全考虑，196–198

### Select 方法
调用，94–95

### SelectCountMethod 属性
68

### Selecting 事件处理程序
82

### Service Broker
启用，173–174

### 保存数据
211–213

### SaveViewState 方法
105

### SpeakerSection 类
314


# services
*参见* 分布式服务

## Remove Indexing.sql, 62–63
## Session
## Remove method, 160
作为缓存的替代方案，150
## Remove Provider Services.cmd script, 10
和 ViewState, 87–88
## Remove SQL Cache Dependencies.cmd, 171
购物篮设置，16–18
## RemoveConstraints.sql, 64–65
## shortcuts
## RemoveLinkTag 存储过程, 213–214
dacreation, 219
## Repeater 控件, 227–228
dagetdr, 220
## ReplaceConfigSections 任务，用于 PostBuild
dagetds, 219
部署, 264–265
danonquery, 220
强类型化数据集, 50
datypes, 219
## RoleManager.ascx.cs 脚本, 29–34
## SiteMap 提供程序。*参见* SQL SiteMap
## RoleProvider, 启用, 118
provider.cs 类
角色和用户，添加, 35–36
## Slashdot, 292
## Roles 提供程序，配置, 14–15。*另请参见*
为缓存数据设置滑动过期，
ASP.NET 提供程序
164–165
## Roles.Enabled 属性, 14–15
## sm_GetSiteMapNodes.sql, 394
## RolesManager.ascx 控件, 18
## sm_InsertSiteMapNode.sql, 142–143, 395
## 轮询 DNS, 293–294
## sm_RepopulateSiteMapNodes.sql
## RowFilter 属性，与 ProductID 一起使用，
加载站点地图, 140–142
187–188
存储过程脚本, 395
## rows count，获取总计, 97–98
## sm_SiteMapNodes 表，数据库
## Row_Number 函数, 67
要求, 140
## Run On 过程，内置静默功能, 57
## sm_SiteMapNodes.sql，表脚本, 393
## RunBuild.cmd 脚本, 270–271
## sm_UpdateSiteMapNode.sql, 397
运行 MSBuild 脚本, 260
## social bookmarking website，构建,
## RunDb.cmd 脚本，运行 db.proj 脚本
203–231
276
## 源代码控制系统，防止损坏代码
## RunDeploy.cmd 脚本, 268–269
进入, 71
## RunSqlCommands 方法, 283–284
## SpeakerSection 类, 314
## 运行时错误, 192–193
## Spring Framework, 245
## SQL 缓存依赖项, 165–186
■**S**
## SQL 注入攻击, 196–198
## 示例应用程序, 289–341
## SQL Photo Album 提供程序。*参见* Photo Album
## SaveControlState 方法, 89–90, 107–108
提供程序
## SaveFavoriteLink 方法, 220–221
## SQL 提供程序, 111–145
## SQL 查询, 196
## SQL Server，混合模式身份验证, 6
## Tools 目录, 242–243
##### SQL Server Management Studio
网站, 243
管理表创建和
## Substitution 控件, 155–156
修改, 57
## SubstitutionFragment 类, 156
移除索引, 62–63
## 开关，通过添加服务, 8
按名字和姓氏选择人员,
## 系统环境变量，添加到
57–58
文件夹, 5
## SQL Server Surface Area Configuration for
## 系统托盘应用程序，CruiseControl.NET,
Features 实用工具, 173
## SQL SiteMap 提供程序
## System.CodeDom 命名空间。*参见* CodeDom
构建, 139–145
命名空间
类, 385
## system.web 节，在 Machine.config 文件中,
存储过程脚本, 394
11–12
## SQL Web 事件提供程序。*参见* ASP.NET
提供程序
■**T**
## SqlCacheDependency
## /t 开关，Blinq 代码生成器, 250
为轮询配置, 171–172
## 表脚本，SQL 站点地图提供程序, 393
构造函数, 167
## 表，反范式化 vs. 使用连接, 187–188
## SqlCommand 对象, 167
## TagCloudControl, 228–229
更快地找到它 a
## SqlDependency 对象, 165–167
## TagCloudEventArgs, 229
## SqlMembershipProvider, 112–115
## tagCloud_OnTagSelected 事件处理程序, 231
## SqlPhotoAlbumProvider.cs, 345
## TagSelected 事件处理程序, 228–229
初始化方法, 134–135
## Target 元素，MSBuild 脚本, 257
## SqlProfileProvider, 118–125
## Template Explorer, 273
t
## SqlRoleProvider，用户分组,
## TemplateField，替换 BoundField,
115–118
## SqlSiteMapHelper.cs 类, 392
## 模板，为创建通用文件夹, 4–5
## SqlSiteMapProvider.cs 类, 385
## 模板化 vs. CodeDom 命名空间,
## SqlTableProfileProvider, 125
241–242
## StartDaysBack 属性, 226
## 测试，设计契约和数据契约, 203
## 静态文件，开发, 295
## 令牌清除列表，组装, 215
存储过程。*另请参见各个存储过程名称*
## 工具，为创建通用文件夹, 4–5
## TotalRowsCount 属性, 97–98
## 流量高峰，对可扩展性的影响, 291–293
## 琐碎数据示例，在 ASP.NET 中, 47–49
## TrivialExample.aspx 与 SqlDataSource,
47–48

# 文档大纲

*   **Pro** ASP.NET for SQL Server：面向 Web 开发人员的高性能数据访问
    *   目录
    *   开始使用
        *   环境准备
            *   项目组织
            *   通用文件夹
            *   数据源配置
            *   代码与数据库分离
        *   管理提供程序服务
            *   使用命令行
            *   混合与匹配提供程序
        *   配置提供程序
            *   成员资格配置
            *   角色配置
            *   配置文件配置
        *   创建用户和角色
            *   保护管理区域
            *   创建管理员用户
        *   小结
    *   数据模型选择
        *   数据访问应用程序块
        *   数据访问代码片段
        *   示例数据库
        *   简单数据示例
        *   复杂数据示例
        *   类型化 DataSet
        *   非类型化 DataSet
        *   DataReader
        *   DataObject
        *   有什么缺点？
        *   小结
    *   数据库管理
        *   使用数据库项目
            *   Visual Studio
            *   SQL Server Management Studio
        *   管理存储过程
        *   管理索引和约束
        *   性能考虑
        *   稳定性考虑
        *   数据的单元测试
        *   持续集成
        *   小结
    *   数据绑定控件
        *   DetailsView
        *   FormView
        *   GridView
        *   编辑和验证字段
        *   绑定输入参数
            *   使用控件绑定输入参数
            *   以编程方式绑定输入参数
            *   绑定用户控件
        *   嵌入数据绑定控件
            *   ViewState 与数据绑定
            *   Session 与 ViewState
            *   分页
            *   禁用 ViewState
            *   ControlState 与 ViewState
        *   创建数据绑定控件
            *   获取数据
            *   获取总行数
            *   连接分页器事件
            *   创建 PersonRow

### 索引

类型化 DataSet 设计器，生成 CRUD

相册提供程序，373–379
其中的方法，47
为“收藏链接”网站做规划，209

类型化 DataSets，192
清除以提高性能，214
构建自定义的，49–50
请求项目子集，66–67
僵化的，50
返回项目范围，66–67
保存数据，211–213

#### U

按名字和姓氏选择人员，58

单元测试
`sm_InsertSiteMapNode.sql`，142–143
针对数据，70–71
`StringBuilder`，153
`DomainTests.cs`，179–180
子域，将服务拆分为，295–296
.NET 最流行的框架，70

SubSonic 代码生成器，242–243
与 MSBuild 配合使用，186
为表属性调整的模板，244–245
`PhotoAlbumProvider`，137–138
AutoScaffold 页面，248–249
内置查询工具，247–248
类模板，243–244
命令行工具，243
运行，243
模板系统，243–245

8601Ch12IndexCMP2 8/27/07 7:09 PM Page 408

**408**

索引

单元测试 (续)
实现，332–337
`Test105_Caching_SqlDependency_Test 方法`，184–185
服务配置，333–334, 337
使用查询通知进行故障排除，262–264

`UnitRun`，与 NUnit 测试一起，179

`UpdateDatabase` 方法，281–282

`Url` 属性，在 `FavoriteLink` 对象中设置，202

#### V

VaryByParam 属性，通配符使用的影响，154

版本号，更改，202

`ViewState`
作为缓存的替代方案，150
和数据绑定，87
与 `ControlState` 比较，89–91
在没有的情况下进行调试，109
禁用，89
手动持久化，105–106
`Session` 和，87–88
在启用和不启用的情况下工作，89, 107–108

Web 开发项目，261

虚拟环境，1–4

虚拟硬盘，创建，1–2

`VirtualPath` 属性，235–236

`Visible` 属性，为 `AdminHyperlink` 控件设置，116

Visual Studio (VS) 2005，向其中添加代码片段，45

Visual Studio 专业版，其中的数据库项目，56–57

#### W

WCF 提供程序
优点，332–333
客户端配置，333, 336
配置，336–337
承载服务，335–336
`IEventService` 接口，334
实现，332–337

`WipeProviderData.sql` 脚本，9–10

Web 部署项目
自动化配置更改，262–264
`configSource` 属性选项，264
创建，261–262
多个替换节，265–266
`PostBuild` 部署，264–271
网站，261

用户控件
绑定，83–84
配置，230
声明，229
标记，159
`SubstitutionFragment`，157–158

`UserManager.ascx`
控件，18–23
脚本，23–29

用户名和密码，存储在 `Web.config` 文件中，118

用户和角色，创建，18–34

工具类
`GetProduct` 方法，158–159
在 `App_Code` 文件夹中，17

实用工具方法脚本，17

#### X

XML 实现，使用，113

`xcopy` 命令，261

XPath 常量，239

`XsdBuildProvider`，`.xsd` 映射到，234

#### Z

zip 文件，在 `PropertyGroup` 中定义名称，271

`Zip` 任务，声明，271

网站
代码与数据库分离，7
CodePlex 信息，242
按角色控制访问，115–117
按角色控制行为，117–118
将文件从源目录复制到目标目录，261
为用户定制内容，119
部署，261–271
将内容和流量分配到多个服务器，293–295

Web 开发项目，261

`Website` 类，相册提供程序，379

网站项目，创建类库，218

Windows Communication Foundation 提供程序。 *参见* WCF 提供程序

Windows Live，125

Web 服务扩展 (WSE)，由 Microsoft 发布，332

`Web.config` 文件，234
自定义的，13
其中的数据源配置，6–7
用于 `PhotoAlbumProvider`，138
在其中保护管理区域，34–35
存储用户名和密码，118

Web 窗体，向其中添加 `ObjectDataSource`，52

网页
性能考虑，65–70
保护安全，116

Web 服务器，到数据库服务器的通信，148


##### 手动持久化视图状态

### 不使用视图状态工作

### 使用调试器

### 小结

## SQL 提供程序

### SqlMembershipProvider

##### 使用 XML 实现

##### 设置数据库连接

##### 创建密码策略

### SqlRoleProvider

##### 按角色控制访问

##### 按角色控制行为

### SqlProfileProvider

#### 为什么需要匿名配置文件？

##### 配置匿名配置文件

##### 管理匿名配置文件

##### 匿名与已验证配置文件的差异

##### 从匿名迁移到已验证

##### 创建用户

#### 动态配置文件和作为 BLOB 的配置文件

#### 使用提供程序驱动的 ASP.NET 控件

#### 构建 SQL 相册提供程序

##### 提供程序要求

##### 配置节类

##### 提供程序集合类

##### 抽象提供程序类

##### 提供程序实现

##### 提供程序服务类

##### 单元测试

#### 最终成果

### 构建 SQL SiteMap 提供程序

#### SiteMap 要求

#### 实现 SiteMapProvider

### 小结

#### 缓存

### 缓存的替代方案

###### 应用程序状态

###### 会话

###### 视图状态

###### 当前上下文

#### 缓存选项

##### 输出缓存

##### 页面缓存

##### 输出缓存的问题

##### 片段缓存

##### 缓存后替换

##### 使用缓存后替换的片段缓存

##### 数据缓存

##### 缓存方法

##### 缓存索引

##### 枚举缓存内容

##### 参数

##### 缓存模拟

##### 使缓存数据失效

###### 绝对过期

###### 滑动过期

#### 缓存依赖项

###### 手动移除

## SQL 缓存依赖项

###### 使用 SqlDependency 和 SqlCacheDependency

#### 为什么使用 SqlDependency？

#### 使用 SqlCacheDependency

###### 轮询

#### 为表启用轮询

#### 配置用于轮询的 SqlCacheDependency

#### 轮询的问题

###### 查询通知

#### 对比轮询与通知

#### 启用 Service Broker

#### 授予权限

#### 查询的要求

### 查询通知的故障排除

#### 使用 SQL Server Profiler 进行故障排除

#### 通过单元测试进行故障排除

##### 缓存存在的问题

##### 性能策略

###### 数据仓库

###### 延迟加载

### 小结

## 手动数据访问层

#### 使用数据集、内联 SQL 和存储过程

##### 数据集

##### 编译时和运行时支持

##### 重构

##### 调试

##### 内联 SQL

##### 维护考虑因素

##### 安全考虑因素

##### 存储过程

### 使用数据对象和 ObjectDataSource

#### 设计契约

#### 数据契约

#### 测试设计契约和数据契约

#### 构建数据库

##### 创建数据库结构

#### 整合数据

#### 管理关系

#### 创建与修改字段

#### 如何处理空值？

##### 防止空值

#### 使用数据库项目

##### 管理脚本

##### 规划存储过程

#### 数据访问层

##### 创建类库

##### 创建数据访问方法

##### 处理异常

##### 创建业务对象

#### 构建网站

##### 连接数据访问层

##### 创建用户控件

##### 属性

##### 事件

##### 创建页面



# 生成的数据访问层
#### 代码生成
#### 构建提供程序
#### CodeDom 命名空间
#### 模板化
#### SubSonic
##### SubSonic 模板化
### 部分类
#### 查询工具
#### 脚手架
#### Blinq
#### 总结
# 部署
## 使用 MSBuild 进行自动化
#### 部署网站
##### 网站部署项目
##### 自动化配置更改
### PostBuild 部署
#### 部署数据库
### 生成更改脚本
##### 自动化数据库更新
#### 使用 MSBuild 和 ExecuteDDL
#### 使用嵌入式脚本
##### 自定义配置节
#### 总结
#### 示例应用程序
#### 理解性能与可伸缩性
##### 并发请求
##### 瓶颈
### 流量高峰
### 分发流量
### 分发内容
### 分发服务
### 分发后端
## 为可伸缩性做规划
#### 示例应用程序
##### 创建数据库
### 获取、保存和删除
#### 创建数据访问提供程序
##### EventProvider 对象
##### 修订后的 DomainObject
#### 管理关系
### 使用 Locations
#### 自定义配置
##### 配置分组
##### 声明自定义配置
###### 配置提供程序
#### 创建新的提供程序
##### 实现 LINQ 提供程序
##### 实现 WCF 提供程序
###### WCF 服务要求
###### 托管服务
###### 定义数据契约
###### 配置提供程序
#### 使用提供程序
#### 总结
# 相册
## 相册提供程序
###### 配置
##### 类
##### 表脚本
###### 约束脚本
##### 存储过程脚本
###### 网站类
## SQL SiteMap 提供程序
##### 类
##### 表脚本
##### 存储过程脚本
### 索引
