# 编辑 OAM 配置文件与 OIM 安装配置

## 对 `oam-config.xml` 文件的编辑

对 `oam-config.xml` 文件的编辑在 OAM 服务器关闭前不会生效。同时，请确保递增文件开头的 `Version` 标签：`<Setting Name="Version" Type="xsd:integer">247</Setting>`。

## 手动更改身份存储

要手动更改身份存储，请找到以下条目并根据你的环境进行编辑。更新文件开头的 `version` 属性并重启 OAM。

```
dc=mycompany,dc=com
false
none
{AES}36AC17B83FF9C6D177B092FD352CB369
false
dc=mycompany,dc=com
false

cn=orcladmin
OID Store

uid
true
true

OID

userPassword
OID

ldap://ldap.mycompany.com:3060
follow

OracleUserRoleAPI
```

## Oracle Identity Manager (OIM)

就其本身而言，OIM 并不是一个难以安装和配置的产品。当与 OID 或其他身份存储一起使用时，配置非常直接。在将 OIM 与 OAM 等 SSO 产品集成时，可能会遇到一些困惑。此集成涉及多个步骤，并且在不同阶段都可能遇到问题。请花时间仔细遵循步骤，你应该就能成功。本节涵盖了一些常见问题。

Oracle Identity Management 二进制文件的安装通常是一个顺利的操作。很少出问题。如果出错，通常是由于之前提到的项目，如缺少操作系统包或 JDK 版本不正确。话虽如此，OIM 的实际配置可能有点令人困惑。很多时候，屏幕上会滚动过大量信息，某个步骤可能看起来正常完成了。然而，一个阶段未解决或遗漏的问题可能会在后续步骤中显现出来。

如前所述，与 OID 和 OAM 不同（它们分别只有一个 `config.sh` 或 `config.bat` 文件），OIM 实际上需要两个配置文件。对于 OIM，你必须先运行位于 `<ORACLE_HOME>/common/bin` 目录下的 `config.sh` 文件来创建域。之后，你将运行 `<ORACLE_HOME>/bin/config.sh` 来实际配置 OIM。对于熟悉 OID 或 OAM 环境但首次安装 OIM 的人来说，这可能会造成混淆。在不正确的时间运行错误的文件将导致无法指示实际问题的错误。

关于配置工具，安装 OIM 所需的错误版本的服务导向架构（SOA）会导致许多问题。更复杂的是，OIM 所需的存储库配置实用程序（RCU）版本与实际 OIM 版本不匹配。OIM 11.1.2.3x 需要 SOA 11.1.1.9。SOA 11.1.1.9 软件作为与 OIM 分开的下载项提供。没有 OIM 11.1.2.3 RCU 可用。相反，你将使用 Fusion Middleware 11.1.1.9 版本。

使用带有 `–configOIM` 标签的 `idmConfigTool.sh` 配置身份存储时，可能会遇到与 `JMXCLIENTLIB` 变量相关的错误。在 Oracle 文档中，有一点不太清楚：即使你没有安装 OIM 开发者控制台，也必须创建 `wlfullclient.jar` 文件。如果你正在配置 OAM 和 OIM 集成，此文件必须存在。文档中，生成 `wlfullclient.jar` 文件的说明被放在一个关于开发者控制台的单独章节中。虽然你不需要将文件复制到任何地方，但遗漏此步骤将导致运行 `idmConfigTool.sh` 文件时出错。要解决此问题，请生成 `wlfullclient.jar` 文件。如果你在同一个服务器上但不同的 WebLogic Server (WLS) 实例中安装 OIM 和 OAM，请确保从 OIM Home 而不是 OAM Home 运行 `idmConfigTool.sh`。这被称为拆分域配置。其次，你可能需要修改你的 `CLASSPATH` 以指向 `wlfullclient.jar` 的位置，而不是 `JMXCLIENTLIB` 目录。



偶尔会需要更改或更新与身份存储的连接。这可能是由于架构变更或需要更换主机的 OID 丢失所致。如果你的环境配置与本书中的示例类似，那么它使用的是 libOVD 连接器。此连接器可以更新，以反映新的 OID 或 LDAP 目录主机及连接信息。在 `<DOMAIN_HOME>/config/fmwconfig/ovd/oim` 目录下有一个名为 `adapters.os_xml` 的文件。在修改此文件之前，请确保管理服务器和 OIM 托管服务器已关闭。查找以下部分，并根据需要更新以反映新的 LDAP 主机。请注意，仅当新的身份存储类型和版本相同，且连接器用户和密码保持不变时，才应使用此方法；例如，OID 11.1.1.9。如果不是这种情况，你可以重新运行第 10 章介绍的 LDAP 同步工具。

## 总结

在 Oracle 身份与访问管理套件的安装和配置过程中，有若干环节可能出现问题。安装后，你可能会遇到同步、认证和授权方面的问题。本章作为入门指南，介绍了一些用于排查和解决常见问题的步骤，这些问题范围从安装问题到运行时和修改问题。

## 索引

### A, B, C

*   管理控制台屏幕
*   管理服务器

### D

*   数据库安全存储
*   目录集成平台 (DIP)
    *   高级选项
    *   连接详情
    *   过滤规则
    *   Fusion Middleware Control 界面
    *   主屏幕
    *   映射规则
    *   配置文件映射
    *   配置文件选择
    *   规则设置
    *   同步配置文件
        *   编辑
        *   启用
    *   排除规则
    *   日志消息
    *   跳过错误
    *   状态
    *   `syncProfileBootstrap` 实用程序
    *   测试器
*   目录集成平台 (DIP)
*   目录服务器配置

### E

*   电子商务套件 (EBS)，访问管理器
    *   AccessGate 文件
    *   架构
    *   清理文件
    *   配置
    *   连接用户
    *   复制构件文件
    *   DBC 文件
    *   主机列表
    *   HTTP 服务器
    *   托管服务器
    *   OAM
    *   OID
    *   配置文件配置
    *   测试
    *   `txkEBSAuth.xml`
    *   验证

### F, G, H

*   Fusion Middleware WLS
    *   组件
    *   主目录配置
    *   安装
    *   JDK 选择屏幕
    *   摘要屏幕
    *   欢迎屏幕

### I, J

*   身份与访问管理
    *   Fusion Middleware
        *   集群
        *   硬件
            *   内存
            *   网络
            *   要求
            *   存储
        *   Fusion Middleware 环境
            *   集群
            *   主机配置
            *   网络规划
        *   OAM
        *   OUD
        *   WLS
    *   OAM
        *   API 和 WSM
        *   云访问门户
        *   身份联合
        *   移动和社交访问
    *   OAAM
    *   OIM
        *   审计
        *   委派管理
        *   自助服务
        *   拓扑
            *   本地高可用性
            *   最大可用性
    *   OAM
    *   OIM
    *   OUD
    *   单节点
*   身份管理服务器
*   身份管理器
    *   数据库准备
    *   组件屏幕
    *   连接详情
    *   密码
    *   先决条件检查
    *   模式创建
    *   摘要屏幕
    *   表空间列表
    *   域配置
        *   管理员密码
        *   集群屏幕
        *   组件
        *   默认端口
        *   JDBC 连接
        *   计算机屏幕
        *   托管服务器
        *   名称和位置
        *   可选屏幕
        *   启动模式
        *   摘要屏幕
        *   WLS 创建
    *   安装
        *   位置屏幕
        *   先决条件检查
        *   进度屏幕
        *   欢迎屏幕
    *   操作系统配置
        *   操作系统包
        *   操作系统用户
    *   SOA 安装
        *   应用服务器
        *   完成屏幕
        *   位置屏幕
        *   先决条件检查
        *   进度屏幕
        *   `runInstaller` 命令
        *   摘要屏幕
*   身份管理器服务器
    *   `IdmConfigTool`
    *   环境变量

### K

*   内核参数

### L, M, N

*   轻量目录访问协议 (LDAP)
    *   安装后
    *   服务器信息
    *   注销回调

### O

*   OAM WebGate
    *   安装
        *   完成屏幕
        *   配置和部署
        *   位置
        *   先决条件检查
        *   进度屏幕
*   Oracle 访问管理器 (OAM)
    *   管理控制台
    *   应用安全选项
    *   认证模块选项卡
    *   认证方案链接
    *   配置
    *   配置问题
    *   配置菜单
    *   目录
    *   域创建
        *   集群配置
        *   组件
        *   JDBC 数据库
        *   位置
        *   计算机配置
        *   托管服务器到集群
        *   托管服务器到计算机
        *   模式配置
        *   可选配置
        *   模式
        *   服务器配置
        *   摘要屏幕
        *   用户密码
        *   欢迎屏幕
    *   `extendOAMPropertyFile`
    *   身份存储
    *   `idmConfigTool.sh` 命令
    *   安装
        *   完成摘要
        *   组件
        *   Middleware Home 目录
        *   先决条件检查
        *   进度条
        *   欢迎屏幕
    *   集成
    *   LDAP 配置
    *   `LDAPScheme` 列表
    *   OID
    *   控制标志属性
    *   创建认证器
    *   `DefaultAuthenticator` 控制标志
    *   身份存储连接
    *   参数
    *   WebLogic Server
    *   操作系统配置
        *   操作系统包
        *   操作系统用户
    *   预配置 OAM 属性
    *   RCU
        *   组件
        *   数据库详细信息
        *   预检查
        *   模式创建
        *   模式密码
        *   摘要屏幕
        *   表空间列表
    *   系统和默认存储
    *   测试连接
    *   故障排除
    *   用户身份存储屏幕
*   Oracle 自适应访问管理 (OAAM)
    *   风险分析组件
    *   面向用户的组件
*   Oracle 目录服务
    *   OID
    *   OUD
    *   OVD
*   Oracle Fusion Middleware 补丁集助手
*   Oracle HTTP 服务器 (OHS)
    *   安装和验证
        *   完成屏幕
        *   组件屏幕
        *   详细信息屏幕
        *   Middleware Home
        *   端口配置
        *   先决条件检查
        *   进度屏幕
        *   摘要屏幕
    *   Web Cache
    *   Web Tier 安装程序
    *   操作系统配置
        *   操作系统包
        *   操作系统用户
*   Oracle HTTP Server WebGate
*   Oracle 身份管理 (OIM)
    *   配置问题
*   Oracle 身份管理器 (OIM)
    *   审计
    *   配置
    *   配置服务器
    *   定制
    *   数据库安全存储
    *   定义
    *   委派管理
    *   集成
    *   LDAP 安装后
    *   预配置步骤
    *   预配置 OID 身份存储
    *   自助服务
    *   UI 定制。沙盒
    *   工作流
*   Oracle Internet Directory (OID)
    *   配置
        *   数据库信息
        *   默认值
        *   实例位置
        *   OVD 信息屏幕
        *   参数
        *   端口配置
        *   领域信息
        *   签名和加密密钥库
        *   摘要信息
        *   URL 和目录位置
        *   WebLogic 域
        *   欢迎屏幕
    *   配置问题
    *   安装
        *   检查屏幕
        *   目录结构
        *   身份管理
        *   进度屏幕
        *   root 脚本
        *   摘要屏幕
        *   类型屏幕
        *   更新屏幕
    *   操作系统配置
        *   操作系统包
        *   操作系统用户
    *   验证
        *   Fusion Middleware Control 屏幕
        *   身份组件状态屏幕
        *   OPMN 控制
        *   `opmnctl status` 命令
*   Oracle 平台安全服务 (OPSS) 模式
    *   数据库安全存储
*   Oracle 进程管理器和通知服务器 (OPMN) 控制
*   Oracle 统一目录 (OUD)
*   Oracle 虚拟目录 (OVD)

### P, Q

*   策略管理
    *   访问策略
    *   管理角色
    *   能力
    *   管理角色
    *   成员角色
    *   控制范围
*   密码策略

### R

*   存储库创建实用程序 (RCU)
    *   配置参数
    *   数据库选择
    *   先决条件检查
    *   模式密码
    *   选择屏幕
    *   摘要屏幕
    *   表空间映射
    *   欢迎屏幕

### S

*   沙盒
    *   内容属性
    *   内容视图
    *   创建
    *   定制链接
    *   徽标图像
    *   结构视图
*   安全套接层 (SSL)
*   安全断言标记语言 (SAML)
*   面向服务的架构 (SOA)
*   单点登录 (SSO) 启用

### T

*   拓扑
    *   拆分配置文件
    *   用户和组群体
*   透明数据加密 (TDE)

### U, V

*   用户和策略存储
    *   LDAP 目录结构
    *   对象类
    *   OID
    *   DIP
    *   目录同步
    *   安全和数据隐私
    *   TDE
    *   易用性和管理
*   OUD
    *   架构
    *   复制
    *   可扩展性
    *   易用性和可管理性
*   OVD
    *   访问管理
    *   聚合
    *   架构
    *   概述

### W, X, Y, Z

*   WebLogic Fusion Middleware Control
*   WebLogic Server (WLS)
    *   组件
    *   环境特性
