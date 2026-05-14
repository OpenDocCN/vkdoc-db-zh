# 检查数据库连接状态

18 * * * * /home/oracle/bin/conn.bsh o18c 1>/home/oracle/bin/log/conn.log 2>&1

```cron
18 * * * * /home/oracle/bin/conn.bsh o18c 1>/home/oracle/bin/log/conn.log 2>&1
```

此 `cron` 任务每小时运行一次脚本。根据您的可用性要求，您可能需要以更高的频率运行此类脚本。

## 使用以下脚本检查正在运行且空间即将满的挂载点

`filesp.bsh`

将脚本放置在 `HOME/bin` 这样的目录中。您需要修改脚本，使 `mntlist` 变量包含数据库服务器上存在的挂载点列表。因为此脚本不运行任何 Oracle 实用程序，所以无需设置 Oracle 相关的 OS 变量（与之前的 shell 脚本不同）：

```bash
#!/bin/bash
mntlist="/orahome /ora01 /ora02 /ora03"
for ml in $mntlist
do
echo $ml
usedSpc=$(df -h $ml | awk '{print $5}' | grep -v capacity | cut -d "%" -f1 -)
BOX=$(uname -a | awk '{print $2}')
#
case $usedSpc in
[0-9])
arcStat="安心，磁盘空间充足：$usedSpc"
;;
[1-7][0-9])
arcStat="磁盘空间正常：$usedSpc"
;;
[8][0-9])
arcStat="空间即将不足：$usedSpc"
echo $arcStat | mailx -s "空间状态：$BOX" dkuhn@gmail.com
;;
[9][0-9])
arcStat="警告，空间即将耗尽：$usedSpc"
echo $arcStat | mailx -s "空间状态：$BOX" dkuhn@gmail.com
;;
[1][0][0])
arcStat="请更新简历，空间已耗尽：$usedSpc"
echo $arcStat | mailx -s "空间状态：$BOX" dkuhn@gmail.com
;;
*)
arcStat="嗯？：$usedSpc"
esac
#
BOX=$(uname -a | awk '{print $2}')
echo $arcStat
#
done
#
exit 0
```

您可以从命令行手动运行此脚本，像这样：

```
$ filesp.bsh
```

以下是此数据库服务器的输出：

```
/orahome
磁盘空间正常：79
/ora01
空间即将不足：84
/ora02
磁盘空间正常：41
/ora03
安心，磁盘空间充足：9
```

这正是您应该使用 `cron` 等调度工具自动运行的脚本类型。以下是一个典型的 `cron` 条目：

```
```



