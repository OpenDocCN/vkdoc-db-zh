# 查找计算机上安装的 SQL Server 服务

```powershell
$SqlService = Get-WmiObject -Query "SELECT * FROM Win32_Service WHERE PathName LIKE '%sqlservr%'"
```