# 第 5 章 ■ SQL 提供程序

## 使用由提供程序驱动的 ASP.NET 控件

现在，你已经为网站配置了用户登录所需的所有成员功能。`Login`控件可以轻松处理这一点。这个过程充满了安全性，建立在多年的遗留技术之上。尽管`Membership` API 和提供程序模型对 ASP.NET 来说是新的，但验证用户身份和维护用户会话的机制从 ASP.NET 诞生之初就已存在。当用户成功登录你的网站时，她会获得一个令牌，该令牌作为 Cookie 存储在她的 Web 浏览器中。这个令牌就像一把钥匙，让她能够访问网站的受保护区域和定制区域。

身份验证令牌是一个加密的`FormsAuthenticationTicket`，它保存着用户名、会话过期时间以及其他数据，包括自定义用户数据。

在下面的场景中，我将展示如何利用这些自定义用户数据来提高已验证会话的安全性，并演示 ASP.NET 中身份验证系统的灵活性。

在这个示例场景中，你正在使用一个银行网站，并在工作时查看你的余额。你被短暂叫走，但没想到锁定电脑。一位不诚实的同事偷偷使用你的电脑，复制了用于验证你与银行会话的 Cookie。这个 Cookie 至少不是包含你账号的明文值（否则任何时间都能访问你的账户），而是一个加密的`FormsAuthenticationTicket`形式，其中包含过期时间和令牌颁发日期。这意味着这位不诚实的同事只有有限的时间可以使用该 Cookie。

对于采用滑动过期的会话，这位同事所能做的是将 Cookie 放入浏览器，这将不断更新过期时间。但它不会改变的是保存在服务器上的最后登录日期。如果你的银行网站非常安全，该网站可能会在允许你执行任何真正重要的操作（例如安排付款或转账到他人账户）之前检查你的最后登录时间。在网站允许你进行这些操作之前，它可以检查你是否在交易发生的几分钟内登录过，如果你没有，则要求你提供密码。即使你已经通过验证，你也可以重新登录，这样做会重置最后登录时间。这个要求将阻止你的同事完成此类操作，因为他只有 Cookie，只能对你的账户进行基本访问。但要让该同事获得执行交易的访问权限，只需等你回到电脑前并再次登录银行网站即可。如果在令牌中存储你的 IP 地址，以便网站可以验证你最近是从那台计算机登录的，这将会很有帮助。这正是你要使用自定义用户数据的地方。

登录过程可以调整，以便将远程地址作为用户数据插入到`FormsAuthenticationTicket`中。只需将`Login`控件拖放到 Web 窗体的设计界面。然后为`Authenticate`事件创建一个处理程序。清单 5-15 中的代码处理该事件，并将用户的远程地址插入到票据中，该地址可用于在用户每次返回网站时验证其位置。



### 第 5 章：SQL 提供程序

## 清单 5-15：登录认证事件处理器

```csharp
protected void Login1_Authenticate(object sender, System.Web.UI.WebControls.AuthenticateEventArgs e)
{
    if (Membership.ValidateUser(Login1.UserName, Login1.Password))
    {
        e.Authenticated = true;
        string username = Login1.UserName;
        bool persist = Login1.RememberMeSet;
        int timeout = GetLoginTimeout();
        string userData = "remoteAddress=" + Request.UserHostAddress;
        FormsAuthenticationTicket ticket = new FormsAuthenticationTicket(1, username, DateTime.Now, DateTime.Now.AddMinutes(timeout), persist, userData);
        string encTicket = FormsAuthentication.Encrypt(ticket);
        HttpCookie cookie = new HttpCookie(FormsAuthentication.FormsCookieName, encTicket);
        cookie.Expires = ticket.Expiration;
        Response.Cookies.Add(cookie);
        Response.Redirect(FormsAuthentication.GetRedirectUrl(username, persist), true);
    }
    else
    {
        e.Authenticated = false;
    }
}
```

为了获取为用户会话配置的超时时间，我创建了一个方法，用于从 `Web.config` 中读取配置的值，如清单 5-16 所示。

## 清单 5-16：GetLoginTimeout 方法

```csharp
private int GetLoginTimeout()
{
    AuthenticationSection authSection = ConfigurationManager.GetSection("system.web/authentication") as AuthenticationSection;
    if (authSection != null && authSection.Forms != null)
    {
        return authSection.Forms.Timeout.Minutes;
    }
    // 返回默认值
    return 30;
}
```

现在票据中有了这个值，你可以在用户即将执行比典型页面请求更关键的操作（例如进行银行转账或更改密码）时进行检查。清单 5-17 中的方法会检查远程地址。

## 清单 5-17：IsRemoteAddressMatched 方法

```csharp
private bool IsRemoteAddressMatched()
{
    if (User.Identity.IsAuthenticated)
    {
        FormsIdentity identity = User.Identity as FormsIdentity;
        if (identity != null)
        {
            string prefix = "remoteAddress=";
            if (!String.IsNullOrEmpty(identity.Ticket.UserData) && identity.Ticket.UserData.IndexOf(prefix) == 0)
            {
                string remoteAddress = identity.Ticket.UserData.Substring(prefix.Length);
                return Request.UserHostAddress.Equals(remoteAddress);
            }
        }
    }
    return false;
}
```

清单 5-17 中的方法从用户身份中提取票据，然后获取远程地址的值。最后，它将存储的值与页面请求中的实际值进行比较。这个额外的检查有助于增强安全性。

然而，你可以更进一步。也许你想让用户选择将其当前位置保存为可信位置，以便当他来自该位置时，安全性可以放宽。例如，如果他出差并从酒店房间登录，登录过程可能会要求他提供不仅仅是用户名和密码。它可能会要求提供社会安全号码的后四位数字和家庭邮政编码。通过授权可信位置，你将能够记录和分析从新添加位置发生的交易。这种方法得到了增强，因为家庭宽带连接倾向于在较长时间内保持相同的 IP 地址，因此这个远程地址不会经常改变。

### 通过 IP 地址确定物理位置

存在一些称为 `GeoIP` 的服务，它们可以告诉你 IP 地址的物理位置。允许用户从可信的物理位置（而不仅仅是一个 IP 地址）在其账户上执行某些操作，可以增强安全性，同时也使用户更方便。如果你的用户在芝加哥生活和工作，当你知道她是从该位置登录时，可以授予她更高信任级别。然而，如果她下午 1:00 在芝加哥登录，而有人在下午 1:40 从洛杉矶登录，你可以假设发生了意外情况，将账户标记为待审查，并提醒用户注意这一不明活动。

`GeoIP` 试图为每个 IP 地址确定一个物理位置，但它并不总是准确的——就像你不能总是在电话簿中查到某人的电话号码一样。你也可以使用按国家分配的 IP 地址块。例如，韩国的 IP 地址范围从 `210.220.128.0` 到 `210.223.255.255`。即使你无法确定用户是否在芝加哥，你也可以确定该用户是从韩国、中国、俄罗斯还是其他国家访问网站。

### 在 Web 场中使用共享的机器密钥

当将多个 Web 服务器放入一个场中以分担高流量网站的负载时，有必要使用机器密钥设置，该设置通常对服务器是唯一的。此配置设置包括验证密钥和解密密钥，用于哈希处理和解密。表单身份验证令牌使用这些值，如果机器密钥在服务器之间不一致，当用户会话在服务器之间跳跃时，你将遇到身份验证问题。使用粘性会话可以部分避免此问题，但在用户离开较长时间后，他可能会失去与负载均衡器的粘性会话，并访问具有不同机器密钥值（这些值将不兼容）的不同服务器。

其他数据也受机器密钥保护，例如 `ViewState`，它被加密以防止篡改。如果用户使页面空闲很长时间，然后导致 `PostBack`，该用户可能会因为这种跨服务器不兼容性而遇到异常。

机器密钥在 `Web.config` 文件中配置，并且必须生成。尽管 Microsoft 不提供生成这些值的实用工具，但 MSDN 文档解释了如何操作。可以在诸如代码项目网站（The Code Project）之类的网站上找到多个执行此操作的实用工具，该网站有一个实用工具，恰当地命名为 ASP.NET `machineKey 生成器`。

#### 构建 SQL 相册提供程序

除了标准提供程序之外，你还可以创建自己的自定义提供程序以满足特定需求。在构建自己的提供程序时，你可以让它与标准提供程序的接口交互，以实现无缝集成。创建自己的自定义提供程序允许多种实现。对于内部应用程序，可能不需要不止一种实现，至少最初是这样。

你可以构建一个由多个应用程序使用的组件。如果你使用提供程序模型来构建它，你就允许多种实现。此组件可以与自定义订单履行系统交互。后来，你的公司可能会购买一个处理相同功能的订单履行系统套件。你不必将应用程序与这个新套件紧密耦合，而是可以为其创建一个提供程序实现。将应用程序迁移到它只需调整配置即可。

相同的场景可以应用于组件的内部版本。自定义提供程序的第一次实现可能与一组复杂的底层依赖项交互。也许你有一座难以维护的遗留代码山，但你和你的团队正在清理它以提高可维护性。之后，随着代码库的重构，你可以创建一个使用更新系统的新实现。构建在提供程序之上的应用程序不必更改。

对程序集进行版本控制也可以实现类似的解决方案。提供程序的不同之处在于，你不是直接针对实现或版本编译应用程序。相反，你将提供程序用作一个抽象层，它将使用配置来实例化所需的程序集。这样做还可以避免将一个程序集直接绑定到另一个程序集的特定版本。


### 第 5 章：SQL 提供程序

##### 提供程序要求

一个提供程序由三个核心部分组成：抽象提供程序、至少一个提供程序实现，以及用于加载该实现的配置。对于一个从头开始构建的自定义提供程序，你需要额外的代码来加载配置并让你能够访问提供程序实现。如果你是要实现某个现有提供程序（例如 `MembershipProvider`）的自定义实现，那么你可以使用已有的加载机制，只需创建你的提供程序实现类即可。

开始使用一个新提供程序时，首先要实现的两个类是配置类和提供程序集合类。为了便于加载提供程序，有一整套基础设施来支持这个过程。使用以下类可以加速创建提供程序的过程。

##### 配置节类

如果你查看 `Machine.config` 文件中对成员资格的引用，你会看到一个引用 `MembershipSection` 类型的节定义，`MemerbershipProvider` 使用这个类来读取其配置。这个类继承自 `ConfigurationSection` 类，你将使用它来创建你自己的 `PhotoAlbumSection` 类，从代码清单 5-18 开始。

**代码清单 5-18.** *PhotoAlbumSection.cs*

```csharp
using System.Configuration;

namespace Chapter04.PhotoAlbumProvider
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

`Providers` 属性加载 providers 子元素，而 `DefaultProvider` 属性加载 `defaultProperty` 属性的值。此时实际上不需要做太多工作，因为提供程序基础设施处理了很大一部分工作。

仅为 `ProviderSettingsCollection` 类型创建一个属性，就足以使这个 `AlbumSection` 类像标准提供程序那样，充当一个提供程序配置节。标准提供程序在配置的 `providers` 元素内包含 `add`、`remove` 和 `clear` 指令。`Providers` 属性会自动处理这些配置元素。

##### 提供程序集合类

因为你可以配置一个提供程序的多个实例，所以你会希望有一个集合来容纳它们。为此，你将创建一个名为 `PhotoAlbumProviderCollection` 的类，它将继承 `ProviderCollection` 并重写其行为以使用 `PhotoAlbumProvider` 类型，而不是泛型的 `Provider` 类型。该类如代码清单 5-19 所示。

**代码清单 5-19.** *PhotoAlbumProviderCollection.cs*

```csharp
using System;
using System.Configuration.Provider;

namespace Chapter04.PhotoAlbumProvider
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
                throw new ArgumentException("Invalid provider type", "provider");
            base.Add(provider);
        }
    }
}
```

有了你自己定制的提供程序基础设施，你就准备好开始定义 `PhotoAlbumProvider` 了。

##### 抽象提供程序类


