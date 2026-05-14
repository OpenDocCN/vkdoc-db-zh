# 第 1 章 入门

**清单 1-16.** `MembersControl.ascx`

```aspx
<%@ Control Language="C#" AutoEventWireup="true"
    CodeFile="MembersControl.ascx.cs"
    Inherits="MembersControl" %>
<%@ Register Src="UserManager.ascx" TagName="UserManager" TagPrefix="uc2" %>
<%@ Register Src="RolesManager.ascx" TagName="RolesManager" TagPrefix="uc1" %>

<br />
<br />

<b>选择视图:</b>

<asp:DropDownList ID="NavDropDownList" runat="server"
    AutoPostBack="True"
    OnSelectedIndexChanged="NavDropDownList_SelectedIndexChanged">
    <asp:ListItem>创建用户</asp:ListItem>
    <asp:ListItem>管理用户</asp:ListItem>
    <asp:ListItem>管理角色</asp:ListItem>
</asp:DropDownList>

<br />
<br />

<asp:MultiView ID="MultiView1" runat="server">
    <asp:View ID="UserCreationView" runat="server"
        OnActivate="UserCreationView_Activate">
        <asp:CreateUserWizard ID="CreateUserWizard1" runat="server"
            AutoGeneratePassword="True"
            LoginCreatedUser="False"
            OnCreatedUser="CreateUserWizard1_CreatedUser">
            <WizardSteps>
                <asp:CreateUserWizardStep
                    ID="CreateUserWizardStep1" runat="server">
                </asp:CreateUserWizardStep>
                <asp:CompleteWizardStep
                    ID="CompleteWizardStep1" runat="server">
                </asp:CompleteWizardStep>
            </WizardSteps>
        </asp:CreateUserWizard>
    </asp:View>
    <asp:View ID="UserManagerView" runat="server"
        OnActivate="UserManagerView_Activate">
        <uc2:UserManager
            ID="UserManager1" runat="server"
            Title="用户"
            TitleBold="true" />
    </asp:View>
    <asp:View ID="RolesManagerView" runat="server">
        <uc1:RolesManager
            ID="RolesManager1" runat="server"
            Title="角色" />
    </asp:View>
</asp:MultiView>
```

**清单 1-17.** `MembersControl.ascx.cs`

```csharp
using System;
using System.Web.UI;

public partial class MembersControl : UserControl
{
    protected void Page_Load(object sender, EventArgs e)
    {
        if (!IsPostBack)
        {
            MultiView1.SetActiveView(UserManagerView);
            NavDropDownList.SelectedValue = "管理用户";
        }
    }

    protected void NavDropDownList_SelectedIndexChanged(object sender, EventArgs e)
    {
        if ("创建用户".Equals(NavDropDownList.SelectedValue))
        {
            MultiView1.SetActiveView(UserCreationView);
        }
        else if ("管理用户".Equals(NavDropDownList.SelectedValue))
        {
            MultiView1.SetActiveView(UserManagerView);
            RefreshUserManager();
        }
        else if ("管理角色".Equals(NavDropDownList.SelectedValue))
        {
            MultiView1.SetActiveView(RolesManagerView);
            RefreshRolesManager();
        }
    }

    protected void CreateUserWizard1_CreatedUser(object sender, EventArgs e)
    {
        MultiView1.SetActiveView(UserManagerView);
        NavDropDownList.SelectedValue = "管理用户";
        RefreshUserManager();
    }

    private void RefreshUserManager()
    {
        MemberControls_UserManager userManager =
            UserManagerView.FindControl("UserManager1")
            as MemberControls_UserManager;
        if (userManager != null)
        {
            userManager.Refresh();
        }
    }

    private void RefreshRolesManager()
    {
        MemberControls_RolesManager rolesManager =
            UserManagerView.FindControl("RolesManager1")
            as MemberControls_RolesManager;
        if (rolesManager != null)
        {
            rolesManager.Refresh();
        }
    }

    protected void UserManagerView_Activate(object sender, EventArgs e)
    {
        UserManager1.Reset();
    }

    protected void UserCreationView_Activate(object sender, EventArgs e)
    {
        CreateUserWizard1.ActiveStepIndex = 0;
    }
}
```

#### 保护管理区域

将用户和角色管理控件放入名为`Admin`的文件夹后，可以通过要求`Admin`角色中的经过身份验证的用户来保护它。在`Web.config`的末尾，你可以为`Admin`路径创建一个`location`配置，并允许`Admin`角色中的成员（见清单 1-18）。

**清单 1-18.** 使用`Web.config`保护管理区域

```xml
<?xml version="1.0"?>
<configuration>
    <!-- 其他配置设置 -->
    <location path="Admin">
        <system.web>
            <authorization>
                <allow roles="Admin"/>
                <deny users="*"/>
            </authorization>
        </system.web>
    </location>
</configuration>
```

#### 创建管理员用户

管理区域准备好后，你自然需要管理员用户，以便登录此区域并使用控件。这有点像是第 22 条军规的场景。但正如你可以使用`成员资格` API 来管理用户和角色一样，使用这些控件你也可以以编程方式创建用户和角色。为了自动确保你的网站拥有必要的管理员用户，你可以将完成所有这些工作的代码添加到网站`Global.asax`文件中的`Application_Start`事件处理程序中。我首先检查三个默认角色是否存在，并添加每一个不存在的角色。然后，如果没有用户，我让方法添加默认的管理员用户，如清单 1-19 所示。

**清单 1-19.** 添加角色和用户

```csharp
public void Application_Start(object sender, EventArgs e)
{
    if (Roles.Enabled)
    {
        String[] requiredRoles = { "Admin", "Users", "Editors" };
        foreach (String role in requiredRoles)
        {
            if (!Roles.RoleExists(role))
            {
                Roles.CreateRole(role);
            }
        }

        string[] users = Roles.GetUsersInRole("Admin");
        if (users.Length == 0)
        {
            // 创建管理员用户
            MembershipCreateStatus status;
            Membership.CreateUser("admin", "CHANGE_ME", "admin@localhost",
                "Favorite color?", "green", true, out status);
            if (MembershipCreateStatus.Success.Equals(status))
            {
                Roles.AddUserToRole("admin", "Admin");
            }
            else
            {
                LogMessage("无法创建管理员用户: " + status, true);
            }
        }
    }
}
```

#### 总结

本章介绍了如何为处理 ASP.NET 网站准备环境，以及如何配置数据库连接。你学习了如何配置和管理提供程序服务，包括以编程方式添加用户和角色，以便新网站在部署后可以立即被管理。

