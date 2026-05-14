# 获取账户的最佳实践
foreach ($Account in $Accounts)
{
    $StorageAccount = Get-AzureRmStorageAccount -ResourceGroupName $RGName -Name $Account
    Write-Host "***** 存储账户检查: " $StorageAccount.StorageAccountName
    if ($StorageAccount.AccountType.ToString().Contains("LRS"))
    {
        Write-Host "[INFO] 该存储账户未启用复制" -ForegroundColor Green
    }
    else
    {
        Write-Host "[ERR] 账户类型: " $StorageAccount.AccountType -ForegroundColor Red
        Write-Host "[ERR] 建议为您的存储账户禁用任何形式的复制" -ForegroundColor Red
    }
    if ($VM.Location -eq $StorageAccount.Location.Replace(" ",""))
    {
        Write-Host "[INFO] 存储和计算位于同一位置" -ForegroundColor Green
    }
    else
    {
        Write-Host "[ERR] 存储和计算不在同一位置" -ForegroundColor Red
    }
}
```

清单 7-2. 用于存储最佳实践的 PowerShell 脚本



### 数据磁盘

在考虑 SQL Server 数据库文件与操作系统磁盘时，无论平台和环境如何，都有一些最佳实践确实值得遵循。将数据文件和日志文件混合存放在同一卷中，或者在 Azure VM 场景下存放在同一数据磁盘上，从来都不是一个好主意。SQL Server 数据文件依赖于随机 I/O，而事务日志执行顺序 I/O。由于 I/O 特性的不同，当数据和日志在短时间内执行大量操作时，存储介质无法达到最佳性能。

对于 Azure 磁盘，存在一种“预热效应”，可能导致吞吐量和带宽在短时间内降低。在数据磁盘一段时间（大约 20 分钟）未被访问的情况下，自适应分区和负载均衡机制会启动。如果在这些算法活动期间访问磁盘，你可能会注意到吞吐量和带宽在短时间内（大约 10 分钟）有所下降，之后它们会恢复到正常水平。这种预热效应是由于 Azure 的自适应分区和负载均衡机制引起的，该机制在多租户存储环境中动态调整以适应工作负载变化。在决定将文件放置在数据磁盘上时，牢记这一点很重要。

操作系统磁盘是本地连接的磁盘，其性能等级与托管高性能、低延迟数据库文件的要求不同。建议不要在操作系统磁盘上存储任何东西，正如其名所示，只存放操作系统文件。从库中部署的所有 Azure 虚拟机都使用 `C:` 作为操作系统驱动器。清单 7-3 展示了一个用于检查数据库文件是否托管在操作系统驱动器上的代码示例。

```powershell
$RulePass = 1
$sqlquery = "select distinct substring(physical_name,1,2) as drive from sys.master_files where substring(physical_name,2,1) = ':'"
$sqlDrives = Invoke-Sqlcmd -ServerInstance $sqlserver -Query $sqlquery
