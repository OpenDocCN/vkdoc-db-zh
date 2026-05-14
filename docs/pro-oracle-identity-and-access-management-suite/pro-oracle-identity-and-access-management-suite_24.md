# Oracle Identity and Access Management Suite 常见配置问题

由于 Oracle Identity and Access Management Suite 有三个主要组件——Oracle Internet Directory (OID)、Oracle Access Manager (OAM) 和 Oracle Identity Manager (OIM)——本节将分别介绍相关问题。

## Oracle Internet Directory

作为 OIM 和 OAM 的后端目录，如果 OID 环境配置不当，整个堆栈必定会出现问题。尽管在此阶段配置向导完成了大部分工作，但仍然可能出现问题。表 16-1 中列出的操作系统包足以满足 OID 安装需求，前述内核参数同样适用。正如 OAM 和 OIM 有 Java 版本范围要求一样，OID 版本对 Java Development Kit (JDK) 版本和更新也相当敏感。对于 OID 11.1.1.9，请确保使用 JDK 1.7 update 80。OID 可以安装在高于 update 80 的环境中，但需要仔细验证兼容性。最常见的情况是，安装似乎运行正确，但在配置阶段，域部署运行时可能会遇到问题。

许多组织使用 Active Directory 环境作为其网络轻量目录访问协议 (LDAP)。通常的做法是建立一个目录集成平台 (DIP) 来保持 OID 环境与 Active Directory 同步。有时，这种同步需要将满足特定条件的用户添加到一个 OID 子树中，而其他用户则放入另一个树中。其他环境可能只想同步满足一组复杂规则的 Active Directory 用户子集。OID 11.1.1.9 目录支持多组同步配置文件，每组都可以有自己的过滤规则集。然而，复杂规则可能会导致问题。很多时候，这仅表现为用户无法同步，并且当尝试在 EM 控制台中加载 DIP 配置文件时，控制台会挂起。此时您还会注意到服务器上的内存使用量激增。在需要 `searchfilter=(&(objectclass=user)(|(company= BB*))(!(objectclass=computer))` 这类规则的情况下，请在搜索过滤器周围添加双引号，使其格式如下：`searchfilter="(&(objectclass=user)(|(company= BB*))(!(objectclass=computer)"`。

## Oracle Access Manager

OAM 处理身份和访问管理环境的单点登录 (SSO) 功能。在配置过程中，可能会出现一些问题。本节介绍一些常见问题及其解决方案。

> **注意**
> 在对 OAM 配置进行任何更改之前，备份 `DOMAIN_HOME/config/fmwconfig/oam_config.xml` 文件可能很有用。

在访问管理器控制台中更改身份存储可能有点棘手。但是，您可能正在更换身份存储，或者将其迁移到新环境。在这些情况下，如果遵循一些准则，此活动可以轻松完成。首先，确保在您计划通过访问管理器管理的新 LDAP 存储中创建一个或多个用户。理想情况下，您会保留一个名为 oamadmin 的用户和一个属于 Administrators 组的用户组。在图 16-1 中，显示了 OAM 管理控制台，其中包含多个身份存储。每个存储中至少配置了一个作为管理员的用户。创建新的身份存储并将其设置为系统存储将在本页面上生成一个新部分，您可以在其中选择用户和组来管理实例，如图 16-2 所示。

![A352855_1_En_16_Fig2_HTML.jpg](img/A352855_1_En_16_Fig2_HTML.jpg)

**图 16-2. OAM 身份存储选择**

![A352855_1_En_16_Fig1_HTML.jpg](img/A352855_1_En_16_Fig1_HTML.jpg)

**图 16-1. OAM 管理用户身份存储屏幕**

执行这些任务后，确保 LDAP 身份验证模块或正在使用的其他身份验证模块配置为使用新的身份存储至关重要。未能执行此步骤可能会阻止您将来访问 OAM 控制台。请查看图 16-3 了解此屏幕示例。

![A352855_1_En_16_Fig3_HTML.jpg](img/A352855_1_En_16_Fig3_HTML.jpg)

**图 16-3. OAM 身份验证模块配置**

即使准备充分，此过程也可能出现问题。如果发生这种情况，并且在进行更改后您完全无法访问 OAM 管理控制台，还有最后一线希望。在 `DOMAIN_HOME/config/fmwconfig` 目录中，您会找到一个名为 `oam-config.xml` 的文件。关闭 OAM 并编辑此文件以更改身份存储位置或其他属性。重新启动 OAM 托管服务器后，一切应恢复正常。

> **注意**


