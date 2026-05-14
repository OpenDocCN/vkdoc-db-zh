# 配置 OAM 集成

创建文件后，你现在可以使用 `–configOAM` 选项运行 `idmConfigTool` 来为集成设置 OAM。运行此步骤将配置 OAM 身份存储并注册必要的认证策略。系统将提示你配置 OAM 软件用户、OAM 管理员用户和 OAM WebLogic 用户的密码。如果没有报告错误，请重启 OAM 环境，包括 OAM 受管服务器和管理服务器。输出应与下面类似。你还应该检查 `Automation.log` 文件是否有任何错误。

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

你现在已经配置好了 OAM，为与 OIM 的集成做好了准备。要继续此过程，你将使用 `idmConfigTool.sh` 命令。如果你在拆分域中工作，或者你的 OIM 实例位于单独的服务器上，请移至该服务器或 OIM 软件主目录。

## 配置 Oracle Identity Manager

OIM 需要一个系统用户来连接身份存储，并浏览、创建和编辑用户。就像你为 OAM 配置所做的那样，你将使用 `idmConfigTool` 命令来完成此任务。以下说明将指导你创建系统账户。

在运行 `idmConfigTool.sh` 命令之前，你必须确保环境变量已正确设置。否则将导致各种错误，其中并非所有错误都会指出原因。按如下方式设置环境变量：

```
MW_HOME=/home/oracle/OAMMiddlewareHome
JAVA_HOME=
IDM_HOME=/home/oracle/OIDMiddlewareHome
ORACLE_HOME=/home/oracle/OAMMiddlewareHome/Oracle_IAM1
IAM_ORACLE_HOME=/home/oracle/OAMMiddlewareHome/OracleIAM1
```

要创建系统账户，首先创建一个名为 `preconfigOIMProperties` 的属性文件，如下所示。

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

表 11-3.
`preconfigOIMProperties` 文件的参数定义

| 参数 | 说明 |
| `IDSTORE_HOST` | 设置为运行 OID 的服务器的主机名 |
| `IDSTORE_PORT` | 应设置为 OID 服务器的 LDAP 端口 |
| `IDSTORE_BINDDN` | `cn=orcladmin` |
| `IDSTORE_USERNAMEATTRIBUTE` | 设置为存储用户名的 LDAP 属性；此值将用于搜索用户 |
| `IDSTORE_LOGINATTRIBUTE` | 将此值设置为存储用户登录名的属性 |
| `IDSTORE_USERSEARCHBASE` | 应设置为你的环境中包含用户的容器 |
| `IDSTORE_GROUPSEARCHBASE` | 输入你的环境中存储组的容器名称 |
| `IDSTORE_SEARCHBASE` | 存储前述 searchbase 值的顶层容器 |
| `POLICYSTORE_SHARES_IDSTORE` | 如果你的身份存储也存储策略信息，则设置为 true |
| `IDSTORE_SYSTEMIDBASE` | 此位置将用于存储来自 Oracle 身份管理的用户协调数据 |
| `IDSTORE_OIMUSER` | 将在 LDAP 存储中创建的供 OIM 使用的账户 |
| `IDSTORE_OIMADMINGROUP` | 输入将存储 OIM 管理员用户的组 |

一旦创建了 `preconfigOIMProperties` 文件，你就可以运行 `idmConfigTools.sh` 命令来准备 OID 身份存储。

在此步骤开始时，你设置了环境变量以支持 `idmConfigTool`。如果尚未完成，请在继续之前重新访问设置步骤。运行 `idmConfigTool`，使用 `–prepareIDStore`，并将模式设置为 OIM，如下一个示例所示。

在 OAM 主目录位置运行：

```
/home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/bin
./idmConfigTools.sh –prepareIDStore mode=OIM input_file=preconfigOIMProperties
```

系统将提示你输入 `orcladmin` 密码并为 `OIMLDAP` 用户和 `XELSYSADM` 用户设置密码。如果一切按预期完成，系统将提示你输入 OIM ldap 用户密码并看到以下输出：

```
Enter ID Store Bind DN Password :
*** Creation of oimLDAP ***
Jan 5, 2016 1:20:01 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_user_template.ldif
Enter User Password for oimLDAP:
Confirm User Password for oimLDAP:
Jan 5, 2016 1:20:06 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_group_template.ldif
Jan 5, 2016 1:20:06 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/common/oim_group_member_template.ldif
Jan 5, 2016 1:20:06 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_groups_acl_template.ldif
Jan 5, 2016 1:20:06 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_reserve_template.ldif
*** Creation of Xel Sys Admin User ***
Jan 5, 2016 1:20:06 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/idm_xelsysadmin_user.ldif
Enter User Password for xelsysadm:
Confirm User Password for xelsysadm:
Jan 5, 2016 1:20:14 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/oid/oim_group_template.ldif
Jan 5, 2016 1:20:14 PM oracle.ldap.util.LDIFLoader loadOneLdifFile
INFO: -> LOADING:  /home/oracle/IAMMiddleware/Oracle_IAM1/idmtools/templates/common/group_member_template.ldif
The tool has completed its operation. Details have been logged to automation.log
```


