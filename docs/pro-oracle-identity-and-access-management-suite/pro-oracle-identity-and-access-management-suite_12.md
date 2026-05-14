# 7. Identity Manager 安装

在前两章中，提供了安装和配置 Oracle Internet Directory (OID) 和 Oracle Access Manager (OAM) 的说明。这两个组件奠定了身份存储和单点登录 (SSO) 的基础。Identity Manager 为环境添加了额外的元素，例如用户自助服务和治理，从而完成了端到端的身份生命周期管理实施。本章重点介绍 Oracle Identity Manager (OIM) 的安装和初始域配置。

## 预安装任务

#### 操作系统用户

对于大多数 Oracle 应用程序安装，应创建操作系统 (OS) 用户和组来执行安装和配置任务。创建 OS 组将允许其他 OS 用户执行与管理应用程序环境相关的某些任务。在 Linux 环境中安装 Oracle 应用程序时，最常见的 OS 用户和组是 `oracle` 用户以及 `oinstall` 或 `dba` 组。

要创建必要的 `oinstall` 和 `dba` 组，请以 root 目录执行以下命令：

```
[root@clouddemolab home]# groupadd oinstall
[root@clouddemolab home]# groupadd dba
```

创建组后，创建 `oracle` 用户：

```
[root@clouddemolab home]# useradd  -g oinstall -G dba oracle
```

**注意**
`-g` 表示用户应添加到的主要组。`-G` 表示任何次要组。

要设置用户密码，请以 root 用户身份使用以下命令。

```
[root@clouddemolab home]# passwd oracle
```

#### 操作系统配置

在安装 Oracle Fusion Middleware 基础架构和 Oracle Identity Management 软件之前，确保 OS 满足最低要求和配置非常重要。以下列出了所需的内核参数、软件包以及文件更改。

需要设置以下内核参数：

```
kernel.sem  256  32000  100  143
kernel.shmmax 10737418240
```

要设置这些参数，请编辑位于 `/etc` 目录下的 `sysctl.conf` 文件。

```
[root@clouddemolab home]# vi /etc/sysctl.conf
```

在该文件的此部分添加或编辑以下行：

```
# Controls the maximum number of shared memory segments, in pages
kernel.shmall = 4294967296
kernel.sem = 256 32000 100 142
kernel.shmmax = 10737418240
```

在 `sysctl.conf` 文件中设置这些值后，必须使用以下命令激活并验证是否显示新值：

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

打开文件限制必须设置为 4096 以支持实例。为此，请编辑 `limits.conf` 文件。

```
[root@clouddemolab home]# vi /etc/security/limits.conf
```

如果环境要安装在 Oracle Linux 或 RedHat Linux 上，还必须编辑 `/etc/security/limits.d/90-nproc.conf`。如果忽略了这一点，此文件中的值可能会覆盖 `limits.conf` 文件中的值。

确保在这两个文件中添加或编辑了以下行：

```
* soft nofile 4096
* hard nofile 65536
* soft nproc 2047
* hard nproc 16384
```

编辑此文件后，必须重新启动服务器以确保所有更改生效。


## 操作系统包

每个 Oracle 应用程序都有自己的一套必需软件包。根据您所使用的 Linux 版本，安装过程可能有所不同。在下面的列表中，请注意，有些软件包需要在 64 位操作系统上同时安装 32 位和 64 位版本。如果这些软件包未安装，安装将无法正常完成。Oracle 安装程序会检查这些包，并在安装过程中显示错误。

```
binutils-2.20.51.0.2-5.28.el6
compat-libcap1-1.10-1
compat-libstdc++-33-3.2.3-69.el6 for x86_64
compat-libstdc++-33-3.2.3-69.el6 for i686
gcc-4.4.4-13.el6 gcc-c++-4.4.4-13.el6
glibc-2.12-1.7.el6 for x86_64
glibc-2.12-1.7.el6 for i686
glibc-devel-2.12-1.7.el6 for i686
libaio-0.3.107-10.el6
libaio-devel-0.3.107-10.el6
libgcc-4.4.4-13.el6
libstdc++-4.4.4-13.el6 for x86_64
libstdc++-4.4.4-13.el6 for i686
libstdc++-devel-4.4.4-13.el6
libXext for i686
libXtst for i686
libXext for x86_64
libXtst for x86_64
openmotif-2.2.3 for x86_64
openmotif22-2.2.3 for x86_64
redhat-lsb-core-4.0-7.el6 for x86_64
sysstat-9.0.4-11.el6
xorg-x11-utils*
xorg-x11-apps*
xorg-x11-xinit*
xorg-x11-server-Xorg*
xterm
pdksh-5.2.14
```

至此，在安装流程中，操作系统应已完全准备好，可以继续进行安装。在安装软件之前执行这些操作，将确保安装过程顺利无误。在多数情况下，如果遗漏了任何内容，安装程序会提供详细的错误信息。如果在安装过程中出现错误，请停止安装，并在继续之前修复所有问题。

#### 数据库准备

与 OID 类似，OIM 的安装也需要创建特定的数据库对象。这包括在多个数据库模式中创建的一系列表、视图和程序包。这是通过使用**仓库创建实用程序** (Repository Creation Utility，RCU) 完成的。虽然可以在 OID 数据库内创建这些数据库对象，但通常建议将 Oracle 身份和访问管理仓库创建在单独的实例中。这样可以简化数据库管理任务和未来的维护工作。为防止安装出现问题，确保所使用的 RCU 版本与要安装的 Fusion Middleware 产品版本相匹配至关重要。在域配置期间发现不匹配将导致进程无法继续。解压下载的文件，并运行 `<RCU_HOME>/rcu/bin` 目录中的 `rcu.sh`。您将首先看到如图 7-1 所示的“创建仓库”屏幕。

![A352855_1_En_7_Fig1_HTML.jpg](img/A352855_1_En_7_Fig1_HTML.jpg)

图 7-1. 创建或删除方案

由于这是首次安装 OIM，请在此屏幕上选择“创建”选项。这将启动 RCU 的创建模式。

选择创建方案后，您必须在如图 7-2 所示的屏幕上输入目标系统的数据库连接详细信息。您可以选择在与 OID 相同的数据库中创建身份管理器方案。然而，为了保持数据库管理更简单，身份管理器方案通常安装在不同的数据库中。此实例可以与 OAM 相同。

![A352855_1_En_7_Fig2_HTML.jpg](img/A352855_1_En_7_Fig2_HTML.jpg)

图 7-2. “数据库连接详细信息”屏幕

**注意：** 在创建 OIM 方案之前，必须运行 `$ORACLE_HOME/rdbms/admin/xaview.sql` 来启用 XA 事务视图和同义词。

RCU 验证连接详细信息后，会提示您选择要在新仓库中安装的组件。此屏幕如图 7-3 所示。

![A352855_1_En_7_Fig3_HTML.jpg](img/A352855_1_En_7_Fig3_HTML.jpg)

图 7-3. “选择组件”屏幕

在此 RCU 步骤中，您必须选择要安装的组件。请注意，在选择 OIM 组件时，其他必需项将被预选。**不要取消选择这些项**，因为它们将在域配置步骤中进行验证。然后，RCU 将验证数据库是否满足所选组件的先决条件，如图 7-4 所示。

![A352855_1_En_7_Fig4_HTML.jpg](img/A352855_1_En_7_Fig4_HTML.jpg)

图 7-4. 检查先决条件

每个 Fusion Middleware 组件都有数据库要求，例如最大连接数或打开的进程数。RCU 将在创建数据库方案和对象之前检查这些先决条件。

如果数据库满足最低要求，下一步是输入新数据库方案要使用的密码，如图 7-5 所示。

![A352855_1_En_7_Fig5_HTML.jpg](img/A352855_1_En_7_Fig5_HTML.jpg)

图 7-5. “方案密码”屏幕

在此步骤中，指定您希望使用的密码值。您可以选择为所有方案使用相同的密码，也可以为每个方案使用不同的密码。请根据安全要求和易管理性做出决定。

图 7-6 显示了表空间复查屏幕，其中显示了将为选定的身份管理组件创建的新数据库表空间。您可以单击“下一步”继续，或选择自定义新表空间。

![A352855_1_En_7_Fig6_HTML.jpg](img/A352855_1_En_7_Fig6_HTML.jpg)

图 7-6. 表空间列表

在实际创建表空间之前，RCU 将向您显示它将执行的操作的摘要。图 7-7 显示了“摘要”屏幕。您应记录这些信息，以备将来与 DBA 讨论与数据库相关的运行时问题时参考。

![A352855_1_En_7_Fig7_HTML.jpg](img/A352855_1_En_7_Fig7_HTML.jpg)

图 7-7. “创建摘要”屏幕

所有数据库对象创建完成后，系统将提供“完成摘要”屏幕，如图 7-8 所示。单击“关闭”完成该过程。

![A352855_1_En_7_Fig8_HTML.jpg](img/A352855_1_En_7_Fig8_HTML.jpg)

图 7-8. “完成摘要”屏幕

至此，OIM 的仓库创建过程已完成。身份管理器及其所需组件的必要数据库方案和对象已安装在目标数据库中。现在可以继续安装过程。

### 身份管理器软件安装

在上一节中，您了解了创建 OIM 数据库方案和对象的步骤。以下部分将讨论身份管理器软件的安装。此过程会创建必要的文件系统结构，并部署 Fusion Middleware 产品所需的二进制文件。

OIM 必须安装在 WLS 主目录中。在第 6 章中，介绍了在单独的 WLS 主目录中安装 OAM 的步骤。您可以选择将身份管理器软件安装在与访问管理器相同的主目录中，也可以创建一个完全独立的主目录专门用于它。通常将访问管理器和身份管理器分离在网络的不同层级或不同的物理主机上。如果您的环境需要这样做，请遵循第 4 章中的 WLS 安装步骤。

### 面向服务的架构安装

OIM 需要 Oracle 面向服务的架构（SOA）才能正常运行。此安装独立于 OIM 进程，但可以安装在同一个 WebLogic 主目录中。SOA 安装完成后，您可以同时安装 OIM 并为这两个产品配置域。这是针对这两个产品的推荐流程。

与许多其他 Fusion Middleware 产品类似，SOA 安装工具是通过 `runInstaller` 命令来启动 Oracle 通用安装程序的。该工具位于安装介质的 Disk 1 上。运行该工具时，您必须指明 Java 运行时环境（JRE）的位置。具体操作如图 7-9 所示。

![A352855_1_En_7_Fig9_HTML.jpg](img/A352855_1_En_7_Fig9_HTML.jpg)
图 7-9. 启动通用安装程序

```
runInstaller -jreLoc /home/oracle/jdk1.6.0_45/jre
```

OIM 需要 SOA 11.1.1.9。将使用此工具安装此版本。启动屏幕会显示与计划安装相关的信息，并提醒运行 RCU。与通用安装程序的其他版本类似，Oracle 会检查操作系统以确保其满足最低要求。已完成的先决条件检查如图 7-10 所示。

![A352855_1_En_7_Fig10_HTML.jpg](img/A352855_1_En_7_Fig10_HTML.jpg)
图 7-10. 先决条件检查屏幕

与 OAM 一样，SOA 也有自己的一套必需的操作系统软件包、内核参数、内存和存储空间要求。在继续安装之前，必须满足这些要求。尽管其中许多要求与访问管理器（Access Manager）的安装相同，但查阅本章开头部分查看这些要求仍然很重要。

### 注意

先决条件失败项将在通用安装程序屏幕上以红色 X 显示。您可以以 root 身份登录并打开终端窗口来纠正任何问题，然后重试先决条件检查，直到所有问题都得到解决。

在下一步中，您必须为此安装选择中间件主目录（Middleware Home）位置。由于此环境为每个访问管理器和身份管理器（Identity Manager）分别配备了独立的 WLS，因此将 SOA 安装在正确的中间件主目录中至关重要。在此情况下，身份管理器将安装在 `IDMMiddleware` 目录中。更多详情请参见图 7-11。

![A352855_1_En_7_Fig11_HTML.jpg](img/A352855_1_En_7_Fig11_HTML.jpg)
图 7-11. 指定安装位置屏幕

Oracle SOA 可以安装在 WLS 或 WebSphere 服务器上。本书侧重于 WLS 安装类型。如图 7-12 所示，选择 WebLogic 服务器，然后继续下一步。

![A352855_1_En_7_Fig12_HTML.jpg](img/A352855_1_En_7_Fig12_HTML.jpg)
图 7-12. 选择应用服务器类型

图 7-13 所示的安装摘要屏幕，显示了在安装程序各屏幕中所做选择的摘要。确认选择后，点击“安装”开始安装。

![A352855_1_En_7_Fig13_HTML.jpg](img/A352855_1_En_7_Fig13_HTML.jpg)
图 7-13. 安装摘要屏幕

实际安装过程可能需要大约 10 分钟。图 7-14 所示的进度屏幕会让您了解当前状态。如果进度似乎暂停了一会儿，请不要惊慌。

![A352855_1_En_7_Fig14_HTML.jpg](img/A352855_1_En_7_Fig14_HTML.jpg)
图 7-14. 安装进度屏幕

安装程序完成必要文件的复制和文件系统的创建后，将显示图 7-15 所示的安装完成屏幕。此屏幕包含与您的新环境直接相关的详细信息。记下内容以备将来参考可能很有用。此时，您可以点击“完成”并退出安装程序。

![A352855_1_En_7_Fig15_HTML.jpg](img/A352855_1_En_7_Fig15_HTML.jpg)
图 7-15. 安装完成屏幕

至此，所需的 Oracle SOA 套件实例已安装完成。在配置 OIM 域时，配置向导将设置 SOA 的必要组件。如您所记得的，此 SOA 实例与身份管理器安装在同一个中间件主目录中。下一节将介绍 OIM 安装。

### 身份管理器安装

WLS 和 SOA 软件已经安装完毕。现在是安装身份管理器软件的时候了。同样，此过程是使用安装介质 Disk 1 上的 `runInstaller` 脚本启动的。与 SOA 安装一样，您必须指明 Java 运行时环境（JRE）的位置，安装程序才能正常运行。图 7-16 显示了通用安装程序。

![A352855_1_En_7_Fig16_HTML.jpg](img/A352855_1_En_7_Fig16_HTML.jpg)
图 7-16. 安装欢迎屏幕

```
./runInstaller.sh -jreLoc /home/oracle/jdk1.6.0_45/jre
```

第一个屏幕显示有关要安装的软件的重要信息。确保显示的版本符合您的要求以及先前运行的 RCU 版本。

与 OAM 一样，OIM 也有自己的一套必需的操作系统软件包、内核参数、内存和存储空间要求。在继续安装之前，必须满足这些要求。如图 7-17 所示，安装程序在允许进程继续之前会检查这些内容。尽管其中许多要求与 OAM 安装相同，但查阅本章开头部分查看这些要求仍然很重要。

![A352855_1_En_7_Fig17_HTML.jpg](img/A352855_1_En_7_Fig17_HTML.jpg)
图 7-17. 先决条件检查屏幕

在“指定安装位置”屏幕上，您必须选择一个中间件主目录来存放安装文件。图 7-18 提供了关于此入口点的详细信息。此位置必须是已安装 WLS 的目录。由于此物理主机托管了两个 WLS 实例，请确保选择当前未用于 OAM 的位置。选择的主目录应与您上一步为 SOA 选择的位置相同。

![A352855_1_En_7_Fig18_HTML.jpg](img/A352855_1_En_7_Fig18_HTML.jpg)
图 7-18. 指定安装位置屏幕

此时，通用安装程序将开始将文件复制到新位置。图 7-19 所示的进度屏幕将显示当前操作和进度。

![A352855_1_En_7_Fig19_HTML.jpg](img/A352855_1_En_7_Fig19_HTML.jpg)
图 7-19. 安装进度屏幕

软件文件的实际安装通常在大约 10 分钟内完成。在此期间，您可以在进度屏幕上看到实际操作，该屏幕还会显示安装日志文件的位置。您可以监控此日志以查看是否有任何错误。完成后，您可以关闭安装程序，因为所有安装操作都已完成。在接下来的章节中，您将配置 OIM 域。


### 配置身份管理器域

在安装必要的软件组件后，您可以继续配置 WebLogic 域以支持 OIM。此过程通过运行位于 `` `IDM_HOME/common/bin` `` 目录下的 `` `config` `` 脚本启动。需要注意的是，此时您仅配置 WebLogic 域。OIM 此时尚未准备好运行。

为避免本章后续内容产生混淆，将使用以下约定。`` `MIDDLEWARE_HOME` `` 是安装 WLS 的基础目录。`` `IDM_HOME` `` 是 WLS 安装中安装 OIM 的目录。这通常是一个名为 `` `Oracle_IDM1` `` 的目录。图 7-20 显示了配置向导。

![A352855_1_En_7_Fig20_HTML.jpg](img/A352855_1_En_7_Fig20_HTML.jpg)

图 7-20. 创建新的 WebLogic 域

**注意**
在 Middleware Home 子目录中可以找到多个 `` `config.sh` `` 脚本实例。运行位于 `` `<MIDDLEWARE_HOME/Oracle_IDM1/common/bin.` `` 中的正确版本至关重要。

由于这是首次在 WLS 环境中创建域，请选择 `` `Create a New WebLogic Domain` ``（创建新的 WebLogic 域）。

在此过程的下一步，选择 OIM 组件。如图 7-21 所示，此列表基于 WebLogic Home 目录中找到的软件。您会注意到 Oracle SOA Suite 已被自动选中。如果在 WLS Home 中未找到 SOA Suite，系统将显示错误提示。在继续之前，请确保已安装任何缺失的软件。

![A352855_1_En_7_Fig21_HTML.jpg](img/A352855_1_En_7_Fig21_HTML.jpg)

图 7-21. 选择域组件

在配置过程中，系统会提示您输入希望创建的域的名称。您可以保留默认名称 `` `"base_domain"` ``，也可以将其命名为在您的环境中更有意义的名称。示例参见图 7-22。在某些情况下，可能希望将相关文件存放在 Middleware Home 目录之外。这通常在需要跨物理主机共享存储的集群环境中进行。

![A352855_1_En_7_Fig22_HTML.jpg](img/A352855_1_En_7_Fig22_HTML.jpg)

图 7-22. 输入域名和位置

`` `weblogic` `` 的用户密码应设置为您环境中的标准密码。此密码将用于启动和停止受管服务器，以及登录 WebLogic 管理控制台和 Fusion Middleware Control。图 7-23 显示了密码配置屏幕的样子。

![A352855_1_En_7_Fig23_HTML.jpg](img/A352855_1_En_7_Fig23_HTML.jpg)

图 7-23. 管理员密码

`` `Startup Mode` ``（启动模式）决定了受管服务器的启动方式。在 `` `Development Mode` ``（开发模式）下，受管服务器可以从命令行启动和停止，无需密码。此外，可以在 WebLogic 控制台中进行配置更改和激活，而无需锁定环境。将 `` `Startup Mode` `` 配置为 `` `Production Mode` ``（生产模式）会锁定环境，以确保在未锁定控制台进行编辑的情况下无法进行任何更改。命令行工具也将需要密码。这些选项如图 7-24 所示。

![A352855_1_En_7_Fig24_HTML.jpg](img/A352855_1_En_7_Fig24_HTML.jpg)

图 7-24. 服务器启动模式

**注意**
锁定管理控制台可防止多个管理员进行更改并相互覆盖。

输入 Java 数据库连接（JDBC）详细信息后，配置工具会验证该信息。任何错误都将显示出来。如果无法验证某个模式，请返回上一步以确保模式名称和详细信息正确。如果数据库模式不存在，请重新运行 RCU 以创建它。图 7-25 显示了一个已完成的数据库检查。

![A352855_1_En_7_Fig25_HTML.jpg](img/A352855_1_En_7_Fig25_HTML.jpg)

图 7-25. 数据库模式检查

正如 AOM 安装章节所讨论的，本书中使用的示例采用拆分域。因此，每个组件（OID、OAM 和 OIM）都将拥有自己的管理服务器。如图 7-26 所示，选择 `` `Administration Server` ``（管理服务器）、`` `Managed Servers` ``（受管服务器）、`` `Clusters` ``（集群）和 `` `Machines` ``（计算机）。

![A352855_1_En_7_Fig26_HTML.jpg](img/A352855_1_En_7_Fig26_HTML.jpg)

图 7-26. 选择可选配置屏幕

管理服务器的默认端口是 `` `7001` ``，该端口在上一章中用于访问管理器的管理侦听端口。使用标准端口约定将有助于消除混淆或遗忘端口。对于本练习，使用端口 `` `7101` ``，如图 7-27 所示。随后，如果安装更多 WLS 实例，则使用端口 `` `7201` ``、`` `7301` `` 等。

![A352855_1_En_7_Fig27_HTML.jpg](img/A352855_1_En_7_Fig27_HTML.jpg)

图 7-27. 配置管理服务器屏幕

在此环境中，访问管理器和身份管理器安装在独立的 Oracle Middleware Homes 中。这意味着每个都包含 WLS、管理服务器和受管服务器。如前所述，这样做是为了便于未来的维护，例如打补丁和升级。如果需要，它也允许在以后将这些层分离到不同的物理主机上。由于它们将在同一物理主机上运行，但位于独立的 WLS 中，因此每个管理服务器都需要自己的侦听端口。

受管服务器将预填充安装使用的标准端口。如果需要，您可以更改端口或填充安全套接字层（SSL）侦听端口。在许多情况下，允许 HTTP 服务器或负载均衡器执行 SSL 端点职责就足够了，从而通过将加密职责卸载到外部设备来获得一定的性能提升。为了便于故障排除和未来维护，请保留默认端口，如图 7-28 所示。

![A352855_1_En_7_Fig28_HTML.jpg](img/A352855_1_En_7_Fig28_HTML.jpg)

图 7-28. 配置受管服务器屏幕

在本书的过程中，集群不是关注点。但是，为便于以后创建集群，在此处输入集群配置信息，如图 7-29 所示。因此，WebLogic 域将预先配置好集群。实际上，此步骤创建了一个单节点集群，以后可以扩展以包含多台计算机和实例。

![A352855_1_En_7_Fig29_HTML.jpg](img/A352855_1_En_7_Fig29_HTML.jpg)

图 7-29. 配置集群屏幕

将受管服务器分配到所需的集群，如图 7-30 所示。请注意，可以创建单个集群并将所有受管服务器分配给它。但是，为简化管理任务，每个受管服务器都将被分配到其在 WebLogic 中的独立集群。

![A352855_1_En_7_Fig30_HTML.jpg](img/A352855_1_En_7_Fig30_HTML.jpg)

图 7-30. 将服务器分配到集群屏幕

此时应当注意，如果您的集群中只有一个节点，您可能会在受管服务器日志中看到与等待集群其他成员通信相关的错误。这些错误可以忽略，并在向集群添加其他成员后消失。



