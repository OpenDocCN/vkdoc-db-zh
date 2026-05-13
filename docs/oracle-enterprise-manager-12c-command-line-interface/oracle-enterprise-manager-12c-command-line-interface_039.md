# =====================================================
export BO_NAME=${1}_scripted_blackout

echo  "\n\nCreating blackout named '${BO_NAME}' ...\n"

if [ `emcli get_blackouts | grep ${BO_NAME} | wc -l` -gt 0 ]; then
        $ECHO "\n\nFound an existing blackout named ${BO_NAME}"
        $ECHO "That blackout will be stopped and deleted prior to starting the new one\n\n"
        emcli stop_blackout -name="${BO_NAME}"
        emcli delete_blackout -name="${BO_NAME}"
fi

emcli create_blackout
-name="${BO_NAME}"
        -add_targets="${thisTARGET}:oracle_database"
-schedule="duration::360"
-reason="Scripted blackout for maintenance or refresh"

sleep 5

echo "\n\nGetting blackout information for '${BO_NAME}' ...\n\n"
emcli get_blackout_details -name="${BO_NAME}"
```

在这个简短的脚本中，有几件事需要注意：

- 每个停用名称必须是唯一的，因此此脚本要求部分名称在命令行传递。在此示例中，我们传递了 SID。
- 用户沟通在任何时候都很重要，因此要完整且简洁。你的 `echo` 语句在运行时很有用，在日志文件中也是必不可少的。
- 此计划停用最多运行六小时（`duration:360`），因此你需要删除同名的现有停用以获得完整的六小时。
- 请注意，`stop_blackout` 和 `delete_blackout` 动词仅将停用名称作为其唯一输入值。你必须先停止停用才能删除它。
- 停用状态的测试对于运行时日志文件很重要。

在实践中，这表现为 crontab 中的一系列包装条目，如下所示：

```
25 19 * * * /scripts/oem/create_blackout.sh gold
30 19 * * * /scripts/refresh/restore_training.sh gold
30 20 * * * /scripts/oem/end_blackout.sh gold
```

当你启动停用时，你也应该计划停止它。当然，脚本对的这一边更简单，因为要做的事情少得多：

```shell
#!/bin/sh
