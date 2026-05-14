# 在 MongoDB 中删除数据

在本节中，我们将使用 `DeleteDBDocument` 应用程序来删除文档。`MongoCollection<TDocument>` 接口提供了多种删除文档的方法，如 表 1-10 所述。

表 1-10. 用于删除文档的 MongoCollection<TDocument> 方法

| 方法 | 描述 |
| --- | --- |
| `deleteMany(Bson filter)` | 使用查询过滤器从集合中删除所有文档，并将结果作为 `DeleteResult` 返回。`DeleteResult` 包含有关删除文档数量以及删除是否被确认的信息。 |
| `deleteOne(Bson filter)` | 根据查询过滤器删除一个文档，并将结果作为 `DeleteResult` 返回。 |
| `findOneAndDelete(Bson filter)` | 根据查询过滤器查找一个文档并将其删除。返回被删除的文档。 |
| `findOneAndDelete(Bson filter, FindOneAndDeleteOptions options)` | 根据查询过滤器和 `FindOneAndDeleteOptions` 选项查找一个文档并将其删除。返回被删除的文档。`FindOneAndDeleteOptions` 选项包括查找和删除的最长时间、排序条件以及要返回的文档字段。 |

1.  在 `DeleteDBDocument` 应用程序中，如前所述创建一个 `MongoClient` 客户端。从 `MongoClient` 实例为 `local` 数据库创建一个 `MongoDatabase` 实例，并从 `MongoDatabase` 实例为 `catalog` 集合创建一个 `MongoCollection<TDocument>` 实例。

```java
MongoClient mongoClient = new MongoClient(Arrays.asList(new ServerAddress("localhost", 27017)));
MongoDatabase db = mongoClient.getDatabase("local");
MongoCollection<Document> coll = db.getCollection("catalog");
```

2.  使用模型类 `Catalog` 创建并添加四个 `Document` 实例来设置 `Document` 字段。



