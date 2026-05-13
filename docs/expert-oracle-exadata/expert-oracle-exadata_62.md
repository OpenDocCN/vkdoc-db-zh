# 使用 GetExaWatcherResults.sh 脚本

每当出现问题时，您会特别关注性能数据在事件发生前后的状态。在同一个目录下，可以使用 `GetExaWatcherResults.sh` 脚本一次性获取所有 ExaWatcher 归档的命令输出。下面的命令会在指定时间戳的前后两小时提取数据，并将其放在默认的归档目录 `/opt/oracle.ExaWatcher/archive/ExtractedResults` 下。该脚本的其他选项可用于指定更精确的时间范围和不同的归档位置。脚本期望日期的格式为 `mm/dd/yyyy`：

```
[root@enkcel04 oracle.ExaWatcher]# ./GetExaWatcherResults.sh --at '01/30/2015 09:00:00'
> --range 2
```

在 Exadata 12.1.2.1.0 中，屏幕上不会有进一步的输出。如果您想了解具体情况，需要查看日志和跟踪文件，路径为 `/var/log/cellos/ExaWatcher.log` 或 `/var/log/cellos/ExaWatcher.trc`。输出内容将被保存到 `/opt/oracle.ExaWatcher/archive/ExtractedResults`。压缩文件包含了指定时间段内 ExaWatcher 监控的所有子系统的信息。使用 `--help` 选项调用该脚本会显示有用的在线帮助。

