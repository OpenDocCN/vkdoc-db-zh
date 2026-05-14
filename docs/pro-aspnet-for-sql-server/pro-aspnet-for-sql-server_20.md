# 第四章：数据绑定控件

## 用户控件的实现

`TypeName="Chapter04.PersonDomain"></asp:ObjectDataSource>` 这里有趣的是，用户控件上的 `Country` 属性可以像 `Label` 控件（一个标准的 ASP.NET 控件）一样，绑定到 `Country` 值。当数据绑定到 `Repeater1` 时，它会自动设置 `CityListingControl1` 上的 `Country` 属性。

这一切都无需在代码后置文件中添加任何额外代码。清单 4-14 展示了城市列表控件。

**清单 4-14.** *CityListingControl.ascx*

```xml
<%@ Control Language="C#" AutoEventWireup="true"

CodeFile="CityListingControl.ascx.cs" Inherits="Controls_CityListingControl" %>

<asp:Repeater ID="Repeater1" runat="server" DataSourceID="ObjectDataSource1">

<HeaderTemplate>

<ul>

</HeaderTemplate>

<ItemTemplate>

<li>

<asp:Label ID="Label1" runat="server"

Text='<%# Bind("City") %>'></asp:Label><br />

</li>

</ItemTemplate>

FooterTemplate>

</ul>

</FooterTemplate>

</asp:Repeater>

<asp:ObjectDataSource ID="ObjectDataSource1" runat="server"

OldValuesParameterFormatString="original_{0}"

SelectMethod="GetCitiesByCountry"

TypeName="Chapter04.PersonDomain"

OnSelecting="ObjectDataSource1_Selecting">

<SelectParameters>

<asp:Parameter Name="country" Type="String" />

</SelectParameters>

</asp:ObjectDataSource>
```

该用户控件同样利用 `Repeater` 来以项目符号列表的形式列出所有数据。在此情况下，列出的就是所选国家内的所有城市。在代码后置文件中，如清单 4-15 所示，添加了一小段代码将父用户控件连接起来以绑定数据。

**清单 4-15.** *CityListingControl.ascx.cs*

```csharp
protected void ObjectDataSource1_Selecting(

object sender, System.Web.UI.WebControls.ObjectDataSourceSelectingEventArgs e)

{

e.InputParameters["country"] = Country;

}

private string _country;

public string Country

{

get {

return _country;

}

set {

_country = value;

ObjectDataSource1.DataBind();

}

}
```

理论上，你可以按照这种将属性传入用户控件作为输入参数进行绑定的模式，继续深入多个层级，比如列出居住在每个城市中的人员。这样做完全封装了每个用户控件的行为。

### 注意递归数据绑定

当属性被设置时，请确保只绑定 `ObjectDataSource`，而不是整个用户控件。

重新绑定用户控件会导致属性上出现无限递归循环，因为用户控件会反复、多次地绑定数据。

## 视图状态与数据绑定

在使用数据绑定时，`ViewState` 作为回发模型的一部分将被广泛使用。所有绑定到控件（如 `GridView`）的数据都保存在 `ViewState` 中。

默认情况下，数据会被序列化为字符串，进行加密，并作为隐藏输入字段包含在页面中。当发生回发时，来自隐藏字段的值会被解密和反序列化。然后，回发过程会尝试使用这些数据来重置控件的值。通过使用 `ViewState` 来重绘数据绑定控件，你可以避免返回数据库重新获取所有必要的数据。在一个典型的页面上，你可能会有几个数据绑定控件会使用此 `ViewState`。以 `GridView` 为例，你可以单击其中一个分页链接来更改选中的索引，这将导致从数据库中提取新数据。在此期间，其他数据绑定控件将仅使用 `ViewState` 数据来重绘自身。如果你将它与其他需要在每次页面请求时都从数据库重新绑定所有数据的解决方案相比，这是一个相当高效的模型。

然而，使用 `ViewState` 确实意味着当页面显示大量数据时，保存在隐藏输入字段中的 `ViewState` 数据也会很大。它也是对显示数据的一种复制，保存在一个加密字符串中以维护数据的完整性。因此，虽然你在回发期间减少了访问数据库的次数，但也增加了页面的加载时间，这对于那些通过有限带宽连接到你的 Web 应用程序的用户来说，可能同样是一种性能损失。这类用户可能使用调制解调器或移动网络连接，无法提供高速访问。他们也可能与办公室里的所有人共享一个高速连接，这会降低他们的访问速度。

无论如何，你应该通过减少页面的大小，使页面加载时间尽可能快。当 `ViewState` 变大时，你需要考虑减小其大小的方法。

### 页面大小与加载时间

Web 可用性领域的权威 Jakob Nielsen 在他的网站上指出，使用调制解调器连接互联网的用户需要十秒钟来加载一个仅 34 KB 的页面 (http://www.useit.com/alertbox/sizelimits.html)。对于有线电视/DSL 连接，加载一个 100KB 的页面需要一秒钟。一个数据密集型的页面很容易超过 100 KB，因此高速用户可能仍然需要等待两到五秒让你的页面加载完成。削减 `ViewState` 可以为高速用户节省几秒钟，而对于连接速度较慢的用户来说节省的时间会更多。当然，这些延迟还会因为准备和开始向用户发送响应的时间而延长。

#### 会话与视图状态

你可以更改 `ViewState` 的持久化方式，以避免将所有这些额外数据放在页面中。

ASP.NET 2.0 引入了将 `ViewState` 存储在 `Session` 中而非隐藏输入字段中的选项。你只需重写 `PageStatePersister` 属性，返回默认的 `HiddenFieldPagePersister` 或 `SessionPageStatePersister`。清单 4-16 展示了一个例子。

**清单 4-16.** *将 PageStatePersister 设置为使用 Session*

```csharp
protected override PageStatePersister PageStatePersister

{

get

{

return new SessionPageStatePersister(Page);

}

}
```

`SessionPageStatePersister` 的实现将一个 `Guid` 值放入隐藏输入字段以替代所有加密数据，并将实际数据放入用户的 `Session` 中。

当发生回发时，`Guid` 值用于查找该数据。此数据被限制在一个循环队列中，该队列仅保存最近十个页面的数据。虽然这减少了页面大小，但将其置于 `Session` 中的限制也带来了一些问题。

明显的问题是，如果数据从 `Session` 中移除，回发将引发异常。如果用户的 `Session` 在默认的 20 分钟超时后过期，就可能发生这种情况。当用户的 `Session` 过期时，服务器端的 `ViewState` 会随 `Session` 一起被放弃。如果用户离开一段时间后返回，并单击一个按钮导致回发，该用户将看到异常。你可以考虑将会话超时时间延长至八小时，但这样做会导致服务器上使用更多的内存。

如果你为八小时内访问网站的所有用户保留 `Session` 状态，你可能会因所需的内存量而感到困扰。

用户也可能在使用你的 Web 应用程序时打开两个浏览器窗口。他们在一个窗口中点击一段时间，然后切换到另一个窗口继续点击，从而导致回发。如果他们超出了十次点击的限制并返回另一个窗口引发回发，他们将会遇到状态异常。幸运的是，这个功能可以按页面启用。如果你的网站只有有限几个数据密集型页面，你可以设置它们使用 `SessionPageStatePersister`，而其余页面则使用默认功能。


