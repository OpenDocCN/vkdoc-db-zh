# 第 3 章

## 概要

对大多数读者而言，使用动词调用是 `EM CLI` 工作中最主要的部分。熟悉 `12.1.0.4` 版本中的重大动词变更至关重要，但在本书的剩余章节中，我们将围绕在企业管理器命令行界面中进行脚本编写的这一目标，来构建你的知识体系。

## 术语与基础

![image](img/frontdot.jpg)

像任何命令行工具一样，`EM CLI` 初看起来可能让人望而生畏。但如同技术领域的其他事物一样，人们对某样东西的工作原理理解得越透彻，它就越有用，使用起来也越容易。

`EM CLI` 的信息量太大，即使只是其中一小部分也难以全部记住。文档与工具本身同等重要。理解文档、内置手册以及工具本身所使用的术语，能够让你通过提出正确的问题来快速找到所需内容。例如，如果你要寻找创建用户的命令，你需要知道你是在寻找一个**动词**，而不是参数或变量。

一旦理解了 `EM CLI` 的术语，对其功能有基本的了解，就能立即开始执行基本任务，而真正的学习也由此开始。本章首先通过解释 `EM CLI` 与企业管理器的协作方式，然后提供一系列详细示例，来介绍其基础知识。

### 术语：动词

`EM CLI` 的功能使用**动词**来执行操作。动词是以用户命令形式呈现的任务或操作，它暴露了企业管理器的功能。理解动词及其如何用于完成任务，对于充分利用 `EM CLI` 至关重要。

动词可以包含一个或多个参数。所使用的**模式**决定了这些参数的形式。例如，在**命令行模式**下使用 `EM CLI` 时，参数是跟在动词后面的**位置参数**。每个参数前都有一个短横线，后跟一个等号。随后可以为该参数赋值，值通常用引号括起来。

有些参数是必需的，有些是可选的。文档通过将可选参数放在方括号中来标识它们。例如，`clear_privilege_delegation_setting` 动词有一个必需参数和两个可选参数（清单 3-1）。

**清单 3-1. clear_privilege_delegation_setting 动词的必需和可选参数**
```
emcli clear_privilege_delegation_setting
        -host_names="name1;name2;..."
        [-input_file="FILE:file_path"]
        [-force="yes/no"]
```

截至 `12.1.0.3` 版本，共有 `362` 个动词。为了更方便地帮助你查找和管理动词，它们被分为 `58` 个类别。这些类别代表了逻辑功能。例如，如果你要查找与修补相关的动词，你会在 *修补动词* 类别中找到它们。如果要查找与黑屏（blackouts）相关的动词，你会在 *黑屏动词* 类别中查找。

动词的命名方式使其功能一目了然。有些名称很短，如 `login` 和 `start_agent`；有些则非常长，如 `update_monitoring_creds_from_agent` 和 `list_target_privilege_delegation_settings`。这种命名法的好处在于，它使得动词相对容易记忆，并且便于根据功能来查找动词。

## 模式

`EM CLI` 可以通过三种不同模式之一调用。模式是使用 `EM CLI` 的方法。选择哪种模式取决于任务的目的和范围。

#### 命令行模式

在 `12.1.0.3` 版本之前，`EM CLI` 仅在一种模式下运行——命令行模式。在命令行模式下，每次调用 `emcli` 可执行文件，`EM CLI` 只执行一个动词。命令行模式下的 `EM CLI` 没有子 shell 或编程接口。任何 `EM CLI` 的脚本编写都需要通过能够调用单个 `EM CLI` 命令并处理其输出的 shell 来完成。

#### 交互模式

随着 `12.1.0.3` 版本的发布，`EM CLI` 现在包含了经典的命令行模式以及交互模式和脚本模式。交互模式允许将 `EM CLI` 作为所使用任何终端的子 shell 调用。当 `EM CLI` 在交互模式下运行时，提示符会更改为自定义的 Python 提示符：
```
cd $OMS_HOME/bin
./emcli
emcli>
```

#### 脚本模式

脚本模式没有子 shell 或提示符。实际上，它与命令行模式非常相似，区别在于其调用允许第一个参数是一个用 Python 兼容编程语言编写的脚本。然后，该脚本会被执行，就如同命令被输入到交互模式一样。脚本模式下的 `EM CLI` 只调用一次，但与命令行模式不同的是，Python 编程语言的全部功能都可以用来完成在命令行模式中需要 shell 来完成的一切工作。此外，脚本模式以原生形式调用 `EM CLI`，这使得解析命令输出等工作变得更加容易和高效，尤其是在使用 JSON 时。

### 帮助！

`EM CLI` 具有内置的帮助功能。获取命令行模式下可用动词的概览列表，只需使用动词 `help` 执行 `EM CLI` 即可。如果你没有使用额外的参数来限定命令，请准备好接收大量数据。如果你需要关于某个特定动词的更多信息，请将该动词作为第二个参数添加。

要在交互模式下使用 `help`，需要调用一个名为 `help()` 的 Python 函数。`help()` 函数只需要一个参数——帮助主题。帮助主题可以是一个动词函数，也可以是某些动词函数的参数。请注意，交互模式中包含的一些帮助主题在命令行模式中并不存在。例如，命令行模式使用设置文件来保存 `EM CLI` 属性，但交互模式和脚本模式则不使用。每次调用交互或脚本模式时，属性都会被设置。我们可以使用 `help()` 函数来查看设置属性时应使用哪些参数，如 清单 3-2 所示。

**清单 3-2. 客户端属性帮助页面**
```
emcli>help('client_properties')
EMCLI_OMS_URL             : 要连接到的 OMS URL。
EMCLI_USERNAME            : OMS 用户名。
EMCLI_AUTOLOGIN           : 可能的值为 true, false。默认为 false。
EMCLI_TRUSTALL            : 可能的值为 true, false。默认为 false。
EMCLI_VERBJAR_DIR         : 绑定目录的位置。
EMCLI_CERT_LOC            : 有效证书文件的位置。
EMCLI_LOG_LOC             : 日志文件存储的目录。
EMCLI_LOG_LEVEL           : 可能的值为 ALL, INFO, FINE, FINER, WARN, SEVERE。默认为 SEVERE。
EMCLI_OUTPUT_TYPE         : 可能的值为 json, JSON, text, TEXT。脚本模式下默认为 json，交互模式下默认为 text。
```

---

*   `create_assoc`: 创建目标关联实例
*   `delete_assoc`: 删除目标关联实例
*   `list_allowed_pairs`: 列出指定源和目标目标类型所允许的关联类型
*   `list_assoc`: 列出指定源目标和目标目标之间的关联
*   `manage_agent_partnership`: 覆盖企业管理器自动将伙伴代理分配给其他代理的默认行为。伙伴代理是指，除了其其他功能外，还被分配给另一个代理作为其“伙伴”的代理，以便远程监控该代理及其主机的可用性。此动词并非设计为常用。它是为了支持特殊情况而提供的，例如管理员可能希望明确分配代理伙伴关系，或排除某些代理成为伙伴，或排除某些代理被其他代理远程监控。

`status()` 将列出所有客户端属性的值。可以使用 `set_client_property(属性名, 值)` 和 `get_client_property(属性名)` 来设置和获取客户端属性。

## 理解错误代码

在使用 EM CLI 时，您可能会遇到多种错误代码。这些错误代码所提供的信息对于确定错误原因至关重要。下面显示的是最常见的一些错误代码；它们几乎总是有简单的解决方法。

命令行模式要求在执行任何命令之前必须先建立登录会话。在安全环境中以及默认情况下，登录会话将在 45 分钟后过期。当在命令行模式下，未建立有效登录会话却执行了有效命令时，系统会抛出一个退出代码为 255 的错误。要继续操作，需要使用 `login` 动词建立一个有效会话，如 清单 3-3 所示。

清单 3-3. 命令行模式下的登录错误消息和退出代码
```
[oracle oms]$ emcli get_targets
Error: Session expired. Run emcli login to establish a session.
[oracle oms]$ echo $?
```

交互模式和脚本模式要求在每次执行脚本或建立交互会话时都必须包含连接参数。如果未指定连接 URL，则会抛出一个错误，其中包含有关如何建立连接的详细信息，如 清单 3-4 所示。

清单 3-4. 交互模式和脚本模式下的设置错误
```
emcli>get_targets()
Error: EM URL is not set. Do set_client_property('EMCLI_OMS_URL', '<value>')
Or set it as environment variable.

[oracle ~]$ emcli @test.py
Traceback (most recent call last):
  File "test.py", line 1, in <module>
    get_targets()
  File "<string>", line 41, in get_targets
emcli.exception.VerbExecutionError: Error: EM URL is not set. Do set_client_property('EMCLI_OMS_URL', '<value>')
Or set it as environment variable.
``

错误可能由无数原因引起。EM CLI 的开发人员在使错误消息描述性强且实用方面做得非常出色。当您遇到错误时，请仔细阅读描述，以了解导致错误的问题是什么。如有疑问，请在文档或互联网上搜索该错误。很可能，其他人已经发现并修复了该错误的原因。

## 语法

EM CLI 程序可以三种不同的模式使用。它们是命令行模式、交互模式和脚本模式。在 12.1.0.2 版本之前，只有命令行模式。交互模式和脚本模式在 12.1.0.3 版本中可用，同时提供了一个使用 Jython 的公开编程接口。

标准的命令行模式自 Grid Control 10g 以来变化不大。除了少数例外，用于早期 EM CLI 版本的脚本在最新版本或 12c 中仍可继续使用。命令行语法是 `emcli` 命令，后跟一个动词，再后跟零个或多个动词选项或参数，如下例所示：
```
emcli create_group -name=my_group -add_targets="mymachine.example.com:hostname"
```

交互模式和脚本模式均使用 Jython 编程语言。交互模式是通过执行不带任何参数的 `emcli` 命令创建的单用户交互会话。

标准的 Linux Shell 命令在交互式 Shell 中不起作用。EM CLI 的交互模式可以使用 `exit()` 命令或在 Linux 中输入 `ctrl-d` 或在 Windows 中输入 `ctrl-z` 来关闭。

交互模式的语法与标准命令行模式不同。一旦建立了交互模式 Shell，就不再需要使用 `emcli` 命令。执行 EM CLI 命令是通过直接将动词放入 Shell 并将动词选项括在括号内来完成的。与命令行模式不同，动词选项前不加短横线，而是在后面跟有其他选项时前加逗号。以下展示了交互模式下带有 `name` 和 `add_targets` 选项的 `create_group` 函数：
```
emcli> create_group(name=my_group, add_targets="mymachine.example.com:hostname")
```

脚本模式使用与交互模式相同的命令语法，但不需要 Jython Shell。所有要执行的 EM CLI 命令都将格式化为上述交互 Shell 命令的形式，并放入脚本中，同时包含任何其他要加入的 Python 代码。该脚本通过在 `emcli` 命令后跟一个 `@` 符号和脚本名称来非交互式执行：
```
[oracle ~]$ emcli @my_script.py
```

## 设置

EM CLI 包含在每个 OMS 安装中，并设置为自动连接到该 OMS。可以使用 `setup` 动词确认设置配置。在命令行模式下运行该命令显示 EM CLI 配置为使用 `SYSMAN` 用户连接到 `https://oem.example.com:7802/em`，如 清单 3-5 所示。

清单 3-5. 命令行模式下 EM CLI 设置的输出
```
[oracle ~]$ emcli status
Oracle Enterprise Manager 12c EMCLI 12.1.0.3.0.
Copyright (c) 1996, 2013 Oracle Corporation and/or its affiliates. All rights reserved.

Instance Home          : /u01/app/oracle/product/12.1.0/gc_inst/em/EMGC_OMS1/sysman/emcli/setup/.emcli
Verb Jars Home         : null
Status                 : Configured
EMCLI Home             : /u01/app/oracle/product/12.1.0/mw_1/oms/bin
EMCLI Version          : 12.1.0.3.0
Java Home              : /u01/app/oracle/product/12.1.0/mw_1/jdk16/jdk/jre
Java Version           : 1.6.0_43
Log file               : /u01/app/oracle/product/12.1.0/gc_inst/em/EMGC_OMS1/sysman/emcli/setup/.emcli/.emcli.log
EM URL                 : https://oem.example.com:7802/em
EM user                : SYSMAN
Auto login             : false
Trust all certificates : true
```

在 OMS 服务器以外的系统上安装 EM CLI 后，必须先对其进行显式配置，它才能与 Enterprise Manager 安装配合使用。对尚未配置的 EM CLI 安装执行 `setup` 命令会返回错误，如 清单 3-6 所示。

清单 3-6. 尚未设置的 EM CLI 安装
```
[oracle ~]$ emcli setup
Oracle Enterprise Manager 12c 3.
Copyright (c) 1996, 2013 Oracle Corporation and/or its affiliates. All rights reserved.

No current OMS
```

配置通过 `setup` 动词后跟多个参数来完成。`url` 参数与在浏览器中登录 Enterprise Manager 所使用的地址相同。`dir` 参数指定安装客户端的目录。`username` 参数指示客户端将用于连接的用户名。`noautologin` 参数决定在建立 EM CLI 会话之前需要提供用户名和密码才能访问 OMS。`trustall` 参数表示将自动信任 OMS 上的证书。以下 EM CLI 设置命令中的 `dir` 参数使用 Shell 扩展来表示安装目录是当前工作目录：
```
[oracle ~]$ emcli -url=https://<hostname of grid server>:1159/em -dir=$(pwd) 
-username=<username> -noautologin -trustall
```


# EM CLI 和 EMCTL 使用指南

在交互模式下运行 `setup` 命令将显示 URL 和用户均未定义。这是因为无论是在交互模式还是脚本模式下，这些参数都需要每次都进行设置，如 清单 3-7 所示。

清单 3-7. 在交互模式下执行 `setup` 函数

```
emcli>status()
Oracle Enterprise Manager 12c EMCLI with Scripting option Version 12.1.0.3.0.
Copyright (c) 1996, 2013 Oracle Corporation and/or its affiliates. All rights reserved.

Verb Jars Home (EMCLI_VERBJAR_DIR)      : /u01/app/oracle/product/12.1.0/mw_1/oms/bin/bindings/default/.emcli
EMCLI Home (EMCLI_INSTALL_HOME)         : /u01/app/oracle/product/12.1.0/mw_1/oms/bin
EMCLI Version                           : 12.1.0.3.0
Java Home                               : /u01/app/oracle/product/12.1.0/mw_1/jdk16/jdk/jre
Java Version                            : 1.6.0_43
Log file (EMCLI_LOG_LOC)                : CONSOLE
Log level (EMCLI_LOG_LEVEL)             : SEVERE
EM URL (EMCLI_OMS_URL)                  : NOT SET
EM user (EMCLI_USERNAME)                : NOT SET
Auto login (EMCLI_AUTOLOGIN)            : false
Trust all certificates (EMCLI_TRUSTALL) : false
```

一旦设置完成并且通过身份验证连接到有效的 Enterprise Manager，EM CLI 就可以准备执行针对 EM 的任务。设置过程通常是发现问题的环节，尤其是在初始配置期间。例如，您可能不知道在安装 EM CLI 的服务器与 OMS 之间，EM 使用的端口被防火墙阻止了。设置过程将揭示这一事实。一旦建立了成功的登录连接，EM CLI 的设置过程就完成了。

## 通信方式

许多任务可以使用 EM CLI 或 EM CTL 来完成。选择使用哪种工具将取决于多种因素，例如哪种工具可用，或者哪种工具对特定任务更高效。工具与 Enterprise Manager 接口的方式也可能是另一个需要考虑的因素。

### EM CLI

EM CLI 是 Oracle Management Server (OMS) 的客户端，就像 SQL*Plus 是数据库的客户端一样。Enterprise Manager OMS 安装包含 EM CLI 的安装。EM CLI 可执行文件位于 OMS 主目录中，即 `<OMS_HOME>/bin/emctl`。EM CLI 使用 HTML 与 OMS 通信，使用与访问 Enterprise Manager 图形界面相同的端口和 URL。

在用于 EM CLI 和 Enterprise Manager 之间 HTTP 通信的端口上运行网络跟踪，会显示来自 EM CLI 的所有请求都是通过 HTTP 的 `GET` 和 `POST` 请求。这些请求与通过 GUI 控制台建立登录时发出的请求相同。

EM CLI 与 Enterprise Manager 接口的方式允许它从任何能够建立到 OMS IP 和端口的网络连接的位置进行连接，这使得在需要从托管 OMS 代理以外的服务器运行命令的情况下，EM CLI 相比 EM CTL 具有明显优势。

### EM CTL

EM CTL 与其安装的代理直接通信。EM CTL 只会与其自身的代理通信，即使同一服务器上运行着其他代理。

![Image](img/sq.jpg) `Note` 还有一个 EM CTL 接口可以直接控制和与 OMS 通信，它有一组完全不同的命令。

位于 `<AGENT_BASE>/agent_inst/bin` 的 `emctl` 命令是一个 shell 或批处理脚本，用于设置环境变量，并将传递给第一个命令的参数执行 `<AGENT_HOME>/bin/emctl` 命令。`<AGENT_HOME>/bin/emctl` 也是一个 shell 或批处理脚本，用于设置环境变量并来源许多其他文件。然后它使用传递给第一个命令的参数执行一个 perl 脚本。这个 perl 脚本会来源许多其他 perl 脚本，并读取现有的状态文件或直接与 java 代理进程交互以获取结果。

因为 EM CTL 直接与 java 代理进程交互，所以它不被视为客户端。这也意味着无法使用 EM CTL 远程控制代理。

### EM CTL 与 EM CLI 的比较

何时使用 EM CTL 还是 EM CLI 主要基于偏好和环境限制。例如，两条命令都可以控制 Enterprise Manager 中的维护窗口。在一个环境中，由于需要的 Java 版本或需要的端口访问，可能会限制远程或本地安装 EM CLI，这使得 EM CTL 成为更好的选择。在另一个环境中，可能对代理服务器上的 shell 访问有限制，这使得 EM CLI 成为更好的选择。了解 EM CTL 和 EM CLI 之间的架构差异将有助于在不同情况下决定使用哪一个。

EM CLI 的语法相对简单；`emcli` 后跟一个动词，然后是必需和可选的参数。EM CTL 的语法不那么容易理解，并且不遵循严格的格式。

对于大多数命令，格式如下：`emctl` 后跟一个动词，然后是 `agent`。对于某些配置命令，格式是：`emctl` 后跟 `config agent`，然后是一个动词，如 `listtargets` 或 `secure`。通常，找出要执行的 EM CTL 命令语法的最简单方法是在线查阅文档，或者通过不带任何参数执行 `emctl` 命令来打印帮助文本。帮助文本将打印到屏幕上。

可以通过将帮助文本重定向到可以打开和搜索的文件，或者通过将输出通过管道传输到可搜索的行读取器（如 `less`）来使此帮助文本可搜索：

```
emctl > help.txt; vim help.txt
emctl | less
```

既然我们已经回顾了许多重要的概念和定义，现在让我们看一些真实世界的示例，这些示例可用于完成许多人可能使用 EM CLI 来完成的最常见任务。

第一个任务，登录，是每个 EM CLI 会话都必须完成的，无论使用哪种模式。其他任务是在 GUI 不合适时使用 EM CLI 的示例，例如在计划脚本中或处理大量目标时。

## 任务：建立登录

如果没有与 OMS 的连接，EM CLI 几乎没有有用的功能。创建该连接通常是使用该接口的第一步。如何建立连接取决于所使用的模式。

当使用命令行模式时，如果在设置期间指定了 `-autologin` 参数，则不需要建立登录，因为此参数指示在 EM CLI 安装期间使用的凭据存储在安装文件中，并自动用于每个命令。使用 `login` 动词建立登录。`login` 动词后必须跟 `-username` 参数，并可选择性地跟 `-password` 参数。

不建议在命令行上包含密码，因为存在暴露密码的安全隐患。如果参数中未包含密码，EM CLI 将提示用户输入密码，并且输入的字符不会回显到屏幕上，在使用 `ps` 命令查看正在运行的进程时密码也不会暴露。如果在设置期间未指定 `-autologin` 参数，登录会话将提示输入密码：

```
[oracle ~]$ emcli login -username=sysman
Enter password :

Login successful
```

在以脚本或交互模式运行 EM CLI 的情况下，每次脚本执行时以及每个交互会话开始时都必须明确提供登录信息。即使使用了 `-autologin` 参数，这两种模式也不会使用在设置期间建立的凭据。



可以创建一个简单的脚本来建立连接，并在交互模式和脚本模式下使用。登录脚本需要指定用于连接的 URL、SSL 身份验证方式以及登录凭据。这两种模式都需要一个 Python 脚本（参见清单 3-8）。

### 创建登录脚本

清单 3-8. 用于交互模式和脚本模式的登录脚本

```
from emcli import *

set_client_property('EMCLI_OMS_URL', 'https://oem.example.com:7802/em')
set_client_property('EMCLI_TRUSTALL', 'true')

myLogin(username='sysman')

myLogin()
```

将此文本放入扩展名为`.py`的文件中，即可使其成为一个可以导入到`EM CLI`中的模块。例如，该脚本可以命名为`login.py`，并放置在`/home/oracle/scripts`目录下。然而，`EM CLI`不一定能找到该脚本，因为此路径不在默认的`Jython`路径中。明确定义`Jython`搜索路径可以确保`EM CLI`能够导入该搜索路径内的任何脚本。可以通过补充`JYTHONPATH`变量来添加额外的目录：

```
export JYTHONPATH=$JYTHONPATH:/home/oracle/scripts
```

类似于使用命令`from emcli import *`将`EM CLI`的`Jython`模块导入到脚本中一样，登录脚本也可以被导入到交互模式或脚本模式中。

### 在交互模式下建立登录

要在交互模式下建立登录，我们可以直接导入模块。然后，系统将提示我们输入脚本中指定用户的密码。清单 3-9 展示了在交互模式下使用登录脚本的示例。

清单 3-9. 使用登录模块在交互模式下建立登录

```
[oracle ~]$ emcli
Oracle Enterprise Manager 12c EMCLI with Scripting option Version 12.1.0.3.0.
Copyright (c) 1996, 2013 Oracle Corporation and/or its affiliates. All rights reserved.

Type help() for help and exit() to get out.

emcli>import login
Enter password :  *********

emcli>
```

### 在脚本模式下建立登录

要在脚本模式下建立登录，需要从正在执行的脚本内部导入模块。例如，以下脚本使用`EM CLI`函数打印企业管理器目标的总数。但首先，它需要通过导入`login`模块来建立登录，如清单 3-10 所示：

清单 3-10. 使用登录模块在脚本模式下建立登录

```
import emcli
import login

mytargets = str(len(emcli.get_targets().out()['data']))

print(mytargets)
```

## 任务：获取目标列表

查看和操作企业管理器目标是`EM CLI`最常见的用途。无论执行哪种操作，第一步都是检索目标信息。

### 检索所有目标信息

`get_targets`动词或函数用于检索目标列表以及许多其他列的信息。检索完整信息列表时，该动词和函数都不需要参数，如清单 3-11 所示。

清单 3-11. 使用命令行模式检索目标信息

```
[oracle ~]$ emcli login -username=sysman
Enter password :

Login successful

[oracle ~]$ emcli get_targets
Status  Status     Target Type     Target Name            ID
1       Up         host            oem.example.com
1       Up         oracle_emd      oem.example.com:3872
-9      n/a        oracle_home     common12c1_20_oem
```

### 筛选检索到的目标

可以通过指定以分号分隔的目标名称和目标类型列表（两者之间用冒号分隔）来筛选检索到的目标。目标列表筛选器中任何位置都不应有空格。清单 3-12 展示了创建包含两个目标的筛选器的示例。

清单 3-12. 使用`targets`参数限制`get_targets`检索的目标

```
[oracle ~]$ emcli get_targets \
   -targets='oem.example.com:host;oms12c1_3_oem:oracle_home'
Status  Status     Target Type     Target Name            ID
1       Up         host            oem.example.com
-9      n/a        oracle_home     oms12c1_3_oem
```

### 在筛选器中使用通配符

`targets`参数可以在目标名称或目标类型值中的任何位置使用通配符，并且对于任一值都可以使用多个通配符，如清单 3-13 所示。

清单 3-13. 在`targets`参数中使用通配符

```
[oracle ~]$ emcli get_targets \
   -targets='oem%:host;oms12c1_%_oem:%oracle%'
Status  Status     Target Type      Target Name           ID
1       Up         host             oem.example.com
-9      n/a        oracle_home      oms12c1_3_oem
```

### 在交互和脚本模式下使用`get_targets`

Python 函数需要特定的格式：`函数名，后跟左括号，然后是逗号分隔的参数及其值列表，最后是右括号`。

以下是典型 Python 函数的示例：

```
function_name(parameter1=value1, parameter2=value2)
```

由于交互模式和脚本模式中的`EM CLI`动词实际上是 Python 函数，因此它们也需要遵循此格式。

调用没有额外参数的函数只需要动词和一对括号。清单 3-14 展示了在交互模式下不带参数使用`get_targets`函数的结果，而清单 3-15 展示了在脚本模式下带`targets`参数的同一函数。

清单 3-14. 在交互模式下不带参数调用`get_targets()`函数

```
[oracle ~]$ emcli
Oracle Enterprise Manager 12c EMCLI with Scripting option Version 12.1.0.3.0.
Copyright (c) 1996, 2013 Oracle Corporation and/or its affiliates. All rights reserved.

Type help() for help and exit() to get out.

emcli>import login
Enter password :  *********

emcli>get_targets()
Status  Status     Target Type      Target Name
          ID
1       Up         host             oem.example.com
1       Up         oracle_emd       oem.example.com:3872
-9      n/a        oracle_home      common12c1_20_oem
```

清单 3-15. 在脚本模式下带`targets`参数调用`get_targets()`函数

```
[oracle ~]$ cat targets2.py
import emcli
import login

mytargets = emcli.get_targets \
    (targets='oraoem1%:%;oms12c1_%_oraoem1:%oracle%') \
    .out()['data']

for targ in mytargets:
    print('Target: ' + targ['Target Name'])

[oracle ~]$ emcli @targets2.py
Target: oraoem1.example.com
Target: oraoem1.example.com:3872
Target: oms12c1_3_oraoem1
```

## 任务：使用停机维护

尝试安排停机维护时间以配合目标主机上的维护任务可能会导致误报警。当停机维护在维护活动开始前结束，或者维护活动在停机维护开始前启动时，就可能发生这种情况。通过让维护作业本身启动和停止停机维护，可以避免这两种情况。

`EMCTL`和`EM CLI`都能够创建、删除、启动和停止停机维护。然而，`EMCTL`只能管理执行它的代理主机本地的目标的停机维护，并且只有在代理运行时`EMCTL`命令才有效。使用`EMCTL`设置停机维护的限制性很强且容易出错。建议使用`EM CLI`从维护作业中设置停机维护。



通常，维护活动不需要使用脚本模式，因为大多数维护脚本都在 shell 脚本中运行。`create_blackout` 命令动词需要四个参数：`name`、`add_targets`、`schedule` 和 `reason`。作业由 `name` 参数标识，这意味着它必须在 EM 中所有其他作业中是唯一的。清单 3-16 展示了一个使用 EM CLI 命令行模式设置名为 “Blackout1” 的维护窗口的示例，该窗口针对 “`em12cr3.example.com`” 主机。此维护窗口不会重复，将持续三小时或直到被停止。设置此维护窗口的原因是 “Testing”。

清单 3-16. 为 “`em12cr3.example.com`” 主机设置名为 “Blackout1” 的维护窗口

```
[oracle ~]$ emcli create_blackout -name='Blackout1' \
-add_targets='em12cr3.example.com:host' \
-schedule='frequency:once;duration:3' \
-reason='Testing'
```

`get_blackouts` 命令动词将显示当前定义的所有维护窗口的详细信息，如清单 3-17 所示。

清单 3-17. 列出所有维护窗口

```
[oracle ~]$ emcli get_blackouts
Name       Created By  Status     Status ID  Next Start            Duration  Reason   Frequency  
Blackout1  SYSMAN      Scheduled  0          2435-07-24  17:17:56  03:00     Testing  once  
Repeat Start Time            End Time             Previous  End  TZ Region        TZ Offset
none   2000-07-24  17:17:56  2000-07-24  20:17:56  none          America/Chicago  +00:00
```

下面的示例使用 `noheader` 和 `script` 参数来更改 `get_blackouts` 的输出，使其易于解析。将输出通过管道传递给 `cut` 命令只显示第一个字段，即维护窗口名称字段。

```
[oracle ~]$ emcli get_blackouts -noheader -script | cut -f1
Blackout1
```

可以使用 `get_blackout_details` 命令动词查询维护窗口作业的详细信息。输出包含标题，并以制表符分隔。如果输出行的长度超过屏幕宽度，它们会换行显示，如清单 3-18 所示。

清单 3-18. 列出 “Blackout1” 维护窗口的详细信息

```
[oracle ~]$ emcli get_blackout_details -name='Blackout1'
Status   Status ID  Run Jobs  Next Start            Duration  Reason   Frequency  Repeat  Days
Started  4          no        2014-04-12  13:37:28  03:00     Testing  once       none    none
Months Start Time            End Time              TZ Region        TZ Offset
none   2014-04-12  13:37:28  2014-04-12  16:37:28  America/Chicago  +00:00
```

上面的格式难以阅读，更难以解析。相反，可以将输出更改为逗号分隔格式，移除标题，并将输出限制为我们需要的列，如清单 3-19 所示。

清单 3-19. 以更易于解析的格式列出 “Blackout1” 维护窗口的详细信息

```
[oracle ~]$ emcli get_blackout_details \
-name='Blackout1' -noheader -format="name:csv"
Started,4,no,2014-04-12 13:37:28,03:00,Testing,once,none,none,none,2014-04-12 13:37:28,2014-04-12 16:37:28,America/Chicago,+00:00
```

一旦输出可以被解析，我们就可以通过将命令添加到 shell 脚本来测试我们寻找的值，创建 “Blackout1” 目标，如清单 3-20 所示。

清单 3-20. 在脚本中创建维护窗口并测试退出值

```
emcli create_blackout -name='Blackout1' -add_targets='em12cr3.example.com:host' -schedule='frequency:once;duration:3' -reason='Testing'

MYSTATUS=$(emcli get_blackout_details -name='Blackout1' \
-noheader -format='name:csv' | cut -d ',' -f 1)

if [ "$MYSTATUS" <> "Started" ]; then
echo 'Blackout not started'
        exit 1
fi

<维护脚本>
```

在脚本中使用上述方法，您会注意到多次运行后，它有时会失败，有时会成功。这种行为发生是因为 `create_blackout` EM CLI 命令动词创建作业但不启动作业。作业由另一个进程启动，因此在检查作业状态的 `if` 语句执行时，维护窗口有可能尚未启动，从而导致脚本失败。一种解决方案是使用额外的 shell 脚本，该脚本要么等待作业启动后再继续执行脚本的其余部分，要么因为作业在 “Scheduled” 模式下停留时间过长而失败，如清单 3-21 所示。

清单 3-21. 添加额外的 shell 脚本以使事件顺序执行

```
MYSTATUS='Scheduled'
STATCOUNT=6

while [ "$MYSTATUS" == "Scheduled" ]; do
        if [ $STATCOUNT -lt 1 ]; then
            echo "Blackout1 stuck in scheduled status"
                exit 1
        fi

        let STATCOUNT-=1
        sleep 5
        MYSTATUS=$(emcli get_blackout_details -name='Blackout1' \
                -noheader -format='name:csv' | cut -d ',' -f 1)
done

if [ "$MYSTATUS" != "Started" ]; then
        echo 'Blackout Blackout1 not started'
        exit 1
fi

<维护脚本>
```

维护完成后，同一个脚本可以使用 EM CLI 命令来关闭和删除维护窗口：

```
emcli stop_blackout -name='Blackout1'
emcli delete_blackout -name='Blackout1'
```

## 任务：创建目标

如果 EM 不能自动发现目标，添加目标可能会很痛苦。特别是当目标使用非默认设置时，这有时会使代理难以检测到它们。幸运的是，EM CLI 能够比 EM 更灵活地添加目标。

需要指定哪些参数在很大程度上取决于要添加的目标类型。例如，添加数据库时，`properties` 参数需要包括：

*   `SID`
*   `Port`
*   `OracleHome`
*   `MachineName`

数据库的凭据几乎总是针对 “dbsnmp” 用户，除非是针对未打开的备用数据库，因此需要一个特权密码文件帐户。清单 3-22 展示了一个通过 EM CLI 命令行模式添加目标的示例。

清单 3-22. 添加 “orcl” 单实例数据库目标

```
[oracle ~]$ emcli add_target -name='ORCL_DB' \
-type='oracle_database' -host='oradb1.example.com' \
-properties='SID:orcl;Port:1521;OracleHome:/u01/app/oracle/product/12.1.0/dbhome_1;MachineName:oradb1-vip.example.com' \
-credentials='Role:NORMAL;UserName:DBSNMP;password:<password>'
Target "ORCL_DB:oracle_database" added successfully
```

如果我们想添加一个 RAC 数据库目标，需要首先添加集群的各个实例，如清单 3-23 和 3-24 所示。

清单 3-23. 添加 RAC 数据库节点一

```
[oracle ~]$ emcli add_target -name='ORCL1_ORCL_RACCLUSTER' \
-type='oracle_database' -host='oradb1.example.com' \
-properties='SID:orclrac1;Port:1521;OracleHome:/u01/app/oracle/product/12.1.0/dbhome_1;MachineName:oradb1-vip.example.com' \
-credentials='Role:NORMAL;UserName:DBSNMP;password:<password>'
Target "ORCL1_ORCL_RACCLUSTER:oracle_database" added successfully
```

清单 3-24. 添加 RAC 数据库节点二

```
[oracle ~]$ emcli add_target -name='ORCL2_ORCL_RACCLUSTER' \
-type='oracle_database' -host='oradb2.example.com' \
-properties='SID:orclrac2;Port:1521;OracleHome:/u01/app/oracle/product/12.1.0/dbhome_1;MachineName:oradb2-vip.example.com' \
-credentials='Role:NORMAL;UserName:DBSNMP;password:<password>'
Target "ORCL2_ORCL_RACCLUSTER:oracle_database" added successfully
```



# 添加数据库集群目标与 EM CLI 作业管理

## 添加数据库集群目标

添加实例后，可以通过指定实例作为 `EM CLI` 命令属性的一部分来添加数据库集群目标。这些实例随后会与数据库集群目标关联。

![Image](img/sq.jpg) **注意** 由于我们正在添加集群目标，因此在添加数据库集群目标之前，必须已存在集群目标，如 Listing 3-25 所示。

Listing 3-25. 添加 RAC 数据库集群，关联之前的两个数据库实例

```
[oracle ~]$ emcli add_target -name='ORCL_RACCLUSTER' \
-host='oradb1.example.com' -monitor_mode='1' \
-type='rac_database' -properties='ServiceName:orclrac;ClusterName:oradb-cluster' \
-instances='ORCL1_ORCL_RACCLUSTER:oracle_database;ORCL2_ORCL_RACCLUSTER:oracle_database'
Target "ORCL_RACCLUSTER:rac_database" added successfully
```

## 使用 EM CLI 添加目标

可以使用 `EM CLI` 添加几乎任何目标类型。EM 中的自动发现工具将准确检测大多数目标，从而轻松提升这些目标。当环境具有自定义配置导致自动发现机制不可靠时，手动添加目标是唯一选择。在某些环境中，编写添加目标的脚本可用于在不同版本之间迁移或复制环境。在这些情况下，使用 `EM CLI` 添加目标成为必需。

`add_target` 动词接受十几个参数；可以使用 `EM CLI` 的帮助功能快速参考。在线文档提供了更详细的描述。如果命令帮助不够用，请参考以下链接：

`http://docs.oracle.com/cd/E24628_01/em.121/e17786/cli_verb_ref.htm#CACHFHCA`

My Oracle Support（MOS）也有数十篇关于特定用例和 `EM CLI` 示例的文章。MOS 说明 1448276.1 和 1543773.1 提供了有关使用 `EM CLI` 添加目标的特定信息。

## 任务：操作作业

作业在 EM 中非常重要。在默认安装中，已经创建并运行了许多作业。许多管理员使用 EM 强大的作业调度和执行功能来处理关键的企业任务。

这些功能可以在 GUI 和 `EM CLI` 中得到充分利用。例如，GUI 在作业库中有一个名为“Create Like”的按钮，允许将一个作业的设置复制到另一个作业，以减少两个类似作业所需的输入量。此功能对于只有个别设置不同的一批作业非常适用。

但是，如果需要创建数十个或数百个类似的作业怎么办？或者需要将一批作业从一个 EM 安装复制到另一个？`EM CLI` 的导出作业到文本文件，然后从同一文件或修改版文件导入作业的功能，使得任何规模的作业操作成为可能。

例如，一个执行 `uname -a` 操作系统命令的简单作业可以导出到文件。可以更改命令和作业名称为 `uname` 并使用不同的标志。然后，可以使用包含新变量的修改后文本文件创建新作业，如 Listing 3-26 所示。

Listing 3-26. 描述 `UNAME_A_OMS` 作业

```
[oracle ~]$ emcli describe_job -name='UNAME_A_OMS'

