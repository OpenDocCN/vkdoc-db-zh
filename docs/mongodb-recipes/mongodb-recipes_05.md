# 配方 2-15\. `limit()` 和 `skip()` 方法

在本配方中，我们将讨论如何使用 `limit()` 和 `skip()` 方法。

### 问题

你想限制并跳过集合中的文档。

### 解决方案

以下语法用于限制查询结果中的文档数量。

```
db.collection.find().limit(2)
```

以下语法用于跳过查询结果中给定数量的文档。

```
db.collection.find().skip(2)
```

### 工作原理

让我们遵循本节中的步骤来操作 `limit()` 和 `skip()` 方法。请参考在配方 2-14 中创建的 `numbers` 集合。

要显示前两个文档，请使用此代码。

```
db.numbers.find().limit(2)
```

输出如下，

```
> db.numbers.find().limit(2)
{ "_id" : 1, "number" : 1 }
{ "_id" : 2, "number" : 2 }
>
```

要跳过前五个文档，请使用此代码。

```
db.numbers.find().skip(5)
```

输出如下，

```
> db.numbers.find().skip(5)
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

## 使用 `Node.js` 和 MongoDB

MongoDB 是与 `Node.js` 一起使用的最流行的数据库之一。

