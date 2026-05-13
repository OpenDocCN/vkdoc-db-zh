# 性能调优与诊断

## SQL 语句管理
*   删除语句，394–395
*   显示内容，393–394
*   传输到另一个数据库，396–398
*   调优建议，372

## `DBMS_AUTO_SQLTUNE` 包
*   `DBMS_AUTO_SQLTUNE` 包，372
*   `DBMS_AUTO_SQLTUNE.SET_AUTO_TUNING_TASK` 参数，377–379
*   详细信息部分（detail section），374–375
*   通过电子邮件发送输出（e-mailing output），373–374
*   错误部分（error section），375
*   发现部分（findings section），375
*   常规信息部分（general information section），374
*   `REPORT_AUTO_TUNING_TASK` 函数，376
*   `SCRIPT_TUNING_TASK` 函数，377
*   摘要部分（summary section），374

## 自动数据库诊断监视器 (ADDM)
*   自动数据库诊断监视器 (ADDM)，92，368
*   `DBMS_ADDM` 包，406
*   企业管理器（enterprise manager），406–407
*   性能建议（performance recommendations），407
*   SQL*Plus 脚本（SQL*plus script），405

## 自动诊断存储库命令解释器 (ADRCI)
*   自动诊断存储库命令解释器 (ADRCI)，229
*   ADR 基目录（ADR base），230
*   警报日志（alert log），233–234
*   批处理模式（in batch mode），230
*   诊断任务（diagnostic tasks），231
*   `HELP` 命令，230
*   `homepath` 命令，231，232
*   交互模式（in interactive mode），229
*   `V$DIAG_INFO` 视图，231
*   查看事件（view incidents），235–236

## 自动内存管理
*   自动内存管理（Automatic memory management），83
*   缓冲区池（buffer pool），87
*   `KEEP`，87，88
*   `RECYCLE`，87，88

## 客户端结果集缓存
*   缓存客户端结果集（caching client result sets），103
*   优点（advantages），104
*   `CLIENT_RESULT_CACHE_LAG`，103
*   `CLIENT_RESULT_CACHE_SIZE`，103
*   `OCIStmtExecute()`，105
*   `OCIStmtFetch()`，105
*   可选客户端配置文件（optional client-side configuration file），104

## PL/SQL 函数结果缓存
*   缓存 PL/SQL 函数（caching PL/SQL function），105
*   考虑因素（considerations），108
*   限制（restrictions），108

## SQL 查询结果缓存
*   缓存 SQL 查询结果（caching SQL query result），99
*   读一致性要求（read consistency requirements），102–103
*   `RESULT_CACHE_MODE` 初始化参数，100
*   表注释和查询提示（table annotations and query hints），101–102

## 配置服务器查询缓存
*   `DBMS_RESULT_CACHE.FLUSH` 过程，96，97
*   初始化参数（initialization parameters），95–96
*   物化视图（materialized views），96
*   PL/SQL 集合（PL/SQL collection），96
*   `RESULT_CACHE_MAX_RESULT`，96
*   `RESULT_CACHE_MAX_SIZE`，96
*   `RESULT_CACHE_REMOTE_EXPIRATION`，97

## 自动内存管理配置
*   `DBCA`，85
*   实施步骤（implementing steps），83–84

## 管理服务器结果缓存
*   管理服务器结果缓存（managing server result cache），97
*   `DBMS_RESULT_CACHE.STATUS()`，97–99
*   共享池百分比（shared pool percentage），99

## 内存调整操作
*   `V$MEMORY_RESIZE_OPS`，91
*   `V$MEMORY_TARGET_ADVICE`，90


