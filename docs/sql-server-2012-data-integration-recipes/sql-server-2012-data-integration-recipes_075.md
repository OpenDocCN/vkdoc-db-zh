# 导出与导入系统和用户 DSN

系统和用户`DSN`存储在注册表中。虽然这使得它们易于使用`ODBC 管理工具`进行调整，但也使得部署变得稍微复杂一些。

有一个特定的注册表项：—`HKEY_LOCAL_MACHINE\SOFTWARE\ODBC\ODBC.INI`—其中包含了您机器上定义的系统`DSN`源的完整列表。您可以使用`regedit`（或类似实用程序）导出注册表项，然后根据需要复制。这避免了手动重新生成`DSN`。操作如下：

1.  打开注册表编辑器。
2.  导航到`ODBC`文件夹（位于`HKEY_LOCAL_MACHINE` > `SOFTWARE`下）。
3.  以合适的名称导出一个`.reg`文件。
4.  使用`regedit.exe`——或通过双击`.reg`文件，将该`.reg`文件导入到目标服务器的注册表中。
5.  重启目标服务器。

用户`DSN`位于`HKEY_CURRENT_USER\Software\ODBC\ODBC.INI`中。

假设您已经创建了一个系统或用户`DSN`，那么值得研究一下`SSIS`如何使用它们。

![image](img/sq.jpg) **注意** 使用注册表编辑器时要极其小心，确保不会对注册表进行任何不必要或危险的更改。

