# 5. 业务含义

含义部分实际上由业务所有。业务人员有权决定他们的术语。应用程序或微服务面向业务，应该使用业务的语言。数据内容也是如此。



## API 中的数据名称至关重要

由于许多物理数据存储可能是**无模式**或**读取时模式**的，即其“模式”在读取数据时才被推断出来，因此我们需要为业务用户提供一个使用业务术语的**表示层**。这个层就是我们的 `GraphQL` API。

更具挑战性的是，人们想要分析的（大）数据大部分或多或少是机器或系统生成的。这再次要求存在一个面向用户、面向业务环境的数据术语层。最后，即使在集成项目中，标准化业务名称对于在业务用户群体中获得认可也是必要的。

名称包括对象类型名称（例如 `originator`）、属性名称（例如地址的 `display name`）以及关系名称（例如 `cc`）。

关系名称很重要，因为它们也被用来推断关系的类型（依赖）。动作动词表示在业务对象上进行的活动，而像“has”这样的弱动词则表示业务对象类型的标识符与其属性之间普通的函数依赖关系。

关系是双向的，如图 5-1 所示。

![../images/471428_1_En_5_Chapter/471428_1_En_5_Fig1_HTML.jpg](img/471428_1_En_5_Fig1_HTML.jpg)

**图 5-1** 从右向左或从左向右读取

基于这些原因，名称很重要。然而，我倾向于只写下来自父级的名称。在这个例子中，即 `originator`，因为应用了从左到右的顺序（西方文化惯例）。

重要的是，一些连接短语（关系的名称）包含动词，而动词暗示是一个“行动者”使某事发生。换句话说，该关系表示某种事物，这不仅仅是依赖于某物身份的普通属性。这使得目标有资格被识别为业务对象类型。

在存在歧义风险的地方，名称应有坚实的定义（作为注释或描述）作为支撑。核心业务概念的定义持续时间长，影响范围广。这意味着它们不仅应该精确且切中要害，还应该易于沟通和记忆。

名称应尽可能具体。例如，如果你讨论的是 `Destination Address`，就应避免使用 `Address`。

请注意，`GraphQL` 通过定义接口提供了一种简单而好的方式来支持子类型化。在电子邮件示例中，我们可以考虑将 `Originator` 和 `Destination` 设置为接口，两者都实现一个 `Contact` 类型。看起来如下。

首先是两个类型：

```
1   type Originator {
2     id: ID!
3     origin_date: Datetime!
4     message: : [Message!]  @relation(name: "Originator")
5     address_from: Address! @relation(name: "From")
6     address_sender: Address @relation(name: "Sender")
7     address_reply_to: Address @relation(name: "ReplyTo")
8   }

10   type Destination {
11     id: ID!
12     destination_role: destination_role!
13     received_date: Datetime!
14     message: Message! @relation(name: "Destination")
15     address_to: [Address]! @relation(name: "To")
16     address_cc: [Address] @relation(name: "Cc")
17     address_bcc: [Address] @relation(name: "Bcc")
18   }
```

然后是一个接口构造：

```
1   interface Contact {
2     id: ID!
3   }

5   type Originator implements Contact {
6   origin_date: Datetime!
7     message: [Message!] @relation(name: "Originator")
8     address_from: Address! @relation(name: "From")
9     address_sender: Address @relation(name: "Sender")
10     address_reply_to: Address @relation(name: "ReplyTo")
11   }

13   type Destination implements Contact {
14   destination_role: destination_role!
15     received_date: Datetime!
16     message: Message! @relation(name: "Destination")
17     address_to: [Address]! @relation(name: "To")
18     address_cc: [Address] @relation(name: "Cc")
19     address_bcc: [Address] @relation(name: "Bcc")
20   }
```

我们在电子邮件设计中没有使用接口，因为两者之间的重叠极少。然而，接口名称也应该是有效的业务名称。

API 返回结果的内容质量取决于业务语义以及 API 实际提供的数据。我们已经处理了属性图中的结构和术语，接下来我们需要妥善处理实际的数据内容。但是，请记住，意义和内容是相辅相成的。如果你改变了语义，那么你可能需要重构数据。

### 注意

与一些领域专家一起检查属性图内容的名称，以理清语义。

当业务术语与数据库中的数据名称不匹配时，需要有人在解析器层面采取措施。更多内容稍后讨论。

## 寻找标准数据结构

在设置模式之前，另一件需要考虑的事情是标准模式。许多不同类型的业务活动及其对象的最佳实践数据模型已经被定义。

例如，`Company Location` 属性似乎设计得有点不足。位置是地理地址，即业务地址。对于许多通用对象类型，已经存在多种最佳实践。例如，可以参考 `schema.org` 或各种国家或国际标准。

我们如何发现例如公司位置相关信息有所缺失？一个指标可能是：精确的连接短语包含一个动作动词：`located at`。动作动词往往表示对象类型之间的关系，而不是对象与其属性之间的关系。

因此，在发布你的模式之前，寻找标准模式是另一项需要进行的检查。


