# Oracle HTTP Server 安装与配置

默认情况下，安装将使用端口 `7777` 作为 OHS 服务器端口。如果您希望使用不同的端口，例如 `80`、`443` 或两者，可以通过修改 `staticports.ini` 文件来配置。或者，您也可以在安装后更新位于实例配置目录中的 `httpd.config` 文件。图 8-8 展示了使用“自动端口配置”设置的安装过程。

![A352855_1_En_8_Fig8_HTML.jpg](img/A352855_1_En_8_Fig8_HTML.jpg)
*图 8-8. 端口配置*

如图 8-9 所示的“安装摘要”屏幕为您提供将要安装的组件概览。点击“安装”继续。

![A352855_1_En_8_Fig9_HTML.jpg](img/A352855_1_En_8_Fig9_HTML.jpg)
*图 8-9. 安装摘要屏幕*

通常，此过程需要 5 到 10 分钟。如图 8-10 所示的“安装进度”屏幕将显示进度以及实际进行的过程。如果出现任何错误，请参考屏幕上显示的日志文件。

![A352855_1_En_8_Fig10_HTML.jpg](img/A352855_1_En_8_Fig10_HTML.jpg)
*图 8-10. 安装进度屏幕*

安装程序完成安装后，将立即开始根据先前屏幕上的输入配置新创建的实例。每个步骤可能都需要一点时间。配置完成后，安装程序将启动 OHS 进程。请监控图 8-11 所示的“配置进度”屏幕，如果出现任何错误，请检查指示的日志文件。

![A352855_1_En_8_Fig11_HTML.jpg](img/A352855_1_En_8_Fig11_HTML.jpg)
*图 8-11. 配置进度屏幕*

图 8-12 中的“安装完成”屏幕指示了新 Web 层的所有安装和配置详细信息。

![A352855_1_En_8_Fig12_HTML.jpg](img/A352855_1_En_8_Fig12_HTML.jpg)
*图 8-12. 安装完成屏幕*

您已完成 OHS 的安装和配置，现在拥有一个可用作您各种应用程序前端的 Web 服务器。通过 OHS，您将获得 Oracle WebLogic 模块，该模块允许您配置各种位置，由底层 WebLogic 应用服务器处理。`Mod_wl_ohs` 包含处理基于 WebLogic 的应用程序的反向代理和集群故障转移所需的配置。本章下一节将介绍在 HTTP 服务器上安装和配置 OAM WebGate 的步骤。

## Oracle Access Manager WebGate 安装与配置

允许 OAM 保护资源和处理 SSO 的关键组件是 WebGate。当请求进入时，OAM WebGate 会评估请求并确定是否需要认证。如果请求的资源受保护，WebGate 将把认证责任移交给 OAM。一旦 OAM 对用户进行了认证，它会在浏览器中设置会话信息，WebGate 利用这些信息允许用户访问被授权的应用程序。

WebGate 安装过程与其他安装程序类似，从图 8-13 所示的“欢迎”屏幕开始。重要的是要确保版本与您已安装的 OAM 版本兼容。

![A352855_1_En_8_Fig13_HTML.jpg](img/A352855_1_En_8_Fig13_HTML.jpg)
*图 8-13. 安装开始*

OAM WebGate 安装的启动方式与其他 Oracle 应用程序的安装非常相似，使用 Universal Installer。从下载的安装介质 `Disk1` 中，执行：
```
./runInstaller.sh –jreLoc /home/oracle/jdk1.6.0_45/jre
```
如果您在开始时已花时间确保满足所有操作系统先决条件，此屏幕应能正常完成所有检查。如果任何检查失败，请在继续前花时间纠正问题。有关已完成的先决条件检查示例，请参见图 8-14。

![A352855_1_En_8_Fig14_HTML.jpg](img/A352855_1_En_8_Fig14_HTML.jpg)
*图 8-14. 先决条件检查屏幕*

Oracle WebGate 应安装在与 OHS 相同的 Middleware Home 位置中。如果这是在 WebLogic Middleware Home 中完成的，请确保 WebGate 软件共享此主目录。列出的 Oracle Home Directory 在该目录中应该是唯一的，如图 8-15 所示。

![A352855_1_En_8_Fig15_HTML.jpg](img/A352855_1_En_8_Fig15_HTML.jpg)
*图 8-15. 安装位置*

如图 8-16 所示的“安装摘要”屏幕展示了要安装的组件及其位置的摘要。请花时间确保一切看起来正确。

![A352855_1_En_8_Fig16_HTML.jpg](img/A352855_1_En_8_Fig16_HTML.jpg)
*图 8-16. 安装详细信息*

与本书中安装的其他产品一样，OHS 11g WebGate 安装程序提供了一个进度指示器，如图 8-17 所示。如果出现任何错误，您可以监控指示的安装日志。

![A352855_1_En_8_Fig17_HTML.jpg](img/A352855_1_En_8_Fig17_HTML.jpg)
*图 8-17. 安装进度屏幕*

在安装阶段结束时，请记下图 8-18 所示完成屏幕上提供的详细信息。

![A352855_1_En_8_Fig18_HTML.jpg](img/A352855_1_En_8_Fig18_HTML.jpg)
*图 8-18. 安装完成屏幕*

这完成了 OAM WebGate 的安装。在下一节中，您将开始进行 WebGate 在 OHS 环境中的初始配置和部署。


### 配置与部署 Oracle WebGate

既然 `OHS` 和 `OAM WebGate` 软件已经安装完毕，现在是将 `WebGate` 部署到 `OHS` 中的时候了。完成此操作的工具位于 `Oracle WebGate` 主目录中。该目录是在安装过程中选择的。

使用命令提示符，导航至以下目录：
```
/webgate/ohs/tools/deployWebGate
```

`deployWebGateInstance.sh` 工具需要两个参数：`WebGate 主目录` 和 `WebGate 实例主目录`。这些参数通过 `–w` 和 `–oh` 选项来指定。`WebGate 主目录`（由 `–oh` 指定）是 `WebGate` 软件的安装目录。`WebGate` 实例目录（由 `–w` 指定）是 `OHS` 实例目录，在许多情况下，该目录位于 `Oracle_WT1/instances/instance1/config/OHS/ohs1` 目录下。

命令应类似于：
```
./deployWebGateInstance.sh –w /home/oracle/OHSMiddleware/Oracle_WT1/instances/instance1/config/OHS/ohs1 –oh /home/oracle/OHSMiddleware/Oracle_OAMWebGate11g
```

运行 `deployWebGateInstance.sh` 工具会将所需的代理文件复制到 `WebGate` 实例目录。你可以通过查看目录 `/home/oracle/Oracle_WT1/instances/instance1/config/OHS/ohs1` 来验证这一点。你应该能看到一个 `webgate` 目录和一个 `webgate.conf` 文件。

部署过程的下一步是修改所需的文件，以使 `OHS` 能够识别 `WebGate`。

为此，你必须首先通过运行以下命令确保 `LD_LIBRARY_PATH` 包含了 `OHS` 的 `lib` 文件夹：
```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/oracle/OHSMiddleware/Oracle_WT1/lib
```

接下来，切换到目录 `/home/oracle/OHSMiddleware/Oracle_OAMWebGate11g/webgate/ohs/tools/setup/InstallTools`。从该目录运行命令：
```
./EditHttpConf –w  -oh 
```

前面的命令需要两个参数：`WebGate 主目录` 和 `WebGate 实例主目录`。这些参数通过 `–w` 和 `–oh` 选项来指定。`WebGate 主目录`（由 `–oh` 指定）是 `WebGate` 软件的安装目录。`WebGate` 实例目录（由 `–w` 指定）是 `OHS` 实例目录，在许多情况下，该目录位于 `Oracle_WT1/instances/instance1/config/OHS/ohs1` 目录下。

验证命令是否做出了适当的更改。在 `/home/oracle/OHSMiddleware/Oracle_WT1/instances/instance1/conf/OHS/ohs1` 目录中查找 `httpd.conf` 文件。它现在应该在文件末尾包含以下行：
```
include "/home/oracle/OHSMiddleware/Oracle_WT1/instances/instance1/config/OHS/ohs1/webgate.conf”
```

## 总结

现在你已经拥有一个 `OHS` 和一个 `OAM 11g WebGate` 实例。你的 `OHS` 可用作 Web 服务器来托管需要受 `OAM` 保护的应用程序。基础组件现已就位，可以实施 `Oracle 标识和访问管理套件`。接下来的几章将提供将它们全部整合在一起所需的信息。

## 9. 配置 Oracle 访问管理器

所有必需的组件现已安装完毕。你已经在独立的 `中间件主目录` 中为每个组件创建了一个域。这简化了未来的升级和维护。完成这组操作后，实际的组件需要进行配置并为实际的集成做好准备。配置过程包括根据你的环境设置组件。接下来的几页将讨论如何在你的环境中配置 `OAM` 以支持单点登录 (`SSO`)。

### 准备访问管理器以使用 Oracle Internet Directory

默认情况下，`OAM` 被配置为使用 `WebLogic` 内置的轻量级目录访问协议 (`LDAP`) 用户存储区，作为系统存储区和默认存储区。强烈建议你在生产环境中配置 `OAM` 使用外部数据存储区。虽然你可能不是在设置生产环境，但建议将你的开发或测试环境配置得与生产系统类似。因此，本章将演示如何配置 `OAM` 使用 `Oracle Internet Directory (OID)` 作为其身份存储区。

要使用 `OID` 作为身份存储区，第一步是配置 `WebLogic` 安全域以识别 `OID`。将浏览器指向 `http://host:port/console` 来打开 `OAM WebLogic` 域的管理控制台。在本书中，`OAM WebLogic Server` 管理控制台配置为 `Host 10.0.70.229 Port 17001`。如果你还记得前面的章节，`OID` 使用 `Port 7001`，而 `Oracle Identity Manager (OIM)` 使用 `27001`。图 9-1 显示了 `WebLogic` 管理控制台的安全域。

![A352855_1_En_9_Fig1_HTML.jpg](img/A352855_1_En_9_Fig1_HTML.jpg)
*图 9-1. WebLogic Server 安全域*

登录到 `OAM` 域的管理控制台后，导航到 安全域 ➤ `myrealm`。屏幕显示在域创建期间配置的默认安全域。它由 `DefaultAuthenticator`、`IAMSuiteAgent` 和 `DefaultIdentityAsserter` 组成。在此步骤中，你将添加一个验证提供程序以支持 `OID`。单击 `新建` 以创建新的验证提供程序。这将打开“创建新身份验证提供程序”屏幕，如图 9-2 所示。

![A352855_1_En_9_Fig2_HTML.jpg](img/A352855_1_En_9_Fig2_HTML.jpg)
*图 9-2. 创建 OID 身份验证器*

在此屏幕上，为新的身份验证器提供一个名称。本例中，将其命名为 `OIDAuthenticator`，并在 `类型` 字段中选择 `OracleInternetDirectoryAuthenticator`。单击 `确定`。

图 9-3 显示了 `WebLogic` 身份验证提供程序的重新排序屏幕。

![A352855_1_En_9_Fig3_HTML.jpg](img/A352855_1_En_9_Fig3_HTML.jpg)
*图 9-3. 重新排序提供程序*

创建新的提供程序后，应将其设置为提供程序列表中的第一位。选择新的 `OIDAuthenticator` 旁边的复选框，并使用箭头将其上移到适当的位置。单击 `确定` 保存此步骤。然后，你可以继续设置如图 9-4 所示的 `控制标志` 属性。

![A352855_1_En_9_Fig4_HTML.jpg](img/A352855_1_En_9_Fig4_HTML.jpg)
*图 9-4. 提供程序通用配置*

重新排序提供程序后，你必须配置 `OIDAuthenticator`。首先将控制标志设置为 `Sufficient`。这允许新创建的身份验证器提供身份验证服务，但不要求用户必须存在于 `OID` 用户存储区中才能访问资源。这是必要的，因为某些用户（例如 `weblogic`）可能不存在于 `OID` 中。完成 `OIDAuthenticator` 的配置后，`DefaultAuthenticator` 上的控制标志也必须设置为 `Sufficient`。

在如图 9-5 所示的 `提供程序详细信息` 选项卡上，输入新身份验证器的配置详细信息。在本例中，根据表 9-1 中的值设置这些值。

*表 9-1. OID 身份验证器参数*

| 主机 | 安装 OID 的主机 |
| --- | --- |
| 端口 | OID LDAP 端口（OID 默认为 3060） |
| 主体 | cn=orcladmin 或其他用于身份验证器的主体 |
| 凭据 | 主体使用的密码 |
| 用户基本 DN | cn=users,dc=mycompany,dc=com |
| 组基本 DN | cn=groups,dc=mycompany,dc=com |

![A352855_1_En_9_Fig5_HTML.jpg](img/A352855_1_En_9_Fig5_HTML.jpg)
*图 9-5. 身份验证器详细信息*



根据需要自定义其他字段。在大多数情况下，默认值即可正常使用。点击**保存**继续。如前所述，您需要将 `DefaultAuthenticator` 的控制标志配置为 `Sufficient`。请参阅图 9-6 作为需要更新字段的示例。

![A352855_1_En_9_Fig6_HTML.jpg](img/A352855_1_En_9_Fig6_HTML.jpg)

图 9-6.
默认身份验证器控制标志

在配置完 `OIDAuthenticator` 属性并为 `OIDAuthenticator` 和 `DefaultAuthenticator` 都设置了控制标志后，您需要重启 OAM 托管服务器和 WebLogic 管理服务器。请按照本书前面介绍的步骤重启所有进程。

## 为 Oracle Access Manager 预配置 OID

在 OAM 域配置了 OID 身份验证器之后，接下来需要在 OAM 内配置身份存储。此配置确保 OAM 使用先前配置的 OID 实例。

首次登录 `OAMConsole` 时，您将需要使用 `weblogic` 用户，因为 OAM 此时使用的是默认身份验证器，而 `oamadmin` 此时尚未拥有 `OAMConsole` 的管理员访问权限。因此，此过程的第一步是在 OID 中创建必要的用户和组，为集成做准备。

在命令提示符中，将您的工作目录设置为 `<ORACLE_HOME>/idmtools/bin`，其中 `<ORACLE_HOME>` 是 OAM 的安装目录。本部分的所有 IDM 工具命令都将从此位置运行。

注意
IDM 工具应始终从同一位置运行，以确保参数文件得到正确更新。但是，在分离域（即 OIM 安装在单独的 WLS 主目录或单独的服务器上）的情况下，您需要在 OIM 主目录中运行 `–configOIM` 命令。

在运行 `idmConfigTool.sh` 命令之前，您必须确保环境变量已正确设置。否则将导致各种错误，且并非所有错误都会明确指出原因。请按如下方式设置环境变量：

```
MW_HOME=/home/oracle/OAMMiddlewareHome
JAVA_HOME=
IDM_HOME=/home/oracle/OIDMiddlewareHome
ORACLE_HOME=/home/oracle/OAMMiddlewareHome/Oracle_IAM1
IAM_ORACLE_HOME=/home/oracle/OAMMiddlewareHome/OracleIAM1
```

这些值应反映您特定环境中使用的安装目录。请注意，`MW_HOME` 和 `ORACLE_HOME` 与 Oracle Access Management 安装相关。`IDM_HOME` 应指向 OID 安装目录。`IDM_HOME` 是一个可选参数，如果 OID 安装在单独的服务器上，可以省略。

创建一个名为 `extendOAMPropertyFile` 的文件：

```
extendOAMPropertyFile
IDSTORE_HOST: 10.0.70.229
IDSTORE_PORT: 3060
IDSTORE_BINDDN: cn=orcladmin
IDSTORE_USERNAMEATTRIBUTE: cn
IDSTORE_LOGINATTRIBUTE: uid
IDSTORE_USERSEARCHBASE: cn=Users,dc=mycompany,dc=com
IDSTORE_GROUPSEARCHBASE: cn=Groups,dc=mycompany,dc=com
IDSTORE_SEARCHBASE: dc=mycompany,dc=com
IDSTORE_SYSTEMIDBASE: cn=systemids,dc=mycompany,dc=com
```

`extendOAMPropertyFile` 中的值应按如下方式设置：

*   `IDSTORE_HOST`：应设置为 OID 实例的主机名，或负载均衡 OID 实例的虚拟主机名。
*   `IDSTORE_PORT`：将其设置为 OID 实例的 LDAP 端口。
*   `IDSTORE_BINDDN`：`cn=orcladmin`。此值将被工具用于登录 OID。此用户应具有在 `usersearchbase` 和 `groupsearchbase` 内创建用户和组的权限。
*   `IDSTORE_LOGINATTRIBUTE`：应设置为 LDAP 存储中包含登录名的属性。
*   `IDSTORE_USERSEARCHBASE`：将此属性设置为 ID 存储中存储用户的位置。
*   `IDSTORE_GROUPSEARCHBASE`：将此属性设置为 ID 存储中存储组的位置。
*   `IDSTORE_SEARCHBASE`：包含上述搜索基础的父容器。
*   `IDSTORE_SYSTEMIDBASE`：将存储身份管理套件所用系统 ID 的新容器。

创建一个名为 `preConfigOAMProperties` 的文件，内容如下：

```
vi preconfigOAMPropertyFile
IDSTORE_HOST : 10.0.70.229
IDSTORE_PORT : 3060
IDSTORE_BINDDN : cn=orcladmin
IDSTORE_USERNAMEATTRIBUTE: cn
IDSTORE_LOGINATTRIBUTE: uid
IDSTORE_USERSEARCHBASE: cn=Users,dc=mycompany,dc=com
IDSTORE_GROUPSEARCHBASE: cn=Groups,dc=mycompany,dc=com
IDSTORE_SEARCHBASE: dc=mycompany,dc=com
IDSTORE_SYSTEMIDBASE: cn=systemids,dc=mycompany,dc=com
POLICYSTORE_SHARES_IDSTORE: true
OAM11G_IDSTORE_ROLE_SECURITY_ADMIN:OAMAdministrators
IDSTORE_OAMSOFTWAREUSER:oamLDAP
IDSTORE_OAMADMINUSER:oamadmin
```

此文件中输入的值应反映您环境中使用的值。



`IDSTORE_HOST`：应设置为 OID 实例的主机名，或是负载均衡 OID 实例的虚拟主机名。
`IDSTORE_PORT`：将其设置为 OID 实例的 LDAP 端口。
`IDSTORE_BINDDN`：`cn=orcladmin`。此值将被工具用来登录 OID。此用户应具有在`usersearchbase`和`groupsearchbase`内创建用户和组的权限。
`IDSTORE_LOGINATTRIBUTE`：应设置为 LDAP 存储中包含登录名的属性。
`IDSTORE_USERSEARCHBASE`：将此属性设置为 ID 存储中存储用户的位置。
`IDSTORE_GROUPSEARCHBASE`：将此属性设置为 ID 存储中存储组的位置。
`IDSTORE_SEARCHBASE`：包含前述搜索库的父容器。
`IDSTORE_SYSTEMIDBASE`：将存储身份管理套件所用系统 ID 的新容器。
`POLICYSTORE_SHARES_IDSTORE`：将其设置为`true`。
`OAM11G_IDSTORE_ROLE_SECURITY_ADMIN`：将包含 OAM 管理员的 LDAP 组的名称。
`IDSTORE_OAMSOFTWAREUSER`：将被 OAM 进程使用的 OAM 用户。
`IDSTORE_OAMADMINUSER`：将在 OAM 控制台中拥有管理员权限的新用户。

现在`idmConfigTool.sh`命令已准备好运行，并为 OAM 创建必要的容器、组和用户。按如下方式运行命令。

首先，为 Access Manager 集成扩展 LDAP 身份存储。

```bash
./idmConfigTool.sh –preConfigIDStore input_file=extendOAMPropertyFile
```

输出应类似于以下内容：

```bash
[oracle@clouddemolab bin]$ ./idmConfigTool.sh -preConfigIDStore input_file=extendOAMPropertyFile
Enter ID Store Bind DN Password :
Jan 5, 2016 1:14:42 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING: /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/idm_idstore_groups_template.ldif
Jan 5, 2016 1:14:43 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING: /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/idm_idstore_groups_acl_template.ldif
Jan 5, 2016 1:14:43 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/idstore_tuning.ldif
Jan 5, 2016 1:14:43 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oid_schema_extn.ldif
Jan 5, 2016 1:14:50 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/../oracle_common/modules/oracle.ipf_11.1.1/scripts/ldap/OID_OracleSchema.ldif
Jan 5, 2016 1:15:01 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/fa_pwdpolicy.ldif
The tool has completed its operation. Details have been logged to automation.log
```

如果报告任何错误，请检查`automation.log`文件并在继续之前纠正错误。

LDAP 身份存储预配置完成后，您需要运行创建的第二个文件来填充容器，如下一个示例所示。

```bash
./idmConfigTool.sh –prepareIDStore mode=OAM input_file=preconfigOAMPropertyFile
```

运行此命令时，系统将提示您为`oblixanonymous`、`oamLDAP`和`oamadmin`用户设置密码。输出将类似于以下摘录。

```bash
[oracle@clouddemolab bin]$ ./idmConfigTool.sh -prepareIDStore mode=OAM input_file=preconfigOAMPropertyFile
Enter ID Store Bind DN Password :
*** Creation of Oblix Anonymous User ***
Jan 5, 2016 1:17:21 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING: /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oam_10g_anonymous_user_template.ldif
Enter User Password for oblixanonymous:
Confirm User Password for oblixanonymous:
*** Creation of oamadmin ***
Jan 5, 2016 1:17:33 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oam_user_template.ldif
*** Creation of oamLDAP ***
Jan 5, 2016 1:17:33 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_user_template.ldif
Enter User Password for oamLDAP:
Confirm User Password for oamLDAP:
...
*** Creation of ESSO acl ***
Jan 5, 2016 1:17:47 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/esso_acl.ldif
The tool has completed its operation. Details have been logged to automation.log
```

完成此操作后，必要的用户和组已在 OID 中创建。请使用 Oracle Directory Service Manager (ODSM)工具在 OID 中检查，以确保`oamAdministrators`和`oamAdmin`用户已创建。



