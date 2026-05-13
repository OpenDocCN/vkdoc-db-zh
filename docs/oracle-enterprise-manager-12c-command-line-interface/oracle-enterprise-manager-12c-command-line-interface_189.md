# 因此我们将它们收集到不同的目标列表中以进行处理
VCS_DATABASES=/tmp/RelocateTargetDatabases_VCS.lst
VCS_LISTENERS=/tmp/RelocateTargetListeners_VCS.lst

EMCLI_DATABASES=/tmp/RelocateTargetDatabases_EMCLI.lst
EMCLI_LISTENERS=/tmp/RelocateTargetListeners_EMCLI.lst

EMCTL_DATABASES=/tmp/RelocateTargetDatabases_EMCTL.lst
EMCTL_LISTENERS=/tmp/RelocateTargetListeners_EMCTL.lst

OEM_DATABASES=/tmp/RelocateTargetDatabases_OEM.lst
OEM_LISTENERS=/tmp/RelocateTargetListeners_OEM.lst

MOVING_DATABASES=/tmp/RelocateTargetDatabases_Moving.lst
MOVING_LISTENERS=/tmp/RelocateTargetListeners_Moving.lst

EMCLI_SCRIPT=/tmp/RelocateTarget_emcli_script.lst

## 函数

### `CleanUpFiles`
```bash
# --------------------------------------------------------------------------------------------------
[ $VCS_DATABASES ] && rm -f ${VCS_DATABASES}
[ $VCS_LISTENERS ] && rm -f ${VCS_LISTENERS}
[ $EMCLI_DATABASES ] && rm -f ${EMCLI_DATABASES}
[ $EMCLI_LISTENERS ] && rm -f ${EMCLI_LISTENERS}
[ $EMCTL_DATABASES ] && rm -f ${EMCTL_DATABASES}
[ $EMCTL_LISTENERS ] && rm -f ${EMCTL_LISTENERS}
[ $OEM_DATABASES ] && rm -f ${OEM_DATABASES}
[ $OEM_LISTENERS ] && rm -f ${OEM_LISTENERS}
[ $MOVING_DATABASES ] && rm -f ${MOVING_DATABASES}
[ $MOVING_LISTENERS ] && rm -f ${MOVING_LISTENERS}
[ $EMCLI_SCRIPT ] && rm -f ${EMCLI_SCRIPT}
```

### `ExitCleanly`
```bash
# --------------------------------------------------------------------------------------------------
CleanUpFiles
exit 0
```

### `ExitDirty`
```bash
# --------------------------------------------------------------------------------------------------
CleanUpFiles
exit 1
```

### `ListNicely`
```bash
# --------------------------------------------------------------------------------------------------
for thisLINE in `cat ${thisFILE}`; do
   echo "  ${thisLINE}"
done
```

## 运行时过程

