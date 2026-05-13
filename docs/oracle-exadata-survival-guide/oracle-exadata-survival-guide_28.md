# 索引

![image](img/square1.jpg)  A

alter table emp compress

Auto DOP

校准

CALIBRATION_TIME 值

DBA_RSRC_IO_CALIBRATE

DBMS_RESOURCE_MANAGER

执行计划

IN PROGRESS 状态

延迟

parallel_min_time_threshold

PL/SQL 块

RESOURCE_IO_CALIBRATE$

串行执行时间

tkprof 输出

V$IO_CALIBRATE_STATUS 结果

V$IO_CALIBRATION_STATUS

DEGREE 设置

INSTANCES 设置

_parallel_cluster_cache_policy 参数

parallel_degree_policy 设置

_parallel_statement_queueing

自动存储管理 (ASM)


## B

![image](img/square1.jpg)

### 自动服务请求（ASR）

自动服务请求（ASR）

### 基础压缩

基础压缩

### 布隆过滤器

布隆过滤器

## C

![image](img/square1.jpg)

### Celladmin

Celladmin

*   已确认和未确认的警报
*   警报历史
*   单元格配置检查
*   `cellsrvstat`
*   磁盘控制器电池
*   固件版本
*   功能与指标
*   学习周期
*   统计类别
*   用户账户

已确认和未确认的警报

警报历史

单元格配置检查

`cellsrvstat`

磁盘控制器电池

固件版本

功能与指标

学习周期

统计类别

用户账户

### 单元管理器取消工作请求

单元管理器取消工作请求

### 单元管理器关闭单元格

单元管理器关闭单元格

### 单元管理器发现磁盘

单元管理器发现磁盘

### 单元管理器打开单元格

单元管理器打开单元格

### CELLSRV

`CELLSRV`

### 单元格等待事件

单元格等待事件

*   单元格相关事件
*   空闲类/其他类
*   单元格智能闪回保持
*   极少报告的单元格等待事件
*   系统 I/O 事件

单元格相关事件

空闲类/其他类

单元格智能闪回保持

极少报告的单元格等待事件

系统 I/O 事件

### 单元格智能增量备份

单元格智能增量备份

### 单元格从备份智能恢复

单元格从备份智能恢复

### 单元格启动和关闭等待事件

单元格启动和关闭等待事件

*   触发事件

触发事件

*   用户 I/O 事件

用户 I/O 事件

### 单元格物理读取的块列表

单元格物理读取的块列表

### 单元格多块物理读

单元格多块物理读

### 单元格单块物理读

单元格单块物理读

### 单元格智能文件创建

单元格智能文件创建

### 单元格智能索引扫描事件

单元格智能索引扫描事件

### 单元格智能表扫描事件

单元格智能表扫描事件

### 单元格统计信息收集

单元格统计信息收集

### 单元格工作进程空闲

单元格工作进程空闲

### 单元格工作进程联机完成

单元格工作进程联机完成

### 单元格工作进程重试

单元格工作进程重试

### 聚集因子

聚集因子

### 压缩

压缩

*   基础压缩
*   HCC
*   加载数据
*   加载时间与数据量
*   结果
*   OLTP
*   查询过程

基础压缩

HCC

加载数据

加载时间与数据量

结果

OLTP

查询过程

*   执行计划
*   执行时间
*   结果
*   比率
*   实际观测值
*   估计值

执行计划

执行时间

结果

比率

*   实际观测值
*   估计值

实际观测值

`DBMS_COMPRESSION.GET_COMPRESSION_RATIO` 过程

估计值

更新

*   基础级别压缩
*   已用时间
*   增大表大小

基础级别压缩

已用时间

增大表大小

### 压缩单元（CUs）

压缩单元（CUs）

*   位图
*   头部
*   初始数据组织
*   布局

位图

头部

初始数据组织

布局

## D

![image](img/square1.jpg)

### 数据库机器管理员（DMA）

数据库机器管理员（DMA）

*   资源

资源


## 技术概念与术语索引

### 基本概念与组件

#### 数据库系统基础
*   职责
*   数据库服务器工具
*   dbID
*   dbUniqueName
*   数据库创建
*   数据迁移
*   环境文件
*   环境配置
*   Grid Infrastructure Oracle Home
*   MOS
*   SCAN 监听器
*   数据仓库
*   定义
*   扩展机架
*   OLTP
*   原始存储
*   智能闪存缓存
*   ERP 系统

#### Exadata 相关
![image](img/square1.jpg) 储能模块 (ESM)
*   Exadata
*   布隆过滤器
*   优势
*   定义
*   卸载处理
*   查询执行
*   列投影
*   配置
*   数据库层
*   卸载功能
*   谓词过滤
*   智能扫描

### 性能监控、诊断与优化

#### 监控视图与报告
*   DBA_HIST_ACTIVE_SESS_HISTORY 视图
*   DISPLAY_CURSOR 函数
*   GV$SQL 和 GV$SQLSTATS
*   实时 SQL 监控报告
*   HTML 报告
*   OEM 12c
*   SQL 执行计划窗口

#### 监控数据与指标
*   有效的监控策略
*   数据库层处理时间
*   全局信息输出
*   索引访问
*   卸载效率
*   计划级监控数据
*   会话级执行监控数据
*   初始基线
*   持续时间
*   持续时间列
*   I/O 窗口
*   计划提供时间
*   会话 ID
*   SQL ID
*   等待统计信息

#### 事件与等待
*   数据文件初始化写事件
*   db 文件分散读
*   db 文件顺序读事件
*   直接路径读事件
*   传统的一致性读处理

### 执行、优化与相关工具

#### 执行计划与优化
*   并行度 (DOP)
*   并行执行
*   autotrace
*   CPU 周期
*   执行

#### 函数、过程与脚本
*   `DBMS_COMPRESSION.GET_COMPRESSION_RATIO` 过程
*   `SYS_OP_BLOOM_FILTER` 函数
*   bloom_fltr_ex.sql 脚本
*   agg_smart_scan_ex.sql 脚本
*   offloadable_fn_tbl.sql 脚本

#### 函数类型与查询
*   COUNT() 查询
*   多行函数
*   单行函数



## F
- `全表扫描`
- `索引范围扫描`
- `io_cell_offload_eligible_bytes`
- `no_smart_scan_ex.sql 脚本`
- `运算符`
- `优化`
- `TABLE ACCESS STORAGE FULL 操作`
- `V$SQL 视图`
- `存储单元 (*参见* 存储单元)`
- `虚拟列`
- `smart_scan_virt_ex.sql`
- `TOTAL_COMP 列`
- `TTL_COMP 跳过`
- `先进先出 (FIFO) 队列`

## G
- `Grid Infrastructure Oracle Home`

## H
- `混合列压缩 (HCC)`
- `ARCHIVE`
- `EMP 表`
- `GET_COMPRESSION_TYPE`
- `高压缩率`
- `低压缩率`
- `压缩单元`
- `Data Guard`
- `Exadata 存储`
- `选项`
- `QUERY`
- `EMP 表`
- `高压缩率`
- `低压缩率`

## I, J
- `INDEX RANGE SCAN`
- `INDEX UNIQUE SCAN`
- `InfiniBand 网络`
- `内存中并行执行`
- `感兴趣的事务列表 (ITL)`
- `进程间通信 (IPC)`
- `每秒 I/O 操作数 (IOPS) 速率`

## K
- `内核数据扫描 (KDS)`

## L
- `最近最少使用算法`

## M
- `指标与计数器`
- `My Oracle Support (MOS)`

## N
- `NO_STATEMENT_QUEUING 提示`

## O
- `objectNumber 属性`
- `卸载函数`
- `agg_smart_scan_ex.sql 脚本`
- `COUNT() 查询`
- `多行函数`
- `offloadable_fn_tbl.sql 脚本`
- `单行函数`
- `OLTP 压缩`
- `联机事务处理 (OLTP)`
- `Oracle 客户支持 (OCS)`
- `Oracle 企业管理器 (OEM)`

## P, Q
- `parallel_min_time_threshold 参数`
- `并行处理`
- `DOP`
- `内存中并行执行`
- `I/O 校准`
- `Oracle 11.2.0.x`
- `V$SQL_MONITOR 视图`
- `并行查询`
- `自动 DOP (*参见* 自动 DOP)`
- `平均薪资结果`
- `控制问题`



## R

-   真正应用集群 (RAC)
-   恢复管理器 (RMAN)
-   ![image](img/square1.jpg)  R

## S

-   串行连接小型计算机系统接口 (SAS)
-   单一客户端访问名称 (SCAN) 监听程序
-   SizeArr 值及由此产生的存储分配
-   智能闪存缓存
-   `ALTER INDEX command`(#9781430260103_Ch04.xhtml#cXXX.95)
-   `ALTER TABLE command`(#9781430260103_Ch04.xhtml#cXXX.94)
-   `CellCLI`(#9781430260103_Ch04.xhtml#cXXX.81)
-   `CELL_FLASH_CACHE setting`(#9781430260103_Ch04.xhtml#cXXX.91)
-   `CELLSRV`(#9781430260103_Ch04.xhtml#cXXX.93)
-   `CREATE TABLE statement`(#9781430260103_Ch04.xhtml#cXXX.86)
-   `DEFAULT setting`(#9781430260103_Ch04.xhtml#cXXX.87)
-   磁盘
-   ASM
-   `CREATE DISKGROUP command`(#9781430260103_Ch04.xhtml#cXXX.99)
-   闪存磁盘组
-   高冗余
-   标准冗余
-   sid
-   储能模块 (ESM)
-   Exadata
-   提示
-   `KEEP setting`(#9781430260103_Ch04.xhtml#cXXX.89)
-   日志
-   监控
-   数据库服务器工具
-   存储服务器工具（*参见* 存储服务器工具）
-   `NONE setting`(#9781430260103_Ch04.xhtml#cXXX.90)
-   非 Exadata 系统
-   非易失性存储器
-   每单元缓存机制
-   Sun 闪存 PCIe 卡



### 存储索引

### 反索引

*对比* B 树索引

### 单元维护

累积指标

跳过数据

定义

误报

I/O 节省

数据聚类

数据分布

### Exadata 报告

分区

模式所有者

表大小

### 执行计划

索引段

关键数据位置

内存驻留结构

NULL 搜索

NULL 值

重建

智能扫描

storage_index_ex.sql 脚本

storage_index_part_ex.sql 脚本

### 存储重配置过程

#### 归档

归档的文件和目录

#### ASM

磁盘组

错误

大容量磁盘

#### ASR

校准过程

单元警报

#### Cellinit.ora 文件

cpio 实用程序

#### 清洁系统 — 一

deploy11203.sh 脚本

#### 破坏性过程

Exadata 配置

Exadata 系统

#### GRID 基础设施

GRID 软件

InfiniBand 连接

新的逻辑存储分区

#### OEM 代理

Oracle 内核

“oracle” 操作系统用户主目录

原始存储配置

无密码 ssh 连接

待处理和已完成的任务

#### 恢复存储

重置 “oracle” 操作系统密码

SCAN 监听器

脚本和配置文件

服务器名称

存储阵列参数

tar 命令

V$ASM_DISKGROUP

### 存储服务器工具

#### CELL_FLASH_CACHE

排除指标

LIST FLASHCACHECONTENT 命令

LIST METRICCURRENT 语句

日志文件和错误文件

指标和描述

object_id 值

输出脚本

查询编写

SYS_OP_BLOOM_FILTER 函数

![image](img/square1.jpg)  T, U

TABLE ACCESS BY INDEX ROWID

TABLE ACCESS STORAGE FULL

tableSpaceNumber 属性

TALENT_CD

耗时的试错过程

![image](img/square1.jpg)  V

V$MYSTAT 查询

V$SQLFN_METADATA 视图

V$SQL_MONITOR 视图

![image](img/square1.jpg)  W, X, Y, Z

等待事件。*另请参见* 单元等待事件
