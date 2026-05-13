# ----------------------------------------------------
SOURCE_DIR=/gold_scripts/rman
TARGET_DIR=/test_scripts/rman
   CopyFiles

SOURCE_DIR=/gold_scripts/security
TARGET_DIR=/test_scripts/security
   CopyFiles
```

正如我们在上一章看到的，每个 EM CLI 会话都以两个命令开始——`login`和`sync`。在脚本中启动会话时也应遵循同样的做法。你可以每次脚本启动 CLI 会话时重复相同的序列，像这样：

```
emcli login -username="SYSMAN"-password=${
emcli sync
```

由于这两个命令总是一起执行，它们构成了一个简单 Shell 脚本函数的基础，如下所示。我们将在本章后面看一些更复杂的函数。

```
function EmcliLogin {
        emcli login -username="SYSMAN" -password="${SECRET_PASSWORD}"
        emcli sync
}
```

### GetTargetName 函数

在上一章中，我们解释了如何使用`get_targets`动词来确定 OEM 目标的确切名称。由于 Shell 脚本是为了处理边缘情况（如目录缺失）而构建的，任何目标操作脚本都应验证以下内容：

1.  目标存在于 OEM 中。
2.  在你的 CLI 命令中使用了目标的确切名称。

通过给它一个名称并将命令包裹在花括号中，将`get_targets`动词构建成一个函数。在此示例中，`${INPUT_VAL}`的值由调用脚本传入函数：

```
function GetTargetName {
if [ `emcli get_targets | grep ${INPUT_VAL}%"| grep oracle_database" |  wc -l` -gt 0 ];
then
        thisTARGET=`emcli get_targets –format –name="name:csv"| grep -i ${INPUT_VAL} | grep oracle_database | cut -d, -f4`
        echo "Target name is ${thisTARGET}"
else
        echo "No matching targets found"
fi
}
```

在此示例中，我们设置了一个名为`${thisTARGET}`的变量，用于存储 OEM 已知的 Oracle 数据库`acme_qa`的确切目标名称。在运行时，目标名称将通过命令行或通过额外的脚本（例如，处理目标列表）传递。这里概述的步骤将在后面详细说明。

1.  首先确定是否存在匹配的目标。
2.  如果存在，将变量`${thisTARGET}`设置为匹配项并宣布成功。记录和交互式用户通信对于故障排除 Shell 脚本至关重要。只要在运行时有新信息可用，就共享状态。
3.  如果不存在，要大声且礼貌地失败。Unix 编程的核心规则之一是程序应尽早且尽可能大声地失败。Shell 脚本也不例外。

过滤和解析以确定最终的变量字符串遵循此序列：

1.  获取名称中带有`oracle%`的所有类型目标的列表。
2.  在该列表中`grep`匹配`acme_qa`的目标。
3.  在那个更短的列表中`grep` `oracle_databases`。
4.  通过逗号分隔符截取列表并返回第四个值。

步骤 1：列出名称中包含“oracle%”的所有目标

```
Status ID,Status,Target Type,Target Name
1,Up,oracle_emd,acme_dev:3872
1,Up,oracle_emd,acme_qa:2480
1,Up,oracle_emd,acme_prod:3872
1,Up,oracle_database,amce_dev
1,Up,oracle_database,acme_qa
1,Up,oracle_database,acme_prod
1,Up,oracle_listener,lsnracmedev
1,Up,oracle_listener,lsnracmeqa
1,Up,oracle_listener,lsnracmeprod
```

步骤 2：过滤该列表以查找包含 acme_qa 的字符串

```
1,Up,oracle_database,acme_qa
1,Up,oracle_listener,lsnracmeqa
```

步骤 3：仅列出数据库

```
1,Up,oracle_database,acme_qa
```

步骤 4：返回逗号分隔符后的第四个字符串

```
acme_qa
```

确保在脚本执行期间通过名称调用函数：

```
read INPUT_VAL?"Enter the database to be managed:"
GetTargetName
```

## 使用 EM CLI 管理黑屏

Enterprise Manager 最常见的用途是为目标停机事件或超出你设定限制的度量发送通知。你通过为单个目标或目标组创建黑屏来在计划停机期间控制这些警报。创建黑屏需要许多设置，而 EM 控制台提供了一种方便、美观的方法来实现。

EM CLI 提供了一种在你的 Shell 脚本内部创建、停止和删除黑屏的方法。控制台中发现的所有相同选项都可以通过 EM CLI 配置，因为它在幕后调用了相同的 Java 代码。

你必须为 EM CLI 的`create_blackout`动词提供三个值。

*   通过`-name`传递的黑屏唯一名称。
*   通过`-add_targets`参数指定的要包含在黑屏中的目标。
*   作为`-schedule`参数输入的有关开始和结束时间的计划信息。

`-reason`变量不是 EM CLI 必需的，但应包含在内，以使 OEM 管理员更容易评估正在运行、已计划和已停止的黑屏。请保持礼貌。

创建简单黑屏的语法相当直接：

```
> emcli create blackout -name="sample_blackout"\
-add_targets="oracla:oracle_database"\
-schedule="duration::360 ..."
-reason="Blackout demonstration using emcli"
```

### Name 参数



# 计划停用（Blackout）

你可以将任何内容用作停用（blackout）的名称。停用名称在创建时被应用，同时在停止和删除停用时也会用到。始终使用描述性名称是良好的实践，例如 `scripted_blackout_$SID` 或类似的、一致的命名标准。随着我们处理这些示例，你将开始看到一致性如何影响解决方案的设计。

## 添加目标参数

目标必须通过目标名称和目标类型来标识。例如：

- `oracla:oracle_database`
- `demohost:host`
- `lsnroracla:oracle_listener`

## 计划参数

每个计划参数在命令行接受两个必填值和几个可选值，类似于控制台中的灵活性：

```
frequency:once
requires => duration or end_time
optional => start_time, tzinfo, tzoffset

frequency:interval
requires => duration, repeat
optional => start_time, end_time, tzinfo, tzoffset

frequency:weekly
requires => duration, days
optional => start_time, end_time, tzinfo, tzoffset

frequency:monthly
requires => duration, days
optional => start_time, end_time, tzinfo, tzoffset

frequency:yearly
requires => duration, days, months
optional => start_time, end_time, tzinfo, tzoffset
```

## 计划停用的一个健壮解决方案

作者的环境中包含一些应用程序培训数据库，这些数据库每晚都会从冷备份中刷新。当然，一旦这些目标被添加到 OEM，他就开始在夜间收到由这些计划中断引起的寻呼通知。你可以想象他的反应。

- 最简单的解决方案是在控制台中按计划将这些目标整夜停用。
- 如果需要在白天运行此刷新怎么办？
- 你真的希望这些目标在夜间不被监控吗？
- 如果刷新失败怎么办？培训师通常不是耐心的客户，尤其是一房间闲置的学员正在等待他们的实验。

这里有一个涉及 Shell 函数、EM CLI 及其相关运行时脚本的解决方案。我们将首先看看在不使用 Shell 函数的情况下通常如何完成此过程，然后我们将把脚本转换为可移植、可扩展的函数。

## 使用 Shell 脚本创建和停止停用

可以通过一系列 CLI 动词创建停用，以执行以下步骤来测试、执行然后验证停用状态：

1.  确定目标是否已处于停用状态。
2.  如果停用已存在，则停止并删除它（以便为你的新停用获得完整持续时间）。
3.  创建并启动新的停用。
4.  通过将其值回显到屏幕以及日志文件中来验证停用是否已启动。

这些步骤在此简化脚本中进行了说明。你的真实脚本应测试数据库名称是否作为 `$1` 值在命令行传递，然后根据运行时条件提示或失败。例如，如果脚本是由活动用户执行的，它应提示输入数据库名称，否则它将失败并将原因回显到日志文件。

```shell
#!/bin/ti
