# 索引

## 锁类型

- `extent-level lock`， 406
- `DBCC IND` 和 `DBCC PAGE`， 245
- `heap/B-tree lock`， 407
- `dbo.Test1`， 245
- `KEY lock`， 405
- `page split`， 244
- `page-level lock`， 406
- `sys.dm_db_index_physical_`
- `row-level lock`， 404–405
- `stats output`， 244
- `TAB lock`， 407

## 索引与存储

- `internal fragmentation`， 239, 245
- `nonclustered index`， 426–428
- `leaf pages`， 237–238
- `schema modification`， 414
- `SELECT statements`， 246–247
- `schema stability modes`， 414
- `small table analyzing`， 250
- `shared mode`， 409
- `sys.dm_db_index_physical_stats`
- `test table, no index`， 425–426
- `clustered index`， 248
- `update mode`
- `detailed scan`， 248–249
- `deadlock`， 412–413
- `mixed extents`， 247
- `lock conversion state`， 411
- `output`， 248
- `lock status`， 411
- `uniform extent`， 247

## 查询与优化

- `REPEATABLEREAD`
- `UPDATE statement`
- `locking hint`， 412
- `clustered index`， 239–240
- `Lookups`
- `DBCC IND output`， 242–243
- `cause`， 184
- `page_count column`， 240
- `clustered index`， 186
- `page split`， 241–242
- `covering index`， 186
- `PageType`， 243
- `DBCC SHOWSTATISTICS`， 189–190
- `SELECT statement`， 240
- `execution plan`， 187
- `sys.dm_db_index_`
- `HireDate`， 186
- `physical_stats`， 241
- `INCLUDE columns`， 187
- `INDEX hint`， 183, 370, 372
- `index storage`， 188
- `Index Seek operation`， 127
- `JobTitle`， 186
- `Internal fragmentation`， 239, 245
- `NationallDNumber`， 188
- `unexpected covering index`， 188
- `drawbacks`， 183
- `execution plan`， 182
- `Keyset-driven cursors`， 465, 471
- `index join`
- `execution plan`， 191
- `Multiple optimization phases`
- `Key Lookup operation`， 191
- `configuration cost`， 275
- `logical reads`， 190, 192
- `DMV`， 277–278
- `ProductID`， 181
- `index variations`， 275
- `SELECT statement`， 181, 185
- `nontrivial plan`， 276
- `WHERE clause`， 182
- `QueryPlanHash`， 277
- `LOOP join hint`， 370
- `size and complexity`， 275
- `T-SQL SELECT operator`， 276–277
- `WHERE clause`， 275

## 内存优化

- `Memory Optimization Advisor`
- `in-memory table migration`， 502
- `InMemoryTest database`， 499–500

[www.it-ebooks.info](http://www.it-ebooks.info/)


## 性能分析相关

*   `本地编译顾问`， 502
*   `选项`页面， 500
*   `非聚集索引`， 113
*   `输出`， 498–499
*   `AdventureWorks2012`， 195
*   `主键`， 501
*   `阻塞与死锁问题`， 138
*   `书签查找`， 137
*   `DBCC MEMORYSTATUS`， 27
*   `覆盖索引`， 138
*   `DMO`， 29
*   `执行计划`， 492–494
*   `动态管理对象`， 19
*   `查找`（参见 `Lookups`）
*   `硬件资源`， 20
*   `行定位器`， 137
*   `性能监视器工具`， 17
*   `UPDATE` 操作， 138

## 内存性能分析

*   `DBCC MEMORYSTATUS`， 27
*   `可用字节`计数器， 25
*   `缓冲区缓存命中率`， 26
*   `缓冲池`， 21
*   `检查点页数/秒`计数器， 26
*   `配置`， 21–22
*   `动态内存`， 24
*   `惰性写入/秒`计数器， 27
*   `最大服务器内存`， 22
*   `内存授予挂起`计数器， 27
*   `内存压力分析`， 24
*   `最小服务器内存`， 22
*   `页面文件使用率百分比`， 25
*   `页面预期寿命`， 26
*   `页数/秒`计数器， 25
*   `RECONFIGURE` 语句， 23
*   `sp_configure` 系统， 23
*   `目标与总服务器内存`， 27

### 不可搜索的搜索条件

*   `BETWEEN` 与 `IN/OR` 对比， 358
*   `!< 条件` 与 `>= 条件` 对比， 361
*   `LIKE` 条件， 360–361
*   `NOT NULL` 约束， 372

### 内存表与优化

*   `内存中表`， 32
*   `内存分配`， 32
*   `优化应用程序工作负载`， 32
*   `进程地址空间`， 3GB， 33
*   `本地临时表`
*   `系统内存`， 33

### SQL Server 管理

*   `扩展事件输出`， 334
*   `架构`， 335
*   `存储过程`
    *   `重编译`， 333
*   `SELECT` 语句， 332
*   `sql_statement_recompile` 事件， 333
*   `表创建`， 332
*   `联机索引创建`， 163
*   `联机事务处理 (OLTP)`

### 编译过程

*   `已编译过程`
    *   `原子块`， 496
    *   `错误`， 496
    *   `估计计划`， 496
    *   `扩展事件`， 496
    *   `性能尖叫`， 495
    *   `SELECT` 算子， 497
*   `数据库设置`， 484
*   `描述`， 483
*   `执行计划`， 489–490

### Hash 索引

*   `深度分布`， 491
*   `Microsoft SQL Server 2012`， 165
*   `定义`， 490
*   `多活动结果集 (MARS)`， 472

### 索引维护

*   `索引维护`， 494–495

---

*   [www.it-ebooks.info](http://www.it-ebooks.info/)
*   `联机事务处理 (OLTP)` （续）



-   `maintenance reexamination`, 319
-   `nonclustered indexes`, 492–494
-   `stored procedure`, 314
-   `shallow distribution`, 491
-   `sys.dm_exec_query_stats` output, 315
-   `sys.dm_db_xtp_hash_index_stats`, 491–492
-   `values`, 314
-   `in-memory index`, 490
-   `Parse tree`, 271
-   `Memory Optimization Advisor`
-   `Partition elimination`, 45
-   `in-memory table migration`, 502
-   `Performance Monitor` counter
-   `InMemoryTest` database, 499–500
-   counter log
-   `Options` page, 500
-   `data collector set`, 61
-   output, 498–499
-   `data logs`, 61–62
-   `primary key`, 501
-   definition, 63–64
-   `Native Compilation Advisor`, 502
-   `Performance Monitor` graph, 65
-   `performance baseline`, 498
-   schedule pane, 62–63
-   `Person.Address` table, 485–487
-   counter number, 65
-   `query`, 486–487
-   `Database` blocking
-   system requirements, 484
-   `alert` properties, 440
-   unsupported data types, 486
-   `blocking` analysis, 438–439
-   workloads, 498
-   `blocking` detection, 441
-   `Optimistic concurrency model`, 462, 468
-   `blocking` session, 439
-   `Optimizer` hints
-   description, 438
-   `INDEX` hints, 370
-   `SQL Server` alerts, 438
-   `JOIN` query hint, 367
-   reusable list, 58–60
-   `execution plan`, 368–369
-   `sampling interval`, 65
-   `LOOP` join hint, 370
-   `Performance tuning` process
-   `SELECT` statement, 368
-   `baseline` performance, 9
-   `SQL Server 2014`, 367
-   `data access layer`, 10
-   `STATISTICS IO` and `TIME` outputs, 369
-   `database` connection, 4
-   types, 367
-   `database` design, 4
-   `hardware` and `software` factors, 2
-   „ **P**
-   high level `database`, 11
-   iteration process
-   `Page-level compression`, 156
-   `costliest query`, 7
-   `Parallel index creation`, 163
-   `user activity`, 5
-   `Parallel plan optimization`
-   low level `database`, 11
-   `affinity setting`, 278
-   `optimization`, 4
-   `cost factors`, 278
-   `performance killers`
-   `cost threshold`, 279
-   `cursors`, 14
-   `DML action queries`, 280
-   `database transaction log`, 14
-   `MAXDOP` query hint, 279
-   excessive `blocking` and `deadlocks`, 13
-   `number of CPUs`, 279
-   excessive `fragmentation`, 14
-   `OLTP queries`, 280
-   frequent `recompilation`, 14
-   `query` execution, 280
-   inaccurate `statistics`, 12
-   `Parameter sniffing`, 389



不恰当的数据库设计, 13

`AddressByCity`, 312

索引不足, 11

不良参数

不可重用的执行计划, 14

标识, 316

非基于集合的操作, 13

I/O 和执行计划, 315

参数探测, 12

`Mentor`, 315

查询设计, 12

缓解行为, 317

`SQL Server`, 11

老派方法, 318

`tempdb`, 15

`OPTIMIZE FOR` 提示, 318–319

`vs`. 价格, 8

运行时和编译时值, 319

优先级, 3

`SELECT` 属性, 319

查询优化, 1, 4

定义, 311

根本原因, 10

局部变量, 311–313

`SQL Server` 配置, 3

[www.it-ebooks.info](http://www.it-ebooks.info/)

■ 索引

`Person.Address` 表, 125–126

`SET NOCOUNT` 语句, 392

计划指南

事务成本

执行计划, 350, 353

减少锁开销, 394

`Index Seek` 操作, 353

减少日志记录开销, 392

`OPTIMIZE FOR` 查询提示, 350

`UNION ALL` 子句, 384

`SELECT` 操作符属性, 351

`UNION` 子句, 383

`sp_create_plan_guide_from_handle`, 354

查询设计建议

`SQL` 查询, 354

避免优化器提示 (参见 优化器提示)

存储过程, 354

域和参照完整性

`T-SQL` 语句, 352

`DRI`, 375

准备工作负载，计划可重用性

`NOT NULL` 约束, 372

准备/执行模型, 302

有效的索引

`sp_executesql`, 300

避免算术运算符, 362

存储过程, 295

避免在 WHERE 子句列上使用函数

`ProductID`, 181

(参见 WHERE 子句列)

`Productld` 列, 118

避免非可搜索的搜索条件

`PurchaseOrderHeader`, 190

(参见 非可搜索的搜索条件)

小结果集

`Q`

有限的列数, 356

多列, 357

查询分析

`WHERE` 子句, 357

缺失统计信息问题

`QueryPlanHash`, 277, 280

`ALTER DATABASE` 命令, 227

查询计划哈希和查询哈希

`CREATE STATISTICS` 语句, 229

创建查询, 303

执行计划, 229

数据分布和索引的差异, 304

图形化计划, 228

截然不同的计划, 305

`Index Scan` 操作符, 228

`SELECT` 条件, 304

`SELECT` 语句, 228

查询重新编译

测试表创建, 227

优点和缺点, 321

`XML` 计划, 228

原因



过时的统计问题

编译过程，326

数据库，232

执行计划老化，336

`DBCC SHOW_STATISTICS` 命令，230

对象解析（参见 对象解析）

估计行数与实际行数

`RECOMPILE` 子句（参见 `RECOMPILE` 子句）

值，231–232

架构/绑定更改，328

执行计划，231

`SET` 选项更改，335

`iFirstIndex`，230

`sp_recompile`，336

`inaccurate_cardinality_estimate`，230

`sql_statement_recompile` 事件，327

`SELECT` 语句，230

语句重新编译，327

`Table Scan` 运算符，231

统计信息更改，328, 330

查询编译，271

执行计划，322

查询设计分析

实现

聚合与排序条件，384

`DDL`/`DML` 语句，340

`EXISTS` 对比 `COUNT(*)` 方法，382

禁用自动统计信息

隐式数据类型转换，379

更新，344

局部变量，批处理查询

`KEEPFIXED PLAN`，342

聚集索引，388

`OPTIMIZE FOR` 查询提示，347, 349

执行计划，385–386

计划指南（参见 计划指南）

参数嗅探，389

`SET` 选项，346

相对成本，386–387

统计信息更改，342

`SELECT` 语句，385

表变量，344

`STATISTICS IO` 和 `TIME`，386

索引 `IX_Test`，322

`WHERE` 子句，385, 389

非有益的重新编译，323

多次查询执行，391

`SELECT` 语句，322–323

命名存储过程，389

`SQL Server` 规则，322

`www.it-ebooks.info`

查询重新编译（续）

代价高昂的查询

`sql_statement_recompile` 事件，323

扩展事件数据，86

语句识别

多次执行，88

事件，324

查询执行计划，86

`Extended Events` 输出，324–325

查询优化器，86

`sp_statement_starting` 事件，326

减少数据库阻塞和压力，85

存储过程，321

单次执行，87

运行缓慢的查询，90

执行计划

实际执行计划与估计执行计划，91, 103

`只读` 并发模型，462, 468

分析索引有效性，96

`重新编译` 阈值 (`RT`)，328

客户端统计信息，105

`RECOMPILE` 子句

成本效益高的执行计划，91

`CREATE PROCEDURE` 语句，338

动态管理视图和函数，105


## 查询优化相关术语

EXECUTE 语句, 339
执行时间, 106
RECOMPILE 查询提示, 339
图形化执行计划, 91

### 存储与硬件

独立磁盘冗余阵列 (RAID), 36
配置, 39
识别, 95
RAID 0, 40
RAID 1, 9, 40
RAID 1+0 (RAID 10), 41
RAID 5, 13, 40
RAID 6, 14, 41
固态硬盘 (SSD), 41

### 连接与访问方式

哈希连接, 99
索引扫描/查找, 93
合并连接, 101
嵌套循环连接, 102

### 查询优化器与属性

操作符选择属性, 94–95
查询优化器, 91
查询资源成本, 105
参照完整性, 551
远程过程调用 (RPC) 事件, 72

### 命令与语句

SET SHOWPLAN_XML 命令, 91
STATISTICS IO, 108
SET NOCOUNT 语句, 392
SET 语句, 444–445
SET STATISTICS IO, 118

### 锁与并发

前滚, 402
逐行痛苦处理 (RBAR) 流程, 459
行级压缩, 156
滚动锁并发模型, 463, 469
可序列化隔离级别
索引的影响, 429
HOLDLOCK 锁定, 422–423
PayBonus 事务, 420–421, 423, 425
幻读, 420
快照隔离, 425
服务器端游标位置, 462, 467

### 其他概念与工具

工具提示表, 94
XML 执行计划, 91

## 扩展事件

数据存储, 78
调试事件, 82
定义, 72
事件字段, 76
事件库, 72–73
筛选器, 75
全局字段, 74
图形用户界面, 69
监视查询完成, 72
新建会话窗口, 70–71
无事件丢失, 83
物理 I/O 操作, 77
资源压力, 72
RPC 事件, 72
会话完成, 79
会话脚本, 80
设置最大文件大小, 82
模板, 70–71
T-SQL, 72, 81

### 事件与表

SP:CacheHit 事件, 390
SP:CacheMiss 事件, 391
SalesOrderHeader 表, 337
可搜索的搜索条件, 358

## SQL Server 优化

### 动态管理视图与技术

sys.dm_exec_procedure_stats, 83
sys.dm_exec_query_stats, 83
sp_executesql 技术
额外输出, 302
参数化计划, 301
计划敏感性, 301

### 语句与工作负载

SELECT 语句, 118
SELECT 语句, 300
即席工作负载, 555


