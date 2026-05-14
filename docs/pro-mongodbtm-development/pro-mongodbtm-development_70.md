# 第 12 章 在 Oracle Data Integrator 中将 MongoDB 与 Oracle Database 集成

![image](img/image00404.jpeg)

MongoDB 是一种 BSON（二进制 JSON）格式的 NoSQL 数据库，也是领先的 NoSQL 数据库。Oracle Database 是领先的关系数据库。用例可能是将 MongoDB 数据加载到 Oracle Database 中。Oracle Loader for Hadoop 不直接支持 MongoDB BSON 格式。Oracle Loader for Hadoop 支持从 Hive 加载。可以使用 Hive Storage Handler for MongoDB 在 MongoDB 数据存储上定义一个 Hive 外部表。Oracle Data Integrator 通过 IKM File-Hive to Oracle 知识模块，自动化了 Oracle Loader for Hadoop 的使用。在本章中，我们将在 Oracle Data Integrator 中将 MongoDB 数据加载到 Oracle Database。本章是第 11 章的延续。

本章涵盖以下主题：
*   设置环境
*   创建物理架构
*   创建逻辑架构
*   创建数据模型
*   创建集成项目
*   创建集成接口
*   运行接口
*   在 Oracle Database 表中选择集成的数据

