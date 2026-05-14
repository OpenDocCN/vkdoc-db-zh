# Loop through the database properties
foreach ($db in $dbs)
{
    if ($db.is_auto_shrink_on -eq $true)
    {
        Write-Host "[ERR] Database" $db.name "has Auto Shrink turned on" -ForegroundColor Red
        $RulePass = 0
    }
    if ($db.is_auto_close_on -eq $true)
    {
        Write-Host "[ERR] Database" $db.name "has Auto Close turned on" -ForegroundColor Red
        $RulePass = 0
    }
}
if ($RulePass -eq 1)
{
    Write-Host "[INFO] No databases found with Auto Close and Auto Shrink turned on" -ForegroundColor Green
}
```
清单 7-7. 用于确定任何数据库上是否启用了 AUTO CLOSE 和 AUTO SHRINK 的 PowerShell 代码

在虚拟机上安装 SQL Server 后，您可能需要执行的一些维护任务包括：

1.  更改数据库数据和日志文件的默认位置，以确保新的数据库文件托管在为托管 SQL Server 数据库文件而配置的虚拟机的附加数据磁盘上。
2.  将系统数据库文件移动到数据磁盘。
3.  将错误日志、跟踪文件位置和扩展事件文件目标移动到数据磁盘。


### 服务账户权限

关于是否为 SQL Server 服务账户启用“在内存中锁定页面”权限，网上一直存在一个长期的争论。“在内存中锁定页面”是 Windows 的一项策略，它决定哪个账户可以使用进程将内存分配锁定在物理内存中。它能防止系统将数据分页到磁盘上的虚拟内存中。当 SQL Server 服务账户被授予此用户权限时，缓冲池内存便无法被 Windows 分页出去。对于运行在 Azure 虚拟机上的 SQL Server 实例，建议您将“在内存中锁定页面”安全权限授予 SQL Server 服务账户。这可以防止 SQL Server 进程的工作集被分页到磁盘。遭受明显性能打击的最大缺点是，虚拟机的页面文件默认驻留在本地连接的磁盘上。清单 7-8 展示了如何确定是否已将“在内存中锁定页面”安全权限授予 SQL Server 服务账户。

**注意**

请务必为 SQL Server 实例限制 `MAX SERVER MEMORY`，尤其是在为 SQL Server 服务账户启用“在内存中锁定页面”安全权限时。

```
$RulePass = 1
$sqlquery = "SELECT locked_page_allocations_kb FROM sys.dm_os_process_memory"
