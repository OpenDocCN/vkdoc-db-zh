# PDB 特性

*   当连接到 PDB 时，您无法看到 CDB 中存在的其他 PDB。这就像您连接到一个隔离的数据库。
*   多个 PDB 在 CDB 内共享通用数据库资源（内存结构、后台进程等）。从根容器级别，您可以像管理单个数据库一样管理许多 DBA 数据库。
*   您可以关闭一个 PDB，而不会影响 CDB 内的其他 PDB。
*   可以通过克隆种子数据库、现有 PDB 或非 CDB 来快速创建新的 PDB。
*   PDB 可以创建并设置为自动刷新。
*   PDB 可以通过从一个 CDB 拔出并插入另一个 CDB 来轻松地从一个 CDB 传输到另一个 CDB。

## 功能与优势

这些特性提供了创建和管理数据库的新方法。PDB 技术的主要优点是它允许您将许多数据库整合到一个总括性数据库中。这产生了规模经济，因为可以在更少的服务器上由更少的支持人员实施和维护更多的数据库。作为 DBA，理解此技术及其实施方法使您对公司更有价值。本地配置的数据库环境在云中同样可用。了解如何管理本地数据库将适用于云中的管理。拥有数据库的 PDB 级别允许进行单独管理，而无需管理 CDB 或数据库服务器。
即使在 Oracle Cloud 的自主数据库中，DBA 仍然需要处理应用程序数据、集成和数据安全策略。当您理解数据库、数据以及如何使用它们时，总有足够的工作要做。

## 索引

## A

*   活动会话历史 (ASH)
*   `ADD LOGFILE GROUP` 语句
*   ADRCI 实用程序
*   `ADR_HOME` 目录
*   高级行压缩
*   `ADVISE FAILURE` 命令
*   `alert.log` 文件
*   `alias` 命令
*   `ALTER DATABASE BACKUP CONTROLFILE TO TRACE` 语句
*   `ALTER DATABASE` 命令
*   `ALTER DATABASE DATAFILE` 语句
*   `ALTER DATABASE DATAFILE ... OFFLINE FOR DROP`
*   `ALTER DATABASE DROP LOGFILE MEMBER` 语句
*   `ALTER DATABASE MOVE DATAFILE` 命令
*   `ALTER DATABASE RENAME FILE` 语句
*   `ALTER SEQUENCE` 语句
*   `ALTER SESSION` 命令
*   `ALTER SYSTEM KILL SESSION` 语句
*   `ALTER SYSTEM SWITCH LOGFILE` 语句
*   `ALTER TABLE` 语句
*   `ALTER TABLE ... MOVE` 语句
*   `ALTER TABLE...SHRINK SPACE` 语句
*   `ALTER TABLESPACE` 语句
*   `ALTER TABLESPACE ... ADD DATAFILE` 语句
*   `ALTER TABLESPACE ... OFFLINE IMMEDIATE`
*   `ALTER TABLESPACE ... OFFLINE NORMAL` 语句
*   `ALTER USER` 命令
*   APP_DATA_LARGE 表空间
*   APP_DATA_SMALL 表空间
*   `ARCHIVE_LAG_TARGET` 初始化参数
*   归档日志模式
*   架构决策
*   归档日志目标
    *   备份归档重做日志文件
    *   禁用
    *   启用
    *   重做文件位置
    *   FRA
    *   `LOG_ARCHIVE_DEST_N` 数据库
    *   用户定义的磁盘位置
*   归档重做日志文件
*   ASM 集群文件系统 (ACFS)
*   `AUTOALLOCATE` 子句
*   `AUTOEXTEND` 特性
*   自增（标识）列
*   自动数据库诊断监视器 (ADDM)
*   自动诊断库命令解释器 (ADRCI)
*   自动段空间管理 (ASSM)
*   自动存储管理 (ASM)
    *   磁盘组
    *   表空间
*   自动工作量存储库 (AWR)
*   自动化作业
    *   `cron` 实用程序
        *   后台进程
        *   `cron` 守护进程
        *   `crontab` 条目
        *   默认编辑器
        *   编辑`cron`表
        *   启用访问
        *   `/etc/crontab` 文件
        *   文件和目录
        *   加载
        *   重定向输出
        *   `root` 用户
        *   故障排除
        *   `/var/spool/cron` 目录
    *   DBA
        *   锁定的生产账户
        *   操作系统脚本
        *   重启/重新启动数据库
        *   重做日志目标
        *   RMAN 备份
        *   SQL*Plus 进程
        *   截断大型日志文件
    *   Linux/Unix 环境
    *   Oracle 调度程序实用程序
        *   `COPY_JOB` 过程
        *   `CREATE_JOB` 过程
        *   *vs.* `cron`
        *   删除
        *   `DISABLE` 过程
        *   `ENABLE` 过程
        *   `JOB_CLASS` 参数
        *   记录历史
        *   修改
        *   `/orahome/oracle/bin` 目录
        *   PL/SQL 包
        *   `REPEAT_INTERVAL` 参数
        *   运行
        *   停止
        *   查看详细信息
*   自治数据库
    *   Oracle 云
        *   打补丁
        *   安全配置
    *   自治数据仓库云 (ADWC)
    *   自治事务处理数据库 (ATP)
*   `autotrace` 工具

## B

*   `BACKUP` 命令



## B
## `BACKUP AS COPY` 命令
## `backup.bsh` 脚本
## 备份保留策略
## `CLEAR` 命令
## 删除过时的备份
## 恢复窗口
## 冗余
## `TO NONE` 命令
## `BACKUP...VALIDATE` 命令
## Bash Shell 反斜杠
## Bash shell 环境
## BASIC 压缩算法
### BasicFiles
## LOB 列
## SecureFile 迁移
## 与 SecureFiles 比较
### 空间
## `bdump`
## `BIGFILE` 子句
## 二进制文件 (`BFILE`)
## 二进制大对象 (`BLOB`)
## 位图索引
## 位图连接索引
## 块级别恢复
## 瓶颈识别
## 操作系统实用程序
## 数据库应用程序
## Linux/Unix 环境
## 映射，操作系统进程
## 性能与监控
## `top` 命令
## `vmstat` 实用程序
## B 树索引
## `BUILD DEFERRED` 子句

