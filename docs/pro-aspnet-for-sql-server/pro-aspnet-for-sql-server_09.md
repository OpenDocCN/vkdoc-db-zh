# 第 1 章：入门

**代码清单 1-8.** *角色配置*
```xml
<roleManager defaultProvider="Chapter01SqlRoleProvider" enabled="true">
    <providers>
        <clear/>
        <add
            name="Chapter01SqlRoleProvider"
            connectionStringName="chapter01db"
            applicationName="/chapter01"
            type="System.Web.Security.SqlRoleProvider, System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"
        />
    </providers>
</roleManager>
```

## 配置文件（Profile）配置

最后，你可以添加对配置文件（Profile）提供程序的支持。与前面两个提供程序配置一样，你需要将 `defaultProvider` 属性设置为与新添加的配置相匹配，并添加 `clear` 元素以确保这是唯一配置的提供程序。配置文件提供程序配置的独特功能是能够定义自定义属性。

在代码清单 1-9 所示的示例配置中，定义了一些自定义属性：`FirstName`、`LastName` 和 `BirthDate`。我稍后会解释这些属性；表 1-2 列出了配置文件配置设置。

**代码清单 1-9.** *配置文件配置*
```xml
<profile defaultProvider="Chapter01SqlProfileProvider" automaticSaveEnabled="true" enabled="true">
    <providers>
        <clear/>
        <add
            name="Chapter01SqlProfileProvider"
            applicationName="/chapter01"
            connectionStringName="chapter01db"
            type="System.Web.Profile.SqlProfileProvider, System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"/>
    </providers>
    <properties>
        <add name="FirstName" type="String" allowAnonymous="true" />
        <add name="LastName" type="String" allowAnonymous="true" />
        <add name="BirthDate" type="DateTime" allowAnonymous="true" />
    </properties>
</profile>
```

**表 1-2.** *配置文件配置设置*

**设置** | **描述**
---|---
`name` | 指定 profile 元素引用的配置名称。
`applicationName` | 定义在配置文件数据库中用作范围的应用程序名称。
`connectionStringName` | 指定此提供程序要使用的连接字符串。

你可以在末尾附近看到那些自定义属性。配置文件提供程序允许你管理绑定到 ASP.NET 用户的属性。你可以定义任何可以序列化的属性。在三个示例中，`String` 和 `DateTime` 对象都可以轻松序列化以存储在 SQL Server 数据库中。然而，属性可以比这些常见的基元类型复杂得多。也许你有一个名为 `Employee` 的业务对象，它包含诸如 `Title`、`Name`、`Department` 和 `Office` 之类的属性。与其配置所有这些属性，你可以直接将 `Employee` 对象指定为一个属性。

然后，通过在你的后台代码类或位于 `App_Code` 文件夹中的类里编写代码，你可以通过 `Profile.Employee` 访问此属性。多亏了作为 ASP.NET 运行时一部分的动态编译器，这些属性在 Visual Studio 中立即可用，并支持 IntelliSense。这听起来真的很强大，不是吗？确实如此。

然而，如果你将许多复杂对象设置为配置文件属性，你可能会很快陷入困境。当你需要向 `Employee` 对象添加或删除属性，但数据库中已经保存了许多该对象的序列化版本时，会发生什么？你将如何升级它们？

当 ASP.NET 2.0 首次发布时，我看到各种示例展示了如何创建一个名为 `ShoppingBasket` 的对象，将名为 `Items` 的对象添加到 `ShoppingBasket` 中，并将 `ShoppingBasket` 设置为配置文件属性。我立刻意识到这不会是我愿意尝试的做法。在 .NET 2.0 发布的前一年，我正在构建一个商业网站来管理包含商品项和客户订单的购物篮，而我从未使用过配置文件属性。我仍然有我的 `ShoppingBasket` 对象，它包含许多 `Item` 对象，但我管理了与这些对象一起使用的表和存储过程，这样我就可以轻松地为它们添加新属性并管理变更。如果 `Item` 开始时只有一个名为 `Price` 的价格值，而后来我需要添加另外几个，如 `SalesPrice` 和 `SeasonalPrice`，添加它们很容易。为了将它们与当前网站用户关联起来，我使用了以下技术。

对于这个特定网站，我需要同时处理匿名用户和已认证用户。当用户首次访问该网站时，会获得一个令牌以在页面间导航时标识他。这是一个匿名用户令牌。用户创建账户后，该匿名令牌将被删除，用户将被迁移到一个已认证用户令牌。已认证用户是使用成员资格（Membership）提供程序的会员。为了无缝处理匿名和已认证用户，我需要一种方法来唯一标识这些用户。不幸的是，没有像 `Profile.UserID` 这样的默认值。有一个 `Profile.UserName` 属性，但它仅对已认证用户可用。所以我最终创建了一个包装属性，无论用户是匿名还是已认证，都提供一个 `Guid` 值。

## 匿名配置文件

要使用配置文件上的属性，必须首先启用匿名配置文件。`system.web` 中名为 `anonymousIdentification` 的元素可以被启用以开启此功能。

在 `App_Code` 文件夹中，我创建了一个名为 `Utility` 的类，并添加了代码清单 1-10 中的属性。

**代码清单 1-10.** *实用工具方法*
```csharp
public static bool IsUserAuthenticated
{
    get
    {
        return HttpContext.Current.User.Identity.IsAuthenticated;
    }
}

public static Guid UserID
{
    get
    {
        if (IsUserAuthenticated)
        {
            return (Guid)Membership.GetUser().ProviderUserKey;
        }
        else
        {
            Guid userId = new Guid(HttpContext.Current.Request.AnonymousID.Substring(0, 36));
            return userId;
        }
    }
}
```

每当我需要当前用户的唯一标识符时，我会通过 `Utility.UserID` 来访问它。但随后我必须处理从匿名用户到已认证用户的过渡。

为此，我在 `Global.asax` 中添加了一个名为 `Profile_OnMigrateAnonymous` 的方法，如代码清单 1-11 所示。

**代码清单 1-11.** *Profile_OnMigrateAnonymous*
```csharp
public void Profile_OnMigrateAnonymous(object sender, ProfileMigrateEventArgs args)
{
    Guid anonID = new Guid(args.AnonymousID);
    Guid authId = (Guid)Membership.GetUser().ProviderUserKey;

    // 将匿名用户资源迁移到已认证用户

    // 删除匿名用户。
    Membership.DeleteUser(args.AnonymousID, true);
}
```

以购物篮为例，我只需将匿名用户购物篮中的商品项添加到已认证用户的购物篮中，然后删除匿名账户及其所有关联数据。

我曾考虑在配置文件属性中添加一个名为 `UserID` 的 `Guid` 属性，但 ASP.NET 运行时的动态编译器仅在后台代码文件中有效。它在位于 `App_Code` 目录中的 `Utility` 类中不起作用，而我大部分代码都放在那里。所以上述技术是当时唯一的选择。

#### 创建用户和角色



当你在 Visual Studio 中处理网站时，可以使用**网站管理工具**来创建用户和角色。但此实用程序并非**微软互联网信息服务 (IIS)** 的内置功能。要管理用户和角色，你必须亲自动手——要么通过谨慎调用存储过程在数据库中操作，要么创建一个界面来安全地使用**成员资格 API**。我选择创建几个用户控件，可以轻松地将它们放入任何网站中。

两个主要控件 `UserManager.ascx` 和 `RolesManager.ascx` 负责处理用户和角色的管理。这些控件被包含在另一个名为 `MembersControl.ascx` 的用户控件中。

这个控件在三种视图之间切换，用于创建新用户、编辑现有用户以及编辑角色。清单 1-12 到 1-17 提供了这些控件的完整源代码。

### 清单 1-12. `UserManager.ascx`

```
<%@ Control Language="C#" AutoEventWireup="true"
CodeFile="UserManager.ascx.cs" Inherits="MemberControls_UserManager" %>

<asp:Label ID="TitleLabel"
runat="server" Text="用户管理器"></asp:Label><br />

<asp:MultiView ID="UsersMultiView" runat="server"
ActiveViewIndex="0">

<asp:View ID="SelectUserView" runat="server"
OnActivate="SelectUserView_Activate">

<table>
<tr>
<td>
<asp:GridView ID="UsersGridView" runat="server"
AllowPaging="True"
AutoGenerateColumns="False"
OnInit="UsersGridView_Init"
OnPageIndexChanging="UsersGridView_PageIndexChanging"
OnRowCommand="UsersGridView_RowCommand"
GridLines="None">
<Columns>
<asp:BoundField DataField="UserName" HeaderText="用户名" />
<asp:BoundField DataField="Email" HeaderText="电子邮件" />
<asp:TemplateField ShowHeader="False">
<ItemTemplate>
<asp:LinkButton ID="LinkButton1" runat="server"
CausesValidation="false"
CommandName="ViewUser"
CommandArgument='<%# Bind("UserName") %>'
Text="查看"></asp:LinkButton>
</ItemTemplate>
</asp:TemplateField>
</Columns>
<RowStyle CssClass="EvenRow" />
<HeaderStyle CssClass="HeaderRow" />
<AlternatingRowStyle CssClass="OddRow" />
</asp:GridView>
</td>
</tr>
<tr>
<td align="center">
<asp:TextBox ID="FilterUsersTextBox" runat="server"
Width="75px"></asp:TextBox>
<asp:Button ID="FilterUsersButton" runat="server"
OnClick="FilterUsersButton_Click" Text="筛选" />
</td>
</tr>
</table>

</asp:View>

<asp:View ID="UserView" runat="server" OnActivate="UserView_Activate">

<table>
<tr>
<td class="Label">
<asp:Label ID="UserNameLabel" runat="server" Text="用户名："
Font-Bold="True"></asp:Label></td>
<td class="Data">
<asp:Label ID="UserNameValueLabel" runat="server"
Text=""></asp:Label></td>
</tr>
<tr>
<td class="Label">
<asp:Label ID="ApprovedLabel" runat="server" Text="已批准："
Font-Bold="True"></asp:Label></td>
<td class="Data">
<asp:Label ID="ApprovedValueLabel" runat="server"
Text=""></asp:Label></td>
</tr>
<tr>
<td class="Label">
<asp:Label ID="LockedOutLabel" runat="server" Text="已锁定："
Font-Bold="True"></asp:Label></td>
<td class="Data">
<asp:Label ID="LockedOutValueLabel" runat="server"
Text=""></asp:Label></td>
</tr>
<tr>
<td class="Label">
<asp:Label ID="OnlineLabel" runat="server" Text="在线："
Font-Bold="True"></asp:Label></td>
<td class="Data">
<asp:Label ID="OnlineValueLabel" runat="server"
Text=""></asp:Label></td>
</tr>
<tr>
<td class="Label">
<asp:Label ID="CreationLabel" runat="server" Text="创建时间："
Font-Bold="True"></asp:Label></td>
<td class="Data">
<asp:Label ID="CreationValueLabel" runat="server"
Text=""></asp:Label></td>
</tr>
<tr>
<td class="Label">
<asp:Label ID="LastActivityLabel" runat="server"
Text="最后活动时间：" Font-Bold="True"></asp:Label></td>
<td class="Data">
<asp:Label ID="LastActivityValueLabel" runat="server"
Text=""></asp:Label></td>
</tr>
<tr>
<td class="Label">
<asp:Label ID="LastLoginLabel" runat="server"
Text="最后登录时间：" Font-Bold="True"></asp:Label></td>
<td class="Data">
<asp:Label ID="LastLoginValueLabel" runat="server"
Text=""></asp:Label></td>
</tr>
<tr>
<td colspan="2" class="Data">
```


