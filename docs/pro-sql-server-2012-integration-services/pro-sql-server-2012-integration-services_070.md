# 第 9 章  变量、参数和表达式

#### 包系统变量

属于包的系统变量显示有关包本身的重要信息。包内定义的所有对象都可以访问这些变量。表 9-2 列出了所有包系统变量及其数据类型。

*表 9-2. 包系统变量*

**名称** | **数据类型** | **描述**
--- | --- | ---
`CancelEvent` | `Int32` | 一个 Windows 事件对象的标识符，用于向任务发出停止执行的信号。
`CreationDate` | `DateTime` | 包的创建日期。
`CreatorComputerName` | `String` | 创建包时所使用的计算机。
`CreatorName` | `String` | 包创建者的用户名。
`ExecutionInstanceGUID` | `String` | 包执行的唯一标识符。该字符串表示十六进制值。


