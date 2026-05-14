# 第九章：变量、参数和表达式

#### 包系统变量

| 名称 | 数据类型 | 描述 |
| :--- | :--- | :--- |
| `InteractiveMode` | `Boolean` | 指示包是否以交互模式执行。交互模式指使用 `SSIS 设计器` (`True`) 执行还是使用 `DTExec 命令行` 工具 (`False`)。 |
| `LocaleID` | `Int32` | 标识包正在使用的区域设置。默认为 `英语(美国)`。 |
| `MachineName` | `String` | 正在执行包的计算机的名称。 |
| `OfflineMode` | `Boolean` | 指示包是否处于脱机模式且未获取数据源的连接。 |
| `PackageID` | `String` | 包的唯一标识符。该字符串表示十六进制值。 |
| `PackageName` | `String` | 不含文件扩展名的包名称。 |
| `StartTime` | `DateTime` | 指示包开始执行的时间戳。 |
| `UserName` | `String` | 当前执行包的帐户用户名。 |
| `VersionBuild` | `Int32` | 包的版本。 |
| `VersionComment` | `String` | 显示有关包版本的注释。 |
| `VersionGUID` | `String` | 唯一标识包的版本。该字符串表示十六进制值。 |
| `VersionMajor` | `Int32` | 标识包的主版本。 |
| `VersionMinor` | `Int32` | 标识包的次版本。 |

#### 容器系统变量

*容器* 只有一个特定于它们的系统变量，即 `LocaleID`。这个 32 位整数标识容器使用的区域设置。此变量可用于 `Sequence`、`For Loop` 和 `Foreach Loop` 容器中。在容器内访问 `LocaleID` 变量将默认为此值，而不是包的 `LocaleID`。

#### 任务系统变量

控制流任务在创建后会获得自己的一组系统变量。除了 `Data Flow` 任务外，大多数任务通常不能直接访问这些变量。`Data Flow` 任务允许你通过使用 `Script 转换任务`、`派生列转换` 和 `条件拆分转换` 来访问其变量。使用这些转换，你可以访问这些变量并将其值用作数据流的一部分。表 9-3 显示了任务可用的所有系统变量。

**表 9-3. 任务系统变量**

| 名称 | 数据类型 | 描述 |
| :--- | :--- | :--- |
| `CreationName` | `String` | `SSIS` 所识别的任务名称。 |
| `LocaleID` | `Int32` | 标识任务使用的区域设置。默认为 `英语(美国)`。 |
| `TaskID` | `String` | 任务的唯一标识符。该字符串表示十六进制值。 |
| `TaskName` | `String` | 在设计时赋予任务的唯一名称。 |
| `TaskTransactionOption` | `Int32` | 标识任务正在使用的事务选项。 |

#### 事件处理程序系统变量

`SSIS` 中最后一类可执行文件，即事件处理程序，也有自己的一组系统变量。这些变量通常不适用于所有事件处理程序。适用于任务的一般规则也适用于事件处理程序系统变量。表 9-4 描述了所有事件处理程序系统变量。

**表 9-4. 事件处理程序系统变量**

| 名称 | 数据类型 | 描述 |
| :--- | :--- | :--- |
| `Cancel` | `Boolean` | 标识事件处理程序是否在错误、警告或查询取消后停止执行。仅适用于 `OnError`、`OnWarning` 和 `OnQueryCancel`。 |
| `ErrorCode` | `Int32` | 标识发生的错误代码。仅适用于 `OnError`、`OnInformation` 和 `OnWarning`。 |
| `ErrorDescription` | `String` | 描述错误事件的字符串。仅适用于 `OnError`、`OnInformation` 和 `OnWarning`。 |
| `ExecutionStatus` | `Boolean` | 标识事件当前是否正在执行。仅适用于 `OnExecStatusChanged`。 |
| `ExecutionValue` | `DBNull` | 返回执行值。仅适用于 `OnTaskFailed`。 |
| `LocaleID` | `Int32` | 标识事件处理程序使用的区域设置。适用于所有事件处理程序。 |
| `PercentComplete` | `Int32` | 返回事件处理程序完成的进度。仅适用于 `OnProgress`。 |


