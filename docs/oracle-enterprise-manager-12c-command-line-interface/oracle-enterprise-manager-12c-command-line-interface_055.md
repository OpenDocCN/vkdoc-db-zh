# =====================================================
function CreateBlackout {
echo "\n\nCreating blackout named '${BO_NAME}' ...\n"
```



## EMCTL 或 EM CLI

Enterprise Manager 控制实用程序，即 EMCTL，随每个 OEM 代理一起安装，用于手动启动和停止 OEM 代理以及管理远程主机上的本地功能。

EM CTL 可以像 EM CLI 一样用于启动和停止（但不能删除）OEM 黑屏，但有一个关键优势：EM CTL 已经安装和配置好。如果您选择 EM CLI 作为使用 shell 脚本管理远程主机黑屏的工具，则必须在每台主机上手动安装和配置 EM CLI 客户端。EM CLI 客户端是 Java 程序的静态安装。客户端软件安装会通过 OEM 本身报告，但不受其管理。如果您的环境中的远程主机数量众多，且您将功能限制为黑屏维护和其他它可管理的任务，那么 EM CTL 可能是更好的解决方案。第 2 章 中介绍的安装过程提供了一种执行该安装的脚本化方法。

使用 EM CTL 管理黑屏与用于 EM CLI 的技术非常相似，并且同样适用于相同的脚本应用程序。

其语法很简单：

```
emctl start blackout <blackout name><target names>
emctl start blackout <blackout name>  -nodelevel
emctl stop blackout <blackout name>
emctl status blackout <blackout name>
```

`nodelevel` 标志会使节点上的所有目标进入黑屏状态，非常适合打补丁活动。

持续时间由 `–d` 标志指定，可以设置为天和小时的组合，如下所示：

```
emctl start blackout WeekendBlackout  -nodelevel –d 2 12:00
```

在此示例中，可以将开发节点上所有目标的黑屏计划为在星期五下午 6:00 开始，并将持续到星期一早上 6:00（两天十二小时）。

## 使用 EM CLI 管理 OEM 管理员

控制台提供了一种清晰、便捷的方法来管理 OEM 管理员帐户，但其灵活性使得创建多个帐户时，需要点击所有屏幕，变得非常不便。根据作者统计，在控制台中创建一个新的管理员至少需要 17 次鼠标点击！EM CLI 将 OEM 代码库固有的灵活性与命令行界面的简洁性相结合。

控制台中定义的所有选项都可以使用下面显示的标志作为选项在 EM CLI 中授予。要创建一个管理员，您只需为帐户指定名称和密码：

```
emcli create_user -name="SuzyQueue"-password="oracle"
```

如果您正在阅读 Apress 出版的书籍，就不会忽略用户安全性，因此您会希望使用 `-expired="true"` 标志使用户的新密码过期，如下所示：

```
emcli create_user \
  -name="SuzyQueue"\
  -password="oracle"\
  -expired="true"
```

其他可选参数允许您执行 OEM 控制台中可用的用户授权，而无需点击流程。

### 角色管理

Enterprise Manager 管理员和用户实际上是存储库数据库中的数据库用户帐户。请避免通过 SQL*Plus 授予角色权限的诱惑，因为 OEM 可能会在帐户管理中执行其他操作。

您可以在创建用户时通过添加 `-roles` 参数来添加角色授权：

```
emcli create_user \
  -name="SuzyQueue"\
  -password="oracle"\
  -roles="em_all_administrator"
```

您也可以通过 `grant_roles` 或 `revoke_roles` 动词修改用户：

```
emcli grant_roles \
  -name="SuzyQueue"\
-roles="em_all_viewer"

emcli revoke_roles \
  -name="SuzyQueue"\
  -roles="em_all_operator"
```

### 使用 Argfile 进行批处理

EM CLI 有一个名为 `argfile` 的批处理动词，与许多其他 Oracle 参数文件类似，它允许您通过提供一个包含要应用的输入值的文本文件的完全限定路径，在单次 EM CLI 执行中执行多个命令。

例如，让我们在 `/tmp/create_several_users.lst` 文件中加载一些用于创建几个用户的 CLI 命令：

```
touch /tmp/create_several_users.lst
for thisADMIN in Sugar Magnolia Blossom Blooming, do
    echo "create_user -name=${thisADMIN} -password=oracle -expired=true";
done

cat /tmp/create_several_users.lst
create_user -name="Sugar"-password="oracle"-expired="true"
create_user -name="Magnolia"-password="oracle"-expired="true"
create_user -name="Blossom"-password="oracle"-expired="true"
create_user -name="Blooming"-password="oracle"-expired="true"

emcli login -username=SYSMAN -password="${SECRET_PASSWORD}"
emcli sync
emcli argfile "/tmp/create_several_users.lst"
```

使用 argfile 的优点如下：

*   EM CLI 实用程序在 argfile 之外被调用，因此动词是串行处理的，无需重新启动 EM CLI。
*   Argfile 是一个文本文件，您选择的后缀对执行完全没有影响。该示例使用 `lst` 后缀，但您也可以使用 `cli` 或 `cmd`。
*   您必须完全限定 argfile 的路径。
*   此示例不包括文件清理，但我相信您的脚本会包含。

正如您可能猜测的那样，这种方法非常高效，并且 argfile 执行通常运行非常快，因为通过 CLI 实用程序对 OMS 只有一次调用。

### 从文本文件填充 Argfile

手动创建和编辑 argfile 并非不可能，但有时（总是？）脚本化解决方案更容易且不易出错。例如，如果您收到一个请求，要求为应用组的几个成员提供 OEM 访问权限，并附带以下 CSV 文件，该怎么办？

![Image](img/sq.jpg) 注意：为清晰起见，本图中包含了反斜杠。请将每个完整字符串放在一行。

如果函数库已经创建，您可以自由地在所有脚本中使用这些相同的函数，甚至在命令行中也可以。例如，假设您需要对特定服务器上的数据库设置黑屏以进行操作系统补丁。在命令行中，您将 source 函数库，然后根据需要调用函数。文件通过键入一个句点、一个空格和文件名来 source。例如，当您登录到 Unix/Linux 服务器时，您的用户配置文件将 source `.bashrc` 和 `.bash_profile`（如果 Bash 是您选择的 shell），以将您的环境变量加载到内存中。当您键入 `env` 命令时，它将反映这些值。Source 其他文件，如函数库，会将新文件中的路径、值、函数或别名加载到内存中，以补充或替换已有的环境变量：

```
. /script/oem/emcli_functions.lib
for thisSID in suzie, sally, betty, veronica; do
    BO_NAME=os_patch_blackout_${thisSID}
    echo "Starting the blackout for ${thisSID}"
    CreateBlackout ${BO_NAME}
done
```

当工作完成后，您将使用类似的过程进行反向操作：

```
. /script/oem/emcli_functions.lib
for thisSID in suzie, sally, betty, veronica; do
    BO_NAME=os_patch_blackout_${thisSID}
    EndBlackout ${BO_NAME}
done
```

除了简化您的运行时环境（无论是在命令行中还是在脚本中）之外，您还可以确信函数库中的代码已在其他用例中经过测试。函数库还消除了命令输入错误和步骤遗漏的机会。

```
if [ `emcli get_blackouts | grep ${BO_NAME} | wc -l` -gt 0 ]; then
echo "\n\nFound an existing blackout named ${BO_NAME}"
        echo "That blackout will be stopped and deleted prior to starting the new one\n\n"
        emcli stop_blackout -name="${BO_NAME}"
        emcli delete_blackout -name="${BO_NAME}"
fi

emcli create_blackout -name="${BO_NAME}"
 -add_targets=${thisTARGET}:oracle_database
 -schedule="duration::360;tzinfo:specified;tzregion:America/Los_Angeles"
 -reason="Scripted blackout for maintenance or refresh"
}

function EndBlackout {

echo "\n\nStopping blackout '${BO_NAME}' ...\n"
emcli stop_blackout -name="${BO_NAME}"

echo "\n\nDeleting blackout '${BO_NAME}' ...\n"
emcli delete_blackout -name="${BO_NAME}"
}
```



# 使用 EM CLI 进行高级管理

## 准备参数文件

```
cat manage_request.txt
login,name,department,email e123450,"Jerry",A100,jerry@samplecorp.com
e123451,"Bobby",A100,bobby@samplecorp.dom
e123452,"Phil",A100,phil@samplecorp.dom
e123449,"Robert",A100,roberth@samplecorp.dom
e123499,"Ron",A100,ronpig@samplecorp.dom
e123460,"Bill",A110,bill@samplecorp.dom
e123461,"Mickey",A110,mickey@samplecorp.dom
e123491,"Keith",A120,keith@samplecorp.dom
e123492,"Donna",A120,donnaj@samplecorp.dom
e123460,"Brent",A120,brent@samplecorp.dom
```

我们将通过重定向循环语句的输出来创建参数文件，如下所示：

```
touch /tmp/user_builder_argfile.lst

for thisENTRY in `cat manage_request.txt`; do
  LOGIN=`echo ${thisENTRY} | cut -d, -f1`
  NAME=`echo ${thisENTRY} | cut -d, -f2`
  DEPT=`echo ${thisENTRY} | cut -d, -f3`
  EMAIL=`echo ${thisENTRY} | cut -d, -f4`
echo "delete_user -name=${NAME} \n >>/tmp/user_builder_argfile.lst
echo "create_user -name=${NAME} \
\t -password=oracle -expired=true \
     \t -desc=${NAME} \
     \t -department=${DEPT} \
     \t -email=${EMAIL} \
     \t -roles="EM_USER \n"
>>/tmp/user_builder_argfile.lst
done
echo "\n\nArgfile has been created\n"
cat /tmp/user_builder_argfile.lst
```

最终我们得到的参数文件内容如下所示：

```
delete_user -name=e123450
create_user -name=e123450 -password=oracle -expired=true \
-desc=Jerry -department=A100 -email=jerry@samplecorp.com-roles=EM_USER
delete_user -name=e123451
create_user -name=e123451 -password=oracle -expired=true \
-desc=Bobby -department=A100 -email=bobby@samplecorp.com-roles=EM_USER
. . .
```

在运行时，我们将看到如下输出：

```
>emcli argfile /tmp/user_builder_argfile.lst
>User "e123450"deleted successfully
>User "e123450"created successfully
>User "e123451"deleted successfully
>User "e123451"created successfully
...
```

这种方法快速高效，因为所有命令都在一个 CLI 会话中发送给 OMS，避免了将每条命令作为单独的事务发送给 OMS 处理的开销。

## 在 OMS 实例间克隆 OEM 角色和用户

沙盒 OEM 服务器是精心设计的 OEM 环境的一部分，但如果没有 EM CLI，将 EM 安全数据从一个存储库复制到另一个可能很困难。本示例结合使用 `SQL*Plus` 和 `EM CLI` 从现有环境中收集数据，然后在沙盒中快速、完整地重现它。

首先，我们将针对现有 OEM 存储库数据库中的 `SYSMAN` 模式执行查询，创建三个假脱机文件。这些假脱机文件将成为我们用于构建角色、用户和权限的 EM CLI 参数文件。

存储库数据库中的 `SYSMAN` 模式包含 OEM 收集或设置的所有信息。当 EM 代理将其最新指标上传到管理服务器时，这些值会发布到 `SYSMAN` 模式中的表中。每当您在控制台中请求页面时，它都是从 `SYSMAN` 中的数据填充的。

```
#!/bin/bash# =====================================================
