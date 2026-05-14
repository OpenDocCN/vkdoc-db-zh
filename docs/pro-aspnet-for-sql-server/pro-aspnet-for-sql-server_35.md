# 第六章：缓存

当开发环境中本地数据库与 Visual Studio 启动的开发 Web 服务器位于同一台机器上时，Web 服务器与数据库服务器之间的通信很容易被忽视。这可能会给你一个错误的印象，即部署后数据访问层同样会如此迅速。

调整查询，只从表中提取必要的列而非所有列，就是一种轻松减少 Web 服务器与数据库服务器之间带宽消耗的示例。

数据库上的查询量突然激增，即使是相同的查询，也可能导致数据库性能急剧下降。这可能是因为这些查询涉及跨多个表的连接，而这些表恰好位于物理磁盘的不同部分。SQL Server 2005 会尝试通过将最近使用的数据和索引保留在内存中来进行补偿，但这些理想的自动优化并不保证会发生。最好将这些查询的结果保存在 Web 服务器上，以便数据库能够处理其他查询。

## 缓存的替代方案

使用 ASP.NET 缓存并不总是提升性能的正确解决方案。放入缓存的数据越多，它可能变得越复杂。并且缓存中的更多数据可能会将你希望保留其中以增强性能的项目挤出去。一些数据可以通过其他开销更低的方式存储，例如使用应用程序状态、`Session`，甚至 `ViewState`。

虽然下文描述的技术有时是合适的解决方案，但它们也存在一些问题。要么它们在内存不足时没有自动释放对象的方式，要么它们保留数据的时间不够长，从而无法发挥作用。即便如此，你可以将这些技术相互结合并与缓存结合使用，以提供一个更全面的解决方案，让你在获得所有好处的同时将负面影响降至最低。

###### 应用程序状态

应用程序状态是一个全局资源，每个用户和请求都可以访问。你可以用它来保存需要快速访问的对象。你可以使用 `Global.asax` 中的 `Application_Start` 事件将项目放入此集合，以便项目在应用程序首次启动时立即可用。列表 6-1 展示了如何使用网站根目录下的 `Global.asax` 文件将数据存储到应用程序状态中。其余代码可以放在 `App_Code` 文件夹的工具类中。列表 6-2 展示了 `GetApplicationState` 方法，该方法由 `Global.asax` 中的方法调用。

**列表 6-1.** 在 `Global.asax` 中将数据加载到应用程序状态

```csharp
void Application_Start(object sender, EventArgs e)
{
    HttpContext.Current.Application["ApplicationState"] = GetApplicationState();
}

void Application_End(object sender, EventArgs e)
{
    HttpContext.Current.Application["ApplicationState"] = null;
}
```

**列表 6-2.** `App_Code\Utility.cs` 中的 `GetApplicationState` 方法

```csharp
public static DataSet GetApplicationState()
{
    return HttpContext.Current.Application["Global Data"] as DataSet;
}
```

无需任何额外工作，放入应用程序状态的数据将不会被更新。如果外部事件需要更新这些数据，可以使用处理程序重新加载数据。列表 6-3 展示了一个可以调用来重新加载应用程序状态的处理程序（`.ashx`）。

**列表 6-3.** `ApplicationStateHandler.ashx`

```csharp
<%@ WebHandler Language="C#" Class="ApplicationStateHandler" %>
using System.Web;
using Chapter06.ClassLibrary;

public class ApplicationStateHandler : IHttpHandler {
    public void ProcessRequest (HttpContext context) {
        context.Response.ContentType = "text/plain";
        string action = context.Request.QueryString["action"];
        if ("ReloadApplicationState".Equals(action))
        {
            context.Response.Write("Reloading Application State\n");
            Domain.LoadApplicationState();
            context.Response.Write("Done.");
        }
        else
        {
            context.Response.Write("Action unknown: " + action);
        }
    }

    public bool IsReusable {
        get {
            return false;
        }
    }
}
```

应用程序状态可以像任何集合一样保存多个项目，因此可以存储各种各样的数据，以供任何请求需要时使用。

## Session

`Session` 为单个用户保存数据，其时间窗口为滑动的 20 分钟。在 `Global.asax` 中，你可以使用 `Session_Start` 和 `Session_End` 事件处理程序来管理 `Session` 的生命周期，这是另一个可以保存任何类型对象的集合。随着用户使用网站，你可以随时间向其添加数据。加载 `Session` 的一个好时机是在用户登录网站后立即进行。它可以保存与该用户相关的所有数据，你预计用户在访问期间最终会从数据库请求这些数据。

## ViewState

`ViewState` 进一步将其保存的数据范围限制为仅当前页面，只要页面维持着回发周期。当数据绑定到 `GridView` 时，由于它使用 `ViewState` 来绘制 `GridView`，因此在回发时不必访问数据库。你还可以将额外的项目放入 `ViewState` 中，这些项目不附加到页面元素，但你可以在回发时访问它们。不过，这仅限于可以序列化以用于 `ViewState` 的对象。

###### 当前上下文

你还可以在当前上下文中保存数据，其范围仅限于当前请求。你可以存储数据，然后在整个请求过程中由任何控件共享。考虑一下，如果你代码隐藏中的 `Page_Load` 在上下文中填充了对象，那么该页面中包含的所有用户控件随后都将能够使用它，而不必去数据库获取数据。对于产品详情页，你可能会将页面的部分内容拆分为用户控件以简化维护，但如果一个页面包含五个用户控件，你可能会为了一个请求分别访问数据库五次来获取产品详情。使用当前上下文将显著减少数据库负载。列表 6-4 展示了如何将数据加载到当前上下文中。列表 6-5 展示了页面上的每个用户控件或任何控件如何访问放置在当前上下文中的数据。

**列表 6-4.** 在 `Page_Load` 方法中将产品加载到当前上下文

```csharp
using System;
using System.Web.UI;

public partial class _Default : Page
{
    protected void Page_Load(object sender, EventArgs e)
    {
        Context.Items["CurrentProduct"] = GetProduct();
    }

    private Product GetProduct()
    {
        Product product = new Product("Product 1");
        return product;
    }
}
```

**列表 6-5.** 在用户控件中使用当前上下文访问产品

```csharp
using System;
using System.Web.UI;

public partial class Controls_ProductControl : UserControl
{
    protected void Page_Load(object sender, EventArgs e)
    {
        Product currentProduct = Context.Items["CurrentProduct"] as Product;
        if (currentProduct != null)
        {
            Label1.Text = currentProduct.Name;
        }
        else
        {
            // 当没有内容可显示时隐藏该控件
            Visible = false;
        }
    }
}
```

## 你最糟糕的敌人

我曾经与一位开发人员共事，他在一次页面请求中无意间访问了数据库数百次。这些查询导致页面加载时间非常长。他没有意识到，在代码循环中调用某些方法会不断重新加载他之前已经请求过的数据。只需将同一请求早期的数据保存下来，供同一请求的后续部分使用，你就可以在不使用 ASP.NET 缓存系统的情况下加快应用程序的速度。



对单个页面请求进行数百次数据库调用的一个副作用是，每个查询的响应速度会异常缓慢，这种延迟会迅速累积。这就像高峰时段的交通堵塞，但对于数据库而言，这一切都发生在几秒钟内，当你期望页面在一秒内加载完成时，这种延迟就非常明显。使用你最近从数据库请求的数据可以分散你的查询，减少查询“交通堵塞”的频率。

当你加载数据并在后续使用时，事件顺序非常重要。当与母版页一起使用时，你可能期望母版页上的 `Load` 事件首先运行，但实际上，页面上的 `Load` 事件会先触发，然后是母版页的，接着是页面上所有用户控件的 `Load` 事件。我曾使用此技术结合自定义的 `SiteMap`，通过 `Url` 与这些对象的关联，将页面的当前对象加载到当前上下文中。

这个 `Url` 可能不对应一个产品，但当它确实对应一个产品时，我可以填充该项，所有用户控件都可以检查并使用它（如果可用）。为了集中所有这些逻辑，你可以将填充这些上下文项的代码放在母版页的 `Init` 事件处理程序中，这样它就在页面的 `Load` 事件之前执行了。

## 你应该跳过性能调优吗？

你不应该过早地优化系统，因为在早期你并不知道什么需要调优。重要的是先在调优前进行测量，调优后再次测量，以便展示差异。

经过几次迭代，你会开始在代码中发现显现的模式，这些模式会向你展示常见的性能问题，这些问题希望有一个通用的解决方案，你可以在整个代码中一致地应用。在应用程序完成之前尝试进行调优，可能会引入问题，从而延迟项目的完成，却无法增加可衡量的价值。

.NET 内部还有一些特性可以解决常见的低效编码技术，它们会自动启用以加速你的应用程序。通过使用你通常的编码技术，你会发现那些看起来低效的做法实际上性能相当好。但如果你试图预先调优应用程序，你就无法获得那种自动优化的好处，并且会为项目投入更多的时间和精力。

## 性能调优：字符串拼接

关于调优的一个常见走廊辩论是关于字符串拼接。你应该使用 `+` 操作符将字符串相加，还是使用 `StringBuilder` 来追加字符串？冲突在于，直接用 `+` 操作符相加字符串比使用 `StringBuilder` 更简单、更快，但 `StringBuilder` 的性能更好。能迅速结束这场辩论的是有人指出，一旦代码被编译，无论哪种方式，最终都会被优化为使用 `StringBuilder`。而这仅仅是 .NET 内置的众多性能增强机制之一，你无需改变当前的编码习惯即可享用。一个好的经验法则是记住，编程代码是给人看的，机器码是给机器看的。让编译器做它的工作吧。

当你完成构成应用程序的各个组件时，你将能够分析实际性能并识别瓶颈。有时性能调优可能只是对存储过程的一个简单的一行调整，或是对数据访问层代码的几行修改。使用常规技术构建应用程序，可以让你尽可能高效，同时让低效之处在接近完成时逐渐显现。慢慢地，随着你越来越得心应手，你会开始形成一些模式，从一开始就构建出性能更佳的应用程序。

#### 缓存选项

说到缓存，有令人眼花缭乱的选项可供选择。`Cache` 对象不仅仅是持有一个对象集合以供快速访问。整个缓存系统包含一套丰富的功能，可用于不同的场景。ASP.NET 中的缓存主要分为两大类：输出缓存和数据缓存。

##### 输出缓存

使用输出缓存，你是在用户界面层工作。你缓存的是页面或用户控件的输出。（缓存用户控件的输出被称为 `fragment` 缓存。可以将用户控件视为生成页面的一部分，或“一个片段”。）配置为输出缓存的页面或用户控件将在第一次请求时正常运行，之后每次后续请求在缓存过期前，都将返回与首次将内容放入缓存的请求相同的输出。从缓存中提取内容可以避免代码隐藏中的服务器端处理，包括所有数据绑定和数据库查询。

虽然输出缓存有一些主要好处，但它有点像是一种“蛮力”方法，而不是一个针对性的解决方案，后者可以考虑被消费数据的性质。一个页面或用户控件可以缓存许多数据片段，这些片段都有独特的依赖关系，当这些数据被分组显示时，会引发许多问题。通过正确组合此处介绍的技术，应该可以最大限度地减少其缺点。

### 页面缓存

在最高层级，你将缓存整个页面的输出，包括页面内包含的所有控件。启用此类缓存的声明方式是在页面中放置一个包含所需设置的指令。

清单 6-6 中的示例将缓存页面的输出，`Duration` 为 60 秒，并且对于任何参数都是唯一的。当使用通配符 (`*`) 时，`VaryByParam` 属性将导致输出缓存在任何查询字符串参数不同时生成新的输出。你也可以使用分号分隔的名称列表来指定特定的一个或多个查询字符串。

**清单 6-6.** `启用输出缓存`

```aspx
<%@ OutputCache Duration="60" VaryByParam="*" %>
```

输出缓存的一个吸引人的特点是，它不仅在返回缓存响应时不必从数据库拉取数据，而且也不必触发任何页面事件，因为它使用的是首次请求（将输出记录到缓存）的输出。因此，它不仅减少了数据库负载，还降低了处理器使用率。

输出缓存也可以在代码隐藏中以编程方式设置，如清单 6-7 所示，尽管这种方法不如声明式方法优雅。它确实让你可以选择使用逻辑来设置过期时间，如果你想根据服务器负载调整缓存，这可能会派上用场。

**清单 6-7.** `以编程方式设置页面进行输出缓存`

```csharp
protected void Page_Load(object sender, EventArgs e)
{
    Response.Cache.SetCacheability(HttpCacheability.Public);
    if (IsPeakLoadTime())
    {
        // 在高峰负载期间缓存十分钟
        Response.Cache.SetExpires(DateTime.Now.AddMinutes(10));
    }
    else
    {
        // 在正常流量期间缓存两分钟
        Response.Cache.SetExpires(DateTime.Now.AddMinutes(2));
    }
    Response.Cache.SetValidUntilExpires(true);
    Product currentProduct = Context.Items["CurrentProduct"] as Product;
    if (currentProduct != null)
    {
        Label1.Text = currentProduct.Name;
    }
    else
    {
        // 当没有内容可显示时隐藏控件
        Visible = false;
    }
}
```

### 输出缓存的问题



一个采用预编译程序集部署的网站，从请求处理的角度来看，已经实现了大量的性能优化。应用程序中较慢的活动通常仍然是从数据库提取数据。通过使用高效的缓存依赖项，您可以确保缓存中的数据始终是最新的，但输出缓存声明并不具备了解缓存依赖项何时变更所需的全部丰富性和细节。如果您为了确保显示的数据是最新的而对数据访问层所做的所有优化努力，可能会因为输出缓存将缓存输出保留过久而付诸东流。因此，您可能会发现输出缓存通常不是一个现实的选择。

## 片段缓存

您不必缓存整个页面，而是可以缓存各个部分，其中一些部分需要比其他部分更频繁地更新。在用户控件中使用输出缓存被称为**片段缓存**。放置在页面上的每个用户控件都可以有不同的输出缓存设置。考虑一个产品详情页面，它展示产品照片以及关于产品的各种细节。我曾使用片段缓存，通过五六个用户控件来显示产品页面的几个部分。其中一些用户控件被缓存，而另一些则没有。某些细节不会经常改变，例如产品照片、描述、尺寸、颜色和供应商关联。但其他细节，如定价和可用状态，可能会更频繁地变化。每个用户控件都根据其显示内容声明了特定的输出缓存设置。产品页面是这些用户控件的组合体，在流量高峰期它的性能表现非常出色。

您可以结合使用页面缓存和片段缓存，但显然，页面上设置的 `Duration` 必须比启用了输出缓存的用户控件的持续时间短。您可以将页面的持续时间设置为 10 秒，而一个包含很少更改数据的用户控件的缓存时间可以设置为 3,600 秒。

## 缓存后替换

另一个输出缓存功能是**缓存后替换**，它允许页面使用输出缓存，同时允许页面的某一部分在每次请求时更新。这可以通过 `Substitution` 控件以声明方式实现。该技术有一个限制：`Substitution` 控件不能在启用了输出缓存的用户控件或母版页中使用，因此您无法将此技术与片段缓存结合使用，至少不能直接结合。您必须改为缓存整个页面，并在需要每次请求都获取新内容的位置使用 `Substitution` 控件。

`Substitution` 控件的工作方式与典型的 ASP.NET 控件不同。像 `Label` 这样的典型控件允许您设置 `Text` 属性，控件会连同其他属性（如 `Font-Bold` 和 `CssClass`）一起用于显示。而对于 `Substitution` 控件，您需要提供一个像 `Literal` 控件那样直接显示的字符串。这是通过 `MethodName` 属性实现的，该属性引用代码隐藏文件中的一个静态方法，该方法以 `HttpContext` 对象作为唯一参数并返回字符串。由于控件上的其他属性，此字符串不经处理或装饰直接显示。

第 6 章 ■ 缓存

在产品详情页的情况下，您会将整个页面设置为使用输出缓存，而用于组装页面的每个用户控件则不使用缓存。对于需要更新的某些细节，您会使用一个 `Substitution` 控件。

在清单 6-8 和 6-9 中，`Label` 控件只会在页面的输出缓存过期时更新，而 `Substitution` 控件的值在每次请求时都是最新的。

### 清单 6-8. Substitution 控件标记

```xml
<asp:Label ID="lblProductName" runat="server" Text=""></asp:Label><br />
<asp:Label ID="lblProductNumber" runat="server" Text=""></asp:Label><br />
<asp:Substitution ID="subPrice" runat="server" MethodName="GetPrice" /><br />
```



## 带有 Postcache 替换的片段缓存

<asp:Substitution ID="subAvailability" runat="server" MethodName="GetAvailability" />

**清单 6-9.** `Substitution 控件后台代码`
```csharp
private static string GetPrice(HttpContext context)
{
    Product product = GetProduct(context);
    return product.Price.ToString("C");
}

private static string GetAvailability(HttpContext context)
{
    Product product = GetProduct(context);
    return product.Availability;
}
```

如果你只是要返回一小段文本，这就足够了。但如果你想返回更多内容，就必须通过某种方式从该方法生成所有要返回的内容。创建一个充满标记的字符串正是你在 ASP.NET 开发中试图避免的，因为有那么多便捷的可视化工具。`Substitution`控件的工作方式并未提供让你直接停留在可视化模式下的直接方法。但这可以间接实现。

在页面设置为输出缓存的情况下，你可以将一个用户控件设置为容纳一个`Substitution`控件，然后该控件再加载另一个用户控件，该用户控件容纳通过`Substitution`方法返回的内容。为了简化这个过程，我创建了`SubstitutionFragment`类（见**清单 6-10**）并将其放在`App_Code`文件夹中。

**清单 6-10.** `SubstitutionFragment 类`
```csharp
using System.IO;
using System.Text;
using System.Web;
using System.Web.UI;

public abstract class SubstitutionFragment : UserControl
{
    public abstract void BindToContext();

    private HttpContext _currentContext;

    public HttpContext CurrentContext
    {
        get
        {
            return _currentContext;
        }
        set
        {
            _currentContext = value;
        }
    }

    public virtual string RenderToString(HttpContext context)
    {
        CurrentContext = context;
        BindToContext();
        DataBind();
        StringBuilder sb = new StringBuilder();
        StringWriter sw = new StringWriter(sb);
        HtmlTextWriter tw = new HtmlTextWriter(sw);
        RenderControl(tw);
        return sb.ToString();
    }
}
```

你首先应该注意到的是，`SubstitutionFragment`类继承自`UserControl`作为基类，然后将`BindToContext`方法声明为抽象方法。任何将被用作替换中片段的用户控件，都需要将其继承的类从`UserControl`更改为`SubstitutionFragment`，并实现`BindToContext`方法（见**清单 6-11**）。其余工作由`RenderToString`方法处理，该方法将用户控件转换为可用于`Substitution`控件的字符串。

**清单 6-11.** `作为 SubstitutionFragment 的用户控件`
```csharp
using System.Web;

public partial class Controls_ProductDetailSF : SubstitutionFragment
{
    public override void BindToContext()
    {
        Product product = GetProduct(CurrentContext);
        if (product != null)
        {
            lblProductName.Text = product.Name;
            lblProductNumber.Text = product.ProductNumber;
            lblPrice.Text = product.Price.ToString("C");
            lblAvailability.Text = product.Availability;
        }
    }

    private static Product GetProduct(HttpContext context)
    {
        return Utility.GetProduct(context);
    }
}
```

上下文使得使用查询字符串值加载产品成为可能，我已将该值放在`App_Code`中一个名为`Utility`的类中。此方法也适用于任何其他页面或用户控件，它们可以简单地传入通常由页面或用户控件类填充的`Context`属性（见**清单 6-12**）。

**清单 6-12.** `Utility 类中的 GetProduct 方法`
```csharp
public static Product GetProduct(HttpContext context)
{
    if (context == null)
    {
        return null;
    }

    Product product = context.Items["CurrentProduct"] as Product;
    if (product == null)
    {
        // 使用 context.Request 来加载 Product
        string productIdStr = context.Request.QueryString["ProductID"];
        int productId = -1;
        int.TryParse(productIdStr, out productId);
        Domain domain = new Domain();
        DataSet productDs = domain.GetProductByID(productId);
        if (productDs != null && productDs.Tables.Count > 0 && productDs.Tables[0].Rows.Count > 0)
        {
            DataRow row = productDs.Tables[0].Rows[0];
            // ... (方法逻辑的剩余部分，用于从 DataRow 填充产品)
        }
    }
    return product;
}
```


