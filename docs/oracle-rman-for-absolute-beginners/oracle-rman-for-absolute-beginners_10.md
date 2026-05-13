# 在 Windows 上

在 Windows 上，操作系统用户必须是 `ora_dba` 组或 `ora_oper` 组的成员。在 Windows 环境中，您可以按照以下步骤验证哪些操作系统用户属于 `ora_dba` 组：选择控制面板 ![image](img/arrow.jpg) 管理工具 ![image](img/arrow.jpg) 计算机管理 ![image](img/arrow.jpg) 本地用户和组 ![image](img/arrow.jpg) 组。您应该能看到一个名为 `ora_dba` 的组。您可以单击该组并查看分配了哪些操作系统用户。此外，要在 Windows 环境中使操作系统认证正常工作，您的 `sqlnet.ora` 文件中必须包含以下条目：`SQLNET.AUTHENTICATION_SERVICES=(NTS)`。

