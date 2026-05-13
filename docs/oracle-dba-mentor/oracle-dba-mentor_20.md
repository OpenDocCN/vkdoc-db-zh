# Oracle 数据库警报日志与实例管理

## 警报日志启动实例

```
2018-07-30T15:21:00.796201-05:00
Starting ORACLE instance (normal) (OS id: 23853)
2018-07-30T15:21:00.801613-05:00
**********************************************************************
LICENSE_MAX_SESSION = 0
LICENSE_SESSIONS_WARNING = 0
Initial number of CPU is 1
Number of processor cores in the system is 1
Number of processor sockets in the system is 1
Using LOG_ARCHIVE_DEST_1 parameter default value as /u01/app/oracle/product/12.2.0.1/dbs/arch
Autotune of undo retention is turned on.
IMODE=BR
ILAT =51
LICENSE_MAX_USERS = 0
SYS auditing is enabled
NOTE: remote asm mode is local (mode 0x1; from cluster type)
NOTE: Using default ASM root directory ASM
NOTE: Cluster configuration type = NONE [2]
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production.
ORACLE_HOME:    /u01/app/oracle/product/12.2.0.1
System name: Linux
Node name:   dbamentor.localdomain
Release:     4.1.12-103.6.1.el7uek.x86_64
Version:     #2 SMP Wed Sep 20 12:15:11 PDT 2017
Machine:     x86_64
Using parameter settings in server-side spfile /u01/app/oracle/product/12.2.0.1/dbs/spfileorcl.ora
System parameters with non-default values:
processes                = 300
nls_language             = "AMERICAN"
nls_territory            = "AMERICA"
sga_target               = 4G
control_files            = "/u01/app/oracle/oradata/orcl/control01.ctl"
control_files            = "/u01/app/oracle/oradata/orcl/control02.ctl"
db_block_size            = 8192
compatible               = "12.2.0"
undo_tablespace          = "UNDOTBS1"
remote_login_passwordfile= "EXCLUSIVE"
dispatchers              = "(PROTOCOL=TCP) (SERVICE=orclXDB)"
audit_file_dest          = "/u01/app/oracle/admin/orcl/adump"
audit_trail              = "DB"
db_name                  = "orcl"
open_cursors             = 300
pga_aggregate_target     = 773M
diagnostic_dest          = "/u01/app/oracle"
ALTER DATABASE   MOUNT
2018-07-30T15:21:04.148241-05:00
Using default pga_aggregate_limit of 2048 MB
2018-07-30T15:21:05.924180-05:00
Network throttle feature is disabled as mount time
2018-07-30T15:21:05.930822-05:00
Successful mount of redo thread 1, with mount id 1510662109
2018-07-30T15:21:05.931015-05:00
Database mounted in Exclusive Mode
Lost write protection disabled
Using STANDBY_ARCHIVE_DEST parameter default value as /u01/app/oracle/product/12.2.0.1/dbs/arch
Completed: ALTER DATABASE   MOUNT
2018-07-30T15:21:05.987184-05:00
ALTER DATABASE OPEN
2018-07-30T15:21:05.990548-05:00
Listing 12-4
Alert Log Instance Startup
```

## 启动失败示例

在下一个示例（Listing 12-5）中，我模拟了控制文件丢失的情况。当启动实例时，会抛出错误。

```
SQL> startup
ORACLE instance started.
Total System Global Area 4294967296 bytes
Fixed Size                  8628936 bytes
Variable Size             939525432 bytes
Database Buffers         3338665984 bytes
Redo Buffers                8146944 bytes
ORA-00205: error in identifying control file, check alert log for more info
Listing 12-5
Failed Startup
```

## 缺失控制文件

正如错误信息所说，我们应该检查警报日志（Alert Log）。在警报日志的末尾，我们可以看到类似 Listing 12-6 中的错误。

```
2018-07-30T15:33:25.549374-05:00
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_m000_25234.trc:
ORA-00202: control file: '/u01/app/oracle/oradata/orcl/control02.ctl'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Listing 12-6
Alert Log Missing Control File
```

错误信息明确地告诉我们哪个文件丢失了（`control02.ctl`），以及没有这个文件或目录。

## 修复控制文件

修复这个问题很容易。控制文件是复用的，意味着任何其他控制文件都是完全相同的副本。如果我们有`control01.ctl`，可以直接将其复制为丢失的文件，然后重新启动实例。Listing 12-7 展示了这个问题解决得多么容易。

```
SQL> !ls -l
total 7867508
-rw-r-----. 1 oracle oinstall 5368717312 Jul 30 15:28 apps_ts01.dbf
-rw-r-----. 1 oracle oinstall   10600448 Jul 30 15:28 control01.ctl
-rw-r-----. 1 oracle oinstall  209715712 Jul 30 15:21 redo01.log
-rw-r-----. 1 oracle oinstall  209715712 Jul 30 15:21 redo02.log
-rw-r-----. 1 oracle oinstall  209715712 Jul 30 15:28 redo03.log
-rw-r-----. 1 oracle oinstall  807411712 Jul 30 15:28 sysaux01.dbf
-rw-r-----. 1 oracle oinstall  734011392 Jul 30 15:28 system01.dbf
-rw-r-----. 1 oracle oinstall   20979712 Jul 30 00:51 temp01.dbf
-rw-r-----. 1 oracle oinstall  492838912 Jul 30 15:28 undotbs01.dbf
-rw-r-----. 1 oracle oinstall    5251072 Jul 30 15:28 users01.dbf
SQL> !cp control01.ctl control02.ctl
SQL> shutdown abort
ORACLE instance shut down.
SQL> startup
ORACLE instance started.
Total System Global Area 4294967296 bytes
Fixed Size                  8628936 bytes
Variable Size             939525432 bytes
Database Buffers         3338665984 bytes
Redo Buffers                8146944 bytes
Database mounted.
Database opened.
Listing 12-7
Fixing Our Control File
```

目录显然缺少那个控制文件，所以我简单地从另一个控制文件复制了它。然后，我终止了部分启动的实例，并成功地重新启动了它。

## 实例关闭

警报日志在实例关闭时也包含有用的信息，尽管信息量不同。看看你是否能在 Listing 12-8 的日志摘录中发现以下信息：

*   开始关闭的日期和时间
*   完成关闭的日期和时间

```
2018-07-30T15:28:43.622558-05:00
Shutting down instance (immediate) (OS id: 24764)
2018-07-30T15:28:45.143579-05:00
Stopping background process SMCO
2018-07-30T15:28:46.160809-05:00
Shutting down instance: further logons disabled
2018-07-30T15:28:46.168713-05:00
Stopping background process CJQ0
License high water mark = 1
2018-07-30T15:28:48.203074-05:00
Dispatchers and shared servers shutdown
ALTER DATABASE CLOSE NORMAL
Stopping Emon pool
Stopping Emon pool
2018-07-30T15:28:48.260284-05:00
Shutting down archive processes
2018-07-30T15:28:49.260600-05:00
Archiving is disabled
2018-07-30T15:28:49.262589-05:00
Thread 1 closed at log sequence 42
Successful close of redo thread 1
2018-07-30T15:28:49.320837-05:00
Completed: ALTER DATABASE CLOSE NORMAL
ALTER DATABASE DISMOUNT
Shutting down archive processes
Archiving is disabled
Completed: ALTER DATABASE DISMOUNT
2018-07-30T15:28:50.329124-05:00
ARCH: Archival disabled due to shutdown: 1089
Shutting down archive processes
Archiving is disabled
2018-07-30T15:28:59.607117-05:00
Instance shutdown complete (OS id: 24764)
Listing 12-8
Alert Log Instance Shutdown
```

## 实例崩溃分析

如果实例崩溃了，找出原因的最佳位置就是警报日志。DBA 应该做的第一件事是尽快启动实例并使其运行，以恢复对数据的访问。第二件事是弄清楚实例崩溃的原因。在 Listing 12-9 所示的警报日志条目中，我们可以看到实例进入了关键状态。一个名为`SMON`的后台进程意外死亡。由于`SMON`必须保持运行，另一个进程`PMON`检测到了这个状况并终止了 Oracle 实例。

