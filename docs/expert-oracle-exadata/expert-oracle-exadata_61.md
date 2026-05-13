# ExaWatcher 数据存储与配置

ExaWatcher 守护进程将其收集的数据存储在 `/opt/oracle.ExaWatcher/archive` 目录中。它不使用任何特殊格式存储数据；只是存储所运行标准 OS 工具的文本输出。这使得使用常规文本处理工具（如 `grep`、AWK 或 Perl/python 脚本）来提取和展示所需信息变得容易。

每个数据收集命令都有自己的目录，每个目录包含该命令输出的存档文件：

```
# ls -l /opt/oracle.ExaWatcher/archive/

total 616

drwxr-xr-x 2 root root 45056 Jan 19 03:04 CellSrvStat.ExaWatcher

drwxr-xr-x 2 root root 40960 Jan 19 03:07 Diskinfo.ExaWatcher

drwxr-xr-x 3 root root  4096 Jan 17 02:11 ExtractedResults

drwxr-xr-x 2 root root 45056 Jan 19 03:08 FlashSpace.ExaWatcher

drwxr-xr-x 2 root root 40960 Jan 19 03:03 IBCardInfo.ExaWatcher

drwxr-xr-x 2 root root 32768 Jan 19 03:02 IBprocs.ExaWatcher

drwxr-xr-x 2 root root 40960 Jan 19 03:03 Iostat.ExaWatcher

drwxr-xr-x 2 root root 40960 Jan 19 03:06 Lsof.ExaWatcher

drwxr-xr-x 2 root root  4096 Jan 18 04:02 MegaRaidFW.ExaWatcher

drwxr-xr-x 2 root root 45056 Jan 19 03:04 Meminfo.ExaWatcher

drwxr-xr-x 2 root root 45056 Jan 19 03:02 Mpstat.ExaWatcher

drwxr-xr-x 2 root root 45056 Jan 19 02:15 Netstat.ExaWatcher

drwxr-xr-x 2 root root 36864 Jan 19 03:14 Ps.ExaWatcher

drwxr-xr-x 2 root root 40960 Jan 19 03:04 RDSinfo.ExaWatcher

drwxr-xr-x 2 root root 45056 Jan 19 03:08 Slabinfo.ExaWatcher

drwxr-xr-x 2 root root 36864 Jan 19 03:09 Top.ExaWatcher

drwxr-xr-x 2 root root 40960 Jan 19 03:04 Vmstat.ExaWatcher
```

切换到 `IOstat` 监控目录 (`IOstat.ExaWatcher`)，可以看到捕获的数据在时间间隔后使用 bzip2 工具进行了归档：

```
# ls -tr  | tail

2015_01_18_18_02_56_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_18_19_02_58_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_18_20_03_00_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_18_21_03_02_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_18_22_03_04_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_18_23_03_06_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_19_00_03_08_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_19_01_03_10_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_19_02_03_11_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2

2015_01_19_03_03_13_IostatExaWatcher_enkx3cel01.enkitec.com.dat
```

每个小时都有一个单独的压缩文件被保存，这便于手动调查过去的数据，或编写一个简单的 AWK、`grep` 或 Perl 脚本，仅提取感兴趣的数据。配置文件 `ExaWatcher.conf` 位于 `/opt/oracle.ExaWatcher` 目录下，因此您可以更改或检查每个数据收集命令的采集间隔。在下面的输出中，每组数据收集命令在 ExaWatcher 压缩当前数据收集文件并生成新文件之前，其数据总量达到 3,600 秒（`Interval` x `Count`）：

```
# cat ExaWatcher.conf | sed -e '/^\#/d' -e '/^$/d'

<ResultDir> /opt/oracle.ExaWatcher/archive

<ZipOption> bzip2

<SpaceLimit> 600

<Group>

<Start> 01/23/2015 18:25:03

<End> 01/22/2025 18:24:43

<Interval:s> 3

<Count> 1200

<CommandMode> SELECTED

<Command> Diskinfo

<Group>

<Start> 01/23/2015 18:25:03

<End> 01/22/2025 18:24:43

<Interval:s> 5

<Count> 720

<CommandMode> ALL

<Command> Iostat;;"/usr/bin/iostat -t -x"

<Command> IBprocs

<Command> Top;;"/usr/bin/top -b"

<Command> Vmstat;;"/usr/bin/vmstat"

<Command> Ps;;"/opt/oracle.ExaWatcher/FlexIntervalMode.sh '/bin/ps -eo flags,s,ruser,pid,ppid,c,psr,pri,ni,addr,sz,wchan,stime,tty,time,cmd'"

<Command> Netstat;;"/opt/oracle.ExaWatcher/FlexIntervalMode.sh '/opt/oracle.ExaWatcher/NetstatExaWatcher.sh'"

<Command> RDSinfo

<Command> Mpstat;;"/usr/bin/mpstat -P ALL"

<Command> Lsof

<Command> IBCardInfo

<Command> Meminfo

<Command> Slabinfo

<RunEnd>
```


