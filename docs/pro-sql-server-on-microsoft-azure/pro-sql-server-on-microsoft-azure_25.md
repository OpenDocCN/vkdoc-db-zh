# 将计算机上的 secpol 权限导出到文件
$filename  = "secpol.inf"
$secpol = secedit /export /cfg $filename | Out-Null
$secpol = Get-Content $filename
