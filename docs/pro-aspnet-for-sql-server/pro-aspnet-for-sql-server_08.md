# 第 1 章 ■ 入门指南

在开发具有这些功能的网站时，您偶尔会希望重置一切并从头开始。为此，您可以使用脚本来移除提供程序服务。但是，如果表中存在数据，该实用程序不允许移除这些服务。为了帮助完成此操作，您必须按照正确的顺序删除数据，因为表之间存在外键约束。

为此，请使用清单 1-2 中名为`WipeProviderData.sql`的以下脚本。

**清单 1-2.** *WipeProviderData.sql*

```
--
-- WipeProviderData.sql
--
-- 清除提供程序服务表中的数据，以便移除服务
-- （并重新添加）
--
-- SELECT name FROM sysobjects WHERE type = 'U' and name like 'aspnet_%'
--

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_WebEvent_Events')
BEGIN
delete from dbo.aspnet_WebEvent_Events
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_PersonalizationAllUsers')
BEGIN
delete from dbo.aspnet_PersonalizationAllUsers
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_PersonalizationPerUser')
BEGIN
delete from dbo.aspnet_PersonalizationPerUser
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_Membership')
BEGIN
delete from dbo.aspnet_Membership
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_Profile')
BEGIN
delete from dbo.aspnet_Profile
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_UsersInRoles')
BEGIN
delete from dbo.aspnet_UsersInRoles
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_Users')
BEGIN
delete from dbo.aspnet_Users
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_Roles')
BEGIN
delete from dbo.aspnet_Roles
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_Paths')
BEGIN
delete from dbo.aspnet_Paths
END
GO

IF EXISTS (SELECT * FROM sysobjects WHERE type = 'U' AND
name = 'aspnet_Applications')
BEGIN
delete from dbo.aspnet_Applications
END
GO
```

您可以将此脚本放入您的脚本文件夹`D:\Projects\Common\Scripts`中，以便在多个项目中使用。当您想要使用它时，将其加载到 SQL Server Management Studio 中，并在您想要清除的数据库上下文中运行它。然后，您可以运行清单 1-3 所示的移除脚本`Remove Provider Services.cmd`。

**清单 1-3.** *Remove Provider Services.cmd*

```
@echo off
set REGSQL="%windir%\Microsoft.NET\Framework\v2.0.50727\aspnet_regsql.exe"
set DSN="Data Source=.\SQLEXPRESS;Initial Catalog=Chapter01;Integrated Security=True"
%REGSQL% -C %DSN% -R mrpc
Pause
```

此脚本与`Add Provider Services.cmd`相同，只是命令行开关从`-A`简单地更改为`-R`，后者指定要移除服务。数据清除后，此脚本将成功运行。

#### 混合与匹配提供程序

由于提供程序的性质，可以将这些提供程序服务部署到网站其余部分所使用的同一数据库，或完全不同的数据库。甚至可以为每个单独的提供程序将服务部署到单独的数据库。这样做可能有很好的理由。在一种情况下，您可能正在运行一个连接到庞大数据库的网站，该数据库需要偶尔下线进行维护。这样做时，可能完全没必要让提供程序服务也一起下线。您可能还会发现，通过在不同的硬件上托管提供程序服务，可以获得更好的性能。

您的配置选项远不止于此，这将在下一节中解释。

###### 配置提供程序


当你使用 Visual Studio 2005 首次创建一个新的 ASP.NET 2.0 网站时，它已经预配置好以使用一组默认设置。这些默认设置在 `Machine.config` 文件中定义，该文件是 .NET 2.0 安装的一部分。通常它位于 `C:\WINDOWS\Microsoft.NET\Framework\v2.0.50727\CONFIG`，或者位于你计算机上安装 .NET 2.0 的任何位置。

提供程序配置的默认设置靠近文件底部，在 `system.web` 节中，如清单 1-4 所示。

**清单 1-4.** Machine.config 中的 system.web

```xml
<system.web>

<processModel autoConfig="true"/>

<httpHandlers />

<membership>

<providers>

<add name="AspNetSqlMembershipProvider"

type="System.Web.Security.SqlMembershipProvider, System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"

connectionStringName="LocalSqlServer"

enablePasswordRetrieval="false"

enablePasswordReset="true"

requiresQuestionAndAnswer="true"

applicationName="/"

requiresUniqueEmail="false"

passwordFormat="Hashed"

maxInvalidPasswordAttempts="5"

minRequiredPasswordLength="7"

minRequiredNonalphanumericCharacters="1"

passwordAttemptWindow="10"

passwordStrengthRegularExpression="" />

</providers>

</membership>

<profile>

<providers>

<add name="AspNetSqlProfileProvider"

connectionStringName="LocalSqlServer" applicationName="/"

type="System.Web.Profile.SqlProfileProvider, System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a" />

</providers>

</profile>

<roleManager>

<providers>

<add name="AspNetSqlRoleProvider"

connectionStringName="LocalSqlServer" applicationName="/"

type="System.Web.Security.SqlRoleProvider, System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a" />

<add name="AspNetWindowsTokenRoleProvider" applicationName="/"

type="System.Web.Security.WindowsTokenRoleProvider, System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a" />

</providers>

</roleManager>

</system.web>
```

同时还定义了一个名为 `LocalSqlServer` 的默认连接字符串，它会在 `App_Data` 文件夹中寻找一个名为 `aspnetdb.mdf` 的文件（见清单 1-5）。

**清单 1-5.** Machine.config 中的 connectionStrings

```xml
<connectionStrings>

<add name="LocalSqlServer"

connectionString="data source=.\SQLEXPRESS;Integrated Security=SSPI;AttachDBFilename=|DataDirectory|aspnetdb.mdf;User Instance=true"

providerName="System.Data.SqlClient"/>

</connectionStrings>
```

默认的 `LocalSqlServer` 连接被 `system.web` 节中的成员资格、角色和配置文件提供程序配置所引用。数据源指定了该数据库存在于 `DATA_DIRECTORY` 中。对于 ASP.NET 2.0 网站，这个 `DATA_DIRECTORY` 就是 `App_Data` 文件夹。如果你创建一个新网站并开始使用成员资格和配置文件服务，这个 SQL Express 数据库将自动在 `App_Data` 文件夹中为你创建。

但是，如果你安装的是 SQL Server 的标准版或专业版，此过程将会失败，因为它需要 SQL Express。这个默认配置在未针对部署环境进行调整时可能非常方便，但也可能存在危险。

对于每个提供程序配置，父块都包含针对各自提供程序实现的多个属性。并且在该块内，你可以添加、移除甚至清除提供程序实现。在你新网站的 `Web.config` 文件中，你需要清除由 `Machine.config` 设置的默认值，并根据你的具体需求进行自定义。

接下来，你将配置一个新网站以使用成员资格、角色和配置文件提供程序的 SQL 实现。在配置这些之前，你必须准备好数据源。

假设你有一个示例网站，它将与一个名为 `Sample` 的数据库一起工作，并且该数据库已按照上一节所述准备好提供程序服务，请使用清单 1-6 中的配置。

**清单 1-6.** 自定义 Web.config

```xml
<connectionStrings>

<add name="sampledb"

connectionString="Data Source=.\SQLEXPRESS;Initial Catalog=Sample; Integrated Security=True"

providerName="System.Data.SqlClient"/>

</connectionStrings>
```

在开发过程中，我使用受信任的数据库连接并连接到本地计算机。现在你已经准备好使用 SQL 实现来配置提供程序了。并且因为 `Machine.config` 中的默认设置已经使用了 SQL 实现，让我们从它们开始，然后进行调整。

#### 成员资格配置

首先，你将组织配置块以使其更具可读性。在开发过程中你会经常查看它，因此使其易于一眼看懂将节省你的时间。将每个属性放在单独的行上，并缩进属性以使它们对齐。然后将 `name`、`applicationName` 和 `connectionStringName` 作为前几个属性。这些是关键值。

接下来，你希望确保这是此网站唯一的成员资格提供程序。

在 `membership` 块中的 `add` 元素之前添加 `clear` 元素，以告知 ASP.NET 运行时清除所有预配置的设置。然后向 `membership` 元素添加一个名为 `defaultProvider` 的属性，并将其设置为与新添加的提供程序配置的 `name` 相同的值。清单 1-7 展示了成员资格配置。表 1-1 涵盖了可用的各种设置。

**清单 1-7.** 成员资格配置

```xml
<membership defaultProvider="Chapter01SqlMembershipProvider">

<providers>

<clear/>

<add

name="Chapter01SqlMembershipProvider"

applicationName="/chapter01"

connectionStringName="chapter01db"

enablePasswordRetrieval="true"

enablePasswordReset="true"

requiresQuestionAndAnswer="true"

requiresUniqueEmail="false"

passwordFormat="Clear"

maxInvalidPasswordAttempts="5"

minRequiredPasswordLength="7"

minRequiredNonalphanumericCharacters="0"

passwordAttemptWindow="10"

passwordStrengthRegularExpression=""

type="System.Web.Security.SqlMembershipProvider, System.Web, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"

/>

</providers>

</membership>
```

**表 1-1.** 成员资格配置设置

| **设置** | **描述** |
| :--- | :--- |
| `name` | 指定由 `membership` 元素引用的配置名称。 |
| `applicationName` | 定义用作成员资格数据库中范围的应用程序名称。 |
| `connectionStringName` | 指定此提供程序要使用的连接字符串。 |
| `enablePasswordRetrieval` | 指定此提供程序是否允许密码检索。 |
| `enablePasswordReset` | 指定此提供程序是否可以重置用户密码（启用 = `true`）。 |
| `requiresQuestionAndAnswer` | 指定密码重置和检索是否需要密码提示问题和答案。 |
| `requiresUniqueEmail` | 指定此提供程序是否要求每个用户具有唯一的电子邮件地址。 |
| `passwordFormat` | 指定密码格式，例如 `cleared`（明文）、`hashed`（哈希，默认）和 `encrypted`（加密）。 |
| `maxInvalidPasswordAttempts` | 指定在用户账户被锁定之前允许的失败登录尝试次数。 |
| `minRequiredPasswordLength` | 指定密码所需的最小字符数。 |
| `minRequiredNonalphanumericCharacters` | 指定密码中必须包含的特殊字符数量。 |
| `passwordAttemptWindow` | 跟踪失败尝试的时间窗口（以分钟为单位）。 |
| `passwordStrengthRegularExpression` | 用于检查密码字符串是否符合所需密码强度的正则表达式。 |

#### 角色配置



现在，你将对角色提供程序（Roles provider）进行大致相同的操作。这个配置块名为 `roleManager`，除了 `defaultProvider` 属性外，它还有一个名为 `enabled` 的属性，允许你开启或关闭它。之后，你可以通过 `Roles.Enabled` 属性在代码中轻松访问 `enabled` 设置。默认情况下，此值设置为 `false`，因此，即使你已经配置了提供程序，也必须显式设置此值以添加对角色的支持。请参见代码清单 1-8 中的角色配置。

