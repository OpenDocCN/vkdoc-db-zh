# 运行冒烟测试
Get-Service -DisplayName ('*{0}*' -f $InstanceName)
Invoke-SqlCmd -Serverinstance $connectionString -Query "SELECT @@SERVERNAME" -TrustServerCertificate
```
`Listing 6-12` 增强版的 PowerShell 自动安装脚本

除了将变量传递到 `setup.exe` 命令外，该脚本还使用 `$InstanceName` 参数作为冒烟测试的输入。该参数可以直接传递给 `Get-Service` cmdlet，两边使用通配符。然而，对于 `Invoke-Sqlcmd`，我们需要多做一点工作。`Invoke-Sqlcmd` 需要实例的全名，包括服务器名称，或者如果脚本总是在本地运行，则可以是 `localhost`。该脚本从 `ComputerName` 环境变量中获取服务器名称，然后将其与 `$InstanceName` 变量连接起来，中间放置一个 `\`。这个连接后的值填充了 `$ConnectionString` 变量，然后可以传递给 `-Serverinstance` 开关。


### 生产环境就绪

最后，您可能希望为脚本添加一些防御性编程，使其达到生产环境就绪的状态。尽管 PowerShell 具有 `try`/`catch` 功能，但由于 `setup.exe` 是一个外部应用程序，其返回退出代码的方式不被 PowerShell 识别，因此在此处无法使用。因此，确保此脚本平稳运行的最有效技术是强制要求使用参数。

清单 6-13 中的代码是该脚本的一个修改版本，我们将其称为 `SQLAutoInstall3.ps1`。此版本的脚本使用 `Parameter` 关键字为每个参数将 `Mandatory` 属性设置为 `$true`。这一点很重要，因为如果运行此脚本的人遗漏了任何参数，或者参数名称有误，安装就会失败。这通过确保在允许脚本运行前所有参数都已输入，提供了一种故障保险机制。我们在此脚本中所做的另一项更改是在每个步骤前后添加了注释，这样如果脚本运行失败，我们可以轻松地看到错误发生在哪里。

```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]       $InstanceName,
    [PSCredential] $SQLServiceAccountCredential = (Get-Credential -Message 'Enter the SQL service account credential'),
    [PSCredential] $AgentServiceAccountCredential = (Get-Credential -Message 'Enter the SQL Server Agent service account credential')
)
