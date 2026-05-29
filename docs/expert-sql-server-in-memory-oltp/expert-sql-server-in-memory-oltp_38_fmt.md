# 索引

## A

管理与监控任务`extended events`另请参阅（`Extended events` `memory-optimized tables` `Memory Usage by Memory Optimized Objects`报告监控`In-Memory OLTP`事务监控内存使用情况`resource governor`另请参阅（`Resource Governor`，限制可用于`In Memory OLTP`的内存）内部和默认资源池恢复过程`sys.dm_db_xtp_table_memory_stats``SQL Server Database Engine`的体系结构`Atomic blocks`

## B

`BEGIN ATOMIC`另请参阅`Atomic blocks``Buffer Manager``Buffer pool``Bw-tree`

## C

`Catalog views`另请参阅`Data management views``Checkpoint file`关闭线程控制器线程I/O要求段日志记录段序列化器线程计时器任务`Checkpoint file pairs (CFPs)`另请参阅`Checkpoint file``ACTIVE`检查点文件数据文件增量文件强制执行`CHECKPOINT`大型数据文件生命周期数据库创建日志备份与垃圾回收合并过程`memory-optimized table`根文件状态`ACTIVE``MERGE TARGET``PRECREATED``UNDER CONSTRUCTION``WAITING FOR LOG TRUNCATION``CHECKSUM`函数`Clustered columnstore indexes`另请参阅`Columnstore indexes`内部对象行组`Clustered index`在基于磁盘的表上

## D

`Data partitioning``Data row``BeginTs`时间戳`EndTs`时间戳`IdxLinkCount`索引指针数组有效负载行头`StmtId`, 34, 结构`Data storage`另请参阅`Checkpoint files``CFP``CHECKPOINT`进程`MERGED SOURCE``CFP`状态`MERGE TARGET`状态`UNDER CONSTRUCTION`状态基于磁盘的表`Data warehouse workload`另请参阅`Columnstore indexes``Deployment and management`管理与监控参见`Administration and monitoring tasks`估算内存需求硬件组件`CPU``I/O subsystem`内存`Design considerations`二进制排序成本/收益分析数据分区索引策略另请参阅（`Index design considerations`）可维护性和管理开销引用完整性具有混合工作负载的系统不支持的数据类型

## E

`Extended events`

## F

`FILESTREAM`

## G

`Garbage collection``BeginTs`和`EndTs`时间戳数据管理视图`DELETE`操作删除/更改表被遗忘的角落扫描代次幽灵行目标空闲工作线程`idxLinkCount`索引页元素`memory-optimized table`表创建内存统计表删除无阻塞过期行摘要统计信息`UPDATE`操作工作流程工作项工作队列

## H

`Halloween effect``HASHBYTES`函数`Hash indexes``bucket_count`合适的数量`sys.dm_db_xtp_hash_index_stats`选择`bucket_count`冲突映射与非聚集索引数据选择执行时间点查找性能`SARGability`规则`Hashing`冲突哈希函数哈希映射哈希表`High Availability Technologies`

## I, J, K

`IBM DB2``In-memory database (IMDB)``IBM DB2``Oracle``SAP HANA``In-memory OLTP migration`另请参阅`Migration tools`二进制排序性能`clustered columnstore indexes`数据管理视图事务管理数据分区数据移动执行计划对象创建订单录入系统查询数据库级限制设计目标设计考虑基于磁盘的表引擎架构目标`IMDB`参见`In-memory database (IMDB)`导入批量行客户端代码`memory-optimized table type`表, `TVP`和存储过程索引注意事项限制`memory-optimized tables`扫描性能语句级重新编译存储过程，执行时间变量和基数估计与基于磁盘的临时对象迁移混合工作负载数据分区本机编译模块性能计数器查询互操作引擎会话/对象状态存储专用存储/缓存`ObjStoreDataAccess`类`ObjStoreService`类`ObjStoreUtils`类复制内容可扩展性问题会话存储实现不支持的数据类型乐观并发

## L

`Latches`在基于磁盘的表上等待统计信息`LEGACY_CARDINALITY_ESTIMATION`数据库范围的配置`Log buffer`

## M

`Management data warehouse`内存和本机编译顾问菜单配置过程级统计信息表争用分析报告表级统计信息使用情况分析报告`Memory consumers`另请参阅`Off-row storage``Memory-optimized table``BeginTs`时间戳数据行`halloween effect`结构`DURABILITY``SCHEMA_AND_DATA``SCHEMA_ONLY``EndTs`时间戳限制数据库级数据类型表`SQL Server 2016`统计信息嵌套循环联接算法变量支持的数据类型支持的表功能`Memory pointers`管理数据修改和并发`Microsoft Azure SQL Databases``Migration tools``In-Memory OLTP Migration Checklists Wizard``Memory Optimization Advisor`本机编译顾问`Stored Procedure Analysis`报告`Tables Analysis`报告`Transaction Performance Analysis Overview`报告

## N

`Native compilation``atomic blocks`另请参阅（`Atomic blocks`）内联表值函数互操作模式性能比较`InsertCustomers`与解释型函数函数创建函数内的循环多次调用`memory-optimized table types``memory-optimized table variables`本机编译存储过程概述性能比较安全上下文存储过程参见`T-SQL stored procedure`支持的T-SQL功能控制流函数运算符查询表面区域触发器和用户定义函数`Nested loop join algorithm``Nonclustered index`在基于磁盘的表上`Nonclustered indexes``Bw-Tree`结构创建数据修改更新增量记录与`hash indexes`数据选择与`hash indexes`执行时间索引页结构中间级别内部页索引扫描操作索引查找操作叶子页逻辑页ID映射表页面合并页面拆分根级别`SARGability`规则排序顺序执行计划索引键列在基于磁盘的表上创建`sys.dm_db_xtp_index_stats`视图

## O

`Objects (In-memory OLTP)` 数据库兼容级别 创建 `FILESTREAM` 闩锁 参见 `Latches` `memory-optimized tables` 创建 持久性设置 `hash indexes` 本机编译存储过程 非聚集索引 性能计数器 `T-SQL stored procedure` 等待统计信息 `Off-row storage` 脱机列的选择 `columnstore index` 支持 设计考虑 内部表 `LOB` 数据 内存消费者 内存开销 `minor_id` 性能影响 `dbo.DataInRow` 表 插入操作 `ROW_OVERFLOW` 数据 表更改 `varheaps` `HKCS` 分配器哈希索引 `LOB` 页分配器 范围索引 堆 表堆 `Operational analytics` 另请参阅 `Columnstore indexes` `Oracle` `Oracle Database In-Memory` 选项 `Oracle TimesTen`

## P

`Page merging` `Page splitting` `Performance counters` `Point-of-sale (POS)` 系统

## Q

`Query interop` 限制 `Query Store` `Top Resource Consuming Queries` 报告

## R

`Range indexes` 另请参阅 `Nonclustered indexes` `Recovery` 另请参阅 `Database recovery` `Resource Governor`, 限制可用于 `In-Memory OLTP` 的内存 `Row-based storage` `Row-level security`

## S

`SAP HANA` `SQL Server 2017` `Stored procedures` `CHECKSUM` 函数 `HASHBYTES` 函数 `sp_recompile` `sys.fn_dblog` 函数 `sys.fn_dblog_xtp` 函数 `sys.sp_xtp_bind_db_resource_pool` `sys.sp_xtp_control_proc_exec_stats` `sys.sp_xtp_control_query_exec_stats` `sys.sp_xtp_flush_temporal_history` `sys.sp_xtp_unbind_db_resource_pool` `Sweep scan` `System-versioned temporal tables`

## T, U

`Table alteration` 日志优化 仅元数据更改 简单日志 常规更改 转换表 `Trace flags` `T4199` `T9851` `Transaction` `BeginTs` 特性 原子性 一致性 持久性 隔离性 并发现象 原子性 一致性 脏读 持久性 隔离性 不可重复读 幻读 `READ COMMITTED` `REPEATABLE READ` `SERIALIZABLE` `SNAPSHOT` `EndTs` `Global Transaction Timestamp` 隔离级别 原子性 一致性 脏读 持久性 隔离性 不可重复读 幻读 `READ COMMITTED` `REPEATABLE READ` `SERIALIZABLE` `SNAPSHOT` 逻辑结束时间 逻辑开始时间 最旧的活动事务 读取集 扫描集 事务隔离级别 `REPEATABLE READ` `SERIALIZABLE` `SNAPSHOT` 原子性 一致性 持久性 隔离性 `TransactionId` 写入集 `write-ahead logging` `Transaction isolation levels` 并发现象 脏读 `in-memory OLTP` 主键冲突 `REPEATABLE READ` `SERIALIZABLE` `SNAPSHOT` 日志记录 `COMMIT` 日志缓冲区 在基于磁盘的表上修改 `UNDO` 和 `REDO` `write-ahead logging` 不可重复读 乐观并发 性能分析 悲观并发 共享锁 `SNAPSHOT` `Transaction processing` 原子性 自动提交事务 `BeginTs` 特性 原子性 一致性 持久性 隔离性 提交依赖关系, 130 提交阶段 跨容器事务 一致性 跨容器数据一致性规则, 122 可重复读验证 可序列化验证 快照验证 持久性 强制引用完整性 隔离性 乐观并发 悲观并发 提交后阶段 引用完整性强制实施 事务生命周期 验证阶段 写/写冲突 #8 `Transparent data encryption (TDE)` `T-SQL stored procedure` `atomic blocks` 另请参阅 (`Atomic blocks`) 功能 `CAST` 控制流 `CONVERT` 日期/时间函数 错误函数 `ISNULL` 数学函数 `NEWID` `NEWSEQUENTIALID` 运算符 查询表面区域 `@@ROWCOUNT` `SCOPE_IDENTITY` 字符串函数 限制 优化 `Usage scenarios` 导入批量行 会话存储 临时表和暂存表

## V, W

`Varheaps` 定义 插入选项 内存消费者

## X, Y, Z

`xtp_object_id`