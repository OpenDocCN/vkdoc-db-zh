# 索引

![images](img/square1.jpg)  A, B

自适应游标共享

绑定感知游标（*参见* 绑定敏感）

绑定变量窥探

绑定敏感

`acs_query.sql`

绑定变量

`DIMENSION`，执行计划

使用 `SQLTXTRACT` 检查

执行计划

`IS_BIND_AWARE`

`IS_BIND_SENSITIVE`

观察项

窥探到的和捕获的绑定变量

执行计划摘要与估算

脚本输出

ACS 章节

设置游标

绑定变量

`CURSOR_SHARING` 参数

共享章节

`SQLTXTRACT` 章节

错误的性能表现

![images](img/square1.jpg)  C

基数反馈

与动态采样

相同的双胞胎

`DDL`

执行计划

良好系统，表统计信息

高值，基数

元数据

低值，基数

`SQLTXECUTE` 报告

表对象

表统计信息，不良系统

用法描述

执行计划

对用法的需求

观察项

工作原理

`dbms_xplan.display_cursor`

启用反馈

估算

测试表

`CBO` 参数

配置参数

基于成本的优化器

变更历史，哈希值

列统计信息

示例

直方图信息

估算值

作为线索

`Exec Ord`

执行计划

上下部分，章节

`SQLT XECUTE`

过时的统计信息

提示

数据操作语言

`MERGE JOIN CARTESIAN`

操作

`SQLT` 报告

`use_nl`



`神秘变化`
`CBO 章节`
`创建时间`
`执行计划`
`哈希连接`
`对象信息`
`optimizer_index_cost_adj`
`PRODUCTION_PROD_STATUS_BIX`
`神秘变化 SQL 文本`
`超出范围`
`参数`
`_b_tree_bitmap_plans`
`环境`
`非标准设置`
`optimizer_dynamic_sampling`
`Siebel 环境`
`CRM 环境`
`调优`
`系统统计信息`
`基准值与合成值`
`CBO 章节`
`DBMS_STATS.IMPORTS_SYSTEM_STATS`
`GATHER_SYSTEM_STATS`
`HTML 报告`
`信息章节`
`间隔参数`
`MREADTIM`
`SQLT 报告`
`SREADTIM`
`SYSTEM_STATICS`
`工作负载`
![图片](img/square1.jpg)  D
`Data Guard`
`物理备用数据库`
`归档日志`
`灾难恢复技术`
`灾难区域`
`联机事务处理（OLTP）`
`roxtract 工具`
`流程图`
`HTML ROXTRACT`
`左侧，计划统计信息`
`输出`
`报告，输出`
`表示，流程图`
`执行右侧`
`模式`
`用法`
`zip 文件目录`
`SQLTXTRSBY`
`数据库链接`
`环境章节`
`错误信息`
`执行计划`
`只读数据库`
`备用库上的 SQL`
`SQLT 仓库`
`SQLTXTRACT 限制`
`TO_STANDBY`
`跟踪收集`
`XTRSBY 视图`
`在 Data Guard 上工作`
`XTRACT`
`XTRSBY 报告`
`使用 XTRSBY`
`zip 文件`



动态采样

控制

转储文件

操作

参数

查询文本

字符串采样

会话级

系统级

user_dump_dest 位置

定义

执行计划

优化器进程

optimizer_dynamic_sampling

CBO 环境

非默认值集

以及并行语句

SQLTXPLAIN 报告

待设置的值

![images](img/square1.jpg)  E

## 执行计划

比较

缓冲区

基数

收集主方法存储库

CPU 时间

数据库创建

数据导入，存储库

直接写入

磁盘读取

耗时

环境部分

提取

第一个数据库

`GV$SQLAREA_PLAN_HASH`

HTML 比较报告

ID 检查

导入，`SQLTCOMPARE`

失效

加载时间

优化器环境

来自不同系统的计划

计划选择

准备，主方法存储库

处理的行数

readme 文件

读取，`SQLTCOMPARE` 报告

报告

第二个系统

SQL ID

SQL 计划

`SQLTCOMPARE`，运行

`SQLTXPLAIN` 模式

源或来源

摘要，计划

目标数据库

时间戳

总执行时间

用户 I/O

版本计数

`SQLTCOMPARE`

SQLT 存储库

![images](img/square1.jpg)  F, G

## 强制执行计划

识别 SQL 概要文件

HTML 报告

手动运行与计划




## SQL Profile 相关术语

- 计划信息列
- SQLT XECUTE
- 跨数据库传输配置文件
- 暂存区
- 步骤
- 工作负载
- SQL 配置文件
- 配置文件生成
- 查询块名称
- SQL 语句调优
- 次优行为
- 超级提示
- SQLT-SQL 配置文件
- 执行计划
- 配置文件 ID
- 计划哈希值 (PHV)
- 脚本
- `sqltprofile.sql`
- SQLT XTRACT 方法
- `utl` 目录
- 使用 SQL 配置文件
- 应急选项
- `EXEC` 脚本
- `force_match` 标志
- `LOB`
- 参数
- 脚本编写
- 解决方案
- `START` 脚本
- 文本签名
- 写入追加 (`wa`) 过程
- 错误执行计划

![images](img/square1.jpg)  H, I, J, K, L, M, N

## 隐藏参数

- 隐藏参数
- `_and_pruning_enabled`
- `_bloom_filters_enabled`
- `_complex_view_merging`
- `_disable_function_based_index`
- `_hash_join_enabled`
- `_optimizer_cartesian_enabled`
- `_optimizer_cost_model`
- `_optimizer_extended_cursor_sharing_rel`
- `_optimizer_ignore_hints`
- `_optimizer_max_permutations`
- `_optimizer_search_limit`
- `_optimizer_use_feedback`

![images](img/square1.jpg)  O, P

### 对象统计信息

- 对象统计信息
- 组成部分
- 定义
- 收集
- `AWR`
- `SQL`，来自 `OEM`
- `SQLT` 报告
- `SQL 调优 advisor`
- `SQLT XTRACT`
- 午夜截断
- 笛卡尔连接
- 列扩展
- 列统计信息，行数
- `Estim card` 列
- 执行计划
- 信息修改
- 插入、删除和更新表
- 参数设置




