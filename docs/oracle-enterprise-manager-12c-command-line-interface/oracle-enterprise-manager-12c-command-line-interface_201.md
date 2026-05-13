# 监听器执行相同的过程
if [ `cat ${EMCTL_LISTENERS} | wc -l` -gt 0 ]; then
   cat /dev/null                            >${OEM_LISTENERS}
   for thisTARGET in `cat ${EMCLI_LISTENERS}`; do
      if [ `cat ${EMCTL_LISTENERS} | grep -i ${thisTARGET} | wc -l` -eq 0 ]; then
         echo "${thisTARGET}"              >>${OEM_LISTENERS}
      fi
   done
else
   mv -f ${EMCLI_LISTENERS} ${OEM_LISTENERS}
fi
```

## 对比 OEM 目标列表与 Veritas 成员并构建 emcli 脚本

```bash
if [ `cat ${VCS_DATABASES} | wc -l` -gt 0 ]; then
   cat /dev/null                            >${MOVING_DATABASES}
   for thisITEM in `cat ${VCS_DATABASES}`; do
      cat ${OEM_DATABASES} | grep -i ${thisITEM} >>${MOVING_DATABASES}
   done
fi

cat /dev/null                            >${EMCLI_SCRIPT}

if [ `cat ${MOVING_DATABASES} | wc -l` -gt 0 ]; then
   echo "\n\n 以下数据库目标将被重定位"
   thisFILE=${MOVING_DATABASES}
   ListNicely
   for thisITEM in `cat ${MOVING_DATABASES}`; do
      EMCLI_TARGET_STRING=`echo "-target_name=\"${thisITEM}\" -target_type=\"oracle_database\""`
      echo "set_standby_agent ${EMCLI_AGENT_STRING} ${EMCLI_TARGET_STRING}"  >>${EMCLI_SCRIPT}
      echo "relocate_targets  ${EMCLI_AGENT_STRING} ${EMCLI_TARGET_STRING} -copy_from_src -force=yes" >>${EMCLI_SCRIPT}
   done
else
   echo "\n\n 没有数据库目标需要迁移到 ${NEW_EMD}"
fi

if [ `cat ${VCS_LISTENERS} | wc -l` -gt 0 ]; then
   cat /dev/null                            >${MOVING_LISTENERS}
   for thisITEM in `cat ${VCS_LISTENERS} | sed 's/lsnr//'`; do
      cat ${OEM_LISTENERS} | grep -i ${thisITEM} >>${MOVING_LISTENERS}
   done
else
   echo "\n\nVCS 没有与 ${HAGROUP} 组关联的监听器"
fi

if [ `cat ${MOVING_LISTENERS} | wc -l` -gt 0 ]; then
   echo "\n\n 以下监听器目标将被重定位到 ${NEW_EMD}"
   thisFILE=${MOVING_LISTENERS}
   ListNicely
   for thisITEM in `cat ${MOVING_LISTENERS}`; do
      EMCLI_TARGET_STRING=`echo "-target_name=\"${thisITEM}\" -target_type=\"oracle_listener\""`
      echo "set_standby_agent ${EMCLI_AGENT_STRING} ${EMCLI_TARGET_STRING}" >>${EMCLI_SCRIPT}
      echo "relocate_targets  ${EMCLI_AGENT_STRING} ${EMCLI_TARGET_STRING} -copy_from_src" >>${EMCLI_SCRIPT}
   done
else
   echo "\n\n 没有监听器目标需要迁移到 ${NEW_EMD}"
fi

sleep 10
```

## 显示并执行 emcli 脚本

```bash
if [ `cat ${EMCLI_SCRIPT} | wc -l` -gt 0 ]; then
   echo "\n\nEMCLI 脚本"
   cat ${EMCLI_SCRIPT}
   echo "\n\nEMCLI 执行结果"
   $EMCLI argfile ${EMCLI_SCRIPT}
fi

echo "\n\n 以下目标现在由本地 OEM Agent 追踪:\n"
$EMCTL config agent listtargets | grep -v "Cloud" | grep -v "reserved." | sort -fu
echo "\n\n\n"

ExitCleanly
```

## 第 3 节：必要的服务器管理脚本

本节中的 Shell 脚本包含了 `EM CLI` 的各种用法，包括备份 OMS 服务器上的所有配置文件，以及控制 OMS 日志文件占用的空间量。

对于他们认为关键的文件，Oracle 在提供这些服务方面一直做得很好。这些脚本负责处理所有其余部分。

## 配置备份脚本

OEM 将关键的 OMS 配置文件备份到仓库数据库。此脚本对这些文件（以及其他几个文件）执行文件级备份。应使用与软件库相同的共享文件系统作为这些备份的存放位置。

此脚本仅在确定源文件与其备份副本内容不同时，才会创建新备份。如果存在差异，旧备份将被重命名为 `.old` 后缀。这为您提供了另一个故障排除工具，在 OMS 打补丁后尤其有用。您可以使用文件创建日期来确定哪些配置文件发生了更改，然后运行快速的 `diff` 命令来找出它们的变化方式：

```ksh
示例脚本: backup_oms_configs.ksh
#!/bin/ksh

#===============================================================================
