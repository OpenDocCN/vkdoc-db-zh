# Run smoke tests
Write-Host 'Run smoke tests...'
Get-Service -DisplayName ('*{0}*' -f $InstanceName)
Write-Host 'Service running check complete'
Invoke-SqlCmd -Serverinstance $connectionString -Query "SELECT @@SERVERNAME" -TrustServerCertificate
Write-Host 'Instance accessibility check complete'
```

`SQLAutoInstall3.ps1` 脚本在运行时没有指定任何参数。在以前版本的脚本中，PowerShell 会继续执行代码，然后 `setup.exe` 会因为没有为所需参数指定值而失败。然而，在这个版本中，您可以看到系统会依次提示您为每个参数输入值。这是通过两种不同的方式实现的——要么通过 `Mandatory` 属性，要么通过提供一个默认值（在本例中，这个默认值恰好是一个会提示用户输入缺失凭据的命令）。

运行脚本时，您会注意到在脚本执行的每个阶段前后，我们的注释都会显示出来。这可以帮助您应对错误，因为您甚至可以在开始解读可能显示的任何错误消息之前，就能轻松地看到是哪个命令导致了问题。

## 使用 DSC 实现自动化安装

在第 3 章中，我们讨论了如何使用 DSC 配置 Windows 操作系统。在本章中，我们将扩展该配置，使其也能安装我们的 SQL Server 实例。为此，我们需要安装 `SqlServerDSC` 模块，此操作可以使用清单 6-14 中的命令完成。这必须在 PowerShell 5 中进行。

```powershell
Install-Module SqlServerDsc
```

现在，我们可以扩展在第 3 章中创建的配置，添加一个资源，该资源将确保创建一个名为 `DSCInstance` 的 SQL Server 实例（使用 Developer 版本）。此资源如清单 6-15 所示。您会注意到我们传递了实例的期望名称、SQL Server 安装介质的源路径以及产品密钥（均为字符串）。我们将 `SQLENGINE` 传递给 `Features` 参数，但如果我们要安装多个组件，这个字符串将由逗号分隔的列表组成。我们还传递了一个 Windows 登录名数组，我们希望将这些登录名添加到 `sysadmins` 固定服务器角色中。在我们的例子中，我们只传递了一个登录名——`Administrator`。

注意：您应更改源路径以反映您的 SQL Server 安装介质的位置。

```powershell
SqlSetup 'InstallInstance' {
    InstanceName = 'DSCInstance'
    Features = 'SQLENGINE'
    SourcePath = 'C:\SQL Media'
    SQLSysAdminAccounts = @('Administrator')
    ProductKey = '22222-00000-00000-00000-00000'
}
```

提示：我们传递的产品密钥是 Developer 版本的产品密钥，这意味着将安装此版本。如果我们传递 Enterprise 或 Standard 版本的产品密钥，这将改变安装的版本。

那么，让我们来检查清单 6-16，它展示了我们更新后的配置（添加了新资源）将是什么样子。关于此配置有两点需要注意。第一点是，我们将 `SQLServerService` 资源管理的服务从默认实例更改为 `DSCInstance` 实例，因为这是我们将要使用的实例。第二点需要注意的重要方面是，在 `SQLServerService` 资源的末尾添加了语法 `DependsOn = '[SqlSetup]InstallInstance'`。这是因为我们不希望配置尝试确保 SQL Server Database Engine 服务在实例安装之前启动。当配置被编译时，无法保证资源的配置顺序。因此，如果资源之间存在依赖关系，我们应始终使用 `DependsOn` 来确保正确的顺序。

```powershell
Configuration WindowsConfig {
    Import-DscResource -ModuleName 'PSDesiredStateConfiguration'
    Import-DscResource -ModuleName 'SqlServerDsc'
    Node 'localhost' {
        File CreateCertificateBackupsFolder {
            Ensure = "Present"
            Type = "Directory"
            DestinationPath = "C:\CertificateBackups"
        }
        Registry OptimizeForBackgroundServices {
            Ensure      = "Present"
            Key         = " HKEY_LOCAL_MACHINE:\SYSTEM\CurrentControlSet\Control\PriorityControl"
            ValueName   = "Win32PrioritySeparation"
            ValueData   = 24
            ValueType   = 'Dword'
        }
        SqlSetup 'InstallInstance' {
            InstanceName        = 'DSCInstance'
            Features            = 'SQLENGINE'
            SourcePath          = 'C:\SQL Media'
            SQLSysAdminAccounts = @('Administrator')
            ProductKey          = '22222-00000-00000-00000-00000'
        }
        Service SQLServerService
        {
            Name        = 'MSSQL$DSCInstance'
            StartupType = 'Automatic'
            State       = 'Running'
            DependsOn    = '[SqlSetup]InstallInstance'
        }
    }
}
WindowsConfig
```


# 目前，此配置有效，但相当不灵活。它总是安装相同名称的实例，并且总是安装开发者版。为了使其更灵活，我们可以将配置参数化，以便在应用配置时，可以传递与相关节点相关的实例名称和产品密钥。

此配置的参数化版本如列表 6-17 所示。您会注意到声明中添加了一个`param`块，它接受实例名称和版本的参数。`$SqlInstanceName`参数的值被设为`'MSSQLSERVER'`，这表示一个默认实例。还添加了一个`IF...ELSE IF...`块，该块根据传递到配置的版本计算要使用的产品密钥。

> **注意**
>
> 出于本示例的目的，评估版的产品密钥被用于企业和标准版。这些产品密钥应替换为您自己的企业和标准版产品密钥。

```powershell
Configuration WindowsConfig {
    param (
        [string] $SqlInstanceName = 'MSSQLSERVER',
        [Parameter(Mandatory)]
        [ValidateSet('Developer', 'Standard', 'Enterprise')]
        [string] $Edition
    )
    Import-DscResource -ModuleName 'PSDesiredStateConfiguration'
    Import-DscResource -ModuleName 'SqlServerDsc'
    if ($Edition -eq 'Developer') {
        $ProductKey = '22222-00000-00000-00000-00000'
    } elseif ($edition -eq 'Standard') {
        $ProductKey = '00000-00000-00000-00000-00000'
    } elseif ($edition -eq 'Enterprise') {
        $ProductKey = '00000-00000-00000-00000-00000'
    }
    Node 'localhost' {
        File CreateCertificateBackupsFolder {
            Ensure = "Present"
            Type = "Directory"
            DestinationPath = "C:\CertificateBackups"
        }
        Registry OptimizeForBackgroundServices {
            Ensure      = "Present"
            Key         = "HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl"
            ValueName   = "Win32PrioritySeparation"
            ValueData   = 24
            ValueType   = 'Dword'
        }
        SqlSetup 'InstallInstance' {
            InstanceName        = $SqlInstanceName
            Features            = 'SQLENGINE'
            SourcePath          = 'C:\SQL Media'
            SQLSysAdminAccounts = @('Administrator')
            ProductKey          = $ProductKey
        }
        Service SQLServerService {
            Name        = 'MSSQL${0}' -f $SqlInstanceName
            StartupType = 'Automatic'
            State       = 'Running'
            DependsOn    = '[SqlSetup]InstallInstance'
        }
    }
}
```
列表 6-17
参数化配置

我们现在可以将参数传递给`Start-DscConfiguration`来应用配置。列表 6-18 中的命令将应用配置，确保`DSCInstance`存在并使用开发者版。这里的一个细微之处是，因为我们传递了参数，所以我们必须先点源化脚本，将其带入与调用上下文相同的会话中。

```powershell
