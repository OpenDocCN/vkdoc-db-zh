# --------------------------------------------------------------------------------------------------

emcli set_credential \
      -target_type=oracle_database \
      -target_name=${thisSID} \
      -credential_set=DBCredsNormal \
      -columns="username:dbsnmp;password:${SysmanPassword};role:Normal"
}
```

### 设置用户凭据

```shell
function SetUserCredential {
