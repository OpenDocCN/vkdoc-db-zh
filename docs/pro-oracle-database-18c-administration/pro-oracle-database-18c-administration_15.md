# 术语表

## 状态检查与基本操作

*   状态检查
*   取消配置
*   掉线
*   Exadata Express
*   历史视图
*   初始化参数
*   监听器
*   锁定配置文件
*   重命名
*   运输容器
*   SHOW 命令
*   空间限制
*   启动/停止
*   可插拔数据库
*   根容器
*   术语
*   调优

## 进程与数据结构

*   进程监视器 (`PMON`)
*   主键约束
*   进程标识符 (`PIDs`)
*   程序全局区 (`PGA`)
*   `ps 命令`
*   `ps 实用程序`
*   `PURGE_LOG` 过程
*   `PURGE_MVIEW_FROM_LOG` 过程
*   `pwd` 命令

## R 与数据类型

*   R
*   `RAW` 数据类型
*   `README.txt`
*   只读表

## 恢复命令

*   `RECOVER` 命令
*   `RECOVER DATABASE` 命令
*   `RECOVER DATABASE UNTIL` 命令
*   `RECOVER DATAFILE` 命令
*   `RECOVER TABLE` 命令
*   `RECOVER TABLESPACE` 命令
*   `RECOVER TABLESPACE UNTIL` 命令
*   恢复管理器 (`RMAN`)
*   `RECYCLEBIN` 功能
*   引用分区类型
*   引用完整性约束，*参见* 外键
*   `REMAP_SCHEMA` 参数
*   远程物化视图刷新

## RMAN 备份

*   架构
*   基表信息
*   中央物化视图日志
*   主数据库
*   设置

### 其他命令与特性

*   `RENAME` 语句
*   `REPAIR FAILURE` 命令
*   `REPORT` 命令
*   `REPORT SCHEMA` 命令
*   `RMAN REPORT SCHEMA` 命令
*   `RESETLOGS`
*   `RESTORE` 命令
*   `RESTORE CONTROLFILE` 命令
*   `RESTORE DATABASE` 命令
*   `RESTORE DATABASE UNTIL` 命令
*   `RESTORE DATAFILE` 命令
*   `RESTORE...PREVIEW` 命令
*   `RESTORE TABLESPACE` 命令
*   `RESTORE...VALIDATE` 命令
*   `RESTORE...VALIDATE HEADER` 命令
*   反向键索引
*   `REVOKE` 语句

## RMAN 备份详解

### 配置与元数据

*   `$RMAN_BACKUP_JOB_DETAILS`
*   命令初始化
*   `ECHO` 命令
*   `NLS_DATE_FORMAT`
*   `SHOW ALL` 命令

### 备份验证

*   损坏
*   `BACKUP...VALIDATE` 命令
*   `RESTORE...VALIDATE` 命令
*   `VALIDATE` 命令

### 执行与设置

*   执行
*   归档重做日志
*   自动备份
*   备份集 *与* 映像副本
*   控制文件
*   数据文件
*   数据文件（未备份）
*   `EXCLUDE` 命令
*   `FRA`
*   完全备份 *与* 增量级别=0
*   并行度
*   储存库
*   跳过离线或不可访问的文件
*   `SKIP READONLY` 命令
*   `spfile`
*   表空间

### 增量备份特性

*   增量备份特性
*   块变更跟踪
*   增量级别
*   更新备份

### 输出与脚本

*   输出
*   `cron`
*   数据字典
*   Linux/Unix
*   shell 脚本 (`rmanback.bsh`)
*   `SPOOL LOG` 命令
*   可插拔数据库
*   根容器
*   `SYSDBA`/`SYSBACKUP` 权限

### 恢复目录

*   恢复目录
*   备份
*   创建
*   `DROP CATALOG` 命令
*   同步
*   注册目标数据库
*   版本

### 报告

*   报告
*   `LIST`
*   `REPORT` 命令
*   `SQL`
*   类型
*   `RMAN CATALOG` 命令

## RMAN 配置

### 架构组件

*   架构组件
*   辅助数据库
*   备份
*   备份片文件
*   备份集
*   通道
*   `DBA`
*   `FRA`
*   映像副本
*   媒体管理器
*   内存缓冲区 (`PGA` 或 `SGA`)
*   Oracle 服务器进程
*   `PL/SQL` 包
*   恢复目录
*   `RMAN` 客户端
*   快照控制文件
*   目标数据库

### 架构决策

*   架构决策
*   归档重做日志目的地和文件格式
*   归档重做日志的删除策略
*   备份
*   `BACKUP...FORMAT`
*   备份保留策略 (*参见* 备份保留策略)
*   备份集/映像副本
*   备份用户规范
*   二进制压缩
*   块变更跟踪
*   `CONFIGURE` 命令
*   `CONFIGURE CHANNEL...FORMAT`
*   控制文件自动备份
*   `CONTROL_FILE_RECORD_KEEP_TIME` 初始化参数
*   `cron` 实用程序
*   `CROSSCHECK` 命令
*   默认位置
*   `DELETE NOPROMPT OBSOLETE`
*   `ECHO` 参数
*   加密算法
*   `FRA`
*   实现
*   增量备份
*   增量更新备份
*   信息性输出
*   媒体管理器
*   其他设置
*   `NLS_DATE_FORMAT` OS 变量
*   在线/离线备份
*   `oraset` 变量
*   并行度
*   恢复目录
*   远程/本地
*   快照控制文件
*   指定位置
*   OS 变量
*   `PATH` 变量
*   `rman` 和 `sqlplus`
*   `SQL` 语句
*   类型

### 备份类型

*   归档日志备份
*   块变更跟踪
*   完全备份
*   增量级别 0 备份
*   增量级别 1 备份
*   增量更新备份

## RMAN 复制数据库

*   RMAN 复制数据库

## RMAN 恢复与数据恢复顾问

### RMAN 恢复与恢复

*   完全恢复 (*参见* 完全恢复)
*   控制文件
*   自动备份
*   恢复目录
*   特定备份片文件

### 数据恢复顾问

*   数据恢复顾问
*   更改故障
*   列出故障
*   修复故障
*   建议纠正措施
*   `DBA`
*   闪回表功能
*   回收站功能
*   `RESTORE POINT`
*   `SCN`
*   `SHOW RECYCLEBIN` 语句
*   `TIMESTAMP`
*   撤销表空间

### 不完全恢复

*   不完全恢复
*   归档重做日志文件
*   Data Pump 转储文件
*   `DBPITR`
*   基于日志序列的恢复
*   `RESTORE DATABASE UNTIL` 命令
*   还原点
*   基于 `SCN` 的恢复
*   `SQL` 查询
*   步骤
*   基于时间的恢复
*   `TSPITR`
*   类型

### 介质恢复与服务器

*   介质恢复
*   `RECOVER` 命令
*   重做日志文件
*   默认位置
*   非默认位置
*   `RESTORE` 命令
*   服务器
*   构建块
*   控制文件
*   控制文件感知
*   副本
*   数据库，来源
*   数据库恢复
*   数据库重命名



## Oracle 数据库关键术语与概念

### Oracle 核心概念与组件

#### 数据文件与控制文件
- `data file`、`control file`和`dump/trace file`
- `data file location`
- `init.ora file creation`
- `online redo logs`
- `Oracle binaries install`
- `OS variable`
- `temp file`
- `spfile`
- `SQL*Plus`
- `Row identifier (ROWID)`
- `data type`
- `ROWNUM pseudocolumn`
- `spfile`
    - `parameter`
    - `scenario`
- `SYSAUX tablespace`
- `SYS_CONTEXT function`
- `SYSDBA role`
- `SYS` vs. `SYSTEM`
- `SYSOPER role`
- `SYSTEM schema`
- `SYSTEM tablespace`
- `System change number (SCN)`
- `System global area (SGA)`
- `system partition type`
- `SYSTEM schema`

#### 实例与连接模式
- `mount mode`
- `nomount mode`
- `service command`
- `Service name (SID)`
- `network connection`
- `Oracle listener`
- `Transparent network substrate (TNS)`
- `tnsping command`

#### 启动与关闭
- `steps`
- `stop/start Oracle`
- `start/stop`
- `SHUTDOWN command`
- `SHUTDOWN IMMEDIATE statement`
- `STARTUP command`
- `STARTUP NOMOUNT command`

#### 容器与模式
- `Root container`
- `administration`
- `name display`
- `SALESPDB pluggable database`
- `Schemas vs. users`
- `Users`
    - `ALTER USER command`
    - `choosing name`
    - `configuration`
    - `creation`
    - `CREATE USER SQL statement`
    - `OS authentication`
    - `schemas`
    - `DBA-created accounts`
    - `DBA role`
    - `DBA_USERS_WITH_DEFPWD view`
    - `default`
    - `default permanent tablespace and temporary tablespace`
    - `DEFAULT_PWD$ view`
    - `different user logging in`
    - `DROP USER statement`
    - `grouping and assigning privileges`
    - `limiting database resource usage`
    - `modifying users`
    - `object privileges`
    - `password change`
    - `password check`
    - `password security`
    - `password strength`
    - `SQL statements`
    - `SYS account`
    - `SYSDBA role`
    - `SYSOPER role`
    - `system privileges`
    - `SYSTEM schema`
- `USERS tablespace`
- `users.sql script`
- `USER_TABLES view`
- `USER_ROLE_PRIVS view`
- `USERENV namespace`

### SQL 语言与命令

#### SQL*Plus 命令
- `S`
- `Scalable sequence`
- `scp command`
- `script command`
- `SET_ATTRIBUTE procedure`
- `SET CONTAINER command`
- `SET EDITOR command`
- `SET HOMEPATH command`
- `SET NEWNAME command`
- `SET_SCHEDULER_ATTRIBUTE procedure`
- `SET SQLPROMPT command`
- `setup.exe command`
- `SHOW command`
- `SHOW ALERT command`
- `SHOW ALL command`
- `SHOW HOMES command`
- `SKIP INACCESSIBLE command`
- `SKIP OFFLINE command`
- `SKIP READONLY command`
- `SPOOL LOG command`
- `SQL`
- `SQL*Plus`
    - `drop database`
    - `Oracle database`
    - `CREATE DATABASE statement`
    - `data dictionary creation`
    - `initialization file`
    - `OFA structures`
    - `OS variables`
    - `TEMP tablespace`
    - `OS variables`
        - `LD_LIBRARY_PATH`
        - `manually intensive approach`
        - `ORACLE_HOME`
        - `ORACLE_SID`
        - `oraenv`
        - `oraset file`
        - `oratab file`
        - `PATH variable`
    - `PASSWORD command`
    - `password file`
    - `variables`
    - `SQL scripts`
    - `.bashrc file`
    - `directories creation`
        - `HOME/bin directory`
        - `HOME/scripts directory`
    - `startup file configuration`
- `SQL TEXT column`
- `STATE column`
- `Statspack`
- `Status table`
- `STOP_JOB procedure`
- `SWITCH command`
- `SWITCH_DIR variable`
- `Synonyms`
    - `creation`
    - `definition`
    - `dropping`
    - `generates synonyms`
    - `metadata`
    - `public synonyms`
    - `renaming`
    - `types`
    - `USER2’s EMP table`
- `TAIL option`
- `tar command`
- `TARGET parameter`
- `tar -tvf <tarfile name> command`
- `tbsp_chk.bsh script`
- `telnet command`
- `THRESH_GET_WORRIED variable`
- `TIMESTAMP data type`
- `TO NONE command`
- `top.sql script`
- `Troubleshooting`
    - `automating job`
    - `bottlenecks (see Bottlenecks identification)`
    - `locking issues`
        - `COMMIT/ROLLBACK`
        - `KILL command`
        - `OS command`
        - `output`
    - `space-related issue`
    - `SQL statement`
    - `open-cursor issues`
    - `oracle installation issues`
    - `OS Watcher tool`
    - `SQL statement`
        - `ADDM report`
        - `ASH report`
        - `AWR report`
        - `diagnosing database`
        - `monitoring real-time statistics`
        - `Oracle performance utility`
        - `Statspack`
        - `V$SYSSTAT and V$SESSTAT views`
    - `temporary tablespace issues`
        - `determination`
        - `SQL view`
        - `triaging`
            - `ADRCI utility`
            - `alert log and trace files`
            - `database availability`
            - `OS tools`
            - `removing files`
    - `undo tablespace issues`
        - `determination`
        - `SQL view`
- `TRUNCATE statement`
- `unalias command`
- `UNIFORM SIZE [size] clause`
- `UNION clauses`
- `UNTIL SCN clause`
- `unzip command`
- `USE_CURRENT_SESSION parameter`
- `userdel command`
- `usermod command`
- `VALIDATE command`
- `VALIDATE DATABASE command`
- `VALIDATE HEADER clause`
- `VARCHAR2 data type`
- `view command`
- `WITHOUT VALIDATION clause`
- `xargs command`
- `xhost command`

#### SQL 语句与函数
- `ROLLBACK statement`
- `ROLLBACK statement`
- `ROLLBACK statement`
- `ROLLBACK statement`
- `SELECT statement`
- `Sequence`
    - `autoincrementing column`
    - `creation, options`
    - `definition`
    - `dropping`
    - `metadata`
    - `pseudocolumns`
    - `renaming`
    - `resetting`
    - `single/multiple sequence`
    - `unique values`
- `ROLLBACK statement`
- `ROLLBACK statement`
- `ROLLBACK statement`

#### 数据操作与表
- `Tables`
    - `character data type`
        - `CHAR`
        - `NVARCHAR2 and NCHAR`
        - `VARCHAR2`
    - `constraints (see Constraints)`
    - `creation`
        - `autoincrementing (identity) column`
        - `compressing table data`
        - `default parallel SQL execution`
        - `deferred segment creation`
        - `factors`
        - `from a query`
        - `guidelines`
        - `heap-organized table`
        - `invisible columns`
        - `IOTs`
        - `read-only tables`
        - `redo creation, avoid`
        - `temporary table`
        - `virtual columns`
    - `Date/Time data type`
    - `definition`
    - `displaying table DDL`
    - `dropping`
    - `LOB data type`
    - `modification`
        - `adding column`
        - `altering column`
        - `dropping unused column`
        - `locking`
        - `renaming column`
        - `renaming table`
    - `numeric data type`
    - `RAW data type`
    - `removing data from table`
    - `restoring dropped table`
    - `ROWID data type`
    - `types`
    - `viewing and adjusting high-water mark`
        - `lowering method`
        - `performance-related issues`
        - `readjusting`
        - `rebuilding table`
        - `selecting from data dictionary extents view`
        - `space detecting methods`
- `Tablespace point-in-time recovery (TSPITR)`
- `Tablespaces`
    - `ampersand variables`
    - `APP user`
    - `best practices for creating and managing`
    - `bigfile features`
    - `CREATE TABLESPACE statement`
    - `database space usage`
    - `data dictionary views`
    - `default table compression`
    - `dropping`
    - `human resources application`
    - `inventory application`
    - `logical storage objects and physical storage`
    - `N OLOGGING clause`
    - `objects`
    - `Oracle managed files`
    - `read-only mode`
    - `read/write mode`
    - `reasons, separate tablespaces`
    - `rename`
    - `resize`
    - `Tablespaces partitions`
        - `CREATE TABLESPACE statement`
        - `data files`
        - `single tablespace`
        - `storage clauses`
- `Temporary table`
- `Temporary tablespace`
- `Temporary tablespace issues`
- `TRUNCATE statement`
- `Undo tablespace`
    - `issues`
- `Unique index`
- `Unique key constraint creation methods`
- `Unpopulated MV`
- `Views`
    - `creation`
    - `definition`
    - `dropping`
    - `INSTEAD OF triggers`
    - `invisible column`
    - `read-only views`
    - `renaming`
    - `SQL query`
    - `updatable join view`
    - `updation`
    - `uses`
    - `WITH CHECK OPTION`
- `Virtual columns`
- `virtual partition type`
- `WITH CHECK OPTION`
- `WITH CHECK OPTION`
- `X, Y, Z`

#### 性能、空间与备份
- `space report`
- `SEGMENT SPACE MANAGEMENT AUTO clause`
- `SecureFile`
    - `advanced features`
    - `deduplication`
    - `degrees of compression`
    - `encryption feature`
    - `vs. BasicFiles`
    - `LOB column`
    - `space`
- `Scalable sequence`
- `Storage area network (SAN)`
- `Smoke test`
- `Storage area network (SAN)`
- `Temporary tablespace issues`
- `Undo tablespace issues`

#### 版本与工具
- `Scalable sequence`
- `Skip-scan feature`
- `Statspack`
- `VRMAN_OUTPUT view`

### 关键视图与数据字典
- `V$CONTROLFILE_RECORD_SECTION:`
- `V$DATABASE_BLOCK_CORRUPTION`
- `V$LOG_HISTORY`
- `V$SESSTAT view`
- `V$SQL_MONITOR view`
- `V$SQLSTATS view`
- `V$SYSSTAT view`
- `V$CONTROLFILE view`

### 系统命令与工具
- `runInstaller command`
- `service command`
- `setup.exe command`
- `startx command`
- `tar command`
- `telnet command`
- `tnsping command`
- `unzip command`
- `userdel command`
- `usermod command`
- `xargs command`
- `xhost command`
