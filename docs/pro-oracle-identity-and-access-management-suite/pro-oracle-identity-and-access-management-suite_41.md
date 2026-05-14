# query the OVD and retrieve the same.
ChangeLogNumber=
```

对文件进行以下更改并保存：

*   `OIMServerType=WLS`
*   将 LDAP URL 留空，因为您未使用 Oracle Virtual Directory (OVD)。
*   将 `LDAPAdminUsername` 留空，因为您未使用 OVD。
*   将 `LIBOVD_PATH_PARAM` 留空，因为您未使用 OVD。

接下来，您将运行命令 `LDAPConfigPostSetup.sh`，为其提供 `ldap_config_util` 的目录。

```
[oracle@clouddemolab ldap_config_util]$ ./LDAPConfigPostSetup.sh /home/oracle/IDMMiddleware/Oracle_IDM1/server/ldap_config_util
```

此命令的输出应类似于以下内容。

```
要运行 `Utilities`，需要设置以下环境变量：
`APP_SERVER` 为 `weblogic`
`OIM_ORACLE_HOME` 为 `/home/oracle/IDMMiddleware/Oracle_IDM1`
`JAVA_HOME` 为 `/home/oracle/java/jdk1.6.0_45`
`MW_HOME` 为 `/home/oracle/IDMMiddleware`
`WL_HOME` 为 `/home/oracle/IDMMiddleware/wlserver_10.3`
`DOMAIN_HOME` 为 `/home/oracle/IDMMiddleware/user_projects/domains/idm_domain`
以 `IPv4` 模式执行 `oracle.iam.platformservice.utils.LDAPConfigPostSetup`
`[请输入 LDAP 管理员密码:]`
`[请输入 OIM 管理员密码:]`
`LDAP` 服务器为 `OID 11.1.1.5.0`
已获取 `LDAP` 连接.....
`UsernamePasswordLoginModule.initialize()`，已启用调试
`UsernamePasswordLoginModule.login()`，用户名 `xelsysadm`
`UsernamePasswordLoginModule.login()`，URL `t3://10.0.70.229:14000`
`log4j:WARN 未找到记录器 (org.springframework.jndi.JndiTemplate) 的任何附加器。`
`log4j:WARN 请正确初始化 log4j 系统。`
已通过 `OIM` 管理员认证.....
已获取 `调度程序` 服务.....

`changelog` 搜索返回了以下属性：
`lastchangenumber`

已成功启用基于 `变更日志` 的 `对账` 计划任务。
已成功使用最后更改编号 `167990` 更新了基于 `变更日志` 的 `对账` 计划任务。
```

现在，您可以采取步骤验证 `LDAP` 同步是否按预期运行。

## 总结

在本章中，您了解了配置 `OIM` 服务器的必要步骤。在之前的章节中，您安装了必要的软件并创建了 `WebLogic` 域。在此基础上，您现在应该拥有一个可用于在 `OID` 身份存储中创建和管理用户的 `OIM` 实例。您还为此安装进行了预配置，为将 `OIM` 与 `OAM` 集成做准备。此过程的下一步是开始这两款产品的实际集成。

# 11. Oracle 身份与访问管理器集成

在之前的章节中，您已安装了 `Oracle Internet Directory` (`OID`)、`Oracle Access Manager` (`OAM`) 和 `Oracle Identity Manager` (`OIM`)，为集成做准备。此集成带来了若干好处，包括提高用户生产力、增强身份安全性以及减少因忘记密码或账户锁定而导致的帮助呼叫。

使用集成环境的组织不仅能够利用 `OIM` 的用户配置和审计功能，还能将用户自助服务功能与 `OAM` 的单点登录 (`SSO`) 启用能力相结合。这两个主要组件的结合为可能需要管理其账户访问权限、密码或个人资料信息的用户创造了无缝界面。正如 `SSO` 环境允许用户登录一次应用程序后，即可自动登录到他们可能需要的其他应用程序一样，此集成将身份管理器添加为受保护的资源。这通过在某些事件（如更改密码、忘记密码等）后自动登录用户，促进了用户自助服务。另一个功能是 `OIM` 和 `OAM` 能够相互通信以确定用户状态，例如首次登录和账户过期。

在本章结束时，您将拥有一个包含一个受保护资源或 URL 的集成环境。您的用户将能够通过登录屏幕，在 `SSO` 环境中使用首次登录、忘记密码和其他身份管理器功能。

## IdmConfigTool

Oracle 提供了一个工具，可用于确保执行所有必要的配置以实现成功集成。有大量的配置参数需要设置或更新。正确运行此工具有助于在设置过程中减少麻烦，实现无缝集成。

将新应用程序和资源添加到 `OAM` `SSO` 环境可以相对简单。对于大多数 Oracle 产品和应用程序，这涉及在 `WebLogic` 域中创建一个新的 `Oracle Access Manager` 身份断言器并配置 `WebGate`。然而，对于 `OIM`，必须配置多个组件以确保两个组件之间的通信正常运行。幸运的是，Oracle 提供了一个名为 `IdmConfigTool` 的工具。在之前的章节中，您已使用此工具预配置了身份存储。在本章中，它用于配置 `OAM`、`OIM` 和 `WLS` 的集成。

`IdmConfigTool` 可以协助集成身份管理器组件的任务。它还可用于验证配置参数并提取集成环境的配置参数，以协助故障排除或在其他环境中复制。它支持在单域和拆分域环境中集成身份管理器组件，支持 `11g` 和 `10g` `WebGates`，以及多种身份存储类型，如 `OID` 和 `Oracle Unified Directory` (`OUD`)。

尽管 `idmConfigTool` 提供了许多集成工具，但工具的正常运行取决于正确的环境管理。注意运行 `idmConfigTool` 的位置非常重要。在本书中，您在拆分域中设置了 `OAM` 和 `OIM`。Oracle 文档提到您应始终从同一位置运行 `idmConfigTool`。但是，如果在拆分域中运行，则必须在 `OAM` 主目录位置运行特定于 `OAM` 的命令，而特定于 `OIM` 的命令则从 `OIM` 主目录运行。

由于本章涵盖 `OAM` 和 `OIM` 任务，因此每个部分都介绍了 `idmConfigTool` 的正确设置。

## 配置 Oracle 访问管理器

在第 9 章中，您已经配置了 OAM 并为其与 OIM 的集成进行了预配置。您还在 OID 轻量目录访问协议 (LDAP) 存储中创建了必要的对象。这一切都是为了让 OAM 启动并运行，并确保创建了正确的管理用户。您还将 OID 集成为 WebLogic 层的身份验证器。

如前所述，此环境是一个拆分域，其中 OAM 和 OIM 安装在不同的 Middleware Home 和不同的 WebLogic 域中。因此，OAM 的配置将使用位于 OAM 主目录中的 `idmConfigTool`。有关如何准备环境变量以使用此工具的详细信息，请参见表 11-1。

表 11-1. `idmConfigTool` 的环境变量

| `MW_HOME` | 将此设置为 OAM 中间件软件的安装目录；例如，`/home/oracle/IAMMiddleware/` |
| --- | --- |
| `JAVA_HOME` | 将此设置为 Java 安装目录；例如，`/home/oracle/java` |
| `IDM_HOME` | 如果您在同一物理服务器上安装了 OID，请将其设置为 OID 安装目录；例如，`/home/oracle/OIDMiddleware`；如果 OID 安装在单独的服务器上，请将其留空 |
| `ORACLE_HOME` | 将此设置为 OAM 安装目录；例如，`/home/oracle/IAMMiddleware/Oracle_IAM` |

`idmConfigTool.sh` 文件位于 `$ORACLE_HOME/idmtools/bin` 目录中。执行这些步骤时，应从该目录运行各个步骤。在此过程中，将创建并追加一个名为 `automation.log` 的文件。

要开始该过程，首先创建一个名为 `OAMConfigPropertyFile` 的文件。此文件将包含为 OAM 和 OIM 集成配置 OAM 所需的相关信息。该文件应包含有关您配置的以下信息。请相应地更新值：

```
vi OAMConfigPropertyFile
WLSHOST: 10.0.70.229
WLSPORT: 17001
WLSADMIN: weblogic
WLSPASSWD: weblogic11
ADMIN_SERVER_USER_PASSWORD: weblogic1
IDSTORE_HOST: 10.0.70.229
IDSTORE_PORT: 3060
IDSTORE_BINDDN: cn=orcladmin
IDSTORE_USERNAMEATTRIBUTE: cn
IDSTORE_LOGINATTRIBUTE: uid
IDSTORE_USERSEARCHBASE: cn=Users,dc=mycompany,dc=com
IDSTORE_SEARCHBASE: dc=mycompany,dc=com
IDSTORE_SYSTEMIDBASE: cn=systemids,dc=mycompany,dc=com
IDSTORE_GROUPSEARCHBASE: cn=Groups,dc=mycompany,dc=com
IDSTORE_OAMSOFTWAREUSER: oamLDAP
IDSTORE_OAMADMINUSER: oamadmin
IDSTORE_DIRECTORYTYPE: OID
POLICYSTORE_SHARES_IDSTORE: true
PRIMARY_OAM_SERVERS: 10.0.70.229:5575
WEBGATE_TYPE: ohsWebgate11g
ACCESS_GATE_ID: Webgate_IDM
OAM11G_IDM_DOMAIN_OHS_HOST:10.0.70.229
OAM11G_IDM_DOMAIN_OHS_PORT:7777
OAM11G_IDM_DOMAIN_OHS_PROTOCOL:http
OAM11G_WG_DENY_ON_NOT_PROTECTED: false
OAM11G_IMPERSONATION_FLAG: false
OAM_TRANSFER_MODE: open
OAM11G_OAM_SERVER_TRANSFER_MODE:open
OAM11G_IDM_DOMAIN_LOGOUT_URLS: /console/jsp/common/logout.jsp,/em/targetauth/emaslogout.jsp,/oamsso/logout.html,/cgi-bin/logout.pl
OAM11G_OIM_WEBGATE_PASSWD: gundam1
OAM11G_SERVER_LOGIN_ATTRIBUTE: uid
COOKIE_DOMAIN: .centroid.com
OAM11G_IDSTORE_NAME: OIDIdentityStore
OAM11G_IDSTORE_ROLE_SECURITY_ADMIN: OAMAdministrators
OAM11G_SSO_ONLY_FLAG: false
OAM11G_OIM_INTEGRATION_REQ: true
OAM11G_SERVER_LBR_HOST:10.0.70.229
OAM11G_SERVER_LBR_PORT:7777
OAM11G_SERVER_LBR_PROTOCOL:http
COOKIE_EXPIRY_INTERVAL: 120
OAM11G_OIM_OHS_URL:http://10.0.70.229:7777/
SPLIT_DOMAIN: false
```

在此文件中，参数值应按表 11-2 所示进行设置。

表 11-2. `OAMConfigPropertyFile` 参数值

| 参数 | 描述 |
| --- | --- |
| `WLSHost` | 安装 OAM 的服务器的主机名 |
| `WLSPort` | OAM WebLogic 管理服务器的监听端口 |
| `WLSADMIN` | OAM WebLogic 管理员用户名 <weblogic> |
| `WLSPASSWD` | OAM WebLogic 管理员密码 |
| `ADMIN_SERVER_USER_PASSWORD` | OAM 管理员密码 |
| `IDSTORE_HOST` | 安装了 OID 的服务器的主机名 |
| `IDSTORE_PORT` | OID 的 LDAP 端口 |
| `IDSTORE_BINDDN` | 身份存储管理员用户的完全限定用户名 |
| `IDSTORE_USERNAMEATTRIBUTE` | 指示包含用户用户名的身份存储属性；例如，`cn` |
| `IDSTORE_LOGINATTRIBUTE` | 输入包含用户登录名的属性 |
| `IDSTORE_USERSEARCHBASE` | 此值指示 OAM 应从身份存储内的何处开始搜索用户；例如，`cn=users,dc=mycompany,dc=com` |
| `IDSTORE_SEARCHBASE` | 使用身份存储的顶层；例如，`dc=mycompany, dc=com` |
| `IDSTORE_SYSTEMIDBASE` | 输入身份存储中管理或系统帐户的位置 |
| `IDSTORE_GROUPSEARCHBASE` | 身份存储内组的位置 |
| `IDSTORE_OAMSOFTWAREUSER` | 这应该是 OAM 将用来连接到身份存储的用户名；例如，`oamLDAP` |
| `IDSTORE_OAMADMINUSER` | OAM 用来修改身份存储条目的用户名 |
| `IDSTORE_DIRECTORYTYPE` | 指示用作身份存储的服务器类型；例如，OID、AD、OpenLDAP |
| `POLICYSTORE_SHARES_IDSTORE` | 如果 OAM 策略与身份存储存储在同一个存储中，则将此值设置为 `true` |
| `PRIMARY_OAM_SERVERS` | 列出 OAM 主机名 |
| `WEBGATE_TYPE` | 输入 WebGate 类型：`ohsWebgate11g` 或 `ohsWebGate10g` |
| `ACCESS_GATE_ID` | 输入在 OAM 配置中用作 WebGate 的名称 |
| `OAM11G_IDM_DOMAIN_OHS_HOST` | 安装 Oracle HTTP Server (OHS) 的主机名 |
| `OAM11G_IDM_DOMAIN_OHS_PORT` | OHS 监听的端口 |
| `OAM11G_IDM_DOMAIN_OHS_PROTOCOL` | 指示此 OHS 是否使用安全套接字层 (SSL)；例如，`http` 或 `https` |
| `OAM11G_WG_DENY_ON_NOT_PROTECTED` | 将此设置为 `true` 可使所有未在 OAM 中另行配置的 URL 模式拒绝访问 |
| `OAM11G_IMPERSONATION_FLAG` | 指示 OAM 是否应允许模拟；这在融合应用程序环境中很有用 |
| `OAM_TRANSFER_MODE` | 配置 OAM 的通信模式；有效值为 `Open` 和 `Simple` |
| `OAM11G_OAM_SERVER_TRANSFER_MODE` | 指示 OAM 将支持的通信类型；请注意，这应与 `OAM_Transfer_Mode` 匹配 |
| `OAM11G_IDM_DOMAIN_LOGOUT_URLS` | 配置将用于注销操作的 URL |
| `OAM11G_OIM_WEBGATE_PASSWD` | 输入 WebGate 的密码 |
| `OAM11G_SERVER_LOGIN_ATTRIBUTE` | 配置用于用户登录的属性名称 |
| `COOKIE_DOMAIN` | 指示 Cookie 有效的域名 |
| `OAM11G_IDSTORE_NAME` | 输入系统身份存储的名称 |
| `OAM11G_IDSTORE_ROLE_SECURITY_ADMIN` | 输入身份存储中包含管理用户的组或角色的名称 |
| `OAM11G_SSO_ONLY_FLAG` | 将此设置为 `true` 仅提供身份验证功能；将其设置为 `false` 将配置 OAM 也执行授权检查 |
| `OAM11G_OIM_INTEGRATION_REQ` | 输入 `true` 表示 OIM 和 OAM 将被集成 |
| `OAM11G_SERVER_LBR_HOST` | 提供 OAM 位置的负载均衡器 URL |
| `OAM11G_SERVER_LBR_PORT` | 提供负载均衡器端口 |
| `OAM11G_SERVER_LBR_PROTOCOL` | 指示 URL 是否为 SSL |
| `COOKIE_EXPIRY_INTERVAL` | 设置此值 |
| `OAM11G_OIM_OHS_URL` | 如果 OAM 和 OIM 前端有 OHS，请提供 URL |
| `SPLIT_DOMAIN` | 如果 OAM 和 OIM 安装在不同的 WebLogic 服务器中，则将此设置为 `true` |

## 配置 Oracle 访问管理器

文件创建完成后，你现在可以运行带 `–configOAM` 选项的 `idmConfigTool` 来设置 OAM 以进行集成。运行此步骤将配置 OAM 标识存储并注册必要的身份验证策略。系统将提示你为 OAM 软件用户、OAM 管理员用户和 OAM WebLogic 用户配置密码。如果没有报告错误，请重启 OAM 环境，包括 OAM 托管服务器和管理服务器。输出应类似于以下内容。你还应检查 `Automation.log` 文件是否有任何错误。

```
[oracle@clouddemolab bin]$ ./idmConfigTool.sh  -configOAM input_file=OAMConfigPropertyFile
Enter ID Store Bind DN Password :
Enter User Password for IDSTORE_PWD_OAMSOFTWAREUSER:
Confirm User Password for IDSTORE_PWD_OAMSOFTWAREUSER:
Enter User Password for IDSTORE_PWD_OAMADMINUSER:
Confirm User Password for IDSTORE_PWD_OAMADMINUSER:
^[[BConnecting to t3://10.0.70.229:17001
Connection to domain runtime mbean server established
Starting edit session
Edit session started
Connected to security realm.
Validating provider configuration
Validated desired authentication providers
Created OAMIDAsserter successfuly
A type of LDAP Authenticator already exists in the security realm. Please create authenticator manually if different LDAP provider is required.
Control flags for authenticators set sucessfully
Reordering of authenticators done sucessfully
Saving the transaction
Transaction saved
Activating the changes
Changes Activated. Edit session ended.
Connection closed sucessfully
The tool has completed its operation. Details have been logged to automation.log
```

你现在已经配置了 OAM，为与 OIM 的集成做好了准备。要继续此过程，你将使用 `idmConfigTool.sh` 命令。如果你在拆分域中工作，或者你的 OIM 实例位于单独的服务器上，请转到该服务器或 OIM 软件主目录。

## 配置 Oracle 身份管理器

OIM 需要一个系统用户来连接到标识存储，并浏览、创建和编辑用户。就像你为 OAM 配置所做的那样，你将使用 `idmConfigTool` 命令来完成此操作。以下说明将指导你完成系统帐户的创建。

在运行 `idmConfigTool.sh` 命令之前，必须确保环境变量已正确设置。否则将导致各种错误，并且并非所有错误都会指明原因。请按如下方式设置环境变量：

```
MW_HOME=/home/oracle/OAMMiddlewareHome
JAVA_HOME=
IDM_HOME=/home/oracle/OIDMiddlewareHome
ORACLE_HOME=/home/oracle/OAMMiddlewareHome/Oracle_IAM1
IAM_ORACLE_HOME=/home/oracle/OAMMiddlewareHome/OracleIAM1
```

要创建系统帐户，首先创建一个名为 `preconfigOIMProperties` 的属性文件，如下所示。

```
IDSTORE_HOST: 10.0.70.229
IDSTORE_PORT: 3060
IDSTORE_BINDDN: cn=orcladmin
IDSTORE_USERNAMEATTRIBUTE: cn
IDSTORE_LOGINATTRIBUTE: uid
IDSTORE_USERSEARCHBASE: cn=Users,dc=mycompany,dc=com
IDSTORE_GROUPSEARCHBASE: cn=Groups,dc=mycompany,dc=com
IDSTORE_SEARCHBASE: dc=mycompany,dc=com
POLICYSTORE_SHARES_IDSTORE: true
IDSTORE_SYSTEMIDBASE: cn=systemids,dc=mycompany,dc=com
IDSTORE_OIMADMINUSER: oimLDAP
IDSTORE_OIMADMINGROUP: OIMAdministrators
```

在 `preconfigOIMProperties` 文件中，应根据表 11-3 中的列表设置值。

表 11-3. `preconfigOIMProperties` 文件的参数定义

| 参数 | 定义 |
| :--- | :--- |
| `IDSTORE_HOST` | 设置为运行 OID 的服务器的主机名 |
| `IDSTORE_PORT` | 应设置为 OID 服务器的 LDAP 端口 |
| `IDSTORE_BINDDN` | `cn=orcladmin` |
| `IDSTORE_USERNAMEATTRIBUTE` | 设置为存储用户名的 LDAP 属性；此值将用于搜索用户 |
| `IDSTORE_LOGINATTRIBUTE` | 将此值设置为存储用户登录名的属性 |
| `IDSTORE_USERSEARCHBASE` | 应设置为包含你环境中用户的容器 |
| `IDSTORE_GROUPSEARCHBASE` | 输入存储你环境中组的容器的名称 |
| `IDSTORE_SEARCHBASE` | 存储前述搜索基准值的整体容器 |
| `POLICYSTORE_SHARES_IDSTORE` | 如果你的标识存储也存储策略信息，则设置为 true |
| `IDSTORE_SYSTEMIDBASE` | 此位置将用于存储来自 Oracle 身份管理的用户对帐数据 |
| `IDSTORE_OIMUSER` | 将在 LDAP 存储中创建的、供 OIM 使用的帐户 |
| `IDSTORE_OIMADMINGROUP` | 输入将存储 OIM 管理员用户的组 |

一旦创建了 `preconfigOIMProperties` 文件，你就可以运行 `idmConfigTools.sh` 命令来准备 OID 标识存储。

在此步骤的开始，你设置了环境变量以支持 `idmConfigTool`。如果尚未完成，请在继续之前重新访问设置步骤。使用 `–prepareIDStore` 运行 `idmConfigTool`，将模式设置为 `OIM`，如下一个示例所示。

在 OAM 主位置运行此命令：

```
/home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/bin
./idmConfigTools.sh –prepareIDStore mode=OIM input_file=preconfigOIMProperties
```

系统将提示你输入 `orcladmin` 密码，并为 `OIMLDAP` 用户和 `XELSYSADM` 用户设置密码。如果一切按预期完成，系统将提示你输入 OIM ldap 用户密码并看到以下输出：

```
Enter ID Store Bind DN Password :
*** Creation of oimLDAP ***
2016 年 1 月 5 日 下午 1:20:01 oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_user_template.ldif
Enter User Password for oimLDAP:
Confirm User Password for oimLDAP:
2016 年 1 月 5 日 下午 1:20:06 oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_group_template.ldif
2016 年 1 月 5 日 下午 1:20:06 oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/common/oim_group_member_template.ldif
2016 年 1 月 5 日 下午 1:20:06 oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_groups_acl_template.ldif
2016 年 1 月 5 日 下午 1:20:06 oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_reserve_template.ldif
*** Creation of Xel Sys Admin User ***
2016 年 1 月 5 日 下午 1:20:06 oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/idm_xelsysadmin_user.ldif
Enter User Password for xelsysadm:
Confirm User Password for xelsysadm:
2016 年 1 月 5 日 下午 1:20:14 oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_group_template.ldif
2016 年 1 月 5 日 下午 1:20:14 oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/common/group_member_template.ldif
The tool has completed its operation. Details have been logged to automation.log
```


## 集成 OIM 与 OAM

与配置 OAM 进行集成类似，你将使用 `idmConfigTool.sh` 命令来执行 OIM 和 OAM 的集成。这涉及创建一个属性文件并使用它来运行 `idmConfigTool.sh –configOIM`。

在运行 `idmConfigTool.sh` 命令之前，必须确保环境变量已正确设置。若未正确设置，将导致各种错误，且并非所有错误都能明确指出问题根源。请按如下方式设置环境变量：

```
MW_HOME=/home/oracle/OIMMiddlewareHome
JAVA_HOME=
IDM_HOME=/home/oracle/OIDMiddlewareHome
ORACLE_HOME=/home/oracle/OIMMiddlewareHome/Oracle_OIM1
IAM_ORACLE_HOME=/home/oracle/OIMMiddlewareHome/Oracle_OIM1
```

创建一个名为 `OIMConfigPropertyFile` 的属性文件，内容如下：

```
LOGINURI: /${app.context}/adfAuthentication
LOGOUTURI: /oamsso/logout.html
AUTOLOGINURI: None
ACCESS_SERVER_HOST: 10.0.70.229
ACCESS_SERVER_PORT: 5575
ACCESS_GATE_ID: Webgate_IDM
COOKIE_DOMAIN: .centroid.com
COOKIE_EXPIRY_INTERVAL: 120
OAM_TRANSFER_MODE: SIMPLE
WEBGATE_TYPE: ohsWebgate11g
OAM_SERVER_VERSION: 11g
OAM11G_WLS_ADMIN_HOST: 10.0.70.229
OAM11G_WLS_ADMIN_PORT: 17001
OAM11G_WLS_ADMIN_USER: weblogic
SSO_ENABLED_FLAG: true
IDSTORE_PORT: 3060
IDSTORE_HOST: 10.0.70.229
IDSTORE_DIRECTORYTYPE: OID
IDSTORE_ADMIN_USER: cn=oamLDAP,cn=systemids,dc=mycompany,dc=com
IDSTORE_LOGINATTRIBUTE: uid
IDSTORE_USERSEARCHBASE: cn=Users,dc=mycompany,dc=com
IDSTORE_GROUPSEARCHBASE: cn=Groups,dc=mycompany,dc=com
IDSTORE_WLSADMINUSER: weblogic_idm
MDS_DB_URL: jdbc:oracle:thin:@10.0.70.229:1521:IDM1
MDS_DB_SCHEMA_USERNAME: dev_mds
WLSHOST: 10.0.70.229
WLSPORT: 27001
WLSADMIN: weblogic
DOMAIN_NAME: idm_domain
OIM_MANAGED_SERVER_NAME: oim_server1
DOMAIN_LOCATION: /home/oracle/IDMMiddleware/user_projects/idm_domain
```

此文件中每个参数的值都应使用特定于你环境的信息来填写。有关值的描述，请参考表 11-4。

表 11-4. `OIMConfigPropertyFile` 参数值

| `LOGINURI` | 此值取自原始文件；请勿修改 |
| `LOGOUTURI` | 你可以保留默认值；但是，如果你有一个符合要求的自定义注销页面，请在此处输入其 URL |
| `AUTOLOGINURI` | 将此项设置为 none |
| `ACCESS_SERVER_HOST` | 输入安装了 OAM 的主机名 |
| `ACCESS_SERVER_PORT` | 输入 Oracle Access Management 端口 5575。 |
| `ACCESS_GATE_ID` | 输入在上一步中创建的 Oracle Access Gate 的名称 |
| `COOKIE_DOMAIN` | 输入 Cookie 有效的域名 |
| `COOKIE_EXPIRY_INTERVAL` | 输入 Cookie 的有效分钟数 |
| `OAM_TRANSFER_MODE` | 输入与上一步相同的通信模式值；例如，open 或 simple |
| `WEBGATE_TYPE` | 指明正在使用的 WebGate 版本；有效值为 ohsWebGate11g 和 ohsWebGate10g |
| `OAM_SERVER_VERSION` | 输入 11g 作为 Access Server 的版本 |
| `OAM11G_WLS_ADMIN_HOST` | 输入安装了 OAM 的 WebLogic Server 管理服务器的主机名 |
| `OAM11G_WLS_ADMIN_PORT` | 输入管理服务器监听的端口 |
| `OAM11G_WLS_ADMIN_USER` | 输入 WebLogic |
| `SSO_ENABLED_FLAG` | 输入是否要启用 SSO；例如，true |
| `IDSTORE_PORT` | 输入 OID 或其他 LDAP 服务器正在监听的端口；这应与 Identity Store 的位置匹配 |
| `IDSTORE_HOST` | 输入安装了 OID 或其他 LDAP 服务器的主机名 |
| `IDSTORE_DIRECTORYTYPE` | 输入正在使用的 LDAP 服务器类型；例如，OID |
| `IDSTORE_ADMIN_USER` | 输入 OIM 用于连接到 Identity Store 的用户名；例如，`cn=oimldap,cn=systemids,dc=mycompany,dc=com` |
| `IDSTORE_LOGINATTRIBUTE` | 指定用于登录名的 Identity Store 属性 |
| `IDSTORE_USERSEARCHBASE` | 指定 OIM 应在其中搜索用户的 Identity Store 顶层位置 |
| `IDSTORE_GROUPSEARCHBASE` | 指定 OIM 应在其中查找组的位置 |
| `IDSTORE_WLSADMINUSER` | 输入 weblogic |
| `MDS_DB_URL` | 输入元数据数据库连接信息；格式应为 `jdbc:oracle:thin:@hostname:port:servicename` |
| `MDS_DB_SCHEMA_USERNAME` | 输入用于连接到元数据存储库数据库模式的用户名 |
| `WLSHOST` | 输入安装了 OIM 的主机名 |
| `WLSPORT` | 输入 OIM WebLogic 环境的管理端口 |
| `WLSADMIN` | 输入 WebLogic 管理员用户名 |
| `DOMAIN_NAME` | 输入安装了 OIM 受管服务器的 WebLogic 域的名称 |
| `OIM_MANAGED_SERVER_NAME` | 输入 OIM 使用的受管服务器的名称 |
| `DOMAIN_LOCATION` | 输入指向域文件的完整文件系统路径 |

文件创建完成后，你现在可以运行带 `–configOIM` 选项的 `idmConfigTool` 来设置 OAM 以进行集成。运行此步骤时，将配置 OIM Identity Store 并设置必要的系统 MBean。系统将提示你输入 SSO Access Gate、Identity Manager MDS 数据库模式、ID 存储密码和管理服务器密码。你还需要为 `ssoKeyStore.jks` 和 SSO 全局密码提供密码。如果没有报告错误，请重启 OIM 环境，包括 OIM 受管服务器、SOA 受管服务器和管理服务器。你应该检查 `Automation.log` 文件是否有任何错误。



## 配置 Oracle HTTP Server WebGate

至此，您已在一个集成环境中配置了 OAM 和 OIM。OAM 为您的应用程序和受保护资源提供单点登录环境。在第 8 章中，您安装并配置了 OHS 和 Oracle WebGate 11g。OAM 和 WebGate 协同工作以保护资源并执行身份验证和授权服务。随着 OIM 组件的加入，您必须使用必要的资源策略配置 OHS、WebGate 和 OAM，以启用 SSO 和自动流程。

在 OAM 主目录中，`IAM_HOME/server/setup/templates` 目录下有一个名为 `oim.conf` 的文件。将此文件复制到您的 OHS 配置目录，并根据您的安装情况使用 OIM 信息对其进行修改。这将配置 HTTP 服务器托管的各个位置，使其受到 WebGate 的保护。该文件至少应包含以下条目。

```
#Callback webservice for SOD

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#oim self and advanced admin webapp consoles(canonic webapp)

SetHandler weblogic-handler
WLCookieName    oimjsessionid
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#xlWebApp - Legacy 9.x webapp (struts based)

SetHandler weblogic-handler
WLCookieName    oimjsessionid
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#Nexaweb WebApp - used for workflow designer and DM

SetHandler weblogic-handler
WLCookieName    oimjsessionid
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#used for identity.

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#used for sysadmin.

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#used for provisioning-callback.

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#used for FA Callback service.

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

