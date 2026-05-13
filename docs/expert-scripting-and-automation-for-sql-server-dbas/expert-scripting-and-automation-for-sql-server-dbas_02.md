# 2. PowerShell 基础

PowerShell 是由微软编写的一种强大的脚本语言。它使用 .NET 公共语言运行时（CLR），提供了开箱即用的庞大功能，并且完全可扩展。它专为自动化和管理而设计，常用于云环境（如 Azure 和 AWS）的自动化或维护。它也是管理诸如 Microsoft Exchange 等平台的首选语言，当然也包括 SQL Server。PowerShell 核心版（v7）不仅支持 Windows，还支持 Mac OS 和 Linux。它完全支持多种 Linux 变体，包括 Red Hat、Ubuntu 和 Debian 等等。

在本章中，我们将回顾 PowerShell 语言的基础知识。我们将从一个简短的加速器开始，它将帮助我们提高效率。然后我们将讨论一些基本的语言元素，例如注释、数据类型、变量和流程控制语句。最后，我们将概述 PowerShell 模块，它们提供了可扩展性。

## 入门指南

要开始使用，我们应该理解 PowerShell 可以通过两种不同的方式使用。第一种是命令行 Shell。它为管理员提供了使用 PowerShell 终端运行临时命令（称为 cmdlet）或简单脚本的能力。在 Windows 环境中，此终端可以替代 cmd shell，并提供更丰富的体验。命令返回的是 .NET 对象而非纯文本，并且可以使用管道（我们将在本章后面讨论）将命令链接在一起。终端本身提供了命令行历史和 Tab 键补全功能。

第二种使用 PowerShell 的方式是将其作为脚本语言。当用作脚本语言时，PowerShell 为创建自动化提供了一个丰富、可扩展的平台，允许管理员和 DevOps 工程师利用 .NET 的力量以及一个完全面向对象的语言（包含类、继承、方法和属性）。这意味着该语言可以通过类、函数和模块完全扩展。

## 执行策略

如果我们计划在 Windows 上使用 PowerShell 进行自动化，那么我们需要考虑执行策略。这是 PowerShell 安全策略的一部分，决定了用户是否可以运行脚本、加载配置文件以及加载配置文件。表 2-1 详细列出了可以在 Windows 上设置的执行策略。

**表 2-1**

**执行策略**

| 策略 | 描述 |
| --- | --- |
| **Undefined** | `Windows 环境中的默认策略。检查组策略中是否定义了策略。如果在所有作用域都未定义，则其效果等同于 Windows 客户端的 Restricted 或 Windows Server 的 RemoteSigned。` |
| **AllSigned** | `所有脚本和配置文件都必须由受信任的发布者签名。即使脚本是在本地计算机上编写的。` |
| **Bypass** | `没有任何阻止。没有警告或提示。` |
| **Default** | `将 Windows 桌面设置为 Restricted，将 Windows 服务器设置为 RemoteSigned。` |
| **RemoteSigned** | `如果脚本或配置文件是下载的，则必须签名或明确解除阻止。本地创建的脚本可以无需签名即可执行。` |
| **Restricted** | `不能运行脚本，也不能加载配置文件。` |
| **Unrestricted** | `所有脚本都可以执行，所有配置文件都可以加载。如果脚本是从互联网下载且未签名，则用户在执行前会收到提示。这是非 Windows 系统上的默认设置。` |

默认情况下，指定执行策略时，它应用于 `LocalMachine` 作用域。但是，可以使用 `-Scope` 参数来指定 `CurrentUser` 或 `Process`。如果使用 `Process` 作用域，则该策略仅适用于当前 PowerShell 会话。

在非 Windows 操作系统上，从 PowerShell 6 开始，默认策略是 `Unrestricted`，并且无法覆盖。然而在 Windows 上，应适当考虑合适的执行策略，通常需要与组织的网络安全团队协作制定。

如果您打算按照本书中的示例操作，并从代码存储库下载代码示例，我建议将执行策略设置为 `Unrestricted`，这可以通过清单 2-1 中的命令实现。

```
Set-ExecutionPolicy Unrestricted
```
**清单 2-1**
**设置执行策略**

> **提示**
> 要设置执行策略，必须以管理员身份运行终端。

## 标准

虽然有例外，但 PowerShell 中的大多数命令都遵循 `动词-名词`（名词为单数）的命名约定。命令名称的第一部分（“动词”）描述了您将执行的操作。这些被称为已批准的动词，使用它们是为了保持一致性，以便命令能够被直观地理解。下面列出了一些最常见的动词，但完整列表可以在 learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7.4 找到：

*   `Add`
*   `Clear`
*   `Connect`
*   `Convert`
*   `Copy`
*   `Disable`
*   `Disconnect`
*   `Enable`
*   `Format`
*   `Get`
*   `Install`
*   `Invoke`
*   `New`
*   `Read`
*   `Remove`
*   `Rename`
*   `Reset`
*   `Revoke`
*   `Save`
*   `Select`
*   `Set`
*   `Show`
*   `Start`
*   `Stop`
*   `Test`
*   `Update`
*   `Wait`
*   `Write`

这个前缀后面跟着一个连字符（`-`），最后是对您将要对其执行操作的资源的描述（“名词”）。例如，`Get-ChildItem` 可用于列出当前 PowerShell 位置的内容。此位置可以是文件系统中的文件夹，也可以是 SQL Server 实例中的表。导航 SQL Server 实例将在本章后面讨论。逻辑且一致的命名约定，结合 IntelliSense，使得在诸如 Visual Studio Code 等开发环境中学习和查找命令成为一个相对容易的过程。



## 别名

为了让学习 PowerShell 变得更加简单，许多命令都有别名，这些别名映射到 DOS 或 Linux 命令。这意味着，如果你有使用 Windows 命令提示符（基于 DOS）或 Bash 的任何经验，就可以继续使用许多你已经熟悉的命令。

例如，我们来看一下 `Get-ChildItem`。如前所述，`Get-ChildItem` 可用于列出文件系统中文件夹的内容。许多不熟悉此命令的 PowerShell 新手会熟悉 DOS 的 `dir` 或 Linux 的 `ls`。因此，PowerShell 提供了这两个别名。例如，代码清单 2-2 中的所有命令在功能上都是等效的。

```
dir *.txt #使用 DOS 别名
ls *.txt #使用 LINUX 别名
gci *.txt #使用 gci 别名
Get-ChildItem *.txt #使用 PowerShell 命令
代码清单 2-2
使用别名
```

> **提示**：当你与 PowerShell 终端进行交互式操作时，别名可以提高工作效率，但在脚本中使用别名被认为是糟糕的做法，因为它会使代码更加晦涩难懂，更难以阅读。

## 语言基础

以下部分将讨论注释、数据类型、变量、管道和筛选以及流程控制语句。PowerShell 是一门非常庞大的语言，但本节包含的结构将为你提供一个加速器，并为你阅读本书其余部分所需的基础知识。

### 注释

PowerShell 支持行内注释和块注释。行内注释由 `#` 符号表示。该符号右侧的所有内容都被视为注释，不会被执行。例如，代码清单 2-3 中的命令与代码清单 2-1 中的命令在功能上是等效的。

```
Set-ExecutionPolicy -ExecutionPolicy Unrestricted #将执行策略设置为 Unrestricted
代码清单 2-3
行内注释
```

块注释以 `<#` 开始，以 `#>` 结束。这些字符序列之间的任何内容都被视为注释，不会被执行。代码清单 2-4 中的脚本演示了块注释的用法。

```
<#
Set-ExecutionPolicy  -ExecutionPolicy Unrestricted
此命令用于将 PowerShell 的执行策略设置为“无限制”。
#>
代码清单 2-4
块注释
```

### 数据类型

PowerShell 使用 .NET CLR 提供的数据类型。.NET 中的每种数据类型都派生自 `Object` 数据类型。因此，`Object` 数据类型是类型层次结构的根，同时它本身也是一个数据类型。

表 2-2 中列出的基本类型直接派生自 `Object` 类型。从这些基本类型中，实际上派生出了数百种数据类型。其中一些派生类型很复杂，例如 `XML`、`DateTimeOffset`（存储相对于 UTC 偏移的日期和时间）和 `Array`。

**表 2-2 基本数据类型**

| 基本数据类型 | 描述 |
| --- | --- |
| `Byte` | 8 位无符号整数 |
| `SByte` | 不符合 CLS 规范的 8 位有符号整数 |
| `Int16` | 16 位有符号整数 |
| `Int32` | 32 位有符号整数 |
| `Int64` | 64 位有符号整数 |
| `UInt16` | 16 位无符号整数 |
| `UInt32` | 不符合 CLS 规范的 32 位无符号整数 |
| `UInt64` | 不符合 CLS 规范的 64 位无符号整数 |
| `Single` | 32 位浮点数 |
| `Double` | 64 位浮点数 |
| `Boolean` | 布尔值/位值 |
| `Char` | Unicode（16 位）字符 |
| `Decimal` | 十进制（128 位）值 |
| `IntPtr` | 有符号整数，在 32 位平台上为 32 位值，在 64 位平台上为 64 位值 |
| `UIntPtr` | 不符合 CLS 规范的无符号整数，在 32 位平台上为 32 位值，在 64 位平台上为 64 位值 |
| `String` | Unicode 字符字符串 |

### 变量

与所有语言一样，变量包含在运行时未知或在执行期间会更改的值。它们对于编写灵活、强大和可维护的代码至关重要。变量用于存储用户输入和循环等目的。每个变量都映射到一个数据类型，并且由于变量是派生自 .NET 类的对象，因此可以对它们调用各种方法。例如，可以对 `Int32` 调用 `ToString()` 方法，将变量值作为字符串而非整数输出。

变量始终以 `$` 为前缀，声明变量时，你在美元符号 `$` 前面的方括号中指定数据类型，最后声明变量的名称。

> **提示**：始终为变量赋予有意义的名称。如果你给变量起名为 `$i` 或 `$a`，它们就变得毫无意义，就像命名为 `$foo` 和 `$bar` 一样。为变量赋予有意义的名称使你的代码更易于阅读和维护。

代码清单 2-5 中的脚本首先创建两个变量。一个是名为 `$HelloWorldInt` 的整数，另一个是名为 `$HelloWorldText` 的字符串，并为每个变量赋值。脚本的第二部分清除控制台的消息，然后输出结果。

```
#声明变量并赋值
[int]$HelloWorldInt = 123
[string]$HelloWorldText = "Hello World"
#将结果打印到控制台
Clear-Host
Write-Host "HelloWorldInt: " $HelloWorldInt
Write-Host "HelloWorldText: " $HelloWorldText
代码清单 2-5
声明和使用变量
```

我们也可以在不指定数据类型的情况下声明变量。当你这样做时，PowerShell 将使用它认为最合适的数据类型。但请注意，PowerShell 并未通灵！它只能猜测变量应该是哪种数据类型。例如，考虑代码清单 2-6 中的脚本。这里，我们声明了六个变量而没有指定数据类型。然后我们使用 `GetType()` 方法来提取每个变量的数据类型。

```
#行内声明变量
$StringVariable1 = "Hello World"
$StringVariable2 = 123
$StringVariable3 = "123"
$Int32Variable = 123
$SingleVariable = 3.3
$XMLVariable = 'MyValue'
#提取数据类型
Clear-Host
"StringVariable1: " + $StringVariable1.GetType()
"StringVariable2: " + $StringVariable2.GetType()
"StringVariable3: " + $StringVariable3.GetType()
"Int32Variable: " + $Int32Variable.GetType()
"SingleVariable: " + $SingleVariable.GetType()
"XMLVariable: " + $XMLVariable.GetType()
代码清单 2-6
不指定数据类型声明变量
```

从下面列出的结果可以看出，数据类型并非如你所愿：

```
StringVariable1: string
StringVariable2: int
StringVariable3: string
Int32Variable: int
SingleVariable: double
XMLVariable: string
```

你可以看到 `$StringVariable`、`$StringVariable3` 和 `$Int32Variable` 被分配了我们想要的数据类型。然而，对于其他变量，我们就不那么幸运了。`$StringVariable2` 被创建为 `int` 数据类型，因为我们没有将赋值的值用引号括起来。这就是为什么 `$StringVariable3` 没有遭受同样问题的原因。`$SingleVariable` 被创建为 `double` 数据类型。这并非世界末日，但这确实意味着我们使用了比所需更多的内存资源。`$XMLVariable` 被创建为 `string` 数据类型。这意味着我们将无法使用 XML 特定的方法和 cmdlet，例如 `GetElementsByTagName` 方法和 `Select-Xml` cmdlet。

总之，重要的是要记住不指定数据类型的潜在复杂性，并且如果数据类型对你的脚本很重要，则应始终指定。另一个需要考虑的因素是代码维护。如果对你（或团队成员）来说，引用变量的数据类型并不容易，那么在你进行故障排除、修订或增强脚本时，可能会减慢你的速度。



### 管道与筛选

如果你有 T-SQL 背景而非编程背景，管道（piping）的概念对你来说可能是全新的。在 PowerShell 中，管道是指将一个命令的结果传递给另一个命令的过程。管道是 PowerShell 的一个核心概念，也是其如此强大的原因之一。

管道的一个简单示例如清单 2-7 所示。此脚本中的第一行从操作系统获取正在运行的进程列表。然后，它将此命令的结果（作为一个或多个对象，而不仅仅是纯文本）传递到 `Where-Object` cmdlet。`Where-Object` 遍历所有接收到的对象，并应用提供的筛选脚本（在花括号中）。该筛选脚本使用特殊的自动 PowerShell 变量 `$_`（它表示当前正在处理的对象），将每个对象的 `ProcessName` 属性与字符串“*p*w*sh*”进行比较（星号用作通配符）。不匹配的对象将从结果中移除。

```
# 返回正在运行的进程列表，然后按进程名称进行筛选
# 选择的筛选字符串将匹配 PowerShell 7 (pwsh) 和 5 (powershell) 的可执行文件
$process = Get-Process | Where-Object { $_.ProcessName -like "*p*w*sh*" }
# 将筛选后的进程列表打印到控制台
$process
清单 2-7
基本管道操作
```

如果没有管道能力，代码会长得多且更难管理。在这种情况下，我们需要一个单独的语句来遍历读入 `$process` 变量的每个进程。对于像这样非常简单的例子，这可能听起来没什么大不了的，但当我们有非常庞大、复杂的脚本时，由此产生的额外开销就会成为一种负担。

### 流程控制

如果你的背景纯粹是 SQL Server，那么你可能对循环的想法感到反感，因为在 SQL Server 中循环是一种非常糟糕的做法，这是由于该语言基于集合的特性。如果你有任何脚本编写或编程经验，你就会知道循环在遍历对象、执行指定次数的操作（或一组操作）或重复执行操作（或一组操作）直到满足条件方面是多么有用。PowerShell 通过 `for` 语句、`foreach` 语句和 `while` 语句提供循环能力。

> **注意**
> 当然，关于循环多么有用的论断并不适用于 T-SQL，因为基于集合的操作总是比循环高效得多。更多详情请参考第 1 章。

不过，在我们深入探讨如何使用循环之前，我们需要确保熟悉 PowerShell 支持的运算符，因为我们将需要它们来控制循环。如果你熟悉 T-SQL，那么你可能已经了解运算符的概念，并且知道诸如 `=`（等于）、`>`（大于）和 `<=`（小于或等于）之类的比较运算符。

PowerShell 中可用的比较运算符列于表 2-3 中，但与 T-SQL 的一个重要区别是，在 PowerShell 中 `=` 仅用于赋值，从不用于比较。PowerShell 中的相等比较运算符是 `-eq`。记住这一点非常重要，以避免代码中一些难以发现的、讨厌的错误。

**表 2-3: 比较运算符**

| 运算符 | 描述 |
| --- | --- |
| `-contains` | 属性值中的任何项等于指定值。当你必须确保数组中存在特定值时，这很有用 |
| `-eq` | 属性的值等于指定值 |
| `-ge` | 属性的值大于或等于指定值 |
| `-gt` | 属性的值大于指定值 |
| `-in` | 属性的值等于指定值数组中的任何一个元素。值数组可以指定为逗号分隔的列表 |
| `-is` | 属性的值的数据类型是指定的数据类型。指定的数据类型必须用方括号括起来 |
| `-isnot` | 属性的值的数据类型与指定的数据类型不同。指定的数据类型必须用方括号括起来 |
| `-le` | 属性的值小于或等于指定值 |
| `-lt` | 属性的值小于指定值 |
| `-like` | 属性的值包含指定值，遵循通配符匹配 |
| `-match` | 属性的值与提供的正则表达式模式匹配 |
| `-ne` | 属性的值不等于指定值 |
| `-notcontains` | 属性值中的任何项都不等于指定值。当你必须确保数组中不存在特定值时，这很有用 |
| `-notin` | 属性的值不等于指定值数组中的任何一个元素。值数组可以指定为逗号分隔的列表 |
| `-notlike` | 属性的值不包含指定值，遵循通配符匹配 |
| `-notmatch` | 属性的值不与提供的正则表达式模式匹配 |
| `-ccontains` | 与 `-Contains` 相同，但区分大小写 |
| `-ceq` | 与 `-EQ` 相同，但区分大小写 |
| `-cge` | 与 `-GE` 相同，但区分大小写 |
| `-cgt` | 与 `-GT` 相同，但区分大小写 |
| `-cin` | 与 `-In` 相同，但区分大小写 |
| `-cle` | 与 `-LE` 相同，但区分大小写 |
| `-clt` | 与 `-LT` 相同，但区分大小写 |
| `-clike` | 与 `-Like` 相同，但区分大小写 |
| `-cmatch` | 与 `-Match` 相同，但区分大小写 |
| `-cne` | 与 `-NE` 相同，但区分大小写 |
| `-cnotcontains` | 与 `-NotContains` 相同，但区分大小写 |
| `-cnotin` | 与 `-NotIn` 相同，但区分大小写 |
| `-cnotlike` | 与 `-NotLike` 相同，但区分大小写 |
| `-cnotmatch` | 与 `-NotMatch` 相同，但区分大小写 |

`for` 循环可用于将一组操作重复指定次数，或遍历数组或集合的子集。`foreach` 循环在你必须遍历数组或集合中的每个项时非常有用，而 `while` 循环用于重复一组操作，直到满足某个条件。

清单 2-8 演示了如何使用 `for` 循环来实现错误处理。该脚本将尝试每 30 秒读取一次文件内容，共尝试三次。for 循环的行为在括号内定义。括号内有三个元素。第一个初始化将要控制循环的变量。在我们的示例中，我们创建一个名为 `$i` 的变量，初始值为 1。第二个定义指定了在我们的循环终止之前必须满足的条件。我们指定当 `$i` 小于或等于 3 时继续循环。最后一个定义控制循环变量的增量。`$i++` 是 `$i = $i + 1` 的简写。因此，在我们的示例中，`$i` 的值将在每次循环迭代时增加一。



在循环的每次迭代中要重复执行的代码被包含在花括号内。这里，我们使用一个 `try...catch` 代码块来尝试读取文件。如果操作成功，我们会将成功信息写入控制台，然后通过 `break` 命令退出循环。如果操作失败，执行流程会移至 `catch` 代码块。在这里，我们将失败信息写入控制台，然后将脚本的执行暂停 30 秒。

**提示**
尽管读取文件失败总会产生一个错误，但它通常不会导致脚本终止。由于我们需要脚本终止才能使流程转入 `catch` 代码块，因此我们使用 `-ErrorAction` 参数来强制脚本终止。

```
for ($i=1; $i -le 3; $i++) {
try {
Get-Content c:\ExpertScripting.txt -ErrorAction Stop
"Attempt " + $i + " Succeeded"
break
} catch {
"Attempt " + $i + " Failed"
Start-Sleep -s 30
}
}
代码清单 2-8
使用 for 循环
```

代码清单 2-9 中的脚本演示了如何使用 `while` 循环来优化代码清单 2-8 中的代码，使得脚本能够持续循环（可能是无限循环），直到文件读取成功为止。只要括号内定义的条件评估为布尔值 `$true`，`while` 循环就会继续。在我们的例子中，由于条件就是 `$true`，`while` 循环将无限期地继续下去。然而，一旦文件内容被成功读取，`break` 命令就会退出循环。当你在编写需要先写入文件才能继续的中间件组件脚本时，这种技术特别有用。

**注意**
如果你正在跟随演示操作，请在脚本运行时，在 `c:\` 根目录下创建一个名为 `ExpertScripting.txt` 的文件。

```
$i = 0
while ($true) {
try {
$i++
Get-Content c:\ExpertScripting.txt -ErrorAction Stop
"Attempt " + $i + " Succeeded"
break
} catch {
"Attempt " + $i + " Failed"
Start-Sleep -s 30
}
}
代码清单 2-9
使用 while 循环
```

代码清单 2-10 中的脚本演示了如何使用 `foreach` 循环来确保所有 SQL Server 服务都在运行，如果它们没有运行，则启动它们。

```
# 用所有 SQL Server 服务的详细信息填充一个新变量
$services = Get-Service | Where-Object { $_.Name -like "*SQL*" -and $_.Status -eq "Stopped" }
# 启动每个服务
foreach ($name in $services) {
Start-Service $name
}
代码清单 2-10
使用 foreach 循环
```

我们将要讨论的最后一个流程控制概念是 `IF`...`ELSE`。一个 `IF`...`ELSE` 代码块允许你根据一个条件来分支你的代码。你也可以嵌套 `IF`...`ELSE` 代码块来实现复杂的逻辑（尽管这样做时应考虑代码的可维护性）。只能有一个 `IF` 代码块。要实现多个链式的 `IF` 条件，你可以添加多个 `ELSEIF` 代码块。`ELSE` 代码块总是放在最后（如果你选择使用它），用于处理所有 `IF` 和 `ELSEIF` 条件都不满足的情况。`IF` 和 `ELSEIF` 代码块的条件被括在圆括号内。要执行的代码块被括在花括号内。代码清单 2-11 中的脚本演示了如何实现一个 `IF`...`ELSE` 代码块来检查服务的状态。

```
$Service = "SQLBrowser"
$ServiceDetails = Get-Service | Where-Object {$_.Name -eq $Service}
if ($ServiceDetails.Status -eq "Running") {
$Service + " is  working"
} elseif ($ServiceDetails.Status -eq "Stopped") {
$Service + " is not working. Please check the Event Log"
} elseif (-not $serviceDetails) {
$Service + " is not installed"
} else {
$Service + " is changing state"
}
代码清单 2-11
使用 IF、ELSEIF 和 ELSE
```

## 模块

为了使 PowerShell 具有完全的可扩展性并能够管理几乎任何应用程序，它采用模块化方法编写，必须导入模块才能让用户访问它们包含的 cmdlet。执行某些命令时会自动导入一些核心模块。例如，使用 `Get-ExecutionPolicy` 命令会导致 `Microsoft.PowerShell.Security` 模块被导入。

有大量可用的模块，提供了使用 PowerShell 管理几乎任何平台或应用程序的能力。其中一些模块由组织提供。例如，AWS 提供 `AWSPowerShell` 模块来管理其公有云平台，而 Microsoft 提供 `Az` 模块来管理 Azure。

此外，还有大量的社区模块可用，例如 `posh-git` 模块（它提供了 Git 状态摘要以及 Git 命令的 Tab 键补全功能）和 `powershell-yaml` 模块（它在 PowerShell 中提供了 YAML 序列化和反序列化功能）。

当模块发布在 PowerShell Gallery 上时，可以使用 `Install-Module` 命令下载并安装它们，如代码清单 2-12 所示，该命令安装了 `Indented.IniFile` 模块。该模块由出色的 Indented Automation 编写（我强烈建议你去了解他们所有关于 PowerShell 的内容），并提供了在 PowerShell 中管理 INI 文件内容的功能。

```
Install-Module Indented.IniFile
代码清单 2-12
安装 PowerShell 模块
```


### SqlServer 模块

SqlServer 模块作为 SQL Server 管理工具下载的一部分提供，但也可以使用 `Install-Module` 命令进行安装。该模块提供了超过 100 个命令，用于通过 PowerShell 会话与 SQL Server 交互。该模块将在第 4 章深入讨论，但在本节中，我们将简要讨论如何使用该模块来导航 SQL Server 实例。对于以前未使用过 PowerShell 的数据库管理员（DBA）来说，能够导航 SQL Server 实例甚至从 SQL Server Management Studio 内部调用 PowerShell，可能是一个很好的突破性入门方式。

除了使用诸如 `Get-ChildItem` 和 `Set-Location` 等命令导航文件夹结构外，PowerShell 还可用于导航实例的 SQL Server 对象层次结构。你可以通过使用 `Set-Location` 导航到 `SQLSERVER:\SQL`，将 PowerShell 连接到 SQL Server 数据库引擎提供程序。`Get-ChildItem` 返回的信息取决于对象层次结构中的当前位置。表 2-4 详细说明了从层次结构的每一级返回的信息。

#### 表 2-4
每个层次结构级别返回的详细信息

| 位置 | 返回的信息 |
| --- | --- |
| `SQLSERVER:\SQL` | 本地计算机的名称 |
| `SQLSERVER:\SQL\ComputerName` | 本地计算机上安装的数据库引擎实例的名称 |
| `SQLSERVER:\SQL\ComputerName\InstanceName` | 实例级别的对象类型 |
| `更低级别` | 包含在当前位置内的对象类型或对象 |

一旦导航到层次结构的适当级别，你就可以使用 PowerShell 对该级别的对象执行基本操作。例如，清单 2-13 中的脚本将导航到 `AdventureWorks2022` 数据库中的 `tables` 命名空间，并将 `dbo.DatabaseLog` 表重命名为 `dbo.DatabaseLogPS`。`dir` 命令将显示表的原始名称和新名称。

提示：此脚本假设你安装了默认实例的 SQL Server（而非命名实例），并且拥有 `AdventureWorks2022` 数据库，该数据库可从 `learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure` 下载。

```powershell
Set-Location SQLSERVER:\SQL\localhost\Default\Databases\AdventureWorks2022\Tables
dir | where { $_.name -like "*DatabaseLog*" }
Rename-Item -LiteralPath dbo.DatabaseLog -NewName DatabaseLogPS
dir | Where-Object { $_.name -like "*DatabaseLog*" }
```

**清单 2-13**
导航对象层次结构并重命名表

也可以从 SQL Server Management Studio (SSMS) 内部启动 PowerShell。为此，在对象资源管理器中的对象文件夹上右键单击并选择 **启动 PowerShell**。这将调用 PowerShell CLI，其初始位置与你用于调用 CLI 的对象文件夹相匹配。

## 总结

PowerShell 提供了两种操作方式。命令行界面可用于立即执行命令，类似于 Windows 命令提示符。这对于需要执行临时操作或进行快速检查的管理员非常有用。第二种操作方式是脚本编写。管理员或 DevOps 工程师可以构建涉及多个命令或计划自动运行的复杂脚本。

PowerShell 语言通过模块实现，使其具有可扩展性。每个模块包含提供功能的 cmdlet。常用的 cmdlet 提供别名，这些别名可以缩短你必须键入的命令长度，并帮助具有 DOS 或 Linux 经验的新 PowerShell 用户学习该语言。但是，别名应保留用于临时命令。在脚本中使用时，它们会使代码难以理解和管理。

PowerShell 中的每个变量都是一个对象，并关联有数据类型。由于这些对象映射到 .NET 类，因此可以使用诸如 `ToString()` 等方法。PowerShell 还提供了一个丰富的框架来控制脚本的执行流。这包括管道，它允许你将一个命令的结果直接传递给下一个命令。

`SqlServer` 模块为 SQL Server 提供支持。加载该模块后，它会公开一组基于 SQL Server 的 cmdlet，允许你从命令 shell 或通过自动化脚本管理 SQL Server。它还允许你导航 SQL Server 实例的对象层次结构，并对对象执行基本任务。

