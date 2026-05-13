# 此字符串将在构建 `emcli` 脚本时多次使用
```bash
EMCLI_AGENT_STRING=`echo "-src_agent=\"${OLD_EMD}\" -dest_agent=\"${NEW_EMD}\""`
```


```markdown
# 创建并精炼 OEM 目标列表

并非所有目标都由 OEM 管理，因此我们需要从仓库获取一个列表，并将其与来自 VCS 的列表进行比较。

```bash
echo "\n\n 与此更改相关的主机目标的 OEM 名称:"
echo "  重定位后的主机         ${NEW_EMD}"
echo "  本地主机名               ${OLD_EMD}"
```

## 获取并展示 OEM 管理的目标列表

```bash
$EMCLI get_targets | grep oracle_database | awk '{print $NF}' | sort -fu >${EMCLI_DATABASES}
echo "\n\n 通过 OEM 管理的数据库目标:"
thisFILE=${EMCLI_DATABASES}
ListNicely

$EMCLI get_targets | grep oracle_listener | awk '{print $NF}' | sort -fu >${EMCLI_LISTENERS}
echo "\n\n 通过 OEM 管理的监听器目标:"
thisFILE=${EMCLI_LISTENERS}
ListNicely
```

## 获取本地 EM Agent 已知的目标列表

现在，我们将对本地 EM Agent 已知的目标执行相同的操作。此列表可能包含仓库中被忽略或未提升的目标，因此我们将把 Agent 列表视为更佳选择。如果本地 Agent 没有任何目标，我们将使用来自仓库的完整列表。

```bash
$EMCTL config agent listtargets | grep oracle_database | cut -d"[" -f2 | cut -d, -f1 | sort -fu >${EMCTL_DATABASES}
echo "\n\n 当前与 ${NEW_EMD} 关联的本地数据库目标:"
thisFILE=${EMCTL_DATABASES}
ListNicely

$EMCTL config agent listtargets | grep oracle_listener | cut -d"[" -f2 | cut -d, -f1 | sort -fu >${EMCTL_LISTENERS}
echo "\n\n 当前与 ${NEW_EMD} 关联的本地监听器目标:"
thisFILE=${EMCTL_LISTENERS}
ListNicely
```

## 用本地 Agent 列表覆盖仓库列表（如果存在）

创建仓库中的较长列表是为了防止本地 Agent 不包含任何目标。如果存在任何已知于本地 Agent 的目标，我们将用它们替换 OMR 列表。

```bash
if [ `cat ${EMCTL_DATABASES} | wc -l` -gt 0 ]; then
   cat /dev/null                            >${OEM_DATABASES}
   for thisTARGET in `cat ${EMCLI_DATABASES}`; do
      if [ `cat ${EMCTL_DATABASES} | grep -i ${thisTARGET} | wc -l` -eq 0 ]; then
         echo "${thisTARGET}"              >>${OEM_DATABASES}
      fi
   done
else
   mv -f ${EMCLI_DATABASES} ${OEM_DATABASES}
fi

