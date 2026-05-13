# --------------------------------------------------------------------------------------------------

EMGC_OMSn=`ls ${DOMAIN_SERVERS_DIR} | grep OMS`

thisDIR=${DOMAIN_SERVERS_DIR}/${EMGC_OMSn}/logs
   thisAGE=7

thisFILE="EMGC_OMS*.out*"
   SweepThese

for eachDIR in `ls -1 /oraoms/exportconfig`; do
   thisDIR=/oraoms/exportconfig/${eachDIR}
   thisFILE="opf_ADMIN_*.bka"
   SweepThese
Done

CleanUpFiles
exit 0
```

# 第 4 节：EM CLI 脚本与交互式脚本

Enterprise Manager 第 3 版包含标准的仅命令行版本的 EM CLI 以及带有强大脚本选项的版本（有时称为高级套件）。后者可以下载并安装在任何受支持的客户端操作系统上，或直接在服务器上使用。EM CLI 与 Oracle Management Server (OMS) 之间的连接方法相同，与 EM CLI 的安装位置无关。高级套件的安装说明在第 2 章中介绍。

EM CLI 的脚本选项同时指脚本模式和交互模式。在这两种模式下，EM CLI 都期望脚本内容或在交互模式下输入的命令符合 Python 语法。要在交互模式下从 EM CLI 连接到 Enterprise Manager，请输入命令 `login(username='sysman', password='foobar')`。此命令是对一个 Python 函数的调用。



本节中的示例与第 6 章中使用的代码示例相同，且全部使用 Python 编程语言编写。上一节的示例使用操作系统 Shell 来实现所有脚本功能（例如变量、`for`循环、`if then else`等），而本节中的示例将全部使用 EM CLI 及其 Jython Shell 来实现。由于这些示例不依赖于任何 Shell 脚本功能，它们可以在 EM CLI 支持的任何平台上使用。

### 简单登录脚本 (`start.py`)

下面的代码示例可以直接复制到文本文件中。它用于设置一些重要的会话属性，并在 EM CLI 客户端与其连接的 OMS 之间建立安全连接。在使用此脚本前，需要根据您的环境更新变量 `myurl`、`myuser` 和 `mypass`：

```
from emcli import *

myurl = 'https://em12cr3.example.com:7802/em'
mytrustall = 'TRUE'
myoutput = 'JSON'
myuser = 'sysman'
mypass = 'foobar'

set_client_property('EMCLI_OMS_URL', myurl)
set_client_property('EMCLI_TRUSTALL', mytrustall)
set_client_property('EMCLI_OUTPUT_TYPE', myoutput)
print(login(username=myuser, password=mypass))
```

`start.py` 脚本可以在脚本模式和交互模式下使用。在脚本模式下，可以直接使用命令 `emcli @start.py` 从 EM CLI 执行该脚本。结果将是成功或失败地在 EM CLI 与 OMS 之间建立连接。除非您只是想确认连接可以建立，否则单独使用此脚本可能用处不大。

`start.py` 通常被其他脚本调用，或者在交互模式下，从 EM CLI Jython 命令行界面调用（不要与命令行模式混淆）。从另一个脚本调用 `start.py` 是通过将脚本作为模块导入来完成的。

例如，一个名为 `sync.py` 的脚本在脚本模式下使用命令 `emcli @sync.py` 调用。在执行命令之前，必须先建立 EM CLI 与 OMS 之间的连接。如果 `start.py` 脚本位于 `JYTHONPATH` 包含的目录中，则可以使用 `import start` 作为第一个命令来调用该登录脚本。

**注意：** `JYTHONPATH` 是一个 Shell 环境变量，EM CLI 将使用它来确定导入模块时要搜索的所有目录。要添加一个目录，请根据 Shell 标准补充该变量，类似于将目录添加到 `PATH` 环境变量的方式。

另一种方法是使用命令 `emcli @start.py` 在脚本模式下执行 `start.py`，然后在 `start.py` 的最后一行调用 `sync.py` (`import sync`) 脚本。此方法假定 `sync.py` 位于 `JYTHONPATH` 环境变量的可搜索路径中。

当使用交互模式时，`start.py` 脚本必须位于 `JYTHONPATH` 的可搜索路径中，并通过 `import start` 调用。登录现在将为当前交互会话的生命周期建立，直到登录被隐式（超时过期）或显式（使用 `logout()` 函数）解除。

许多企业环境不鼓励或不允许脚本包含凭据密码。可以对 `start.py` 脚本稍作修改，以利用 Python 读取外部文件的功能，如下一个示例所示。

### 安全登录脚本 (`secureStart.py`)

`start.py` 在安全性方面有一个主要缺陷，因为它包含凭据密码。然而，`secureStart.py` 与 `start.py` 具有相同的能力，可以在不泄露脚本中凭据密码的情况下，在 EM CLI 与 OMS 之间建立连接。该脚本改为打开文件系统中的一个文件，该文件包含密码，并将密码读入一个变量。然后，它在 `login()` 函数中使用此变量的内容作为凭据密码。

`secureStart.py` 的使用方法与 `start.py` 相同。但是，有额外的要求。第一个要求是在 EM CLI 可访问的文件系统上创建一个仅包含明文凭据密码的文件。请根据您的环境使用适当的权限保护此文件。除了更新脚本变量 `myurl` 和 `myuser` 以匹配您的环境外，还需要将脚本变量 `mypassloc` 更改为包含凭据密码的文件的完整路径位置。请看以下示例：

```
from emcli import *

myurl = 'https://em12cr3.example.com:7802/em'
mytrustall = 'TRUE'
myoutput = 'JSON'
myuser = 'sysman'
mypassloc = '/home/oracle/.secret'

set_client_property('EMCLI_OMS_URL', myurl)
set_client_property('EMCLI_TRUSTALL', mytrustall)
set_client_property('EMCLI_OUTPUT_TYPE', myoutput)

myfile = open(mypassloc,'r')
mypass = myfile.read()

print(login(username=myuser, password=mypass))
```

登录脚本仅建立 EM CLI 与 OMS 之间的连接。它们对 Enterprise Manager 目标没有任何影响。本节剩余的示例将演示如何使用交互模式或脚本模式操作目标。

### 更新目标属性命令 (`updatePropsFunc.py`)

`updatePropsFunc.py` 脚本包含更新 Enterprise Manager 目标属性所需的命令。这些命令可以在建立登录后，逐个复制到 EM CLI 交互会话中。完整的示例也可以放入脚本中，并使用命令 `emcli @updatePropsFunc.py` 直接在脚本模式下执行（需要先建立登录）。此示例包含注释（以井号字符开头的行），说明了复制到脚本时应包含的脚本命令。在 EM CLI（以及 Python 和 Jython）中，任何以井号字符开头的行都会被忽略。

**注意：** 请参阅本节开头的登录示例，了解如何在脚本模式或交互模式下建立登录。

```
from emcli import *
import re

