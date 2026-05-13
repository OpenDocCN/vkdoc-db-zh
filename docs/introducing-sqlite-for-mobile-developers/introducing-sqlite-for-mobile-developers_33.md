# 第 8 章 ■ 在 iOS 和 OS X 中使用 Core Data 与 SQLite

### 处理属性

你可以（并且应该）使用有意义的名称来重命名你的属性。（参见侧边栏“命名 Core Data 模型对象”。）

#### 命名 Core Data 模型对象

按照惯例，Core Data 实体的名称以大写字母开头。此外，它们通常是单数形式。因此，一个跟踪用户的实体通常被称为 `User`，而一个跟踪用户分数的实体通常被称为 `Score`。

如果你在 `User` 和 `Score` 之间有一个`对多`关系，使得一个用户可以有多个分数，那么该*关系*被命名为 `scores`（复数形式）。

这些在 Core Data 中已经是一段时间内的惯例（并且它们与其他环境中的惯例和最佳实践并无不同）。在 Xcode Core Data 中，这个惯例已成为一项要求：你不能将实体命名为 `user`（小写），也不能将该实体中的属性命名为 `Name`（大写）。

请注意，尽管 Core Data 的惯例与其他环境中的惯例相似，但这种相似性仅限于存在大小写、单复数等标准。是否要大写等具体细节是存在差异的。

每个属性都必须有一个类型。Core Data 中可用的类型如下：

*   `Integer16`、`Integer32` 和 `Integer64`
*   `Decimal`
*   `Double`
*   `Float`
*   `String`
*   `Boolean`
*   `Binary Data`
*   `Transformable`

存在一个初始的 `Undefined` 值，但当你尝试保存数据模型时，将被要求将其更改为适当的值。

![](img/index-80_1.jpg)

