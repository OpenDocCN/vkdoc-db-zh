# --------------------------------------------------------------------------------------------------

WORKFILE01=/tmp/oem_secure_agents.tmp

CleanUpFiles

emcli login -user=sysman -pass="${SysmanPassword}"

emcli get_targets | grep oracle_emd | sort -u | awk '{ print $4 }' >${WORKFILE01}

emcli secure_agents -agt_names_file=${WORKFILE01} -use_pref_creds

ExitCleanly
}
```

## 用户账户凭据

### 设置数据库用户凭据

```shell
function SetDBUserCredential {

