# SQL 最佳实践评估

此选项是一个工具，允许你扫描 SQL Server 部署的配置，并根据 SQL 社区和 Microsoft 的通用知识，为可能非最优或未遵循最佳实践的设置提供建议。你可以按需或按计划运行评估。该工具提供了一些非常棒的建议，因此我推荐你试一试，看看它能如何帮助你。你可以在 [`https://github.com/microsoft/sql-server-samples/blob/master/samples/manage/sql-assessment-api`](https://github.com/microsoft/sql-server-samples/blob/master/samples/manage/sql-assessment-api) 查看它使用的完整规则集，并在 [`https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-assessment-for-sql-vm`](https://learn.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-assessment-for-sql-vm) 了解更多关于此功能的信息。

### 安全性

安全性选项包括与 Microsoft Entra 和 Microsoft Defender for Cloud 等服务的集成。

### 安全性配置

此选项允许你启用与 `Azure Key Vault` 的集成，这对于使用透明数据加密（自带密钥）等功能来说是一个非常不错的选择。

你还可以启用与 `Microsoft Entra`（前身为 Azure Active Directory）的集成。此选项需要 SQL Server 2022，是为身份验证提供替代方案（替代基本或 SQL 身份验证）的绝佳解决方案，特别是因为你不需要像 Windows 身份验证那样需要域控制器。此功能的一个重要用途是使用托管身份，从而可以构建连接到 SQL Server 的 `无密码` 应用程序。在 [`https://learn.microsoft.com/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview`](https://learn.microsoft.com/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview) 了解更多。

### Microsoft Defender for Cloud

此功能以前称为高级数据安全，但现在已归入 Microsoft Defender 旗下。此处的能力保持不变：

*   漏洞评估
    根据已知的行业标准（例如 FedRAMP），扫描可能导致潜在安全漏洞的任何配置设置（例如，启用了 sa 帐户）。

*   高级威胁防护
    对检测到的任何威胁发出警报，例如暴力登录攻击（启用了 SQL 身份验证且有人试图破解 sa 密码）和 SQL 注入攻击。

你可以在 [`https://learn.microsoft.com/azure/defender-for-cloud/defender-for-sql-usage`](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-sql-usage) 了解更多关于 Microsoft Defender for Cloud 和 Azure VM 上 SQL 的信息。

**提示**

Defender 团队有两个值得关注的新功能，包括 `数据安全仪表板`（[`https://learn.microsoft.com/azure/defender-for-cloud/data-aware-security-dashboard-overview`](https://learn.microsoft.com/azure/defender-for-cloud/data-aware-security-dashboard-overview)）和 `Copilot 体验`（[`https://learn.microsoft.com/azure/defender-for-cloud/copilot-security-in-defender-for-cloud`](https://learn.microsoft.com/azure/defender-for-cloud/copilot-security-in-defender-for-cloud)）。

服务菜单上还有另一个值得关注的选项叫做 `SQL Server IaaS 代理扩展设置`。这可以帮助你修复扩展的任何问题，并设置自动升级。

除了使用门户，你还可以使用 `az sql vm` CLI。这允许你注册你的 VM 或更改 SQL Server 属性。你可以在 [`https://learn.microsoft.com/cli/azure/sql/vm`](https://learn.microsoft.com/cli/azure/sql/vm) 阅读所有这些选项。

