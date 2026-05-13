# --------------------------------------------------------------------------------------------------

GetTargetName

echo "停止黑名单 '${BO_NAME}' ..."
emcli stop_blackout -name="${BO_NAME}"

echo "删除黑名单 '${BO_NAME}' ..."
emcli delete_blackout -name="${BO_NAME}"
}
```

## 代理同步

### 重新同步不可达的代理

```shell
function ResyncUnreachableAgents {

