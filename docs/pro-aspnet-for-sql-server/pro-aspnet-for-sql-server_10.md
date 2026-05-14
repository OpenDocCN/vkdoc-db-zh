# 第 1 章 ■ 入门

## 清单 1-13. `UserManager.ascx.cs`

```xml
<asp:Label ID="UserCommentLabel" runat="server" Text="备注：" Font-Bold="True"></asp:Label><br />

<asp:Label ID="UserCommentValueLabel" runat="server" Text=""></asp:Label>

<asp:Button ID="EditUserButton" runat="server" Text="编辑用户" OnClick="EditUserButton_Click" />
<asp:Button ID="ResetPasswordButton" runat="server" Text="重置密码" OnClick="ResetPasswordButton_Click" Visible="False" />
<asp:Button ID="UnlockUserButton" runat="server" OnClick="UnlockUserButton_Click" Text="解锁" />
<asp:Button ID="ReturnViewUserButton" runat="server" Text="返回" OnClick="ReturnViewUserButton_Click" />
```

```xml
<asp:View ID="EditorView" runat="server" OnActivate="EditorView_Activate">
    <table>
        <tr>
            <td class="Label">
                <asp:Label ID="UserName2Label" runat="server" Text="用户名：" Font-Bold="True"></asp:Label>
            </td>
            <td class="Data">
                <asp:Label ID="UserNameValue2Label" runat="server" Text=""></asp:Label>
            </td>
        </tr>
        <tr>
            <td class="Label">
                <asp:Label ID="EmailLabel" runat="server" Text="电子邮件：" Font-Bold="True"></asp:Label>
            </td>
            <td class="Data">
                <asp:TextBox ID="EmailTextBox" runat="server" AutoCompleteType="Email"></asp:TextBox>
            </td>
            <td>
                <asp:RequiredFieldValidator ID="EmailRequiredFieldValidator" runat="server" ErrorMessage="*" ControlToValidate="EmailTextBox" EnableClientScript="False"></asp:RequiredFieldValidator>
                <asp:RegularExpressionValidator ID="RegularExpressionValidator1" runat="server" ControlToValidate="EmailTextBox" EnableClientScript="False" ErrorMessage="*" ValidationExpression="\w+([-+.']\w+)*@\w+([-.]\w+)*\.\w+([-.]\w+)*"></asp:RegularExpressionValidator>
            </td>
        </tr>
        <tr>
            <td class="Label">
                <asp:Label ID="CommentLabel" runat="server" Text="备注：" Font-Bold="True"></asp:Label>
            </td>
            <td class="Data">
                <asp:TextBox ID="CommentTextBox" runat="server" AutoCompleteType="Email" TextMode="MultiLine"></asp:TextBox>
            </td>
        </tr>
        <tr>
            <td class="Label">
                <asp:Label ID="Approved2Label" runat="server" Text="已批准：" Font-Bold="True"></asp:Label>
            </td>
            <td class="Data">
                <asp:CheckBox ID="ApprovedCheckBox" runat="server" />
            </td>
        </tr>
        <tr>
            <td class="Label">
                <asp:Label ID="RolesLabel" runat="server" Text="角色 " Font-Bold="True"></asp:Label>
            </td>
            <td class="Data">
                <asp:CheckBoxList ID="RolesCheckBoxList" runat="server"></asp:CheckBoxList>
            </td>
        </tr>
        <tr>
            <td colspan="3">
                <asp:Button ID="UpdateUserButton" runat="server" Text="更新用户" OnClick="UpdateUserButton_Click" />
                <asp:Button ID="CancelEditUserButton" runat="server" OnClick="CancelEditUserButton_Click" Text="取消" />
            </td>
        </tr>
    </table>
</asp:View>
```

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Web;
using System.Web.Security;
using System.Web.UI;
using System.Web.UI.WebControls;

public partial class MemberControls_UserManager : UserControl
{
    #region " 事件 "

    protected void Page_Init(object sender, EventArgs e)
    {
    }

    protected void Page_Load(object sender, EventArgs e)
    {
    }

    protected void FilterUsersButton_Click(object sender, EventArgs e)
    {
        BindUsersGridView();
    }

    protected void EditUserButton_Click(object sender, EventArgs e)
    {
        UsersMultiView.SetActiveView(EditorView);
    }

    protected void UnlockUserButton_Click(object sender, EventArgs e)
    {
        MembershipUser user = CurrentUser;
        if (user != null)
        {
            user.UnlockUser();
            Membership.UpdateUser(user);
            BindUserView();
        }
    }

    protected void ResetPasswordButton_Click(object sender, EventArgs e)
    {
        MembershipUser user = CurrentUser;
        if (user != null)
        {
            user.ResetPassword();
        }
        UsersMultiView.SetActiveView(UserView);
    }

    protected void ReturnViewUserButton_Click(object sender, EventArgs e)
    {
    }
    #endregion
}
```

