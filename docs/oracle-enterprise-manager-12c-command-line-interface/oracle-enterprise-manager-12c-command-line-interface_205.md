# 独立变量
#===============================================================================
export OMS_HOME=<在此设置您的 OMS 主目录>
export BACKUP_DIR=<在此设置备份目标目录>
```



## 函数

## DashedBreak
```bash
function DashedBreak {
echo "\n--------------------------------------------------------------------------------\n"
}
```

## PrepareBackupDir
```bash
function PrepareBackupDir {

SOURCE_DIR=${OMS_HOME}/${DIRNAME}
TARGET_DIR=${BACKUP_DIR}/${DIRNAME}
STAGE_DIR=${TARGET_DIR}/staging
if [ -d ${TARGET_DIR} ]; then
   echo "\n\nFound directory ${TARGET_DIR}\n\n"
else
   echo "\n\nCreating directory ${TARGET_DIR}\n\n"
   mkdir -p ${TARGET_DIR}
fi

chmod 770 ${TARGET_DIR}

if [ ! -d ${STAGE_DIR} ]; then
   mkdir ${STAGE_DIR}
fi

chmod 770 ${STAGE_DIR}
}
```

## BackupConfigFile
```bash
function BackupConfigFile {

sourceFILE=${SOURCE_DIR}/${FILENAME}
stageFILE=${TARGET_DIR}/staging/${FILENAME}
targetFILE=${TARGET_DIR}/${FILENAME}
oldFILE=${targetFILE}.old
oldZIP=${oldFILE}.zip

DashedBreak

if [ -f ${sourceFILE} ]; then
   if [ -f ${targetFILE} ]; then
      echo "\nBacking up ${sourceFILE}\n"
      cp -f ${sourceFILE} ${stageFILE}
      if [ `diff ${stageFILE} ${targetFILE} | wc -l` -gt 1 ]; then
         echo "\n\tExisting backup file will be renamed and gzipped\n"
            if [ -f ${targetFILE} ]; then
               ## 删除这两个文件的旧副本
               [ $oldZIP ] && rm -f ${oldZIP}
               [ $oldFILE ] && rm -f ${oldFILE}
               ## 重命名最新的备份副本并使用 gzip 压缩
               echo "\tRenaming the existing backup file\n"
               mv ${targetFILE} ${oldFILE}
               echo "\tGzipping that copy\n"
               gzip -f ${oldFILE} --fast
               chmod 600 ${oldZIP}
            fi
         echo "\nBacking up ${sourceFILE}\n"
         mv ${stageFILE} ${targetFILE}
         chmod 640 ${targetFILE}
         echo "\n\tNew backup file moved into place\n"
      else
         echo "\n\tThe current copy of ${FILENAME} is identical to the backup copy\n"
         rm -f ${stageFILE}
      fi
   else
      echo "\n\tMaking a copy of ${sourceFILE}\n"
      cp ${sourceFILE} ${targetFILE}
   fi
   echo "\n\t${sourceFILE} does not exist on this server\n\n"
fi

chmod 640 ${targetFILE}

echo "\n\tFinished\n\n"
}
```

# 运行时流程

```bash
echo "\nBackup directories and contents before processing\n\n"
ls -lARp ${BACKUP_DIR} | grep -v "^total"

DashedBreak

