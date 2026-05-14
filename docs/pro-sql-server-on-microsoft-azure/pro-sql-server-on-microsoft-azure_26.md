# 在输出中搜索卷维护任务权限
$IFI = Select-String -Path $filename -Pattern "SeManageVolumePrivilege"
