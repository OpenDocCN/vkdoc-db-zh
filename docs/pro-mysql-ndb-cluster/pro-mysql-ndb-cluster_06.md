# 第五部分：开发与性能调优

## 第 18 章：使用 SQL 开发 MySQL NDB 集群应用程序
- 设计表
- 创建 NDB 集群表
- 支持的数据类型
- 三种索引类型
- 定义索引
- T 树索引
- 估算表大小
- 估算每个表所需的对象数
- 定义外键约束
- 审查表定义
- 磁盘数据表
- 关于规范化的考量
- 关于表设计的主要限制
- 通过 SQL 访问数据
- 连接到 SQL 节点
- `NDBCluster` 表的事务处理
- 错误处理技术
- 总结

## 第 19 章：作为 NoSQL 数据库的 MySQL NDB 集群
- 为何选择 NoSQL？
- 通过 `memcached` 访问数据
- 为何使用 `NDB-memcached`
- 设置 `NDB-memcached`
- 定义到 NDB 集群表的映射
- 通过 NDB API 访问数据
- 为何使用 NDB API？
- 为 NDB API 安装头文件与库
- 使用 NDB API 构建应用程序
- 参考与示例
- 典型的程序流程
- 简单读取示例
- 使用 `NdbRecord` 访问数据
- 扫描示例
- 错误处理考量
- 通过 `ClusterJ` 访问数据
- 安装 `ClusterJ`
- 编写 `ClusterJ` 应用程序
- `ClusterJ` 示例
- 总结

## 第 20 章：MySQL NDB 集群与应用程序性能调优
- `MySQL NDB Cluster` 调优
- 禁用节能与 CPU 频率调节
- CPU 绑定策略
- 磁盘类型与文件系统块大小
- SQL 调优
- 提交规模
- 非事务性批处理
- 引擎条件下推优化
- 优化连接
- 优化分区
- 优化从 SQL 节点到数据节点的访问
- 添加节点
- 将 NoSQL API 与 SQL 结合使用
- 总结

## 索引

## 内容概览

## 关于作者

## 关于技术审稿人

## 致谢

