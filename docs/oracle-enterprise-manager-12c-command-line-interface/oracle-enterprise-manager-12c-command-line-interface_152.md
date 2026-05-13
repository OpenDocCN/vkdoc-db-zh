# 验证是否在命令行上传递了 SID
if [ ${#1} -eq 0 ]; then
   if tty -s; then
      read thisSID? "请提供数据库名称："
  else
      echo "命令行上未提供数据库名称"
      ExitCleanly
   fi
   read thisSID?"输入要监控的数据库名称：  "
else
   thisSID=`print $@ | awk '{print$NF}' | tr '[A-Z]' '[a-z]'`
fi

BO_NAME=scripted_blackout_${thisSID}

CleanUpFiles
CreateBlackout
ExitCleanly
```

### 示例脚本：`emcli_stop_blackout.ksh`

```shell
#!/bin/ksh
