# 1. 安装 Oracle 二进制文件

过去的 Oracle 安装可能庞大、复杂且笨重。Oracle 数据库管理员负责规划和执行安装，因为他或她知道如何在步骤中出现的问题进行故障排除和解决。有多个配置和安装选项需要审查，因此安装 Oracle 软件（二进制文件）是一项要求每位数据库管理员都熟练掌握的任务。现在，随着 Oracle 18c 乃至 12c 的推出，Oracle 软件安装已变得更加自动化，但理解安装的步骤和配置对于数据库管理员来说仍然很重要。对于大型环境，Oracle 二进制文件的安装应该是可重复的，数据库管理员需要设置安装程序，以便能够按需并一致地配置数据库。



#### 提示

DBA 的职责正在发生变化。在云环境中，DBA 的任务可能是准备自助数据库，甚至可能不再需要安装 Oracle 二进制文件。此外，如果你对 Oracle 还比较陌生，那么安装过程中一些令人望而生畏的环节已经得到了简化。很可能另一位 DBA 已经安装了 Oracle 二进制文件，你只需要根据需要创建数据库即可。然而，理解安装的组成部分以及下一节“理解最佳灵活架构（OFA）”仍然很有价值。

许多 DBA 不使用自动化安装技术。有些人不了解这些方法；另一些人则认为它们不可靠。因此，大多数 DBA 通常使用 Oracle 通用安装程序 (OUI) 的图形界面模式。虽然图形安装程序是一个好工具，但它本身不具备可重复性和自动化能力。运行图形安装程序是一个手动过程，期间你需要在多个屏幕上选择选项。即使你知道该选择哪些选项，仍然可能无意中点击了错误的选项。

当你进行远程安装且网络带宽不足时，图形安装程序也可能出现问题。在这种情况下，你可能会发现自己需要等待数十分钟，以便屏幕在本地重新绘制。你需要一种不同的技术来在远程服务器上进行高效安装。

本章重点介绍以高效且可重复的方式安装 Oracle 的技术。这包括依赖响应文件进行的静默安装。响应文件是一个文本文件，你在其中为控制安装的变量赋值。DBA 们常常没有意识到使用响应文件所能实现的强大可重复性和效率。

## 注意

本章仅涵盖安装 Oracle 软件。创建数据库的任务在第 2 章中介绍。云环境也改变了软件和数据库创建的任务及观点，这将在本章后面的“在云中安装及 DBA 的职责”一节中讨论。

## 理解 OFA

在安装 Oracle 并开始创建数据库之前，你必须理解 Oracle 的最佳灵活架构 (OFA) 标准。该标准被广泛用于指定一致的目录结构以及安装和创建 Oracle 数据库时使用的文件命名约定。

## 注意

这个无处不在的 OFA “标准”的一个讽刺之处在于，几乎每个 DBA 都会以某种方式对其进行定制，以适应其环境的独特需求。

OFA 标准提供了在一致的基础上定位日志文件的方法。如果遵循了这些标准，由于环境间的一致性，安全性、迁移和自动化将更容易实施。日志文件的一致位置也允许被其他工具使用以及确保其安全。18c 中的 `ORACLE_BASE` 目录提供了一种将 `ORACLE_HOME` 目录分离为只读目录，并将可写文件放在 `ORACLE_BASE` 中的方法。只读的 `ORACLE_HOME` 目录允许实施安装与配置的分离，这对于云环境和保护环境非常重要。这简化了修补过程，因为一个镜像可以用于大规模部署，并将补丁分发到许多服务器，减少了修补和更新 Oracle 软件的停机时间。

由于大多数企业都实施了某种形式的 OFA 标准，理解这种结构至关重要。图 1-1 显示了 OFA 标准使用的目录结构和文件名。并非 Oracle 环境中的所有目录和文件都出现在此图中（没有足够的空间）。然而，关键且最常用的目录和文件都已显示。

![../images/214899_3_En_1_Chapter/214899_3_En_1_Fig1_HTML.jpg](img/214899_3_En_1_Fig1_HTML.jpg)

图 1-1

Oracle 的 OFA 标准

OFA 标准包括几个你应该熟悉的目录：

*   Oracle 清单目录
*   Oracle 基目录 (`ORACLE_BASE`)
*   Oracle 主目录 (`ORACLE_HOME`)
*   Oracle 网络文件目录 (`TNS_ADMIN`)
*   自动诊断库 (`ADR_HOME`)

这些目录将在以下部分中讨论。

### Oracle 清单目录

Oracle 清单目录存储服务器上安装的 Oracle 软件的清单。此目录是必需的，并在服务器上所有 Oracle 软件安装之间共享。当你首次安装 Oracle 时，安装程序会检查是否存在格式为 `/u[01–09]/app` 的现有符合 OFA 的目录结构。如果存在这样的目录，则安装程序会创建一个 Oracle 清单目录，例如：

```
/u01/app/oraInventory
```

如果为 `oracle` 操作系统 (OS) 用户定义了 `ORACLE_BASE` 变量，则安装程序会为 Oracle 清单的位置创建一个目录，如下所示：

```
ORACLE_BASE/../oraInventory
```

例如，如果 `ORACLE_BASE` 定义为 `/ora01/app/oracle`，则安装程序将 Oracle 清单的位置定义为：

```
/ora01/app/oraInventory
```

如果安装程序找不到可识别的符合 OFA 的目录结构或 `ORACLE_BASE` 变量，则 Oracle 清单的位置将在 `oracle` 用户的 `HOME` 目录下创建。例如，如果 `HOME` 目录是 `/home/oracle`，则 Oracle 清单的位置是：

```
/home/oracle/oraInventory
```

### Oracle 基目录

Oracle 基目录是 Oracle 软件安装的最高级目录。你可以在此目录下安装一个或多个版本的 Oracle 软件。Oracle 基目录的 OFA 标准如下：

```
/<mount_point>/app/<software_owner>
```

挂载点的典型名称包括 `/u01`、`/ora01`、`/oracle` 和 `/oracle01`。你可以根据环境的标准为挂载点命名。我更喜欢使用诸如 `/ora01` 之类的挂载点名称。它很短，当我查看数据库服务器上的挂载点时，我可以立即分辨出哪些用于 Oracle 数据库。此外，较短的挂载点名称在通过操作系统命令浏览目录时更容易输入。

软件所有者通常名为 `oracle`。这是你用来安装 Oracle 软件（二进制文件）的 OS 用户。下面列出了一个完整形成的 Oracle 基目录路径示例：

```
/u01/app/oracle
```

### Oracle 主目录

Oracle 主目录定义了特定产品的软件安装位置，例如 Oracle Database 18c 或 Oracle Database 12c。你必须将不同产品或产品的不同版本安装在单独的 Oracle 主目录中。推荐的符合 OFA 的 Oracle 主目录如下：

```
ORACLE_BASE/product/<version>/<install_name>
```

在前面的代码行中，可能的版本包括 `18.1.0.1` 和 `12.2.0.1`。可能的 `install_name` 值包括 `db_1`、`devdb1`、`test2` 和 `prod1`。以下是 18c 数据库的 Oracle 主目录名称示例：

```
/u01/app/oracle/product/18.1.0.1/dbhome_1/db1
```

## 注意

一些 DBA 不喜欢 `ORACLE_HOME` 目录末尾的 `db1` 字符串，认为没有必要。`db1` 存在的原因是，你可能有两个独立的二进制安装：一个开发安装和一个测试安装。如果你的环境不需要这种配置，可以随意去掉额外的字符串（`db1`）。



## Oracle 网络文件目录

一些 Oracle 工具使用`TNS_ADMIN`的值来定位网络配置文件。该目录定义为`ORACLE_HOME/network/admin`。它通常包含 Oracle Net 的`tnsnames.ora`和`listener.ora`文件。`listener.ora`文件现在通常随 Oracle Grid 安装，而不是在数据库主目录中。监听器通常由管理网格、集群和 ASM 软件的系统维护。`tnsnames`提供了连接到其他数据库的方法，因此这些文件是集中目录的一部分，或是数据库网络文件的一部分。

#### 提示

有时 DBA 会将`TNS_ADMIN`设置为指向一个中央目录位置（如`/etc`或`/var/opt/oracle`）。这使他们能够维护一套 Oracle 网络文件（而不是为每个`ORACLE_HOME`维护一套）。这种方法还有一个优点，就是在数据库升级（可能会改变`ORACLE_HOME`的位置）时，不需要复制或移动文件。

## 自动诊断存储库

从 Oracle Database 11g 开始，`ADR_HOME`目录指定了与 Oracle 相关的诊断文件的位置。这些文件对于排查 Oracle 数据库问题至关重要。该目录定义为`ORACLE_BASE/diag/rdbms/lower(db_unique_name)/instance_name`。你可以查询`V$PARAMETER`视图以获取`db_unique_name`和`instance_name`的值。

例如，在下一行中，小写的数据库唯一名是`db18c`，实例名是`DB18C`：

```
/u01/app/oracle/diag/rdbms/db18c/DB18C
```

或者在集群环境中，小写的数据库唯一名是`db18c`，实例名是`DB18C01`：

```
/u01/app/oracle/diag/rdbms/db18c/DB18C01
```

你可以通过此查询验证`ADR_HOME`目录的位置：

```
SQL> select value from v$diag_info where name='ADR Home';
```

以下是一些示例输出：

```
VALUE

/u01/app/oracle/diag/rdbms/db18c/DB18C
```

现在你了解了 OFA 标准，接下来你将看到在安装 Oracle 二进制文件时如何使用它。例如，在运行 Oracle 安装程序时，你需要为`ORACLE_BASE`和`ORACLE_HOME`目录指定目录值。

#### 提示

有关 OFA 的完整详情，请参阅*Oracle Database Installation Guide*。该文档可以从 Oracle 网站的技术网络区域免费下载（[`http://otn.oracle.com`](http://otn.oracle.com)）。

## 安装 Oracle

假设你是新来的，你的经理问你在服务器上安装一套新的 Oracle Database 18c 软件需要多长时间。你回答说不到一个小时。你的老板表示怀疑，并说之前的 DBA 总是估计至少需要一天时间在新服务器上安装 Oracle 二进制文件。你回答：“其实没那么复杂，但 DBA 们确实倾向于高估安装时间，因为很难预料所有可能出错的事情。”

当你拿到一台新服务器并被分配安装 Oracle 二进制文件的任务时，这通常指的是在创建 Oracle 数据库之前，下载和安装所需软件的过程。此过程涉及几个步骤：

1.  创建适当的操作系统组。在 Oracle Database 18c 中，有几个你可以创建并使用的操作系统组来管理`SYSDBA`权限的粒度级别。至少，你需要创建一个操作系统`dba`组和操作系统`oracle`用户。
2.  确保操作系统已为 Oracle 数据库进行充分配置。
3.  从 Oracle 获取数据库安装软件。
4.  解压数据库安装软件。
5.  如果在首次在服务器上安装 Oracle 软件时使用静默安装程序，请创建一个`oraInst.loc`文件。此步骤每个服务器只需执行一次。后续安装不需要执行此步骤。
6.  配置响应文件并运行 Oracle 静默安装程序。
7.  对任何问题进行故障排除。
8.  应用任何额外的补丁。

这些步骤将在以下小节中详细说明。

## 注意

Oracle 指定的任何基础版本（10.1.0.2、10.2.0.1、11.1.0.6、11.2.0.1、12.1.0.1、18.1.0.1 等）的数据库都可以从 Oracle 网站的技术网络区域（[`http://otn.oracle.com`](http://otn.oracle.com)）免费下载。但是，请注意，任何后续补丁的下载都需要购买许可。换句话说，下载基础软件需要 Oracle 技术网络（OTN）登录（免费），而下载补丁集则需要 My Oracle Support 账户（收费）。

### 步骤 1. 创建操作系统组和用户

如果你所在的机构有系统管理员（SA），那么步骤 1 和 2 通常由 SA 执行。如果你没有 SA，那么你必须自己执行这些步骤（这在小型机构中很常见，你可能需要执行许多不同的工作职能）。你需要`root`访问权限来完成这些步骤。

过去，典型的 Oracle 安装包含一个操作系统组（`dba`）和一个操作系统用户（`oracle`）。你仍然可以使用这种极简的、一组一用户的方法安装 Oracle 软件；它工作正常。如果你的机构只有一名 DBA，并且你不需要在团队成员之间进行更精细的权限划分，那么尽管去创建`dba`组和`oracle`操作系统用户。这种方法没有任何问题。

如今，Oracle 建议你创建多个操作系统组——其思想是你可以根据工作职能的需要，添加不同的操作系统用户并将他们分配到组中。当一个操作系统用户被分配到一个组时，该分配会为用户提供特定的数据库权限。表 1-1 记录了操作系统组以及每个组如何映射到相应的数据库权限。例如，如果你有一个用户只负责监控数据库，并且只需要启动和关闭数据库的权限，那么该用户将被分配到`oper`组（这确保后续可以以`sysoper`权限连接到数据库）。

**表 1-1**
操作系统组与备份恢复相关权限的映射

| **操作系统组** | **数据库系统权限** | **授权操作** | **引用位置** |
| --- | --- | --- | --- |
| `oinstall` | 无 | 安装和升级 Oracle 二进制文件的操作系统权限 | `oraInst.loc`文件中的`inst_group`变量；也由响应文件中的`UNIX_GROUP_NAME`变量定义 |
| `dba` | `sysdba` | 所有数据库权限：启动、关闭、修改数据库、创建和删除数据库、切换归档日志模式、备份和恢复数据库 | 响应文件中的`DBA_GROUP`变量或在 OUI 图形安装程序提示时 |
| `oper` | `sysoper` | 启动、关闭、修改数据库、切换归档日志模式、备份和恢复数据库 | 响应文件中的`OPER_GROUP`变量或在 OUI 图形安装程序提示时 |
| `asmdba` | `sysdba for asm` | Oracle 自动存储管理（ASM）实例的管理权限 | 不适用 |
| `asmoper` | `sysoper for asm` | 启动和停止 Oracle ASM 实例 | 不适用 |
| `asmadmin` | `sysasm` | 磁盘组的挂载和卸载以及其他存储管理 | 不适用 |
| `backupdba` | `sysbackup` | 12c 新增；允许用户启动、关闭以及执行所有备份和恢复操作的权限 | 响应文件中的`BACKUPDBA_GROUP`变量或在 OUI 图形安装程序提示时 |
| `dgdba` | `sysdg` | 12c 新增；与管理 Data Guard 环境相关的权限 | 响应文件中的`DGDBA_GROUP`变量或在 OUI 图形安装程序提示时 |
| `kmdba` | `syskm` | 12c 新增；与加密管理相关的权限 | 响应文件中的`KMDBA_GROUP`变量或在 OUI 图形安装程序提示时 |



### Oracle 数据库安装前的系统准备

表 1-1 包含推荐的组名。您不一定必须使用列出的组名；可以根据您的需求进行调整。例如，如果您有两组独立的用户共用同一台服务器，您可能希望创建两个独立的 Oracle 安装，分别由不同的 DBA 管理；开发 DBA 组可以使用名为 `dbadev` 的组来创建和安装 Oracle 二进制文件，而使用同一台机器的测试组可能安装另一套由 `dbatest` 组管理的 Oracle 二进制文件。每组仅拥有操作其对应二进制文件的权限。或者，如前所述，您也可以决定只为所有操作使用一个组 (`dba`)。这完全取决于您的环境。

一旦决定了需要哪些组，您就需要获取 `root` 用户权限来运行 `groupadd` 命令。以 `root` 身份，添加您需要的操作系统组。这里，我添加三个预估需要的组：

```
#### groupadd oinstall
#### groupadd dba
#### groupadd oper
```

如果您无法访问 `root` 帐户，那么您需要请您的系统管理员（SA）运行上述命令。您可以通过检查 `/etc/group` 文件的内容来验证每个组是否已成功添加。以下是 `/etc/group` 文件中创建的典型条目：

```
oinstall:x:500:
dba:x:501:
oper:x:502:
```

现在，创建 `oracle` 操作系统用户。以下示例显式地将组 ID 设置为 500（您的公司可能要求所有安装使用相同的组 ID），将主组设为 `oinstall`，并将 `dba` 和 `oper` 组分配给新创建的 `oracle` 用户：

```
#### useradd -u 500 -g oinstall -G dba,oper oracle
```

您可以通过查看 `/etc/passwd` 文件来验证用户帐户信息。以下是 `oracle` 用户的预期条目：

```
oracle:x:500:500::/home/oracle:/bin/bash
```

如果您需要修改一个组，请以 `root` 身份使用 `groupmod` 命令。如果出于任何原因需要删除一个组（以 `root` 身份），请使用 `groupdel` 命令。

如果您需要修改一个用户，请以 `root` 身份使用 `usermod` 命令。如果您需要删除一个操作系统用户，请使用 `userdel` 命令。运行 `userdel` 命令需要 `root` 权限。此示例从服务器中移除 `oracle` 用户：

```
#### userdel oracle
```

### 步骤 2. 确保操作系统配置充分

此步骤相关的任务因数据库版本和操作系统的不同而有所差异。您必须查阅针对数据库版本和操作系统供应商的 Oracle 安装手册以获取确切要求。执行此步骤需要验证和配置以下操作系统组件：

*   内存和交换空间
*   系统架构（处理器）
*   可用磁盘空间（Oracle 现在安装几乎需要 5GB 空间）
*   操作系统版本和内核
*   操作系统软件（所需的软件包和补丁）

在 Linux 服务器上运行以下命令确认内存大小：

```
$ grep MemTotal /proc/meminfo
```

要验证内存和交换空间的总量，请运行以下命令：

```
$ free -t
```

要验证 `/tmp` 目录的空间量，请输入此命令：

```
$ df -h /tmp
```

要显示可用磁盘空间量，请执行此命令：

```
$ df -h
```

要验证操作系统版本，请输入此命令：

```
$ cat /proc/version
```

要验证内核信息，请运行以下命令：

```
$ uname -r
```

要确定是否已安装所需的软件包，请执行此查询，并提供所需的软件包名称：

```
$ rpm -q 
```

再次强调，数据库服务器的要求因操作系统和数据库版本差异很大。您可以从 Oracle 网站的文档页面 ([`www.oracle.com/documentation`](http://www.oracle.com/documentation)) 下载特定的安装手册。

## 注意

OUI（Oracle Universal Installer）会显示操作系统软件和硬件的任何不足。运行安装程序将在步骤 6 中介绍。

### 步骤 3. 获取 Oracle 安装软件

通常，获取 Oracle 软件最简单的方法是从 Oracle 网站下载。导航到软件下载页面 ([`www.oracle.com/technology/software`](http://www.oracle.com/technology/software))，并下载适合您计划安装它的操作系统和硬件类型（Linux、Solaris、Windows 等）的 Oracle 数据库版本。

### 步骤 4. 解压缩文件

对于之前的版本，建议将文件解压缩到可以放置 Oracle 安装介质的标准目录中。现在对于 Oracle 18c，介质的提供方式有所不同，可以基于映像或通过 RPM。映像软件现在必须解压到 `ORACLE_HOME` 目录中。压缩文件可以放在临时目录中，但必须解压到 `ORACLE_HOME`。`runInstaller` 将从 `ORACLE_HOME` 目录运行以进行安装。

为 `ORACLE_HOME` 创建目录：

```
$ mkdir -p /u01/app/oracle/product/18.1.0/dbhome_1
$ chown oracle:oinstall /u01/app/oracle/product/18.1.0/dbhome_1
```

可以将 zip 文件下载或复制到临时目录，例如 `/tmp` 或 `/home/oracle`。

使用 `unzip` 命令解压到新创建的 `ORACLE_HOME` 目录：

```
$ cd /u01/app/oracle/product/18.1.0/dbhome_1
$ unzip -q /tmp/db_home.zip
```

现在可以使用 RPM 与 Oracle 18c 来执行单数据库实例的安装。RPM 以前用作预安装检查，现在可用于安装，甚至适用于 Oracle 客户端。这需要以 `root` 身份执行。

```
$ yum -y install oracle-database-server-18c-preinstall
$ ls -lt /opt
$ chown -R oracle:oinstall /opt
```

现在进入 rpm 的目录并运行命令执行基于 RPM 的安装。`ORACLE_HOME` 目录将在 `/opt/oracle/product/18.1.0.0.0-1/dbhome_1` 创建。

```
$ cd /tmp/rpm
$ rpm -ivh oracle-ee-db-18.1.0.0.0-1.x86_64.rpm  -- rpm 名称可能因版本而异
```

#### 提示

在某些早期版本的 Oracle 安装中，您可能会发现分发文件是作为压缩的 `cpio` 文件提供的。您可以使用一个命令解压缩和解包该文件，如下所示：

```
$ cat 10gr2_db_sol.cpio.gz | gunzip | cpio -idvm
```


#### 第 5 步：创建 oraInst.loc 文件

如果您的服务器上已经存在 `oraInst.loc` 文件，则可以跳过此步骤。创建 `oraInst.loc` 文件仅在首次使用静默安装方法在服务器上安装二进制文件时才需要执行。如果您使用 `OUI` 图形安装程序，则 `oraInst.loc` 文件会自动为您创建。

在 Linux 服务器上，`oraInst.loc` 文件通常位于 `/etc` 目录中。在其他 Unix 系统（如 `Solaris`）上，此文件位于 `/var/opt/oracle` 目录中。`oraInst.loc` 文件包含以下信息：

*   Oracle 清单目录路径
*   拥有安装和升级 Oracle 软件权限的操作系统组名称

Oracle 清单目录路径是用于管理 Oracle 安装和升级的相关文件的存储位置。通常，每个主机有一个 Oracle 清单。在此目录结构中包含 `inventory.xml` 文件，该文件记录了服务器上安装的各种 Oracle 版本的位置。

Oracle 清单操作系统组拥有安装和升级 Oracle 软件所需的操作系统权限。Oracle 建议您将此组命名为 `oinstall`。您会发现有时 DBA 会将清单组分配给 `dba` 组。如果您的环境不需要单独的组（如 `oinstall`），那么使用 `dba` 组也可以。

您可以使用 `vi` 等实用程序创建 `oraInst.loc` 文件。以下是该文件中的一些示例条目：
```
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
```

以 `root` 用户身份，确保响应文件的所有者为 `oracle` 操作系统用户，并且具有适当的文件访问权限：
```
#### chown oracle:oinstall oraInst.loc
#### chmod 664 oraInst.loc
```

#### 第 6 步：配置响应文件并运行安装程序

您可以在两种模式之一运行 `OUI`：图形模式或静默模式。通常，DBA 使用图形安装程序。然而，我强烈倾向于使用静默安装选项，原因如下：

*   静默安装不需要 `X Window System` 软件的可用性。
*   您可以避免远程图形安装的性能问题，这在尝试在本地绘制屏幕时可能非常缓慢。
*   静默安装可以编写脚本并实现自动化。这意味着无论由哪位团队成员执行安装，每次安装都可以按照相同、一致的标准进行（我甚至让 `SA` 这样安装 Oracle 二进制文件）。

执行静默安装的关键是使用响应文件。

解压缩 Oracle 软件后，导航到 `ORACLE_HOME` 目录；例如，
```
$ cd /u01/app/oracle/product/18.1.0/dbhome_1
```

接下来，查找 Oracle 提供的示例响应文件：
```
$ find . -name "*.rsp"
```

根据 Oracle 版本和操作系统平台，您找到的响应文件名称和数量可能大不相同。接下来的两节展示了两种场景：Oracle Database 12c 第 1 版静默安装和 Oracle Database 18c 第 1 版静默安装。

请记住，响应文件的格式可能因 Oracle 数据库版本的不同而有很大差异。例如，Oracle Database 11g 和 12c 之间甚至在这些版本的第 2 版之间也存在重大差异。安装新版本时，您必须检查响应文件并确定必须设置哪些参数。请务必根据您的环境修改相应的参数。如果您不确定 `ORACLE_HOME` 和 `ORACLE_BASE` 的值应设置为何值，请参阅本章前面的“理解最佳灵活架构”部分，了解 `OFA` 标准目录的描述。

这些参数有时存在特定于某个版本的特性。例如，如果您不想指定您的 My Oracle Support (`MOS`) 登录信息，则需要按如下方式设置参数：
```
DECLINE_SECURITY_UPDATES=true
```

如果您未将 `DECLINE_SECURITY_UPDATES` 设置为 `TRUE`，则系统将要求您提供 `MOS` 登录信息。未能这样做将导致安装失败。配置好响应文件后，您可以在静默模式下运行 Oracle 安装程序。请注意，您必须输入响应文件位置的完整目录路径。

**注意**
在 Windows 上，`setup.exe` 命令等同于 Linux/Unix 上的 `runInstaller` 命令。

如果您在安装过程中遇到错误，可以查看相关的日志文件。每次尝试运行安装程序时，它都会创建一个具有唯一名称（包含时间戳）的日志文件。日志文件位于 `oraInventory/logs` 目录中。您可以在 `OUI` 写入日志文件时，将输出流式传输到您的屏幕：
```
$ tail -f 
```

以下是一个日志文件名示例：
```
installActions2012-04-33 11-42-52AM.log
```

##### Oracle Database 12c 第 1 版场景

导航到 `database` 目录并发出 `find` 命令以定位示例响应文件。以下是 Oracle Database 12c 第 1 版在 Linux 服务器上提供的响应文件：
```
$ find . -name "*.rsp"
./response/db_install.rsp
./response/netca.rsp
./response/dbca.rsp
```

复制其中一个响应文件以便修改。此示例将 `db_install.rsp` 文件复制到当前工作目录，并将文件命名为 `inst.rsp`：
```
$ cp response/db_install.rsp inst.rsp
```

修改 `inst.rsp` 文件。以下是 Oracle Database 12c 第 1 版响应文件的部分列表（前两行实际上是一行代码，但为了适应页面而放在了两行）。这些代码行是我唯一修改过的变量。我删除了注释，以便您更清楚地看到修改了哪些变量：
```
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v12.1.0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=oraserv1
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/home/oracle/orainst/12.1.0.1/database/stage/products.xml
SELECTED_LANGUAGES=en
ORACLE_HOME=/u01/app/oracle/product/12.1.0.1/db_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=oper
oracle.install.db.BACKUPDBA_GROUP=dba
oracle.install.db.DGDBA_GROUP=dba
oracle.install.db.KMDBA_GROUP=dba
DECLINE_SECURITY_UPDATES=true
```

请务必根据您的环境修改相应的参数。如果您不确定 `ORACLE_HOME` 和 `ORACLE_BASE` 的值应设置为何值，请参阅本章前面的“理解最佳灵活架构”部分，了解 `OFA` 标准目录的描述。

配置好响应文件后，您可以在静默模式下运行 Oracle 安装程序。请注意，您必须输入响应文件位置的完整目录路径：
```
$ ./runInstaller -ignoreSysPrereqs -force -silent -responseFile \
/home/oracle/orainst/12.1.0.1/database/inst.rsp
```

前面的命令分两行输入。第一行通过反斜杠 (`\`) 续接到第二行。

如果您在安装过程中遇到错误，可以查看相关的日志文件。每次尝试运行安装程序时，它都会创建一个具有唯一名称（包含时间戳）的日志文件。日志文件创建在 `oraInventory/logs` 目录中。您可以在 `OUI` 写入日志文件时，将输出流式传输到您的屏幕：
```
$ tail -f 
```

以下是一个日志文件名示例：
```
installActions2012-11-04_02-57-29PM.log
```

如果一切运行成功，输出中会通知您需要以 `root` 用户身份运行 `root.sh` 脚本：
```
/u01/app/oracle/product/12.1.0.1/db_1/root.sh
```

以 `root` 操作系统用户身份运行 `root.sh` 脚本。然后，您应该能够创建 Oracle 数据库（数据库创建在第 2 章中介绍）。


##### Oracle Database 18c Release 1 场景示例

导航至 `database` 目录并使用 `find` 命令查找示例响应文件。以下是 Oracle Database 18c Release 1 在 Linux 服务器上提供的响应文件：

```bash
$ find . -name "*.rsp"
./response/db_install.rsp
./response/netca.rsp
./response/dbca.rsp
```

复制其中一个响应文件以便修改。本例将 `db_install.rsp` 文件复制到当前工作目录并重命名为 `inst.rsp`：

```bash
$ cp response/db_install.rsp inst.rsp
```

修改 `inst.rsp` 文件。以下是 Oracle Database 18c Release 1 响应文件的部分内容（前两行实际是一行代码，为适应页面分为两行显示）。以下代码行是我唯一修改的变量。我移除了注释，以便您更清楚地看到哪些变量被修改：

```
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v18.0.0
oracle.install.option=INSTALL_DB_SWONLY
ORACLE_HOSTNAME=oraserv1
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/home/oracle/orainst/18.1.0.1/database/stage/products.xml
SELECTED_LANGUAGES=en
ORACLE_HOME=/u01/app/oracle/product/18.1.0.1/db_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=oper
oracle.install.db.BACKUPDBA_GROUP=dba
oracle.install.db.DGDBA_GROUP=dba
oracle.install.db.KMDBA_GROUP=dba
DECLINE_SECURITY_UPDATES=true
```

请务必根据您的环境修改相应的参数。如果您不确定 `ORACLE_HOME` 和 `ORACLE_BASE` 的值应设置为何值，请参阅本章前面的“理解最佳灵活架构”部分，了解 OFA 标准目录的描述。

配置好响应文件后，您可以以静默模式运行 Oracle 安装程序。请注意，您必须输入响应文件位置的完整目录路径：

```bash
$ ./runInstaller -ignoreSysPrereqs -force -silent -responseFile \
/home/oracle/orainst/18.1.0.1/database/inst.rsp
```

上述命令分两行输入。第一行通过反斜杠 (`\`) 连接到第二行。

如果在安装过程中遇到错误，您可以查看相关的日志文件。每次尝试运行安装程序时，它都会创建一个包含时间戳的唯一名称的日志文件。日志文件创建在 `oraInventory/logs` 目录中。当 OUI 写入日志时，您可以将其输出流式传输到屏幕上：

```bash
$ tail -f
```

以下是日志文件名的示例：

```
installActions2017-11-04_02-57-29PM.log
```

如果一切运行成功，输出中会通知您需要以 `root` 用户身份运行 `root.sh` 脚本：

```bash
/u01/app/oracle/product/18.1.0.1/db_1/root.sh
```

以 `root` 操作系统用户身份运行 `root.sh` 脚本。然后，您应该就能创建 Oracle 数据库了（数据库创建将在第 2 章介绍）。配置助手可以在响应文件或静默模式下运行，以执行网络配置和数据库配置助手。

### 步骤 7. 排查任何问题

如果在使用响应文件时遇到错误，90% 的情况是由于文件中变量设置方式有问题。请仔细检查这些变量并确保它们设置正确。此外，如果您没有完全指定响应文件的命令行路径，您会收到类似以下错误：

```
OUI-10203: The specified response file ... is not found.
```

当响应文件的路径或名称指定错误时，另一个常见错误如下：

```
OUI-10202: No response file is specified for this session.
```

接下来列出的是，如果您在响应文件的 `FROM_LOCATION` 变量中输入了错误的 `products.xml` 文件路径时会收到的错误消息：

```
OUI-10133: Invalid staging area
```

此外，请确保在运行响应文件时提供正确的命令行语法。如果您错误指定或拼错了某个选项，您可能会收到误导性的错误消息，例如 `DISPLAY not set`。使用响应文件时，您无需设置 `DISPLAY` 变量。此消息令人困惑，因为在此场景中，错误是由错误指定的命令行选项引起的，与 `DISPLAY` 变量无关。请检查从命令行输入的所有选项，确保您没有拼错选项。

当您指定了 `ORACLE_HOME`，而静默安装“认为”给定的主目录已存在时，也可能出现问题：

```
Check complete: Failed <<<<
Recommendation: Choose a new Oracle Home for installing this product.
```

检查您的 `inventory.xml` 文件（位于 `oraInventory/ContentsXML` 目录中），确保与现有 Oracle 主目录名称没有冲突。

安装过程中会生成日志文件，以及作为清单一部分的文件。`/tmp` 目录将包含基于安装执行时间戳的日志文件。在尝试排查问题时，请确保检查所有日志文件；即使在过程中遇到进程或内存问题，系统日志也很有用。当您对 Oracle 安装问题进行故障排除时，请记住安装程序使用两个关键文件来跟踪已安装的软件及其位置：`oraInst.loc` 和 `inventory.xml`。表 1-2 描述了 Oracle 安装程序使用的文件。

#### 表 1-2
用于排查 Oracle 安装问题的有用文件

| 文件名 | 目录位置 | 内容 |
| --- | --- | --- |
| `oraInst.loc` | 此文件的位置因操作系统而异。在 Linux 上，文件位于 `/etc`；在 Solaris 上，位于 `/var/opt/oracle`。 | `oraInventory` 目录位置和安装操作系统组 |
| `inst.loc` | `\\HKEY_LOCAL_MACHINE\\Software\Oracle` (Windows 注册表) | 清单信息 |
| `inventory.xml` | `oraInventory/ContentsXML/inventory.xml` | Oracle 主目录名称及相应的目录位置 |
| `.log` 文件 | `oraInventory/logs` | 安装日志文件，对于故障排除极其有用 |

### 步骤 8. 应用任何额外的补丁

正如第一步之前所述，Oracle 软件在基础发行版中可用。但是，如果有额外的发行版、补丁集和安全补丁可用，则应在推出新的 Oracle 二进制文件之前全部应用。安装在服务器或环境中的原因可能不同，但安装应与其他环境相同，安全补丁可能例外。

本章后面的“升级 Oracle 软件”和“应用临时补丁”部分详细介绍了如何应用补丁，但在此步骤中确保在软件投入使用前达到最新版本非常重要。在二进制文件安装完成后，是确保所有内容已更新并准备好创建数据库的绝佳时机。




## 通过复制现有安装进行安装

数据库管理员有时会使用诸如 `tar` 之类的实用程序，通过复制 Oracle 二进制文件的现有安装到另一台服务器（或同一服务器上的不同位置）来安装 Oracle 软件。这种方法快速而简单（尤其是与下载并运行 Oracle 安装程序相比）。该技术允许数据库管理员轻松地在多台服务器上安装 Oracle 软件，同时确保每次安装都完全相同。

交付媒体文件的新方式提供了在 `ORACLE_HOME` 目录中解压缩的方法，或者直接运行 `rpm` 软件包来进行安装。使用此方法的优势在于能够复制一组已打补丁的二进制文件。在将文件复制为静态副本期间，不得有任何数据库或 Oracle 进程正在运行。

使用现有的二进制文件副本来安装 Oracle 是一个两步过程：

1.  使用操作系统实用程序复制二进制文件。
2.  附加 Oracle 主目录。

这些步骤将在接下来的两节中详细介绍。

#### 提示

有关如何克隆现有 Oracle 安装的说明，请参阅 MOS 注释 300062.1。

### 步骤 1：使用操作系统实用程序复制二进制文件

您可以使用任何操作系统复制实用程序来执行此步骤。Linux/Unix 的 `tar`、`scp` 和 `rsync` 实用程序是数据库管理员常用的文件复制工具。此示例展示了如何使用 Linux/Unix 的 `tar` 实用程序将一组现有的 Oracle 二进制文件复制到另一台服务器。首先，定位您要复制的目标 Oracle 主目录二进制文件：

```bash
$ echo $ORACLE_HOME
/ora01/app/oracle/product/18.1.0.1/db_1
```

在此示例中，`tar` 实用程序复制 `db_1` 目录内或其下的所有文件和子目录：

```bash
$ cd $ORACLE_HOME
$ cd ..
$ tar -cvf orahome.tar db_1
```

现在，将 `orahome.tar` 文件复制到您要在其上安装 Oracle 软件的服务器。在此示例中，tar 文件被复制到另一台服务器上的 `/u01/app/oracle/product/18.1.0.1` 目录中。tar 文件在该目录下被提取，并作为提取过程的一部分创建一个 `db_1` 目录：

```bash
$ cd /u01/app/oracle/product/18.1.0.1
```

确保您有足够的磁盘空间来提取文件。典型的 Oracle 安装至少会占用 3-4GB 的空间。使用 Linux/Unix 的 `df` 命令验证您是否有足够的空间：

```bash
$ df -h | sort
```

接下来，提取文件：

```bash
$ tar -xvf orahome.tar
```

`tar` 命令会将文件提取到 `/u01/app/oracle/product/18.1.0.1` 目录下的 `db_1` 目录中。

#### 提示

使用 `tar -tvf <tarfile_name>` 命令可以在不恢复文件的情况下预览哪些目录和文件将被恢复。

下面列出了一个强大的单行组合命令，它允许您捆绑 Oracle 文件，将它们复制到远程服务器，并在远程进行提取：

```bash
$ tar -cvf -  | ssh  "cd ; tar -xvf -"
```

例如，以下命令将 `dev_1` 目录中的所有内容复制到远程 `ora03` 服务器的 `/home/oracle` 目录：

```bash
$ tar -cvf - dev_1 | ssh ora03 "cd /home/oracle; tar -xvf -"
```

### 绝对路径与相对路径

一些较旧的非 GNU 版本的 `tar` 在提取文件时会使用绝对路径。下一行代码展示了在创建归档文件时指定绝对路径的示例：

```bash
$ tar -cvf orahome.tar /home/oracle
```

对非 GNU 版本的 `tar` 指定绝对路径可能是危险的。这些旧版本的 `tar` 会按照它们被复制时相同的目录和文件名来恢复内容。这意味着磁盘上先前存在的任何目录和文件名都会被覆盖。

当使用旧版本的 `tar` 时，使用相对路径名要安全得多。此示例首先切换到 `/home` 目录，然后创建 `oracle` 目录的归档文件（相对于当前工作目录）：

```bash
$ cd /home
$ tar -cvf orahome.tar oracle
```

前面的示例使用了相对路径名。

在大多数 Linux 系统上，您不必担心绝对路径与相对路径的问题。这是因为这些系统使用 GNU 版本的 `tar`。此版本会剥离前导斜杠 (`/`)，并将文件恢复到相对于您当前工作目录的位置。

如果您不确定是否拥有 GNU 版本的 `tar` 实用程序，请使用 `man tar` 命令。您也可以使用 `tar -tvf <tarfile name>` 命令来预览哪些目录和文件将被恢复到什么位置。




## 步骤 2. 附加 Oracle 主目录

使用现有安装的副本来安装 Oracle 软件时存在一个问题：如果您后续尝试升级该软件，升级过程将会报错并中止。这是因为副本安装并未在 `oraInventory` 中注册。在升级通过副本安装的一组二进制文件之前，您必须先注册 Oracle 主目录，使其出现在 `inventory.xml` 文件中。这称为附加 Oracle 主目录。

要附加 Oracle 主目录，您需要知道服务器上 `oraInst.loc` 文件的位置。在 Linux 服务器上，此文件通常位于 `/etc` 目录。在 Solaris 上，此文件通常可以在 `/var/opt/oracle` 目录中找到。

定位到 `oraInst.loc` 文件后，导航至 `ORACLE_HOME/oui/bin` 目录（在您从副本安装 Oracle 二进制文件的服务器上）：

```
$ cd $ORACLE_HOME/oui/bin
```

现在，通过运行 `runInstaller` 实用程序来附加 Oracle 主目录，如下所示：

```
$ ./runInstaller -silent -attachHome -invPtrLoc /etc/oraInst.loc \
ORACLE_HOME="/u01/app/oracle/product/18.1.0.1/db_1" ORACLE_HOME_NAME="ONEW"
```

如果成功，您应该在输出的最后看到这条消息：

```
'AttachHome' was successful.
```

您也可以检查 `oraInventory/ContentsXML/inventory.xml` 文件的内容。以下是由于运行带有 `attachHome` 选项的 `runInstaller` 实用程序而插入到 `inventory.xml` 文件中的一行内容片段：

## 安装只读 Oracle 主目录

Oracle 18c 的一个新特性是为二进制文件提供只读的 Oracle 主目录。数据库工具和进程将位于 `ORACLE_BASE` 路径下，而不是 `ORACLE_HOME` 路径。`ORACLE_HOME` 目录将包含为创建的数据库准备的数据库配置和日志。

只读的 Oracle 二进制文件主目录将软件与数据库信息分离，并允许在多个部署之间共享软件。这使得二进制文件的无缝修补和更新成为可能，从而最大限度地减少了数据库停机时间，并允许将补丁应用到一个映像以分发到多台服务器。这种分离还简化了配置过程，因为可以专注于数据库配置。

现在有了额外的环境变量来包含 Oracle 主目录的目录路径，即 `ORACLE_BASE_HOME` 和 `ORACLE_BASE_CONFIG`。

要启用只读 Oracle 主目录，需要如前所述仅安装软件（二进制文件），而不安装配置助手。然后运行以下命令：

```
$ cd /u01/app/oracle/product/18.1.0.1/dbhome18c/bin
$ roohctl -enable
```

启用只读主目录后，可以运行 DBCA 来创建数据库。有一个检查可以了解数据库是否位于只读主目录中：

```
$ cd $ORACLE_HOME/bin
$ ./orabasehome
```

如果返回一个目录，则表示 Oracle 主目录是只读的。

## 升级 Oracle 软件

您也可以使用静默安装方法来升级 Oracle 软件的版本。首先从 MOS 网站 ([`http://support.oracle.com`](http://support.oracle.com)) 下载升级版本（您需要有效的支持合同才能执行此操作）。阅读新软件附带的升级文档。升级过程可能因您使用的 Oracle 版本而有很大差异。

根据我最近执行的升级经验，该过程与安装一组新的 Oracle 二进制文件非常相似。您可以使用 OUI（图形或静默模式）来安装软件。有关使用静默模式安装方法的信息，请参阅本章前面的“安装 Oracle”部分。

使用数据库升级助手 (`DBUA`) 可成功将数据库迁移到新版本。这将执行到最新版本数据库的迁移并升级服务。18c 的新特性是服务必须通过此方法升级：安装最新版本的软件，然后运行 `DBUA` 完成迁移。

> **注意**
> 升级 Oracle 软件不同于升级 Oracle 数据库。本节仅涉及使用静默安装方法升级 Oracle 软件。升级数据库涉及额外的步骤。有关如何升级数据库的说明，请参阅 MOS 笔记 730365.1。

根据要升级的版本，您可能会遇到两种不同的场景。以下是场景 A：

1.  关闭所有使用要升级的 Oracle 主目录的数据库。
2.  升级 Oracle 主目录二进制文件。
3.  启动数据库并运行任何必需的升级脚本。

以下是场景 B 升级方法的步骤：

1.  保持现有的 Oracle 主目录不变——不升级它。
2.  安装一个与旧 Oracle 主目录版本相同的新 Oracle 主目录。
3.  将新 Oracle 主目录升级到所需版本。
4.  当您准备就绪时，关闭使用旧 Oracle 主目录的数据库；设置操作系统变量以指向新的、已升级的 Oracle 主目录；启动数据库；并运行任何必需的升级脚本。

上述两种场景中哪一种更好？场景 B 的优点在于保持旧的 Oracle 主目录不变；因此，如果由于任何原因您需要切换回旧的 Oracle 主目录，那些二进制文件仍然可用。场景 B 的缺点是需要额外的磁盘空间来容纳两个 Oracle 主目录的安装。这通常不是问题，因为升级完成后，您可以在方便时删除旧的 Oracle 主目录。

在云环境中，使用基础设施即服务 (`IaaS`) 时，也会安装 Oracle 二进制文件，这使得服务器和数据库的配置非常高效。`IaaS` 允许 DBA 拥有对配置服务器的完全访问权限以执行数据库创建，但数据库二进制文件的安装被简化，并且类似于响应文件安装。许多 DBA 更习惯使用 Oracle 的图形安装程序来安装数据库软件。但是，当服务器位于远程位置或深嵌在安全网络中时，图形安装程序可能会很麻烦。缓慢的网络或安全功能会严重阻碍图形安装过程。在这些情况下，请确保正确配置所需的 X 软件和操作系统变量（如 `DISPLAY`）。

作为 DBA，精通 Oracle 安装过程至关重要。如果 Oracle 安装软件未正确安装，您将无法成功创建数据库。一旦正确安装了 Oracle，您就可以进入下一步：启动后台进程并创建数据库。接下来，在第 [2] 章中讨论启动 Oracle 以及发出和创建数据库的主题。

## 2. 创建数据库

第 [1] 章详细介绍了如何高效安装 Oracle 二进制文件。安装 Oracle 软件后，下一个逻辑任务是创建数据库。有几种创建 Oracle 数据库的标准方法：

*   使用数据库配置助手 (`dbca`) 实用程序。
*   从 SQL*Plus 运行 `CREATE DATABASE` 语句。
*   从现有数据库克隆数据库。

Oracle 的 `dbca` 实用程序具有图形界面，您可以通过它配置和创建数据库。这个可视化工具易于使用，界面非常直观。如果您需要创建开发数据库并快速开始工作，那么这个工具绰绰有余。话虽如此，我通常不使用 `dbca` 实用程序来创建数据库。在 Linux/Unix 环境中，`dbca` 工具依赖于 X 软件和操作系统 `DISPLAY` 变量的适当设置。因此，`dbca` 实用程序需要一些设置，如果您在远程服务器上安装且网络吞吐量较慢时，其性能可能会很差。

`dbca` 实用程序还允许您以静默模式创建数据库，无需图形组件。使用带有响应文件的 `dbca` 静默模式是创建一致且可重复数据库的高效方法。`dbca` 工具可以在二进制文件安装后以静默模式运行，也可以单独启动。当您在远程服务器上安装时，这种方法也很有效，因为远程服务器的网络连接可能较慢，或者可能未安装适当的 X 软件。

在远程服务器上创建数据库时，通常使用 SQL*Plus 更简单、更高效。SQL*Plus 方法简单且本质上是可编写脚本的。此外，无论网络连接速度有多慢，SQL*Plus 都能工作，并且它不依赖于图形组件。然而，`dbca` 实用程序允许在创建的数据库中快速采用新特性。本章首先向您展示如何使用 SQL*Plus 快速创建数据库，以及如何通过启用监听器进程使您的数据库可供远程访问。稍后，本章将演示如何使用带有响应文件的 `dbca` 静默模式来创建数据库。

## 设置操作系统变量

在创建数据库之前，您需要了解一些关于操作系统变量（通常称为环境变量）的知识。在运行 SQL*Plus（或任何其他 Oracle 实用程序）之前，您必须设置几个操作系统变量：

*   `ORACLE_HOME`
*   `ORACLE_SID`
*   `LD_LIBRARY_PATH`
*   `PATH`

`ORACLE_HOME` 变量定义了默认初始化文件位置的起始目录，在 Linux/Unix 上是 `ORACLE_HOME/dbs`。在 Windows 上，此目录通常是 `ORACLE_HOME\database`。`ORACLE_HOME` 变量也很重要，因为它定义了定位 Oracle 二进制文件（如 `sqlplus`、`dbca`、`netca`、`rman` 等）的起始目录，这些文件位于 `ORACLE_HOME/bin` 中。

`ORACLE_SID` 变量定义了您尝试创建的数据库的默认名称。`ORACLE_SID` 也用作参数文件的默认名称，即 `init<ORACLE_SID>.ora` 或 `spfile<ORACLE_SID>.ora`。

`LD_LIBRARY_PATH` 变量很重要，因为它指定了在 Linux/Unix 机器上搜索库的位置。此变量的值通常设置为包含 `ORACLE_HOME/lib`。

`PATH` 变量指定了当您从操作系统提示符键入命令时默认查找的目录。在几乎所有情况下，`ORACLE_HOME/bin`（Oracle 二进制文件的位置）都必须包含在您的 `PATH` 变量中。

您可以采用几种不同的方法来设置上述变量。本章讨论三种方法，从硬编码的手动方法开始，以我个人喜欢的方法结束：利用 `oratab` 文件。为什么要讨论不同的方法？因为了解环境配置各不相同很重要。理解需要这些步骤才能连接到数据库将有助于故障排除并验证二进制文件是否已安装并可用。由于策略和服务器配置的不同，也有不同的工具可用，例如进行静默安装与使用 UI。

### 手动方法

在 Linux/Unix 中，当您使用 Bourne、Bash 或 Korn shell 时，可以使用 `export` 命令从操作系统命令行手动设置操作系统变量：

```
$ export ORACLE_HOME=/u01/app/oracle/product/18.0.0.0/db_1
$ export ORACLE_SID=o12c
$ export LD_LIBRARY_PATH=/usr/lib:$ORACLE_HOME/lib
$ export PATH=$ORACLE_HOME/bin:$PATH
```

对于 C 或 tcsh shell，使用 `setenv` 命令设置变量：

```
$ setenv ORACLE_HOME 
$ setenv ORACLE_SID 
$ setenv LD_LIBRARY_PATH 
$ setenv PATH 
```

DBA 设置这些变量的另一种方法是将前面的 `export` 或 `setenv` 命令放入 Linux/Unix 启动文件中，例如 `.bash_profile`、`.bashrc` 或 `.profile`。这样，变量会在登录时自动设置。这只需编辑启动文件或配置文件插入变量即可完成。即使有其他选项，在启动文件中设置默认的 `ORACLE_HOME` 仍然是好的做法。

然而，手动设置操作系统变量（无论是从命令行还是通过将值硬编码到启动文件中）并不是实例化这些变量的最佳方式。例如，如果一台机器上有多个数据库和多个 Oracle 主目录，手动设置这些变量很快就会变得难以管理且不可维护。

### Oracle 的方法

设置操作系统变量的一种好得多的方法是使用脚本，该脚本利用一个包含服务器上所有 Oracle 数据库名称及其关联 Oracle 主目录的文件。这种方法灵活且可维护。例如，如果某个数据库的 `ORACLE_HOME` 发生变化（例如升级后），您只需在服务器上修改一个文件，而无需费力寻找 `ORACLE_HOME` 变量可能被硬编码到脚本中的位置。

Oracle 提供了一种自动设置所需操作系统变量的机制。Oracle 的方法依赖于两个文件：`oratab` 和 `oraenv`。

### 理解 oratab

您可以将 `oratab` 文件中的条目视为一个注册表，记录了机器上安装了哪些数据库及其对应的 Oracle 主目录。`oratab` 文件在您安装 Oracle 软件时会自动创建。在 Linux 机器上，`oratab` 通常放在 `/etc` 目录中。在 Solaris 服务器上，`oratab` 文件位于 `/var/opt/oracle` 目录中。如果由于某种原因 `oratab` 文件没有自动创建，您可以手动创建目录和文件。

`oratab` 文件在 Linux/Unix 环境中用于以下目的：

*   自动获取所需的操作系统变量
*   自动启动和停止服务器上的 Oracle 数据库

`oratab` 文件有三列，格式如下：

```
::Y|N
```

`Y` 或 `N` 表示您是否希望 Oracle 在系统重启时自动重启；`Y` 表示是，`N` 表示否。数据库的自动启动和关闭将在第 [20] 章中详细讨论。Oracle `srvctl` 也有管理策略，可以为不使用 `oratab` 的数据库设置自动重启。

`oratab` 文件中的注释以井号 (`#`) 开头。以下是一个典型的 `oratab` 文件条目：

```
o12c:/u01/app/oracle/product/18.0.0.0/db_1:N
rcat:/u01/app/oracle/product/18.0.0.0/db_1:N
```

前面几行中的数据库名称是 `o12c` 和 `rcat`。每行接下来是数据库 `ORACLE_HOME` 目录的路径（与数据库名称用冒号 [`:`] 分隔）。

几个 Oracle 提供的实用程序使用 `oratab` 文件：

*   `oraenv` 使用 `oratab` 来设置操作系统变量。
*   `dbstart` 使用它来在服务器重启时自动启动数据库（如果 `oratab` 中的第三个字段为 `Y`）。
*   `dbshut` 使用它来在服务器重启时自动停止数据库（如果 `oratab` 中的第三个字段为 `Y`）。

`oraenv` 工具将在下一节中讨论。

### 使用 oraenv

如果您没有为 Oracle 环境正确设置所需的操作系统变量，那么诸如 SQL*Plus、Oracle Recovery Manager (`RMAN`)、Data Pump 等实用程序将无法正常工作。`oraenv` 实用程序自动在 Oracle 数据库服务器上设置所需的操作系统变量（如 `ORACLE_HOME`、`ORACLE_SID` 和 `PATH`）。此实用程序用于 Bash、Korn 和 Bourne shell 环境（如果您在 C shell 环境中，有一个对应的 `coraenv` 实用程序）。

`oraenv` 实用程序位于 `ORACLE_HOME/bin` 目录中。您可以手动运行它，如下所示：

```
$ . oraenv
```

请注意，从命令行运行此命令的语法要求在点 (`.`) 和 `oraenv` 工具之间有一个空格。系统会提示您输入 `ORACLE_SID`，如果 `ORACLE_SID` 不在 `oratab` 文件中，它将提示输入 `ORACLE_HOME` 值：

```
ORACLE_SID = [oracle] ?
ORACLE_HOME = [/home/oracle] ?
```

您也可以在运行 `oraenv` 实用程序之前设置操作系统变量，以非交互方式运行。这在脚本编写中很有用，当您不想被提示输入时：

```
$ export ORACLE_SID=o18c
$ export ORAENV_ASK=NO
$ . oraenv
```

请记住，如果您将 `ORACLE_SID` 设置为在 `oratab` 文件中找不到的值，那么您可能会被提示输入诸如 `ORACLE_HOME` 之类的值。

### 我设置操作系统变量的方法

我不使用 Oracle 的 `oraenv` 文件来设置操作系统变量（有关 Oracle 方法的详细信息，请参阅上一节“使用 oraenv”）。相反，我使用一个名为 `oraset` 的脚本。`oraset` 脚本依赖于 `oratab` 文件位于正确的目录和预期的格式中：

```
::Y|N
```

如前一节所述，Oracle 安装程序应在正确的目录中为您创建一个 `oratab` 文件。如果没有，您可以手动创建并填充该文件。在 Linux 中，`oratab` 文件通常创建在 `/etc` 目录中。在 Solaris 服务器上，`oratab` 文件位于 `/var/opt/oracle` 目录中。

接下来，使用一个读取 `oratab` 文件并设置操作系统变量的脚本。以下是一个 `oraset` 脚本的示例，该脚本读取 `oratab` 文件并呈现一个选择菜单（基于 `oratab` 文件中的数据库名称）：

```
#!/bin/bash
```


#### 设置 Oracle 环境变量。

## 安装设置

1.  将 `oraset` 文件置于 `/etc`（Linux 系统）或 `/var/opt/oracle`（Solaris 系统）
2.  确保 `/etc` 或 `/var/opt/oracle` 路径已添加到 `$PATH` 变量中

## 使用方法

-   批处理模式：`. oraset`
-   菜单模式：`. oraset`

```
if [ -f /etc/oratab ]; then
OTAB=/etc/oratab
elif [ -f /var/opt/oracle/oratab ]; then
OTAB=/var/opt/oracle/oratab
else
echo 'oratab file not found.'
exit
fi
#
if [ -z $1 ]; then
SIDLIST=$(egrep -v '#|\*' ${OTAB} | cut -f1 -d:)
```



#### PS3 指示了用于 Bash `select` 命令的提示符。
`PS3='SID? '`
`select sid in ${SIDLIST}; do`
`if [ -n $sid ]; then`
`HOLD_SID=$sid`
`break`
`fi`
`done`
`else`
`if egrep -v '#|\*' ${OTAB} | grep -w "${1}:">/dev/null; then`
`HOLD_SID=$1`
`else`
`echo "SID: $1 not found in $OTAB"`
`fi`
`shift`
`fi`
#
`export ORACLE_SID=$HOLD_SID`
`export ORACLE_HOME=$(egrep -v '#|\*' $OTAB|grep -w $ORACLE_SID:|cut -f2 -d:)`
`export ORACLE_BASE=${ORACLE_HOME%%/product*}`
`export TNS_ADMIN=$ORACLE_HOME/network/admin`
`export ADR_BASE=$ORACLE_BASE/diag`
`export PATH=$ORACLE_HOME/bin:/usr/ccs/bin:/opt/SENSsshc/bin/\
:/bin:/usr/bin:.:/var/opt/oracle:/usr/sbin`
`export LD_LIBRARY_PATH=/usr/lib:$ORACLE_HOME/lib`

你可以从命令行或启动文件（如`.profile`、`.bash_profile`或`.bashrc`）中运行`oraset`脚本。要从命令行运行`oraset`，请将`oraset`文件放置在一个标准位置，例如`/var/opt/oracle`（Solaris）或`/etc`（Linux），并按如下方式运行：

```
$ . /etc/oraset
```

请注意，从命令行运行此命令的语法要求在点号（`.`）和命令的其余部分之间有一个空格。当你从命令行运行`oraset`时，系统应显示如下菜单：

```
1) o18c
2) rcat
SID?
```

在此示例中，你现在可以输入`1`或`2`来设置你想要使用的数据库所需的操作系统变量。这允许你以交互方式设置操作系统变量，而不管服务器上有多少个数据库安装。

你也可以从操作系统启动文件中调用`oraset`文件。以下是在`.bashrc`文件中的一个示例条目：

```
. /etc/oraset
```

现在，每次登录服务器时，系统都会显示一个选项菜单，你可以使用该菜单来指定要为其设置操作系统变量的数据库。如果你想将操作系统变量自动设置为特定数据库，那么请在`.bashrc`文件中添加如下条目：

```
. /etc/oraset o18c
```

前面这行将为`o18c`数据库运行`oraset`文件，并相应地设置操作系统变量。

## 创建数据库

本节说明如何使用 SQL*Plus 的 `CREATE DATABASE` 语句手动创建 Oracle 数据库。创建数据库需要以下步骤：

1.  设置操作系统变量。
2.  配置初始化文件。
3.  创建所需的目录。
4.  创建数据库。
5.  创建数据字典。

接下来的部分将详细介绍这些步骤。

### 步骤 1. 设置操作系统变量

如前所述，在运行 SQL*Plus（或任何其他 Oracle 实用程序）之前，必须设置几个操作系统变量。你可以手动设置这些变量，也可以结合使用文件和脚本来设置变量。以下是手动设置这些变量的示例：

```
$ export ORACLE_HOME=/u01/app/oracle/product/18.0.0.0/db_1
$ export ORACLE_SID=o18c
$ export LD_LIBRARY_PATH=/usr/lib:$ORACLE_HOME/lib
$ export PATH=$ORACLE_HOME/bin:$PATH
```

有关这些变量以及设置技术的完整说明，请参阅本章前面的“设置操作系统变量”部分。

### 步骤 2. 配置初始化文件

Oracle 要求在尝试启动实例之前必须有一个初始化文件。初始化文件用于配置诸如内存和控制文件位置等特性。你可以使用两种类型的初始化文件：

*   服务器参数二进制文件 (`spfile`)
*   `init.ora` 文本文件

Oracle 建议你使用 `spfile`，原因如下：

*   你可以使用 SQL `ALTER SYSTEM` 语句修改 `spfile` 的内容。
*   你可以使用远程客户端 SQL 会话来启动数据库，而不需要本地（客户端）初始化文件。
*   有更多可以使用 spfile 设置的动态参数，而无需任何停机时间。

这些是使用 `spfile` 的充分理由。然而，一些公司仍然使用传统的 `init.ora` 文件。`init.ora` 文件也有其优势：

*   你可以直接用操作系统文本编辑器编辑它。
*   你可以在其中添加注释，详细记录修改历史。

当我第一次创建数据库时，我发现使用 `init.ora` 文件更容易。如果需要，这个文件以后可以轻松转换为 `spfile`（通过 `CREATE SPFILE FROM PFILE` 语句）。在此示例中，我的数据库名称是 `o18c`，因此我将以下内容放入一个名为 `inito18c.ora` 的文件中，并将该文件放在 `ORACLE_HOME/dbs` 目录下：

```
db_name=o18c
db_block_size=8192
memory_target=300M
memory_max_target=300M
processes=200
control_files=(/u01/dbfile/o18c/control01.ctl,/u02/dbfile/o18c/control02.ctl)
job_queue_processes=10
open_cursors=500
fast_start_mttr_target=500
undo_management=AUTO
undo_tablespace=UNDOTBS1
remote_login_passwordfile=EXCLUSIVE
```

确保初始化文件命名正确并位于适当的目录中。这一点至关重要，因为在启动实例时，Oracle 首先会在 `ORACLE_HOME/dbs` 目录中按以下特定格式和顺序查找参数文件：

*   `spfile<SID>.ora`
*   `spfile.ora`
*   `init<SID>.ora`

换句话说，Oracle 首先查找名为 `spfile<SID>.ora` 的文件。如果找到，则启动实例；如果未找到，Oracle 会查找 `spfile.ora`，然后查找 `init<SID>.ora`。如果这些文件都未找到，Oracle 将抛出错误。如果你不了解 Oracle 查找的文件及其顺序，这可能会导致一些困惑。例如，你可能更改了 `init<SID>.ora` 文件，并期望在停止并启动实例后生效。但如果存在 `spfile<SID>.ora`，则 `init<SID>.ora` 将被完全忽略。

> 注意：你可以使用 `startup` 命令的 `pfile=<directory/filename>` 子句，手动指示 Oracle 在特定目录中查找文本参数文件；在正常情况下，你不需要这样做。你希望使用默认行为，即让 Oracle 在 `ORACLE_HOME/dbs` 目录中查找参数文件（对于 Linux/Unix）。Windows 上的默认目录是 `ORACLE_HOME/database`。

表 2-1 列出了配置 Oracle 初始化文件时需要考虑的最佳实践。

#### 表 2-1：初始化文件最佳实践

| 最佳实践 | 理由 |
| --- | --- |
| Oracle 建议你使用二进制服务器参数文件 (`spfile`)。 | Spfile 允许动态更改参数。如果有可接受的维护窗口，那么使用 init.ora 文件也可以。 |
| 一般来说，如果不确定初始化参数的用途，就不要设置它们。如有疑问，请使用默认值。 | 设置初始化参数可能会对数据库性能产生深远影响。只有在知道最终行为的情况下才修改参数。 |
| 对于 11g 及更高版本，设置 `memory_target` 和 `memory_max_target` 初始化参数。 | 这样做允许 Oracle 为你管理所有内存组件。 |
| 对于 10g，设置 `sga_target` 和 `sga_target_max` 初始化参数。 | 这样做可以让 Oracle 为你管理大部分内存组件。 |
| 对于 10g，设置 `pga_aggregate_target` 和 `workarea_size_policy`。 | 这样做允许 Oracle 管理用于排序空间的内存。 |
| 从 10g 开始，使用自动 `UNDO` 功能。这通过 `undo_management` 和 `undo_tablespace` 参数设置。 | 这样做允许 Oracle 管理 `UNDO` 表空间的大部分特性。 |
| 将 `open_cursors` 设置为比默认值更高的值。我通常将其设置为 500。活跃的在线事务处理 (OLTP) 数据库可能需要高得多的值。 | 默认值 50 几乎从来不够。即使是一个单用户的小型应用程序也可能超过 50 个开放游标的默认值。 |
| 使用 `/<mount_point>/dbfile/<database_name>/control0N.ctl` 模式命名控制文件。 | 这与 OFA 标准略有不同。我发现这个位置比位于 `ORACLE_BASE` 下更容易导航。 |
| 至少使用两个控制文件，最好位于不同的位置，使用不同的磁盘。 | 如果一个控制文件损坏，至少还有另一个可用的控制文件总是一个好主意。 |

### 步骤 3. 创建所需的目录

在尝试创建数据库之前，必须在服务器上创建参数文件或 `CREATE DATABASE` 语句中引用的任何操作系统目录。例如，在上一节的初始化文件中，控制文件定义为：

```
control_files=(/u01/dbfile/o18c/control01.ctl,/u02/dbfile/o18c/control02.ctl)
```

根据上一行，确保你已经创建了目录 `/u01/dbfile/o18c` 和 `/u02/dbfile/o18c`（请根据你的环境进行修改）。在 Linux/Unix 中，你可以使用带有 `-p` 开关的 `mkdir` 命令创建目录（包括所需的任何父目录）：

```
$ mkdir -p /u01/dbfile/o18c
$ mkdir -p /u02/dbfile/o18c
```

同时，确保你为 `CREATE DATABASE` 语句中引用的数据文件和联机重做日志创建了所需的目录（参见步骤 4）。对于此示例，以下是所需的附加目录：

```
$ mkdir -p /u01/oraredo/o18c
$ mkdir -p /u02/oraredo/o18c
```

如果你以 `root` 用户身份创建了上述目录，请确保 `oracle` 用户和 `dba` 组被正确设置为拥有这些目录、子目录和文件。此示例递归地更改以下目录的所有者和组：


#### chown -R oracle:dba /u01
#### chown -R oracle:dba /u02

