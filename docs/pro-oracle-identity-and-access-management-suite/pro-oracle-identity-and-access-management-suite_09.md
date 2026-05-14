# Controls the maximum number of shared memory segments, in pages
kernel.shmall = 4294967296
kernel.sem = 256 32000 100 142
kernel.shmmax = 10737418240
```

在`sysctl.conf`文件中设置这些值后，您必须激活并验证新值是否已显示，使用以下命令：

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

打开文件限制必须设置为 4096 以支持该实例。为此，编辑`limits.conf`文件：

```
[root@clouddemolab home]# vi /etc/security/limits.conf
```

如果环境要安装在 Oracle Linux 或 RedHat Linux 上，您还必须在`/etc/security/limits.d/90-nproc.conf`中进行编辑。如果遗漏了此步骤，该文件中的值可能会覆盖`limits.conf`文件中的值。

在列出的两个文件中，确保添加或编辑了以下行：

```
* soft nofile 4096
* hard nofile 65536
* soft nproc 2047
* hard nproc 16384
```

编辑此文件后，必须重新启动服务器以确保所有更改生效。

### 操作系统软件包

每个 Oracle 应用程序都有自己的一套必需软件包。根据您使用的 Linux 版本，安装过程可能有所不同。在以下列表中，请注意某些软件包需要在 64 位操作系统上同时安装 32 位和 64 位版本。如果这些软件包未安装，安装将无法正确完成。Oracle 安装程序会检查这些，并在安装过程中显示错误。

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

至此，操作系统应已完全准备好，可以开始安装。在安装软件之前执行上述操作将确保安装过程顺利进行。在许多情况下，如果有任何遗漏，安装程序会提供详细的消息。如果在安装过程中出现错误，请停止安装，并在继续之前解决任何问题。


### 数据库准备

与 OID 类似，OAM 的安装也需要创建特定的数据库对象。这包括在各种数据库模式中创建多个表、视图和包。这是使用**Repository Creation Utility (RCU)** 完成的。虽然数据库对象可以在同一个数据库中创建，但通常建议将 Oracle 身份与访问管理存储库创建在一个单独的实例中。这简化了数据库管理任务和未来的维护。RCU 的“创建存储库”屏幕如图 6-1 所示。请注意下载并运行与您正在安装的 Oracle 身份与访问管理版本指定的 RCU。解压下载文件并运行 `<RCU_HOME>/rcu/bin` 目录下的 `rcu.sh`。

![A352855_1_En_6_Fig1_HTML.jpg](img/A352855_1_En_6_Fig1_HTML.jpg)

图 6-1.
创建新模式

创建数据库存储库对象的过程与为 OID 所做的相同。启动 RCU 并选择“创建”。

输入 Oracle 数据库详细信息。图 6-1 屏幕中的详细信息指向与 Oracle Directory Services 相同的数据库服务器。但是，数据库实例是专门为 Oracle Identity and Access Manager 创建的，以使两个系统分离。图 6-2 显示了在屏幕上输入的数据库详细信息。

![A352855_1_En_6_Fig2_HTML.jpg](img/A352855_1_En_6_Fig2_HTML.jpg)

图 6-2.
输入数据库连接详细信息

如图 6-3 所示，RCU 将检查以确保数据库满足最低先决条件。如果存在任何失败，必须在继续流程之前予以纠正。

![A352855_1_En_6_Fig3_HTML.jpg](img/A352855_1_En_6_Fig3_HTML.jpg)

图 6-3.
数据库预检查完成

在 RCU 的“选择组件”屏幕上，选择 `Oracle Access Manager`，如图 6-4 所示。如果需要，将选择其他组件。这包括诸如 `Audit Services`、`Metadata Services` 和 `Oracle Platform Security Services` 等项目。请确保如果项目在选择过程中被添加，您不要取消选择它们。图 6-4 显示了“选择组件”屏幕。

![A352855_1_En_6_Fig4_HTML.jpg](img/A352855_1_En_6_Fig4_HTML.jpg)

图 6-4.
组件选择

RCU 检查目标数据库以确保其满足所选组件的必要要求。RCU 日志将显示遇到的任何错误，这些错误必须在继续之前解决。如图 6-5 所示，RCU 检查所选组件的先决条件。

![A352855_1_En_6_Fig5_HTML.jpg](img/A352855_1_En_6_Fig5_HTML.jpg)

图 6-5.
数据库先决条件检查

RCU 验证先决条件后，您将被提示输入新数据库模式使用的密码，如图 6-6 所示。

![A352855_1_En_6_Fig6_HTML.jpg](img/A352855_1_En_6_Fig6_HTML.jpg)

图 6-6.
模式密码

您可以将所有模式设置为使用相同的密码，也可以为每个组件设置不同的密码。尽管您的组织可能要求不同的密码，但通常使用相同的密码就足够了。

“映射表空间”屏幕显示了将为每个组件创建的基本表空间。每个组件都会创建其他对象，可以通过单击屏幕右下角的“管理表空间”来显示它们。图 6-7 显示了为 OAM 元数据存储库创建的表空间列表。

![A352855_1_En_6_Fig7_HTML.jpg](img/A352855_1_En_6_Fig7_HTML.jpg)

图 6-7.
表空间列表

图 6-8 显示了一个可选屏幕，允许您自定义下一步中要创建的表空间的属性。

![A352855_1_En_6_Fig8_HTML.jpg](img/A352855_1_En_6_Fig8_HTML.jpg)

图 6-8.
`管理表空间` 屏幕

除非与数据库管理员 (DBA) 紧密合作，否则应避免修改 RCU 期间提供的默认值。但是，您的 DBA 可能会有提高性能的建议或要求。

图 6-9 展示了“摘要”屏幕。此屏幕让您有机会查看将根据您在先前屏幕上的输入创建的所有对象。如果任何内容看起来不合适，这是在数据库对象创建之前进行最后调整的最后机会。

![A352855_1_En_6_Fig9_HTML.jpg](img/A352855_1_En_6_Fig9_HTML.jpg)

图 6-9.
创建摘要屏幕

当 RCU 创建元数据存储库所需的对象时，将显示一个进度屏幕，如图 6-10 所示。请允许此过程在不受中断的情况下完成。如果由于任何原因此过程停止，您应该重新启动 RCU 以丢弃本次运行的所有组件，并从头开始重新创建。

![A352855_1_En_6_Fig10_HTML.jpg](img/A352855_1_En_6_Fig10_HTML.jpg)

图 6-10.
RCU 完成

RCU 已完成，并且所有数据库模式对象都已创建以支持 OAM。现在数据库已准备就绪，可以安装 OAM 软件并配置域了。


## Access Manager 软件安装

Oracle 软件安装使用 Universal Installer。该工具作为引导程序，确保正确的软件二进制文件被安装在正确的位置。`runInstaller`工具要求在运行时指定 Java Runtime Environment。通过使用`–jreLoc`参数来完成此操作。

在 Linux 中，按如下方式运行：

```
[oracle@clouddemolab Disk1]$ ./runInstaller -jreLoc /home/oracle/jdk1.6.0.45/jre
```

图 6-11 显示了 Oracle Universal Installer 的欢迎屏幕。文本中应能看到 Oracle Identity and Access Management 的版本信息，本例中为`11.1.2.3`。如果这与您正在安装的版本不匹配，请找到正确的安装程序软件。上一节运行的 RCU 仅与`11.1.2.3`兼容。

![A352855_1_En_6_Fig11_HTML.jpg](img/A352855_1_En_6_Fig11_HTML.jpg)

**图 6-11. 欢迎屏幕**

与其他 Oracle 产品一样，安装程序首先验证目标主机是否满足最低要求，以及所有必要的 OS 软件包是否已安装。详情请参阅本章开头的先决条件列表。图 6-12 显示先决条件检查已成功完成。

![A352855_1_En_6_Fig12_HTML.jpg](img/A352855_1_En_6_Fig12_HTML.jpg)

**图 6-12. 先决条件检查屏幕**

如果在先决条件检查期间发现任何问题，则必须予以解决。这包括 OS 软件包和参数。

### 运行安装程序

安装程序确认先决条件后，将要求您提供新的 Oracle Middleware Home 目录。如图 6-13 所示，需要指定一个 Middleware Home 目录位置。您应仅将此软件安装在现有的 Middleware Home 内。此位置是 WebLogic 在此过程开始时安装的位置。

![A352855_1_En_6_Fig13_HTML.jpg](img/A352855_1_En_6_Fig13_HTML.jpg)

**图 6-13. Oracle Middleware Home 选择屏幕**

OAM 软件必须安装到现有的 Fusion Middleware Home 中。安装 WLS 会创建 Fusion Middleware home。安装 WLS 的说明可以在第 4 章找到。按照这些说明，在名为`IAMMiddleware`的目录中创建一个 home。这确保了 OAM 环境与 OID 中间件目录结构分离。

Oracle Identity and Access Manager Installer 仅用于安装 Oracle Access Manager 和 Oracle Identity Manager。因此，它不会提示您选择要安装的组件。如图 6-14 所示，您将看到计划安装内容的摘要。

![A352855_1_En_6_Fig14_HTML.jpg](img/A352855_1_En_6_Fig14_HTML.jpg)

**图 6-14. 组件摘要**

### 先决条件检查

在完成之前，Universal Installer 将显示将安装到 Middleware Home 中的组件摘要。单击“Install”以放置二进制文件。

图 6-15 中显示的进度条在`process`完成时将显示为 100%。请耐心等待，因为有时此进度条可能看起来没有移动。如果出现任何问题，请检查屏幕上显示位置的安装日志。

![A352855_1_En_6_Fig15_HTML.jpg](img/A352855_1_En_6_Fig15_HTML.jpg)

**图 6-15. Oracle Access Manager 安装完成**

### 选择 Middleware Home

完成后，安装程序将提供已安装组件的摘要和文件位置。请记录完成数据中显示的信息。此完成摘要的示例如图 6-16 所示。

![A352855_1_En_6_Fig16_HTML.jpg](img/A352855_1_En_6_Fig16_HTML.jpg)

**图 6-16. Oracle Identity and Access Management 安装完成**

### 安装完成

Oracle Identity and Access Management 软件安装现已完成。如前所述，环境可以在拆分域或组合域中实施。在拆分域中，OAM 安装在单独的 Middleware Home 中，甚至安装在单独的物理服务器上。组合域将两个组件放在同一个 Middleware Home 中。出于本书的目的，将使用拆分域。本章的其余部分将重点介绍创建 OAM 域。



## 创建访问管理器域

### 运行配置脚本

安装完 OAM 后，可以配置一个新域。这是通过运行位于 `IDM_HOME/common/bin` 目录中的 `config.sh` 脚本来完成的。需要注意的是，存在多个位置的 `config.sh`。要配置 OAM 域，必须使用指定的配置脚本。

此操作从 `<MIDDLEWARE_HOME>/Oracle_IDM1/common/bin` 目录运行。

```
[oracle@clouddemolab bin]$ pwd
/home/oracle/IAMMiddleware/Oracle_IDM1/common/bin
[oracle@clouddemolab bin]$ ./config.sh
```

### 选择配置类型和组件

从 `ORACLE_HOME/common/bin` 目录运行配置实用程序将启动访问管理器域配置向导。首先会看到如图 6-17 所示的欢迎屏幕。

![A352855_1_En_6_Fig17_HTML.jpg](img/A352855_1_En_6_Fig17_HTML.jpg)
*图 6-17. 创建新域的欢迎屏幕*

Fusion Middleware 软件在 WLS 环境中的域内部署。首先，选择 `创建新的 WebLogic 域` 选项。

选择配置类型后，将看到一个可以在新域内配置的身份管理组件列表。此选择屏幕如图 6-18 所示。域配置过程涉及选择将在域创建时部署到其中的软件组件。此列表基于上一步中安装的软件。在选择过程中，根据需要预选必需的组件。在此步骤中，选择了 Oracle Access Management 和 Mobile Security Suite 11.1.2.3.0，并且 Oracle Enterprise Manager 组件也被预选。

![A352855_1_En_6_Fig18_HTML.jpg](img/A352855_1_En_6_Fig18_HTML.jpg)
*图 6-18. 为域选择组件*

### 指定域位置和管理用户

在此步骤中，您需要提供域的名称并指定文件系统位置。根据文件系统要求和高可用性需求，这些文件可能需要部署在不同的位置。这些将在本书的高可用性部分进行讨论。图 6-19 展示了此步骤的示例。

![A352855_1_En_6_Fig19_HTML.jpg](img/A352855_1_En_6_Fig19_HTML.jpg)
*图 6-19. 指定域位置*

为 `weblogic` 用户提供密码，如图 6-20 所示。虽然在此阶段可以指定不同的用户名，但通常使用 `weblogic`。之后您可以创建其他有权访问管理实用程序的用户。`weblogic` 用户密码应设置为您环境中标准的密码。此密码将用于启动和停止受管服务器，以及登录 WebLogic 管理控制台和 Fusion Middleware Control。

![A352855_1_En_6_Fig20_HTML.jpg](img/A352855_1_En_6_Fig20_HTML.jpg)
*图 6-20. 设置管理用户密码*

### 配置域启动模式

图 6-21 展示了 `启动模式` 选择屏幕。

![A352855_1_En_6_Fig21_HTML.jpg](img/A352855_1_En_6_Fig21_HTML.jpg)
*图 6-21. 域启动模式配置*

启动模式决定了受管服务器的启动方式。在开发模式下，受管服务器可以通过命令行启动和停止而无需密码。此外，可以在 WebLogic 控制台中进行配置更改并激活，而无需锁定环境。将启动模式配置为生产模式会锁定环境，以确保在未锁定控制台进行编辑的情况下无法进行更改。命令行工具也将需要密码。

**注意：** 锁定管理控制台可防止多个管理员进行更改并相互覆盖。

### 配置 JDBC 数据库连接

如图 6-22 所示的数据库连接屏幕，需要为要在 WebLogic 域中配置的每个组件输入数据库连接详细信息。显示的数据库模式是预填充的，并且应该已使用 RCU 创建。

![A352855_1_En_6_Fig22_HTML.jpg](img/A352855_1_En_6_Fig22_HTML.jpg)
*图 6-22. 配置 JDBC 数据库连接*

输入数据库详细信息，如主机、端口和服务名。必须输入每个模式的密码。此步骤在 WebLogic 域内创建 Java 数据库连接性（JDBC）数据源。

**注意：** 在此屏幕上选中所有模式旁边的复选框，可以复制条目而无需多次输入数据。

输入 JDBC 连接详细信息后，配置工具会验证信息。任何错误都将显示。如果某个模式无法验证，请返回上一步以确保模式名称和详细信息正确。如果数据库模式不存在，请重新运行 RCU 创建它。图 6-23 显示了数据库存储库验证完成。

![A352855_1_En_6_Fig23_HTML.jpg](img/A352855_1_En_6_Fig23_HTML.jpg)
*图 6-23. 检查模式*

### 选择可选配置

出于本章的目的，您将创建管理服务器、受管服务器、集群和计算机，如图 6-24 所示。如果此时您的环境需要部署和服务或 RDBMS 安全存储，请立即选择它们。此屏幕上选择的项目决定了后续屏幕。只有所选内容需要的屏幕才会显示。

![A352855_1_En_6_Fig24_HTML.jpg](img/A352855_1_En_6_Fig24_HTML.jpg)
*图 6-24. 可选配置*

### 配置管理服务器

在以下步骤中，将配置管理服务器、受管服务器、集群和计算机。图 6-25 显示了配置管理服务器屏幕。

![A352855_1_En_6_Fig25_HTML.jpg](img/A352855_1_En_6_Fig25_HTML.jpg)
*图 6-25. 管理服务器配置*

一个域中一次只能运行一个管理服务器。使用配置工具，可以指定一个管理服务器。在集群环境中，可以在辅助节点上配置备份管理服务器，以防主节点丢失。在此步骤中，请确保指定的端口在物理主机上可用。

### 配置受管服务器

输入管理服务器详细信息后，您将看到图 6-26 所示的配置受管服务器屏幕。受管服务器将根据先前选择的组件进行预填充。

![A352855_1_En_6_Fig26_HTML.jpg](img/A352855_1_En_6_Fig26_HTML.jpg)
*图 6-26. 受管服务器配置*

在域配置期间，选定的产品将部署在受管服务器中。图 6-26 显示了要创建的受管服务器。您应确保在此屏幕上输入的端口是可用的。

### 配置集群

如果您计划安装多个 OAM 实例以提供故障转移或负载均衡能力，如图 6-27 所示的配置集群屏幕允许进行此配置。

![A352855_1_En_6_Fig27_HTML.jpg](img/A352855_1_En_6_Fig27_HTML.jpg)
*图 6-27. 集群配置*

集群允许多个应用程序实例部署并协同工作，以提供高可用性环境或更好的性能。配置集群屏幕允许配置工具在此阶段定义集群。请继续配置 OAM 集群。集群可以稍后使用管理控制台创建。

### 将受管服务器分配到集群

指定集群名称后，您需要将受管服务器分配给每个集群，如图 6-28 所示。


![A352855_1_En_6_Fig28_HTML.jpg](img/A352855_1_En_6_Fig28_HTML.jpg)

图 6-28.

将托管服务器分配到集群

托管服务器可以分配到前面屏幕定义的集群中。根据给定环境中部署的产品，您可能希望将托管服务器分散到不同的集群。这些分配可以稍后使用管理控制台重新分配。

机器定义了一个管理托管服务器内进程的主机。您可以在给定主机上创建多个机器，或者将机器分散到多个主机上。图 6-29 显示了“配置机器”屏幕。

![A352855_1_En_6_Fig29_HTML.jpg](img/A352855_1_En_6_Fig29_HTML.jpg)

图 6-29.

机器配置

如果主机基于 Linux 或 UNIX，则创建 UNIX 机器非常重要。机器配置告诉 WebLogic 环境如何联系节点管理器以控制托管服务器。

机器配置完成后，是时候将要创建的托管服务器分配到适当的机器上，如图 6-30 所示。通常，在集群环境中，存在多台机器，每台机器都有一个托管服务器节点。这使得单个管理服务器能够联系并控制不同主机或不同 Middleware Homes 中的托管服务器。

![A352855_1_En_6_Fig30_HTML.jpg](img/A352855_1_En_6_Fig30_HTML.jpg)

图 6-30.

将托管服务器分配到机器

“配置摘要”屏幕，如图 6-31 所示，提供了在执行配置之前对所有配置参数的最后查看。如果此时有任何内容看起来不正确，请返回并进行必要的修复，然后再继续。

![A352855_1_En_6_Fig31_HTML.jpg](img/A352855_1_En_6_Fig31_HTML.jpg)

图 6-31.

配置摘要屏幕

检查“配置摘要”屏幕后，单击“创建”以开始配置过程。图 6-32 显示了完成的配置。

![A352855_1_En_6_Fig32_HTML.jpg](img/A352855_1_En_6_Fig32_HTML.jpg)

图 6-32.

域配置完成

OAM 域配置已完成。在启动之前，还需要几个步骤。下一章将介绍 Identity Manager 组件的安装和配置。随后，整体身份和访问管理器环境将被配置为协同工作。

## 总结

本章是 OAM 安装过程的演练。在安装软件文件之后，向您展示了 OAM 域的配置。此时，所有必要的文件已复制完毕，文件系统已准备就绪。如果您的实施只需要 SSO，可以跳过下一章关于 Identity Manager 安装的内容。

# 7. Identity Manager 安装

在前两章中，提供了安装和配置 Oracle Internet Directory (OID) 和 Oracle Access Manager (OAM) 的说明。这两个组件奠定了身份存储和单点登录 (SSO) 的基础。Identity Manager 为环境添加了额外的元素，例如用户自助服务和治理，从而完成了端到端身份生命周期管理的实施。本章重点介绍 Oracle Identity Manager (OIM) 的安装和初始域配置。

## 预安装任务

### 操作系统用户

对于大多数 Oracle 应用程序安装，应创建操作系统 (OS) 用户和组来执行安装和配置任务。创建 OS 组将允许其他 OS 用户执行与管理应用程序环境相关的特定任务。在 Linux 环境中安装 Oracle 应用程序最常见的 OS 用户和组是 `oracle` 用户以及 `oinstall` 或 `dba` 组。

要创建必要的 `oinstall` 和 `dba` 组，请以 root 目录执行以下命令：

```
[root@clouddemolab home]# groupadd oinstall
[root@clouddemolab home]# groupadd dba
```

创建组后，创建 `oracle` 用户：

```
[root@clouddemolab home]# useradd  -g oinstall -G dba oracle
```

注意

`-g` 表示用户应添加到的主组。`-G` 表示任何辅助组。

要设置用户密码，请以 root 用户身份使用以下命令。

```
[root@clouddemolab home]# passwd oracle
```

### 操作系统配置

在安装 Oracle Fusion Middleware 基础架构和 Oracle Identity Management 软件之前，确保操作系统满足最低要求和配置非常重要。以下列出了所需的内核参数、软件包以及文件更改。

需要设置以下内核参数：

```
kernel.sem  256  32000  100  143
kernel.shmmax 10737418240
```

要设置这些参数，请编辑 `/etc` 目录中的 `sysctl.conf` 文件。

```
[root@clouddemolab home]# vi /etc/sysctl.conf
```

在文件的此部分添加或编辑以下行：

```
