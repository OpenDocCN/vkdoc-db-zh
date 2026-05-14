# 第四章

## 调试与 ViewState

为了确切了解发生了什么，我在 `PersonListingControl` 和 `PersonRow` 类的各个事件处理方法以及 `CreateChildControl` 方法的开头设置了断点。然后，我在包含单个 `PersonListingControl` 的页面上启动了调试进程。调试器启动后，我添加了一些监视项，如 `dataSource`、`dataBinding`、`_firstName` 和 `IsViewStateEnabled`，以便在每一步查看它们的值。

通常，在 `ViewState` 启用的情况下，回发事件如果不会特别引发加载新数据（例如当 `PageIndex` 改变时），则回发只会导致 `CreateChildControls` 方法被调用一次。在 `ViewState` 禁用的情况下，我发现 `CreateChildControls` 方法在 `PreRender` 事件期间被第二次调用，并且 `dataSource` 参数已定义，这使得 `PersonRow` 项获得了它们所需的值。如 **图 4-5** 所示，`ViewState` 被禁用，并且 `_firstName` 成员变量已被定义。这个值是在第二次调用 `CreateChildControls` 方法时设置的。

`图 4-5.` 在没有 `ViewState` 的情况下调试

使用这个示例数据绑定控件，你现在可以尝试自己的实验，并在调试会话中逐步跟踪代码时准确观察发生的情况。你可以看到 `ViewState` 如何影响性能，而将 `ViewState` 移到 `Session` 可以带来一些好处，同时也伴随着不同的风险。为你的应用程序选择正确的数据绑定控件设置组合，能够以最小的努力带来一些性能改进。

## 常用文件夹补充

这个自定义数据绑定控件是一个有用的代码示例，你可以将其放入 Templates 子文件夹的 Common 文件夹中（`D:\Projects\Common\Templates\Databound Control`）。虽然你可能不会构建自定义数据绑定控件，但你偶尔会希望创建一个更有效地使用 `ViewState` 的控件，甚至可能禁用 `ViewState`。此代码可与本书其余示例代码一起下载。

## 小结

在本章中，你了解了 ASP.NET 所有可用的数据绑定选项。你研究了几种标准控件，将它们与一系列用户控件结合使用，并最终创建了自己的自定义数据绑定控件。有了这些选择，你现在可以决定哪些数据绑定控件适合你的需求，并且通过选择正确的控件以及 `ViewState` 的最佳设置，你可以构建一个更高效的表现层。

