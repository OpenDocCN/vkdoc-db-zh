# 第二部分：技术细节

## 第八章：分布式数据库模式

分布式关系型数据库
复制
无共享与共享磁盘
非关系型分布式数据库
MongoDB 的分片与复制
分片
分片机制
集群平衡
复制
写关注与读偏好
`HBase`
表、区域与区域服务器
缓存与数据局部性
行键排序
区域服务器的分裂、平衡与故障
区域副本
`Cassandra`
`Gossip` 协议
一致性哈希
副本
`Snitches`
总结

## 第九章：一致性模型

一致性类型
ACID 与 MVCC
全局事务序列号
两阶段提交
其他一致性级别
MongoDB 中的一致性
MongoDB 锁机制
副本集与最终一致性
HBase 一致性
最终一致的区域副本
Cassandra 一致性
复制因子
写一致性
读一致性
一致性级别之间的交互
提示移交与读修复
时间戳与粒度
向量时钟
轻量级事务
结论

## 第十章：数据模型与存储

数据模型
关系数据模型回顾
键值存储
BigTable 与 HBase 中的数据模型
`Cassandra`
JSON 数据模型
存储
典型的关系型存储模型
日志结构合并树
二级索引
结论

## 第十一章：语言与编程接口

`SQL`
NoSQL API
`Riak`
`Hbase`
`MongoDB`
Cassandra 查询语言 (CQL)
MapReduce
`Pig`
有向无环图
`Cascading`
`Spark`
SQL 的回归
`Hive`
`Impala`
`Spark SQL`
`Couchbase N1QL`
`Apache Drill`
其他 SQL on NoSQL 方案
结论
注释

## 第十二章：未来的数据库

革命再审视
反革命者
我们是否绕回了原点？
选择的烦恼
能否兼得所有？
一致性模型
模式
数据库语言
存储
融合数据库的愿景
与此同时，在 Oracle 总部
Oracle JSON 支持
通过 Oracle REST 访问 JSON
通过 REST 访问 Oracle 表
Oracle 图数据库
Oracle 分片
Oracle 作为混合数据库
其他融合型数据库
颠覆性数据库技术
存储技术
区块链
量子计算
结论
注释

