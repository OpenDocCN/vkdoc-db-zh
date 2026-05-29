
+   [Oracle Enterprise Manager 12c 命令行界面](README.md)
+   [Oracle Enterprise Manager 12c 命令行界面](oracle-enterprise-manager-12c-command-line-interface_001.md)
+   [第三章：术语与基础](oracle-enterprise-manager-12c-command-line-interface_002.md)
+   [第四章：命令行操作](oracle-enterprise-manager-12c-command-line-interface_003.md)
+   [第五章：通过 `Shell` 脚本实现自动化](oracle-enterprise-manager-12c-command-line-interface_004.md)
+   [第六章：高级脚本](oracle-enterprise-manager-12c-command-line-interface_005.md)
+   [关于作者](oracle-enterprise-manager-12c-command-line-interface_006.md)
+   [关于技术评审者](oracle-enterprise-manager-12c-command-line-interface_007.md)
+   [第一章](oracle-enterprise-manager-12c-command-line-interface_008.md)
+   [EM CLI 使用指南与示例](oracle-enterprise-manager-12c-command-line-interface_009.md)
+   [第 2 章](oracle-enterprise-manager-12c-command-line-interface_010.md)
+   [WebLogic 服务器配置要求与概览](oracle-enterprise-manager-12c-command-line-interface_011.md)
+   [客户端或远程目标安装](oracle-enterprise-manager-12c-command-line-interface_012.md)
+   [第 3 章](oracle-enterprise-manager-12c-command-line-interface_013.md)
+   [Current status of the job is EXPIRED.](oracle-enterprise-manager-12c-command-line-interface_014.md)
+   [Credential Usage: defaultHostCred](oracle-enterprise-manager-12c-command-line-interface_015.md)
+   [Description:](oracle-enterprise-manager-12c-command-line-interface_016.md)
+   [Description: (Optional) Comma separated list of parameters to the command.](oracle-enterprise-manager-12c-command-line-interface_017.md)
+   [Description: (Optional) Command to run on the target.](oracle-enterprise-manager-12c-command-line-interface_018.md)
+   [Current status of the job is EXPIRED.](oracle-enterprise-manager-12c-command-line-interface_019.md)
+   [Credential Usage: defaultHostCred](oracle-enterprise-manager-12c-command-line-interface_020.md)
+   [Description:](oracle-enterprise-manager-12c-command-line-interface_021.md)
+   [Description: (Optional) Comma separated list of parameters to the command.](oracle-enterprise-manager-12c-command-line-interface_022.md)
+   [Description: (Optional) Command to run on the target.](oracle-enterprise-manager-12c-command-line-interface_023.md)
+   [Current status of the job is ACTIVE.](oracle-enterprise-manager-12c-command-line-interface_024.md)
+   [Credential Usage: defaultHostCred](oracle-enterprise-manager-12c-command-line-interface_025.md)
+   [Description:](oracle-enterprise-manager-12c-command-line-interface_026.md)
+   [Description: (Optional) Comma separated list of parameters to the command.](oracle-enterprise-manager-12c-command-line-interface_027.md)
+   [Description: (Optional) Command to run on the target.](oracle-enterprise-manager-12c-command-line-interface_028.md)
+   [----------------------------------------------------](oracle-enterprise-manager-12c-command-line-interface_029.md)
+   [Functions](oracle-enterprise-manager-12c-command-line-interface_030.md)
+   [----------------------------------------------------](oracle-enterprise-manager-12c-command-line-interface_031.md)
+   [----------------------------------------------------](oracle-enterprise-manager-12c-command-line-interface_032.md)
+   [Run-time Procedure](oracle-enterprise-manager-12c-command-line-interface_033.md)
+   [----------------------------------------------------](oracle-enterprise-manager-12c-command-line-interface_034.md)
+   [=====================================================](oracle-enterprise-manager-12c-command-line-interface_035.md)
+   [File:        create_blackout.sh](oracle-enterprise-manager-12c-command-line-interface_036.md)
+   [Purpose:     Create and initiate an OEM blackout](oracle-enterprise-manager-12c-command-line-interface_037.md)
+   [Parameters:  Blackout name as BO_NAME](oracle-enterprise-manager-12c-command-line-interface_038.md)
+   [=====================================================](oracle-enterprise-manager-12c-command-line-interface_039.md)



## 脚本文件说明

### `end_blackout.sh`
- **文件**: `end_blackout.sh`
- **目的**: 结束一个 OEM 黑屏。
- **参数**: 黑屏名称，作为 `BO_NAME`。

### `restore_training.sh`
- **文件**: `restore_training.sh`
- **目的**: 从数据文件的冷备份副本刷新数据库。
- **参数**: 数据库名称，作为 `ORASID`。

### `emcli_functions.lib`
- **文件**: `emcli_functions.lib`
- **目的**: EM CLI 脚本的共享函数库。
- **注意**: 引入此库以将其内容加载到内存中。

### `clone_em_users.sh`
- **文件**: `clone_em_users.sh`
- **目的**: 在实例之间复制 OEM 角色和用户账户。
- **参数**: 无。

### `change_admin_depts.sh`
- **文件**: `change_admin_depts.sh`
- **目的**: 更改 EM 管理员的部门编号引用。
- **参数**: 无。

## `emcli_functions.lib` 详细说明

- **文件**: `emcli_functions.lib`
- **目的**: 用于常见 `emcli` 任务的 Shell 脚本函数。
- **文件权限**: 此文件被引入但从不直接执行。
- **Echo 语句**: 为此脚本示例移除了 `echo` 语句的格式化，以保持逻辑清晰。请善待你的用户，在你自己的副本中自由添加空格。

`SysmanPassword` 变量仅在一处用于调用你的专有密码加密解决方案，或在一处记录 `SYSMAN` 密码以供你的脚本和本库中的函数使用。强烈建议你实施更安全的解决方案，例如通过 Java 或其他工具进行加密/解密。


# 通用函数

`CleanUpFiles` 在每个脚本开始时被调用，以确保移除旧的临时工作文件副本，从而全新开始。`CleanUpFiles` 也会被 `ExitCleanly` 函数调用，以在脚本执行完成后清理。

## 文件

`ExitCleanly` 在脚本被指示退出时调用，无论脚本是成功还是失败结束。干净的服务器是快乐的服务器。

如 `echo` 语句所示，`GetAgentNames` 可用于列出 OEM 已知的所有 `oracle_emd` 目标类型的目标。

任何脚本或函数都应使用此函数来查找数据库的确切名称。如果数据库不为 OEM 所知，它会返回一条清晰的失败消息。你可以轻松调整 `GetTargetName` 来查找主机或代理名称。

调用脚本必须将数据库名作为变量 `thisSID` 传递。

# 维护期函数

此函数在第 5 章中作为在数据库目标上创建为期六小时的维护期的方法进行了说明。


+   [请为你的环境调整时区。](oracle-enterprise-manager-12c-command-line-interface_106.md)

+   [考虑为主机或 `oracle_sys` 目标编写类似的函数](oracle-enterprise-manager-12c-command-line-interface_107.md)

+   [调用脚本必须提供一个维护期名称作为 `BO_NAME`，以及](oracle-enterprise-manager-12c-command-line-interface_109.md)

+   [数据库目标名称作为 `thisSID`](oracle-enterprise-manager-12c-command-line-interface_110.md)

+   [此函数在第 5 章中作为停止和删除 OEM 黑名单的方法进行过演示。](oracle-enterprise-manager-12c-command-line-interface_113.md)

+   [调用脚本必须提供一个黑名单名称作为 `BO_NAME`](oracle-enterprise-manager-12c-command-line-interface_115.md)

+   [此函数不需要输入值，用于重新同步所有处于“无法访问”状态的代理。](oracle-enterprise-manager-12c-command-line-interface_118.md)

+   [可能存在一些底层条件，导致单个代理无法重新同步，因此在此函数执行后请检查代理状态。](oracle-enterprise-manager-12c-command-line-interface_119.md)

+   [为每个代理只尝试一次重新同步，以防止在遇到问题代理时陷入无限循环。](oracle-enterprise-manager-12c-command-line-interface_120.md)

+   [此函数和 `SecureAllAgents` 设计为通过从命令行源化（source）此库文件，然后按名称调用函数来执行。](oracle-enterprise-manager-12c-command-line-interface_121.md)

+   [示例：](oracle-enterprise-manager-12c-command-line-interface_122.md)

    ```
    . emcli_functions.lib
    ResyncUnreachableAgents
    . emcli_functions.lib
    SecureAllAgents
    ```

+   [顾名思义，此函数将使用其主机的首选凭据来保护您的所有代理。](oracle-enterprise-manager-12c-command-line-interface_129.md)

+   [此函数将设置常规数据库首选凭据，使用此文件顶部定义的 `SYSMAN` 密码。](oracle-enterprise-manager-12c-command-line-interface_132.md)

+   [调用脚本必须提供数据库名称作为 `thisSID`](oracle-enterprise-manager-12c-command-line-interface_134.md)

+   [此函数将从命令行执行 `SetDBUserCredential`。](oracle-enterprise-manager-12c-command-line-interface_137.md)



您可以使用 `read` 语句来补充失败逻辑，以提示输入数据库名称。

此脚本为 Korn shell 编写。您必须调整 `read` 语句以用于 Bash。

**文件：`emcli_start_blackout.ksh`**
**目的：** 创建并启动一个 OEM 黑名单
**参数：** 数据库名称

输入您的 OEM 脚本存储库路径。

`WORKFILE` 是函数库清理的几个文件名变量之一。

源化函数库。

验证是否在命令行上传递了 `SID`。

**文件：`emcli_stop_blackout.ksh`**
**目的：** 停止并删除一个已命名的 OEM 黑名单
**参数：** 数据库名称

输入您的 OEM 脚本存储库路径。

`WORKFILE` 是函数库清理的几个文件名变量之一。

源化函数库。

验证命令行中是否传入了 `SID`。

**示例脚本：`emcli_relocate_target_vcs.ksh`**

**文件：`emcli_relocate_target_vcs.sh`**
**目的：** 为受监控的 VCS 目标重新定位 OEM 代理
**参数：** 需要 VCS 组名
**注意：** 必须作为 VCS 启动的一部分运行，在故障转移到备用主机之后执行

环境测试。



## 变量部分

将 `ORACLE_HOME` 设置为代理的基础目录。

EM 代理自带 jdk 安装，因此在本脚本中引用它。

显示的设置假设 EM CLI 二进制文件位于代理的 Oracle 主目录中。

设置路径以匹配上面设定的值。

在此设置可执行文件的完全限定路径。

AIX 环境所需的设置。

文件名变量 - 保持与 `CleanUpFiles` 函数中的顺序一致。

列表将从每个列出的源收集，并整理到最终的 EM CLI 脚本中。

`emcli relocate_targets` 动词的语法对于数据库和监听器略有不同，因此我们将它们收集到不同的目标列表中以进行处理。

为 VCS 日志文件添加时间戳。

收集集群成员的 VCS 值。

通过检查 `OFFLINE` 模式下是否存在节点，测试 VCS 是否识别您在命令行传递的集群名称。

根据这个信息，我们可以获取 `HAGROUP` 的名称，然后根据 VCS 故障切换后的状态，将离线和在线节点与相应的变量关联起来。

为日志文件回显这些值。

选择与 `HAGROUP` 关联的数据库和监听器名称。

此列表可能与这些主机上 `OEM` 所知的目标列表不匹配，因此我们将在脚本后面进行比较。

获取两个节点的 `OEM` 代理名称（包括端口）。

代理总是安装在静态文件系统上，从不进行故障切换。

此字符串将在构建 `emcli` 脚本时多次使用。

监听器执行相同的过程。

## 脚本设置

脚本名称  : `backup_oms_configs.ksh`

调用参数 : 无

目的      :   备份关键的 OMS 配置文件

## 独立变量

脚本设置目录名，然后列出每个需要备份的文件。



+   [这些函数将设置为 DIRNAME 和 FILENAME 的值，](oracle-enterprise-manager-12c-command-line-interface_208.md)
+   [转换为用于处理的源文件和目标文件名。](oracle-enterprise-manager-12c-command-line-interface_209.md)
+   [脚本名称  : log_blaster_oms.ksh](oracle-enterprise-manager-12c-command-line-interface_232.md)



## 调用参数
无

## 目的
清理 OMS 的日志文件

## 参考来源
MOS Note 1445743.1

## 独立变量
将这些变量设置为你的本地目录设置。

`EM_INSTANCE_HOME` 通常是 `MIDDLEWARE_HOME/gc_inst`。

`webtier` 目录名从环境中获取，通常发现的值是 `OMS1`，但也可能不同。

此目录的保留期通过 `thisAGE` 变量设置为七天。

应用于目标名称的正则表达式，用于筛选目标列表：
- `'.*'` 包含所有目标。
- `'^orcl'` 包含所有以 `'orcl'` 开头的目标。
- `'^em12cr3\.example\.com$'` 包含名称精确为 `'em12cr3.example.com'` 的目标。

包含要应用于目标列表的属性名称和值的字典对象。

启用调试时，命令将被打印但不会执行。

分隔符应使用不会出现在任何目标名称中的字符。

## 执行步骤
1. 编译目标名称筛选器。
2. 从 OMS 检索完整的目标列表。
3. 循环遍历完整的目标列表。
4. 如果 `debug` 不为 `True`，则打印消息并执行命令。



+   [创建类的一个实例。](oracle-enterprise-manager-12c-command-line-interface_268.md)
+   [设置要应用的单个属性。](oracle-enterprise-manager-12c-command-line-interface_269.md)
+   [仅包含名称完全为 `em12cr3.example.com` 的主机目标。](oracle-enterprise-manager-12c-command-line-interface_270.md)
+   [将当前定义的目标列表更新到当前定义的属性。](oracle-enterprise-manager-12c-command-line-interface_271.md)
+   [设置要应用的多个属性。](oracle-enterprise-manager-12c-command-line-interface_272.md)
+   [显示当前定义的要应用的属性。](oracle-enterprise-manager-12c-command-line-interface_273.md)
+   [仅包含以 `em12cr3.example.com` 开头的主机或代理目标。](oracle-enterprise-manager-12c-command-line-interface_274.md)
+   [仅包含代理目标。](oracle-enterprise-manager-12c-command-line-interface_275.md)
+   [仅包含由 `em12cr3.example.com:3872` 代理管理的目标。](oracle-enterprise-manager-12c-command-line-interface_276.md)
+   [仅包含以 `EM` 开头、以 `Service` 结尾，并且在 `EM` 和 `Service` 之间只有一个单词的主机目标。](oracle-enterprise-manager-12c-command-line-interface_277.md)
+   [在单个命令中创建类的一个实例，定义要应用的属性，并指定目标列表筛选器。](oracle-enterprise-manager-12c-command-line-interface_278.md)
+   [清除所有筛选器，将实例重置为包含所有目标。](oracle-enterprise-manager-12c-command-line-interface_279.md)
+   [显示 `myinst` 实例当前定义的目标列表以及每个目标当前定义的属性。](oracle-enterprise-manager-12c-command-line-interface_280.md)
+   [将当前定义的目标列表更新到当前定义的属性，且不重新打印目标和目标属性列表。](oracle-enterprise-manager-12c-command-line-interface_281.md)
+   [使用类应用属性更新而不创建实例。](oracle-enterprise-manager-12c-command-line-interface_282.md)
