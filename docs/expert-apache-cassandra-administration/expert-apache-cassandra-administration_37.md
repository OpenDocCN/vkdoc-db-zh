# Set trustore and truststore_password if require_client_auth is true
truststore: /cassandra/apache-cassandra-3.10/conf/server-truststore.jks
truststore_password: truststorePass
```



# 更高级的默认配置如下

```yaml
protocol: TLS
algorithm: SunX509
store_type: JKS
cipher_suites: [TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA]
```

此时，您必须重启集群。

在节点启动过程中生成的日志中，您将看到以下消息，表明客户端与数据库服务器之间现已启用 SSL 加密：

```
INFO  [main] 2017-09-05 14:17:10,681 Server.java:145 - Enabling encrypted CQL connections between client and server
```

集群恢复后，由于现已启用 SSL 验证，您必须指定 `--ssl` 选项才能连接到 Cassandra：

```
$ cqlsh --ssl
Validation is enabled; SSL transport factory requires a valid certfile to be specified. Please provide path to the certfile in [ssl] section as 'certfile' option in /home/cassandra/.cassandra/cqlshrc (or use [certfiles] section) or set SSL_CERTFILE environment variable.
$
```

您看到此错误的原因是因为您已为客户端通信启用了 SSL，因此 `cqlsh`（作为客户端）现在不信任 Cassandra 节点。为了让它信任该节点，它需要看到您之前生成的 CA 证书（`rootCa.crt`）。这确保了 Cassandra 使用的是由 CA 签名的证书。您可以通过在 `cqlshrc` 文件（`/home/cassandra/.cassandra/cqlshrc`）的 `[ssl]` 部分下列出 CA 证书来指向它，如下所示：

```ini
[connection]
hostname=192.168.159.130
port=9042
factory = cqlshlib.ssl.ssl_transport_factory
[ssl]
certfile = /cassandra/apache-cassandra-3.10/conf/rootCa.crt
validate = true
```

现在，您将能够通过 `cqlsh` 使用 SSL 连接到 Cassandra。

```
$ cqlsh –ssl
```

您现在通过 `cqlsh` 客户端与 Cassandra 节点建立了安全连接。

### JMX 认证与授权

JMX 认证和授权允许您控制用户对基于 JMX 的工具（如 `nodetool` 和 `JConsole`）的访问。您可以配置 JMX 连接使用 Cassandra 的内部认证和授权机制，就像 CQL 客户端一样。

如果您已通过本章前面描述的认证和授权机制在数据库中配置了用户名和密码，那么您也必须使用已配置的认证和授权信息来执行 JMX 工具。

在您能够使用带有认证功能的 `nodetool` 或 `JConsole` 之前，必须先启用 JMX 认证和授权。

### 启用 JMX 认证与授权

默认情况下，JMX 安全性未配置，这意味着您只能从 `localhost` 访问 JMX 工具（如 `nodetool` 和 `JConsole`）。您可以使用本地密码文件和访问文件来配置 JMX 认证和授权，以设置用户的凭据和访问权限。然而，从 Cassandra 3.6 开始，您也可以通过“搭乘” Cassandra 内部认证和授权的顺风车来配置 JMX 安全性。

在本节中，我将展示如何使用本地文件配置 JMX 认证和授权。

以下是使用本地文件配置 JMX 安全性的步骤。

1.  编辑 `cassandra-env.sh` 文件（位于 `$CASSANDRA_HOME/conf` 目录中），并在此处显示的代码块中更改两个设置：

```bash
    if [ "$LOCAL_JMX" = "yes" ]; then
    JVM_OPTS="$JVM_OPTS -Dcassandra.jmx.local.port=$JMX_PORT"
    JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.authenticate=false"
    else
```

在第一行，将 `"yes"` 改为 `"no"`。在第三行，将 `"false"` 改为 `"true"`。

2.  创建密码文件 `/etc/cassandra/jmxremote.password`（默认位置），并在文件中添加以下一行：

```
    cassandra  cassandra
```

我这里使用的是默认的超级用户账户，但这在生产系统中并不安全，您必须用一个或多个您希望其能够访问 JMX 兼容实用程序（如 `nodetool`）的用户凭据来替换这组凭据。

3.  创建访问文件（默认）`/etc/cassandra/jmxremote.access`，并在访问文件中添加以下信息：

```
    cassandra readwrite
    create javax.management.monitor.*,javax.management.timer.*
    \unregister
```

您授予角色 `cassandra` 的 `readwrite` 权限使此 JMX 客户端能够处理 MBeans。更具体地说，它允许此客户端设置属性、调用操作、接收通知等。

4.  重启数据库。

5.  尝试运行 `nodetool status` 命令。由于您配置了 JMX 授权和认证，您应该无法像往常一样运行此命令。如果您已正确配置所有内容，您应该会看到以下错误：

```
    $ nodetool status
    nodetool: Failed to connect to '127.0.0.1:7199' - SecurityException: 'Authentication failed! Credentials required'.
```

6.  通过提供角色 `cassandra` 的凭据来运行 `nodetool status` 命令。

```
    $ nodetool -u cassandra -pw cassandra status
    Datacenter: datacenter1
    =======================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --  Address          Load       Tokens       Owns (effective)  Host ID                               Rack
    UN  192.168.159.129  24.21 MiB  256          49.9% 0dbb9e0e-867e-4179-b6b6-631d38dd68f9  rack1
    UN  192.168.159.130  24.25 MiB  256          50.1% 001399d4-49fc-467c-b188-e93629a0f118
```

数据库现已配置好 JMX 认证和授权。与 `nodetool` 实用程序类似，当您使用 `JConsole` 远程连接到 Cassandra 集群时，现在也需要提供用户名和密码。

## 使用带有认证功能的 `cqlsh`

您可以配置认证，使得登录 `cqlsh` 需要密码。通过创建或修改 `cqlshrc` 文件来实现。以下是配置此认证的步骤。

1.  编辑 `cqlshrc` 文件，或者如果您没有该文件，请在以下位置创建一个：

```
    /etc/cassandra/cqlshrc.sample                 /* 包管理器安装
    install_location/conf/cqlshrc.sample          /* tarball 安装
```

2.  在 `cqlshrc` 文件中输入以下行：

```ini
    [authentication]
    username = sam
    password = !!bang!!$
```

3.  通过设置以下权限来保护文件安全：

```
    $ sudo chmod 400 /home/.cassandra/cqlshrc
```

## 总结

保护 Cassandra 的安全是一项多方面的工作。要全面保护您的数据，除了配置集群间和客户端-集群通信的认证、授权和 SSL 加密外，您可能还希望加密静态数据。

市面上有几种不错的第三方加密解决方案。通过使用其中一种或多种方案，您可以完成安全防护的闭环，降低安全漏洞发生的可能性。本章中我没有讨论这些产品，但通过互联网快速搜索“静态数据加密”应该就能看到可用的选择。

# 索引

## A

- 应计检测机制
- 管理工具
    - `cassandra-stress` 工具
    - `cassandra` 实用程序
    - `nodetool` 实用程序
    - SSTable 实用程序
- Akka
- 分配算法
- `ALTER KEYSPACE` 权限
- Amazon EC2
    - AMI
    - AWS
        - 配置 Cassandra 集群
            - 创建实例
            - 添加存储
            - AMI
            - 选择类型
            - 配置详细信息
            - 配置安全组
            - 密钥对
            - 检查启动
            - 标签
        - 安装 Cassandra
- Amazon 机器映像 (AMI)
- Amazon 的 DynamoDB
- Amazon Web Services (AWS)
- 反熵修复
    - 定义
    - 默克尔树
    - `nodetool repair` 命令
- `ANY` 一致性级别
- Apache Cassandra
    - 布隆过滤器
    - 从源码构建
    - `cassandra.yaml` 文件
    - 检查状态
    - 清除 cassandra 数据
    - 集群
    - 压实
    - 压缩
    - 定义
    - 分布式数据
    - 缺点
        - 不可变表与变更
        - 缺乏对事务的支持
        - 无索引
        - 无连接
    - 查询数据



# 技术词汇表与概念

## BASE 与一致性
- 最终一致性
- 基本可用、最终一致（BASE）系统
- 原子性、一致性、隔离性和持久性（ACID）
- 原子性
- 一致性
- 隔离性
- 持久性

## 性能与扩展性
- 高度可扩展
- 高性能数据库
- 大型环境
- 写操作密集型工作负载

## 基础设施与部署
### 集群
- 集群
- 节点
- 启动集群
- 集群健康检查
- 集群部署
- 集群维护任务
- `cluster_name` 参数
- 廉价服务器
- Linux 服务器
- 选择 CPU
- 选择存储
- 磁盘存储
- 网络考量
- NFS、SAN 和 NAS
- RAM
- 存储要求
- 可用磁盘容量

### 工具与服务
- Apache 配置
- Apache Kafka
- Apache Mesos
- Apache Spark
- `pyspark command`
- `spark-shell command`
- `spark-Cassandra-connector`
- Apache Sqoop
- Bundler
- JConsole
- JMX 客户端
- Nagios 服务器配置
- 安装与配置 NRPE
- PDSH
- `service` 命令
- 停止
- 验证版本

## Cassandra 数据库
### 核心概念
- Cassandra 数据库
- Cassandra，节点
- Cassandra 特定插件
- Cassandra 访问控制矩阵
- Cassandra 授权器
- Cassandra 集群管理器 (CCM)
- Cassandra 查询语言 (CQL)
- Cassandra 数据建模
- 对比关系数据库管理系统 (RDBMS)

### 安装与配置
- 先决条件
- Java
- Python
- 安装
- 配置
- `auto_bootstrap` 属性
- `cassandra.yaml` 文件
- Apache 配置
- 安装与配置 NRPE
- Nagios 服务器配置
- Bin 目录
- 缓存目录
- 提交日志
- 清除 Cassandra 数据

### 数据模型与结构
- 列族数据库
- 列式数据库
- 物化视图
- 主键
- 分区键
- 复合键与聚类列
- 复合分区键
- 聚类键/列
- 静态列
- 表创建
- 表选项
- 压缩表
- 聚类顺序
- 布隆过滤器
- B 树
- 基数
- 高基数与低基数列
- 元组
- 用户定义类型 (UDT)
- 集合数据类型 (list, map, set)
- 冻结值
- TTL 值
- 时间戳
- 墓碑
- 僵尸记录与节点修复

### CQL 操作与语法
- 创建键空间
- 修改键空间
- 删除键空间
- 修复键空间
- 创建表
- 修改表
- 删除表
- 截断表
- `describe` 命令
- `column_definition`
- 表创建
- 条件语句
- `INSERT` 语句
- `UPDATE` 语句
- `SELECT` 语句
- 过滤
- `WHERE` 子句
- `ORDER BY` 子句
- `GROUP BY` 子句
- `LIMIT N` 选项
- JSON 格式
- `IN` 关键字
- 内置函数
- 函数、聚合与用户类型
- `copy` 命令
- `COPY` 命令
- 捕获命令
- `expand` 命令
- `tracing` 命令
- 启动
- 获取帮助
- 创建索引
- 删除索引
- 主索引
- 二级索引
- SASI
- 计数器列以跟踪值
- 删除行和列，删除表
- 插入测试数据
- 查询季度表
- 多数据中心
- 持久写入属性
- 复制因子
- 副本，数据中心
- 关系数据库系统
- 限定名
- 集合 set，list/map
- 多个/电子邮件地址

### SSTable 与存储
- SSTable
- SSTables
- 批量数据
- 导入
- 加载外部数据
- `sstableloader`
- 备份数据
- 自动快照
- 增量
- 快照

### 运维与管理
- `nodetool info` 命令
- `nodetool status` 命令
- `nodetool repair` 命令
- `system.size_estimates` 表
- 刷新与清空数据
- 处理数据损坏
- 重建索引
- 修复数据
- 线程池统计
- 清屏 (CLS)
- 变更数据捕获 (CDC) 日志

### 压测与性能
- Cassandra-stress 工具
- 命令
- 复制、压缩与压缩选项
- 运行
- 混合工作负载
- 多节点
- 读取测试
- 写入测试
- 基于 YAML 的配置文件

## 大数据生态
- 大数据
- Apache Spark
- Apache Kafka
- Apache Mesos
- Apache Sqoop
- 原子操作
- 弹性分布式数据集 (RDD)
- `pyspark` 命令
- `spark-shell` 命令
- `spark-Cassandra-connector`

## 开发与测试
### 行为驱动开发 (BDD)
- 行为驱动开发 (BDD)
- Gherkin 语法
- Coshxlabs 代码
- 安装 Cucumber
- 安装 Docker
- 安装 Docker-compose
- 使用 Cucumber 运行

## 其他术语
- 用户与组
- `service` 命令
- 停止
- 表行
- 验证版本
- 写入
- 批处理方法
- 批处理日志
- 批处理操作，单分区和多分区
- `INSERT` 语句
- 周期性
- 要求
- 认证
- 配置



### 压缩

## 压缩策略

### 启用与禁用
### 全局压缩参数，配置
### `compaction_throughput__mb_per_sec` 参数
### `concurrent_compactors` 属性
### `snapshot_before_compaction` 属性
### `sstable_preemptive_open_interval_in_mb` 属性

## LCS

### 日志记录
### 设置

## STCS

### 测试

## TWCS

# 比较并设置 (CAS)

# 复合主键

# 概念建模

# 并发标记清除 (CMS)

### `conf` 目录

# 一致性、可用性与分区容忍性 (CAP) 定理

## BASE 系统

## 一致性

### 原则
### 可用性
### 一致性
### 分区容忍性

# 容器

# 协调器

# 计数器缓存

# 计数器数据类型

# Cucumber

# D

### 数据缓存

# 数据中心

# 数据中心相关维护任务

# 数据损坏

## 检查
## 修复

# 数据文件目录

# 数据操作语言 (DML)

# 数据建模

## 集群节点
## 数据驱动 vs. 查询驱动
## 数据
## 易用性
## 分区
## 物理
## 职业自行车赛统计
## 查询
### 读取限制
## 可靠性
## 可扩展性
## 排序顺序
## 结构化流程
### 写入限制

# DataStax

## 课程
## 开发工具

# DSE

# DataStax 企业版 (DSE)

# DataStax, Inc.

# Debian 包

# 去中心化数据库

# 数据中心退役

# 磁盘存储

# 分布式数据库系统

# Docker

## `broadcast_address`
## Cassandra 集群
## 命令行实用程序
## 容器
## `dc` 属性
## `endpoint_snitch`
## 环境变量
## 主机服务器
## 镜像
## `listen_address`
## `num_tokens`
## `rack` 属性
## 运行 `cqlsh`
## `systemctl status` 命令
## Ubuntu 16.04 服务器安装
## 使用卷

### 文档数据库

# 动态环参与

# E

# 端点范围 vs. 子范围
## 修复

# 最终一致性

## 反熵
## 一致性级别
## 协调
### 修复数据
## 要求

# F

# Facebook

# 故障检测机制

# 容错能力

# 防火墙

## 端口访问
## 端口配置

# 灵活的数据模型

# 全量 vs. 增量修复

# G

# 垃圾回收

# Gossip 管理

# Gossip 协议

## 累积检测机制
## `cluster_name`
## 故障检测机制
## `listen_address`
## 过程
## 种子节点与
## `seed_provider`
## `storage_port`

# 图数据库，节点

# H

### 处理一致性

# 交接过程

# 哈希环

# 提示移交

## 组成
## 针对数据中心
## 定义
## 启用集群
## `max_hint_window_in_ms` 属性
## `sethintedhandoffthrottlekb` 属性
## `statushandoff` 命令
## 存储于目录
## `truncatehints` 命令
## `write_request_timeout_in_ms` 属性

# I

# 增量修复

# 索引文件

# J

### Javadoc 目录

# Java 垃圾回收 (GC)

# Java 堆大小

# Java 大页面设置

# Java 管理扩展 (JMX)

# Java 虚拟机 (JVM)

### JConsole

## 连接登录页面
## `jmxsh`
## 概览选项卡

### JMX 认证与授权

# K

# Kafka, Apache

# 键缓存

# 关键的 `nodetool` 维护命令

## 节点退役
## `cassandra.override_decommission=true` 选项
## 命令
## 数据移除与重启
## `nodetool assassinate`

# 键空间

### 键值数据库

# L

# `LeveledCompactionStrategy` (LCS)

### `lib` 目录

### 轻量级事务

## 注意事项
## 插入带有身份证号的骑行者
## 操作集

# 可线性化一致性

# Linux 服务器

## 禁用交换空间
## 禁用 `zone_reclaim_mode`
## Java 堆大小
## Java 大页面设置
## Java 版本
## 与内核设置
## NUMA 系统
## PAM 安全设置
## 设置 Shell 限制
## 同步时钟并启用 NTP
## TCP 设置
## 用户资源限制

# `listen_address` 属性

# `listen_address` 参数

# `listen_interface` 参数

# Logback

### 日志记录

## 配置
## 位置设置
## `logback` 配置

# `logback` 日志框架

## Appender
## 优点
## 布局

# `logback.xml` 文件

# Logger 类

# 设置日志轮转

# `nodetool setlogginglevel` 命令

### 逻辑建模

# 日志结构合并树 (LSM 树)

### 日志结构存储引擎

# M

# 中型数据

# Memtable

## 定义
## 阈值

# `memtable_cleanup_threshold` 参数

# `memtable_flush_writers` 参数

# Merkle 树

# Mesos, Apache

# 最小配置属性

# MongoDB

# 监控 Cassandra

## LAMP 栈安装

# Nagios

## 配置文件
## 安装
## 插件安装
## NRPE 安装

# 多节点 Cassandra 集群

## `auto_bootstrap` 属性
## `broadcast_rpc_address` 属性
## 更改节点 IP 地址
## 客户端端口
## 配置
## 数据中心配置
## `endpoint_snitch` 选项
## 防火墙端口访问配置
## 初始化包含多个数据中心的集群
## 节点间端口
## IP 地址
## 键空间
## `listen_address` 参数
## 节点宕机
## `num_tokens` 属性
## 端口
## 机架名称
## `rpc_address` 属性
## 种子节点
## `seeds` 属性



# A

# B

# C

# D

# E

# F

# G

# H

# I

# J

# K

# L

# M

# N

选择用于`数据中心`的`name`

`启动`进程

`停止`

`版本`不匹配

`Murmur3`分区策略

`N`

`Nagios`

`构建`依赖项

`Cassandra`集群主机，监控

`配置`

`安装`

`NRPE`

`插件`

`用户`和`组`

`Nagios 远程插件执行器 (NRPE)`

`网络附加存储 (NAS)`

`网络信息`

`网络接口卡 (NIC)`

`网络时间协议 (NTP)`

`NetworkTopologyStrategy`

`节点`管理

`添加`，`数据中心`

`集群`加入

`宕机`节点替换

`下线` `数据中心`

`移动`

`移除`节点

`运行中`节点替换

`节点`修复

`定义`

`暗示`交接

`参见 Hinted`交接

`节点`重启方法

`Nodetool drain 命令`

`nodetool proxyhistograms 命令`

`nodetool tablehistograms 命令`

`nodetool upgradetsstables 命令`

`Nodetool` 实用程序

`nodetool info 命令`

`nodetool status 命令`

`规范化理论`

`NoSQL 数据库`

`列`族

`文档`

`图`

`键值`

`num_tokens` 属性

# O

`ONE` 一致性级别

`开源`数据库

`OpsCenter`

`最佳`存储

`乐观`复制

`Oracle JDK`

# P

`PAM 安全设置`

`并行分布式 Shell (PDSH)`

`并行`修复

`分区器函数`

`分区器范围`修复

`分区器`

`Murmur3Partioner`

及分区策略

`RandomPartitioner`

`分区键缓存`

`Paxos` 协议

`对等网络`架构

`对等网络`系统

`性能`，`Cassandra`

`压缩`数据

`ALTER TABLE` 语句

`配置`

`有效性`测试

`修改`，压缩算法

`关闭`

`数据`缓存

`Cassandra 存储`

`配置`

`计数器`缓存

`全局缓存参数`

`监控`

`追踪`数据库操作

`类型`

`JVM` 和垃圾回收策略

`压力测试 cassandra`

`参见 Cassandra-stress 工具`

`追踪`以分析性能

`管理追踪`

`读请求`

`写请求`

`调优布隆过滤器`

`Phi 崩溃故障检测`

`物理数据建模`

`概率性追踪策略`

`Python`

# Q

`查询驱动的数据建模`

`查询数据`

`参见 Cassandra 查询语言 (CQL)`，`SELECT` 语句

`Quorum`

`计算`

`数据中心集群`

`EACH_QUORUM`

`LOCAL_QUORUM`

`读一致性级别`

`副本因子`

`写一致性级别`

`Quorum 读写`

# R

`机架`

`随机选择算法`

`快速读保护`

`一致性级别`

`speculative_retry` 属性

`支持`

`读一致性级别`

`ALL`

`包含两个`数据中心`的集群`

`请求`

`单数据中心`

`读取数据`

`协调器`

`过滤器`命令

`Gossip` 协议`

`请求`

`写数据`影响

`读修复`

`定义`

`read_repair_chance` 属性

`读请求`

`直接`

`修复请求`

`副本节点`

`参照完整性`

`关系数据库管理系统 (RDBMS)`

`关系型数据库`

`数据局部性`

`更低成本`

`无故障转移`

`对等网络`架构

`RDMS` 和大数据`

`可靠性`

`分片`

`第三范式`

`修复数据`

`副本放置策略`

`复制策略`

`定义`

`组`

`NetworkTopologyStrategy`

`SimpleStrategy`

`切换键空间`

`弹性分布式数据集 (RDD)`

`恢复数据`

`提交日志`

`手动归档`

`时间点恢复`

`恢复`

`循环键空间`

`节点重启方法`

`运行修复`

`设置位置`

`设置时间戳`

`从快照恢复`

`使用 sstableloader`

`基于角色的访问控制`

`cycling_admin`

`授予权限`

`登录账户`

`对象权限`

`permissions` 命令

`查看`，已授予的权限

`行缓存`

# S

`SAN`

`二级索引`

`安全性`

`配置认证`

`防火墙端口配置`

`JMX 认证和授权`

`角色创建`

`管理员权限`

`AllowAuthenticator`

`分配权限`，`登录账户`

`配置认证`

`日志记录`

`更改密码`

`属性`

`超级用户账户`

`SSL 加密`

`参见 SSL 加密`

`种子节点`

`二级索引`

`串行 vs . 并行修复`

`串行一致性设置`

`分片`

`简单构建工具 (SBT)`

`单令牌架构`

`SizeTieredCompactionStrategy`

`SMACK` 技术栈`

`快照`

`压缩数据前`

`复制数据`

`列出节点`

`恢复数据`

`运行修复`

`使用 sstableloader`

`模式`

`Snitch`

`cassandra-rackdc.properties`

`cassandra-topology.properties`

`CloudstackSnitch`

`默认为动态`

`Ec2MultiRegionSnitch`

`Ec2Snitch`

`GoogleCloudSnitch`

`GossipingPropertyFileSnitch`

`PropertyFileSnitch`

`RackInferringSnitch`

`SimpleSnitch`

`Snitch` 的作用

`固态硬盘 (SSD)`

`排序字符串表 (SSTable)`

`推测性重试`

`Sqoop`，Apache

`SSH` 工具

`SSL 加密`

`客户端加密`，启用

`节点间加密`，启用

`Java 加密扩展文件`安装

`服务器证书`

`CA 到密钥库`

`证书颁发机构`，创建

`集群配置`，`密钥库`

`创建`，`节点`

`服务器信任库`

`已签名证书`，`密钥库`

`签名`，`CA 的公钥`

`签名请求`

`SSTable 附带二级索引 (SASI)`

`SSTable`

`缓存数据`

`压缩操作`

`数据文件`

`数据结构`

`用于持久化存储`

`存储数据`

`四节点`

`哈希值`

`严格一致性`

`子范围修复`

`切换 Snitch`

`符号链接`

# T

`表统计信息`

`TCP 设置`

`测试驱动开发 (TDD)`

`第三范式`

`生存时间 (TTL)`

`TimeWindowCompactionStrategy (TWCS)`

`令牌`

`墓碑`

`工具目录`

`追踪数据`

`可调一致性`

# U

`Ubuntu 16.04 服务器`

`通用唯一标识符 (UUID)`

`用户定义聚合 (UDA)`

`用户定义函数 (UDF)`

`用户定义类型 (UDT)`

# V

`Vagrant` 工具

`虚拟机 (VM)`

`虚拟节点 (vnode)`

`禁用`

`num_tokens` 参数

`重新平衡数据`

`带有`的环

`令牌`

# W, X

`写放大 (WA)`

`写一致性`

`ALL` 一致性级别

`ANY` 一致性级别

`默认`

`暗示交接`

`LOCAL_ONE` 一致性级别

`ONE` 一致性级别

`Quorum` 相关级别

`串行一致性设置`

`TWO 和 THREE` 一致性级别

`写入数据`

`布隆过滤器`

`提交日志`

`二进制文件`

`以保护变更`

`空间阈值`

`索引文件`

`内部操作`

`内存表`

`配置清理阈值`

`配置刷新数据`

`数据库刷新`

`持久性`

`刷新到磁盘`

`nodetool drain` 命令`

`nodetool flush` 命令`

`请求流程`

`暗示的作用`

`SSTable`

`参见 SSTable`

# Y

`基于 YAML` 的配置文件

# Z

`僵尸节点`
