# =====================================================
export BO_NAME=${1}_scripted_blackout
echo  "\n\nStopping blackout named '${BO_NAME}' ...\n"
if [ `emcli get_blackouts | grep ${BO_NAME} | wc -l` -gt 0 ]; then
        $ECHO "\n\nFound an existing blackout named ${BO_NAME}"
        emcli stop_blackout -name="${BO_NAME}"
        emcli delete_blackout -name="${BO_NAME}"
fi
```

## 创建和使用函数库

前面的示例假设刷新一切顺利，并且 OEM 在一小时后恢复监控。当然，即使没有 `end_blackout.sh`，停用也会在启动后六小时结束。为什么不在这项工作完成后立即关闭停用呢？

将此活动转换为中心库中的 Shell 函数，允许你在 `restore_training.sh` 脚本内部启动和停止停用，如下所示：

```shell
#!/bin/sh
