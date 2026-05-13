# 代理总是安装在静态文件系统上，从不进行故障切换。
```bash
NEW_EMD=`$EMCLI get_targets | grep ${ONLINE_NODE}  | grep oracle_emd |  awk '{print $NF }'`
OLD_EMD=`$EMCLI get_targets | grep ${OFFLINE_NODE} | grep oracle_emd |  awk '{print $NF }'`
```

