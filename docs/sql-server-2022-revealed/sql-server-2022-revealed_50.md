# 在 Linux 上部署 SQL Server 2022

在 Linux 上部署 SQL Server 2022 的过程与 SQL Server 2017 和 2019 相同。您选择一个受支持的 Linux 发行版，然后使用该 Linux 发行版自带的`包管理器`来安装 SQL Server。这是因为我们发布了适用于 RHEL、Ubuntu 和 SLES 的 Linux 软件包。我们提供了如何下载`仓库文件`的说明，该文件包含了可在哪里找到我们软件包的网络源。

**提示**

您知道可以在 `https://packages.microsoft.com` 浏览我们所有的软件包吗？

例如，在 RHEL 上，有两个简单的命令来安装核心 SQL Server 引擎软件包：

```bash
sudo curl -o /etc/yum.repos.d/mssql-server.repo https://packages.microsoft.com/config/rhel/8/mssql-server-preview.repo
sudo yum install -y mssql-server
```

**注意**

当 SQL Server 正式发布后，软件包名称将从 `mssql-server-preview.repo` 更改为其他名称，如 `mssql-server-2022.repo`。您可以在 `https://docs.microsoft.com/sql/linux/quickstart-install-connect-red-hat` 保持信息更新。

这是一个非常简单的安装过程，核心引擎软件包几乎包含了基础功能所需的一切，所有内容都内置在这些步骤中。

在 Linux 上部署 SQL Server 有一些独特之处：

*   无论使用何种发行版，您都必须运行以下命令来完成设置：

    ```bash
    sudo /opt/mssql/bin/mssql-conf setup
    ```

    `mssql-conf` 是一个可以在 SQL 实例外部执行配置操作的脚本。`setup`（设置）选项用于建立管理员密码、选择版本以及其他选项以完成设置过程。

*   与 Windows 上的 SQL Server 不同，其他功能可能需要您安装额外的软件包，例如工具、全文搜索、SSIS、Java 扩展和 ML 服务、Polybase 以及 HA（用于可用性组）。您可以在以下网站的“软件包详细信息”部分获取所有这些软件包的列表：`https://docs.microsoft.com/sql/linux/sql-server-linux-release-notes-2022#supported-platforms`。

*   Linux 的一个有趣之处在于，每个累积更新（CU）和常规分发版本（GDR）都是独立的软件包，因此您可以直接安装 CU，而无需先安装 RTM，然后再应用 CU 软件包（这是 Windows 所必需的）。

## 我还应该了解什么？

*   Linux 上的 SQL Server 在 Linux 上作为服务安装。因此，您可以随时使用以下命令查看 SQL Server 的状态（运行中、已停止等）：

    ```bash
    systemctl status mssql-server
    ```

*   数据、日志、日志文件和备份的默认目录基于 `/var/opt/mssql`。此目录必须始终存在，但您可以更改文件（如数据库和备份）的默认位置。

*   Linux 上的 SQL Server 允许您执行卸载、升级、离线安装和自动安装（使用环境变量）。

*   Linux 上的 SQL Server 许可与 Windows 上相同。您可以在任一操作系统上使用现有的 SQL Server 许可证。

## 与 RHEL 配合使用 Ansible

Red Hat Enterprise Linux (RHEL) 独有的一个强大功能是 `Ansible`。Ansible 为`自动化`提供了一个出色的平台。因此，通过称为 `playbook` 的概念，它可以成为自动化为多台计算机或虚拟机安装 Linux 上 SQL Server 的解决方案。

我们与 Red Hat 合作，为在 Linux 上使用 SQL Server 的 playbook 包含了一个特殊的角色。

您可以通过以下资源了解更多关于如何实现这一点的信息：

*   `https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/administration_and_configuration_tasks_using_system_roles_in_rhel/assembly_configuring-microsoft-sql-server-using-microsoft-sql-server-ansible-role_assembly_updating-packages-to-enable-automation-for-the-rhel-system-roles`

*   `https://docs.microsoft.com/sql/linux/sql-server-linux-deploy-ansible`

