# 16. 故障排除与常见问题

## 摘要

按照本章开头描述的步骤，您应该会被重定向到 OAM 认证页面。使用一组存在于身份存储中的有效凭据登录。成功认证后，您应该会被重定向到您的 Oracle EBS 主页。

需要注意的是，只有在 EBS 中具有 `Local Login Allowed` 配置文件的用户才能使用 EBS 本地登录。这对管理用户和故障排除很有帮助。

在本章中，为您介绍了集成 Oracle EBS 和 OAM 时存在的操作顺序和数据流。需要注意的是，要集成 OID 和 EBS，需要 OAM，因为此集成利用了 EBS AccessGate 部署。本章提供了在支持单个 EBS 实例的单个节点上实例化单个 AccessGate 所需的步骤。这些步骤可以修改以支持集群环境，如果您的某个 AccessGate 托管服务器发生故障，则可提供更高的可用性和容错能力。多个 AccessGate 部署也可用于支持多个 EBS 实例。

您已遵循 Oracle 的文档以确保您的环境符合认证环境。您的操作系统 (OS) 是最新的，并且系统要求已满足。您运行安装软件，一切看起来都很顺利。突然，安装程序似乎挂起了。用 Douglas Adams 的话说，“不要惊慌”。

考虑到必须安装、打补丁、配置然后集成以成功实施 Oracle Identity and Access Management Suite 的组件数量，您有可能会遇到问题。这些问题可能在安装和初始配置期间出现；也可能在您尝试将组件彼此集成或与其他应用程序集成时浮现。无论您何时遇到问题，它们都可能令人沮丧，有时可能没有明显的解决方案。很多时候，可能遗漏了一个补丁或配置文件中存在一个简单的排版错误。有些问题很容易解决，而其他问题可能需要大量的研究。本章旨在提供一些常见问题和解决方案的列表，以及一些实施者可能需要解决的罕见问题。这些内容来源于多次安装的汇编和同事们的笔记。

### 安装问题

本书的开头向您介绍了构成 Oracle Identity and Access Management Suite 的各种组件。随后，介绍了使环境工作所需的每个主要组件的安装过程。尽管安装看起来相当简单，但遇到一些问题并不罕见。如果您在过程中走的还不太远，重新启动安装软件可能很容易。然而，在这个阶段，您很可能会再次遇到同样的问题，除非您已采取措施解决根本问题。

在安装阶段，您正在放置应用程序二进制文件。Oracle 通用安装程序将检查您的操作系统以确保其满足最低规格。最常见的问题是在此预检查期间发现的。理想情况下，您已经检查过以确保安装了正确的操作系统软件包。然而，列表很长，如果缺少任何软件包，Oracle 通用安装程序会指出它们供您解决。如果您有 `root` 或 `sudo` 权限，这些问题可能很容易解决。根据您选择的操作系统，您可以安装它们然后继续。

使用基于 UNIX 的系统，您可以使用 `yum` 安装缺少的软件包，或下载缺少的软件包 `rpm` 文件并进行安装。请注意，这需要 `sudo` 或 `root` 权限。从 `rpm` 文件安装缺少的软件包可能是一个漫长的过程，因为您需要找到该软件包以及所有必需的软件包才能正确安装它们。大多数情况下，像 `yum` 这样的软件包安装工具会执行所有必需软件包的下载和安装。缺少的操作系统软件包是一些最容易诊断和解决的问题。请参见表 16-1 以获取本书涵盖的每个主要组件所需软件包的列表。根据您的环境，这些软件包可能需要安装在所有服务器上，或仅安装在特定服务器上，具体取决于每个服务器上安装的组件。

**表 16-1.** Oracle Identity Management 操作系统软件包要求

| 身份管理组件 | 所需组件 |
| :--- | :--- |
| Oracle Internet Directory 11.1.1.9 | `binutils-2.20.51.0.2-5.28.el6`<br>`compat-libcap1-1.10-1`<br>`compat-libstdc++-33-3.2.3-69.el6 for x86_64`<br>`compat-libstdc++-33-3.2.3-69.el6 for i686`<br>`gcc-4.4.4-13.el6`<br>`gcc-c++-4.4.4-13.el6`<br>`glibc-2.12-1.7.el6 for x86_64`<br>`glibc-2.12-1.7.el6 for i686`<br>`glibc-devel-2.12-1.7.el6 for i686`<br>`libaio-0.3.107-10.el6`<br>`libaio-devel-0.3.107-10.el6`<br>`libgcc-4.4.4-13.el6`<br>`libstdc++-4.4.4-13.el6 for x86_64`<br>`libstdc++-4.4.4-13.el6 for i686`<br>`libstdc++-devel-4.4.4-13.el6`<br>`libXext for i686`<br>`libXtst for i686`<br>`libXext for x86_64`<br>`libXtst for x86_64`<br>`openmotif-2.2.3 for x86_64`<br>`openmotif22-2.2.3 for x86_64`<br>`redhat-lsb-core-4.0-7.el6 for x86_64`<br>`sysstat-9.0.4-11.el6`<br>`xorg-x11-utils*`<br>`xorg-x11-apps*`<br>`xorg-x11-xinit*`<br>`xorg-x11-server-Xorg*`<br>`xterm` |
| Oracle Access Manager 11.1.2.3 | `binutils-2.20.51.0.2-5.28.el6`<br>`compat-libcap1-1.10-1`<br>`compat-libstdc++-33-3.2.3-69.el6 for x86_64`<br>`compat-libstdc++-33-3.2.3-69.el6 for i686`<br>`gcc-4.4.4-13.el6`<br>`gcc-c++-4.4.4-13.el6`<br>`glibc-2.12-1.7.el6 for x86_64`<br>`glibc-2.12-1.7.el6 for i686`<br>`glibc-devel-2.12-1.7.el6 for i686`<br>`libaio-0.3.107-10.el6`<br>`libaio-devel-0.3.107-10.el6`<br>`libgcc-4.4.4-13.el6`<br>`libstdc++-4.4.4-13.el6 for x86_64`<br>`libstdc++-4.4.4-13.el6 for i686`<br>`libstdc++-devel-4.4.4-13.el6`<br>`libXext for i686`<br>`libXtst for i686`<br>`libXext for x86_64`<br>`libXtst for x86_64`<br>`openmotif-2.2.3 for x86_64`<br>`openmotif22-2.2.3 for x86_64`<br>`redhat-lsb-core-4.0-7.el6 for x86_64`<br>`sysstat-9.0.4-11.el6`<br>`xorg-x11-utils*`<br>`xorg-x11-apps*`<br>`xorg-x11-xinit*`<br>`xorg-x11-server-Xorg*`<br>`xterm`<br>`pdksh-5.2.14` |
| Oracle Identity Manager 11.1.2.3 | `binutils-2.20.51.0.2-5.28.el6`<br>`compat-libcap1-1.10-1`<br>`compat-libstdc++-33-3.2.3-69.el6 for x86_64`<br>`compat-libstdc++-33-3.2.3-69.el6 for i686`<br>`gcc-4.4.4-13.el6`<br>`gcc-c++-4.4.4-13.el6`<br>`glibc-2.12-1.7.el6 for x86_64`<br>`glibc-2.12-1.7.el6 for i686`<br>`glibc-devel-2.12-1.7.el6 for i686`<br>`libaio-0.3.107-10.el6`<br>`libaio-devel-0.3.107-10.el6`<br>`libgcc-4.4.4-13.el6`<br>`libstdc++-4.4.4-13.el6 for x86_64`<br>`libstdc++-4.4.4-13.el6 for i686`<br>`libstdc++-devel-4.4.4-13.el6`<br>`libXext for i686`<br>`libXtst for i686`<br>`libXext for x86_64`<br>`libXtst for x86_64`<br>`openmotif-2.2.3 for x86_64`<br>`openmotif22-2.2.3 for x86_64`<br>`redhat-lsb-core-4.0-7.el6 for x86_64`<br>`sysstat-9.0.4-11.el6`<br>`xorg-x11-utils*`<br>`xorg-x11-apps*`<br>`xorg-x11-xinit*`<br>`xorg-x11-server-Xorg*`<br>`xterm`<br>`pdksh-5.2.14` |
| Oracle HTTP Server/WebGate 11g | `binutils-2.20.51.0.2-5.28.el6`<br>`compat-libcap1-1.10-1`<br>`compat-libstdc++-33-3.2.3-69.el6 for x86_64`<br>`compat-libstdc++-33-3.2.3-69.el6 for i686`<br>`gcc-4.4.4-13.el6`<br>`gcc-c++-4.4.4-13.el6`<br>`glibc-2.12-1.7.el6 for x86_64`<br>`glibc-2.12-1.7.el6 for i686`<br>`glibc-devel-2.12-1.7.el6 for i686`<br>`libaio-0.3.107-10.el6`<br>`libaio-devel-0.3.107-10.el6`<br>`libgcc-4.4.4-13.el6`<br>`libstdc++-4.4.4-13.el6 for x86_64`<br>`libstdc++-4.4.4-13.el6 for i686`<br>`libstdc++-devel-4.4.4-13.el6`<br>`libXext for i686`<br>`libXtst for i686`<br>`libXext for x86_64`<br>`libXtst for x86_64`<br>`openmotif-2.2.3 for x86_64`<br>`openmotif22-2.2.3 for x86_64`<br>`redhat-lsb-core-4.0-7.el6 for x86_64` |


### 常见配置问题

以下是安装或配置过程中可能遇到的软件包依赖：

*   `sysstat-9.0.4-11.el6`
*   `xorg-x11-utils*`
*   `xorg-x11-apps*`
*   `xorg-x11-xinit*`
*   `xorg-x11-server-Xorg*`
*   `xterm`
*   `pdksh-5.2.14`

## 操作系统环境配置

在安装甚至后续的配置或运行时过程中，可能遇到的其他问题与操作系统环境及配置相关。以下信息将概述如何为成功安装设置环境。

需要设置以下内核参数：

```
kernel.sem  256  32000  100  143
kernel.shmmax 10737418240
```

要设置这些参数，请编辑位于 `/etc` 目录下的 `sysctl.conf` 文件。

```
[root@clouddemolab home]# vi /etc/sysctl.conf
```

在该文件的相应部分添加或编辑以下行：

```
# Controls the maximum number of shared memory segments, in pages
kernel.shmall = 4294967296
kernel.sem = 256 32000 100 142
kernel.shmmax = 10737418240
```

在 `sysctl.conf` 文件中设置这些值后，必须使用以下命令激活并验证新值是否生效：

```
[root@clouddemolab home]# /sbin/sysctl –p
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
kernel.sem = 256 32000 100 142
kernel.shmmax = 10737418240
```

必须将打开的文件限制设置为 4096 以支持实例。为此，请编辑 `limits.conf` 文件。

```
[root@clouddemolab home]# vi /etc/security/limits.conf
```

如果环境要安装在 Oracle Linux 或 RedHat Linux 上，您还必须同时编辑 `/etc/security/limits.d/90-nproc.conf` 文件。如果遗漏了此步骤，该文件中的值可能会覆盖 `limits.conf` 文件中的值。

在这两个文件中，确保添加或编辑了以下行：

```
* soft nofile 4096
* hard nofile 65536
* soft nproc 2047
* hard nproc 16384
```

编辑此文件后，必须重新启动服务器以确保所有更改生效。

大多数安装问题可以通过确保在启动安装程序之前满足上述先决条件来解决或预防。

