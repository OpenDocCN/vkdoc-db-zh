# 第 1 章 ■ 安装 Oracle 二进制文件

## Oracle 清单目录

Oracle 清单目录存储服务器上安装的 Oracle 软件清单。此目录是必需的，并且在服务器上所有 Oracle 软件安装之间共享。当您首次安装 Oracle 时，安装程序会检查是否存在格式为 `/u[01–09]/app` 的现有符合 OFA 标准的目录结构。如果存在这样的目录，则安装程序会创建 Oracle 清单目录，例如 `/u01/app/oraInventory`。
如果为 `oracle` 操作系统用户定义了 `ORACLE_BASE` 变量，则安装程序会为 Oracle 清单的位置创建如下目录：`ORACLE_BASE/../oraInventory`。
例如，如果 `ORACLE_BASE` 定义为 `/ora01/app/oracle`，则安装程序将 Oracle 清单的位置定义为 `/ora01/app/oraInventory`。
如果安装程序找不到可识别的符合 OFA 标准的目录结构或 `ORACLE_BASE` 变量，则 Oracle 清单的位置将在 `oracle` 用户的 `HOME` 目录下创建。例如，如果 `HOME` 目录是 `/home/oracle`，则 Oracle 清单的位置是 `/home/oracle/oraInventory`。



## Oracle 基础目录

Oracle 基础目录是 Oracle 软件安装的顶层目录。你可以在此目录下安装一个或多个版本的 Oracle 软件。Oracle 基础目录的 OFA 标准如下：

`/<挂载点>/app/<软件所有者>`

所使用的挂载点通常命名为类似 `/u01`、`/ora01`、`/oracle` 或 `/oracle01`。你可以根据环境的任何标准来命名此挂载点。我更喜欢使用类似 `/ora01` 这样的挂载点名称。它很短，并且当查看数据库服务器上的挂载点时，我能立即分辨出哪些挂载点用于 Oracle 数据库。此外，较短的挂载点名称在查询数据字典以报告数据库物理方面时也更易于使用。再者，通过操作系统命令导航目录时，较短的挂载点名称可以减少输入量。

软件所有者通常命名为 `oracle`。这是你用于安装 Oracle 软件（二进制文件）的操作系统用户。一个完整的 Oracle 基础目录路径类似于：

`/ora01/app/oracle`

## Oracle 家目录

Oracle 家目录定义了特定产品（如 Oracle Database 11g、Oracle Database 10g 等）的软件安装位置。你必须将不同产品或同一产品的不同发行版安装到单独的 Oracle 家中。推荐的符合 OFA 标准的 Oracle 家目录如下：

`ORACLE_BASE/product/<版本>/<安装名称>`

在上一行代码中，版本类似于 `11.2.0.1` 或 `10.2.0.1`。`install_name` 值类似于 `db_1`、`devdb1`、`test2` 或 `prod1`。以下是一个 11.2 数据库的 Oracle 家名称示例：

`/ora01/app/oracle/product/11.2.0.1/db_1`

**注意** 有些 DBA 不喜欢 `ORACLE_HOME` 目录末尾的 `db_1` 字符串，认为没有必要。`db_1` 存在的原因是，你可能有两个独立的二进制文件安装：一个开发安装和一个测试安装。如果你的环境不需要这种配置，可以随意去掉末尾的额外字符串。

## Oracle 网络文件目录

一些 Oracle 实用程序使用 `TNS_ADMIN` 的值来定位网络配置文件。此目录被定义为 `ORACLE_HOME/network/admin`。此目录通常包含 `tnsnames.ora` 和 `listener.ora` Oracle Net 文件。

## 自动诊断仓库

从 Oracle Database 11g 开始，`ADR_HOME` 目录指定了与 Oracle 相关的诊断文件的位置。这些文件对于解决 Oracle 数据库的问题至关重要。此目录被定义为 `ORACLE_BASE/diag/rdbms/dbname/instname`，其中 `dbname` 是你的数据库名称，`instname` 是你的实例名称。在单实例环境中，数据库名称和实例名称相同，不同之处在于数据库名称为小写，而实例名称为大写。例如，在下一行中，数据库名称由 `o11r2` 指定，而实例名称指定为 `O11R2`：

`/ora01/app/oracle/diag/rdbms/o11r2/O11R2`

现在你了解了 OFA 标准，也就明白了在安装 Oracle 二进制文件时如何使用它。例如，在运行 Oracle 安装程序时，你需要为 `ORACLE_BASE` 和 `ORACLE_HOME` 目录指定目录值。

## 安装 Oracle

假设你是新入职的，你的经理问你在一台服务器上安装一套新的 Oracle Database 11g 二进制文件需要多长时间。你回答说需要不到一个小时。你的老板表示怀疑，并指出之前的 DBA 在新服务器上安装 Oracle 二进制文件（软件）时总是估计至少需要一天。你回答说：“实际上，它没那么复杂，但 DBA 们确实倾向于高估安装时间，因为很难预测所有可能出现的问题。”

当你拿到一台新服务器并被要求安装 Oracle 二进制文件时，这通常指的是下载和安装在创建 Oracle 数据库之前所需的软件的过程。这个过程涉及几个步骤：

1.  创建操作系统 `dba` 组和操作系统 `oracle` 用户。
2.  确保操作系统已为 Oracle 数据库进行了充分配置。
3.  从 Oracle 获取数据库安装软件。
4.  解压缩数据库安装软件。
5.  配置响应文件，并运行 Oracle 静默安装程序。
6.  对任何问题进行故障排除。

以下小节将详细介绍这些步骤。

**注意** 任何 Oracle 指定为基础版本（`10.1.0.2`、`10.2.0.1`、`11.1.0.6` 和 `11.2.0.1`）的数据库版本都可以从 Oracle 的技术网络网站 (`http://otn.oracle.com`) 免费下载。但是请注意，任何后续的补丁下载都需要购买许可证。换句话说，下载基础软件需要 Oracle Technology Network (OTN) 登录（免费），而下载补丁集则需要 My Oracle Support 帐户（付费支持帐户）。

### 步骤 1：创建操作系统组和用户

如果你在一个拥有优秀系统管理员（SA）的公司工作，那么步骤 1 和 2 通常由 SA 执行。如果你没有 SA，那么你就必须自己执行这些步骤（在需要你履行许多不同工作职能的小公司中，这种情况很常见）。你需要 `root` 访问权限来完成这些步骤。

以 `root` 身份，使用 `groupadd` 命令添加操作系统组。在 Linux/Unix 环境中，Oracle 建议你创建三个操作系统组：`oinstall`、`dba` 和 `oper`。`oinstall` 组拥有操作安装文件以及执行安装和升级的权限。分配给 `dba` 组的操作系统用户可以以 `sysdba` 数据库特权连接到数据库。分配给 `oper` 组的操作系统用户可以以 `sysoper` 数据库特权连接到数据库。

设置三个独立组背后的理念是，能够允许不同的操作系统用户执行各种数据库任务。例如，你可能有一个通常安装 Oracle 软件并被分配到 `oinstall` 组的用户，以及其他需要 `sysdba` 或 `sysoper` 权限的用户，他们分别被分配到 `dba` 和 `oper` 组。

话虽如此，许多公司只创建一个操作系统组 `dba`，并使用该组来安装软件和执行所有 `sysdba` 和 `sysoper` 功能。我通常只使用一个组 (`dba`)。

如果你有 `root` 访问权限，可以像这样运行 `groupadd` 命令：

```
# groupadd dba
```

如果你无法访问 `root` 帐户，那么你需要让你的系统管理员运行之前的命令。如果你的公司要求在不同服务器上设置具有相同组 ID 的组，请使用 `-g` 选项。此示例将组 ID 显式设置为 `505`：

```
# groupadd -g 505 dba
```

你可以通过检查 `/etc/group` 文件的内容来验证组是否已成功添加。这是在 `/etc/group` 文件中创建的典型条目：

`dba:x:501:`

如果出于任何原因需要删除组，请使用 `groupdel` 命令。如果需要修改组，请使用 `groupmod` 命令。

接下来，使用 `useradd` 命令添加操作系统用户。此命令需要 `root` 访问权限。

以下命令创建一个名为 `oracle` 的操作系统帐户，主组为 `dba`，并指定 `oinstall` 组为附加组：

```
# useradd -g dba -G oinstall oracle
```

如果你无法访问 `root` 帐户，那么你需要你的系统管理员运行 `useradd` 命令。如果你的公司要求在多个服务器上设置具有相同用户 ID 的用户，请使用 `-u` 选项。此示例将用户 ID 显式设置为 `500`：



`# useradd -u 500 -g dba -G oinstall oracle`

您可以通过查看 `/etc/passwd` 文件来验证用户账户信息。运行 `useradd` 命令后，您预期会看到以下内容：

`oracle:x:500:500::/home/oracle:/bin/bash`

您也可以使用 `id` 命令来显示操作系统用户和组信息：`$ id`

`uid=500(oracle) gid=500(dba) groups=500(dba)`

在大多数 Linux 系统上，`HOME` 变量的默认值是 `/home/oracle`，默认的 shell 是 Bash shell。您可以按如下方式显示 `HOME` 和 `SHELL` 的值：`$ echo $HOME`

`$ echo $SHELL`

如果需要修改用户，请使用 `usermod` 命令。如果需要删除操作系统用户，请使用 `userdel` 命令。运行 `userdel` 命令需要 root 权限。此示例从服务器中删除 `oracle` 用户：

`# userdel oracle`

## 步骤 2. 确保操作系统已充分配置

与此步骤相关的任务因每个数据库版本和操作系统而有所不同。您必须参考针对数据库版本和操作系统供应商的 Oracle 安装手册，以获取确切要求。要执行此步骤，您需要验证和配置操作系统组件，例如：

- 内存和交换空间
- 系统架构（处理器）
- 可用磁盘空间（Oracle 现在安装几乎需要 5GB 空间）
- 操作系统版本和内核
- 操作系统软件（所需的软件包和补丁）

运行以下命令以确认 Linux 服务器上的内存大小：`$ grep MemTotal /proc/meminfo`

要验证内存和交换空间的大小，请运行以下命令：

`$ free -t`

要验证 `/tmp` 目录的空间大小，请输入此命令：`$ df -h /tmp`

要显示可用磁盘空间量，请执行此命令：

`$ df -h`

要验证操作系统版本，请输入此命令：

`$ cat /proc/version`

要验证内核信息，请运行以下命令：

`$ uname -r`

要确定是否已安装所需的软件包，请执行以下命令并提供所需的软件包名称：

`$ rpm -q <package_name>`

再次强调，数据库服务器的要求因操作系统和数据库版本而有很大差异。您可以从 Oracle 的网站 `www.oracle.com/documentation` 下载具体的安装手册。

■ **注意** OUI（Oracle 通用安装程序）会显示操作系统软件和硬件的任何不足。运行安装程序将在“步骤 5”部分介绍。

## 步骤 3. 获取 Oracle 安装软件

通常，获取 Oracle 软件最简单的方法是从 Oracle 软件网站下载：`www.oracle.com/technology/software`。导航到软件下载页面，并下载适合您要安装软件的操作系统和硬件类型（Linux、Solaris、Windows 等）的 Oracle 数据库版本。

## 步骤 4. 解压缩文件

在解压缩文件之前，我建议您创建一个标准目录来放置 Oracle 安装介质。您应该这样做有几个原因：当您一周、一个月或一年后回到这台机器时，您会希望能够轻松找到安装介质。

标准的目录结构可以帮助您快速组织和理解机器上已安装或未安装的内容。

创建一组标准目录来存放用于安装 Oracle 软件的文件。我喜欢将安装介质存储在诸如 `/ora01/orainst` 的目录中，然后在其中为安装在机器上的每个 Oracle 软件版本创建一个子目录：

`$ mkdir -p /ora01/orainst/10.2.0.1`

`$ mkdir -p /ora01/orainst/10.2.0.4`

`$ mkdir -p /ora01/orainst/11.2.0.1`

现在，将安装文件移动到相应的目录，并在那里解压缩它们：

`$ mv linux_11gR2_database_1of2.zip /ora01/orainst/11.2.0.1`

`$ mv linux_11gR2_database_2of2.zip /ora01/orainst/11.2.0.1`

使用 `unzip` 命令解压缩 zip 文件。Oracle Database 11 *g* release 2 软件的解压缩操作如下所示：

```
$ unzip linux_11gR2_database_1of2.zip
$ unzip linux_11gR2_database_2of2.zip
```

在某些 Oracle 安装中，您可能会发现分发文件是以压缩的 cpio 文件形式提供的。您可以使用一个命令进行解压缩和解包，如下所示：

`$ cat 10gr2_db_sol.cpio.gz | gunzip | cpio -idvm`

## 步骤 5. 配置响应文件并运行安装程序

您可以在两种模式之一运行 OUI：图形化模式或静默模式。通常，DBA 使用图形化安装程序。

然而，我强烈偏好使用静默安装选项，原因如下：静默安装不需要 X Window System 软件的可用性。

您可以避免远程图形化安装可能遇到的性能问题，因为尝试在本地绘制屏幕时速度可能极慢。

静默安装可以编写脚本并实现自动化。这意味着每次安装都可以按照相同的一致标准执行，无论哪个团队成员执行安装（我甚至让系统管理员以此方式安装 Oracle 二进制文件）。

执行静默安装的关键是使用响应文件。解压缩 Oracle 软件后，查找 Oracle 提供的示例响应文件：

`$ find . -name "*.rsp"`

根据 Oracle 的版本和操作系统平台，您找到的响应文件的名称和数量可能会有很大不同。接下来的两个小节展示了两种场景：Oracle Database 10 *g* release 2 和 Oracle Database 11 *g* release 2 的静默安装。

#### Oracle Database 10g 场景

Oracle 安装软件附带了各种默认响应文件。这些文件因数据库版本和操作系统而异。例如，以下是 Oracle Database 10 *g* release 2 (Solaris) 环境中提供的响应文件：

`$ find . -name "*.rsp"`

`./response/emca.rsp`

`./response/custom.rsp`

`./response/enterprise.rsp`

`./response/dbca.rsp`

`./response/netca.rsp`

`./response/standard.rsp`

`./install/response/se.rsp`

`./install/response/pe.rsp`

`./install/response/ee.rsp`

`./inst.rsp`

将 `enterprise.rsp` 文件复制到当前工作目录，并为其指定一个不同的名称（这样您始终可以参考原始文件）：

`$ cp response/enterprise.rsp inst.rsp`

使用操作系统实用程序（如 `vi`）编辑 `inst.rsp` 文件，并为响应文件变量提供值。以下是典型的 Oracle Database 10 *g* 安装需要提供的最小值：

```
RESPONSEFILE_VERSION=2.2.1.0.0
# 安装组，我使用 dba，许多其他人使用 oinstall
UNIX_GROUP_NAME=dba
FROM_LOCATION="/ora01/orainst/10.2.0.1/stage/products.xml"
n_configurationOption=3
s_nameForDBAGrp=dba
s_nameForOPERGrp=dba
ORACLE_HOME=/ora01/app/oracle/product/10.2.0.4/db_1
ORACLE_HOME_NAME=OHOME10
```

■ **注意** 有关所有响应文件变量的完整说明，请参阅 Oracle OTN 网站上提供的 Oracle 通用安装程序指南。

在此场景中，我不修改任何其他响应文件值。我让它们保持空白或保留已指定的默认值。现在，以静默模式运行 `runInstaller` 实用程序，并提供响应文件位置的完整目录路径：

`$ ./runInstaller -ignoreSysPrereqs -force -silent -responseFile /ora01/orainst/10.2.0.1/inst.rsp`

■ **注意** 在 Windows 上，`setup.exe` 命令等同于 Linux/Unix 的 `runInstaller` 命令。

安装程序会显示相当多的输出。如果所有系统检查都通过，安装 Oracle 软件需要几分钟时间。成功完成后，您应该会看到此消息：需要以 root 用户身份执行以下配置脚本 `/ora01/app/oracle/product/10.2.0.4/db_1/root.sh` 来配置系统。

以 root 身份登录，并运行 `root.sh`：


