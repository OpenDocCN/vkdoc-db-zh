# --------------------------------------------------------------------------------------------------

GetTargetName

echo "Creating blackout named '${BO_NAME}' ..."

if [ `emcli get_blackouts | grep ${BO_NAME} | wc -l` -gt 0 ]; then
   echo "Found an existing blackout named ${BO_NAME}"
   echo "That blackout will be stopped and deleted prior to starting the new one"
   emcli stop_blackout -name="${BO_NAME}"
   emcli delete_blackout -name="${BO_NAME}"
fi
```


# EMCLI 函数库与示例脚本

## 创建黑名单

```shell
emcli create_blackout -name="${BO_NAME}" \
   -add_targets=${thisTARGET}:oracle_database \
   -schedule="duration::360;tzinfo:specified;tzregion:America/Los_Angeles" \
   -reason="Scripted blackout for maintenance or refresh"

sleep 5

echo "Getting blackout information for '${BO_NAME}' ..."
emcli get_blackout_details -name="${BO_NAME}"
}
```

## 停止黑名单

```shell
function EndBlackout {

