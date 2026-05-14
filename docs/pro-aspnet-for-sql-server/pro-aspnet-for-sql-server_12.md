# 第 1 章 ■ 入门指南

## 清单 1-14. `RoleManager.ascx`

```
<%@ Control Language="C#" AutoEventWireup="true" CodeFile="RolesManager.ascx.cs"
Inherits="MemberControls_RolesManager" %>
<asp:Label ID="TitleLabel" runat="server"
Text="角色管理器" Font-Bold="true"></asp:Label>
<table>
<tr>
<td align="center">
<asp:GridView ID="RolesGridView" runat="server"
AutoGenerateColumns="False"
OnRowCommand="RolesGridView_RowCommand"
OnRowDataBound="RolesGridView_RowDataBound"
GridLines="None"
Width="100%">
<Columns>
<asp:TemplateField HeaderText="角色">
<ItemTemplate>
<asp:Label ID="Label1" runat="server"
Text="角色"></asp:Label>
</ItemTemplate>
</asp:TemplateField>
<asp:TemplateField HeaderText="用户">
<ItemTemplate>
<asp:Label ID="Label1" runat="server"
Text="用户"></asp:Label>
</ItemTemplate>
</asp:TemplateField>
<asp:ButtonField CommandName="RemoveRole"
Text="移除" />
</Columns>
<EmptyDataTemplate>
&nbsp;- 暂无角色 -
</EmptyDataTemplate>
<RowStyle CssClass="EvenRow" />
<AlternatingRowStyle CssClass="OddRow" />
<HeaderStyle CssClass="HeaderRow" />
</asp:GridView>
</td>
</tr>
<tr>
<td align="center">
<asp:Label ID="Label3" runat="server"
Font-Bold="True" Text="角色："></asp:Label>
<asp:TextBox ID="AddRoleTextBox" runat="server"
Width="75px"></asp:TextBox>
<asp:Button ID="AddRoleButton" runat="server"
Text="添加" OnClick="AddRoleButton_Click" /></td>
</tr>
</table>
```

## 清单 1-15. `RoleManager.ascx.cs`

```csharp
using System;
using System.ComponentModel;
using System.Web.Security;
using System.Web.UI;
using System.Web.UI.WebControls;

public partial class MemberControls_RolesManager : UserControl
{
    #region " 事件 "

    protected void Page_PreRender(object sender, EventArgs e)
    {
        BindRolesGridView();
    }

    protected void RolesGridView_RowDataBound(object sender, GridViewRowEventArgs e)
    {
        if (e.Row.RowType == DataControlRowType.DataRow)
        {
            string role = e.Row.DataItem as string;

            foreach (TableCell cell in e.Row.Cells)
            {
                foreach (Control control in cell.Controls)
                {
                    Label label = control as Label;

                    if (label != null)
                    {
                        if ("角色".Equals(label.Text))
                        {
                            label.Text = role;
                        }
                        else if ("用户".Equals(label.Text))
                        {
                            label.Text = Roles.GetUsersInRole(role).Length.ToString();
                        }
                    }
                    else
                    {
                        LinkButton button = control as LinkButton;

                        if (button != null)
                        {
                            button.Enabled = Roles.GetUsersInRole(role).Length == 0;

                            if (button.Enabled)
                            {
                                button.CommandArgument = role;
                                button.Attributes.Add("onclick",
                                    "return confirm('您确定吗？');");
                            }
                        }
                    }
                }
            }
        }
    }

    protected void RolesGridView_RowCommand(
        object sender, GridViewCommandEventArgs e)
    {
        if ("RemoveRole".Equals(e.CommandName))
        {
            string role = e.CommandArgument as string;
            Roles.DeleteRole(role, true);
            BindRolesGridView();
        }
    }

    protected void AddRoleButton_Click(object sender, EventArgs e)
    {
        if (Page.IsValid)
        {
            string role = AddRoleTextBox.Text;

            if (!Roles.RoleExists(role))
            {
                Roles.CreateRole(role);
                AddRoleTextBox.Text = String.Empty;
                BindRolesGridView();
            }
        }
    }

    #endregion

    #region " 方法 "

    public void Refresh()
    {
        BindRolesGridView();
    }

    private void BindRolesGridView()
    {
        RolesGridView.DataSource = Roles.GetAllRoles();
        RolesGridView.DataBind();
    }

    #endregion

    #region " 属性 "

    [Category("外观"), Browsable(true), DefaultValue("角色管理器")]
    public string Title
    {
        get
        {
            return TitleLabel.Text;
        }
        set
        {
            TitleLabel.Text = value;
        }
    }

    #endregion
}
```


