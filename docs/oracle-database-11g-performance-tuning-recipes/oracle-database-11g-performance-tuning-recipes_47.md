# ![images](img/square.jpg)D

数据库配置助手 (Database Configuration Assistant, DBCA), 85

## 数据加载速度

直接路径加载, 14–16

`NOLOGGING`, 14–16

`db file scattered read` 等待事件, 156

`db file sequential read` 等待事件, 156

`DBMS_AUTO_TASK_ADMIN package`, 448

`DBMS_MONITOR package`, 327

`DBMS_WORKLOAD_REPOSITORY package`, 124–126, 129–131

`DBMS_WORKLOAD_REPOSITORY PL/SQL package`, 116

并行度 (Degree of parallelism, DOP), 452, 526, 527



## E

`Estimate_percent` 参数，453， 457

执行计划，SQL
- AUTOTRACE 特性，300–302
- `DBMS_SQLPA` 包，321–325
- `DBMS_XPLAN.DISPLAY` 函数，303–306
    - `ALL`，304
    - `BASIC`，304
    - 成本信息，305
    - 格式选项，304
    - `SERIAL`，304
    - `TYPICAL`，304
- `DISPLAY` 函数，303
- 执行统计信息，312–315
- GUI 视图，306–307
- 识别资源消耗型 SQL 语句
    - `DBA_HIST_SQL_PLAN` 视图，320
    - `DBA_HIST_SQLSTAT` 视图，319–320
    - `DBA_HIST_SQLTEXT` 视图，319
- 监控
    - `DBMS_SQLTUNE.REPORT_SQL_MONITOR` 函数，317–318
    - 长时间运行的查询，310–311
    - `V$SQL_PLAN_MONITOR` 视图，316–318
- 优化，409
    - 制定步骤，410–411
    - 提示，410， 412
    - 初始化参数，410， 412
    - 开箱即用设置，410
    - 计划基线，412
    - 草图式图表，410， 411
    - SQL 概要文件，412–417 (另见 SQL 概要文件)
    - 统计信息，412
    - 存储大纲，412
- 阅读
    - AUTOTRACE，307
    - 因素，309
    - 连接方法，310
    - 查询处理步骤，308
    - 资源消耗型 SQL 语句，311–312
- SQL 性能分析器
    - AWR 快照，324
    - 注意事项，324
    - 创建分析任务，321
    - `DBA_ADVISOR`，325
    - 执行分析任务，322
    - 报告分析任务函数，322， 325

## F

外键列，59–60
基于函数的索引，64–66

## G

`GRANULARITY` 参数，455

## H

混合列压缩，38–39

## I

`INCREMENTAL` 首选项，455–456
索引，43
- 位图，44
- 位图索引，星型模式，72
- `位图连接`，44， 73
- `B-树集群`，44
- B-树。(参见 B-树索引)
- 压缩，63
    - 优点，64
    - `COMPRESS N` 子句，63
- 连接索引，60–62
- 创建方面，43
- 决定索引哪些列
    - 外键列，51
    - 索引创建与维护指南，53
    - 索引创建标准，51
    - 带 `NOSEGMENT` 子句的索引，53–54
    - 主键约束，51
    - 唯一键约束，51
- 域，45
- 外键列，59–60
- 释放未使用的空间，78
    - 重建索引，79， 80
    - 收缩索引，79， 81
- 基于函数的，44， 64， 65， 66
- 全局分区索引，45
- `哈希集群`，44
- 索引虚拟列，44
- 索引组织表，74
    - `DBA/ALL/USER_TABLES`，74
    - `INCLUDING` 子句，75
    - `ORGANIZATION INDEX`，74
- 不可见索引
    - 优点，71
    - `OPTIMIZER_USE_INVISIBLE_INDEXES`，70
    - 用途，71
- 键压缩，44
- 本地分区索引，45
- 最大化索引创建速度，77
    - 增加并行度，77–78
    - `NOLOGGING`，78
    - 关闭重做生成，77
- 监控使用情况，75
    - 优点，76
    - `ALTER INDEX...MONITORING USAGE`，75
    - `V$OBJECT_USAGE`，76
- Oracle 索引类型，44
- 主键约束，54
    - `ALTER TABLE...AND CONSTRAINT` 语句，54
    - 内联创建约束，55–56
    - 脱机创建约束，56
    - 创建索引和约束，55
- 反向键，44， 68 (另见 B-树索引)
    - `REBUILD REVERSE`，69
    - `REVERSE` 子句，69
- 唯一索引，56
    - 添加约束，58
    - `CREATE TABLE` 语句，57–58
    - 技术，57
- 虚拟列，67
    - 注意事项，68
    - 定义，68
    - 与基于函数的索引比较，67
    - 提高性能，67
索引创建标准，51
索引快速全扫描 (IndexFFS)，361
索引组织表 (IOTs)，74–75
内连接
- 优点，259
- 过滤条件，259
- ISO 语法，258， 259
    - `JOIN ... ON` 子句，258
    - `JOIN ... USING` 子句，258
    - `NATURAL JOIN` 子句，258
- 传统 Oracle SQL，258， 259
内部查询，264
不可见索引，70
`IOSEEKTIM` 系统统计信息，465
`IOTFRSPEED` 系统统计信息，465
ISO 语法，253
- 优点，259
- 方法，258



### J

**连接条件**

交叉连接，261，262

全外连接，262

内连接，258，262。另见 `内连接`

左外连接，262

外连接。请参阅 `外连接`

右外连接，262

### K

`KEEP` 缓冲池，87–88

### L

`低基数`索引，158

### M

`METHOD_OPT` 参数，453–455

`MMON` 后台进程，139

`多列`索引，62

`多列`子查询，266–267

`多行`子查询，265

`ALL` 运算符，266

`ANY` 和 `SOME` 运算符，265

`IN` 运算符，265

### N

`NO_INVALIDATE` 参数，455

`NOLOGGING`

优点，78

数据加载速度，14–16

关闭重做日志生成，77

### O

`操作系统`性能分析，185

瓶颈

CPU 和内存 (`ps`)，197–198

I/O 统计信息，198–201

网络密集型进程，201–202

`Solaris` 系统，192–194

`vmstat`，190–192

数据库网络连接问题，202–203

决策过程，185

磁盘空间问题，187

`df` 命令，187

`du`、`sort` 和 `head` 命令，188

`filesp.bsh`，190

`find`、`ls`、`sort` 和 `head` 命令，187

`挂载点`，187

监控脚本，188–189

`usedSpc` 变量，189

识别消耗服务器资源最多的进程，194

`top` 输出的列描述，196

更改 `top` 输出的命令，196

`19888` 进程 ID，195

`top` 命令，194

隔离数据库性能问题，185

`oradebug`，206–207

`OS Watcher`，192

资源密集型进程

到数据库进程的映射，204–206

终止，207–208

排查性能不佳问题，186，187

`优化器`，447

自适应游标共享

`BIND_AWARE` 列，479，480

`绑定窥探`，476

`BIND_SHAREABLE` 列，478

`子游标`，481

`INDEX FAST FULL SCAN`，479，480

`INDEX RANGE SCAN`，479，480

`IS_BIND_AWARE` 列，478

`Oracle Database 11g`，476，481

`STATUS` 列，478

自动统计信息收集

`dbms_auto_task_admin.disable` 过程，449

`dbms_auto_task_admin.enable` 过程，449

`DBMS_STATS.GATHER_DATABASE_STATS` 过程，450

`DBMS_STATS.GATHER_DATABASE_STATS_JOB_PROC` 过程，450

`enable` 过程，448

`GATHER_DATABASE_STATS` 过程，450

`SYS` 和 `SYSTEM` 方案，450

`绑定窥探`行为，447

`批量加载`的表，457

`列组`，484–486

并发统计信息收集

`DBMS_STATS.GATHER_TABLES_STATS` 过程，488

`job_queue_processes` 参数，488

监控并发统计信息收集作业，489

多处理器环境，488

`Oracle Database 11g Release 2`，488

并行执行策略，489

`SET_GLOBAL_PREFS` 过程，488

导出统计信息，460–462

目标，447–448

`直方图`，472–473



## 索引
`索引`, 468–470

锁定统计信息, 458

缺失的统计信息, 459–460

新建统计信息, 466–468

未使用绑定变量, 473–476

分区表, 486–487

查询优化器特性, 470–471

相关列, 483–484

恢复先前版本, 462–463

表达式上的统计信息, 482–483

## 系统统计信息
### I/O 与 CPU 特性
463

### 间隔参数
464

### `IOTFRSPEED`、`IOSEEKTIM` 和 `CPUSPEEDNW`
465

### `mbrc` 和 `mreadtim` 统计信息
466

### 无负载统计信息
463

### Oracle 11g 数据库
465

### 负载模式
465

### 负载统计信息
464

## 统计信息类型
### `add_sys`
451

### `AUTOSTATS_TARGET` 参数
456

### `CASCADE` 参数
452

### `DEGREE` 参数
452

### `ESTIMATE_PERCENT` 参数
452–453

### `GRANULARITY` 参数
455

### `INCREMENTAL` 首选项
455–456

### `METHOD_OPT` 参数
453–455

### `NO_INVALIDATE` 参数
455

### `pname`
451

### `PUBLISH` 参数
455

### `pvalue`
451

### `SET_DATABASE_PREFS`
451

### `SET_GLOBAL_PREFS`
451

### `SET_SCHEMA_PREFS`
451

### `SET_TABLE_PREFS`
451

### `STALE_PERCENT` 首选项
456

### 易变表
457

## `Optimizer_dynamic_sampling` 初始化参数
459

## `Optimizer_features_enable` 参数
470–471

## `Optimizer_index_cost_adj` 参数
468–470

## `Optimizer_use_pending_statistics` 参数
468

## Oracle 自动存储管理 (ASM)
7

## Oracle Autotrace 实用程序
47, 48

## Oracle 基本压缩
混合列式压缩, 38–39

直接路径加载, 34

优势, 35

`ALTER` 语句, 35

`COMPRESS` 子句, 35

`CREATE TABLE...AS SELECT` 语句, 34

`DBA/ALL/USER_TABLES` 视图, 34

DML 语句, 35

`MOVE COMPRES` 子句, 35

DML

`ALTER TABLE` 语句, 37

`COMPRESS FOR OLTP` 子句, 36, 37

I/O 性能, 36

OLTP 压缩, 36, 37

Oracle Database 11g, 328, 331

Oracle Database 11g R2, 3, 6, 12–14

Oracle 诊断包, 115

Oracle 企业管理器, 331

`$ORACLE_HOME/rdbms/admin` 目录, 117

Oracle 监听器, 363–365

Oracle 锁, 164

Oracle 标准审计特性, 40

## Oracle Trace Analyzer
327, 344

`/trca/install/trcreate.sql` 脚本, 341

CoE, 342

单个 SQL, 343

安装与运行步骤, 341

非默认初始化参数, 343

非递归时间与总计, 343

自身时间、总计、等待、绑定和行源计划, 343

表和索引, 343

`tacreate.sql` 脚本, 341

顶级 SQL, 343

`trca_instructions` HTML 文档, 342

ZIP 文件, 342

## `oradebug`
206

## 外连接
交叉连接, 261

`FULL OUTER JOIN`, 261

ISO SQL 语法, 260

ISO 语法, 261

左外连接, 260

Oracle SQL, 261

右外连接, 260

语法, 260

传统 Oracle SQL, 260


### ![images](img/square.jpg)P

*   并行， 525
*   瓶颈， 550–552
*   创建索引， 538–539
*   创建表， 536
*   DDL 优势， 537
*   删除行， 537
*   缺点， 538
*   原因， 536
*   并行度， 526–528, 543–545
*   DML 操作， 533
    *   `ALTER SESSION ENABLE PARALLEL DML`， 533
    *   `ALTER SESSION FORCE PARALLEL DML`， 533
    *   注意事项， 535–536
    *   `DBMS_PARALLEL_EXECUTE` PL/SQL 包， 534
    *   `DOP`， 535
    *   `INSERT` 语句， 533
    *   限制， 535
    *   `UPDATE`、`MERGE` 和 `DELETE` 语句， 534
*   已有对象， 532
*   执行计划， 545, 546
*   执行步骤， 547
*   操作， 548
*   通用规则， 525
*   索引， 526–528, 529–530
*   监控操作， 548–550
*   移动分区， 541
*   对象创建， 530–531
*   陷阱， 525
*   重建索引， 539–541
*   会话详细信息， 552–553
*   拆分分区， 542–543
*   表， 526, 528–529
*   类型， 528
*   理解因素， 525
*   理解系统组件， 527

执行计划基线

*   更改
    *   `ALTER_SQL_PLAN_BASELINE` 函数， 434, 435
    *   `ATTRIBUTE_NAME` 和 `ATTRIBUTE_VALUE`， 435
    *   `DBMS_SPM` 包， 434
*   优势， 410
*   `DBA_SQL_BASELINES`， 436
*   禁用， 442
*   `DISPLAY_SQL_PLAN_BASELINE` 函数， 437–439
*   `EVOLVE_SQL_PLAN_BASELINE` 函数， 439–441
*   管理任务， 428
*   `OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES`， 427
*   `PACK STGTAB BASELINE` 函数， 446
*   移除， 443–444
*   针对 SQL 语句， 428
    *   AWR 基线， 431
    *   `DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE` 函数， 429
    *   `LOAD_PLANS CURSOR CACHE` 函数， 430
    *   资源密集型查询， 433
    *   调优集对象， 431–433
*   传输， 444–456

主键索引， 54–56

程序全局区 (`PGA`)， 85

`PUBLISH` 参数， 455, 467

### ![images](img/square.jpg)Q

查询协调器 (`QC`) 控制， 345

查询提示， 491

*   缓存查询结果， 509–513
    *   配置层级， 512
    *   初始化参数， 513
*   更改访问路径
    *   索引， 493–496
    *   全表扫描， 493, 495
*   更改连接方法， 498
    *   哈希连接， 499, 501
    *   必要因素， 500
    *   嵌套循环， 498–499, 501
    *   查询多表， 500–501
    *   智能合并连接， 500
    *   排序合并， 499–501
*   更改连接顺序， 497
    *   `LEADING` 提示， 497–498
    *   `ORDERED` 提示， 497
*   更改优化器版本， 501–502
*   引导分布式查询， 513
    *   缺点， 514
    *   驱动站点， 514–517
*   直接路径插入技术， 505–506
*   快速响应与整体优化， 502–504
*   `GATHER_PLAN_STATISTICS` 提示， 517–519
*   提示编写， 491–493
*   在视图中， 506
    *   复杂视图， 507
    *   可合并视图， 507, 508
    *   不可合并视图， 508
    *   简单视图， 507
*   `REWRITE` 提示， 519–521
*   星型信息/事实提示， 521–523

### ![images](img/square.jpg)R

恢复写入器进程 (`RVWR`)， 162–163

`RECYCLE` 缓冲区池， 87, 88

资源密集型进程

*   转换为数据库进程， 204
*   终止
    *   操作系统 `Kill` 命令， 208
    *   使用 SQL， 207

反向键索引， 68–69


![images](img/square.jpg)

