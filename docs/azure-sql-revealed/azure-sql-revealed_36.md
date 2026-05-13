# xp_cmdshell

`xp_cmdshell` 是 SQL Server 中一个系统支持的扩展存储过程 (`xproc`)。此 `xproc` 作为 `DLL` 在 SQL Server 的进程空间中运行，但会在 SQL Server 计算机上创建一个新进程来执行你作为参数提供的命令。尽管多年来这一直是 SQL Server 的一个便捷功能，但一些管理员会禁用此过程，因为它被认为是一个安全漏洞。因此，我们在 Azure SQL 中不支持 `xp_cmdshell`。

在 Azure SQL 中，确实没有任何等效功能可以从由引擎调用的 T-SQL 过程中运行命令。你需要将此代码从服务器端编程逻辑中移出，放到脚本或应用程序中。

