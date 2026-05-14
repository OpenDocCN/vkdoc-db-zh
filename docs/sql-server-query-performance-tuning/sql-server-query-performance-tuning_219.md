# 索引

## 死锁

*   回放机制，506
*   数据定义语言 (DDL)，273
*   查询捕获机制，506
*   数据操作语言 (DML)，273
*   可重复的过程，506
*   数据检索机制，194
*   服务器端跟踪，507
*   `DBCC SHOW_STATISTICS` 命令，210, 215, 230
*   `@DateTime`，510
*   `DBCC SQLPERF()` 函数，394
*   分布式重放，508
*   `DBCC TRACEON` 语句，447
*   事件和列，507–509
*   分析器，508
*   分析
    *   SQL Server 2005–2014，509
*   所有者模式，449
*   标准性能测试，510
*   分析器工具，449
*   `TSQL` 文件，508, 510
*   `Purchasing.PurchaseOrderHeader` 表，454
*   SQL 分析器，505
*   `sqlhandle`，450
*   SQL server 2012，505
*   `system_health` 会话，454
*   `DATABASEPROPERTYEX` 函数，202
*   跟踪标志 1222，454

## 数据库工作负载优化

*   `PurchaseOrderDetail`，450
*   `AdventureWorks2012` 数据库，517
*   `xml_deadlock_report`，448
*   `ALTER EVENT SESSION` 命令，520
*   XML 文件，449–450
*   XML 图形数据，451
*   笛卡尔连接，545
*   规避方法
*   开销最大查询的识别
*   隔离级别，457
*   详细的资源使用情况，524
*   `NOLOCK`/`READUNCOMMITTED` 锁定提示，457
*   整体资源使用情况，524
*   非聚集索引到聚集索引，456
*   SQL 工作负载，522
*   资源访问，物理顺序，455
*   `SSMS`/查询技术，522
*   行版本控制，457
*   性能最差的查询，522–523
*   `SELECT` 语句，456

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 死锁 (续)

*   `DROP_EXISTING` 子句，252
*   `DBCC TRACEON` 语句，447
*   动态游标，466, 471
*   致命拥抱，443–444
*   动态管理对象 (DMO)，19
*   错误处理，445
*   `Sys.dm_db_xtp_table_memory_stats`，30
*   `lock_deadlock_chain`，446
*   `Sys.dm_os_memory_brokers`，29
*   锁监视器，444
*   `Sys.dm_os_memory_clerks`，29
*   并行操作，444
*   `Sys.dm_os_ring_buffers`，29
*   SQL Server 启动，447–448
*   `Sys.dm_xtp_system_memory_consumers`，30
*   `system_health`，446
*   动态管理视图 (DMV)，277
*   跟踪标志 1204，447
*   跟踪标志 1222，447
*   牺牲品，444
*   `xml_deadlock_report`，446
*   实体完整性约束，549–550



声明式引用完整性 (DRI)
执行计划缓存
碎片整理
即席工作负载

ALTER INDEX REBUILD 语句
定义
特性
强制参数化
DROP_EXISTING 子句
优化
HumanResources.Employee 表
计划可重用性
性能调优过程
准备好的工作负载
Purchasing.PurchaseOrderDetail
简单参数化
Purchasing.PurchaseOrderHeader 表
计划重用
sys.dm_db_index_physicalstats
准备好的工作负载，计划可重用性

磁盘性能分析
查询计划哈希和查询哈希
对齐
创建查询
平均磁盘读取秒数和平均磁盘写入秒数
数据分布和索引
当前磁盘队列长度
截然不同的计划
磁盘字节/秒计数器
SELECT 条件
% 磁盘时间计数器
建议
磁盘传输/秒监视器
避免即席查询
更快的 I/O 路径
避免隐式解析
文件组配置
显式参数化可变部分
I/O 监控工具
日志文件
参数化可变部分
新的磁盘子系统
准备/执行模型
优化应用程序工作负载
sp_executesql 编码
物理磁盘和逻辑磁盘计数器
存储过程创建
RAID 阵列
配置

执行计划生成
老化
绑定
DDL
DML
错误语句
查询处理器树
基于语法的优化
警告指示器
基于成本的优化
解析器
查询优化

域完整性


