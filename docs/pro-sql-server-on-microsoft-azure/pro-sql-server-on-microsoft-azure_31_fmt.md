# 针对 SQL Server 实例执行查询

```powershell
$backups = Invoke-Sqlcmd -ServerInstance $sqlserver -Query $sqlquery
```