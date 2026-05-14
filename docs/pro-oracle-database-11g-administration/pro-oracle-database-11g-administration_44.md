# 首先仅使用元数据创建导出转储文件

```bash
expdp darl/foo dumpfile=${SID}.${DAY}.dmp content=metadata_only \
directory= dwrep_dp full=y logfile=${SID}.${DAY}.log
```

#---------------------------------------------------



