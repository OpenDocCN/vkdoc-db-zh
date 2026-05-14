# 第五章 SQL Server 工具

##### 执行包

选择“新建”按钮，保持格式的“分隔符”默认设置，然后点击“确定”。系统将显示一个窗口，让你输入文件名（这将是在 Linux 服务器上创建的新文件的名称，默认情况下是执行 `dtexec` 的目录）。

我输入了文件名 `people.txt` 并勾选了“Unicode”复选框。

现在在“平面文件目标编辑器”页面上选择“映射”选项。编辑器将自动将列名映射到文件的字段名。在此屏幕上点击“确定”。完成此操作后，屏幕如图 5-48 所示。

*图 5-48. 包含源和目标的数据流任务*

在将包复制到 Linux 服务器之前，我还有最后一步。默认情况下，包中的敏感信息（如密码和用户名）是加密的。出于本示例的目的，我将更改包的属性，通过密码对其进行保护。

当你点击设计器画布时，工具右下角应该会显示一个“属性”窗格。它可能当前显示的是某个数据流任务。如果是这样，我点击了该任务旁边的向下箭头按钮并选择了“包”。我向下滚动属性列表，将 `ProtectionLevel` 更改为 `EncryptSensitiveWithPassword`，并在 `PackagePassword` 字段中提供了一个密码。以后每次我想打开或执行这个包时，包括在 Linux 上执行时，都需要用到这个密码。

##### 执行包

要执行此包，我需要将其从 Windows 计算机复制到 Linux 服务器。我可以方便地使用 MobaXterm 或 `scp` 或 `winscp` 之类的程序来完成此操作。

SSDT 生成的包文件默认存储在 `<用户>\Projects` 目录中，因此我使用该工具将包（`.dtsx` 文件）保存到我计算机上类似 `c:\temp` 的位置。然后，我将文件 (`Package.dtsx`) 复制到了我的 Linux 服务器主目录中。

现在我准备使用 `dtexec` 程序来执行包，该程序在你 Linux 上安装 `mssql-server-is` 包时会一同安装。

**注意：** 要执行此包，必须首先在 Linux 上安装用于 SQL Server 的 Microsoft ODBC Driver 17。如果你已按照步骤安装了 SQL Server 工具（`sqlcmd`，……），那么此驱动应该已经安装好了。

以下是运行 `dtexec` 来执行名为 `Package.dtsx` 的包文件的命令：

```
dtexec /F Package.dtsx /DE <包密码>
```

*图 5-49. 在 Linux 上执行 dtexec*

文件 `people.txt` 已在我的主目录中创建。你还可以检查为此包创建的 `Package.dtsx` 文件的 XML 详细信息。

## 深入了解 SSIS

要了解更多关于 Linux 上 SSIS 的信息，包括其功能和限制，请参阅我们的文档：[`docs.microsoft.com/sql/linux/sql-server-linux-migrate-ssis`](https://docs.microsoft.com/sql/linux/sql-server-linux-migrate-ssis)。

要全面了解 SSIS，请参阅我们的主文档页面：[`docs.microsoft.com/sql/integration-services/sql-server-integration-services`](https://docs.microsoft.com/sql/integration-services/sql-server-integration-services)。

要了解 SSIS 的典型使用场景，请参阅我们的文档页面：[`msdn.microsoft.com/en-us/library/ms137795(v=sql.105).aspx`](https://msdn.microsoft.com/en-us/library/ms137795(v=sql.105).aspx)。

#### 总结

在本章中，我向你展示了 SQL Server 引擎内置的一系列令人惊叹的工具、程序和功能。你现在已能够使用 SQL Server 的其他功能，无论是启用 SQL Server 的功能以实现最大……



