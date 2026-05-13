# 将离线和在线节点与相应的变量关联起来。
```bash
if [ `sudo ${VCSBIN}/hastatus -summary | grep OFFLINE | grep ${1} | wc -l` -eq 0 ]; then
   echo "\n\nVCS 集群 ${1} 在 ${LOCAL_HOST} 上不存在\n\n"
   ExitDirty
else
   export HAGROUP=`sudo ${VCSBIN}/hastatus -summary | grep OFFLINE | grep ${1} | awk '{ print $2 }'`
   export OFFLINE_NODE=`sudo ${VCSBIN}/hastatus -summary | grep OFFLINE | grep ${HAGROUP} | awk '{ print $3 }'`
   if [ `sudo ${VCSBIN}/hastatus -summary | grep ONLINE | grep ${HAGROUP} | wc -l` -gt 0 ]; then
      export ONLINE_NODE=`sudo ${VCSBIN}/hastatus -summary | grep ONLINE | grep ${HAGROUP} | awk '{ print $3 }'`
   else
      export ONLINE_NODE=`sudo ${VCSBIN}/hastatus -summary | grep PARTIAL | grep ${HAGROUP} | awk '{ print $3 }'`
   fi
fi
```

