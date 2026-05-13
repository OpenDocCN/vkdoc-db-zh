# MongoDB 数据库与集合操作

## 6.1 创建与插入

尽管上下文已切换至 `MYDBPOC`，但若执行 `show dbs` 命令，数据库名称不会显示。这是因为 MongoDB 仅在数据插入数据库后才真正创建该数据库。此行为符合 MongoDB 动态分配命名空间、简化并加速开发流程的动态数据处理理念。此时若执行 `show dbs` 命令，`MYDBPOC` 数据库将不会出现在数据库列表中。

以下示例假设有一个名为 `users` 的多态集合，其中包含以下三种原型的文档：

```json
{
  "_id": ObjectID(),
  "FName": "First Name",
  "LName": "Last Name",
  "Age": 30,
  "Gender": "M",
  "Country": "Country"
}
```

以及

```json
{
  "_id": ObjectID(),
  "Name": "Full Name",
  "Age": 30,
  "Gender": "M",
  "Country": "Country"
}
```

以及

```json
{
  "_id": ObjectID(),
  "Name": "Full Name",
  "Age": 30
}
```

### 6.1.1 创建与插入操作

现在，我们将了解数据库和集合是如何创建的。如前所述，MongoDB 中的文档采用 JSON 格式。

首先，通过执行 `db` 命令，可以确认当前上下文是 `mydbpoc` 数据库。

```
> db
mydbpoc
>
```

现在来看如何创建文档。

第一个文档符合第一种原型，而第二个文档符合第二种原型。你创建了两个名为 `user1` 和 `user2` 的文档。

```
> user1 = {FName: "Test", LName: "User", Age:30, Gender: "M", Country: "US"}
{
  "FName" : "Test",
  "LName" : "User",
  "Age" : 30,
  "Gender" : "M",
  "Country" : "US"
}
> user2 = {Name: "Test User", Age:45, Gender: "F", Country: "US"}
{ "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "US" }
>
```

接下来，你将按以下操作顺序将这两个文档（`user1` 和 `user2`）添加到 `users` 集合中：

```
> db.users.insert(user1)
> db.users.insert(user2)
>
```

上述操作不仅会将两个文档插入 `users` 集合，还会创建该集合以及所属的数据库。可以使用 `show collections` 和 `show dbs` 命令进行验证。

如前所述，`show dbs` 将显示数据库列表。

```
> show dbs
admin   0.078GB
local   0.078GB
mydb    0.078GB
mydbproc 0.078GB
```

而 `show collections` 将显示当前数据库中的集合列表。

```
> show collections
system.indexes
users
>
```

除了 `users` 集合，还会显示 `system.indexes` 集合。这个 `system.indexes` 集合在数据库创建时默认生成，用于管理数据库内所有集合的索引信息。

执行命令 `db.users.find()` 将显示 `users` 集合中的文档。

```
> db.users.find()
{ "_id" : ObjectId("5450c048199484c9a4d26b0a"), "FName" : "Test", "LName" : "User", "Age" : 30, "Gender": "M", "Country" : "US" }
{ "_id" : ObjectId("5450c05d199484c9a4d26b0b"), "Name" : "Test", User", "Age" : 45,"Gender" : "F", "Country" : "US" }
>
```

可以看到，你创建的两个文档被显示出来。除了你添加的字段外，还有一个额外的 `_id` 字段，该字段为所有文档自动生成。

所有文档必须有一个唯一的 `_id` 字段。如果未由你显式指定，MongoDB 将自动为其分配一个唯一的对象 ID，如上例所示。

你没有显式插入 `_id` 字段，但当使用 `find()` 命令显示文档时，可以看到每个文档都关联了一个 `_id` 字段。

原因在于，默认情况下会在 `_id` 字段上创建一个索引，这可以通过在 `system.indexes` 集合上执行 `find` 命令来验证。

```
>db.system.indexes.find()
{ "v" : 1, "key" : { "_id" : 1 }, "ns" : "mydbpoc.users", "name" : "_id_" }
>
```

可以使用 `ensureIndex()` 和 `dropIndex()` 命令向集合添加或删除新索引。我们将在本章后面介绍这些内容。默认情况下，所有集合的 `_id` 字段上都会创建一个索引，这个默认索引无法被删除。

### 6.1.2 显式创建集合

在上面的例子中，第一次插入操作隐式地创建了集合。但是，用户也可以在执行插入语句之前显式地创建集合。

```
db.createCollection("users")
```

### 6.1.3 使用循环插入文档

也可以使用 `for` 循环向集合添加文档。以下代码使用 `for` 循环插入用户。

```
> for(var i=1; i<=20; i++) db.users.insert({"Name" : "Test User" + i, "Age": 10+i, "Gender" : "F", "Country" : "India"})
>
```

为了验证插入是否成功，可以在集合上运行 `find` 命令。

```
> db.users.find()
{ "_id" : ObjectId("52f48cf474f8fdcfcae84f79"), "FName" : "Test", "LName" : "User", "Age" : 30, "Gender" : "M", "Country" : "US" }
{ "_id" : ObjectId("52f48cfb74f8fdcfcae84f7a"), "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "US" }
................
{ "_id" : ObjectId("52f48eeb74f8fdcfcae84f8c"), "Name" : "Test User18", "Age" : 28, "Gender" : "F", "Country" : "India" }
Type "it" for more
>
```

用户出现在集合中。在继续之前，让我们理解一下 “Type “it” for more” 这条语句。

`find` 命令返回一个指向结果集的游标。它不会一次性在屏幕上显示所有文档（结果可能成千上万或数百万），而是先显示前 20 个文档，然后等待迭代请求（输入 `it`）以显示接下来的 20 个，以此类推，直到整个结果集显示完毕。

返回的游标也可以赋值给一个变量，然后使用 `while` 循环以编程方式遍历它。游标对象也可以作为数组进行操作。

在你的例子中，如果输入 `it` 并按回车键，将显示以下内容：

```
> it
{ "_id" : ObjectId("52f48eeb74f8fdcfcae84f8d"), "Name" : "Test User19", "Age" : 29, "Gender" : "F", "Country" : "India" }
{ "_id" : ObjectId("52f48eeb74f8fdcfcae84f8e"), "Name" : "Test User20", "Age" : 30, "Gender" : "F", "Country" : "India" }
>
```

由于只剩两个文档，所以它显示了剩余的两个文档。

### 6.1.4 通过显式指定 `_id` 插入

在之前的插入示例中，`_id` 字段未被指定，因此是隐式添加的。在下面的例子中，你将看到在向集合插入文档时如何显式指定 `_id` 字段。

显式指定 `_id` 字段时，必须牢记该字段的唯一性；否则插入操作将失败。

以下命令显式指定了 `_id` 字段：

```
> db.users.insert({"_id":10, "Name": "explicit id"})
```

插入操作在 `users` 集合中创建了如下文档：

```json
{ "_id" : 10, "Name" : "explicit id" }
```

这可以通过执行以下命令来确认：

```
>db.users.find()
```



### 6.1.5 更新

在本节中，你将学习 `update()` 命令，该命令用于更新集合中的文档。

`update()` 方法默认更新单个文档。如果需要更新所有符合选择条件的文档，可以通过将 `multi` 选项设置为 `true` 来实现。

我们从更新现有列的值开始。`$set` 操作符将用于更新记录。

以下命令将所有女性用户的国家更新为英国：
```
> db.users.update({"Gender":"F"}, {$set:{"Country":"UK"}})
```

要检查更新是否已发生，可以使用 `find` 命令检查所有女性用户：
```
> db.users.find({"Gender":"F"})
{ "_id" : ObjectId("52f48cfb74f8fdcfcae84f7a"), "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "UK" }
{ "_id" : ObjectId("52f48eeb74f8fdcfcae84f7b"), "Name" : "Test User1", "Age" : 11, "Gender" : "F", "Country" : "India" }
{ "_id" : ObjectId("52f48eeb74f8fdcfcae84f7c"), "Name" : "Test User2", "Age" : 12, "Gender" : "F", "Country" : "India" }
...................
Type "it" for more
>
```

检查输出，你会发现只有第一条文档记录被更新了，这是 `update` 命令的默认行为，因为没有指定 `multi` 选项。

现在，让我们修改 `update` 命令并包含 `multi` 选项：
```
>db.users.update({"Gender":"F"},{$set:{"Country":"UK"}},{multi:true})
>
```

再次执行 `find` 命令，检查所有女性员工的国家是否都已更新。执行 `find` 命令将返回以下输出：
```
> db.users.find({"Gender":"F"})
{ "_id" : ObjectId("52f48cfb74f8fdcfcae84f7a"), "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "UK" }
..............
Type "it" for more
>
```

如你所见，所有符合条件的记录中国家字段都已更新为英国。

在实际应用中，你可能会遇到模式演变的情况，需要向文档中添加或移除字段。下面看看如何在 MongoDB 数据库中执行这些更改。

`update()` 操作可以在文档级别使用，这有助于更新集合中的单个文档或一组文档。

接下来，我们看看如何向文档添加新字段。要向文档添加字段，可以结合使用带 `$set` 操作符和 `multi` 选项的 `update()` 命令。

如果你使用的字段名在 `$set` 中不存在，那么该字段将被添加到文档中。以下命令将向所有文档添加 `company` 字段：
```
> db.users.update({},{$set:{"Company":"TestComp"}},{multi:true})
>
```

对用户集合执行 `find` 命令，你会发现新字段已添加到所有文档中。
```
> db.users.find()
{ "Age" : 30, "Company" : "TestComp", "Country" : "US", "FName" : "Test", "Gender" : "M", "LName" : "User", "_id" : ObjectId("52f48cf474f8fdcfcae84f79") }
{ "Age" : 45, "Company" : "TestComp", "Country" : "UK", "Gender" : "F", "Name" : "Test User", "_id" : ObjectId("52f48cfb74f8fdcfcae84f7a") }
{ "Age" : 11, "Company" : "TestComp", "Country" : "UK", "Gender" : "F", ....................
Type "it" for more
>
```

如果 `update()` 命令中指定的字段在文档中已存在，它将更新该字段的值；如果该字段不存在，则会将该字段添加到文档中。

接下来，你将看到如何使用相同的 `update()` 命令配合 `$unset` 操作符从文档中移除字段。

以下命令将从所有文档中移除 `Company` 字段：
```
> db.users.update({},{$unset:{"Company":""}},{multi:true})
>
```

可以通过对 `Users` 集合执行 `find()` 命令来验证。你可以看到 `Company` 字段已从文档中删除。
```
> db.users.find()
{ "Age" : 30, "Country" : "US", "FName" : "Test", "Gender" : "M", "LName" : "User", "_id" : ObjectId("52f48cf474f8fdcfcae84f79") }
.............
Type "it" for more
```

### 6.1.6 删除

要删除集合中的文档，请使用 `remove()` 方法。如果指定了选择条件，则仅删除符合条件的文档。如果未指定条件，则所有文档都将被删除。

以下命令将删除 `Gender = 'M'` 的文档：
```
> db.users.remove({"Gender":"M"})
>
```

可以通过在 `Users` 上执行 `find()` 命令进行验证：
```
> db.users.find({"Gender":"M"})
>
```
没有文档返回。

以下命令将删除所有文档：
```
> db.users.remove({})
> db.users.find()
>
```
如你所见，没有文档返回。

最后，如果你想删除整个集合，可以使用以下命令：
```
> db.users.drop()
true
>
```

为了验证集合是否已被删除，可以执行 `show collections` 命令。
```
> show collections
system.indexes
>
```
如你所见，集合名称没有显示，这确认了该集合已从数据库中移除。

在介绍了基本的创建、更新和删除操作之后，下一节将展示如何执行读取操作。

### 6.1.7 读取

在本章的这一部分，你将通过各种示例了解 MongoDB 提供的查询功能，这些功能使你能够从数据库中读取存储的数据。

为了从基本查询开始，首先使用 `insert` 命令创建 `users` 集合并插入数据。
```
> user1 = {FName: "Test", LName: "User", Age:30, Gender: "M", Country: "US"}
{
    "FName" : "Test",
    "LName" : "User",
    "Age" : 30,
    "Gender" : "M",
    "Country" : "US"
}
> user2 = {Name: "Test User", Age:45, Gender: "F", Country: "US"}
{ "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "US" }
> db.users.insert(user1)
> db.users.insert(user2)
> for(var i=1; i<=20; i++) db.users.insert({"Name" : "Test User" + i, "Age": 10+i, "Gender" : "F", "Country" : "India"})
```

现在开始基本查询。`find()` 命令用于从数据库中检索数据。

执行 `find()` 命令会返回集合中的所有文档。
```
> db.users.find()
{ "_id" : ObjectId("52f4a823958073ea07e15070"), "FName" : "Test", "LName" : "User", "Age" : 30, "Gender" : "M", "Country" : "US" }
{ "_id" : ObjectId("52f4a826958073ea07e15071"), "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "US" }
......
{ "_id" : ObjectId("52f4a83f958073ea07e15083"), "Name" : "Test User18", "Age" :28, "Gender" : "F", "Country" : "India" }
Type "it" for more
>
```

#### 6.1.7.1 查询文档

MongoDB 提供了丰富的查询系统。可以将查询文档作为参数传递给 `find()` 方法，以过滤集合中的文档。

查询文档在花括号 `{` 和 `}` 内指定。在返回结果集之前，查询文档会与集合中的所有文档进行匹配。

使用不带查询文档的 `find()` 命令或使用空查询文档（如 `find({}`）将返回集合中的所有文档。

查询文档可以包含选择器和投影器。

选择器类似于 SQL 中的 `where` 条件或过滤器，用于过滤结果。

投影器类似于 `select` 条件或选择列表，用于显示数据字段。


## 6.1.7.2 选择器

你现在将学习如何使用选择器。以下命令将返回所有女性用户：

```
> db.users.find({"Gender":"F"})

{ "_id" : ObjectId("52f4a826958073ea07e15071"), "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "US" }
.............
{ "_id" : ObjectId("52f4a83f958073ea07e15084"), "Name" : "Test User19", "Age" :29, "Gender" : "F", "Country" : "India" }
Type "it" for more
>
```

让我们更进一步。MongoDB 还支持操作符，可以将不同的条件合并在一起，以便根据你的需求优化搜索。

让我们优化上面的查询，现在查找来自印度的女性用户。以下命令将返回相同的结果：

```
> db.users.find({"Gender":"F", $or: [{"Country":"India"}]})

{ "_id" : ObjectId("52f4a83f958073ea07e15072"), "Name" : "Test User1", "Age" : 11, "Gender" : "F", "Country" : "India" }
...........
{ "_id" : ObjectId("52f4a83f958073ea07e15085"), "Name" : "Test User20", "Age" :30, "Gender" : "F", "Country" : "India" }
>
```

接下来，如果你想找到所有属于印度或美国的女性用户，请执行以下命令：

```
>db.users.find({"Gender":"F",$or:[{"Country":"India"},{"Country":"US"}]})

{ "_id" : ObjectId("52f4a826958073ea07e15071"), "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "US" }
..............
{ "_id" : ObjectId("52f4a83f958073ea07e15084"), "Name" : "Test User19", "Age" :29, "Gender" : "F", "Country" : "India" }
Type "it" for more
```

对于聚合需求，需要使用聚合函数。接下来，你将学习如何使用`count()`函数进行聚合。

在上面的例子中，你不想显示文档，而是想找出居住在印度或美国的女性用户的数量。因此请执行以下命令：

```
>db.users.find({"Gender":"F",$or:[{"Country":"India"}, {"Country":"US"}]}).count()

21
>
```

如果你想查找不考虑任何选择器的用户数量，请执行以下命令：

```
> db.users.find().count()

22
>
```

## 6.1.7.3 投影器

你已经学习了如何使用选择器来过滤集合中的文档。在上面的例子中，`find()`命令返回匹配选择器的文档的所有字段。

让我们在查询文档中添加一个投影器，在选择器之外，你还将指定需要显示的特定详情或字段。

假设你想显示所有女性员工的名字和年龄。在这种情况下，除了选择器，还需要使用投影器。

执行以下命令以返回所需的结果集：

```
> db.users.find({"Gender":"F"}, {"Name":1,"Age":1})

{ "_id" : ObjectId("52f4a826958073ea07e15071"), "Name" : "Test User", "Age" : 45 }
..........
Type "it" for more
>
```

## 6.1.7.4 sort()

在 MongoDB 中，排序顺序指定如下：1 表示升序，-1 表示降序。

如果在上面的例子中，你想按年龄升序对记录进行排序，请执行以下命令：

```
>db.users.find({"Gender":"F"}, {"Name":1,"Age":1}).sort({"Age":1})

{ "_id" : ObjectId("52f4a83f958073ea07e15072"), "Name" : "Test User1", "Age" : 11 }
{ "_id" : ObjectId("52f4a83f958073ea07e15073"), "Name" : "Test User2", "Age" : 12 }
{ "_id" : ObjectId("52f4a83f958073ea07e15074"), "Name" : "Test User3", "Age" : 13 }
..............
{ "_id" : ObjectId("52f4a83f958073ea07e15085"), "Name" : "Test User20", "Age" :30 }
Type "it" for more
```

如果你想按姓名降序、年龄升序显示记录，请执行以下命令：

```
>db.users.find({"Gender":"F"},{"Name":1,"Age":1}).sort({"Name":-1,"Age":1})

{ "_id" : ObjectId("52f4a83f958073ea07e1507a"), "Name" : "Test User9", "Age" : 19 }
............
{ "_id" : ObjectId("52f4a83f958073ea07e15072"), "Name" : "Test User1", "Age" : 11 }
Type "it" for more
```

## 6.1.7.5 limit()

你现在将了解如何限制结果集中的记录数。例如，在包含数千个文档的大型集合中，如果你想只返回五个匹配的文档，就会使用`limit`命令，它正好可以让你做到这一点。

回到之前关于居住在印度或美国的女性用户的查询，假设你想限制结果集，只返回两个用户。需要执行以下命令：

```
>db.users.find({"Gender":"F",$or:[{"Country":"India"},{"Country":"US"}]}).limit(2)

{ "_id" : ObjectId("52f4a826958073ea07e15071"), "Name" : "Test User", "Age" : 45, "Gender" : "F", "Country" : "US" }
{ "_id" : ObjectId("52f4a83f958073ea07e15072"), "Name" : "Test User1", "Age" : 11, "Gender" : "F", "Country" : "India" }
```

## 6.1.7.6 skip()

如果需求是跳过前两条记录并返回第三和第四个用户，则需要使用`skip`命令。需要执行以下命令：

```
>db.users.find({"Gender":"F",$or:[{"Country":"India"}, {"Country":"US"}]}).limit(2).skip(2)

{ "_id" : ObjectId("52f4a83f958073ea07e15073"), "Name" : "Test User2", "Age" : 12, "Gender" : "F", "Country" : "India" }
{ "_id" : ObjectId("52f4a83f958073ea07e15074"), "Name" : "Test User3", "Age" : 13, "Gender" : "F", "Country" : "India" }
>
```

## 6.1.7.7 findOne()

与`find()`类似的是`findOne()`命令。`findOne()`方法可以接受与`find()`相同的参数，但它不返回游标，而是返回单个文档。假设你想返回一个居住在印度或美国的女性用户。这可以通过以下命令实现：

```
> db.users.findOne({"Gender":"F"}, {"Name":1,"Age":1})

{
"_id" : ObjectId("52f4a826958073ea07e15071"),
"Name" : "Test User",
"Age" : 45
}
>
```

类似地，如果你想不考虑任何选择器返回第一条记录，在这种情况下，你可以使用`findOne()`，它将返回集合中的第一个文档。

```
> db.users.findOne()

{
"_id" : ObjectId("52f4a823958073ea07e15070"),
"FName" : "Test",
"LName" : "User",
"Age" : 30,
"Gender" : "M",
"Country" : "US"}
```

## 6.1.7.8 使用游标

当使用`find()`方法时，MongoDB 将查询结果作为游标对象返回。为了显示结果，mongo shell 会遍历返回的游标。

MongoDB 使用户能够使用`find`方法的游标对象。在下一个例子中，你将看到如何将游标对象存储到变量中，并使用 while 循环对其进行操作。

假设你想返回所有在美国的用户。为此，你创建了一个变量，将`find()`的输出赋值给该变量（这是一个游标），然后使用 while 循环进行迭代并打印输出。

代码片段如下：

```
> var c = db.users.find({"Country":"US"})
> while(c.hasNext()) printjson(c.next())

{
"_id" : ObjectId("52f4a823958073ea07e15070"),
"FName" : "Test",
"LName" : "User",
"Age" : 30,
"Gender" : "M",
"Country" : "US"
}
{
"_id" : ObjectId("52f4a826958073ea07e15071"),
"Name" : "Test User",
"Age" : 45,
"Gender" : "F",
"Country" : "US"
}
>
```

`next()`函数返回下一个文档。`hasNext()`函数在文档存在时返回 true，而`printjson()`以 JSON 格式呈现输出。

游标对象被赋值给的变量也可以作为数组进行操作。如果你想显示数组索引 1 处的文档，而不是遍历该变量，你可以运行以下命令：

```
> var c = db.users.find({"Country":"US"})
> printjson(c[1])

{
"_id" : ObjectId("52f4a826958073ea07e15071"),
"Name" : "Test User",
.... "Gender" : "F",
"Country" : "US"}
>
```


### 6.1.7.9 `explain()`

`explain()` 函数可用于查看 MongoDB 数据库在执行查询时运行的步骤。从 3.0 版本开始，该函数的输出格式和传递给它的参数发生了变化。它接受一个名为 `verbose` 的可选参数，该参数决定解释输出的显示方式。以下是详细程度模式：`allPlansExecution`、`executionStats` 和 `queryPlanner`。默认的详细程度模式是 `queryPlanner`，这意味着如果不指定任何参数，则默认使用 `queryPlanner`。

以下代码展示了在 `username` 字段上进行过滤时执行的步骤：

```bash
> db.users.find({"Name":"Test User"}).explain("allPlansExecution")
```

```json
{
  "queryPlanner" : {
    "plannerVersion" : 1,
    "namespace" : "mydbproc.users",
    "indexFilterSet" : false,
    "parsedQuery" : {
      "$and" : [ ]
    },
    "winningPlan" : {
      "stage" : "COLLSCAN",
      "filter" : {
        "$and" : [ ]
      },
      "direction" : "forward"
    },
    "rejectedPlans" : [ ]
  },
  "executionStats" : {
    "executionSuccess" : true,
    "nReturned" : 20,
    "executionTimeMillis" : 0,
    "totalKeysExamined" : 0,
    "totalDocsExamined" : 20,
    "executionStages" : {
      "stage" : "COLLSCAN",
      "filter" : {
        "$and" : [ ]
      },
      "nReturned" : 20,
      "executionTimeMillisEstimate" : 0,
      "works" : 22,
      "advanced" : 20,
      "needTime" : 1,
      "needFetch" : 0,
      "saveState" : 0,
      "restoreState" : 0,
      "isEOF" : 1,
      "invalidates" : 0,
      "direction" : "forward",
      "docsExamined" : 20
    },
    "allPlansExecution" : [ ]
  },
  "serverInfo" : {
    "host" : " ANOC9",
    "port" : 27017,
    "version" : "3.0.4",
    "gitVersion" : "534b5a3f9d10f00cd27737fbcd951032248b5952"
  },
  "ok" : 1
}
```

如您所见，`explain()` 输出返回了关于 `queryPlanner`、`executionStats` 和 `serverInfo` 的信息。如上所示，输出返回的信息取决于所选的详细程度模式。

您已经了解了如何执行基本的查询、排序、限制等。您还看到了如何使用 `while` 循环或作为数组来操作结果集。在下一节中，您将了解索引以及如何在查询中使用它们。

### 6.1.8 使用索引

索引用于为频繁使用的查询提供高性能的读取操作。默认情况下，每当创建一个集合并向其中添加文档时，就会在文档的 `_id` 字段上创建一个索引。

在本节中，您将了解如何创建不同类型的索引。让我们从使用 `for` 循环在一个名为 `testindx` 的新集合中插入 100 万份文档开始。

```bash
>for(i=0;i<1000000;i++){db.testindx.insert({"Name":"user"+i,"Age":Math.floor(Math.random()*120)})}
```

接下来，发出 `find()` 命令以获取 `Name` 值为 `user101` 的文档。运行 `explain()` 命令以检查 MongoDB 为返回结果集正在执行哪些步骤。

```bash
> db.testindx.find({"Name":"user101"}).explain("allPlansExecution")
```

```json
{
  "queryPlanner" : {
    "plannerVersion" : 1,
    "namespace" : "mydbproc.testindx",
    "indexFilterSet" : false,
    "parsedQuery" : {
      "Name" : {
        "$eq" : "user101"
      }
    },
    "winningPlan" : {
      "stage" : "COLLSCAN",
      "filter" : {
        "Name" : {
          "$eq" : "user101"
        }
      },
      "direction" : "forward"
    },
    "rejectedPlans" : [ ]
  },
  "executionStats" : {
    "executionSuccess" : true,
    "nReturned" : 1,
    "executionTimeMillis" : 645,
    "totalKeysExamined" : 0,
    "totalDocsExamined" : 1000000,
    "executionStages" : {
      "stage" : "COLLSCAN",
      "filter" : {
        "Name" : {
          "$eq" : "user101"
        }
      },
      "nReturned" : 1,
      "executionTimeMillisEstimate" : 20,
      "works" : 1000002,
      "advanced" : 1,
      "needTime" : 1000000,
      "needFetch" : 0,
      "saveState" : 7812,
      "restoreState" : 7812,
      "isEOF" : 1,
      "invalidates" : 0,
      "direction" : "forward",
      "docsExamined" : 1000000
    },
    "allPlansExecution" : [ ]
  },
  "serverInfo" : {
    "host" : " ANOC9",
    "port" : 27017,
    "version" : "3.0.4",
    "gitVersion" : "534b5a3f9d10f00cd27737fbcd951032248b5952"
  },
  "ok" : 1
}
```

如您所见，数据库扫描了整个表。这对性能有显著影响，发生这种情况是因为没有索引。

#### 6.1.8.1 单键索引

让我们在文档的 `Name` 字段上创建一个索引。使用 `ensureIndex()` 创建索引。

```bash
> db.testindx.ensureIndex({"Name":1})
```

索引创建将花费几分钟时间，具体取决于服务器和集合大小。

让我们运行与之前相同的查询，并使用 `explain()` 来检查索引创建后数据库正在执行哪些步骤。检查输出中的 `n`、`nscanned` 和 `millis` 字段。

```bash
>db.testindx.find({"Name":"user101"}).explain("allPathsExecution")
```

```json
{
  "queryPlanner" : {
    "plannerVersion" : 1,
    "namespace" : "mydbproc.testindx",
    "indexFilterSet" : false,
    "parsedQuery" : {
      "Name" : {
        "$eq" : "user101"
      }
    },
    "winningPlan" : {
      "stage" : "FETCH",
      "inputStage" : {
        "stage" : "IXSCAN",
        "keyPattern" : {
          "Name" : 1
        },
        "indexName" : "Name_1",
        "isMultiKey" : false,
        "direction" : "forward",
        "indexBounds" : {
          "Name" : [
            "[\"user101\", \"user101\"]"
          ]
        }
      }
    },
    "rejectedPlans" : [ ]
  },
  "executionStats" : {
    "executionSuccess" : true,
    "nReturned" : 1,
    "executionTimeMillis" : 0,
    "totalKeysExamined" : 1,
    "totalDocsExamined" : 1,
    "executionStages" : {
      "stage" : "FETCH",
      "nReturned" : 1,
      "executionTimeMillisEstimate" : 0,
      "works" : 2,
      "advanced" : 1,
      "needTime" : 0,
      "needFetch" : 0,
      "saveState" : 0,
      "restoreState" : 0,
      "isEOF" : 1,
      "invalidates" : 0,
      "docsExamined" : 1,
      "alreadyHasObj" : 0,
      "inputStage" : {
        "stage" : "IXSCAN",
        "nReturned" : 1,
        "executionTimeMillisEstimate" : 0,
        "works" : 2,
        "advanced" : 1,
        "needTime" : 0,
        "needFetch" : 0,
        "saveState" : 0,
        "restoreState" : 0,
        "isEOF" : 1,
        "invalidates" : 0,
        "keyPattern" : {
          "Name" : 1
        },
        "indexName" : "Name_1",
        "isMultiKey" : false,
        "direction" : "forward",
        "indexBounds" : {
          "Name" : [
            "[\"user101\", \"user101\"]"
          ]
        },
        "keysExamined" : 1,
        "dupsTested" : 0,
        "dupsDropped" : 0,
        "seenInvalidated" : 0,
        "matchTested" : 0
      }
    },
    "allPlansExecution" : [ ]
  },
  "serverInfo" : {
    "host" : "ANOC9",
    "port" : 27017,
    "version" : "3.0.4",
    "gitVersion" : "534b5a3f9d10f00cd27737fbcd951032248b5952"
  },
  "ok" : 1
}
```

如您在结果中所见，没有表扫描。索引创建对查询执行时间产生了显著差异。



## 6.1.8.2 复合索引

创建索引时，你需要记住索引应覆盖你的大多数查询。如果你有时只查询`Name`字段，而有时又同时查询`Name`和`Age`字段，那么创建一个在`Name`和`Age`字段上的复合索引，会比仅在其中一个字段上创建索引更有益，因为这个复合索引将覆盖这两种查询。

以下命令在集合`testindx`的字段`Name`和`Age`上创建一个复合索引。

```
> db.testindx.ensureIndex({"Name":1, "Age": 1})
```

复合索引帮助 MongoDB 更高效地执行包含多个子句的查询。创建复合索引时，同样重要的是要记住，用于精确匹配的字段（例如 `Name` : `"S1"` ）应放在前面，随后是用于范围查询的字段（例如 `Age` : `{"$gt":20}`）。

因此，上述索引将对以下查询有益：

```
>db.testindx.find({"Name": "user5","Age":{"$gt":25}}).explain("allPlansExecution")
```

```
{
    "queryPlanner" : {
        "plannerVersion" : 1,
        "namespace" : "mydbproc.testindx",
        "indexFilterSet" : false,
        "parsedQuery" : {
            "$and" : [
                {
                    "Name" : {
                        "$eq" : "user5"
                    }
                },
                {
                    "Age" : {
                        "$gt" : 25
                    }
                }
            ]
        },
        "winningPlan" : {
            "stage" : "KEEP_MUTATIONS",
            "inputStage" : {
                "stage" : "FETCH",
                "filter" : {
                    "Age" : {
                        "$gt" : 25
                    }
                },
                ............................
                "indexBounds" : {
                    "Name" : [
                        "[\"user5\", \"user5\""
                    }
                },
                "rejectedPlans" : [
                    {
                        "stage" : "FETCH",
                        ......................................................
                        "indexName" : "Name_1_Age_1",
                        "isMultiKey" : false,
                        "direction" : "forward",
                        .....................................................
                        "executionStats" : {
                            "executionSuccess" : true,
                            "nReturned" : 1,
                            "executionTimeMillis" : 0,
                            "totalKeysExamined" : 1,
                            "totalDocsExamined" : 1,
                            .....................................................
                            "inputStage" : {
                                "stage" : "FETCH",
                                "filter" : {
                                    "Age" : {
                                        "$gt" : 25
                                    }
                                },
                                "nReturned" : 1,
                                "executionTimeMillisEstimate" : 0,
                                "works" : 2,
                                "advanced" : 1,
                                "allPlansExecution" : [
                                    {
                                        "nReturned" : 1,
                                        "executionTimeMillisEstimate" : 0,
                                        "totalKeysExamined" : 1,
                                        "totalDocsExamined" : 1,
                                        "executionStages" : {
                                            .............................................................
                                            "serverInfo" : {
                                                "host" : " ANOC9",
                                                "port" : 27017,
                                                "version" : "3.0.4",
                                                "gitVersion" : "534b5a3f9d10f00cd27737fbcd951032248b5952"
                                            },
                                            "ok" : 1
                                        }
                                    }
```

## 6.1.8.3 支持排序操作

在 MongoDB 中，使用索引字段对文档进行排序的操作能提供最佳性能。

与其他数据库类似，MongoDB 中的索引也具有顺序。如果使用索引来访问文档，它将按照与索引相同的顺序返回结果。

当需要在多个字段上排序时，需要创建复合索引。在复合索引中，输出可以按照索引前缀或整个索引的排序顺序排列。

索引前缀是复合索引的一个子集，包含从索引开头的一个或多个字段。

例如，复合索引 `{ x:1, y: 1, z: 1}` 的索引前缀如下所示。

排序操作可以是任何索引前缀的组合，例如 `{x: 1}` 或 `{x: 1, y: 1}`。

复合索引只有当它是排序的前缀时，才能帮助排序。

例如，在 `Age` 、 `Name` 和 `Class` 上创建的复合索引，如下所示：

```
> db.testindx.ensureIndex({"Age": 1, "Name": 1, "Class": 1})
```

将对以下查询有用：

```
> db.testindx.find().sort({"Age":1})
> db.testindx.find().sort({"Age":1,"Name":1})
> db.testindx.find().sort({"Age":1,"Name":1, "Class":1})
```

上述索引对以下查询帮助不大：

```
> db.testindx.find().sort({"Gender":1, "Age":1, "Name": 1})
```

你可以使用`explain()`命令来诊断 MongoDB 如何处理查询。

## 6.1.8.4 唯一索引

在某个字段上创建索引并不能保证唯一性，所以如果在`Name`字段上创建索引，那么两个或多个文档可以具有相同的名称。然而，如果需要启用唯一性约束，则在创建索引时需要将`unique`属性设置为`true`。

首先，让我们删除现有的索引。

```
>db.testindx.dropIndexes()
```

以下命令将在`testindx`集合的`Name`字段上创建一个唯一索引：

```
> db.testindx.ensureIndex({"Name":1},{"unique":true})
```

现在，如果你尝试向集合中插入重复的名称，如下所示，MongoDB 会返回错误并不允许插入重复记录：

```
> db.testindx.insert({"Name":"uniquename"})
> db.testindx.insert({"Name":"uniquename"})
```

```
"E11000 duplicate key error index: mydbpoc.testindx.$Name_1 dup key: { : "uniquename" }"
```

如果你检查集合，你会看到只有第一个`uniquename`被存储了。

```
> db.testindx.find({"Name":"uniquename"})
{ "_id" : ObjectId("52f4b3c3958073ea07f092ca"), "Name" : "uniquename" }
>
```

唯一性也可以为复合索引启用，这意味着虽然单个字段可以有重复值，但字段的组合将始终是唯一的。

例如，如果你在`{"name":1, "age":1}`上有一个唯一索引，

```
> db.testindx.ensureIndex({"Name":1, "Age":1},{"unique":true})
>
```

那么以下插入操作是被允许的：

```
> db.testindx.insert({"Name":"usercit"})
> db.testindx.insert({"Name":"usercit", "Age":30})
```

然而，如果你执行

```
> db.testindx.insert({"Name":"usercit", "Age":30})
```

它将抛出一个类似下面的错误

```
E11000 duplicate key error index: mydbpoc.testindx.$Name_1_Age_1 dup key: { : "usercit", : 30.0 }
```

你可能先创建集合并插入文档，然后在集合上创建索引。如果你在可能包含重复值的字段上为集合创建唯一索引，索引创建将会失败。

为了应对这种情况，MongoDB 提供了`dropDups`选项。`dropDups`选项会保存找到的第一个文档，并删除任何后续具有重复值的文档。

以下命令将在`name`字段上创建一个唯一索引，并删除任何重复的文档：

```
>db.testindx.ensureIndex({"Name":1},{"unique":true, "dropDups":true})
>
```

## 6.1.8.5 system.indexes

每当你创建一个数据库时，默认会创建一个`system.indexes`集合。有关数据库索引的所有信息都存储在`system.indexes`集合中。这是一个保留集合，因此你不能修改或删除其中的文档。你只能通过`ensureIndex`和`dropIndexes`数据库命令来操作它。

每当创建一个索引时，其元信息可以在`system.indexes`中看到。可以使用以下命令获取指定集合的所有索引信息：

```
db.collectionName.getIndexes()
```

例如，以下命令将返回在`testindx`集合上创建的所有索引：

```
> db.testindx.getIndexes()
```

## 6.1.8.6 dropIndex

`dropIndex`命令用于删除索引。

以下命令将从`testindx`集合中删除`Name`字段的索引：

```
> db.testindx.dropIndex({"Name":1})
{ "nIndexesWas" : 3, "ok" : 1 }
>
```



## 6.1.8.7 `reIndex`

当你在集合上执行了多次插入和删除操作后，可能需要重建索引，以便索引能够以最佳状态使用。`reIndex` 命令用于重建索引。

以下命令重建集合的所有索引。它会首先删除索引（包括 `_id` 字段上的默认索引），然后重建这些索引。

```
db.collectionname.reIndex()
```

以下命令重建 `testindx` 集合的索引：

```
> db.testindx.reIndex()
{
    "nIndexesWas" : 2,
    "msg" : "indexes dropped for collection",
    "nIndexes" : 2,
    ..............
    "ok" : 1
}
>
```

我们将在下一章详细讨论 MongoDB 中可用的不同类型的索引。

## 6.1.8.8 索引如何工作

MongoDB 将索引存储在 BTree 结构中，因此自动支持范围查询。

如果在查询中使用了多个选择条件，MongoDB 会尝试找到最佳的单一索引来选择候选集。之后，它会顺序遍历该集合以评估其他条件。

当首次执行查询时，MongoDB 会为该查询可用的每个索引创建多个执行计划。它让这些计划轮流在一定数量的滴答（ticks）内执行，直到执行最快的计划完成。结果随后返回给系统，系统会记住最快执行计划所使用的索引。

对于后续查询，将使用记住的索引，直到集合内发生的更新次数达到某个特定限制。超过更新限制后，系统将再次遵循该过程以找出当时适用的最佳索引。

当发生以下任一事件时，查询计划将重新评估：

*   集合接收到 1,000 次写操作。
*   添加或删除了一个索引。
*   `mongod` 进程重启。
*   为重建索引而重新索引。

如果你想覆盖 MongoDB 的默认索引选择，可以使用 `hint()` 方法来实现。

索引过滤器在 2.6 版本中引入。它由优化器将为查询评估的索引组成，包括查询、投影和排序。MongoDB 将使用索引过滤器提供的索引，并忽略 `hint()`。

在 2.6 版本之前，MongoDB 在任何时间点只使用一个索引，因此你需要确保存在复合索引以更好地匹配你的查询。这可以通过检查查询的排序和搜索条件来实现。

索引交集在 2.6 版本中引入。它支持通过满足复合条件的查询来交集索引，其中部分条件由一个索引满足，另一部分由另一个索引满足。

通常，索引交集由两个索引组成；然而，可以使用多个索引交集来解析查询。这种能力提供了更好的优化。

与其他数据库一样，索引维护是有成本的。每个更改集合的操作（如创建、更新或删除）都会产生开销，因为索引也需要更新。为保持最佳平衡，你需要定期检查拥有索引的有效性，这可以通过系统上的读写比率来衡量。识别使用较少的索引并删除它们。

## 6.2 超越基础

本节将涵盖使用条件操作符和正则表达式在选择器部分进行高级查询。这些操作符和正则表达式中的每一个都使你能够更好地控制所编写的查询，从而控制可以从 MongoDB 数据库中获取的信息。

### 6.2.1 使用条件操作符

条件操作符使你能够更好地控制试图从数据库中提取的数据。在本节中，你将重点关注以下操作符：`$lt`、`$lte`、`$gt`、`$gte`、`$in`、`$nin` 和 `$not`。

以下示例假设一个名为 `Students` 的集合包含以下类型的文档：

```
{
    _id: ObjectID(),
    Name: "Full Name",
    Age: 30,
    Gender: "M",
    Class: "C1",
    Score: 95
}
```

你将首先创建集合并插入一些示例文档。

```
>db.students.insert({Name:"S1",Age:25,Gender:"M",Class:"C1",Score:95})
>db.students.insert({Name:"S2",Age:18,Gender:"M",Class:"C1",Score:85})
>db.students.insert({Name:"S3",Age:18,Gender:"F",Class:"C1",Score:85})
>db.students.insert({Name:"S4",Age:18,Gender:"F",Class:"C1",Score:75})
>db.students.insert({Name:"S5",Age:18,Gender:"F",Class:"C2",Score:75})
>db.students.insert({Name:"S6",Age:21,Gender:"M",Class:"C2",Score:100})
>db.students.insert({Name:"S7",Age:21,Gender:"M",Class:"C2",Score:100})
>db.students.insert({Name:"S8",Age:25,Gender:"F",Class:"C2",Score:100})
>db.students.insert({Name:"S9",Age:25,Gender:"F",Class:"C2",Score:90})
>db.students.insert({Name:"S10",Age:28,Gender:"F",Class:"C3",Score:90})
> db.students.find()
{ "_id" : ObjectId("52f874faa13cd6a65998734d"), "Name" : "S1", "Age" : 25, "Gender" : "M", "Class" : "C1", "Score" : 95 }
.......................
{ "_id" : ObjectId("52f8758da13cd6a659987356"), "Name" : "S10", "Age" : 28, "Gender" : "F", "Class" : "C3", "Score" : 90 }
>
```

#### 6.2.1.1 `$lt` 和 `$lte`

让我们从 `$lt` 和 `$lte` 操作符开始。它们分别代表“小于”和“小于等于”。

如果你想找到所有年龄小于 25 岁（Age < 25）的学生，你可以执行以下带有选择器的 `find`：

```
> db.students.find({"Age":{"$lt":25}})
{ "_id" : ObjectId("52f8750ca13cd6a65998734e"), "Name" : "S2", "Age" : 18, "Gender" : "M", "Class" : "C1", "Score" : 85 }
.............................
{ "_id" : ObjectId("52f87556a13cd6a659987353"), "Name" : "S7", "Age" : 21, "Gender" : "M", "Class" : "C2", "Score" : 100 }
>
```

如果你想找出所有年龄小于等于 25 岁（Age <= 25）的学生，请执行：

```
> db.students.find({"Age":{"$lte":25}})
{ "_id" : ObjectId("52f874faa13cd6a65998734d"), "Name" : "S1", "Age" : 25, "Gender" : "M", "Class" : "C1", "Score" : 95 }
....................
{ "_id" : ObjectId("52f87578a13cd6a659987355"), "Name" : "S9", "Age" : 25, "Gender" : "F", "Class" : "C2", "Score" : 90 }
>
```

#### 6.2.1.2 `$gt` 和 `$gte`

`$gt` 和 `$gte` 操作符分别代表“大于”和“大于等于”。

让我们找出所有年龄大于 25 岁（Age > 25）的学生。这可以通过执行以下命令来实现：

```
> db.students.find({"Age":{"$gt":25}})
{ "_id" : ObjectId("52f8758da13cd6a659987356"), "Name" : "S10", "Age" : 28, "Gender" : "F", "Class" : "C3", "Score" : 90 }
>
```

如果你将上面的示例更改为返回年龄大于等于 25 岁（Age >= 25）的学生，那么命令是：

```
> db.students.find({"Age":{"$gte":25}})
{ "_id" : ObjectId("52f874faa13cd6a65998734d"), "Name" : "S1", "Age" : 25, "Gender" : "M", "Class" : "C1", "Score" : 95 }
......................................
{ "_id" : ObjectId("52f8758da13cd6a659987356"), "Name" : "S10", "Age" : 28, "Gender" : "F", "Class" : "C3", "Score" : 90 }
>
```



#### 6.2.1.3 `$in` 和 `$nin`

让我们找出所有属于 C1 或 C2 班的学生。相应的命令是：

```
> db.students.find({"Class":{"$in":["C1","C2"]}})
{ "_id" : ObjectId("52f874faa13cd6a65998734d"), "Name" : "S1", "Age" : 25, "Gender" : "M", "Class" : "C1", "Score" : 95 }
................................
{ "_id" : ObjectId("52f87578a13cd6a659987355"), "Name" : "S9", "Age" : 25, "Gender" : "F", "Class" : "C2", "Score" : 90 }
>
```

它的反向操作可以使用 `$nin` 来完成。

接下来，让我们找出不属于 C1 或 C2 班的学生。命令是：

```
> db.students.find({"Class":{"$nin":["C1","C2"]}})
{ "_id" : ObjectId("52f8758da13cd6a659987356"), "Name" : "S10", "Age" : 28, "Gender" : "F", "Class" : "C3", "Score" : 90 }
>
```

接下来，让我们看看如何组合以上所有操作符来编写查询。假设你想找出所有性别为“M”**或者**属于“C1”**或**“C2”班，并且年龄大于或等于 25 岁的学生。这可以通过执行以下命令实现：

```
> db.students.find({$or:[{"Gender":"M","Class":{"$in":["C1","C2"]}}], "Age":{"$gte":25}})
{ "_id" : ObjectId("52f874faa13cd6a65998734d"), "Name" : "S1", "Age" : 25, "Gender" : "M", "Class" : "C1", "Score" : 95 }
>
```

### 6.2.2 正则表达式

在本节中，你将学习如何使用正则表达式。当你想要查找名字以“A”开头的学生时，正则表达式非常有用。

为了理解这一点，让我们再添加三四个名字不同的学生。

```
> db.students.insert({Name:"Student1", Age:30, Gender:"M", Class: "Biology", Score:90})
> db.students.insert({Name:"Student2", Age:30, Gender:"M", Class: "Chemistry", Score:90})
> db.students.insert({Name:"Test1", Age:30, Gender:"M", Class: "Chemistry", Score:90})
> db.students.insert({Name:"Test2", Age:30, Gender:"M", Class: "Chemistry", Score:90})
> db.students.insert({Name:"Test3", Age:30, Gender:"M", Class: "Chemistry", Score:90})
>
```

假设你想找出所有名字以“St”或“Te”开头，并且班级以“Che”开头的学生。这可以使用正则表达式进行过滤，如下所示：

```
> db.students.find({"Name":/(St|Te)*/i, "Class":/(Che)/i})
{ "_id" : ObjectId("52f89ecae451bb7a56e59086"), "Name" : "Student2", "Age" : 30, "Gender" : "M", "Class" : "Chemistry", "Score" : 90 }
.........................
{ "_id" : ObjectId("52f89f06e451bb7a56e59089"), "Name" : "Test3", "Age" : 30, "Gender" : "M", "Class" : "Chemistry", "Score" : 90 }
>
```

为了理解正则表达式是如何工作的，让我们以查询 `"Name":/(St|Te)*/i` 为例。

*   `//i` 表示该正则表达式不区分大小写。
*   `(St|Te)*` 表示 `Name` 字符串必须以“St”或“Te”开头。
*   末尾的 `*` 表示它将匹配之后的任何内容。

当你将所有部分组合在一起时，你正在执行一个不区分大小写的匹配，查找名字以“St”或“Te”开头的记录。在 `Class` 的正则表达式中也使用了类似的规则。

接下来，让我们把查询复杂化一点。将其与上面介绍的操作符结合起来。

获取名字为 student1、student2，且为年龄 >=25 的男性学生。相应的命令如下：

```
> db.students.find({"Name":/(student*)/i,"Age":{"$gte":25},"Gender":"M"})
{ "_id" : ObjectId("52f89eb1e451bb7a56e59085"), "Name" : "Student1", "Age" : 30,
"Gender" : "M", "Class" : "Biology", "Score" : 90 }
{ "_id" : ObjectId("52f89ecae451bb7a56e59086"), "Name" : "Student2", "Age" : 30,
"Gender" : "M", "Class" : "Chemistry", "Score" : 90 }
```

### 6.2.3 MapReduce

MapReduce 框架能够划分任务，在本例中，即跨计算机集群进行数据聚合，以减少聚合数据集所需的时间。它包含两个部分：Map 和 Reduce。

以下是更具体的描述：MapReduce 是一个用于处理那些可以在海量数据集上高度分布式、并通过多个节点运行的问题的框架。如果所有节点都具有相同的硬件，这些节点统称为集群；否则，称为网格。这种处理可以发生在结构化数据（存储在数据库中的数据）和非结构化数据（存储在文件系统中的数据）上。

*   **“Map”**：在此步骤中，充当主节点的节点接收输入参数，并将大问题分解为多个小问题。然后，这些子问题被分配给工作节点。工作节点可能进一步将问题分解为子问题，从而形成一个多级树结构。工作节点随后处理它们内部的子问题，并将答案返回给主节点。
*   **“Reduce”**：在此步骤中，所有子问题的答案都已汇总到主节点，主节点随后组合所有答案并生成最终输出，即你要解决的大问题的答案。

为了理解其工作原理，让我们考虑一个小例子：统计集合中男性和女性学生的人数。

这涉及以下步骤：首先创建 `map` 和 `reduce` 函数，然后调用 `mapReduce` 函数并传递必要的参数。

让我们从定义 `map` 函数开始：

```
> var map = function(){emit(this.Gender,1);};
>
```

此步骤以文档为输入，并根据 `Gender` 字段，发出 `{"F", 1}` 或 `{"M", 1}` 类型的文档。

接下来，创建 `reduce` 函数：

```
> var reduce = function(key, value){return Array.sum(value);};
>
```

此函数将根据键字段（在你的例子中是 `Gender`）对 `map` 函数发出的文档进行分组，并返回值的总和（在上例中发出的值是“1”）。上面定义的 `reduce` 函数的输出是按性别统计的数量。

最后，你使用 `mapReduce` 函数将它们组合起来，如下所示：

```
> db.students.mapReduce(map, reduce, {out: "mapreducecount1"})
{
"result" : "mapreducecount1",
"timeMillis" : 29,
"counts" : {
"input" : 15,
"emit" : 15,
"reduce" : 2,
"output" : 2
},
"ok" : 1,
}
>
```

这实际上是在 `students` 集合上应用你定义的 `map` 和 `reduce` 函数。最终结果存储在一个名为 `mapreducecount1` 的新集合中。

为了验证它，在 `mapreducecount1` 集合上运行 `find()` 命令，如下所示：

```
> db.mapreducecount1.find()
{ "_id" : "F", "value" : 6 }
{ "_id" : "M", "value" : 9 }
>
```

这里还有一个例子来解释 `MapReduce` 的工作原理。让我们使用 `MapReduce` 来找出班级平均分。正如你在上面的例子中看到的，你需要先创建 `map` 函数，然后是 `reduce` 函数，最后将它们组合起来，将输出存储在你数据库的一个集合中。代码片段如下：

```
> var map_1 = function(){emit(this.Class,this.Score);};
> var reduce_1 = function(key, value){return Array.avg(value);};
> db.students.mapReduce(map_1,reduce_1, {out:"MR_ClassAvg_1"})
{
"result" : "MR_ClassAvg_1",
"timeMillis" : 4,
"counts" : {
"input" : 15, "emit" : 15,
"reduce" : 3 , "output" : 5
},
"ok" : 1,
}
> db.MR_ClassAvg_1.find()
{ "_id" : "Biology", "value" : 90 }
{ "_id" : "C1", "value" : 85 }
{ "_id" : "C2", "value" : 93 }
{ "_id" : "C3", "value" : 90 }
{ "_id" : "Chemistry", "value" : 90 }
>
```



