# 查看 iostat 输出示例

下一个示例从一个上午 9 点的 `bzipped` 文件中提取 `iostat` I/O 统计信息：

```
# bzcat 2015_01_17_09_02_39_IostatExaWatcher_enkx3cel01.enkitec.com.dat.bz2 | head -20
############################################################
# Starting Time:        01/17/2015 09:02:39
# Sample Interval(s):   5
# Archive Count:        720
# Collection Module:    IostatExaWatcher
# Collection Command:   /usr/bin/iostat -t -x  5  720
############################################################
zzz <01/17/2015 09:02:39> Count:720
Linux 2.6.39-400.128.17.el5uek (enkx3cel01.enkitec.com)          01/17/2015

Time: 09:02:39 AM

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          0.54    0.00    0.89    0.22    0.00   98.34

Device:          rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda              0.48    17.27    1.52    5.31  1074.19   181.77   183.93     0.03    3.97   0.95   0.65
sda1             0.00     0.00    0.00    0.00     0.44     0.00    93.97     0.00    4.47   0.38   0.00
sda2             0.00     0.00    0.00    0.00     0.00     0.00     2.55     0.00    0.77   0.76   0.00
sda3             0.47     0.00    0.47    0.45   956.01     4.74  1038.28     0.01    5.69   5.67   0.52
sda4             0.00     0.00    0.00    0.00     0.00     0.00     1.79     0.00    1.16   1.16   0.00
```

