# EM CLI 使用指南与示例

## 输出列
`Status ID` `Status` `Target Type` `Target Name` `Critical` `Warning`

## 示例

```
emcli get_targets
```
显示所有目标。`Critical` 和 `Warning` 列不显示。

```
emcli get_targets
          -alerts
```
显示所有目标。`Critical` 和 `Warning` 列显示。

```
emcli get_targets
          -targets="oracle_database"
```
显示所有 `oracle_database` 目标。

```
emcli get_targets
          -targets="%oracle%"
```
显示所有类型包含字符串 `"oracle"` 的目标。

```
emcli get_targets
          -targets="database%:%oracle%"
```
显示所有名称以 `"database"` 开头且类型包含 `"oracle"` 的目标。

```
emcli get_targets
          -targets="database3:oracle_database"
          -alerts
```
显示名为 `"database3"` 的 Oracle 数据库的状态和告警信息。

```
emcli get_targets
          -config_search="Search File Systems on Hosts"
          -targets="oracle%:host"
          -alerts
```
显示来自名为 `"Search File Systems on Hosts"` 的配置搜索结果中，名称以 `"oracle"` 开头且类型为 `"host"` 的目标的状态和告警信息。

```
emcli get_targets
          -targets="host"
          -unmanaged
```
显示未受管的主机目标的名称和类型信息。

## EM CLI 客户端软件

管理服务器上的基本 OEM 安装会预配置一个作为 `OMS Oracle Home` 一部分的 EM CLI 客户端。在第 2 章中，我们将向您展示如何将该客户端升级到 EM CLI 高级套件。

EM CLI 的部分优势源于其灵活性。除了在 OMS 服务器上安装客户端外，您还可以在非 OMS 主机甚至桌面上安装 EM CLI 客户端。

安装 EM CLI 客户端包括下载并提取一个安装 `jar` 文件以安装二进制文件，然后使用连接信息配置客户端以连接到您的 OMS 服务器。该 `jar` 文件及其使用安装程序可通过 OEM 控制台中的 `Setup > Command-Line Interface` 获取。请按照该网页上的说明将 EM CLI 安装到您的工作站。

## EM CLI 与 EMCTL

一些 EM CLI 功能可以通过代理控制实用程序 `EMCTL` 来执行。您的技术选择取决于多种因素的组合。

*   当通过远程主机上的 shell 脚本调用时，EM CLI 客户端必须手动安装并维护在远程主机上。控制台会显示远程 CLI 客户端安装列表，但您仍然必须手动更新客户端软件。
*   远程主机上的 EM CLI 配置需要用于与 OMS 服务器交互的连接信息。当此信息更改时，您必须访问每个 EM CLI 安装。
*   `EMCTL` 命令特定于特定代理所知的目标，因此在命令行上传递的命令通常要简单得多。
*   您必须登录到远程主机才能执行 `EMCTL` 命令。EM CLI 允许您远程执行许多 `EMCTL` 等效命令，以避免前往服务器。这在一次会话中管理多个服务器时特别有用。

我们建议您刚开始使用或如果您的安装规模较小时使用 `EMCTL`。`EMCTL` 中的命令往往更简单，事先的设置也更简单。然而，从长远来看，使用 EM CLI 所投入的“时间成本”是值得的。那些需要支持大型基础设施的人会发现自己倾向于使用 EM CLI。

## 代理启动与停止

`EM` 代理可以从 `OEM` 控制台内部、通过 `EM CLI`，当然也可以通过 `EMCTL` 来启动和停止。由于 `EMCTL` 命令是针对单个代理执行的，因此命令通常非常简单：

```
emctl start agent

emctl stop agent
```

类似的功能可以从管理服务器、您的桌面或任何安装了 EM CLI 客户端的主机执行。可移植性带来了复杂性，因为您不仅需要指定要控制的代理，还需要指定要使用的凭据。

您可以指定主机用户名、命名凭据或凭据集。当您传递用户名时，还必须提供密码。在纯交互模式下，可以提示您输入密码，但在 shell 脚本中使用此技术可能会将密码暴露给其他操作系统用户。使用 OEM 命名凭据可以避免此问题：

```
emcli start_agent –agent_name="dbservera:3872" –host_username="oracle" –host_pwd="Souper_53cre3t"
emcli start_agent –agent_name="dbservera:3872" –credential_name="oraprod"
emcli start_agent –agent_name="dbservera:3872" –credential_setname="HostCreds"
```

停止命令需要相同的条件值；例如：

```
emcli stop_agent –agent_name="dbservera:3872" –host_username="oracle"
```

我们将在第 4 章中更深入地探讨其中一些选项。

## 集中管理

也许您已决定在物理服务器迁移期间或操作系统修补期间关闭部分 EM 代理。您可以使用 `emcli get_targets` 命令快速构建类型为 `oracle_emd` 的受影响代理列表，并将该列表转换为两个 CLI 参数文件（argfiles）——一个用于停止代理，另一个用于启动它们。

![Image](img/sq.jpg) **注意** 参数文件用于批量处理 CLI 命令。它们在第 5 章中有更详细的讨论。

以下是一个使用参数文件的示例：

```
touch argfile_stop.lst
touch argfile_start.lst
emcli get_targets | grep oracle_emd > workfile.lst
for thisAGENT in `cat workfile.lst`; do
          echo "start_agent –agent_name=${thisAGENT} –credential_name=oraprod" > argfile_start.lst
          echo "stop_agent –agent_name=${thisAGENT} –credential_name=oraprod" > argfile_stop.lst
done

emcli login –user="SYSMAN"
emcli sync
emcli argfile ./argfile_stop.lst
logout
```

## 访问控制

较大的 Oracle 环境可能在 OEM 管理员和常规 DBA 员工之间存在职责分离，或者您的安全规则可能使得难以访问服务器进行例行维护。在这些情况下，运行 CLI 命令或通过控制台管理上下线是有意义的。

## 安全保护

在 OEM 控制台中，每当您请求执行危险任务时，系统都会提示您进行确认。EM CLI 没有相同的功能。当您给出命令时，您的任务会按照您的请求精确执行，因此在删除或修改目标时要格外小心。尽管如此，许多人更喜欢命令行，因为它操作直接，没有多余的反馈。只需小心即可。

