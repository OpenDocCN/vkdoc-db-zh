# Microsoft SQL Server 2022 云连接功能

## Microsoft Purview

Microsoft Purview 具备许多功能，但与 SQL Server 2022 集成的具体新功能是 `policy management`（策略管理）。您可以使用 Purview 发布一个将推送到 SQL Server 2022 的策略，以支持身份验证和访问。Purview 策略管理与 SQL Server 2022 要求您已为 SQL Server 配置了 Azure Active Directory (AAD) 身份验证，因为策略将基于 AAD 帐户。Purview 策略管理还需要 Azure 扩展 for SQL Server。

## Microsoft Defender for SQL

Microsoft Defender for SQL 是 Microsoft Defender for Cloud 家族的一员，支持对 SQL Server 2022 进行漏洞评估和高级威胁防护 (ATP)。Microsoft Defender for SQL 适用于多种 SQL 技术，并支持早期版本的 SQL Server。适用于本地 SQL Server 的 Microsoft Defender for SQL 需要 Azure Arc Agent 和用于监视的扩展。本章不会详细介绍此服务。您可以在 [`https://docs.microsoft.com/azure/defender-for-cloud/defender-for-sql-usage`](https://docs.microsoft.com/azure/defender-for-cloud/defender-for-sql-usage) 阅读更多关于 Microsoft Defender for SQL 的信息。

## Azure Arc Agents and Azure Extension for SQL Server

`Azure Arc Agent` 是在您选择在 SQL Server 安装期间或通过脚本方法连接到 Azure 时安装的，用于连接 Microsoft Purview、启用 Azure Active Directory 身份验证 (AAD) 并支持 Microsoft Defender for SQL。Azure Arc Agent 支持一个用于特定功能的扩展框架。我们构建了一个名为 `Azure extension for SQL Server` 的扩展。此扩展与 Azure 通信，将信息存储在注册表（对于 Linux 是 `mssql.conf`）中，数据库引擎已增强以支持 AAD 和 Purview。Microsoft Defender for SQL 使用另一个名为 Monitoring Agent 的扩展。您可以在 [`https://docs.microsoft.com/azure/azure-arc/servers/agent-overview`](https://docs.microsoft.com/azure/azure-arc/servers/agent-overview) 阅读有关 Azure Arc Agent 架构的详细信息。Azure Arc Agent 和扩展是为本地 SQL Server 设计的。Azure 虚拟机上的 SQL Server 有类似的概念，但使用不同的架构，包括 SQL Server IaaS Agent Extension ([`https://docs.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-iaas-agent-extension-automate-management`](https://docs.microsoft.com/azure/azure-sql/virtual-machines/windows/sql-server-iaas-agent-extension-automate-management))。在本书撰写时，Microsoft Purview 和 Azure Active Directory 身份验证尚未得到 IaaS Agent Extension 的支持。然而，我们计划启用此功能，并且我们可以在不同于发布 SQL Server 主要版本的时间框架内增强该扩展。

本章的其余部分将深入介绍这些云连接功能中的每一项。您可以阅读所有部分，或直接跳转到您想了解的特定云服务。

