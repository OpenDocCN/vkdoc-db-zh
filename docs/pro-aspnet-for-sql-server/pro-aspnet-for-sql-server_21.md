# 第四章 ■ 数据绑定控件

你也可以实现自己的自定义 `PageStatePersister`，其工作方式与 `SessionPageStatePersister` 类似，但不是将数据限制为十项，而是可以将较旧的数据存储到数据库中以备需要。可以通过计划任务在每晚清除这些数据。

#### 分页

也可以通过分页来减小 `ViewState` 的大小。考虑一个总共有 1,000 项、页面大小设置为 10 项的 `GridView`。只有这 10 项的数据会持久化到 `ViewState` 中，而不是全部 1,000 项的数据。当分页与上一章介绍的子集选择技术结合使用时，可以确保仅从数据库中提取显示的 10 行数据，从而减轻数据库负载。稍后在“创建数据绑定控件”一节中构建自定义数据绑定控件时，你将了解更多关于分页优势的内容。

## 禁用 ViewState

另一种减小页面大小的方法是禁用 `ViewState`。你可以对整个页面进行禁用，也可以通过 `EnableViewState` 属性在控件级别进行禁用。如果你为页面禁用 `ViewState`，该决定将影响页面上包含的每个控件。每个控件都无法覆盖该设置，因此影响是全局性的，并且对于需要 `ViewState` 的控件来说可能具有破坏性。对于任何包含下级控件的控件也是如此。该设置会影响其下的所有控件。

由于如果整个页面的 `ViewState` 被禁用，就无法为某个控件启用 `ViewState`，因此你需要识别出那些不需要它的单个控件。每个页面上都包含的导航控件（可能作为母版页的一部分）就是一个理想的候选者。如果它只包含标记和一些以声明方式设置了 `NavigateUrl` 的 `HyperLink` 控件，那么即使在 `ViewState` 被禁用时，该值在回发期间也将可用。

对于自定义用户控件，你也可以将其设计为在有和没有 `ViewState` 的情况下都能工作。清单 4-17 展示了如何做到这一点。

**清单 4-17.** 在有和没有 `ViewState` 的情况下工作
```
if (! IsPostBack)
{
    GridView1.DataSource = GetDataSource();
    GridView1.DataBind();
}
else if (! IsViewStateEnabled)
{
    GridView1.DataSource = GetDataSource();
    GridView1.DataBind();
}
```

`GridView` 的数据源在首次页面加载（非回发时）以及当 `ViewState` 被禁用的回发时设置。这样做将在控件需要数据时加载 `GridView` 数据。然而，这种方法相当手动。最好简单地使用声明的 `ObjectDataSource` 引用。这样做将允许 `GridView` 自动检测 `ViewState` 何时被禁用，并在需要数据时使用 `ObjectDataSource` 进行绑定。

## ControlState 与 ViewState

因为某些数据对于控件的功能至关重要，所以即使在 `ViewState` 启用时，也需要保留对这些细节的访问权限。像 `GridView` 这样的控件足够智能，可以在有和没有 `ViewState` 的情况下工作，但当启用分页时，它们仍然需要保留与位置相关的控件当前状态。每次回发时都可以从数据库提取各列的数据，但数据库不会知道选择了 `GridView` 的第四页。这个值通过 `SelectedItem` 属性保留，该属性与 `ControlState`（`ViewState` 的一种变体）一起持久化。

`ControlState` 随 ASP.NET 2.0 引入。它不像 `ViewState` 那样是自动特性。你必须自己实现它，并向 `Page` 注册以持久化数据。

清单 4-18 展示了如何保存 `ControlState`。

**清单 4-18.** SaveControlState
```
protected override object SaveControlState()
{
    Pair state = new Pair();
    state.First = base.SaveControlState();
    state.Second = _controlStateData;
    return state;
}
```

`ControlState` 和 `ViewState` 通常使用 `System.Web.UI` 命名空间中的 `Pair` 和 `Triplet` 类型进行持久化。它们只是保存数据的简单小型数组。你也可以使用可序列化的对象数组，这意味着最好坚持使用 `string`、`int` 和 `DateTime` 等类型。你需要使用最少的数据来持久化必要的状态。使用 `Pair` 类型是一个很好的第一层，它将基类的 `ControlState` 存储在 `First` 属性中，将当前实例的状态存储在 `Second` 属性中。如果你有许多值需要持久化，存储在 `Second` 属性中的值可以是许多对象的数组。然后，在加载时将展开此状态的层次结构，如清单 4-19 所示。

**清单 4-19.** LoadControlState
```
protected override void LoadControlState(object savedState)
{
    Pair pair = savedState as Pair;
    if (pair != null)
    {
        base.LoadControlState(pair.First);
        _controlStateData = (string)pair.Second;
    }
    else
    {
        base.LoadControlState(null);
    }
}
```

状态被强制转换为 `Pair` 类型，并用于使用 `First` 属性加载基类的 `ControlState`，然后使用 `Second` 属性加载当前实例的状态。定义了这些值后，需要告诉 `Page` 该控件需要 `ControlState`。这必须在 `Init` 事件期间完成，在事件生命周期的后期触发加载 `ControlState` 的事件之前，如清单 4-20 所示。

**清单 4-20.** 要求 ControlState
```
protected void Page_Init(object sender, EventArgs e)
{
    Page.RegisterRequiresControlState(this);
}
```

前面关于 `ControlState` 的示例只是持久化了一个 `string` 变量。很多时候你不应该有太多数据需要持久化。事件排序将导致 `ControlState` 在 `Init` 和 `Load` 事件之间持久化和加载，因此当你的 `Load` 事件触发时，持久化的数据将是可用的。稍后将触发 `PreRender` 事件，大多数数据绑定控件实际上在此处加载数据。了解这些数据何时对你的页面或控件可用非常重要。

考虑到可以选择将会话状态持久化在 `Session` 或 `ControlState` 中，应该注意并非所有状态都必须持久化。一些数据可以安全丢失，因为使用它们的输入字段已经在持久化数据，例如 `TextBox`。来自表单的值将始终在 `Text` 属性中设置。然而，`TextChanged` 事件现在每次回发都会触发，因为控件无法比较值是否确实发生了变化。

如果你的控件确实处理此事件，那么 `ViewState` 被禁用将不会影响你。

其他控件，如 `ListBox` 和 `RadioButtonList`，可能是以声明方式填充的，而不是绑定到数据源。这些控件的工作方式与 `TextBox` 类似，它们可以正确地向你提供当前选定的值，并使用声明的值用所有可用选项重绘自身。同样，当选择改变时触发的事件将不像启用 `ViewState` 时那样工作。无论哪种情况，你仍然可以获取用户设置的值并根据需要采取行动。

#### 创建数据绑定控件

为了加深你对数据绑定控件工作原理的理解，你将学习如何创建一个将绑定到 `ObjectDataSource` 的控件。你将能够使用这个控件做比标准控件（如 `GridView` 和 `DetailsView`）更多的事情，因为你将拥有完整的源代码，可以修改并使用调试器跟踪。你还可以直接看到那些修改带来的后果。



这个名为 `PersonListingControl` 的新控件将接收与人员和位置相关的多个字段，并以两种格式之一进行显示。它还将提供可选的**分页支持**，该功能与数据源协调工作，通过控件上的 `DataSourceID` 属性进行声明式定义。无论是否启用 `ViewState`，它都能正常工作。

此控件将通过诸如 `GridView` 等功能丰富的控件来实现大量功能。

## 为何要创建自定义数据绑定控件？

此数据绑定控件的目的纯粹是作为一种学习工具。你很难想到许多无法使用现有数据绑定控件（从 `GridView` 到 `Repeater`）来满足需求的场景。使用这些控件无法做到的是，通过调试器逐步执行它们，以查看内部究竟发生了什么，以及进行各种测试实验，看看如何改进流程。因为此数据绑定控件同样继承自 `CompositeDataBoundControl`，它的行为将类似于标准的数据绑定控件。

然而，你可以将此示例作为基础，创建一个实用的解决方案，使其紧密贴合你的应用程序进行定制，从而利用所有可用的优势。仅仅将用户控件转换为服务器控件，你可能看不到显著的性能提升。借助此示例入门，你或许能够比较两者并衡量其差异。

8601Ch04CMP3 8/25/07 1:07 PM Page 92

92

