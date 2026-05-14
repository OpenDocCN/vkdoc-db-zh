# 10. Oracle 身份管理配置

Oracle 身份管理器（OIM）提供了一个用户界面，管理员可以使用它来创建和管理用户账户、设置角色、配置工作流等。它还可以与 Oracle 访问管理器（OAM）集成，以提供用户自助服务界面，允许最终用户重置忘记的密码、管理其个人信息并设置安全提示问题。这一切都为组织构建了一个完整的身份生命周期管理环境。

第 7 章介绍了为 OIM 配置新域的步骤。那仅仅设置了在 WebLogic Server 中运行 OIM 所必需的结构，并部署了必要的组件。本章将指导您完成运行身份管理器工具所需的实际配置。

### 预配置步骤

创建新的 OIM 域后，在实际配置身份管理器服务器之前，还必须完成几个步骤。首先通过升级 Oracle 平台安全服务（OPSS）模式来开始预配置。

将使用 Oracle Fusion Middleware 补丁集助手来升级 OPSS。您可以在目录 `/home/oracle/IDMMiddleware/oracle_common/bin/psa` 中找到补丁集助手的可执行文件。这将启动一个图形用户界面（GUI）来引导您完成该过程。图 10-1 显示了补丁集助手。

![A352855_1_En_10_Fig1_HTML.jpg](img/A352855_1_En_10_Fig1_HTML.jpg)

图 10-1. 启动补丁集助手

在“选择组件”屏幕上，选中 Oracle Platform Security Services 复选框。

补丁集助手将要求您确认已完成数据库备份，并且所使用的数据库版本已通过 Fusion Middleware 认证。如图 10-2 所示。如果您尚未检查 Oracle 的认证矩阵，请立即检查。完成后，选中复选框，然后单击“下一步”。

![A352855_1_En_10_Fig2_HTML.jpg](img/A352855_1_En_10_Fig2_HTML.jpg)

图 10-2. 检查先决条件

在 OPSS 模式屏幕上，您需要输入环境中数据库的连接详细信息。输入的字段和值如图 10-3 所示。

![A352855_1_En_10_Fig3_HTML.jpg](img/A352855_1_En_10_Fig3_HTML.jpg)

图 10-3. 连接详细信息

有关字段的描述以及每个字段应输入的数据，请参见表 10-1。

表 10-1. OPSS 模式升级连接详细信息

| `数据库类型` | Oracle Database |
| --- | --- |
| `连接字符串` | 连接字符串应按以下格式输入： hostname:port:servicename |
| `DBA 用户名` | 此字段中输入的用户必须具有 sysdba 权限。按“sys as sysdba”格式输入 |
| `DBA 密码` | 上述用户的密码 |
| `模式用户名` | 这应该是使用存储库创建实用程序（RCU）创建的 OPSS 模式名称 |
| `模式密码` | 输入在 RCU 创建期间设置的密码 |

在实际升级 OPSS 模式之前，补丁集助手将验证数据库和模式对象，以确保它们满足要求。图 10-4 显示了验证过程。需要注意的是，如果 OPSS 已经是正确版本，或者不满足最低要求，此屏幕会告知您该情况。

![A352855_1_En_10_Fig4_HTML.jpg](img/A352855_1_En_10_Fig4_HTML.jpg)

图 10-4. 检查要升级的模式

补丁集助手将检查模式并验证其升级状态，然后为您提供将要升级的环境摘要。请花时间查看图 10-5 所示的“升级摘要”屏幕，然后再继续。

![A352855_1_En_10_Fig5_HTML.jpg](img/A352855_1_En_10_Fig5_HTML.jpg)

图 10-5. 确认升级组件

图 10-6 显示了补丁集助手的进度指示器。一旦显示 100%，您可以单击“下一步”继续。

![A352855_1_En_10_Fig6_HTML.jpg](img/A352855_1_En_10_Fig6_HTML.jpg)

图 10-6. 升级进度

在过程结束时，您将看到“升级成功”屏幕，如图 10-7 所示。如果遇到任何错误，请在继续之前解决它们。

![A352855_1_En_10_Fig7_HTML.jpg](img/A352855_1_En_10_Fig7_HTML.jpg)

图 10-7. 升级完成

现在 OPSS 模式已成功升级，您可以继续下一步，即设置数据库安全存储。

### 配置数据库安全存储

升级 OPSS 数据库模式后，您还必须为新的 OIM 实例配置数据库安全存储。Oracle 身份与访问管理仅支持数据库安全存储。因此，以下步骤将帮助您配置此存储。

您在配置 OAM 时已执行过相同的活动。如果在共享同一数据库的单一域中配置 OAM 和 OIM，您将需要遵循一些步骤，使 OIM 加入访问管理器数据库安全存储。但是，如果您遵循了本书介绍的步骤，则是在分离的域中安装了 OAM 和 OIM。存储库对象也安装在单独的数据库中。这要求您为每个产品创建单独的安全存储。

`configureSecurityStore.py` 脚本位于 OIM 主目录下的 `common/tools` 中。要运行该脚本，您需要使用位于 Middleware Home 目录下 `oracle_common/common/tools` 中的 `wlst.sh`。

如前所述，如果您在单一域中安装了 OIM 和 OAM，并且已经为访问管理器运行过此操作，那么您将需要使用 `–m join` 选项。但在本例中，您将使用 `-m create` 选项在 OIM 数据库存储库中创建新的安全存储。

要配置新的安全存储，请从 `/home/oracle/IDMMiddleware/oracle_common/common/tools` 运行以下命令。

```
./ wlst.sh /home/oracle/IDMMiddleware/Oracle_IDM1/common/tools/configureSecurityStore.py -d /home/oracle/IDMMiddleware/user_projects/idm_domain -c IAM -p Password123 -m create
```

在此命令中，您需要指定 `configureSecurityStore.py` 文件的位置以及 OIM 域的位置。将 IAM 指定为组件，并输入 OPSS 模式密码。


### 为 OIM 预配置 OID 身份存储

由于计划包含集成 OIM 和 OAM，因此必须在 OIM 服务器内启用轻量目录访问协议 (LDAP) 同步。在配置过程中将会有一步启用 LDAP 同步。但是，为该步骤做准备，必须在继续之前完成先决条件步骤。这确保您的 LDAP 存储，即 OID，将准备好用作 OIM 身份存储。这些说明适用于使用 OID。Oracle 支持其他 LDAP 存储，例如 Oracle Unified Directory、Oracle Virtual Directory、Active Directory 和 iPlanet/ODSEE，但这些不在本书讨论范围内。

继续之前，按如下方式设置环境变量：

```bash
MW_HOME=/home/oracle/OIDMiddleware
ORACLE_INSTANCE=/home/oracle/OIDMiddleware/asinst_1
ORACLE_HOME=/home/oracle/OIDMiddleware/Oracle_IDM1
```

要将 OID 预配置为 OIM 的 LDAP 身份存储，首先创建一个名为 `oidContainers.ldif` 的文件，内容如下：

```ldif
dn: cn=oracleAccounts,dc=mycompany,dc=com
cn: oracleAccounts
objectClass: top
objectClass: orclContainer
dn: cn=Users,cn=oracleAccounts,dc=mycompany,dc=com
cn: Users
objectClass: top
objectClass: orclContainer
dn: cn=Groups,cn=oracleAccounts,dc=mycompany,dc=com
cn: Groups
objectClass: top
objectClass: orclContainer
dn: cn=Reserve,cn=orcleAccounts,dc=mycompany,dc=com
cn: Reserve
objectClass: top
objectClass: orclContainer
```

此文件包含创建 OIM 用于同步的新容器所需的信息。使用 `ldapadd` 命令在 OID 中创建这些容器。例如：

```bash
ldapadd –h 10.0.70.229 –p  3060 –D cn=orcladmin –w ****** –c –f oidContainers.ldif
```

在此示例中，`-h` 是 OID 服务器的主机名，`-p` 是 OID 服务器的 LDAP 端口，`-D` 是 `cn=orcladmin`，`-w` 是 oracladmin 的密码，`-f` 是 `oidContainers.ldif` 文件的完整路径。

`ldapadd` 命令位于 OID 的 `ORACLE_HOME/bin` 目录中。

创建用于启用 LDAP 同步的容器后，必须创建 Oracle Identity Manager 的 LDAP 同步过程所需的用户、组和 ACI。首先创建另一个名为 `oimAdmin.ldif` 的文件，内容如下。

```ldif
dn: cn=systemids,dc=mycompany,dc=com
changetype: add
objectclass: orclContainer
objectclass: top
cn: systemids
dn: cn=oimAdminUser,cn=systemids,dc=mycompany,dc=com
changetype: add
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetorgperson
objectclass: orclusers
objectclass: orcluserV2
mail: oimAdminUser
givenname: oimAdminUser
sn: oimAdminUser
cn: oimAdminUser
uid: oimAdminUser
userPassword: gundam1
dn: cn=oimAdminGroup,cn=systemids,dc=mycompany,dc=com
changetype: add
objectclass: groupOfUniqueNames
objectclass: orclPrivilegeGroup
objectclass: top
cn: oimAdminGroup
description: OIM administrator role
uniquemember: cn=oimAdminUser,cn=systemids,dc=mycompany,dc=com
dn: cn=oracleAccounts,dc=mycompany,dc=com
changetype: modify
orclaci: access to entry by group="cn=oimAdminGroup,cn=systemids,dc=mycompany,dc=com" (add,browse,delete) by * (none)
orclaci: access to attr=(*) by group="cn=oimAdminGroup,cn=systemids,dc=mycompany,dc=com" (read,search,write,compare) by * (none)
dn: cn=changelog
changetype: modify
add: orclaci
orclaci: access to entry by group="cn=oimAdminGroup,cn=systemids,dc=mycompany,dc=com" (browse) by * (none)
orclaci: access to attr=(*) by group="cn=oimAdminGroup,cn=systemids,dc=mycompany,dc=com" (read,search,compare) by * (none)
```

创建此文件后，将使用同样位于 `ORACLE_HOME/bin` 目录中的 `ldapmodify` 命令将新对象导入 OID。

```bash
ldapmodify -h 10.0.70.229 -p 3060 -d cn=orcladmin -w gundam1 -c -v –f oidAdmin.ldif
```

通过执行以下命令来验证此 `ldapmodify` 命令是否正常工作。它们都应返回无错误。

```bash
ldapsearch -h  -p  -D "cn=orcladmin"  -w gundam1-b "dc=mycompany,dc=com" -sone "objectclass=*" orclaci
ldapsearch -h  -p  -D  "cn=oimAdminUser,cn=systemids,dc=mycompany,dc=com" -w * -b  "cn=changelog" -s sub "changenumber>=0"
```

在 OAM 和 OIM 需要集成的此类环境中，必须通过扩展 OAM 模式来扩展 OID 以用于 OAM 和 OIM 集成。您将需要使用 `ldapmodify` 脚本导入位于 OAM 主目录中的四个文件。

将您的工作目录移动到 `/home/oracle/IAMMiddleware/Oracle_IAM1/oam/server/oim-intg/ldif/oid/schema/oim-intg/ldif/oid/schema`。这里有四个文件：`OID_oblix_pwd_schema_add.ldif`、`OID_oblix_schema_add.ldif`、`OID_oim_pwd_schema_add.ldif` 和 `OID_oblix_schema_index_add.ldif`。使用以下示例导入新的模式信息。

```bash
ldapmodify -h 10.0.70.229 -p 3060 -D cn=orcladmin -w gundam1 -f OID_oblix_pwd_schema_add.ldif
ldapmodify -h 10.0.70.229 -p 3060 -D cn=orcladmin -w gundam1 -f OID_oblix_schema_add.ldif
ldapmodify -h 10.0.70.229 -p 3060 -D cn=orcladmin -w gundam1 -f OID_oim_pwd_schema_add.ldif
ldapmodify -h 10.0.70.229 -p 3060 -D cn=orcladmin -w gundam1 -f OID_oblix_schema_index_add.ldif
```



## 配置 Oracle Identity Manager 服务器

至此，`OID` 已配置为 `OIM` 的身份存储，可以继续配置 Identity Manager 服务器。在之前的章节中，您已在独立的 WebLogic Middleware Home 中创建了新的 `OIM` 域。该步骤配置了管理服务器、面向服务的架构（`SOA`）服务器和 Identity Management 服务器。本节将在此基础上继续，为启动 `OIM` 做准备。

是时候启动 WebLogic 管理服务器和 `SOA` 服务器了。在配置 Identity Manager 之前，这些服务必须正在运行。切换到 Identity Management 域目录 `/home/oracle/IDMMiddleware/user_projects/oim_domain`。执行命令 `startWeblogic.sh`。如果您希望此进程在后台启动，可以使用 `nohup` 执行该命令，例如 `nohup ./startWeblogic.sh > weblogic.out &`。这将把命令的输出重定向到文件 `weblogic.out`。要实现此操作，`WLS` 必须配置为以开发模式启动。如果服务器配置为生产模式启动，则您需要确保 `boot.properties` 文件已就位。接下来启动 `SOA`。这可以通过在 `bin` 目录中执行 `./startManagedWebLogic.sh soa_server1` 来完成。

**注意**

要在后台启动服务器，必须在安全目录中存在 `boot.properties` 文件，或者服务器必须处于开发模式。

在 `<Domain Home>/servers/<ServerName>/security` 目录中，创建一个名为 `boot.properties` 的文件，格式如下：

```
username=weblogic
password=<your_password>
```

现在，WebLogic 管理服务器和 `SOA` 服务器已启动，您可以继续配置 Identity Manager 服务器。

导航到 `<Middleware_Home>/Oracle_IDM1/bin`，其中 `Middleware_Home` 是您安装 `OIM` 的目录。在本例中，它是 `/home/oracle/IDMMiddleware`。在 `bin` 目录中，执行 `config.sh`：

```
./config.sh
```

**注意**

此步骤仅运行此目录中的 `config.sh` 文件。其他位置可能也存在 `config.sh` 文件，但只有这个会启动正确的 GUI。

第一步是选择在此步骤中要配置的组件。必需的组件是 `OIM` 服务器。`OIM` 设计控制台和 `OIM` 远程管理器只能安装在 Microsoft Windows 上，因此您不会在此处安装它们。如果您决定稍后在单独的服务器上安装它们，可以进行安装。请如图 10-8 所示进行选择。

![A352855_1_En_10_Fig8_HTML.jpg](img/A352855_1_En_10_Fig8_HTML.jpg)

图 10-8. 选择要配置的组件

选择要配置的组件后，工具将询问有关 `OIM` 安装的数据库信息。您在安装软件时已创建了必要的存储库数据库对象。这里您将指定详细信息。图 10-9 显示了数据库连接屏幕。

![A352855_1_En_10_Fig9_HTML.jpg](img/A352855_1_En_10_Fig9_HTML.jpg)

图 10-9. 数据库信息

连接字符串应按 `hostname:port:sid` 格式输入。在本例中，您有一个数据库实例。如果您使用的是 `RAC` 数据库，可以按如下方式输入两个节点：`host1:port1:instancename1^host2:port2:instancename2@<ServiceName>`。

`OIM` 模式用户名是本书前面 `RCU` 操作期间输入的模式名称。本例中，使用 `imdev_oim`。

`OIM` 模式密码在下一行输入。您在 `RCU` 操作期间提供了此密码。

`MDS` 连接字符串默认与已输入的连接字符串值相同。如果您将 `MDS` 模式安装在不同的数据库中，请选中“为 `MDS` 模式选择不同的数据库”复选框，您将能够编辑此字段。

`MDS` 模式用户名应包含 `RCU` 操作期间配置的用户名，本例中为 `imdev_mds`。


## OIM 服务器配置

MDS Schema Password 应包含 `imdev_mds` 的密码。

在初始域创建步骤中，您已配置了一个用于承载 OIM 的 WebLogic 域。您将 Administration Server 配置在端口 27001 上。请输入 Administration Server 的 URL，包括端口，如图 10-10 所示。请注意，在此情况下，您必须使用 T3 协议以 `t3://hostname:port` 的格式输入 URL。输入 WebLogic 用户名和密码，然后单击“下一步”继续。

![A352855_1_En_10_Fig10_HTML.jpg](img/A352855_1_En_10_Fig10_HTML.jpg)
*图 10-10. WLS 信息*

下一步允许您配置新的 OIM 服务器的详细信息。图 10-11 显示了收集新的 OIM 管理员密码和密钥库密码的屏幕。请记下这些值，因为它们在后续过程中是必需的。

![A352855_1_En_10_Fig11_HTML.jpg](img/A352855_1_En_10_Fig11_HTML.jpg)
*图 10-11. OIM 服务器配置*

OIM Administrator Password 应设置为您能记住的密码。OIM HTTP URL 是将用于 Identity Manager 的 URL。如果您在 OIM 服务器前部署了 Oracle HTTP Server (OHS)，请在此处输入其 URL。如果您尚未安装 OHS 服务器，可以稍后更新此设置。输入并确认一个 Keystore Password。

选中“Enable OIM for Suite Integration”复选框。此环境将包含 OAM 和 OIM 集成。同步是必需的。本章前面已介绍了此操作的先决条件步骤。如果您尚未完成这些步骤，请立即返回完成后再继续。

在 10 个步骤中的第 6 步，系统将提示您输入 OIM 使用的 LDAP 身份存储。图 10-12 显示了 LDAP 服务器屏幕。

![A352855_1_En_10_Fig12_HTML.jpg](img/A352855_1_En_10_Fig12_HTML.jpg)
*图 10-12. 目录服务器配置*

在之前的步骤中，您已预配置了此身份存储。在此示例中，您使用的是 OID。

*   选择 OID 作为 Directory Server Type。
*   输入一个 Directory Server ID。这应该是唯一的。
*   以 `ldap://hostname:ldapport.` 的格式输入 LDAP 服务器连接信息。
*   输入 OIM 将用于连接和执行操作的 Server User。这不应是 `cn=orcladmin` 用户。相反，在预配置步骤中，您创建了一个名为 `oimadminuser` 的用户。请使用完整专有名称输入它：`cn=oimadminuser,cn=systemids,dc=mycompany,dc=com`。
*   Server Password 期望您在预配置步骤中设置的密码。
*   输入 Server SearchDN 作为包含用户和组容器的顶层容器名称。

由于您选择了启用 LDAP 同步，配置工具将提醒您完成预配置步骤，如图 10-13 所示。如果您尚未完成，请返回本章相应部分并立即完成步骤。

![A352855_1_En_10_Fig13_HTML.jpg](img/A352855_1_En_10_Fig13_HTML.jpg)
*图 10-13. 先决条件检查*

图 10-14 所示的 LDAP Server Continued 屏幕继续配置 LDAP 身份目录。

![A352855_1_En_10_Fig14_HTML.jpg](img/A352855_1_En_10_Fig14_HTML.jpg)
*图 10-14. LDAP 服务器信息*

在此屏幕上，您将输入有关 OIM 将用于创建用户的用户和组容器的信息。

*   LDAP RoleContainer 是组容器。
*   输入描述。
*   LDAP UserContainer 是 OIM 将管理的所有用户的位置。
*   输入描述。
*   User Reservation Container 是等待批准的新创建用户的临时存放区。

图 10-15 所示的 Configuration Summary 屏幕显示了已输入的配置参数的汇总。

![A352855_1_En_10_Fig15_HTML.jpg](img/A352855_1_En_10_Fig15_HTML.jpg)
*图 10-15. 配置摘要屏幕*

确保列出了您选择配置的组件。这也是再次检查您用于配置的值的机会。单击“Configure”继续。

您现已完成 OIM 配置。服务器现在应配置为具有 OID 身份存储并启用了 LDAP 同步。图 10-16 显示了已完成的配置。

![A352855_1_En_10_Fig16_HTML.jpg](img/A352855_1_En_10_Fig16_HTML.jpg)
*图 10-16. 配置完成屏幕*

### 完成 LDAP 安装后配置

在上一节中，您完成了 OIM 配置。在该步骤中，您选择了“Enable OIM for Suite Integration”。这告知 OIM 您将使用 LDAP 同步，这是 OAM 和 OIM 集成所必需的。在预配置期间，您运行了实用程序来准备 OID 以进行同步。现在是运行安装后任务以使其完成的时候了。

在继续之前，请确保您的环境变量已为 `ldapconfigpostsetup` 实用程序正确设置。必须如下设置：

*   `APP_SERVER=weblogic`
*   `JAVA_HOME=/home/oracle/java/jdk1.6.0_45`
*   `MW_HOME=/home/oracle/IDMMiddleware`
*   `OIM_ORACLE_HOME=/home/oracle/IDMMiddleware/Oracle_IDM1`
*   `WL_HOME=/home/oracle/IDMMiddleware/wlserver_10.3`
*   `DOMAIN_HOME=/home/oracle/IDMMiddleware/user_projects/domains/idm_domain`

在目录 `/home/oracle/IDMMiddleware/Oracle_IDM1/server/ldap_config_util` 中，您将找到一个名为 `ldapconfig.props` 的文件。为此文件创建一个备份副本，并按如下所示进行编辑：

```
# OIMServer Type, Valid values can be WLS, JBOSS, WAS
# e.g.: OIMServerType=WLS
OIMServerType=WLS
# OIM Provider URL
# if OIMServerType is WAS then
#    OIMProviderURL=corbaloc:iiop:OIMServerBootStrapHost:OIMServerBootStrapPort
#    hint:look for the OIM server bootStrapPort number in WAS admin console
#    e.g.: corbaloc:iiop:hostName:2809
# else OIMServerType is WLS then
#      OIMProviderURL=t3://localhost:ManagedServerPort
#      e.g.: OIMProviderURL=t3://localhost:14000
OIMProviderURL=t3://10.0.70.229:14000
# LDAP/OVD Server URL
# If OVD server is selected during OIM installation, then provide
# value for LDAPURL.
# LDAPURL=ldap://:
# e.g.: LDAPURL=ldap://OVDserver.us.oracle.com:6501
#
# If not, leave LDAPURL empty
# e.g.: LDAPURL=
LDAPURL=
# If OVD server is selected during OIM installation, then provide
# Admin user name to connect to LDAP/OVD Server
# e.g.: LDAPAdminUsername=cn=oimAdminUser,cn=systemids,dc=us,dc=oracle,dc=com
#
# If not, leave LDAPAdminUsername empty
# e.g.: LDAPAdminUsername=
LDAPAdminUsername=
# If OVD server is NOT selected during OIM installation, then provide
# LIBOVD_PATH_PARAM=
# e.g.: #LIBOVD_PATH_PARAM=/scratch/suramesh/Oracle/Middleware/user_projects/domains/base_domain/config/fmwconfig/ovd/oim
LIBOVD_PATH_PARAM=
# This is for FA C9, MT scenario where last changenumber from the standby
# server is to be updated to the incremental recon. jobs, to be picked up,
# once the primary/secondary replicated server comes up.
# If the change number is not passed through this parameter, then logic is to
# query the OVD and retrieve the same.
ChangeLogNumber=
```

对文件进行以下更改并保存：

*   `OIMServerType=WLS`
*   将 LDAP URL 留空，因为您未使用 Oracle Virtual Directory (OVD)。
*   将 LDAPUsername 留空，因为您未使用 OVD。
*   将 LIBOVD_PATH_PARAM 留空，因为您未使用 OVD。

接下来，您将运行命令 `LDAPConfigPostSetup.sh`，为其提供 `ldap_config_util` 的目录：

```
[oracle@clouddemolab ldap_config_util]$ ./LDAPConfigPostSetup.sh /home/oracle/IDMMiddleware/Oracle_IDM1/server/ldap_config_util
```

此命令的输出应与以下内容类似。



要运行实用工具，需要设置以下环境变量：
APP_SERVER 为 weblogic
OIM_ORACLE_HOME 为 /home/oracle/IDMMiddleware/Oracle_IDM1
JAVA_HOME 为 /home/oracle/java/jdk1.6.0_45
MW_HOME 为 /home/oracle/IDMMiddleware
WL_HOME 为 /home/oracle/IDMMiddleware/wlserver_10.3
DOMAIN_HOME 为 /home/oracle/IDMMiddleware/user_projects/domains/idm_domain
正在 IPv4 模式下执行 oracle.iam.platformservice.utils.LDAPConfigPostSetup
[输入 LDAP 管理密码:]
[输入 OIM 管理密码:]
LDAP 服务器为 OID 11.1.1.5.0
已获取 LDAP 连接.....
UsernamePasswordLoginModule.initialize()，调试已启用
UsernamePasswordLoginModule.login()，用户名 xelsysadm
UsernamePasswordLoginModule.login()，URL t3://10.0.70.229:14000
log4j:WARN 找不到日志记录器 (org.springframework.jndi.JndiTemplate) 的附加器。
log4j:WARN 请正确初始化 log4j 系统。
已通过 OIM 管理员认证.....
已获取调度器服务.....

更改日志搜索返回了以下属性：
lastchangenumber

已成功启用基于更改日志的对账计划任务。
已成功使用最后更改号 : 167990 更新了基于更改日志的对账计划任务。

您现在可以采取步骤验证 LDAP 同步是否按预期运行。

## 本章摘要

在本章中，我们介绍了配置 OIM 服务器的必要步骤。在之前的章节中，您安装了必要的软件并创建了一个 WebLogic 域。在此基础上，您现在应该拥有一个可用于在 OID 身份存储中创建和管理用户的 OIM 实例。您还为此安装进行了预配置，为将 OIM 与 OAM 集成做准备。此过程的下一步将是开始这两个产品的实际集成。

