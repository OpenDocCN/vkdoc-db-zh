# 第 10 章 - 脚本编写

`CreateNewOutputRow()`方法是实际进行处理的地方。首先，我们获取了 RSS 源的头部信息，包括源标题、链接和发布日期，并将其放入字符串变量中。然后，我们循环遍历 RSS 源中的`<item>`标签，每个标签代表一篇联合发布的文章，并将文章标题、链接和描述放入字符串变量中。在循环中，我们调用了输出缓冲区的`AddRow()`方法，并将从 XML 文档中提取的字符串值分配给输出缓冲区列。循环完成后，并且所有行都已添加到输出后，我们在缓冲区上调用`SetEndOfRowset()`方法，以通知 SSIS 不再有更多行传入。

**提示：** 即使你现在认为不需要任何特殊的错误处理，在 .NET 代码周围放置 try-catch 块将使将来更容易地添加针对错误的特殊处理。

#### 脚本组件事件

与脚本任务类似，脚本组件允许您引发 SSIS 根据您的设置可能记录的事件。在脚本任务中，事件引发方法通过`Dts.Events`对象访问。在脚本组件中，它们通过`ComponentMetaData`对象访问。`ComponentMetaData`对象公开的方法包括：

-   `FireError()`引发`OnError`事件，该事件指示一个影响处理并可能阻止包运行的严重状况。请注意，`FireError()`本身不会停止数据流运行，但当错误向上传播回数据流任务时，SSIS 将根据其属性决定控制流是否会失败。要在数据流中强制硬停止，您应该像我们在示例中所做的那样抛出异常。
-   `FireWarning()`引发`OnWarning`事件，该事件指示一个不如错误严重但仍可能影响处理的问题。
-   `FireInformation()`引发`OnInformation`事件，其中包含信息性消息。这些消息通常对调试和排除故障问题很有用。
-   `FireProgress()`引发`OnProgress`事件，这本身指示组件进度。我们在示例中使用了`FireProgress()`事件。
-   `FireCustomEvent()`引发您在包内创建的自定义事件。

`ComponentMetaData`对象公开的事件引发方法在调试时极其有用，我们强烈建议您在脚本组件中广泛使用它们。

[www.it-ebooks.info](http://www.it-ebooks.info/)

在此过程中，我们持续记录已处理的行数，并通过使用`FireProgress()`方法引发`OnProgress`事件来报告进度。与`OnInformation`类似，这些事件对排除故障和调试代码很有用。如果在此代码中遇到异常，通常涉及单行，因此我们调用`FireError()`来记录错误，并让 SSIS 决定是否在任务级别停止处理。代码清单如下：

```csharp
public override void CreateNewOutputRows()
{
    // 为启动引发 OnInformation 事件
    this.ComponentMetaData.FireInformation(-1, ComponentName,
        "Start - Create new output rows", "", 0, ref b);

    try
    {
        // 获取源属性
        string feed_title = RssXML.SelectSingleNode("(/rss/channel/title)[1]").InnerText;
        string feed_link = RssXML.SelectSingleNode("(/rss/channel/link)[1]").InnerText;
        string feed_language = RssXML.SelectSingleNode("(/rss/channel/language)[1]").InnerText;
        string feed_pubDate = RssXML.SelectSingleNode("(/rss/channel/pubDate)[1]").InnerText;

        // 初始化计数器并解析文章
        int CurrentRowCount = 0;
        int CurrentPercent = 0;
        int LastPercent = 0;
        XmlNodeList xl = RssXML.SelectNodes("/rss/channel/item");
        int TotalRows = xl.Count;

        // 遍历文章
        foreach (XmlNode xn in xl)
        {
            // 获取文章属性
            string title = xn.SelectSingleNode("(title)[1]").InnerText;
            string link = xn.SelectSingleNode("(link)[1]").InnerText;
            string description = xn.SelectSingleNode("(description)[1]").InnerText;

            // 创建新的输出行
            Output0Buffer.AddRow();

            // 为输出缓冲区列赋值
```



