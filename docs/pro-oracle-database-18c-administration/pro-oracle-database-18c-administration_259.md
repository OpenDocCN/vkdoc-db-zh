# Oracle 数据库术语

## A
*   状态检查
*   拔出
*   删除
*   Exadata Express
*   历史视图
*   初始化参数
*   监听器
*   锁定配置文件
*   重命名
*   运输容器（此处可能指 Oracle 数据库中的特定概念或技术上下文，通常不直译）
*   SHOW 命令
*   空间限制
*   启动/停止
*   可插拔数据库
*   根容器
*   条款
*   调优

## P
*   进程监控器 (`PMON`)
*   主键约束
*   进程标识符 (`PIDs`)
*   程序全局区 (`PGA`)
*   `ps` 命令
*   `ps` 实用程序
*   `PURGE_LOG` 过程
*   `PURGE_MVIEW_FROM_LOG` 过程
*   `pwd` 命令

## R
*   `RAW` 数据类型
*   `README.txt`
*   只读表
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
*   参照完整性约束，*参见* 外键
*   `REMAP_SCHEMA` 参数
*   远程物化视图刷新
    *   架构
    *   基表信息
    *   中心物化视图日志
    *   主数据库
    *   设置
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

## RMAN 备份
*   `$RMAN_BACKUP_JOB_DETAILS`
*   命令初始化
    *   `ECHO` 命令
    *   `NLS_DATE_FORMAT`
    *   `SHOW ALL` 命令
*   损坏
    *   `BACKUP...VALIDATE` 命令
    *   `RESTORE...VALIDATE` 命令
    *   `VALIDATE` 命令
*   执行
    *   归档重做日志
    *   自动备份
    *   备份集 vs. 镜像副本
    *   控制文件
    *   数据文件
    *   数据文件（未备份）
    *   `EXCLUDE` 命令
    *   `FRA`
    *   完全备份 vs. 增量级别=0
    *   并行度
    *   目录（存储库）
    *   跳过离线或不可访问的文件
    *   `SKIP READONLY` 命令
    *   `spfile`
    *   表空间
*   增量备份特性
    *   块变更跟踪
    *   增量级别
    *   更新备份
*   输出
    *   `cron`
    *   数据字典
    *   Linux/Unix
    *   Shell 脚本 (`rmanback.bsh`)
    *   `SPOOL LOG` 命令
*   可插拔数据库
*   根容器
*   `SYSDBA`/`SYSBACKUP` 特权

## RMAN 目录
*   恢复目录
    *   备份
    *   创建
    *   `DROP CATALOG` 命令
    *   同步
    *   目标数据库注册
    *   版本
*   报告
    *   `LIST`
    *   `REPORT` 命令
*   `SQL`
*   类型
    *   `RMAN CATALOG` 命令

## RMAN 配置
*   架构组件
    *   辅助数据库
    *   备份/进行备份
    *   备份片文件
    *   备份集
    *   通道
    *   `DBA`
    *   `FRA`
    *   镜像副本
    *   媒体管理器
    *   内存缓冲区 (`PGA` 或 `SGA`)
    *   Oracle 服务器进程
    *   `PL/SQL` 包
    *   恢复目录
    *   `RMAN` 客户端
    *   快照控制文件
    *   目标数据库
*   架构决策
    *   归档重做日志目标位置和文件格式
    *   归档重做日志的删除策略
*   进行备份
    *   `BACKUP...FORMAT`
    *   备份保留策略 (参见 备份保留策略)
    *   备份集/镜像副本
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
    *   信息输出
    *   媒体管理器
    *   杂项设置
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
    *   SQL 语句
    *   类型
*   归档日志备份
    *   块变更跟踪
    *   完全备份
    *   增量级别 0 备份
    *   增量级别 1 备份
    *   增量更新备份

## RMAN 复制数据库
*   `RMAN duplicate database`

## RMAN 恢复与还原
*   完全恢复 (参见 完全恢复)
*   控制文件
    *   自动备份
    *   恢复目录
    *   特定备份片文件
*   数据恢复顾问
    *   更改故障
    *   列出故障
    *   修复故障
    *   建议纠正措施
*   `DBA`
*   闪回表特性
*   回收站特性
*   `RESTORE POINT`
*   `SCN`
*   `SHOW RECYCLEBIN` 语句
*   `TIMESTAMP`
*   撤消表空间
*   不完全恢复
    *   归档重做日志文件
    *   `Data Pump` 转储文件
    *   `DBPITR`
    *   基于日志序列的恢复
    *   `RESTORE DATABASE UNTIL` 命令
    *   还原点
    *   基于 `SCN` 的恢复
    *   SQL 查询
    *   步骤
    *   基于时间的恢复
    *   `TSPITR`
    *   类型
*   介质恢复
    *   `RECOVER` 命令
*   重做日志文件
    *   默认位置
    *   非默认位置
*   `RESTORE` 命令
*   服务器
    *   构建块
    *   控制文件
    *   感知控制文件
    *   副本
    *   数据库，来源
    *   数据库恢复
    *   数据库重命名


## 数据文件、控制文件和转储/跟踪文件

`数据文件位置`

`init.ora 文件创建`

`装载模式`

`非装载模式`

`在线重做日志`

`Oracle 二进制文件安装`

`操作系统变量`

`临时文件`

`服务器参数文件`

`SQL*Plus`

`步骤`

`停止/启动 Oracle`

`ROLLBACK 语句`

`根容器`

`管理`

`名称显示`

`网络连接`

`空间报告`

`启动/停止`

`行标识符 (ROWID)`

`数据类型`

`ROWNUM 伪列`

`runInstaller 命令`

## S

`SALESPDB 可插拔数据库`

`可扩展序列`

模式与用户

`scp 命令`

`script 命令`

`安全文件`

`高级特性`

`去重`

`压缩级别`

`加密特性`

对比 BasicFiles

`LOB 列`

`空间`

`SEGMENT SPACE MANAGEMENT AUTO 子句`

`SELECT 语句`

`序列`

`自增列`

`创建，选项`

`定义`

`删除`

`元数据`

`伪列`

`重命名`

`重置`

`单/多重序列`

`唯一值`

`服务命令`

`服务名 (SID)`

`SET_ATTRIBUTE 过程`

`SET CONTAINER 命令`

`SET EDITOR 命令`

`SET HOMEPATH 命令`

`SET NEWNAME 命令`

`SET_SCHEDULER_ATTRIBUTE 过程`

`SET SQLPROMPT 命令`

`setup.exe 命令`

`SHOW 命令`

`SHOW ALERT 命令`

`SHOW ALL 命令`

`SHOW HOMES 命令`

`SHUTDOWN 命令`

`SHUTDOWN IMMEDIATE 语句`

`SKIP INACCESSIBLE 命令`

`SKIP OFFLINE 命令`

`SKIP READONLY 命令`

`跳过扫描特性`

`冒烟测试`

`服务器参数文件`

`参数`

`场景`

`SPOOL LOG 命令`

`SQL`

`SQL*Plus`

`删除数据库`

`Oracle 数据库`

`CREATE DATABASE 语句`

`数据字典创建`

`初始化文件`

`OFA 结构`

`操作系统变量`

`TEMP 表空间`

`Oracle 监听器`

`操作系统变量`

`LD_LIBRARY_PATH`

`手动密集型方法`

`ORACLE_HOME`

`ORACLE_SID`

`oraenv`

`oraset 文件`

`oratab 文件`

`PATH 变量`

`PASSWORD 命令`

`密码文件`

`变量`

`SQL 脚本`

`.bashrc 文件`

`目录创建`

`HOME/bin 目录`

`HOME/scripts 目录`

`启动文件配置`

`SQL TEXT 列`

`STARTUP 命令`

`STARTUP NOMOUNT 命令`

`START WITH 子句`

`startx 命令`

`STATE 列`

`Statspack`

`状态表`

`STOP_JOB 过程`

`存储区域网络 (SAN)`

`SWITCH 命令`

`SWITCH_DIR 变量`

`同义词`

`创建`

`定义`

`删除`

`生成同义词`

`元数据`

`公共同义词`

`重命名`

`类型`

`USER2 的 EMP 表`

`SYSAUX 表空间`

`SYS_CONTEXT 函数`

`SYSDBA 角色`

SYS 与 SYSTEM

`SYSOPER 角色`

`SYSTEM 模式`

`SYSTEM 表空间`

`系统变更号 (SCN)`

`系统全局区 (SGA)`

`系统分区类型`

`SYSTEM 模式`

## T

`表`

`字符数据类型`

`CHAR`

`NVARCHAR2 和 NCHAR`

`VARCHAR2`

约束（参见约束）

`创建`

`自增（标识）列`

`压缩表数据`

`默认并行 SQL 执行`

`延迟段创建`

`因素`

`从查询创建`

`指南`

`堆组织表`

`不可见列`

`索引组织表`

`只读表`

`避免重做创建`

`临时表`

`虚拟列`

`日期/时间数据类型`

`定义`

`显示表 DDL`

`删除`

`LOB 数据类型`

`修改`

`添加列`

`修改列`

`删除未使用的列`

`锁定`

`重命名列`

`重命名表`

`数字数据类型`

`RAW 数据类型`

`从表中删除数据`

`恢复已删除的表`

`ROWID 数据类型`

`类型`

`查看和调整高水位线`

`降低方法`

`性能相关问题`

`重新调整`

`重建表`

`从数据字典扩展视图中选择`

`空间检测方法`

`表空间时间点恢复 (TSPITR)`

`表空间`

`& 变量`

`APP 用户`

`创建和管理的最佳实践`

`大文件特性`

`CREATE TABLESPACE 语句`

`数据库空间使用情况`

`数据字典视图`

`默认表压缩`

`删除`

`人力资源应用程序`

`库存应用程序`

`逻辑存储对象和物理存储`

`N OLOGGING 子句`

`对象`

`Oracle 管理的文件`

`只读模式`

`读/写模式`

`分离表空间的原因`

`重命名`

`调整大小`

`表空间分区`

`CREATE TABLESPACE 语句`

`数据文件`

`单个表空间`

`存储子句`

`TAIL 选项`

`tar 命令`

`TARGET 参数`

`tar -tvf <tar 文件名> 命令`

`tbsp_chk.bsh 脚本`

`telnet 命令`

`TEMP 表空间`

`临时表`

`临时表空间`

`临时表空间问题`

`THRESH_GET_WORRIED 变量`

`TIMESTAMP 数据类型`

`tnsping 命令`

`TO NONE 命令`

`top.sql 脚本`

`透明网络底层 (TNS)`

`故障排除`

`自动化作业`

瓶颈（参见瓶颈识别）

`锁定问题`

`COMMIT/ROLLBACK`

`KILL 命令`

`操作系统命令`

`输出`

`空间相关问题`

`SQL 语句`

`开放游标问题`

`Oracle 安装问题`

`OS Watcher 工具`

`SQL 语句`

`ADDM 报告`

`ASH 报告`

`AWR 报告`

`诊断数据库`

`监控实时统计信息`

`Oracle 性能工具`

`Statspack`

`V$SYSSTAT 和 V$SESSTAT 视图`

`临时表空间问题`

`判定`

`SQL 视图`

`初步诊断`

`ADRCI 实用程序`

`警报日志和跟踪文件`

`数据库可用性`

`操作系统工具`

`删除文件`

`UNDO 表空间问题`

`判定`

`SQL 视图`

`TRUNCATE 语句`

## U

`取消别名命令`

`UNDO 表空间`

`问题`

`UNIFORM SIZE [大小] 子句`

`UNION 子句`

`唯一索引`

`唯一键约束创建方法`

`未填充的物化视图`

`UNTIL SCN 子句`

`unzip 命令`

`USE_CURRENT_SESSION 参数`

`userdel 命令`

`USERENV 命名空间`

`用户管理的备份和恢复`

`归档日志模式数据库`

完全恢复（参见完全恢复）

不完全恢复（参见不完全恢复）

冷备份策略（参见冷备份策略）

热备份策略（参见热备份策略）

`usermod 命令`

`USER_ROLE_PRIVS 视图`

`用户`

`ALTER USER 命令`

`选择名称`

`配置`

`创建`

`CREATE USER SQL 语句`

`操作系统认证`

`模式`

`DBA 创建的账户`

`DBA 角色`

`DBA_USERS_WITH_DEFPWD 视图`

`默认`

`默认永久表空间和临时表空间`

`DEFAULT_PWD$ 视图`

`不同用户登录`

`DROP USER 语句`

`分组和分配权限`

`限制数据库资源使用`

`修改用户`

`对象权限`

`密码更改`

`密码检查`

`密码安全`

`密码强度`

`SQL 语句`

`SYS 账户`

`SYSDBA 角色`

`SYSOPER 角色`

`系统权限`

`SYSTEM 模式`

`USERS 表空间`

`users.sql 脚本`

`USER_TABLES 视图`

## V

`V$CONTROLFILE_RECORD_SECTION:`

`V$DATABASE_BLOCK_CORRUPTION`

`V$LOG_HISTORY`

`V$SESSTAT 视图`

`V$SQL_MONITOR 视图`

`V$SQLSTATS 视图`

`V$SYSSTAT 视图`

`VALIDATE 命令`

`VALIDATE DATABASE 命令`

`VALIDATE HEADER 子句`

`VARCHAR2 数据类型`

`view 命令`

`视图`

`创建`

`定义`

`删除`

`INSTEAD OF 触发器`

`不可见列`

`只读视图`

`重命名`

`SQL 查询`

`可更新连接视图`

`更新`

`用途`

`WITH CHECK OPTION`

`虚拟列`

`虚拟分区类型`

`VRMAN_OUTPUT 视图`

`V$CONTROLFILE 视图`

## W

`WITH CHECK OPTION`

`WITHOUT VALIDATION 子句`

## X, Y, Z

`xargs 命令`

`xhost 命令`