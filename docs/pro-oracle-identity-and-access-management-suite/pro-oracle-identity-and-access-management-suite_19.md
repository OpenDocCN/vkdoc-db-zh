# 集成 OIM 和 OAM

与配置 OAM 以进行集成的操作类似，您将使用 `idmConfigTool.sh` 命令来执行 OIM 和 OAM 的集成。这涉及创建一个属性文件并使用它来运行 `idmConfigTool.sh –configOIM`。

在运行 `idmConfigTool.sh` 命令之前，您必须确保环境变量已正确设置。若不这样做，将导致各种错误，且并非所有错误都会明确指出原因。请按如下方式设置环境变量：

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

此文件中每个参数的值都应填充您环境中特定的信息。有关值的说明，请参阅表 11-4。

## 表 11-4. `OIMConfigPropertyFile` 参数值

| 参数 | 说明 |
| --- | --- |
| `LOGINURI` | 此值取自原始文件；请勿修改 |
| `LOGOUTURI` | 您可以保留默认值；但是，如果您有符合要求的自定义注销页面，请在此处输入 URL |
| `AUTOLOGINURI` | 将此值设为 none |
| `ACCESS_SERVER_HOST` | 输入安装了 OAM 的主机名 |
| `ACCESS_SERVER_PORT` | 输入 Oracle Access Management 端口 5575 |
| `ACCESS_GATE_ID` | 输入上一步中创建的 Oracle Access Gate 的名称 |
| `COOKIE_DOMAIN` | 输入 cookie 有效的域名 |
| `COOKIE_EXPIRY_INTERVAL` | 输入 cookie 有效的分钟数 |
| `OAM_TRANSFER_MODE` | 输入与上一步相同的通信模式值；例如，open 或 simple |
| `WEBGATE_TYPE` | 指示所使用的 WebGate 版本；有效值为 ohsWebGate11g 和 ohsWebGate10g |
| `OAM_SERVER_VERSION` | 输入 Access Server 的版本号 `11g` |
| `OAM11G_WLS_ADMIN_HOST` | 输入安装了 OAM 的 WebLogic Server 管理服务器的主机名 |
| `OAM11G_WLS_ADMIN_PORT` | 输入管理服务器侦听的端口 |
| `OAM11G_WLS_ADMIN_USER` | 输入 WebLogic |
| `SSO_ENABLED_FLAG` | 输入是否要启用 SSO；例如，true |
| `IDSTORE_PORT` | 输入 OID 或其他 LDAP 服务器正在侦听的端口；这应与身份存储的位置匹配 |
| `IDSTORE_HOST` | 输入安装了 OID 或其他 LDAP 服务器的主机名 |
| `IDSTORE_DIRECTORYTYPE` | 输入所使用的 LDAP 服务器类型；例如，OID |
| `IDSTORE_ADMIN_USER` | 输入 OIM 应使用来连接身份存储的用户名；例如，`cn=oimldap,cn=systemids,dc=mycompany,dc=com` |
| `IDSTORE_LOGINATTRIBUTE` | 指定用于登录名的身份存储属性 |
| `IDSTORE_USERSEARCHBASE` | 指定 OIM 应在其中搜索用户的顶级身份存储位置 |
| `IDSTORE_GROUPSEARCHBASE` | 指定 OIM 应在其中查找组的位置 |
| `IDSTORE_WLSADMIN_USER` | 输入 weblogic |
| `MDS_DB_URL` | 输入元数据数据库连接信息；格式应为 `jdbc:oracle:thin:@hostname:port:servicename` |
| `MDS_DB_SCHEMA_USERNAME` | 输入用于连接元数据存储库数据库模式的用户名 |
| `WLSHOST` | 输入安装了 OIM 的主机名 |
| `WLSPORT` | 输入 OIM WebLogic 环境的管理端口 |
| `WLSADMIN` | 输入用于 WebLogic 管理的用户名 |
| `DOMAIN_NAME` | 输入安装了 OIM 受管服务器的 WebLogic 域的名称 |
| `OIM_MANAGED_SERVER_NAME` | 输入 OIM 使用的受管服务器的名称 |
| `DOMAIN_LOCATION` | 输入域文件的完整文件系统路径 |

文件创建完成后，您现在可以使用 `–configOIM` 选项运行 `idmConfigTool` 以设置 OAM 进行集成。运行此步骤时，将配置 OIM 身份存储并设置必要的系统 MBean。系统将提示您输入 SSO Access Gate、Identity Manager MDS 数据库模式、ID 存储密码和管理服务器密码。您还需要为 `ssoKeyStore.jks` 和 SSO 全局密码短语提供密码。如果没有报告错误，请重新启动 OIM 环境，包括 OIM 受管服务器、SOA 受管服务器和管理服务器。您应检查 `Automation.log` 文件是否有任何错误。

## 配置 Oracle HTTP Server WebGate

至此，您已完成集成环境中 OAM 和 OIM 的配置。OAM 为您的应用程序和受保护资源提供了 SSO 环境。在第 8 章中，您已安装并配置了 OHS 和 Oracle WebGate 11g。OAM 与 WebGate 协同工作以保护资源，并执行身份验证和授权服务。随着 OIM 组件的加入，您必须为 OHS、WebGate 和 OAM 配置必要的资源策略，以启用 SSO 和自动化流程。

### 修改 oim.conf 配置文件

在 OAM 主目录中，`IAM_HOME/server/setup/templates` 路径下有一个名为 `oim.conf` 的文件。将此文件复制到您的 OHS 配置目录，并根据您的安装信息修改其中的 OIM 部分。这将配置 HTTP 服务器托管的各个位置，使其受到 WebGate 的保护。该文件至少应包含以下条目。

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

# OIM Callback webservice for SOA. SOA calls end up at this OIM webservice

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#spml dsml profile

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

#spml xsd profile

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

# Fusion role-sod

SetHandler weblogic-handler
WebLogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

SetHandler weblogic-handler
WLCookieName oimjsessionid
WeblogicHost 10.0.70.229
WeblogicPort 14000
WLLogFile "${ORACLE_INSTANCE}/diagnostics/logs/mod_wl/oim_component.log"

SetHandler weblogic-handler
WeblogicHost 10.0.70.229
WeblogicPort 14100

SetHandler weblogic-handler
WeblogicHost 10.0.70.229
WeblogicPort 14100

```

更新此文件后，停止并重新启动 OHS 以使更改生效。

### 配置 OAM 认证策略

下一步是将 OIM 特定的资源添加到 OAM 认证策略中。您将使用 OAM 管理控制台来添加 OIM 特定的认证策略。

通过在浏览器中导航到 `http://<oam-hostname>:<OAMWLSAdminPort>/oamconsole` 登录到 OAM 管理控制台。使用 OAMAdmin 用户名和您之前设置的密码。访问管理的登录页面如图 11-1 所示。

![A352855_1_En_11_Fig1_HTML.jpg](img/A352855_1_En_11_Fig1_HTML.jpg)
*图 11-1. Oracle Access Manager 管理控制台*

注意：`<OAMWLSAdminPort>` 应设置为 WebLogic 管理服务器的端口号。即使 OAM 托管服务器未运行，您也可以访问 OAM 管理控制台。

在此屏幕上，在“访问管理器”下，单击“应用程序域”链接。如图 11-2 所示的 OAM 应用程序域选项卡显示了开箱即用的现有域。请注意，IAM Suite 和 Webgate 是为您创建的。

![A352855_1_En_11_Fig2_HTML.jpg](img/A352855_1_En_11_Fig2_HTML.jpg)
*图 11-2. 搜索应用程序域屏幕*

在“搜索应用程序域”屏幕上，单击“搜索”以显示已创建域的列表。您应该会看到在第 8 章中创建的原始域，以及本章前面使用 idmConfigTool 创建的 IAM Suite 域。单击 IAM Suite 域进行编辑。IAM Suite 域编辑屏幕请参见图 11-3。

![A352855_1_En_11_Fig3_HTML.jpg](img/A352855_1_En_11_Fig3_HTML.jpg)
*图 11-3. IAM Suite 域摘要*

在第一页，您将看到有关 IAM Suite 域的摘要详细信息，如图 11-4 所示。单击“资源”选项卡以显示此域中的所有受保护资源。

![A352855_1_En_11_Fig4_HTML.jpg](img/A352855_1_En_11_Fig4_HTML.jpg)
*图 11-4. IAM Suite 域资源*

进入“资源”选项卡后，单击“搜索”以显示所有资源 URL 和配置。滚动直到找到 `/identity/**` 资源。单击它以编辑该资源。这将打开如图 11-5 所示的编辑资源屏幕。

![A352855_1_En_11_Fig5_HTML.jpg](img/A352855_1_En_11_Fig5_HTML.jpg)
*图 11-5. 编辑资源屏幕*

在编辑资源屏幕上，将“保护级别”设置为“已排除”。确保“认证策略”和“授权策略”字段为空。单击“应用”。

停留在此页面时，单击“复制”以创建新资源。将“资源 URL”设置为 `/provisioning-callback/**`。单击“应用”以保存更改。您现在已将 OIM URL 设置为 OAM 环境中的资源。

## 总结

在安装了 OIM 和 OAM 的软件组件后，有必要为集成配置每个层。在此过程的这一点上，您应该拥有一个允许您使用 OIM 实用程序创建用户的环境。使用新创建的用户，您可以登录并会立即提示您更改密码并设置密码提示问题。设置完此基本配置文件后，应直接将用户带到最初请求的资源。下一章将演示流程流以及测试集成的过程。

