# --------------------------------------------------------------------------------------------------

WORKFILE01=/tmp/oem_resyncagents.tmp

CleanUpFiles

emcli login -user=sysman -pass="${SysmanPassword}"

echo "获取无法访问代理的目标名称..."
if [ `emcli get_targets -targets="oracle_emd" | grep Unreach | awk '{ print $5 }' | wc -l` -gt 0 ]; then
   echo "创建处于“无法访问”状态的代理列表"
   emcli get_targets -targets="oracle_emd" | grep Unreachable | awk '{ print $5 }' >${WORKFILE01}
   if [ `cat ${WORKFILE01} | wc -l` -gt 0 ]; then
      for thisAGENT in `cat ${WORKFILE01}`; do
         echo "尝试重新同步 ${thisAGENT}"
         echo "这会花费相当长的时间，所以请耐心等待"
         emcli resyncAgent -agent="${thisAGENT}"
      done
   else
      echo "当前没有无法访问的代理"
   fi
else
   echo "当前没有无法访问的代理"
fi

ExitCleanly
}
```

### 保护所有代理

```shell
function SecureAllAgents {

