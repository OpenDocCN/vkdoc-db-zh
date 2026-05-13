# CockroachDB 权威指南

## 目录
*   目录
*   关于作者
*   关于技术评审
*   致谢
*   前言

## 第 1 章：CockroachDB 的诞生原因
*   什么是 CockroachDB？
*   CockroachDB 的架构
*   CockroachDB 解决什么问题？
*   CockroachDB 适用于谁？

## 第 2 章：安装 CockroachDB
### 许可
*   免费版
*   付费版
*   CockroachDB 核心版
### 本地安装
*   二进制文件安装
*   Docker 安装
*   Kubernetes 安装
    *   1_pod-disruption-budget.yaml
    *   2_stateful-set.yaml
    *   3_private-service.yaml
    *   4_public-service.yaml
    *   5_init.yaml
*   多节点集群
*   多区域集群
    *   多区域部署
    *   区域故障目标
    *   地区故障目标
*   demo 命令
### CockroachDB Serverless/Dedicated
*   创建集群
*   连接到你的集群
### 小结

## 第 3 章：概念
### 数据库对象
### 数据类型
*   UUID
*   ARRAY
*   BIT
*   BOOL
*   BYTES
*   DATE
*   ENUM
*   DECIMAL
*   FLOAT
*   INET
*   INTERVAL
*   JSONB
*   SERIAL
*   STRING
*   TIME/TIMETZ
*   TIMESTAMP/TIMESTAMPTZ
*   GEOMETRY
### 函数
### 地理分区数据
*   REGION BY ROW
*   REGION BY TABLE

## 第 4 章：通过命令行管理 CockroachDB
### Cockroach 二进制文件
### start 和 start-single-node 命令
### demo 命令
### cert 命令
### sql 命令
### node 命令
### import 命令
### sqlfmt 命令
### workload 命令

## 第 5 章：与 CockroachDB 交互
### 连接到 CockroachDB
*   使用工具连接
*   通过编程方式连接
    *   Go 示例
    *   Python 示例
    *   Ruby 示例
    *   Crystal 示例
    *   C# 示例
### 设计数据库
*   数据库设计
*   模式设计
*   表设计
    *   命名
    *   列数据类型
    *   索引
*   视图设计
    *   简化查询
    *   限制表数据
### 移动数据
*   导出和导入数据
*   监控数据库变更
    *   Kafka 汇
    *   Webhook 汇

## 第 6 章：数据隐私
### 全球法规
### 特定地点的考量
*   拥有英国和欧洲客户的英国公司
*   拥有欧洲和美国客户的欧洲公司
*   拥有美国客户的美国公司
*   拥有美国和欧洲客户的美国公司
*   拥有中国客户的非中国公司
### 个人身份信息
### 加密
*   传输中加密
*   静态加密
    *   数据库加密
    *   表加密

## 第 7 章：部署拓扑
### 单区域拓扑
*   开发环境
*   基础生产环境
### 多区域拓扑
*   区域表
*   全局表
*   跟随者读取
*   跟随工作负载
### 反模式
*   小结

## 第 8 章：测试
### 结构测试
### 功能测试
*   黑盒测试
    *   创建一个简单应用
    *   测试应用
    *   使用应用代码进行测试
*   白盒测试
    *   引用完整性检查
    *   索引和查询计划
### 非功能测试
*   性能测试
    *   扩展测试环境
*   弹性测试

## 第 9 章：生产环境
### 最佳实践
*   SELECT 性能
*   INSERT 性能
*   UPDATE 性能
### 集群维护
### 移动集群
### 备份和恢复数据
*   全量备份
*   增量备份
*   加密备份
*   位置感知备份
*   计划备份
### 集群设计
*   集群规模调整
*   节点规模调整
### 监控

## 索引
*   索引
