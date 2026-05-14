# 第三章 ■ 配置高效环境

## 检查与数据库的连接

45 * * * * /orahome/bin/conn.bsh INVPRD 1>/orahome/bin/log/conn.log 2>&1

此 cron 条目每小时运行一次脚本。根据您的可用性要求，您可能希望更频繁地运行类似脚本。

### filesp.bsh

使用以下脚本检查正在填充的可用挂载点。将此脚本放置于 `HOME/bin` 等目录中。您需要修改脚本，使 `mntlist` 变量包含数据库服务器上存在的挂载点列表。由于此脚本未运行任何 Oracle 实用程序，因此无需设置 Oracle 相关的操作系统变量（与前面的 shell 脚本不同）：

```bash
#!/bin/bash

mntlist="/orahome /oraredo /oraarch /ora01 /oradump01 /"

for ml in $mntlist
do
  echo $ml
  usedSpc=$(df -h $ml | awk '{print $5}' | grep -v capacity | cut -d "%" -f1 -)
  BOX=$(uname -a | awk '{print $2}')
  #
  case $usedSpc in
    [0-9])
      arcStat="放松，磁盘空间充足: $usedSpc"
      ;;
    [1-7][0-9])
      arcStat="磁盘空间正常: $usedSpc"
      ;;
    [8][0-9])
      arcStat="空间即将不足: $usedSpc"
      echo $arcStat | mailx -s "空间不足: $BOX" dkuhn@sun.com
      ;;
    [9][0-9])
      arcStat="警告，空间即将耗尽: $usedSpc"
      echo $arcStat | mailx -s "空间不足: $BOX" dkuhn@sun.com
      ;;
    [1][0][0])
      arcStat="准备更新简历吧，空间已耗尽: $usedSpc"
      echo $arcStat | mailx -s "空间不足: $BOX" dkuhn@sun.com
      ;;
    *)
      arcStat="嗯？: $usedSpc"
  esac
  #
  BOX=$(uname -a | awk '{print $2}')
  echo $arcStat
  #
done
#
exit 0
```

您可以像这样从命令行手动运行此脚本：

$ filesp.bsh

以下是此数据库服务器的输出：

/orahome
磁盘空间正常: 56
/oraredo1
磁盘空间正常: 17
/oraarch1
磁盘空间正常: 37
/ora01
空间即将不足: 81
/oradump01
磁盘空间正常: 29
/
磁盘空间正常: 65

这正是您应该通过 `cron` 等调度工具以自动化方式运行的脚本。这是一个典型的 cron 条目：

