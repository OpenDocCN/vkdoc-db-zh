# 11. Oracle 身份与访问管理器集成

在之前的章节中，您安装了 Oracle Internet Directory (OID)、Oracle Access Manager (OAM) 和 Oracle Identity Manager (OIM)，为集成做准备。此集成提供了多项优势，包括提高用户生产力、增强身份安全性，以及减少因忘记密码或账户锁定而产生的帮助请求。

使用集成环境的组织不仅能够利用 OIM 的用户配置和审计功能，还能将用户自助服务功能与 OAM 的单点登录 (SSO) 功能相结合。这两个主要组件的结合为需要管理其账户访问权限、密码或个人资料信息的用户创造了无缝的界面。正如 SSO 环境允许用户登录一次应用程序后，自动登录到他们可能需要的其他应用程序一样，此集成将身份管理器添加为受保护的资源。这通过在某些事件（如更改密码、忘记密码等）后自动登录用户，从而促进了用户自助服务。另一个特性是 OIM 和 OAM 能够通信以确定用户状态，例如首次登录和账户过期。

在本章结束时，您将拥有一个包含一个受保护资源或 URL 的集成环境。从登录屏幕，您的用户将能够在 SSO 环境中使用首次登录、忘记密码和其他身份管理器功能。

### IdmConfigTool

Oracle 提供了一个工具，可用于确保为成功集成执行所有必要的配置。有大量的配置参数必须设置或更新。正确运行此工具可确保在设置过程中实现无缝集成，减少麻烦。

将新的应用程序和资源添加到 OAM SSO 环境可能相对简单。对于大多数 Oracle 产品和应用程序，这涉及在 WebLogic 域中创建一个新的 Oracle Access Manager Identity Asserter 并配置 WebGate。然而，对于 OIM，必须配置许多组件以确保两个组件之间的通信正常进行。幸运的是，Oracle 提供了一个名为 `IDMConfigTool` 的工具。在之前的章节中，您使用此工具预配置了身份存储。在本章中，它用于配置 OAM、OIM 和 WLS 的集成。

`IdmConfigTool` 可以协助完成集成身份管理器组件的任务。它还可用于验证配置参数，并从集成环境中提取配置参数，以协助进行故障排除或在另一环境中进行复制。它支持在单域和拆分域环境、11g 和 10g WebGate 中集成身份管理器组件，并支持多种身份存储类型，如 OID 和 Oracle Unified Directory (OUD)。

虽然 `idmConfigTool` 提供了多种集成工具，但其正常运行取决于正确的环境管理。重要的是要注意您在哪里运行 `idmConfigTool`。在本书中，您在拆分域中设置了 OAM 和 OIM。Oracle 文档提到您应该始终从同一位置运行 `idmConfigTool`。但是，如果在拆分域中运行，您必须在 OAM 主目录位置运行 OAM 特定命令，而 OIM 特定命令则从 OIM 主目录运行。

由于本章涵盖 OAM 和 OIM 任务，因此每个部分都介绍了 `idmConfigTool` 的正确设置。


### 配置 Oracle 访问管理器

在第 9 章中，您配置了 OAM 并为其与 OIM 的集成进行了预配置。您还在 OID 轻量级目录访问协议（LDAP）存储中创建了必要的对象。这一切都是为了启动和运行 OAM 并确保创建了正确的管理用户。您还在 WebLogic 层将 OID 集成为验证器。

如前所述，此环境是一个拆分域，其中 OAM 和 OIM 安装在不同的 Middleware Home 和不同的 WebLogic 域中。因此，OAM 的配置将使用位于 OAM 主目录中的 `idmConfigTool`。有关如何准备环境变量以使用此工具的详细信息，请参见表 11-1。

## 表 11-1. `idmConfigTool` 的环境变量

| `MW_HOME` | 将其设置为 OAM 中间件软件的安装目录；例如，`/home/oracle/IAMMiddleware/` |
| `JAVA_HOME` | 将其设置为 Java 安装目录；例如，`/home/oracle/java` |
| `IDM_HOME` | 如果您在同一物理服务器上安装了 OID，请将其设置为 OID 安装目录；例如，`/home/oracle/OIDMiddleware`；如果 OID 安装在单独的服务器上，请将其保留为空 |
| `ORACLE_HOME` | 将其设置为 OAM 安装目录；例如，`/home/oracle/IAMMiddleware/Oracle_IAM` |

`idmConfigTool.sh` 文件位于 `$ORACLE_HOME/idmtools/bin` 目录中。执行这些步骤时，应从此目录运行各个步骤。在此过程中，将创建并追加一个名为 `automation.log` 的文件。

要启动该过程，请创建一个名为 `OAMConfigPropertyFile` 的文件。此文件将包含用于为 OAM 和 OIM 集成配置 OAM 的相关信息。该文件应包含有关您配置的以下信息。相应地更新值：

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

在此文件中，应如表 11-2 所示设置值。

## 表 11-2. `OAMConfigPropertyFile` 参数值

| `WLSHost` | 安装 OAM 的服务器的主机名 |
| `WLSPort` | OAM WebLogic 管理服务器的监听端口 |
| `WLSADMIN` | OAM WebLogic 管理员用户名 `<weblogic>` |
| `WLSPASSWD` | OAM WebLogic 管理员密码 |
| `ADMIN_SERVER_USER_PASSWORD` | OAM 管理员密码 |
| `IDSTORE_HOST` | 安装了 OID 的服务器的主机名 |
| `IDSTORE_PORT` | OID 的 LDAP 端口 |
| `IDSTORE_BINDDN` | 身份存储管理员用户的完全限定用户名 |
| `IDSTORE_USERNAMEATTRIBUTE` | 指示包含用户用户名的身份存储属性；例如，`cn` |
| `IDSTORE_LOGINATTRIBUTE` | 输入包含用户登录名的属性 |
| `IDSTORE_USERSEARCHBASE` | 此值指示身份存储中 OAM 应开始搜索用户的位置；例如，`cn=users,dc=mycompany,dc=com` |
| `IDSTORE_SEARCHBASE` | 使用身份存储的顶层；例如，`dc=mycompany,dc=com` |
| `IDSTORE_SYSTEMIDBASE` | 输入身份存储中管理员或系统帐户的位置 |
| `IDSTORE_GROUPSEARCHBASE` | 身份存储中组的位置 |
| `IDSTORE_OAMSOFTWAREUSER` | 这应该是 OAM 用于连接到身份存储的用户名；例如，`oamLDAP` |
| `IDSTORE_OAMADMINUSER` | OAM 将用于修改身份存储条目的用户的用户名 |
| `IDSTORE_DIRECTORYTYPE` | 指示用作身份存储的服务器类型；例如，OID、AD、OpenLDAP |
| `POLICYSTORE_SHARES_IDSTORE` | 如果 OAM 策略存储在与身份存储相同的位置，请将此值设置为 `true` |
| `PRIMARY_OAM_SERVERS` | 列出 OAM 主机名 |
| `WEBGATE_TYPE` | 输入 WebGate 的类型：`ohsWebgate11g` 或 `ohsWebGate10g` |
| `ACCESS_GATE_ID` | 输入在 OAM 配置中用作 WebGate 的名称 |
| `OAM11G_IDM_DOMAIN_OHS_HOST` | 安装 Oracle HTTP Server (OHS) 的主机名 |
| `OAM11G_IDM_DOMAIN_OHS_PORT` | OHS 正在监听的端口 |
| `OAM11G_IDM_DOMAIN_OHS_PROTOCOL` | 指示此 OHS 是否为安全套接字层 (SSL)；例如，`http` 或 `https` |
| `OAM11G_WG_DENY_ON_NOT_PROTECTED` | 将此设置为 `true` 可使所有未在 OAM 中另行配置的 URL 模式拒绝访问 |
| `OAM11G_IMPERSONATION_FLAG` | 指示 OAM 是否应允许模拟；这在融合应用程序环境中很有用 |
| `OAM_TRANSFER_MODE` | 配置 OAM 的通信模式；有效值为 `Open` 和 `Simple` |
| `OAM11G_OAM_SERVER_TRANSFER_MODE` | 指示 OAM 将支持的通信类型；请注意，这应与 `OAM_Transfer_Mode` 匹配 |
| `OAM11G_IDM_DOMAIN_LOGOUT_URLS` | 配置将用于注销操作的 URL |
| `OAM11G_OIM_WEBGATE_PASSWD` | 输入 WebGate 的密码 |
| `OAM11G_SERVER_LOGIN_ATTRIBUTE` | 配置用于用户登录的属性名称 |
| `COOKIE_DOMAIN` | 指示 cookie 有效的域名称 |
| `OAM11G_IDSTORE_NAME` | 输入系统身份存储的名称 |
| `OAM11G_IDSTORE_ROLE_SECURITY_ADMIN` | 输入身份存储中包含管理用户的组或角色的名称 |
| `OAM11G_SSO_ONLY_FLAG` | 将此设置为 `true` 仅提供身份验证功能；设置为 `false` 将配置 OAM 同时执行授权检查 |
| `OAM11G_OIM_INTEGRATION_REQ` | 输入 `true` 以指示将集成 OIM 和 OAM |
| `OAM11G_SERVER_LBR_HOST` | 提供 OAM 位置的负载均衡器 URL |
| `OAM11G_SERVER_LBR_PORT` | 提供负载均衡器端口 |
| `OAM11G_SERVER_LBR_PROTOCOL` | 指示 URL 是否为 SSL |
| `COOKIE_EXPIRY_INTERVAL` | 设置此值 |
| `OAM11G_OIM_OHS_URL` | 如果 OAM 和 OIM 前置有 OHS，请提供 URL |
| `SPLIT_DOMAIN` | 如果 OAM 和 OIM 安装在不同的 WebLogic 服务器中，请将此设置为 `true` |


