# 索引

## A
管理与监控任务 `extended events` 另请参阅（`Extended events` `memory-optimized tables` `Memory Usage by Memory Optimized Objects` 报告监控 `In-Memory OLTP` 事务监控内存使用情况 `resource governor` 另请参阅 (`Resource Governor`，限制可用于 `In Memory OLTP` 的内存）内部和默认资源池恢复过程 `sys.dm_db_xtp_table_memory_stats` `SQL Server Database Engine` 的体系结构 `Atomic blocks`

## B
`BEGIN ATOMIC` 另请参阅 `Atomic blocks` `Buffer Manager` `Buffer pool` `Bw-tree`

## C
`Catalog views` 另请参阅 `Data management views` `Checkpoint file` 关闭线程 控制器线程 I/O 要求 段 日志记录段 序列化器线程 计时器任务 `Checkpoint file pairs (CFPs)` 另请参阅 `Checkpoint file` `ACTIVE` 检查点文件 数据文件 增量文件 强制执行 `CHECKPOINT` 大型数据文件 生命周期 数据库创建 日志备份与垃圾回收 合并过程 `memory-optimized table` 根文件 状态 `ACTIVE` `MERGE TARGET` `PRECREATED` `UNDER CONSTRUCTION` `WAITING FOR LOG TRUNCATION` `CHECKSUM` 函数 `Clustered columnstore indexes` 另请参阅 `Columnstore indexes` 内部对象 行组 `Clustered index` 在基于磁盘的表上

## D
`Data partitioning` `Data row` `BeginTs` 时间戳 `EndTs` 时间戳 `IdxLinkCount` 索引指针数组 有效负载 行头 `StmtId`, 34, 结构 `Data storage` 另请参阅 `Checkpoint files` `CFP` `CHECKPOINT` 进程 `MERGED SOURCE` `CFP` 状态 `MERGE TARGET` 状态 `UNDER CONSTRUCTION` 状态 基于磁盘的表 `Data warehouse workload` 另请参阅 `Columnstore indexes` `Deployment and management` 管理与监控 参见 `Administration and monitoring tasks` 估算内存需求 硬件组件 `CPU` `I/O subsystem` 内存 `Design considerations` 二进制排序 成本/收益分析 数据分区 索引策略 另请参阅 (`Index design considerations`) 可维护性和管理开销 引用完整性 具有混合工作负载的系统 不支持的数据类型

## E
`Extended events`

## F
`FILESTREAM`

## G
`Garbage collection` `BeginTs` 和 `EndTs` 时间戳 数据管理视图 `DELETE` 操作 删除/更改表 被遗忘的角落扫描 代次 幽灵行 目标 空闲工作线程 `idxLinkCount` 索引页元素 `memory-optimized table` 表创建 内存统计 表删除 无阻塞 过期行 摘要统计信息 `UPDATE` 操作 工作流程 工作项 工作队列

## H
`Halloween effect` `HASHBYTES` 函数 `Hash indexes` `bucket_count` 合适的数量 `sys.dm_db_xtp_hash_index_stats` 选择 `bucket_count` 冲突映射 与 非聚集索引 数据选择 执行时间 点查找性能 `SARGability` 规则 `Hashing` 冲突 哈希函数 哈希映射 哈希表 `High Availability Technologies`

## I, J, K
`IBM DB2` `In-memory database (IMDB)` `IBM DB2` `Oracle` `SAP HANA` `In-memory OLTP migration` 另请参阅 `Migration tools` 二进制排序 性能 `clustered columnstore indexes` 数据管理视图 事务管理 数据分区 数据移动 执行计划 对象创建 订单录入系统查询 数据库级限制 设计目标 设计考虑 基于磁盘的表 引擎架构 目标 `IMDB` 参见 `In-memory database (IMDB)` 导入批量行 客户端代码 `memory-optimized table type` 表, `TVP` 和存储过程 索引注意事项 限制 `memory-optimized tables` 扫描性能 语句级重新编译 存储过程，执行时间 变量和基数估计 与基于磁盘的临时对象 迁移 混合工作负载 数据分区 本机编译模块 性能计数器 查询互操作引擎 会话/对象状态存储 专用存储/缓存 `ObjStoreDataAccess` 类 `ObjStoreService` 类 `ObjStoreUtils` 类 复制内容 可扩展性问题 会话存储实现 不支持的数据类型 乐观并发

## L
`Latches` 在基于磁盘的表上 等待统计信息 `LEGACY_CARDINALITY_ESTIMATION` 数据库范围的配置 `Log buffer`

## M
`Management data warehouse` 内存和本机编译 顾问 菜单配置 过程级统计信息 表争用分析 报告 表级统计信息 使用情况分析报告 `Memory consumers` 另请参阅 `Off-row storage` `Memory-optimized table` `BeginTs` 时间戳 数据行 `halloween effect` 结构 `DURABILITY` `SCHEMA_AND_DATA` `SCHEMA_ONLY` `EndTs` 时间戳 限制 数据库级 数据类型 表 `SQL Server 2016` 统计信息 嵌套循环联接算法 变量 支持的数据类型 支持的表功能 `Memory pointers` 管理 数据修改和并发 `Microsoft Azure SQL Databases` `Migration tools` `In-Memory OLTP Migration Checklists Wizard` `Memory Optimization Advisor` 本机编译顾问 `Stored Procedure Analysis` 报告 `Tables Analysis` 报告 `Transaction Performance Analysis Overview` 报告

## N
`Native compilation` `atomic blocks` 另请参阅 (`Atomic blocks`) 内联表值函数 互操作模式 性能比较 `InsertCustomers` 与 解释型函数 函数创建 函数内的循环 多次调用 `memory-optimized table types` `memory-optimized table variables` 本机编译存储过程 概述 性能比较 安全上下文 存储过程 参见 `T-SQL stored procedure` 支持的 T-SQL 功能 控制流 函数 运算符 查询表面区域 触发器和用户定义函数 `Nested loop join algorithm` `Nonclustered index` 在基于磁盘的表上 `Nonclustered indexes` `Bw-Tree` 结构 创建 数据修改 更新增量记录 与 `hash indexes` 数据选择 与 `hash indexes` 执行时间 索引页结构 中间级别 内部页 索引扫描操作 索引查找操作 叶子页 逻辑页 ID 映射表 页面合并 页面拆分 根级别 `SARGability` 规则 排序顺序 执行计划 索引键列 在基于磁盘的表上创建 `sys.dm_db_xtp_index_stats` 视图

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
