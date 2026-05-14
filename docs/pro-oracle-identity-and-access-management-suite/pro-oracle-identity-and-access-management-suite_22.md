# 15. 将访问管理器与电子商务套件集成

许多组织在 Oracle 电子商务套件 (EBS) 以及 Oracle WebCenter Content 和 Imaging、Oracle Business Intelligence 和面向服务的架构 (SOA) Suite 等辅助应用程序和产品上进行了投资。为了在保持高安全性的同时提高用户生产力，Oracle 访问管理器 (OAM) 提供了单点登录 (SSO) 功能，将这些产品绑定在一起，这样用户在一天中从一个应用程序切换到另一个应用程序时，就不会被反复要求登录。

### 架构

Oracle EBS 设计为与充当其身份存储的 Oracle Internet Directory (OID) 集成。此配置允许使用 OAM 提供 SSO 功能。EBS 管理员会知道，该套件以 `FND_USER` 表的形式拥有自己的用户存储。在 SSO 环境中，EBS 用户通过基于工作流的业务事件系统引发的事件与 OID 同步。

OAM 使用称为 `AccessGates` 的代理与 EBS 集成。这些代理，连同前面讨论的访问管理器 `WebGates`，用于提供从登录到应用程序的无缝过渡。

在 OAM 集成的 EBS 环境中会发生一系列操作流程。当 HTTP 服务器接收到 EBS 请求时，OAM `WebGate` 会拦截该请求并将其路由到 OAM。收到请求后，OAM 会确定资源是否受保护以及所需的身份验证和授权级别。尽管 OAM 提供凭据收集，但它会将凭据移交给 OID 进行身份验证。整个过程详述如下：

```
1.  用户尝试使用浏览器访问 Oracle EBS 资源之一，并被重定向到 EBS AccessGate。
2.  AccessGate 由带有 WebGate 的 Oracle HTTP Server (OHS) 保护。
3.  HTTP 服务器和 WebGate 检查是否存在已验证的会话。如果会话存在，则允许用户访问 EBS。如果会话不存在，请求将被路由到 OAM。
4.  OAM 执行凭据收集。
5.  OAM 根据身份存储验证提交的用户凭据。
6.  OAM 将用户信息提供给 Oracle EBS AccessGate。
7.  EBS AccessGate 将用户信息和请求提交给 EBS 数据库。
8.  Oracle EBS 数据库检查 E 中是否存在与身份存储用户关联的用户。
9.  如果未找到关联的 OID 用户，则将用户重定向到 EBS 链接页面以关联其 Oracle EBS 用户名。
10. 原始请求的资源随有效的已验证 Oracle EBS 用户会话一起返回。
```


### 准备 EBS AccessGate 文件

下载 Oracle EBS AccessGate 文件（包含在补丁 `13704814` 中）后，将它们解压到 `$MW_HOME/appsutil/accessgate/{instance}` 目录下。例如：

```
mkdir -p $MW_HOME/appsutil/accessgate/ebsAG
cd $MW_HOME/appsutil/accessgate/ebsAG
unzip [location to patch 13704814]/p13704814_R12_GENERIC.zip
```

AccessGate 压缩文件包含为 EBS 配置 AccessGate 所需的文件。这些文件将在后续配置步骤中被引用。

*   `fndauth.war`：这是 Oracle EBS AccessGate 应用程序，将部署在下一步骤中创建的受管服务器内。
*   `fndext.jar`：此文件包含启用应用服务器与 EBS 数据库之间通信所需的 Java 库。
*   `txkEBSAuth.xml`：这是一个 ANT 脚本，用于帮助自动化部署 Oracle AccessGate。
*   `fndauth_deployment_plan.tmp`：这是一个模板文件，用于部署自动化工具。
*   `LogConfig.properties`：此文件可用作设置日志记录的示例。
*   `samplecleanup.html`：这是一个示例 HTML 文件，用于在 EBS 注销后清理会话信息。

### 创建 EBS AccessGate 安装目录

Oracle EBS AccessGate 必须安装在 Middleware Home 目录内。从命令行，在 `$MW_HOME/appsutil/accessgate/ebsAG` 处为 AccessGate 创建一个目录。如果您为 AccessGate 使用了不同的实例名称，请将 `ebsAG` 替换为该名称。接下来，将 AccessGate 文件复制到新的实例目录。本例中，`MW_HOME` 目录是指安装了 OAM 的 Middleware 安装目录。

### 准备 EBS 和 OID

在开始集成过程之前，您应确保在 EBS 和 Oracle 身份与访问管理环境中都已应用了必要的补丁和更新。请咨询 Oracle 支持以规划并实施所需的兼容性更新。

### 将 EBS 主目录注册到 OAM

将 EBS 与 OAM 集成的第一步是将 EBS 主目录注册到 OAM。这必须在尝试集成 OID 之前完成。每个 EBS 部署只需注册实例一次。即使在 EBS 的多节点实例中也是如此。

注意：在尝试与 OID 集成之前，您必须先将 EBS 主目录注册到 OAM。

首先，运行位于 `EBS FND_HOME` 目录中的 `txkrun.pl` 脚本。

```
$FND_TOP/bin/txkrun.pl -script=SetSSOReg -registerinstance=yes -infradbhost=10.0.70.221 -ldapport=3060 -ldapportssl=3131 -ldaphost=10.0.70.229 -oidadminuserpass=***** -appspass=*****
```

在此命令中，按照表 15-1 所述设置参数。

表 15-1. SSO 注册参数

| 参数 | 描述 |
| --- | --- |
| `-script` | `SetSSOReg` 设置 `txkrun.pl` 脚本来运行 EBS 实例与 OID 的注册过程。 |
| `-registerinstance` | 将此设置为 `yes` 以确保注册 EBS 实例。 |
| `-infradbhost` | 此参数应设置为 OID 数据库主机名。请注意，这与 EBS 数据库不同。 |
| `-ldapport` | 将此设置为非安全套接字层（SSL）轻量级目录访问协议（LDAP）端口。 |
| `-ldapportssl` | 将此设置为 OID 的 SSL LDAP 端口。 |
| `-ldaphost` | 将此设置为运行 OID 的服务器的主机名。 |
| `-oidadminuserpass` | 在此输入 `orcladmin` 密码。 |
| `-appspass` | 在此输入 EBS 应用程序数据库管理员（DBA）密码。 |

### 将 EBS 注册到 OID

成功将 EBS 实例注册到 OAM 后，您可以继续注册 EBS 和 OAM。为此，您将运行与上一步相同的 `txkrun.pl`。

```
$FND_TOP/bin/txkrun.pl -script=SetSSOReg -registeroid=yes -ldaphost=10.0.70.229 -ldapport=3060 -oidadminuserpass=***** -appspass=***** -instpass=***** -provisiontype=4
```

表 15-2 显示了此脚本设置的参数。

表 15-2. OAM 注册参数

| 参数 | 描述 |
| --- | --- |
| `-script` | `SetSSOReg` 设置 `txkrun.pl` 脚本来运行 EBS 实例与 OID 的注册过程。 |
| `-registeroid` | 将此标志设置为 `yes` 以注册 EBS 到 OID。 |
| `-ldaphost` | 将此设置为 OID 主机名。如果您使用负载均衡器，应设置为实际主机名，而不是 VIP。 |
| `-ldapport` | 将此设置为 OID 实例使用的 LDAP 端口。 |
| `-oidadminuserpass` | 此应设置为 `orcladmin` 密码或其他管理密码。 |
| `-appspass` | 将此设置为应用程序 DBA 密码。 |
| `-instpass` | 此应设置为与 OID 管理员密码相同的密码。 |
| `-provisiontype` | 此值可以设置为 1、2、3 或 4。本例中设置为 4，表示双向 – 无创建 provisioning。其他值为：1. 双向 Provisioning：这是默认值。2. 入站 Provisioning：EBS 到 OID。3. 出站 Provisioning：OID 到 EBS。4. 双向，无 Provisioning。 |

完成此步骤后，必须重启 EBS 中间层服务。

### 创建 EBS 连接用户

将 EBS 主目录注册到 OAM 并将 EBS 注册到 OID 后，现在可以为连接到 OID 创建所需的本地 EBS 用户。新 EBS 用户应使用 UMS/Apps 模式权限创建。创建此用户后，使用 EBS 本地登录名登录并重置新用户的密码。验证该用户可以连接到 EBS 并具有正确的角色：`UMX|Apps Schema Connect`。

### 配置 EBS AccessGate

由于 Oracle EBS 使用了大量的 cookie 和会话信息，EBS AccessGate 的配置要求其安装在与 EBS 本身使用的中间层服务器相同的域中。在本例中，创建了 `.example.com` cookie 域。确保使用对您的环境必要的相同信息来安装 AccessGate。

### 为 AccessGate 创建受管服务器

在配置 OAM 期间，已为 OAM 创建了 WebLogic Server (WLS) 域。因此，您可以为新的 AccessGate 使用相同的 WLS 和域。登录到 OAM WLS 管理控制台后，导航到 Environment > Clusters，然后创建一个新集群。为集群分配适合您环境的名称。本例中使用名称 `ebsAG_cluster`。使用以下值配置集群参数。

*   消息传递模式：单播 (Unicast)。
*   单播广播通道：留空。
*   多播地址：使用默认值。
*   多播端口：使用默认值。

创建新集群后，就可以配置受管服务器。导航到 Environment > Servers。在现有受管服务器列表中，单击 New。使用以下值填充字段：

*   服务器名称：`ebsAG_server1`
*   服务器侦听主机：`<OAM 主机名>`
*   服务器侦听端口：`17043` 或其他开放端口
*   集群：选择上一步创建的集群名称。

创建受管服务器后，编辑新服务器并将其分配给现有的计算机。

注意：在集群环境中创建此配置时，请确保创建两个受管服务器，并将每个服务器分配到集群中相应的计算机上。


### 复制制品文件

在本章前面部分，您已经下载了 AccessGate 文件并创建了安装目录。当时您已看到下载的 zip 文件中包含的文件。在这里，您需要将这些文件复制到已创建的安装目录内的正确位置。

首先，将 `samplecleanup.html` 从 `$MW_HOME/appsutil/accessgate/ebsAG/sample` 目录复制到 OHS 安装目录内 htdocs 位置下创建的 `/public` 目录中。按照如下方式将文件重命名为 `oacleanup.html`：

```
cp $MW_HOME/appsutil/accessgate/ebsAG/sample/samplecleanup.html
/instances/instance1/config/OHS/ohs1/htdocs/public/oacleanup.html
```

将文件复制到正确目录后，使用 Web 浏览器访问它。导航至 [`http://10.0.70.229:7777/public/oacleanup.html`](http://10.0.70.229:7777/public/oacleanup.html)。

由于此页面不是受 OAM 保护的资源，因此您应该能够直接访问它而无需提示登录信息。但是，此页面将显示为空。此 URL 将在后续步骤中使用，届时您将部署实际的 WebGate 并配置一个集中注销页面。

接下来，您需要将 `fndext.jar` 文件复制到 OAM 的 `lib` 目录。执行以下命令来完成此操作。

```
cp $MW_HOME/appsutil/accessgate/ebsAG/fndext.jar $MW_HOME/user_projects/domains/OAMDomain/lib
```

重启 OAM 托管服务器和管理服务器，以加载您复制到 `lib` 目录的新 `fndext.jar` 文件。

### 在 EBS 中生成 DBC 文件

EBS 使用数据库连接文件来为 AccessGate 创建连接池，以便连接到 EBS。在本节中，您将使用 EBS AdminDesktop 实用程序来生成所需的 DBC 文件。

设置环境变量，并确保 `JAVA_HOME` 指向 OAM 实例的 JAVA 目录。使用以下命令为您的实例生成 DBC 文件。

```
java oracle.apps.fnd.security.AdminDesktop apps/ CREATE NODE_NAME=10.0.70.229 DBC=$FND_SECURE/ebsAG.dbc
```

在上面的命令中，`NODE_NAME` 是 OAM 实例所在的主机名。为了在具有多个 EBS AccessGate 的环境中更容易参考，可以将 DBC 文件命名为与 AccessGate 和 EBS 实例相关的内容。将生成的文件复制到 OAM 服务器上 EBS AccessGate 的安装目录下，在本例中为 `<MW_HOME>/appsutil/accessgate/ebsAG`。

### 将 EBS AccessGate 主机添加到外部表列表

在部署 EBS AccessGate 期间，定义了一个 Java 数据库连接（JDBC）连接。默认情况下，EBS 数据库不允许来自外部机器的通信。为了允许此通信，托管 OAM AccessGate 的 WLS 必须在 EBS 数据库中注册为受信任的机器。为此，您将使用 EBS Java 开发工具包（JDK）。

出于安全目的，Oracle 建议使用互联网协议（IP）地址限制来防止未经授权的访问。通过将 EBS AccessGate 主机名添加到 EBS 配置文件中，您可以设置适当的限制。在用户级别将 `FND_SERVER_DESKTOP_USER` 配置文件设置为包含 AccessGate 的主机名列表。需要注意的是，此值可以接受以逗号分隔的外部机器列表。如果未执行此步骤，用户尝试认证时可能会导致用户名/密码无效的错误。

### 使用 txkEBSAuth.xml 部署 AccessGate

您已配置了设置 AccessGate 所需的组件。作为这些步骤的一部分，您已暂存软件、注册了 AccessGate 主机、配置了 AccessGate 集群和托管服务器，并设置了中间件主目录结构。现在是时候使用 WLST 实用程序部署 AccessGate 应用程序了。

开始之前，确保环境变量设置正确非常重要。未能正确设置可能会导致配置问题。您可以运行位于 `$DOMAIN_HOME/bin` 目录中的 `setDomainEnv.sh` 来执行此步骤：

```
. ./setDomainEnv.sh
```

您也可以手动设置环境变量：

```
export MW_HOME=/home/oracle/OAMMiddleware
export DOMAIN_HOME=$MW_HOME/user_projects/domains/oam_domain
```

接下来，使用位于 WLS 主目录中的 `setWLSEnv.sh` 设置 WLST 环境。

```
. $MW_HOME/wlserver_10.3/server/bin/setWLSEnv.sh
```

切换到您在上一步中安装 Oracle EBS AccessGate 的目录；例如：

```
cd $MW_HOME/appsutil/accessgate/ebsAG
```

部署 EBS AccessGate 要求您为托管服务器创建一个数据源。执行此步骤会使用 ANT 和前面讨论的 `txkEBSAuth.xml` 模板。此文件位于 AccessGate 安装目录中。运行以下命令时，请注意它可能需要相当长的时间才能完成。请让进程运行而不中断。最后，您将看到一条完成消息。

```
ant -f txkEBSAuth.xml createDataSource \
-Dwlshosturl=10.0.70.229:17001 \
-DdataSourceName=EBS_DS \
-DdataSourceJNDIName=jndi/EBS_DS \
-DasadminUser=OAM_EBS_USER \
-DdbcFile=$MW_HOME/appsutil/accessgate/ebsAG/ebsAG.dbc \
-DserverName=ebsAG_server1 \
-DdeploymentName=ebsauth_dev \
-DfndauthWarFile=$MW_HOME/appsutil/accessgate/ebsAG/fndauth.war \
-DplanPath=$MW_HOME/appsutil/accessgate/ebsAG/plan/Plan.xml \
-DSSOServerRelease=11 \
-DSSOServerURL=http://10.0.70.229:14100 \
-DWebgateLogoutURL=http://10.0.70.229:7777/public/oacleanup.html
构建成功：38 分 29 秒
```

该命令将保存在实例主目录中。完整路径和文件名为：

```
$MW_HOME/appsutil/accessgate/EBS_DB/createDS.sh.
```

创建数据源后，您将使用 WLS 管理控制台将数据源定向到环境中适当的托管服务器。使用浏览器导航到 OAM WebLogic 管理控制台。访问 [`http://10.0.70.229:17001/console`](http://10.0.70.229:17001/console)，并使用 weblogic 用户登录。

进入 WebLogic 管理控制台后，导航到“服务”➤“数据源”。选择名为 `EBS_DS` 的新数据源。单击“目标”选项卡，并选择之前创建的新 `ebsAG` 托管服务器。完成后，单击“保存”以完成配置。此步骤无需重启。

确保数据源已配置并定向到正确的托管服务器后，您就可以在 WLS 上部署 AccessGate 应用程序了。再次，您将使用 ANT 来执行此步骤；例如：

```
ant -f txkEBSAuth.xml deployApplication \
-Dwlshosturl=10.0.70.229:17001 \
-DdataSourceName=EBS_DS \
-DdataSourceJNDIName=jndi/EBS_DS \
-DasadminUser=OAM_EBS_USER \
-DdbcFile=$MW_HOME/appsutil/accessgate/ebsAG/ebsAG.dbc \
-DserverName=ebsag_server1 \
-DdeploymentName=ebsauth_dev \
-DfndauthWarFile=$MW_HOME/appsutil/accessgate/ebsAG/fndauth.war \
-DplanPath=$MW_HOME/appsutil/accessgate/ebsAG/plan/Plan.xml \
-DSSOServerRelease=11 \
-DSSOServerURL=http://10.0.70.229:14100 \
-DWebgateLogoutURL=http://10.0.70.229:7777/public/oacleanup.html
```



此命令是交互式的。虽然您可以在命令中提供参数值，但建议您以交互方式运行命令，并根据提示输入值，以防止以明文形式输入密码，并避免因输入错误而导致实施问题。请参考表 15-3，根据提示输入脚本变量。

## AccessGate 部署参数

| 脚本变量 | 描述 |
| --- | --- |
| `-dDeploymentName` | 输入一个可用于标识应用程序部署的值。在本例中，该值以 `ebsauth_` 为前缀，后接所使用的 EBS 实例 `dev`。此值还将用作受 WebGate 保护的上下文根。 |
| `-DasadminUser` | 输入先前创建的、具有 UMX/APPS 模式连接权限的 EBS 用户。您应确保可以使用此用户在 EBS 中本地登录。 |
| `-DasadminPassword` | 运行脚本时的可选参数。出于安全考虑，您可以选择不包含它。如果命令中未包含此参数，命令运行时将会提示您输入。 |
| `-DWebGateLogoutURL` | 此值应设置为之前创建的 `oacleanup.html` 文件的完整 URL。例如，应输入为 [`http://10.0.70.229:7777/public/oacleanup.html`](http://10.0.70.229:7777/public/oacleanup.html)。请勿使用相对 URL。 |
| `-DOAMLogoutURL` | 输入 OAM 全局登出的完整 URL；例如，[`http://10.0.70.229:7777/oam/server/logout`](http://10.0.70.229:7777/oam/server/logout)。 |
| `-DserverName` | 将此值设置为您在 WebLogic 环境中使用的部署服务器。在上一步中，您创建了一个受管服务器。作为最佳实践，请勿将任何内容部署到管理服务器。最好使用专用的受管服务器。本例中使用 `ebsag_server1`。 |
| `-Dwlspwd` | 根据提示输入 weblogic 用户密码。虽然您可以将其包含在命令中，但最好不要在命令中包含明文密码。 |

至此，Oracle EBS AccessGate 应用程序已部署在 OAM WLS 环境中。您还创建了支持该应用程序所需的数据源。应用程序已配置为知道如何使用管理员用户连接到 EBS。

### 验证 AccessGate 应用程序部署

在配置了 Oracle EBS AccessGate 应用程序资源并将应用程序部署到受管服务器后，建议在继续之前验证其操作并检查是否存在任何错误配置。如果发现任何问题，此时是纠正它们的时机。要验证应用程序设置，您可以使用位于 `http://<hostname>:<adminserver_port>/console` 的 OAM WLS 管理控制台；例如，[`http://10.0.70.229:17001/console`](http://10.0.70.229:17001/console)。

登录 WLS 管理控制台后，导航到 Environment ➤ Servers，并确保新创建的 EBS AccessGate 受管服务器 `ebsag_server1` 正在指定端口上运行。

接下来，检查为 EBS AccessGate 受管服务器创建的数据源。使用左侧菜单，转到 Services ➤ DataSources，并检查您在部署期间创建的数据源（例如 `ebsAG_ds`）是否存在，且其目标指向正确的受管服务器（例如 `ebsag_server1`）。单击数据源链接以查看设置。在设置视图中，使用 Connection Pool 选项卡，确保 Properties user 和 dbcFile 的值根据部署参数中的指定正确无误。您还应使用 Monitoring 选项卡确保数据源已启用并正在运行。如果需要，您还可以在 Monitoring 选项卡上使用 Testing 实用程序来测试连接。

验证受管服务器和数据源后，导航到 Deployments，并查找名为 `ebsauth_dev` 的 Oracle EBS AccessGate 应用程序。它应处于已部署、活动状态且运行状况正常。如果不是这种情况，请尝试停止并重新启动该部署。检查 WLS 日志中是否有任何可能表明配置问题的错误。

如果所有组件都已启动并运行，您应该能够使用浏览器访问 AccessGate URL。导航到 `http://<hostname>:<EBSAccessGatePort>/<accessgatename>/ssologout_callback`；例如，[`http://10.0.70.229:17043/ebsauth_dev/ssologout_callback`](http://10.0.70.229:17043/ebsauth_dev/ssologout_callback)。如果成功，您将看到一个空白页面。如果收到任何错误页面，请检查服务器是否存在故障。

### 在 Oracle Access Manager 中配置资源

配置好 AccessGate 应用程序后，就该配置 OAM 和相应的 WebGate 来保护 EBS 应用程序 URL 了。为此，您将使用图 15-1 所示的 OAM `oamadmin` 屏幕。使用浏览器登录 `OAMConsole` 后，导航到 Application Security Launch Pad，然后单击 Application Domains 以检索当前配置的域列表。您将使用本书前面创建的 WebGate 域。在此域中，您将创建受 OAM 保护的新资源。

![A352855_1_En_15_Fig1_HTML.jpg](img/A352855_1_En_15_Fig1_HTML.jpg)
*图 15-1. Oracle Access Manager WebGate 配置*

新资源应按表 15-4 所示进行配置。

## 用于 EBS 集成的受保护资源和公共资源

| 资源类型 | 主机标识符 | 资源 URL | 认证策略 | 授权策略 |
| --- | --- | --- | --- | --- |
| HTTP | WebGate | `/**` | 公共资源 | 公共资源 |
| HTTP | WebGate | `/public/index.html` | 公共资源 | 公共资源 |
| HTTP | WebGate | `/excluded/index/html` | 排除 | 排除 |
| HTTP | WebGate | `/ebsauth_dev/` | 受保护资源 | 受保护资源 |
| HTTP | WebGate | `/ebsauth_dev/…/*` | 受保护资源 | 受保护资源 |
| HTTP | WebGate | `/ebsauth_dev/OAMLogin.jsp` | 公共资源 | 公共资源 |
| HTTP | WebGate | `/ebsauth_dev/style/` | 公共资源 | 公共资源 |
| HTTP | WebGate | `/ebsauth_dev/style/…/*` | 公共资源 | 公共资源 |
| HTTP | WebGate | `/ebsauth_dev/ssologout.do` | 公共资源 | 公共资源 |
| HTTP | WebGate | `/ebsauth_dev/ssologout_callback` | 公共资源 | 公共资源 |
| HTTP | WebGate | `/oacleanup.html` | 公共资源 | 公共资源 |
| HTTP | WebGate | `/cgi-bin/printenv` | 受保护资源 | 受保护资源 |



### 将 HTTP 服务器重定向到 WebLogic 服务器以访问 EBS AccessGate

正如本章开头所述，HTTP 服务器会将请求重定向到 AccessGate 进行认证。此重定向必须在 OHS 上配置。WebGate 将作为 Oracle EBS 资源认证请求的代理。因此，该请求将由部署在您的 WLS 实例上的 Oracle EBS AccessGate 应用程序处理。

使用命令提示符，找到第 9 章中创建的 OHS 实例的配置目录。该目录应位于 `/home/oracle/OHSMiddleware/Oracle_WT1/instances/instance1/config/OHS/ohs1`。在此目录中，您将找到名为 `mod_wl_ohs.conf` 的 WLS 模块配置文件。默认情况下，`httpd.conf` 文件应包含必要的 include 指令，以便在启动时加载此配置。将以下几行添加到 `mod_wl_ohs.conf` 文件中，以包含 EBS 位置，如下所示。

```
WebLogicHost 10.0.70.229
WebLogicPort 17043

SetHandler weblogic-handler
```

此条目配置 OHS，将 EBS 认证请求重定向到部署了 AccessGate 的托管服务器。确保 `WebLogicHost` 和 `WebLogicPort` 指向安装的正确位置。重启 OHS。

现在，您应该能够通过浏览器从您的 HTTP 服务器和 WebGate 访问 Oracle ECBS AccessGate 资源；例如，[`http://10.0.70.229:7777/ebsauth_dev/ssologout_callback`](http://10.0.70.229:7777/ebsauth_dev/ssologout_callback)。

此 URL 已在 OAM 应用程序域资源下的公共资源配置策略中配置。因此，此时不应提示您进行认证。OHS 现在已配置为代理 EBS AccessGate 应用程序部署作为认证服务提供者。

## 配置集中式登出

您可能记得，OAM 充当凭据收集代理，并根据配置的身份存储（本例中为 OID）对用户进行认证。当用户登录到受保护资源时，OAM 会创建必要的会话和 Cookie，以确保无缝过渡到其他受保护资源，并提供 SSO 环境。在登出过程中，OAM 会清理其自身的会话信息。然而，它不会清理合作伙伴应用程序的会话。因此，用户可能已从 OAM 登出，但 Oracle EBS 会话 Cookie 可能仍然存在，直到浏览器完全关闭。AccessGate 软件安装提供了必要的文件，以确保在执行登出操作时清理 EBS 会话信息，从而增强安全性。配置集中式登出将确保在用户从 SSO 应用程序登出时执行清理操作并消除 EBS 会话。

### 为登出配置清理文件

当您解压 EBS AccessGate zip 文件时，其中包含一个名为 `oacleanup.html` 的文件。在部署过程中，您已将此文件复制到 OHS `htdocs` 目录下的 `/public` 子目录中。找到该文件并使用文本编辑器根据需要更新上下文根，以确保脚本被正确调用。

在 `oacleanup.html` 中找到以下行：

```
function doLoad()
```

更新此条目，将 `<CONTEXT_ROOT>` 替换为在 OAM 中配置的资源名称。此名称应与托管服务器的名称匹配；例如，`//ssologout_callback`。

### 配置其他登出回调

某些环境可能拥有多个 EBS AccessGate 实例或其他应用程序，您可能希望在登出后为其执行会话清理。如果是这种情况，您可以为每个额外的 AccessGate 或自定义应用程序添加如下回调。在 `oacleanup.html` 文件中搜索以下行：

```
function doLoad()
```

为其他应用程序添加回调。

```
logoutHandler.addCallback('//ssologout_callback');
logoutHandler.addCallback('http://webgatehost2.example.com:7777/ /ssologout_callback');
```

在登出过程中，这确保 AccessGate 调用清理过程以销毁 Oracle EBS 会话。OAM 集中式登出还应包括对 OAM 配置保护的其他合作伙伴应用程序的清理。

在此示例中，使用了 Oracle 11g WebGate。因此，可以利用 11g WebGate 登出回调 URL 实现集中式登出过程。使用 OAM 控制台管理界面设置登出回调的 URL。

在 OAM 管理控制台中，单击“系统配置”选项卡。在“访问管理器”部分，单击“SSO 代理”节点，然后选择“OAM 代理”。单击“搜索”并选择先前创建的 SSO 代理。图 15-2 显示了设置 SSO 代理 WebGate 时使用的配置参数。

![A352855_1_En_15_Fig2_HTML.jpg](img/A352855_1_En_15_Fig2_HTML.jpg)

**图 15-2.** 登出回调 URL 配置

将“登出回调 URL”的值设置为 `http://<ebshost>.<domain>:<port>/OA_HTML/AppsLogout`。

至此，OAM 和 AccessGate 集中式登出过程的配置完成。

### EBS 配置文件配置

配置完 Oracle EBS AccessGate 应用程序和 OAM 资源后，需要在实际的 EBS 环境中设置登录配置文件。设置表 15-5 中的 Oracle EBS 配置文件选项，或让 EBS 管理员进行设置。

**表 15-5.** EBS 配置文件配置参数

| 配置文件 | 级别 | 值 |
| :--- | :--- | :--- |
| 应用程序认证代理 (`APPS_AUTH_AGENT`) | 站点 | `http://<webgatehost>:<port>/ebsauth_dev/` |
| 应用程序 SSO 类型 (`APPS_SSO`) | 站点 | `SSWA w/SSO` |
| 应用程序单点登录提示 Cookie (`APPS_SSO_HINT_COOKIE_NAME`) | 站点 | `<空白> (null)`<br>此项应为空；如果存在诸如 `ORASSO_AUTH_HINT` 的值，请将其移除。 |

重启 Oracle EBS 中间层上的应用程序服务，然后重启部署了 Oracle EBS AccessGate 的 WLS 和托管服务器。

### 测试电子商务套件单点登录

现在所有必要的组件都已部署和配置，并且您已为 SSO 设置了 Oracle EBS 配置文件，可以测试整个环境了。此前，您已设置了一个测试 URL 以确保正确的认证机制已就位。在此步骤中，您将尝试登录 Oracle EBS 系统，随后登录另一个受保护资源。如果一切正常，在浏览器会话中您应仅被提示进行一次认证。对受保护资源的其他请求应使用现有的会话信息和 Cookie。在登出时，所有会话信息都应被清理，后续请求将被提示进行认证。使用浏览器导航至 EBS 登录页面。



