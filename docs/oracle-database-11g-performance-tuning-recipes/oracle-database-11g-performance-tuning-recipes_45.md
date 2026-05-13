# Oracle 数据库性能与内存管理索引

PGA, 83–87

SGA, 83–87

`MEMORY_MAX_TARGET` 参数, 86, 87

`MEMORY_TARGET` 参数, 84, 85

优化内存使用, 91

ADDM 报告, 92

数据库中的内存大小建议图, 91

调整步骤, 91

Oracle Database 11g, 85

Oracle 数据库智能闪存缓存, 109–110

`DB_FLASH_CACHE_FILE`, 109

`DB_FLASH_CACHE_SIZE`, 109

### PGA 内存分配

AWR, 95

`PGA_AGGREGATE_TARGET` 参数, 93, 94

步骤, 93

`V$SQL_WORKAREA_HISTOGRAM`, 94

`V$SYSSTAT` 和 `V$SESSTAT`, 95

`pga_memory_target`, 84

### 重做日志缓冲区调整, 110–112

`SCOPE` 参数, 84

设置最小值, 89

`自动段空间管理 (ASSM)`, 5

`自动工作负载仓库 (AWR)`, 21, 95, 113

活动会话信息。*参见* 活动历史会话信息

### 基线统计信息

自适应指标, 125

`AWR_REPORT_TEXT` 函数, 131

`awrextr.sql` 脚本, 130

`awrload.sql` 脚本, 130

`awrrpt.sql` 脚本, 130

`CREATE_BASELINE_TEMPLATE` 过程, 131–132

`DBA_HIST_BASELINE_TEMPLATE` 视图, 132

`DROP_BASELINE` 过程, 129

`DROP_BASELINE_TEMPLATE` 过程, 132

固定基线, 123

移动基线, 124–126

性能统计信息, 123

`RENAME_BASELINE` 过程, 129

保留期, 124–126

模板, 131–132

通过企业管理器, 126–128

类别, 115

历史数据库性能统计信息, 113

间隔期和保留期, 116–117

基于间隔的历史统计信息, 114

输出, 133–134

### 报告生成

`awrrpt.sql` 脚本, 117, 120

数据字典, 119

数据库实例, 119

报告名称, 119

报告类型, 118

单个 SQL 语句, 121–122

快照 ID, 119

通过企业管理器, 120–121

快照, 134, 324

统计组件, 114–115

`STATISTICS_LEVEL` 参数, 114

时间范围, 113

信息类型, 115

`UTLBSTAT/UTLESTAT` 和 `Statspack`, 113

`AUTOSTATS_TARGET` 参数, 456

## ![images](img/square.jpg) B

`位图连接索引`, 73

### 瓶颈

CPU 和内存 (ps), 197–198

`iostat` 命令, 198

AWR, 200

列描述, 199

检查输出, 198

`Statspack`, 200

`V$` 视图, 200

网络密集型进程, 201–202

Solaris 系统, 192

列描述, 193, 194

`prstat` 实用程序, 193

`vmstat`, 190

解读输出, 191

输出列描述, 191–192

`B-树索引`, 50

`DBMS_SPACE` `CREATE_INDEX` 过程, 50

### 索引块

`索引快速全扫描`, 47

`索引范围扫描`, 47

Oracle 的 `Autotrace` 实用程序, 47

`INDEX RANGE SCAN`, 49

`ROWID`, 46

表块, 49–50

表布局, 46

技术方面, 45


