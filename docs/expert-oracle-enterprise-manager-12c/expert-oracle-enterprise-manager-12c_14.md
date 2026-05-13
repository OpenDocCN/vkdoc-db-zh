# 第 2 章 安装与代理部署

![image](img/frontdot.jpg)

作者：Gokhan Atil

在本章中，您将学习如何创建仓库数据库并安装 Oracle Enterprise Manager Cloud Control 12c。如果您有任何安装 Enterprise Manager Grid Control（Cloud Control 的前身）的经验，您会发现 EM12c 配备了一个更智能的安装向导，因此安装过程要容易得多。

EM12c 由三个组件组成：Oracle Management Service（`OMS`）、Oracle Management Agents（管理代理）和 Oracle Management Repository（管理仓库）。

本章的安装和代理部署演示使用三台服务器：

*   `cloudcontrol12.testdomain.com`：Oracle Management Service 将安装到此服务器。
*   `repositorydb.testdomain.com`：此数据库服务器将托管管理仓库。
*   `target.testdomain.com`：管理代理将部署到此目标服务器。

所有这些服务器都运行 Oracle Linux 5.8（64 位）以及 GNOME 桌面环境和 X Window 系统。

虽然您可以将管理仓库和`OMS`安装在同一服务器上，但我们更倾向于将管理仓库安装在单独的服务器上。如果您计划将两者安装在同一服务器上，则应满足两次安装的要求。

我们建议您使用域名系统（`DNS`）服务器来解析服务器的主机名。如果您没有`DNS`服务器，则需要在 Oracle Management Server 上的`/etc/hosts`文件中输入所有目标的主机名和相应的 IP 地址。您还需要在所有目标服务器（您将在其上部署管理代理）上的`/etc/hosts`文件中输入`OMS`的主机名/IP 地址。



## 硬件要求与安装准备

使用完全限定主机名也很重要。完全限定域名（FQDN）是服务器的完整域名。它包含主机名和域名，以明确其在 DNS 层次结构中的确切位置。例如，`cloudcontrol12.testdomain.com`就是一个完全限定的主机名。至少应确保您的 Oracle Management Server 拥有一个完全限定的主机名。

Oracle Enterprise Manager Cloud Control 12c 可以从 My Oracle Support 为您的服务器获取最新的补丁信息，并可以为事件创建服务请求。因此，我们建议您使 Oracle Management Service 能够访问 My Oracle Support 网站。如果不希望服务器直接访问互联网，可以设置一个代理服务器，使 OMS 能够访问 My Oracle Support 网站。

