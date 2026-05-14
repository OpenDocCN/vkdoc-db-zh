# 第 10 章 – 脚本编写

### 列定义

| 列名          | 数据类型                        | 内容                                           |
| :------------ | :------------------------------ | :--------------------------------------------- |
| **KeyWord**   | 字符串 [DT_STR], 长度 100       | 在描述列中找到的关键词                         |
| **Occurrences** | 四字节有符号整数 [DT_I4]       | 每篇文章的描述列中该关键词出现的次数           |

### 脚本组件方法

在脚本组件异步转换中，我们需要重写四个主要方法：

*   `PreExecute()` 方法在处理任何行之前，为组件触发一次。
*   `PostExecute()` 方法在所有行处理完毕后，为组件触发一次。
*   `ProcessInput()` 方法在组件执行期间触发一次。该方法遍历缓冲区中的行，并调用另一个方法 `ProcessInputRow()` 来处理它们。`ProcessInput()` 方法以输入名称为前缀；在我们的示例中，它是 `Input0_ProcessInput()`。
*   `ProcessInputRow()` 方法为传递给它的每一行触发一次。通常，这将是输入缓冲区中的每一行触发一次，但这取决于 `ProcessInput()` 方法。`ProcessInputRow()` 方法也以输入名称为前缀。在我们的示例中，它被命名为 `Input0_ProcessInputRow()`。

我们的示例中的 `PreExecute()` 和 `PostExecute()` 方法只是调用此处显示的基础方法：

```csharp
public override void PreExecute()
{
    // 调用基方法
    base.PreExecute();
}

public override void PostExecute()
{
    // 调用基方法
    base.PostExecute();
}
```

`Input0_ProcessInput()` 方法使用 while 循环和输入缓冲区的 `NextRow()` 方法来遍历传入的行。它沿途为每一行调用 `Input0_ProcessInputRow()` 方法。

在输入缓冲区完全处理完毕后，调用基 `ProcessInput()` 方法，然后为两个输出缓冲区调用 `SetEndOfRowset()` 方法，以向 SSIS 运行时发出信号，表明所有行都已通过脚本组件处理完毕。

```csharp
public override void Input0_ProcessInput(Input0Buffer Buffer)
{
    // 获取组件名称
    ComponentName = this.ComponentMetaData.Name;

    // 遍历行并一次处理一行
    while (Buffer.NextRow())
    {
        Input0_ProcessInputRow(Buffer);
    }

    // 调用基方法
    base.Input0_ProcessInput(Buffer);

    // 在输入缓冲区结束时，为两个输出设置行集结束标记
    if (Buffer.EndOfRowset())
    {
        KeywordOutputBuffer.SetEndOfRowset();
        RssFeedOutputBuffer.SetEndOfRowset();
    }
}
```

最后，`Input0_ProcessInputRow()` 方法处理缓冲区中的每一传入行。在我们的示例中，我们为每一行向 `RssFeedOutputBuffer` 添加一个新行，然后将传入的值分配给输出缓冲区的相应列。对于 `KeywordOutputBuffer`，我们做了更多的工作。

首先，我们清理传入的 `Description` 列，消除除空格外的所有标点符号。然后，我们使用 .NET 的 `Regex`（正则表达式）类将文本拆分为单个单词。接着，我们遍历这些单词并进行计数，将单词及其出现次数存储在 .NET 的 `Hashtable` 中。

最后一步是遍历哈希表中的条目并将它们添加到输出中。

> **注意**：关于 .NET `Regex` 和 `Hashtable` 类的详细讨论超出了本书的范围。详细信息可以在 MSDN（网址为 `http://msdn.microsoft.com`）上搜索找到。

```csharp
public override void Input0_ProcessInputRow(Input0Buffer Row)
{
    bool b = true;

    try
    {
        // 向 RssFeedOutputBuffer 添加一个新行
        RssFeedOutputBuffer.AddRow();

        // 在输出上设置新行的列
        RssFeedOutputBuffer.Description = Row.Description;
        RssFeedOutputBuffer.FeedLanguage = Row.FeedLanguage;
        RssFeedOutputBuffer.FeedLink = Row.FeedLink;
        RssFeedOutputBuffer.FeedPubDate = Row.FeedPubDate;
        RssFeedOutputBuffer.FeedTitle = Row.FeedTitle;
        RssFeedOutputBuffer.Link = Row.Link;
        RssFeedOutputBuffer.RowNumber = Row.RowNumber;
        RssFeedOutputBuffer.Title = Row.Title;

        // 从描述中消除除空格外的所有标点符号
```


