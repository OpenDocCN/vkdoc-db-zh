# B

## 后台等待类型
后台等待类型包括 `CHECKPOINT_QUEUE`、`DIRTY_PAGE_POLL` 等待类型（与内部进程相关）、`LAZYWRITER_SLEEP` 等待类型（与杂项相关）、`MSQL_XP` 等待类型、`OLEDB` 等待类型、`TRACEWRITE` 等待类型、`WAITFOR` 等待类型。

## BACKUPBUFFER 等待类型
此等待类型提供额外信息。涉及缓冲区缓存、缓冲区计数/大小参数、代码生成、数据库备份定义、GalacticWorks 数据库内部机制、降低查询、`MAXTRANSFERSIZE` 选项、读取器和写入器缓冲区、`sys.dm_os_wait_stats` 动态管理视图 (DMV)。

## BACKUPIO 等待类型
此等待类型与数据库选项、相同任务、内部机制修改、结果、存储子系统/网络位置、条带化备份文件相关。

## 备份相关等待类型
备份相关等待类型包括 `BACKUPBUFFER`、`BACKUPIO`、备份操作、备份策略、`BACKUPTHREAD`（性能下降）、恢复时间目标/恢复点目标 (RTO/RPO) 要求、`BACKUPTHREAD` 等待类型。涉及 AdventureWorks 数据库、`BufferCount/MaxTransferSize` 选项、数据库数据文件、即时文件初始化、关系、还原操作、`sys.dm_os_waiting_tasks` 动态管理视图 (DMV)。

## 基线
基线的好处包括大型测量、比较、`CXPACKET` 等待类型定义、磁盘读取延迟、金融交易、收集信息、影响流程、深入分析、间隔、迭代过程、测量、测量拆分、指标表示、`PAGEIOLATCH_SH`、性能分析流程图、性能分析过程、性能问题、统计方法、`sys.dm_os_waiting_tasks` 类型/统计、可视化、等待统计分析（优缺点）。

