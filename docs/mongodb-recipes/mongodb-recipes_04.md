# 2. MongoDB CRUD 操作

在第 1 章中，我们讨论了 MongoDB 的特性以及如何在 Windows 和 Linux 上安装 MongoDB。我们还讨论了 MongoDB 中使用的术语以及如何创建数据库。在本章中，我们将讨论如何在 MongoDB 中执行创建、读取、更新和删除 (CRUD) 操作。本章还描述了如何处理嵌入式文档和数组。我们将讨论以下主题：

*   集合。
*   MongoDB CRUD 操作。
*   批量写操作。
*   MongoDB 导入和导出。
*   嵌入式文档。
*   处理数组。
*   嵌入式文档数组。
*   投影。
*   处理 null 和缺失值。
*   使用 `limit()` 和 `skip()`。
*   使用 `Node.js` 和 MongoDB。

## 集合

MongoDB 的集合类似于关系型数据库管理系统 (RDBMS) 中的表。在 MongoDB 中，当我们通过命令引用集合时，它们会自动创建。例如，考虑以下命令。

```javascript
db.person.insert({_id:100001,name:"Taanushree A S",age:10})
```

如果名为 `person` 的集合不存在，此命令将创建它。如果它已存在，则只是将一个文档插入到 `person` 集合中。

### 配方 2-1. 创建集合

在本配方中，我们将讨论如何创建集合。

### 问题

你想使用 `db.createCollection()` 命令创建一个名为 `person` 的集合。

### 解决方案

使用以下语法来创建集合。

```javascript
db.createCollection()
```

### 工作原理

让我们按照本节的步骤来创建一个名为 `person` 的集合。

##### 步骤 1：创建集合

要创建一个名为 `person` 的集合，请使用以下命令。

```javascript
db.createCollection("person")
```

输出如下，

```javascript
> db.createCollection("person")
{ "ok" : 1 }
>
```

要确认集合的存在，请在 `mongo` Shell 中键入以下命令。

```javascript
> show collections
person
>
```

我们现在可以看到名为 `person` 的集合了。



### 配方 2-2. 创建固定集合

在本配方中，我们将讨论什么是固定集合以及如何创建一个。固定集合是一种固定大小的集合，我们可以指定集合的最大字节数和允许的文档数量。固定集合的工作方式类似于循环缓冲区：一旦填满其分配的空间，它就会通过覆盖集合中最旧的文档来为新文档腾出空间。

固定集合有一些限制：

-   你无法对固定集合进行**分片**。
-   你无法使用聚合管道操作符 `$out` 将结果写入固定集合。
-   你无法从固定集合中删除文档。
-   创建索引可以让你执行高效的更新操作。

### 问题

你想使用 `db.createCollection()` 方法创建一个名为 `student` 的固定集合。

### 解决方案

使用以下语法创建集合。

```
db.createCollection(,{capped:,size:,max:})
```

### 工作原理

让我们按照本节的步骤创建一个名为 `student` 的固定集合。

##### 步骤 1：创建固定集合

要创建一个名为 `student` 的固定集合，请使用以下命令。

```
db.createCollection("student",{capped: true,size:1000,max:2})
```

这里，`size` 表示集合的最大字节数，`max` 表示允许的最大文档数量。

输出如下：

```
> db.createCollection("student",{capped: true,size:1000,max:2})
{ "ok" : 1 }
>
```

要检查一个集合是否是固定的，请在 `mongo` shell 中输入以下命令。

```
> db.student.isCapped()
true
>
```

我们可以看到 `student` 集合是固定的。现在，我们将向 `student` 集合中插入一些文档，如下所示。

```
db.student.insert([{ "_id" : 10001, "name" : "Taanushree A S", "age" : 10 },{ "_id" : 10002, "name" : "Aruna M S", "age" : 14 }])
```

输出如下：

```
> db.student.insert([{ "_id" : 10001, "name" : "Taanushree A S", "age" : 10 },{ "_id" : 10002, "name" : "Aruna M S", "age" : 14 }])
BulkWriteResult({
"writeErrors" : [ ],
"writeConcernErrors" : [ ],
"nInserted" : 2,
"nUpserted" : 0,
"nMatched" : 0,
"nModified" : 0,
"nRemoved" : 0,
"upserted" : [ ]
})
>
```

##### 步骤 2：查询固定集合

要查询一个固定集合，请使用以下命令。

```
db.student.find()
```

输出如下：

```
> db.student.find()
{ "_id" : 10001, "name" : "Taanushree A S", "age" : 10 }
{ "_id" : 10002, "name" : "Aruna M S", "age" : 14 }
>
```

这里，MongoDB 按插入顺序检索结果。要按相反顺序检索文档，请使用 `sort()` 并将 `$natural` 参数设置为 -1，如下所示。

```
> db.student.find().sort({$natural:-1})
{ "_id" : 10002, "name" : "Aruna M S", "age" : 14 }
{ "_id" : 10001, "name" : "Taanushree A S", "age" : 10 }
>
```

我们现在可以看到结果是反向的了。

##### 步骤 3：向已满的固定集合插入文档

让我们看看当我们尝试向一个已经达到容量的固定集合插入文档时会发生什么，使用以下命令。

```
db.student.insert({_id:10003,name:"Anba V M",age:16})
```

输出如下：

```
> db.student.insert({_id:10003,name:"Anba V M",age:16})
WriteResult({ "nInserted" : 1 })
```

要查询 `student` 集合，请使用以下命令。

```
> db.student.find()
{ "_id" : 10002, "name" : "Aruna M S", "age" : 14 }
{ "_id" : 10003, "name" : "Anba V M", "age" : 16 }
>
```

注意，第一个文档被第三个文档覆盖了，因为 `student` 固定集合允许的最大文档数量是两个。

## 创建操作

创建操作允许我们将文档插入到集合中。这些插入操作针对单个集合。所有写操作在单个文档级别是原子性的。

MongoDB 提供了几种插入文档的方法，列于表 2-1。

表 2-1 插入方法

| 方法 | 描述 |
| :--- | :--- |
| `db.collection.insertOne` | 插入单个文档 |
| `db.collection.insertMany` | 插入多个文档 |
| `db.collection.insert` | 插入单个或多个文档 |

### 配方 2-3. 插入文档

在本配方中，我们将讨论向集合插入文档的各种方法。

### 问题

你想向 `person` 集合插入文档。

### 解决方案

使用以下语法插入文档。

```
db.collection.insertOne()
db.collection.insertMany()
```

### 工作原理

让我们按照本节的步骤向 `person` 集合插入文档。

##### 步骤 1：插入单个文档

要向 `person` 集合插入单个文档，请使用以下命令。

```
db.person.insertOne({_id:1001,name:"Taanushree AS",age:10})
```

输出如下：

```
> db.person.insertOne({_id:1001,name:"Taanushree AS",age:10})
{ "acknowledged" : true, "insertedId" : 1001 }
>
```

`insertOne()` 返回一个包含新插入文档 `_id` 的文档。

如果你没有指定 `_id` 字段，MongoDB 会生成一个带有 `ObjectId` 值的 `_id` 字段。`_id` 字段充当主键。

这里有一个例子：

```
> db.person.insertOne({name:"Aruna MS",age:14})
{
"acknowledged" : true,
"insertedId" : ObjectId("5bac7a5113572c1fb994d2fe")
}
>
```

##### 步骤 2：插入多个文档

要向 `person` 集合插入多个文档，请使用以下命令。

```
db.person.insertMany([{_id:1003,name:"Anba V M",age:16},{_id:1004,name:"shobana",age:44}])
```

向 `insertMany()` 方法传递一个文档数组以插入多个文档。

输出如下：

```
> db.person.insertMany([{_id:1003,name:"Anba V M",age:16},{_id:1004,name:"shobana",age:44}])
{ "acknowledged" : true, "insertedIds" : [ 1003, 1004 ] }
>
```

## 读操作

读操作允许我们从集合中检索文档。MongoDB 提供了 `find()` 方法来查询文档。

`find()` 命令的语法是

```
db.collection.find()
```

你可以在 `find()` 方法内部指定查询过滤器或条件，以基于条件查询文档。



### 配方 2-4. 查询文档

在本配方中，我们将讨论如何从集合中检索文档。

### 问题

你想从一个集合中检索文档。

### 解决方案

使用以下语法从集合中检索文档。

```
db.collection.find()
```

### 工作原理

让我们按照本节的步骤来查询集合中的文档。

##### 步骤 1：选择集合中的所有文档

要选择集合中的所有文档，请使用以下命令。

```
db.person.find({})
```

你需要向 `find()` 方法传递一个空的文档作为查询参数。

输出如下：

```
> db.person.find({})
{ "_id" : 1001, "name" : "Taanushree AS", "age" : 10 }
{ "_id" : ObjectId("5bac86dc773204ddade95819"), "name" : "Aruna MS", "age" : 14 }
{ "_id" : 1003, "name" : "Anba V M", "age" : 16 }
{ "_id" : 1004, "name" : "shobana", "age" : 44 }
>
```

##### 步骤 2：指定相等条件

要指定一个相等条件，你需要在查询中使用 `<字段>:<值>` 表达式来过滤文档。

要从 `person` 集合中选择 `name` 等于 `"shobana"` 的文档，命令如下：

```
db.person.find({name:"shobana"})
```

输出如下：

```
> db.person.find({name:"shobana"})
{ "_id" : 1004, "name" : "shobana", "age" : 44 }
>
```

##### 步骤 3：使用查询操作符指定条件

要从 `person` 集合中选择所有 `age` 大于 10 的文档，命令如下：

```
db.person.find({age:{$gt:10}})
```

输出如下：

```
> db.person.find({age:{$gt:10}})
{ "_id" : ObjectId("5bac86dc773204ddade95819"), "name" : "Aruna MS", "age" : 14 }
{ "_id" : 1003, "name" : "Anba V M", "age" : 16 }
{ "_id" : 1004, "name" : "shobana", "age" : 44 }
>
```

##### 步骤 4：指定 AND 条件

在一个复合查询中，你可以为多个字段指定条件。要从 `person` 集合中选择 `name` 等于 `"shobana"` **且** `age` 大于 10 的所有文档，命令如下：

```
db.person.find({ name:"shobana",age:{$gt:10}})
```

输出如下：

```
> db.person.find({ name:"shobana",age:{$gt:10}})
{ "_id" : 1004, "name" : "shobana", "age" : 44 }
>
```

##### 步骤 5：指定 OR 条件

你可以在复合查询中使用 `$or` 来通过逻辑或连接每个子句。

操作符 `$or` 从集合中选择至少匹配所选条件之一的文档。

要从 `person` 集合中选择 `name` 等于 `"shobana"` **或** `age` 等于 20 的所有文档，命令如下：

```
db.person.find( { $or: [ { name: "shobana" }, { age: { $eq: 20 } } ] } )
```

输出如下：

```
> db.person.find( { $or: [ { name: "shobana" }, { age: { $eq: 20 } } ] } )
{ "_id" : 1004, "name" : "shobana", "age" : 44 }
>
```

## 更新操作

更新操作允许我们修改集合中的现有文档。MongoDB 提供了表 2-2 中列出的更新方法。

表 2-2
更新方法

| `db.collection.updateOne` | 修改单个文档 |
| `db.collection.updateMany` | 修改多个文档 |
| `db.collection.replaceOne` | 替换集合中匹配过滤器的第一个文档 |

在 MongoDB 中，更新操作针对单个集合。

### 配方 2-5. 更新文档

在本配方中，我们将讨论如何更新文档。

### 问题

你想更新集合中的文档。

### 解决方案

以下方法用于更新集合中的文档。

```
db.collection.updateOne()
db.collection.updateMany()
db.collection.replaceOne()
```

在 `mongo` shell 中执行以下代码来创建 `student` 集合。

```
db.student.insertMany([{_id:1001,name:"John",marks:{english:35,maths:38},result:"pass"},{_id:1002,name:"Jack",marks:{english:15,maths:18},result:"fail"},{_id:1003,name:"John",marks:{english:25,maths:28},result:"pass"},{_id:1004,name:"James",marks:{english:32,maths:40},result:"pass"},{_id:1005,name:"Joshi",marks:{english:15,maths:18},result:"fail"},{_id:1006,name:"Jack",marks:{english:35,maths:36},result:"pass"}])
```

### 工作原理

让我们按照本节的步骤来更新集合中的文档。

##### 步骤 1：更新单个文档

MongoDB 提供了修改操作符（如 `$set`）来修改字段值。

要使用更新操作符，需要向 `update` 方法传递一个形式如下的更新文档：

```
{
  <更新操作符>: { <字段 1>: <值 1>, ... },
  <更新操作符>: { <字段 2>: <值 2>, ... },
  ...
}
```

要更新 `name` 等于 `joshi` 的第一个文档，请执行以下命令。

```
db.student.updateOne({name: "Joshi"},{$set:{"marks.english": 20}})
```

输出如下：

```
> db.student.updateOne({name: "Joshi"},{$set:{"marks.english": 20}})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 1 }
>
```

确认 `student` 集合中的 `marks` 字段已更新：

```
> db.student.find({name:"Joshi"})
{ "_id" : 1005, "name" : "Joshi", "marks" : { "english" : 20, "maths" : 18 }, "result" : "fail" }
>
```

##### 注意

你无法更新 `_id` 字段。

##### 步骤 2：更新多个文档

使用以下代码更新 `student` 集合中所有 `result` 等于 `fail` 的文档。

```
db.student.updateMany( { "result":"fail" }, { $set: { "marks.english": 20, "marks.maths": 20 }  })
```

输出如下：

```
> db.student.updateMany( { "result":"fail" }, { $set: {"marks.english": 20, "marks.maths": 20 }})
{ "acknowledged" : true, "matchedCount" : 2, "modifiedCount" : 2 }
>
```

这里的 `modifiedCount` 是 2，表明前面的命令修改了 `student` 集合中的两个文档。

##### 步骤 3：替换文档

你可以通过向 `db.collection.replaceOne()` 传递一个全新的文档作为第二个参数，来替换除 `_id` 字段外的整个文档内容，如下所示。

执行以下命令，从 `student` 集合中替换第一个 `name` 等于 `"John"` 的文档。

```
db.student.replaceOne( { name: "John" }, {_id:1001,name:"John",marks:{english:36,maths:39},result:"pass"})
```

输出如下：

```
> db.student.replaceOne( { name: "John" }, {_id:1001,name:"John",marks:{english:36,maths:39},result:"pass"})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 1 }
>
```

##### 注意

不要在替换文档中包含更新操作符。你可以在替换文档中省略 `_id` 字段，因为 `_id` 字段是不可变的。但是，如果你想包含 `_id` 字段，请使用与当前值相同的值。

## 删除操作

删除操作允许我们从集合中删除文档。MongoDB 提供了两种删除方法，如表 2-3 所示。

表 2-3
删除方法

| `db.collection.deleteOne` | 删除单个文档 |
| `db.collection.deleteMany` | 删除多个文档 |

在 MongoDB 中，删除操作针对单个集合。



### 配方 2-6. 删除文档

在本配方中，我们将讨论如何删除文档。

### 问题

你想从集合中删除文档。

### 解决方案

以下方法用于从集合中删除文档。

```
db.collection.deleteOne()
db.collection.deleteMany()
```

### 工作原理

让我们按照本节的步骤来删除集合中的文档。

##### 步骤 1：删除匹配条件的单个文档

要从 `student` 集合中删除 `name` 为 `"John"` 的文档，请使用以下代码。

```
db.student.deleteOne({name: "John"})
```

输出如下：

```
> db.student.deleteOne({name: "John"})
{ "acknowledged" : true, "deletedCount" : 1 }
>
```

##### 步骤 2：删除所有匹配条件的文档

要从 `student` 集合中删除所有 `name` 为 `"Jack"` 的文档，请使用以下代码。

```
db.student.deleteMany({name: "Jack"})
```

输出如下：

```
> db.student.deleteMany({name: "Jack"})
{ "acknowledged" : true, "deletedCount" : 2 }
>
```

##### 步骤 3：删除集合中的所有文档

要删除 `student` 集合中的所有文档，请向 `db.student.deleteMany()` 方法传递一个空的 `filter` 文档。

```
db.student.deleteMany({})
```

输出如下：

```
> db.student.deleteMany({})
{ "acknowledged" : true, "deletedCount" : 3 }
>
```

## MongoDB 导入与导出

MongoDB 导入工具允许我们从 JSON、逗号分隔值 (CSV) 和制表符分隔值 (TSV) 文件中导入内容。MongoDB 导入仅支持 UTF-8 编码的文件。

MongoDB 导出工具允许我们将存储在 MongoDB 实例中的数据导出为 JSON 或 CSV 文件。

### 配方 2-7. 使用 Mongo Import

在本配方中，我们将讨论如何从 CSV 文件导入数据。

### 问题

你想将 `student.csv` 文件中的数据导入到 `students` 集合中。

### 解决方案

以下命令用于执行 Mongo 导入。

```
mongoimport
```

##### 注意

要使用 Mongo 导入命令，需要启动 `mongod` 进程，并打开另一个命令提示符来执行 `mongoimport` 命令。

### 工作原理

让我们按照本节的步骤来使用 Mongo 导入。

在 `C:\Sample\` 目录下，使用以下数据创建一个 `student.csv` 文件。

```
_id,name,class
1,John,II
2,James,III
3,Joshi,I
```

执行以下命令，将数据从 `student.csv` 文件导入到 `students` 集合。

```
mongoimport --db student --collection students --type csv --headerline --file c:\Sample\student.csv
```

输出如下：

```
C:\Program Files\MongoDB\Server\4.0\bin>mongoimport --db student --collection students --type csv --headerline --file c:\Sample\student.csv
2018-09-28T07:00:20.909+0530    connected to: localhost
2018-09-28T07:00:20.969+0530    imported 3 documents
C:\Program Files\MongoDB\Server\4.0\bin>
```

要确认 `students` 集合已存在，请执行以下命令。

```
> use student
switched to db student
> show collections
students
> db.students.find()
{ "_id" : 1, "name" : "John", "class" : "II" }
{ "_id" : 3, "name" : "Joshi", "class" : "I" }
{ "_id" : 2, "name" : "James", "class" : "III" }
>
```

### 配方 2-8. 使用 Mongo Export

在本配方中，我们将讨论如何将 `students` 集合中的数据导出到 `student.json` 文件。

### 问题

你想将 `students` 集合中的数据导出到 `student.json` 文件。

### 解决方案

以下命令用于执行 Mongo 导出。

```
mongoexport
```

##### 注意

要使用 Mongo 导出命令，需要启动 `mongod` 进程，并打开另一个命令提示符来执行 `mongoexport` 命令。

### 工作原理

让我们按照本节的步骤来使用 Mongo 导出。

执行以下命令，将数据从 `students` 集合导出到 `student.json` 文件。

```
mongoexport --db student --collection students --out C:\Sample\student.json
```

输出如下：

```
C:\Program Files\MongoDB\Server\4.0\bin>mongoexport --db student --collection students --out C:\Sample\student.json
2018-09-28T07:11:19.446+0530    connected to: localhost
2018-09-28T07:11:19.459+0530    exported 3 records
```

要确认导出成功，请打开 `student.json` 文件。

```
{"_id":1,"name":"John","class":"II"}
{"_id":2,"name":"James","class":"III"}
{"_id":3,"name":"Joshi","class":"I"}
```

## MongoDB 中的嵌入式文档

使用嵌入式文档允许我们将一个文档嵌入到另一个文档中。考虑以下示例。

```
{_id:1001,name:"John",marks:{english:35,maths:38},result:"pass"}
```

这里，`marks` 字段包含一个嵌入式文档。

执行以下代码创建一个 `employee` 数据库和 `employee` 集合。

```
use employee
db.employee.insertMany([{_id:1001,name:"John",address:{previous:"123,1st Main",current:"234,2nd Main"},unit:"Hadoop"},{_id:1002,name:"Jack", address:{previous:"Cresent Street",current:"234,Bald Hill Street"},unit:"MongoDB"},{_id:1003,name:"James", address:{previous:"Cresent Street",current:"234,Hill Street"},unit:"Spark"}])
```

### 配方 2-9. 查询嵌入式文档

在本配方中，我们将讨论如何查询嵌入式文档。

### 问题

你想查询嵌入式文档。

### 解决方案

以下命令用于查询嵌入式文档。

```
db.collection.find()
```

你需要将过滤文档 `{<field>:<value>}` 传递给 `find()` 方法，其中 `<value>` 是要匹配的文档。

### 工作原理

让我们按照本节的步骤来查询嵌入式文档。

##### 步骤 1：匹配嵌入式或嵌套文档

要对嵌入式文档执行相等匹配，你需要在 `<value>` 文档中指定完全匹配的文档，包括字段顺序。

要选择所有 `address` 字段等于文档 `{previous:"Cresent Street",current:"234,Bald Hill Street"}` 的文档：

```
db.employee.find( { address: { previous:"Cresent Street",current:"234,Bald Hill Street" }} )
```

输出如下：

```
> db.employee.find( { address: { previous:"Cresent Street",current:"234,Bald Hill Street" }} )
{ "_id" : 1002, "name" : "Jack", "address" : { "previous" : "Cresent Street", "current" : "234,Bald Hill Street" }, "unit" : "MongoDB" }
>
```

##### 步骤 2：查询嵌套字段

我们可以使用点表示法来查询嵌入式文档的字段。

要选择所有嵌套在 `address` 字段中的 `previous` 字段等于 `"Cresent Street"` 的文档，

```
db.employee.find( { "address.previous": "Cresent Street" } )
```

输出如下：

```
> db.employee.find( { "address.previous": "Cresent Street" } )
{ "_id" : 1002, "name" : "Jack", "address" : { "previous" : "Cresent Street", "current" : "234,Bald Hill Street" }, "unit" : "MongoDB" }
{ "_id" : 1003, "name" : "James", "address" : { "previous" : "Cresent Street", "current" : "234,Hill Street" }, "unit" : "Spark" }
>
```

## 处理数组

在 MongoDB 中，我们可以将字段值指定为数组。

### 配方 2-10. 处理数组

在本配方中，我们将讨论如何查询数组。

### 问题

你想查询数组。

### 解决方案

以下命令用于查询数组。

```
db.collection.find()
```

### 工作原理

让我们按照本节的步骤来查询嵌入式文档。



### MongoDB 数组操作指南

#### 步骤 1：匹配数组

执行此处显示的代码，在 `employee` 数据库下创建 `employeedetails` 集合。

```javascript
db.employeedetails.insertMany([
{ name: "John", projects: ["MongoDB", "Hadoop","Spark"], scores:[25,28,29] }, { name: "James", projects: [ "Cassandra","Spark"], scores:[26,24,23]}, { name: "Smith", projects: [ "Hadoop","MongoDB"], scores:[22,28,26]} ] )
```

要在数组上指定相等条件，需要在查询文档 `{<field>:<value>}` 的 `<value>` 中指定要匹配的确切数组。

要选择所有 `projects` 值是一个数组，且恰好包含 `"Hadoop"` 和 `"MongoDB"` 两个元素（并按指定顺序排列）的文档，请使用以下命令。

```javascript
db.employeedetails.find( { projects: ["Hadoop", "MongoDB"] } )
```

输出结果如下：

```javascript
> db.employeedetails.find( { projects: ["Hadoop", "MongoDB"] } )
{ "_id" : ObjectId("5badcbd5f10ab299920f072a"), "name" : "Smith", "projects" : [ "Hadoop", "MongoDB" ], "scores" : [ 22, 28, 26 ] }
>
```

#### 步骤 2：查询数组中的元素

要查询所有 `projects` 是一个数组且包含字符串 `"MongoDB"` 作为其元素之一的文档，请执行以下命令。

```javascript
db.employeedetails.find( { projects: "MongoDB" } )
```

输出结果如下：

```javascript
> db.employeedetails.find( { projects: "MongoDB" } )
{ "_id" : ObjectId("5badcbd5f10ab299920f0728"), "name" : "John", "projects" : [ "MongoDB", "Hadoop", "Spark" ], "scores" : [ 25, 28, 29 ] }
{ "_id" : ObjectId("5badcbd5f10ab299920f072a"), "name" : "Smith", "projects" : [ "Hadoop", "MongoDB" ], "scores" : [ 22, 28, 26 ] }
>
```

#### 步骤 3：指定查询操作符

要查询所有 `scores` 字段是一个数组，且至少包含一个值大于 26 的元素的文档，我们可以使用 `$gt` 操作符。

```javascript
db.employeedetails.find( { scores:{$gt:26} } )
```

输出结果如下：

```javascript
> db.employeedetails.find( { scores:{$gt:26} } )
{ "_id" : ObjectId("5badcbd5f10ab299920f0728"), "name" : "John", "projects" : [ "MongoDB", "Hadoop", "Spark" ], "scores" : [ 25, 28, 29 ] }
{ "_id" : ObjectId("5badcbd5f10ab299920f072a"), "name" : "Smith", "projects" : [ "Hadoop", "MongoDB" ], "scores" : [ 22, 28, 26 ] }
>
```

#### 步骤 4：对数组元素使用复合筛选条件

可以在数组上指定复合筛选条件，如下所示。

```javascript
db.employeedetails.find( { scores: { $gt: 20, $lt: 24 } } )
```

输出结果如下：

```javascript
> db.employeedetails.find( { scores: { $gt: 20, $lt: 24 } } )
{ "_id" : ObjectId("5badcbd5f10ab299920f0729"), "name" : "James", "projects" : [ "Cassandra", "Spark" ], "scores" : [ 26, 24, 23 ] }
{ "_id" : ObjectId("5badcbd5f10ab299920f072a"), "name" : "Smith", "projects" : [ "Hadoop", "MongoDB" ], "scores" : [ 22, 28, 26 ] }
>
```

在这里，`scores` 数组中的一个元素可以满足大于 20 的条件，另一个元素可以满足小于 24 的条件，或者单个元素可以同时满足这两个条件。

#### 步骤 5：使用 `$elemMatch` 操作符

`$elemMatch` 操作符允许我们对数组元素指定多个条件，要求至少有一个数组元素满足所有指定条件。

```javascript
db.employeedetails.find( { scores: { $elemMatch: { $gt: 23, $lt: 27 } } } )
```

输出结果如下：

```javascript
> db.employeedetails.find( { scores: { $elemMatch: { $gt: 23, $lt: 27 } } } )
{ "_id" : ObjectId("5badcbd5f10ab299920f0728"), "name" : "John", "projects" : [ "MongoDB", "Hadoop", "Spark" ], "scores" : [ 25, 28, 29 ] }
{ "_id" : ObjectId("5badcbd5f10ab299920f0729"), "name" : "James", "projects" : [ "Cassandra", "Spark" ], "scores" : [ 26, 24, 23 ] }
{ "_id" : ObjectId("5badcbd5f10ab299920f072a"), "name" : "Smith", "projects" : [ "Hadoop", "MongoDB" ], "scores" : [ 22, 28, 26 ] }
>
```

#### 步骤 6：按索引位置查询数组元素

使用点表示法按索引位置查询数组元素。

使用此代码选择所有 `scores` 数组中第三个元素大于 26 的文档。

```javascript
db.employeedetails.find( { "scores.2": { $gt: 26 } } )
```

输出结果如下：

```javascript
> db.employeedetails.find( { "scores.2": { $gt: 26 } } )
{ "_id" : ObjectId("5badcbd5f10ab299920f0728"), "name" : "John", "projects" : [ "MongoDB", "Hadoop", "Spark" ], "scores" : [ 25, 28, 29 ] }
>
```

#### 步骤 7：使用 `$size` 操作符

您可以使用 `$size` 操作符按元素数量查询数组。

要选择所有 `projects` 数组有两个元素的文档，请使用此代码。

```javascript
db.employeedetails.find( { "projects": { $size: 2 } } )
```

输出结果如下：

```javascript
> db.employeedetails.find( { "projects": { $size: 2 } } )
{ "_id" : ObjectId("5badcbd5f10ab299920f0729"), "name" : "James", "projects" : [ "Cassandra", "Spark" ], "scores" : [ 26, 24, 23 ] }
{ "_id" : ObjectId("5badcbd5f10ab299920f072a"), "name" : "Smith", "projects" : [ "Hadoop", "MongoDB" ], "scores" : [ 22, 28, 26 ] }
```

#### 步骤 8：使用 `$push` 操作符

您可以使用 `$push` 操作符向数组中添加一个值。

要添加工作地点详情，请使用此代码。

```javascript
db.employeedetails.update({name:"James"},{$push:{Location: "US"}})
```

以下是用于检查结果的查询：

```javascript
> db.employeedetails.find({name:"James"})
{ "_id" : ObjectId("5c04bef3540e90478dd92f4e"), "name" : "James", "projects" : [ "Cassandra", "Spark" ], "scores" : [ 26, 24, 23 ], "Location" : [ "US" ] }
```

要向数组追加多个值，请使用 `$each` 操作符。

```javascript
db.employeedetails.update({name: "Smith"},{$push:{Location:{$each:["US","UK"]}}})
```

以下是用于检查结果的查询：

```javascript
> db.employeedetails.find({name:"Smith"})
{ "_id" : ObjectId("5c04bef3540e90478dd92f4f"), "name" : "Smith", "projects" : [ "Hadoop", "MongoDB" ], "scores" : [ 22, 28, 26 ], "Location" : [ "US", "UK" ] }
>
```

#### 步骤 9：使用 `$addToSet` 操作符

您可以使用 `$addToSet` 操作符向数组中添加一个值。`$addToSet` 仅当值不存在时才将其添加到数组。如果值已存在，则不执行任何操作。

要向 `employeedetails` 集合添加 `hobbies`，请使用此代码。

```javascript
db.employeedetails.update( {name: "James"},  { $addToSet: {hobbies: [ "drawing", "dancing"]} })
```

以下是用于检查结果的查询：

```javascript
> db.employeedetails.find({name:"James"})
{ "_id" : ObjectId("5c04bef3540e90478dd92f4e"), "name" : "James", "projects" : [ "Cassandra", "Spark" ], "scores" : [ 26, 24, 23 ], "Location" : [ "US" ], "hobbies" : [ [ "drawing", "dancing" ] ] }
>
```

#### 步骤 10：使用 `$pop` 操作符

您可以使用 `$pop` 操作符删除数组的第一个或最后一个元素。

要删除 `scores` 数组中的第一个元素，请使用此代码。

```javascript
db.employeedetails.update( {name: "James"},{ $pop: {scores:-1}})
```

以下是用于检查结果的查询：

```javascript
> db.employeedetails.find({name:"James"})
{ "_id" : ObjectId("5c04bef3540e90478dd92f4e"), "name" : "James", "projects" : [ "Cassandra", "Spark" ], "scores" : [ 24, 23 ], "Location" : [ "US" ], "hobbies" : [ [ "drawing", "dancing" ] ] }
>
```

要删除 `scores` 数组中的最后一个元素，请使用此代码。

```javascript
db.employeedetails.update( {name: "James"},{ $pop: {scores:1}})
```

以下是用于检查结果的查询：

```javascript
> db.employeedetails.find({name:"James"})
{ "_id" : ObjectId("5c04bef3540e90478dd92f4e"), "name" : "James", "projects" : [ "Cassandra", "Spark" ], "scores" : [ 24 ], "Location" : [ "US" ], "hobbies" : [ [ "drawing", "dancing" ] ] }
>
```



## 食谱 2-11. 查询嵌入文档数组

在本节中，我们将讨论如何查询嵌入文档的数组。

### 问题

您需要查询一个包含嵌入文档的数组。

### 解决方案

以下命令用于查询包含嵌入文档的数组。

```
db.collection.find()
```

### 工作原理

让我们按照本节的步骤来查询嵌入文档的数组。

##### 步骤 1：查询数组中的文档

执行以下代码创建 `student` 数据库和 `studentmarks` 集合。

```
> use student
> db.studentmarks.insertMany([{name:"John",marks:[{class: "II", total: 489},{ class: "III", total: 490 }]},{name: "James",marks:[{class: "III", total: 469 },{class: "IV", total: 450}]},{name:"Jack",marks:[{class:"II", total: 489 },{ class: "III", total: 390}]},{name:"Smith", marks:[{class: "III", total: 489}, {class: "IV", total: 490}]}, {name:"Joshi",marks:[{class: "II", total: 465}, { class: "III", total: 470}]}])
```

此处所示的命令选择 `marks` 数组字段中所有匹配指定文档的文档。

```
db.studentmarks.find( { "marks": {class: "II", total: 489}})
```

输出如下，

```
> db.studentmarks.find( { "marks": {class: "II", total: 489}})
{ "_id" : ObjectId("5bae10e6f10ab299920f073f"), "name" : "John", "marks" : [ { "class" : "II", "total" : 489 }, { "class" : "III", "total" : 490 } ] }
{ "_id" : ObjectId("5bae10e6f10ab299920f0741"), "name" : "Jack", "marks" : [ { "class" : "II", "total" : 489 }, { "class" : "III", "total" : 390 } ] }
>
```

##### 注意

对嵌入文档数组进行相等匹配要求指定的文档完全匹配，包括字段顺序。

##### 步骤 2：查询文档数组中嵌入的字段

您可以使用点号表示法来查询文档数组中嵌入的字段。以下命令选择 `marks` 数组中所有至少有一个嵌入文档包含 `total` 字段且该字段值小于 400 的文档。

```
db.studentmarks.find( { 'marks.total': { $lt: 400 } } )
```

输出如下，

```
> db.studentmarks.find( { 'marks.total': { $lt: 400 } } )
{ "_id" : ObjectId("5bae10e6f10ab299920f0741"), "name" : "Jack", "marks" : [ { "class" : "II", "total" : 489 }, { "class" : "III", "total" : 390 } ] }
>
```

##### 步骤 3：使用数组索引查询嵌入文档中的字段

此处所示的命令选择 `marks` 数组中所有文档的第一个元素包含 `class` 字段且该字段值等于 `"II"` 的文档。

```
db.studentmarks.find( { 'marks.0.class': "II" } )
```

输出如下，

```
> db.studentmarks.find( { 'marks.0.class': "II" } )
{ "_id" : ObjectId("5bae10e6f10ab299920f073f"), "name" : "John", "marks" : [ { "class" : "II", "total" : 489 }, { "class" : "III", "total" : 490 } ] }
{ "_id" : ObjectId("5bae10e6f10ab299920f0741"), "name" : "Jack", "marks" : [ { "class" : "II", "total" : 489 }, { "class" : "III", "total" : 390 } ] }
{ "_id" : ObjectId("5bae10e6f10ab299920f0743"), "name" : "Joshi", "marks" : [ { "class" : "II", "total" : 465 }, { "class" : "III", "total" : 470 } ] }
>
```

## 从查询中投影要返回的字段

默认情况下，MongoDB 中的查询会返回匹配文档中的所有字段。您可以使用投影文档来限制要返回的字段。

执行以下代码在 `student` 数据库下创建 `studentdetails` 集合。

```
db.studentdetails.insertMany( [ {name: "John", result: "pass", marks: { english: 25, maths: 23, science: 25 }, grade: [ { class: "A", total: 73 } ] },{name: "James", result: "pass", marks: { english: 24, maths: 25, science: 25 }, grade: [ { class: "A", total: 74 } ] },{name: "James", result: "fail", marks: { english: 12, maths: 13, science: 15 }, grade: [ { class: "C", total: 40 } ] }])
```

## 食谱 2-12. 限制查询返回的字段

在本节中，我们将讨论如何限制查询返回的字段。

### 问题

您需要限制查询返回的字段。

### 解决方案

以下命令用于限制查询返回的字段。

```
db.collection.find()
```

### 工作原理

让我们按照本节的步骤来限制查询返回的字段。

##### 步骤 1：仅返回指定字段和 `_id` 字段

您可以通过在投影文档中将 `<field>` 值设置为 `1` 来投影所需的字段。

此处所示的命令仅返回结果集中 `name` 等于 `"John"` 的文档的 `_id`、`marks` 和 `result` 字段。

```
db.studentdetails.find( { name: "John" }, { marks: 1, result: 1 } )
```

输出如下，

```
> db.studentdetails.find( { name: "John" }, { marks: 1, result: 1 } )
{ "_id" : ObjectId("5bae3ed2f10ab299920f0744"), "result" : "pass", "marks" : { "english" : 25, "maths" : 23, "science" : 25 } }
>
```

##### 步骤 2：抑制 `_id` 字段

您可以通过在投影文档中将其排除 `<field>` 设置为 0 来抑制 `_id` 字段。

以下命令在结果集中抑制 `_id` 字段。

```
db.studentdetails.find( { name: "John" }, { marks: 1, result: 1,_id:0 } )
```

输出如下，

```
> db.studentdetails.find( { name: "John" }, { marks: 1, result: 1,_id:0 } )
{ "result" : "pass", "marks" : { "english" : 25, "maths" : 23, "science" : 25 } }
>
```

##### 步骤 3：排除多个字段

您可以通过在投影文档中将 `<field>` 设置为 0 来排除字段。

以下命令在结果集中抑制 `_id` 和 `result` 字段。

```
db.studentdetails.find( { name: "John" }, { result:0,_id:0 } )
```

输出如下，

```
> db.studentdetails.find( { name: "John" }, { result:0,_id:0 } )
{ "name" : "John", "marks" : { "english" : 25, "maths" : 23, "science" : 25 }, "grade" : [ { "class" : "A", "total" : 73 } ] }
>
```

##### 步骤 4：返回嵌入文档中的特定字段

您可以使用点号表示法来引用嵌入的字段，并在投影文档中将 `<field>` 设置为 `1`。

示例如下：

```
db.studentdetails.find( { name: "John }, { result:1, grade: 1, "marks.english": 1 })
```

输出如下，

```
> db.studentdetails.find( { name: "John" }, { result:1, grade: 1, "marks.english": 1 })
{ "_id" : ObjectId("5bae3ed2f10ab299920f0744"), "result" : "pass", "marks" : { "english" : 25 }, "grade" : [ { "class" : "A", "total" : 73 } ] }
>
```

## 查询 Null 或缺失的字段

执行以下代码创建 `sample` 集合。

```
db.sample.insertMany([ { _id: 1, name: null }, { _id: 2 }])
```



## 食谱 2-13. 如何查询 Null 或缺失字段

在本食谱中，我们将讨论如何在 MongoDB 中查询 null 或缺失值。

### 问题

你希望查询 null 或缺失值。

### 解决方案

以下命令用于查询 null 或缺失值。
```
db.collection.find()
```

### 工作原理

让我们按照本节的步骤来查询 null 或缺失值。

##### 步骤 1：相等过滤器

`{name:null}` 查询会匹配那些包含值为 `null` 的 `name` 字段，或者不包含 `name` 字段的文档。
```
db.sample.find( { name: null } )
```
输出如下：
```
> db.sample.insertMany([{ _id: 1, name: null }, { _id: 2 }])
{ "acknowledged" : true, "insertedIds" : [ 1, 2 ] }
>
```
这里，它返回了两个文档。

##### 步骤 2：类型检查

此处显示的查询仅返回那些包含值为 `null` 的 `name` 字段的文档。在 BSON 中，将 `null` 值设置为 `10`。
```
db.sample.find( { name: { $type: 10 } } )
```
输出如下：
```
> db.sample.find( { name: { $type: 10 } } )
{ "_id" : 1, "name" : null }
>
```

##### 步骤 3：存在性检查

你可以使用 `$exists` 操作符来检查字段是否存在。

以下查询返回不包含 `name` 字段的文档。
```
db.sample.find( { name: { $exists: false } } )
```
输出如下：
```
> db.sample.find( { name: { $exists: false } } )
{ "_id" : 2 }
>
```

## 食谱 2-14. 如何迭代游标

在本食谱中，我们将讨论如何在 `mongo` shell 中迭代游标。

### 问题

你希望在 `mongo` shell 中迭代一个游标。

### 解决方案

以下语法用于迭代游标。
```
var myCursor = db.collection.find()
myCursor
```

### 工作原理

让我们按照本节的步骤来迭代游标。首先，创建一个 `numbers` 集合。
```
db.numbers.insert({_id:1,number:1});
db.numbers.insert({_id:2,number:2});
db.numbers.insert({_id:3,number:3});
db.numbers.insert({_id:4,number:4});
db.numbers.insert({_id:5,number:5});
db.numbers.insert({_id:6,number:6});
db.numbers.insert({_id:7,number:7});
db.numbers.insert({_id:8,number:8});
db.numbers.insert({_id:9,number:9});
db.numbers.insert({_id:10,number:10});
db.numbers.insert({_id:11,number:11});
db.numbers.insert({_id:12,number:12});
db.numbers.insert({_id:13,number:13});
db.numbers.insert({_id:14,number:14});
db.numbers.insert({_id:15,number:15});
db.numbers.insert({_id:16,number:16});
db.numbers.insert({_id:17,number:17});
db.numbers.insert({_id:18,number:18});
db.numbers.insert({_id:19,number:19});
db.numbers.insert({_id:20,number:20});
db.numbers.insert({_id:21,number:21});
db.numbers.insert({_id:22,number:22});
db.numbers.insert({_id:23,number:23});
db.numbers.insert({_id:24,number:24});
db.numbers.insert({_id:25,number:25});
```
发出以下命令以迭代游标。
```
var myCursor = db.numbers.find()
myCursor
```
输出如下：
```
> var myCursor = db.numbers.find()
> myCursor
{ "_id" : 1, "number" : 1 }
{ "_id" : 2, "number" : 2 }
{ "_id" : 3, "number" : 3 }
{ "_id" : 4, "number" : 4 }
{ "_id" : 5, "number" : 5 }
{ "_id" : 6, "number" : 6 }
{ "_id" : 7, "number" : 7 }
{ "_id" : 8, "number" : 8 }
{ "_id" : 9, "number" : 9 }
{ "_id" : 10, "number" : 10 }
{ "_id" : 11, "number" : 11 }
{ "_id" : 12, "number" : 12 }
{ "_id" : 13, "number" : 13 }
{ "_id" : 14, "number" : 14 }
{ "_id" : 15, "number" : 15 }
{ "_id" : 16, "number" : 16 }
{ "_id" : 17, "number" : 17 }
{ "_id" : 18, "number" : 18 }
{ "_id" : 19, "number" : 19 }
{ "_id" : 20, "number" : 20 }
Type "it" for more
> it
{ "_id" : 21, "number" : 21 }
{ "_id" : 22, "number" : 22 }
{ "_id" : 23, "number" : 23 }
{ "_id" : 24, "number" : 24 }
{ "_id" : 25, "number" : 25 }
>
```
你也可以使用游标方法 `next()` 来访问文档。
```
var myCursor=db.numbers.find({});
while(myCursor.hasNext()){
print(tojson(myCursor.next()));
}
```
输出如下：
```
> var myCursor=db.numbers.find({});
> while(myCursor.hasNext()){
... print(tojson(myCursor.next()));
... }
{ "_id" : 1, "number" : 1 }
{ "_id" : 2, "number" : 2 }
{ "_id" : 3, "number" : 3 }
{ "_id" : 4, "number" : 4 }
{ "_id" : 5, "number" : 5 }
{ "_id" : 6, "number" : 6 }
{ "_id" : 7, "number" : 7 }
{ "_id" : 8, "number" : 8 }
{ "_id" : 9, "number" : 9 }
{ "_id" : 10, "number" : 10 }
{ "_id" : 11, "number" : 11 }
{ "_id" : 12, "number" : 12 }
{ "_id" : 13, "number" : 13 }
{ "_id" : 14, "number" : 14 }
{ "_id" : 15, "number" : 15 }
{ "_id" : 16, "number" : 16 }
{ "_id" : 17, "number" : 17 }
{ "_id" : 18, "number" : 18 }
{ "_id" : 19, "number" : 19 }
{ "_id" : 20, "number" : 20 }
{ "_id" : 21, "number" : 21 }
{ "_id" : 22, "number" : 22 }
{ "_id" : 23, "number" : 23 }
{ "_id" : 24, "number" : 24 }
{ "_id" : 25, "number" : 25 }
>
```
你也可以使用 `forEach` 来迭代游标并访问文档。
```
var myCursor = db.numbers.find()
myCursor.forEach(printjson);
> var myCursor = db.numbers.find()
> myCursor.forEach(printjson);
{ "_id" : 1, "number" : 1 }
{ "_id" : 2, "number" : 2 }
{ "_id" : 3, "number" : 3 }
{ "_id" : 4, "number" : 4 }
{ "_id" : 5, "number" : 5 }
{ "_id" : 6, "number" : 6 }
{ "_id" : 7, "number" : 7 }
{ "_id" : 8, "number" : 8 }
{ "_id" : 9, "number" : 9 }
{ "_id" : 10, "number" : 10 }
{ "_id" : 11, "number" : 11 }
{ "_id" : 12, "number" : 12 }
{ "_id" : 13, "number" : 13 }
{ "_id" : 14, "number" : 14 }
{ "_id" : 15, "number" : 15 }
{ "_id" : 16, "number" : 16 }
{ "_id" : 17, "number" : 17 }
{ "_id" : 18, "number" : 18 }
{ "_id" : 19, "number" : 19 }
{ "_id" : 20, "number" : 20 }
{ "_id" : 21, "number" : 21 }
{ "_id" : 22, "number" : 22 }
{ "_id" : 23, "number" : 23 }
{ "_id" : 24, "number" : 24 }
{ "_id" : 25, "number" : 25 }
>
```

#### 使用 `limit()` 和 `skip()` 方法

`limit()` 方法用于限制查询结果中的文档数量，而 `skip()` 方法则跳过查询结果中给定数量的文档。



