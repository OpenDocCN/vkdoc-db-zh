# 此列表可能与这些主机上 `OEM` 所知的目标列表不匹配，因此我们将在脚本后面进行比较。
```bash
sudo ${VCSBIN}/hares -display -type Oracle  | grep ${HAGROUP} | awk '{ print $1 }' >${VCS_DATABASES}
sudo ${VCSBIN}/hares -display -type Netlsnr | grep ${HAGROUP} | awk '{ print $1 }' >${VCS_LISTENERS}

if [ `cat ${VCS_DATABASES} | wc -l` -gt 0 ]; then
   echo "\n\nVCS 显示 ${HAGROUP} 组中存在以下数据库"
   thisFILE=${VCS_DATABASES}
   ListNicely
else
   echo "\n\nVCS 在 ${HAGROUP} 组中没有关联的数据库"
   ExitDirty
Fi

if [ `cat ${VCS_LISTENERS} | wc -l` -gt 0 ]; then
        echo "\n\nVCS 显示 ${HAGROUP} 组中存在以下监听器"
        thisFILE=${VCS_LISTENERS}
        ListNicely
Else
        echo "\n\nVCS 在 ${HAGROUP} 组中没有关联的监听器"
fi

if [ ${OFFLINE_NODE} == ${LOCAL_HOST} ]; then
   echo "\n\n${HAGROUP} 组位于集群的另一端"
   echo "此脚本必须在活动节点 ${ONLINE_NODE} 上运行\n\n\n"
   ExitDirty
Fi
```

