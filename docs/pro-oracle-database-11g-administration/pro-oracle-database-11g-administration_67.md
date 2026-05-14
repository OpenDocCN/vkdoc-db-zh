# 检查被锁定的生产账户

通常，我会配置一个数据库配置文件，指定数据库账户在指定次数的登录尝试失败后被锁定。例如，我将 `DEFAULT` 配置文件的 `FAILED_LOGIN_ATTEMPTS` 设置为 `5`。有时会发生的情况是，某个恶意用户或开发人员试图猜测生产账户密码，经过 5 次尝试后，这会锁定生产账户。当这种情况发生时，我需要尽快知晓，以便调查问题然后解锁账户。

**清单 21–6.** 用于检查 `DBA_USERS` 中 `LOCK_DATE` 值的 shell 脚本
```
#!/bin/bash
if [ $# -ne 1 ]; then
    echo "Usage: $0 SID"
    exit 1
fi
# 加载 oracle 操作系统变量
. /var/opt/oracle/oraset $1
#
crit_var=$(sqlplus -s <<EOF
/ as sysdba
SET HEAD OFF FEED OFF
SELECT count(*)
FROM dba_users
WHERE lock_date IS NOT NULL
AND username in ('CIAP','REPV','CIAL','STARPROD');
EOF)
#
if [ $crit_var -ne 0 ]; then
    echo $crit_var
    echo "locked acct. issue with $1" | mailx -s "locked acct. issue" dkuhn@sun.com
else
    echo $crit_var
    echo "no locked accounts"
fi
exit 0
```

此 shell 脚本从诸如 `cron` 的调度工具调用。例如，此 `cron` 条目指示作业每十分钟运行一次：
```
# 检测被锁定的数据库账户的作业
0,10,20,30,40,50 * * * * /home/oracle/bin/lock.bsh DWREP
1>/home/oracle/bin/log/lock.log 2>&1
```

通过这种方式，当生产数据库账户之一被锁定时，我会收到通知。在此 `cron` 条目中，代码应位于一行。本书中将其置于两行是为了适应页面排版。

