# 清理工作文件

```bash
[ $WORKFILE ] && rm -f ${WORKFILE}

[ $WORKFILE01 ] && rm -f ${WORKFILE01}

[ $WORKFILE02 ] && rm -f ${WORKFILE02}
```

## 退出清理函数

```bash
function ExitCleanly {
```