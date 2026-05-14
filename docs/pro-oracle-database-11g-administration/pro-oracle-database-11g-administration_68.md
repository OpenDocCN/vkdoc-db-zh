# 检查超过一定期限的文件

对于我从 shell 脚本运行的一些作业（如备份），我会首先检查另一个备份作业是否已经在运行。该检查涉及查找一个锁文件。如果锁文件存在，则 shell 脚本退出。

如果锁文件不存在，则创建一个。在作业结束时，锁文件被删除。

有时发生的情况是作业出现问题，并在删除锁文件之前异常中止。在这些情况下，我想知道服务器上是否存在超过一天的旧锁文件。存在旧锁文件表明出现问题，因此我需要调查。**清单 21–7** 检查 `/tmp` 目录中任何超过一天的锁文件。

**清单 21–7.** 用于检查 `/tmp` 目录中任何超过一天锁文件的脚本
```
#!/bin/bash
BOX=$(uname -a | awk '{print $2}')
# 查找所有超过 1 天的旧锁文件。
# 在 /tmp 中查找所有锁文件，如果找到，则查找任何超过一天的
ls /tmp/*.lock 2>/dev/null && \
filevar=$(find /tmp/*lock -type f -mtime +1 | wc -l) || filevar=0
if [ $filevar -gt 0 ]; then
    echo "$BOX, lockfile issue: $filevar" | \
    mailx -s "$BOX lockfile problem" dkuhn@sun.com
else
    echo "Lock file ok: $filevar"
fi
exit 0
```

我通常每天检查锁文件的存在。以下是一个典型的 `cron` 条目，用于运行名为 `lock_chk.bsh` 的前述脚本：
```
```


