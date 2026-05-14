# 第十章 - 脚本编写

### 输入列页面

在编辑器的“输入列”页面中，您可以选择输入列和输出列。您选择的输入列可在脚本内部访问；所有其他列将“传递通过”该组件，到达数据流中的下一个组件。您也可以在此选项卡上向数据流添加输出列。在我们的简单示例中，我们向输出添加了一个名为 `RowNumber` 的四字节有符号整数 (`DT_I4`) 类型列，如图 10-21 所示。这个新列将保存分配给每一行的编号，从 1 开始。

图 10-21. 向脚本组件输出添加新列

### 脚本组件同步转换

脚本组件同步转换重写了三个方法：

-   与脚本组件源类似，异步转换类型重写了 ` PreExecute()` 方法。此方法在脚本组件的开头、任何数据处理之前精确执行一次。此方法对于在转换中执行初始化任务非常有用。
-   脚本组件同步转换也支持 `PostExecute()` 方法。此方法在组件处理完所有数据后精确执行一次。此方法允许您在转换完成后执行任何清理步骤。
-   最后，转换提供了 `ProcessInputRow()` 方法，其名称前缀是您的输入名称。在我们的示例中，输入名为 Input 0，`ProcessInputRow()` 方法被命名为 `Input0_ProcessInputRow()`。在同步转换中，对于通过它的每一行数据，都会调用一次此方法。此方法是转换中“发生魔法”的地方。

### 示例脚本分析

我们在此脚本组件中使用的脚本首先声明了一个名为 `row_number` 的类级整数变量来跟踪行数，如下所示：

```csharp
[Microsoft.SqlServer.Dts.Pipeline.SSISScriptComponentEntryPointAttribute]
public class ScriptMain : UserComponent
{
    int row_number;
    ...
}
```

` PreExecute()` 方法调用了 `base.PreExecute()` 方法并将 `row_number` 变量初始化为 1：

```csharp
public override void PreExecute()
{
    // Execute base PreExecute() method
    base.PreExecute();
    // Initialize row number
    row_number = 1;
}
```

`PostExecute()` 方法只是调用了 `base.PostExecute()` 方法：

```csharp
public override void PostExecute()
{
    base.PostExecute();
}
```

`Input0_ProcessRow()` 方法对输入中的每一行调用一次。对于传入的每一行，我们将 `row_number` 变量的值分配给数据流的 `RowNumber` 列，然后将 `row_number` 递增 1。

```csharp
public override void Input0_ProcessInputRow(Input0Buffer Row)
{
    // Assign row_number to RowNumber column
    Row.RowNumber = row_number;
    // Increment row_number
    row_number++;
}
```

我们简单的行编号脚本组件的结果如图 10-22 所示。

图 10-22. 行编号脚本组件的结果

### 异步脚本组件转换

除了同步转换，脚本组件还允许您创建强大的异步转换。正如我们在第 8 章讨论的，当您想通过透视、聚合或添加和删除行来更改数据集的“形状”时，异步转换非常有用。此示例扩展了先前的包，将一个新的异步脚本组件集成到数据流中。我们将解析文章的 `Title` 和 `Description` 列为单个单词，并将它们输出到单个列中，同时为每个单词提供一个出现计数。在现实世界中，这种类型的转换可用于执行文本数据挖掘和分析。

我们通过向任务添加另一个脚本组件开始此示例，就像添加同步脚本组件转换时一样。与之前一样，我们从对话框中选择了“转换”选项。我们将数据流从同步行编号脚本组件定向到新的脚本组件。我们还添加了一个新的“行数”转换来捕获第二个输出的输入，并在其输出上启用了数据查看器。结果如图 10-23 所示。

图 10-23. 将新的单词解析脚本组件添加到数据流

在此示例中，我们将介绍脚本转换的两个特性：(1) 使用异步转换处理输入数据，以及 (2) 多个输出。对于其中一个输出，我们将其重命名为 `RssFeedOutput`，我们将以检索时的相同形式输出 RSS 源文章。该转换还将从 RSS 源的 `Description` 列中提取关键词，并计算该词在每篇文章中的出现次数。结果被发送到第二个输出，我们将其重命名为 `KeywordOutput`。

将新的脚本组件拖到设计器界面后，我们进入了编辑器的“输入列”页面。我们选择了来自数据流的所有入站列，使它们可用于脚本组件，如图 10-24 所示。

图 10-24. 将入站列添加到脚本组件的输入列中

将入站列添加到脚本组件的输入列后，我们转到了“输入和输出”页面。在那里，我们向“输入和输出”页面添加了第二个输出。然后我们将两个输出上的 `SynchronousOutputID` 更改为 `None`。这会将它们与输入断开连接并使它们成为异步。

**提示**：虽然您可以向脚本组件转换添加多个输出，但您只能有一个输入。如果您需要多个输入（一个例子是 Union All 组件），您将需要创建一个自定义组件，这将在第 22 章讨论。

默认情况下，输出名为 Output 0 和 Output 1——对于此示例，我们将它们重命名为 `RssFeedOutput` 和 `KeywordOutput`，以更好地反映我们将要推入其中的数据。然后我们在每个输出下创建了输出列。`RssFeedOutput` 将包含 RSS 源内容，就像它们进入组件时一样，而 `KeywordOutput` 将保存所有解析的关键词。您可以在图 10-25 中看到具有两个异步输出的“输入和输出”页面。

图 10-25. 向脚本组件添加两个异步输出

在构建脚本组件同步转换时，如我们之前所见，默认情况下，入站列会流向输出。这是因为在同步转换中，输入和输出缓冲区是相同的。您只需要显式命名那些您的转换添加到输出的列。在异步转换中，输入和输出是断开的，因此您需要为您想要在每个输出中出现的每一个列显式创建。在我们的示例中，`RssFeedOutput` 列镜像输入列，而 `KeywordOutput` 仅包含四列，如图 10-26 所示。表 10-1 列出了 `KeywordOutput` 列的详细信息。

图 10-26. 向脚本组件的两个输出添加列

表 10-1. KeywordOutput 列列表

| 列名 | 数据类型 | 内容 |
| :--- | :--- | :--- |
| `RowNumber` | 四字节有符号整数 (`DT_I4`) | 分配给输入每一行的 `RowNumber` 值的副本 |
| `KeyWordNumber` | 四字节有符号整数 (`DT_I4`) | 分配给在 RSS 数据 `Description` 列中找到的每个关键词的唯一编号 |

