# 高级压缩

顾名思义，`高级压缩` 是一个可选许可的产品，可让您压缩数据库中的数据。压缩数据需要更少的磁盘空间，这可以节省硬件成本。压缩数据还可以带来更好的性能，因为单次磁盘读取能够比未压缩时读取更多数据。应用程序通常需要更少的磁盘读取次数，从而带来更快的响应时间。由于数据在数据块中被压缩，`缓冲区高速缓存` 可能不需要那么大，因为每个块存储了更多数据。`RMAN` 备份和恢复也更快，因为需要与备份设备之间移动的磁盘块更少。最后，`高级压缩` 对应用程序是透明的。


## 真实应用测试

向数据库系统引入任何重大变更时，面临的其中一个问题是能否对应用进行充分测试，以确保变更不会引发任何问题。真实应用测试（`RAT`）是一项需额外付费的选件，它让数据库管理员能够捕获针对生产数据库运行的实际工作负载。该工作负载可以迁移到另一个数据库，并在目标数据库上回放。随后，`RAT` 会将新性能与原始性能进行分析比较，并报告任何不足之处。这在升级到新 Oracle 版本时非常有益。捕获生产工作负载，将一个副本数据库升级到新版本，并在该升级后的数据库上回放工作负载。数据库管理员将能很好地了解版本变更对工作负载的影响程度（如果有的话）。

## 后续内容

本章概述了多项 Oracle 数据库选件。其中大多数都需要额外付费，您在使用该功能前需要了解这一点。在许多情况下（尽管并非全部），对于企业而言，考虑到节省的时间，这项额外费用是值得的。如果您尚未获得许可证，我强烈建议您购买诊断包。

本章标志着本书的结束。感谢您的阅读，并希望您发现这些建议有助于推进您的 Oracle DBA 职业生涯。您的旅程才刚刚开始，还有很多需要学习，但这应该是一段激动人心的冒险！祝您好运！

## 索引

### A

活动会话历史（`ASH`）
管理文件目录
管理
管理员指南
高级压缩
敏捷开发方法学
告警日志
`DBMS_SYSTEM` 程序包
错误信息修复
控制文件
实例崩溃
实例关闭
实例启动
控制文件丢失
Oracle DBA
Oracle RAC
查询
`ALTER` 命令
`ALTER SESSION` 命令
应用开发者
归档日志模式数据库的配置
重做日志
归档进程（`ARCn`）
Ask TOM 网站
受众，DBA
审计，开启
自动数据库诊断监视器（`ADDM`）
自动诊断知识库命令解释器（`ADRCI`）
程序包创建
问题
清除跟踪文件
跟踪文件
自动列表分区
自动内存管理（`AMM`）
自动工作负载知识库（`AWR`）

### B

后台进程跟踪文件
备份与恢复文档
备份
基表
`DBA_TABLES` 创建
`SYS.OBJ$` 表
`SYS.TAB$` 表
`X$` 表
博客内容
数据库管理员
记忆留存
NewsBlur
问题解决
分享信息
WordPress
书架
缓冲区缓存
检查点
直接路径读取
`SGA`
捆绑补丁（`BP`）

### C

容量规划
云提供商
`CPU` 利用率
数据字典
DBA 定义
预测
内存利用率
资源利用率
`catpvf.sql` 脚本
字符集扫描器（`csscan`）
检查点（`CKPT`）
清理辅助进程（`CLnn`）
清理主进程（`CLMN`）
集群就绪服务（`CRS`）
冷备份文件位置，缺失数据库
`MTTR`
恢复数据库步骤
事务日志
`COLLABORATE`
复杂性
复合分区
概念指南
会议与用户组
累积补丁更新（`CPU`）
客户服务标识（`CSI`）
`MOS` 支持标识符

### D

达拉斯 Oracle 用户组（`DOUG`）
数据库管理员（`DBA`）
答案
云技术
职责与责任
面试中“视情况而定”的回答
回应粗鲁的 `DBA`
数据库配置助手（`DBCA`）
配置选项
创建选项
数据库创建
数据库选项
部署类型
环境变量
快速恢复选项
人为错误识别
管理选项
模式
网络配置
操作进度屏幕
存储选项
模板
用户凭证
数据库文件
`OFA`
在线重做日志
子目录
Windows 系统
数据库参考
数据库系统，成功标准
`DBA` 的经理
测试用例
数据库升级指南
安装 18c（*参见* `18c 安装`）
管理选项
方法
`DBUA`
导出/导入实用程序
手动升级
风险与错误
网络选项
新特性指南
已废弃特性
已过时特性
升级后（*参见* 升级后）
先决条件检查
进度屏幕
原因
恢复选项
结果页面
目标选择
升级详细信息
升级指南
升级选项
验证
数据库升级助手（`DBUA`）
数据库验证（`dbv`）
数据库写入器（`DBWn`）
数据泄露
数据字典
基表（*参见* 基表）
缓存
数据库的文件
数据库的表空间
`DBA` 用户
默认配置文件设置
元数据
非默认参数
非 Oracle 用户
计划作业
`SQL` 语句
静态视图
表空间可用空间
`V$INSTANCE` 查询
数据生成器实用程序
数据卫士
架构
块损坏
故障转移操作
`MRP0`
重做传输
滚动升级
切换操作
零数据丢失
数据卫士守护者
数据库引擎
`DBA`
黑客
`PII`
数据卫士监视器（`DMON`）
数据保护
数据知识库
数据源名称（`DSN`）
`DBA_USERS` 视图
`DB_BLOCK_SIZE` 初始化参数
`DBMS_CRYPTO` 程序包
`DBMS_MONITOR` 程序包
专用服务器进程
`DEFAULT` 配置文件
诊断目标
`ASM`
`crs diag dest` 内容
`em`
`RAC`
`rdbms`
`show parameter` 命令
`tnslsnr`
诊断文件
诊断包，企业管理器
`ADDM` 运行图标
`AWR`
`CPU` 使用率
数据库性能活动
性能分析，`ADDM`
性能图表
问题解决
`SQL` 监控
`SQL` 语句统计选项卡，`SQL`
Statspack
目录结构
讨论论坛
博客
`MOSC`
Oracle 开发者社区
Oracle 文档
Oracle-L
Oracle 产品
`SQL` 语句
Web 浏览器
降级
动态性能视图
常见的 `V$` 视图
`GV$` 视图
Oracle 实例
会话数

### E

`EM` 云控制（`EM CC`）
企业管理器（`EM`）云控制
`CPU` 趋势线
`CPU` 利用率知识库查询
数据库 Express
数据库
`DB Express`
磁盘趋势线
磁盘利用率知识库查询
主页
超链接，`CPU` 利用率
内存指标
类别
选件
性能页面
资源利用率
`SQL` 屏幕
表空间利用率
以太网适配器
导出/导入升级
`EZ connect`
`ODBC`（*参见* 开放数据库连接（`ODBC`））
`SQL*Plus`
`TNS` 别名

### F

闪回恢复区
外键（`FK`）约束

### G

网格基础设施技术

### H

黑客技术
`HammerDB`
哈希分区
热备份
归档重做日志
缺失数据文件，`RMAN`
`RMAN` 配置
`rsync/robocopy`
热备份与冷备份
人力资源系统

### I

`IDENTITY` 列
独立 Oracle 用户组（`IOUG`）
每秒输入/输出操作数（`IOPS`）
18c 安装
创建主目录
`OUI`
`runInstaller` 实用程序
软件位置
安装与升级
交互式快速参考
后台进程
`CKPT` 详情
数据库架构
`DBA_TABLESPACES` 视图
`DBA` 视图选项卡
间隔分区
`IT` 安全

### J

Java 归档（`JAR`）文件
Java 数据库连接（`JDBC`）
Java 池
Java 虚拟机（`JVM`）
作业队列进程
初级数据库管理员

### K

知识库，`MOS`
书签文档
搜索框
搜索结果

### L

大池
`LDAP` 认证
监听器控制（`lsnrctl`）
监听器日志
监听器注册（`LREG`）
列表分区
`LiveSQL`
本地连接
遗赠协议
环境设置
`oraenv`
进程
`SID`
`LOCAL_LISTENER` 参数
锁监视器（`LMON`）
`LOG_CHECKPOINT_TIMEOUT` 值
逻辑备份
日志切换
日志写入器（`LGWR`）

### M

可管理性监视器（`MMON`）
可管理性轻量级监视器（`MMNL`）
托管恢复进程（`MRP`）
强制性进程
`CKPT`
`CLMN`
`CLnn`
`DBWn`
`LGWR`
`LREG`
`MMNL`
`MMON`
`PMAN`
`PMON`
`RECO`
`SMON`
手动 IP 地址
`MobaXterm`, `SSH`
`MongoDB` 数据库
`MOS` 认证
`EM` 结果
快速链接
搜索部分
`MOS` 仪表板
多平台工作，`DBA`
备忘单
数据库引擎
可雇佣技能
Oracle 关系数据库引擎
`SQL Server`
多租户
My Oracle Support 社区（`MOSC`）
My Oracle Support（`MOS`）
导航选项卡
相对时间
登录屏幕
培训视频

### N

`NAT` 网络适配器
网络附加存储（`NAS`）
网络配置助手（`NETCA`）
`NewsBlur`
非生产区域
NoSQL 数据库


### O

对象权限
在线重做日志（ORLs）
开放数据库连接（ODBC）数据源窗口驱动程序配置 DSN Oracle 驱动程序 Python
最佳灵活架构（OFA）
可选后台进程
`ora12_verify_function`
Oracle 客户端数据库软件至虚拟机安装指南文档按钮 Linux 链接 选择版本 共享文件夹 Yum 搜索结果
`ORACLE_BASE`，OFA
Oracle 数据库管理员 应用程序连接 归档重做日志 缓冲区缓存，读取数据块 控制文件 数据文件 错误消息 引擎 `vs.` Oracle 实例 ORLs 参数文件 密码文件 安全临时文件 Undo 表空间 Oracle 文档 数据库实用程序 错误消息 真正应用集群 RTFM（请参阅 仔细阅读手册（RTFM））
Oracle 企业管理器
Oracle 错误（`oerr`）
`$ORACLE_HOME` 目录 `ORACLE_HOME`，OFA
Oracle 实例 `MOUNT` 模式 `NOMOUNT` 模式 `RESTRICTED` 模式 启动阶段
Oracle-L
Oracle Linux 购物车下载页面 下载链接
Oracle 开发者网络
Oracle 单点登录账户 测试服务器
Oracle 监听器服务状态 停止与启动
Oracle 维护的用户
Oracle 托管文件（OMF）
Oracle 密码实用程序（`orapwd`）
Oracle 补丁（`OPatch`）
Oracle 真正应用集群 集群 网格基础设施 实例间通信 NAS 一对一关系 SAN TAF Oracle 自动存储管理器
Oracle SQL Developer
Oracle 支持分析师
Oracle 通用安装程序（`OUI`） 配置脚本 屏幕 数据库版本 安装位置 安装选项 安装进度 清单 步骤，创建 操作系统组 步骤 保存响应文件，按钮 安全更新 步骤 终端窗口
Oracle 版本 云客户 数据库后端 特性与功能 网格计算 IT 供应商 补丁集 RAC
`ora_complexity_check` 函数
订单录入模式

### P

并行查询从进程（`Pnnn`）
分区 Oracle 的分区剪裁
`ORDER_DETAILS` 表 方案表 分区 基于 Web 的商店
密码验证函数
补丁与更新，MOS
补丁概述
补丁集更新（`PSU`）
性能调优指南
个人可识别信息（`PII`）
`PGA_AGGREGATE_LIMIT` 参数
`PGA_AGGREGATE_TARGET` 参数
Ping 虚拟机
PL/SQL 包
升级后修复脚本 无效对象检查 监听器升级 Oratab 更新 版本检查
升级前信息工具 环境变量 修复脚本 日志文件
主键（`PK`）
进程管理器（`PMAN`）
进程监视器（`PMON`）
程序全局区（`PGA`） 分配 实例初始化参数 Oracle 实例 目标建议
`PSU` 安装 数据库实例 `datapatch` 实用程序 `DBA` 视图 主目录 `OPatch` `PATH` 环境变量

### Q

质量分析（`QA`）
队列监视器进程（`QMNn`）

### R

范围分区
仔细阅读手册（RTFM） 概念指南 数据库文档 文档导航 安装指南 Oracle 文档分类
真正应用集群（`RAC`）
真正应用测试（`RAT`）
恢复进程（`RECO`）
恢复 备份阶段 要求 DBA `RMAN` `RPO` `RTO` `SLA`
恢复管理器（`RMAN`） 用户管理的备份
重做日志缓冲区
版本更新修订（`RUR`） 补丁
还原阶段
修订版更新（`RU`）
落基山 Oracle 用户组（`RMOUG`）
角色的权限

### S

示例模式
范围蔓延
`SCOTT/TIGER`
搜索引擎
安全补丁更新（`SPU`）
安全配置文件 管理配置文件 DBA 用户 参数 查询限制
职责分离
服务器进程标识符（`SPID`）
服务请求（`SR`） 导航按钮 `ORA-600` 错误 严重性选项 建议注释
会话的跟踪文件
共享服务器进程
快照 机器工具图标 多个快照 升级前快照 还原操作 VirtualBox 管理器 虚拟化技术 虚拟机快照
社交媒体 Facebook LinkedIn Oracle 平台 Twitter YouTube
软件供应商
SQL 开发者 连接 DDL 选项卡 运行 SQL SQL 注入 SQL 语言参考
SQL*Plus 缓冲区 更改文本 执行行 删除 列表 保存 不正确命令
`SQL_TRACE` 静态字典视图 `ALL_` `DBA_` `DBA` 视图 `DICT` 非 DBA 用户 有用视图 `USER_`
存储区域网络（`SAN`）
Swingbench
系统管理员（`SysAdmin`）
系统变更号（`SCN`）
系统全局区（`SGA`） 缓冲区缓存 组件 库缓存 结果缓存 共享池
系统监视器（`SMON`）
系统权限

### T

TCP 网络协议
测试床 数据库管理员 QA 服务器 虚拟化与云技术 虚拟机
测试服务器创建 安装目标 ISO 文件 语言 许可协议 内存分配 内存大小 网络适配器设置 存储选项 VirtualBox 管理器
测试用例 删除表失败 数据库管理员 学习过程 Oracle 论坛 `FK` 父表 分区表 `PK` 特定任务
TNS 错误 12514 12541 12545
`TNSNAMES` 别名
`TNSPING` 实用程序
TNS Ping（`tnsping`）
跟踪文件 操作系统进程标识符 原始跟踪文件内容 SQL 跟踪 `TKPROF`
事务日志
透明应用故障切换（`TAF`）
透明数据加密（`TDE`）
透明网络底层（`TNS`）
调优包 建议 执行 整体发现 SQL 访问顾问 选项 建议 选项卡 摘要 选项卡 工作负载 SQL，调度 SQL 调优顾问 按钮 统计信息 发现
Twitter
类型参考

### U

统一审计 操作 `ORA_ACCOUNT_MGMT` `ORA_LOGON_FAILURES`
用户进程

### V

VirtualBox，下载
虚拟硬盘
虚拟化技术
虚拟机（`VMs`） `IP 地址` 测试服务器
虚拟专用数据库（`VPD`）

### W, X

全球 Oracle 社区

### Y, Z
YouTube
