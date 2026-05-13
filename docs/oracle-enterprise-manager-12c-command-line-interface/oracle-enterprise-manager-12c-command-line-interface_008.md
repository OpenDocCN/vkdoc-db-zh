# 第一章

![image](img/frontdot.jpg)

## 架构

Oracle Enterprise Manager 12c 提供了一个可扩展且可靠的中央存储库、一个控制台以及用于管理所有 Oracle 产品的服务。用户通常通过 OEM 控制台与 OEM 交互，该控制台具有丰富的直观图形界面。

Enterprise Manager 命令行界面 (EM CLI) 提供了在控制台之外访问 OEM 系统功能的途径。例如，作为其有用性的一个例证，交互式 EM CLI 任务可以取代定义 EM 管理员账户和角色时冗长的点击流。EM CLI 交互命令可以在 shell 脚本中使用，或者通过 CLI 自身的 Jython 脚本模式进行 CLI 调用。

本书探讨了您可以应用这些技术来简化和自动化 Oracle 环境中的任务的不同方式。

### 企业管理器框架

Oracle Enterprise Manager 应用程序作为 JEE 应用程序，在 WebLogic 服务器上的 WebLogic Server J2EE 域中运行。这种组合被称为 Oracle Management Server，或 OMS。

在 OMS 上运行的 Java 进程收集并处理来自您远程主机上的 EM 代理的 XML 文件上传。该信息被发布到存储库数据库，并存储在 SYSMAN 模式中。

当您在 OEM 控制台上查看页面时，数据是从存储库数据库中组装以供呈现的。同样，您从控制台发出的命令通过 OMS 进行处理，以更新存储库信息（例如，指标收集或通知），或通过调用 EM 代理或通过到远程数据库或主机的经过身份验证的连接来操纵受管目标。

控制台发出的每个命令都会执行一个 Java 程序。控制台不仅征集和组装数据，还提供这些例程执行所需的输入命令。大部分操作和查询代码库都可以通过 EM CLI 访问。

EM CLI 程序本身是一个轻量级的 Java 程序，它执行与控制台页面相同的活动，但使用作为命令行输入传递的值立即执行 OMS 模块；它通常用于 shell 脚本或 Jython 程序中。

### EM CLI 动词

接口命令被称为 `动词`。每个动词执行单个任务，并以合理的反馈成功返回，或者返回快速且明显的失败消息。

许多动词需要在命令行上提供输入值。与 PL/SQL 包类似，您的输入必须使用非常特定的语法传递给 OMS。这些值前面总是一个过滤器关键字，并且大多数输入要求您的字符串用双引号括起来。

![Image](img/sq.jpg) `注意` 作者们对引号的使用经验不一。推荐使用，但通常不是必需的。为了清晰起见，我们将在示例中使用它们。您可能会发现并不总是需要它们，或者您更倾向于不使用它们。

使用 `get_targets` 动词显示或捕获环境中目标的列表，如下所示：

```
emcli get_targets
```

要仅查找 Oracle 数据库目标，您需要使用 `targets` 关键字过滤请求：

```
emcli get_targets -targets="oracle_database"
```

本书中的众多示例展示了动词和输入值如何应用。EM CLI 动词及其语法的目录可在 Oracle 支持文档 E17786-x 中找到。请注意，有些动词与需要许可费的管理包相关联。您还可以在命令行上找到在线帮助，列出所有可用的动词及其预期用途。例如：

```
emcli help
Summary of commands:

argfile    -- Execute emcli verbs from a file
    help       -- Get help for emcli verbs (Usage: emcli help [verb_name])
    login      -- Login to the EM Management Server (OMS)
    logout     -- Logout from the EM Management Server
    setup      -- Setup emcli to work with an EM Management Server
    status     -- List emcli configuration details
    sync       -- Synchronize with the EM Management Server
    version    -- List emcli verb versions or the emcli client version

Add Host Verbs
    continue_add_host             -- Continue a failed Add Host session
    get_add_host_status           -- Displays the latest status of an Add Host session.
    list_add_host_platforms       -- Lists the platforms on which the Add Host operation can be performed.
    list_add_host_sessions        -- Lists all the Add Host sessions.
    retry_add_host                -- Retry a failed Add Host session
    submit_add_host               -- Submits an Add Host session.
...
```

`help` 动词可以用特定动词过滤以显示详细的使用说明：

```
emcli help get_targets
  emcli get_targets
        [-targets="[name1:]type1;[name2:]type2;..."]
        [-alerts]
        [-noheader]
        [-script | -format=
                           [name:<pretty|script|csv>];
                           [column_separator:"column_sep_string"];
                           [row_separator:"row_sep_string"];
        ]
        [-config_search="Configuration Search UI Name"]
        [-unmanaged]

Description:
    Obtain status and alert information for targets.

Options:
    -targets=name:type
        Name or type can be either a full value or a pattern match
        using "%". Also, name is optional, so the type may be
        specified alone.
     -config_search="Configuration Search UI Name"
         Search UI Name should be the display name of the configuration search.
     -alerts
        Shows the count of critical and warning alerts for each target.
    -noheader
        Display tabular output without column headers.
    -script
        This option is equivalent to -format="name:script".
    -format
        Format specification (default is -format="name:pretty").
        -format="name:pretty" prints the output table
            in a readable format but is not intended to be parsed by scripts.
        -format="name:script" sets the default column separator
            to a tab and the default row separator to a newline.
            The column and row separator strings may be specified
            to change these defaults.
        -format="name:csv" sets the column separator to a comma
            and the row separator to a newline.
    -unmanaged
        Get unmanaged targets (no status or alert information)
```


