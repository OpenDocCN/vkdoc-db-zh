# MongoDB 连接字符串

以下是一些用于连接 MongoDB 服务器的较简单的连接字符串示例，适用于大多数场景：

```
mongodb://
mongodb://localhost
mongodb://user1:password1@localhost
mongodb://user1:password1@localhost/test
mongodb://localhost:27017,localhost:27018,localhost:27019
mongodb://example1.com,example2.com
```

一些主要的连接字符串选项在表 5-4 中讨论。

**表 5-4 连接字符串选项**

| 选项 | 描述 |
| --- | --- |
| `uri.replicaSet` | 如果 MongoDB 是副本集的成员，则为副本集的名称。与至少包含两个 MongoDB 实例的种子列表结合使用。 |
| `uri.connectTimeoutMS` | 连接超时（毫秒）。默认为永不超时，具体实现取决于驱动程序。 |
| `uri.maxPoolSize` | 如果驱动程序支持连接池（大多数驱动都支持），则指定连接池中的最大连接数。默认为 100。 |
| `uri.minPoolSize` | 连接池中的最小连接数。默认为 0。 |
| `uri.maxIdleTimeMS` | 连接在连接池中保持空闲状态的最大毫秒数，超过则被关闭。并非所有驱动都支持此选项。 |
| `uri.waitQueueMultiple` | 与 `maxPoolSize` 相乘，以提供在连接池中等待连接的最大线程数。 |
| `uri.w` | 写关注选项，以数字或字符串指定写关注级别。不同的写关注级别在表 5-5 中讨论。 |
| `uri.wtimeoutMS` | 等待复制成功超时的毫秒数。默认为 0，表示永不超时。 |
| `uri.journal` | 如果设置为 `true`，写操作会等待 MongoDB 确认写操作并将数据提交到磁盘日志后才返回。默认为 `false`。 |
| `uri.readPreference` | 适用于副本集。指定读取偏好模式。值可以为 `primary`、`primaryPreferred`、`secondary`、`secondaryPreferred`、`nearest`。 |
| `uri.authSource` | 指定与用户凭据关联的数据库名称。如果连接字符串中未指定用户名，则此设置被忽略，其值默认为连接字符串中指定的数据库。 |
| `uri.authMechanism` | 认证机制。值可以为 `SCRAM-SHA-1`、`MONGODB-CR`、`MONGODB-X509`、`GSSAPI` (Kerberos)、`PLAIN` (LDAP SASL)。 |

写关注选项描述了写操作的保证级别，例如写操作是否已提交。不同的写关注级别在表 5-5 中讨论。

**表 5-5 写关注级别**

| 级别 | 描述 |
| --- | --- |
| 0 | 驱动程序不确认写操作。 |
| 1 | 基本确认级别。独立的 MongoDB 实例或副本集中的主节点确认所有写操作。 |
| majority | 自 MongoDB 数据库 3.0 版本起，写操作仅在副本集的大多数投票成员确认后才返回。 |
| n | 适用于副本集。写操作仅在副本集中指定的 n 台服务器确认后才返回。不应设置为大于副本集成员数的值，否则写操作可能无限期等待副本集成员变得可用。 |
| tags | 适用于副本集。与副本集成员配置的标签集。写操作会等待配置了这些标签的副本集成员确认。 |

`connect(url, options, callback)` 方法中的 `options` 参数支持表 5-6 中讨论的选项。

**表 5-6 `connect()` 方法的选项**

| 名称 | 类型 | 描述 |
| --- | --- | --- |
| `uri_decode_auth` | boolean | 指定是否对用户名和密码进行 URI 解码。默认为 `false`。 |
| `db` | object | 要设置在 `Db` 对象上的选项哈希。默认值为 `null`。 |
| `server` | object | 要设置在服务器对象上的选项哈希。默认值为 `null`。 |
| `replSet` | object | 要设置在 `replSet` 对象上的选项哈希。默认值为 `null`。 |
| `mongos` | object | 要设置在 mongos 对象上的选项哈希。默认值为 `null`。 |
| `promiseLibrary` | object | 供应用程序使用的 ES6 兼容的 Promise 库。默认值为 `null`。Promise 是一个包装器函数，可与支持回调的 Node 函数一起使用，关于 Promise 的详细讨论超出了本章范围。 |

`connect()` 方法的 `callback` 参数类型为 `connectCallback(error, db)`，定义了结果的回调格式。`error` 参数类型为 `MongoError`，表示错误实例。`db` 参数类型为 `Db`，表示已连接的数据库。

### 连接步骤

1.  在 `C:\Program Files\nodejs\scripts` 目录下创建一个 `connection.js` 脚本。Node.js 中连接到 MongoDB 的唯一可用方法是 `connect()` 方法，该方法有静态版本和实例版本，并使用连接字符串。如前所列，连接字符串的语法如下：

    ```
    mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]
    ```

    连接 URL 的必需部分是 `mongodb://host1`。`port1` 默认为 `:27017`。如果不使用副本集（本章中我们不会使用），则不需要 `host2[:port2],...[,hostN[:portN]]` 段。我们也不会使用连接凭据 `username:password`。

2.  使用 `require` 语句从 `mongodb` 模块导入 `MongoClient` 类，如下所示：

    ```
    MongoClient = require('mongodb').MongoClient;
    ```

3.  创建一个 `MongoClient` 实例：

    ```
    var mongoclient = new MongoClient();
    ```

    接下来，我们将使用 `connect(url, options, callback)` 方法进行连接，其中第一个参数是连接字符串，第二个是选项，第三个是回调函数。回调函数在 `connect()` 方法完成后调用。回调函数的第一个参数是发生错误时的 `MongoError` 对象或 `null`，第二个参数是初始化的 `Db` 对象。

4.  使用 `connect()` 方法连接到 MongoDB 服务器，并在连接字符串中指定数据库为 `test`。在回调函数的方法块中，如果发生错误则将错误消息记录到控制台，否则记录一条消息表明连接已建立。

    ```
    mongoclient.connect("mongodb://localhost:27017/test", function(error, db) {
        if (error)
            console.log(error);
        else
            console.log('Connected with MongoDB');
    });
    ```

    静态方法 `MongoClient.connect()` 也可用于连接到 MongoDB 服务器，并支持与实例方法 `connect()` 相同的参数。

    `connection.js` 脚本如下：

    ```
    MongoClient = require('mongodb').MongoClient;
    var mongoclient = new MongoClient();
    mongoclient.connect("mongodb://localhost:27017/test", function(error, db) {
        if (error)
            console.log(error);
        else
            console.log('Connected with MongoDB');
    });
    ```

5.  打开一个新的终端/控制台，导航到 `C:\Program Files\nodejs`。使用以下命令在 Node.js 中运行 `connection.js` 脚本。


```
>node connection.js
```

如图 5-11 中的输出所示，`connect()`方法建立了与 MongoDB 的连接。

![9781484215999_Fig05-11.jpg](img/image00557.jpeg)

图 5-11. 连接 MongoDB

## 使用数据库

在本节中，我们将讨论 `Db` 类并创建一个 MongoDB 数据库实例。`Db` 类构造函数的语法如下。

```
Db(databaseName, topology, options)
```

构造函数参数在表 5-7 中讨论。

表 5-7. Db 构造函数参数

| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `databaseName` | string | 数据库名称。 |
| `topology` | Server &#124; ReplSet &#124; Mongos | 服务器的拓扑结构。 |
| `options` | object | 可选设置。 |

`Db` 类构造函数支持的选项在表 5-8 中讨论。

表 5-8. Db 选项

![image](img/image00558.jpeg)

`Db` 类提供的属性在表 5-9 中讨论。

表 5-9. Db 类属性

| 选项 | 类型 | 描述 |
| --- | --- | --- |
| `serverConfig` | Server &#124; ReplSet &#124; Mongos | 获取当前的拓扑结构。 |
| `bufferMaxEntries` | number | 获取当前的 `bufferMaxEntries` 值。 |
| `databaseName` | string | 数据库名称。 |
| `options` | object | 与关联实例相关的选项。 |
| `native_parser` | boolean | 获取参数 native_parser 的当前值。 |
| `slaveOk` | boolean | 获取 `slaveOk` 的当前值。 |
| `writeConcern` | object | 获取写关注的当前值。 |

`Db` 类支持的一些方法在表 5-10 中讨论。

表 5-10. Db 类方法

| 方法 | 返回类型 | 描述 |
| --- | --- | --- |
| `addUser(username, password, options, callback)` | 如果未指定回调则返回 Promise。 | 向数据库添加用户。 |
| `admin()` | Admin db。 | 返回 Admin 实例。 |
| `authenticate(username, password, options, callback)` | 如果未指定回调则返回 Promise。 | 验证用户身份。 |
| `close(force, callback)` | Object。 | 关闭数据库及底层连接。 |
| `collection(name, options, callback)` | Collection。 | 获取一个集合。 |
| `collections(callback)` | 如果未指定回调则返回 Promise。 | 获取当前数据库实例的所有集合。 |
| `command(command, options, callback)` | 如果未指定回调则返回 Promise。 | 执行命令。 |
| `createCollection(name, options, callback)` | 如果未指定回调则返回 Promise。 | 创建集合。 |
| `db(name, options)` | Db。 | 创建新的数据库实例。 |
| `dropCollection(name, callback)` | 如果未指定回调则返回 Promise。 | 删除集合。 |
| `dropDatabase(callback)` | 如果未指定回调则返回 Promise。 | 删除数据库。 |
| `executeDbAdminCommand(command, options, callback)` | 如果未指定回调则返回 Promise。 | 以管理员身份执行命令。 |
| `listCollections(filter, options)` | CommandCursor。 | 列出所有集合。 |
| `logout(options, callback)` | 如果未指定回调则返回 Promise。 | 从服务器注销用户。 |
| `open(callback)` | 如果未指定回调则返回 Promise。 | 打开数据库。 |
| `removeUser(username, options, callback)` | 如果未指定回调则返回 Promise。 | 从数据库中移除用户。 |
| `renameCollection(fromCollection, toCollection, options, callback)` | 如果未指定回调则返回 Promise。 | 重命名集合。 |

1.  在 `C:\Program Files\nodejs\scripts` 目录中创建一个 JavaScript 脚本 `db.js`。
2.  在脚本中，我们将获取一个数据库实例并从中获取数据库集合的列表。首先，使用 `require` 从 `mongodb` 模块导入 `MongoClient` 和 `Db` 类。

    ```
    MongoClient = require('mongodb').MongoClient;
    Db = require('mongodb').Db;
    ```

    无需直接使用类构造函数创建 `Db` 实例，它可以通过 `MongoClient connect()` 方法作为回调函数的第二个参数隐式创建。
3.  创建 `MongoClient` 实例。

    ```
    var mongoclient = new MongoClient();
    ```
4.  使用 `connect(url, options, callback)` 方法连接数据库。将 URI 指定为 `mongodb://localhost:27017/local`，其中包含 `local` 数据库实例。在回调函数的方法块中，如果发生错误则将错误消息记录到控制台；否则记录消息 `'Connected with MongoDB'`。

    ```
    mongoclient.connect("mongodb://localhost:27017/local", function(error, db) {
         if (error)
             console.log(error);
         else
             console.log('Connected with MongoDB');
      });
    ```

    我们将使用 `listCollections(filter, options)` 方法列出本地数据库中的集合。`listCollections(filter, options)` 可以选择性地指定一个过滤器，例如，过滤器 `{name: "collection1"}` 仅获取 `collection1` 集合。`listCollections()` 唯一支持的选项是 `batchSize`。`listCollections()` 方法返回一个 `CommandCursor`，它提供了几种方法，包括用于返回文档数组的 `toArray(callback)` 方法、用于遍历集合中文档的 `each(callback)` 方法以及用于获取下一个文档的 `next()` 方法。
5.  在 `connect()` 方法的回调函数的方法块中，输出集合列表。

    ```
    db.listCollections().toArray(function(err, items) {
            console.log(items);
            db.close();
          });
    ```

    `db.js` 脚本如下所示。

    ```
    MongoClient = require('mongodb').MongoClient;
    Db = require('mongodb').Db;
      var mongoclient = new MongoClient();
     mongoclient.connect("mongodb://localhost:27017/local", function(error, db) {
         if (error)
             console.log(error);
         else
             console.log('Connected with MongoDB');
    db.listCollections().toArray(function(err, items) {
            console.log(items);
            db.close();
          });
    });
    ```
6.  打开一个新的终端/窗口并导航到 `C:\Program Files\nodejs`。使用以下命令运行 `db.js` 脚本。

    ```
    >node db.js
    ```

本地数据库中不同的集合及其选项会被列出，如图 5-12 所示。

![9781484215999_Fig05-12.jpg](img/image00559.jpeg)

图 5-12. 列出本地数据库中的集合

`Db()` 类构造函数创建新的数据库实例，但不会创建新的数据库，我们接下来将演示这一点。

1.  使用相同的脚本 `db.js`，注释掉调用 `mongoclient.connect()` 方法的代码段。为 `Server` 类添加一个 `require` 语句。

    ```
    Server = require('mongodb').Server;
    ```
2.  使用类构造函数 `new Db(databaseName, topology, options)` 创建新的 `Db` 实例，数据库名为 `'mongo'`，拓扑结构使用 `Server` 类创建，主机为 `localhost`，端口为 27017。

    ```
    var db = new Db('mongo', new Server('localhost', 27017));
    ```
3.  使用 `open(callback)` 方法打开与数据库的连接。`open()` 方法的回调类型是 `openCallback(error, db)`。如果产生错误则记录。如果打开成功，使用返回的 `Db` 实例输出集合列表。

    ```
    db.
```


```javascript
open(function(error, db) {
  if (error)
    console.log(error);
  db.listCollections().toArray(function(error, items) {
    console.log(items);
    db.close();
  });
});
```

以下用于前述示例的`db.js`文件内容如下：

```javascript
Db = require('mongodb').Db;
Server = require('mongodb').Server;
var db = new Db('local', new Server('localhost', 27017));
db.open(function(error, db) {
  if (error)
    console.log(error);
  db.listCollections().toArray(function(error, items) {
    console.log(items);
    db.close();
  });
});
```

4.  运行`db.js`脚本以列出`mongodb`数据库中的集合，如图 5-13 所示。
    ![9781484215999_Fig05-13.jpg](img/image00560.jpeg)
    图 5-13. 使用类构造函数创建 Db 实例并运行 db.js 脚本

在上面的例子中，我们使用了`Server`拓扑。拓扑也可以是`Mongos`或`ReplSet`。例如，要使用`Mongos`拓扑，请遵循以下步骤：

1.  为`Mongos`添加一条`require`语句，并使用一个`Server`实例数组创建`Mongos`实例，如下所示。数组中的`Server`实例并不全是必须是正在运行且可用的服务器。例如，我们列出了一个端口为 50000 的服务器，但在运行示例时，该端口上并没有运行 MongoDB 服务器。

    ```javascript
    var mongos = new Mongos([
        new Server("localhost", 50000),
        new Server("localhost", 27017)
      ]);
    ```

2.  使用`Db`类构造函数创建`Db`实例，数据库名作为第一个参数，`Mongos`类实例作为第二个参数。

    ```javascript
    var db = new Db('local', mongos);
    ```

3.  使用`open()`方法打开数据库，并在回调函数块中像之前一样列出集合。

    用于`Mongos`类型拓扑的`db.js`脚本内容如下：

    ```javascript
    Mongos = require('mongodb').Mongos;
    Db = require('mongodb').Db;
    Server = require('mongodb').Server;
    var mongos = new Mongos([
        new Server("localhost", 50000),
        new Server("localhost", 27017)
      ]);
    var db = new Db('local', mongos);
    db.open(function(error, db) {
      if (error)
        console.log(error);
      db.listCollections().toArray(function(error, items) {
        console.log(items);
        db.close();
      });
    });
    ```

4.  运行`db.js`脚本以列出`mongodb`数据库中的集合，如图 5-14 所示。
    ![9781484215999_Fig05-14.jpg](img/image00561.jpeg)
    图 5-14. 使用 Mongos 拓扑运行 db.js 脚本

## 使用集合

集合由`Collection`类对象表示。`Collection`类的构造函数不接受任何参数，可以使用`new Collection()`创建一个新的`Collection`实例。`Collection`类提供了表 5-11 中讨论的属性。

表 5-11. 集合类属性
| 属性 | 类型 | 描述 |
| --- | --- | --- |
| `collectionName` | 字符串 | 获取集合名称。 |
| `namespace` | 字符串 | 获取完全限定的集合命名空间。 |
| `writeConcern` | 对象 | 获取当前的写关注值。 |
| `hint` | 对象 | 获取当前的索引提示。 |

`Collection`构造函数并非旨在直接调用，而是在内部调用。`Collection`类提供的一些方法在表 5-12 中讨论。

表 5-12. 部分集合类方法
| 方法 | 返回类型 | 描述 |
| --- | --- | --- |
| `bulkWrite(operations, options, callback)` | 如果未指定回调，则为 Promise。 | 执行批量写操作。 |
| `count(query, options, callback)` | 如果未指定回调，则为 Promise。 | 计算匹配文档的数量。 |
| `createIndex(fieldOrSpec, options, callback)` | 如果未指定回调，则为 Promise。 | 创建索引。 |
| `deleteMany(filter, options, callback)` | 如果未指定回调，则为 Promise。 | 删除多个文档。 |
| `deleteOne(filter, options, callback)` | 如果未指定回调，则为 Promise。 | 删除单个文档。 |
| `distinct(key, query, options, callback)` | 如果未指定回调，则为 Promise。 | 获取给定键的不同文档。 |
| `drop(callback)` | 如果未指定回调，则为 Promise。 | 删除集合。 |
| `dropIndex(indexName, options, callback)` | 如果未指定回调，则为 Promise。 | 删除索引。 |
| `find(query)` | 游标。 | 查找文档。在结果集上创建游标。 |
| `findAndModify(query, sort, doc, options, callback)` | 如果未指定回调，则为 Promise。 | 查找并更新文档。该方法已弃用，其替代方法是`findOneAndUpdate`、`findOneAndReplace`或`findOneAndDelete`。仍可使用已弃用的方法。 |
| `findAndRemove(query, sort, options, callback)` | 如果未指定回调，则为 Promise。 | 查找并删除文档。该方法已弃用，其替代方法是`findOneAndDelete`。仍可使用已弃用的方法。 |
| `findOne(query, options, callback)` | 如果未指定回调，则为 Promise。 | 查找单个文档。 |
| `findOneAndDelete(filter, options, callback)` | 如果未指定回调，则为 Promise。 | 查找单个文档并将其删除。 |
| `findOneAndReplace(filter, replacement, options, callback)` | 如果未指定回调，则为 Promise。 | 查找单个文档并替换它。 |
| `findOneAndUpdate(filter, update, options, callback)` | 如果未指定回调，则为 Promise。 | 查找单个文档并更新它。 |
| `indexes(callback)` | 如果未指定回调，则为 Promise。 | 返回所有索引。 |
| `insertMany(docs, options, callback)` | 如果未指定回调，则为 Promise。 | 插入文档数组。 |
| `insertOne(doc, options, callback)` | 如果未指定回调，则为 Promise。 | 插入单个文档。 |
| `isCapped(callback)` | 如果未指定回调，则为 Promise。 | 检查集合是否为固定集合。 |
| `listIndexes(options)` | CommandCursor。 | 列出所有索引。 |
| `mapReduce(map, reduce, options, callback)` | 如果未指定回调，则为 Promise。 | 在集合上运行 Map Reduce。 |
| `options(callback)` | 如果未指定回调，则为 Promise。 | 返回集合的选项集。 |
| `parallelCollectionScan(options, callback)` | 如果未指定回调，则为 Promise。 | 返回集合上的 n 个并行游标。 |
| `rename(newName, options, callback)` | 如果未指定回调，则为 Promise。 | 重命名集合。 |
| `replaceOne(filter, doc, options, callback)` | 如果未指定回调，则为 Promise。 | 替换单个文档。 |
| `save(doc, options, callback)` | 如果未指定回调，则为 Promise。 | 将文档保存为完整文档替换。该方法已弃用，其替代方法是`insertOne`、`insertMany`、`updateOne`或`updateMany`。 |
| `stats(options, callback)` | 如果未指定回调，则为 Promise。 | 获取所有集合统计信息。 |
| `updateMany(filter, update, options, callback)` | 如果未指定回调，则为 Promise。 | 更新多个文档。 |
| `updateOne(filter, update, options, callback)` | 如果未指定回调，则为 Promise。 | 更新单个文档。 |

`Db`类提供了几个与集合相关的方法（`collection`、`collections`、`createCollection`、`dropCollection`、`listCollections`、`renameCollection`），如表 5-10 中所讨论。在本节中，我们将使用其中一些方法来获取、创建、重命名。


## 集合操作

本节将介绍如何在 MongoDB 中创建、重命名和删除集合，以及输出集合相关信息。我们在前一节已经使用了 `listCollections()` 方法。

1.  在 `C:\Program Files\nodejs\scripts` 目录下创建一个 JavaScript 脚本 `collection.js`。
2.  导入所需的类 `MongoClient`、`Server` 和 `Db`；根据 `Db` 实例的创建方式，可能会部分或全部用到这些类来创建和访问集合。
    ```javascript
    MongoClient = require('mongodb').MongoClient;
    Server = require('mongodb').Server;
    Db = require('mongodb').Db;
    ```
3.  为本地数据库创建一个 `Db` 实例，如之前所述。
    ```javascript
    var db = new Db('local', new Server('localhost', 27017));
    db.open(function(error, db) {
        if (error)
            console.log(error);
    });
    ```
4.  在 `open()` 方法的回调函数块中，使用 `db` 实例调用 `collections(callback)` 方法。在 `collections(callback)` 方法的回调函数块中，输出集合信息。
    ```javascript
    db.collections(function(error, collections){
        if (error)
            console.log(error);
        else{
            console.log("Collections in database local");
            console.log(collections);
        }
    });
    ```
5.  类似地，在 `open()` 方法的回调函数块中使用 `db` 实例，调用 `collection(name, options, callback)` 方法来获取 `mongodb` 集合实例。在 `collection(name, options, callback)` 方法的回调函数块中，输出有关集合的信息，例如集合名称、集合是否为 capped 集合以及集合中的文档数量。
    ```javascript
    db.collection('mongodb', function(error, collection){
        if (error)
            console.log(error);
        else{
            console.log("Got collection from local database, collection name: "+collection.collectionName);
            collection.isCapped(function(error, result){
                console.log("Is collection capped?: " +result);
            });
            collection.count(function(error, result){
                console.log("Document count in the collection: "+result);
            });
        }
    });
    ```
6.  接下来，使用 `createCollection(name, options, callback)` 方法创建一个名为 `mongo` 的集合。在回调函数中输出集合名称。
    ```javascript
    db.createCollection('mongo', function(error, collection){
        if (error)
            console.log(error);
        else{
            console.log("Collection created. Collection name: "+collection.collectionName);
        }
    });
    ```
7.  使用 `renameCollection(fromCollection, toCollection, options, callback)` 方法将 `mongodb` 集合重命名为 `mongocoll`。输出回调函数中返回的集合名称。
    ```javascript
    db.renameCollection('mongodb', 'mongocoll', function(error, collection){
        if (error)
            console.log(error);
        else{
            console.log("Collection renamed: "+collection.collectionName);
        }
    });
    ```
8.  使用 `dropCollection(name, callback)` 方法删除 `mongocoll` 集合。
    ```javascript
    db.dropCollection('mongocoll', function(error, result){
        if (error)
            console.log(error);
        else{
            console.log("Collection mongocoll dropped: "+result);
        }
    });
    ```

`collection.js` 脚本如下所示。
```javascript
MongoClient = require('mongodb').MongoClient;
Server = require('mongodb').Server;
Db = require('mongodb').Db;
var db = new Db('local', new Server('localhost', 27017));
db.open(function(error, db) {
    if (error)
        console.log(error);
    db.collections(function(error, collections){
        if (error)
            console.log(error);
        else{
            console.log("Collections in database local");
            console.log(collections);
        }
    });
    db.collection('mongodb', function(error, collection){
        if (error)
            console.log(error);
        else{
            console.log("Got collection from local database, collection name: "+collection.collectionName);
            collection.isCapped(function(error, result){
                console.log("Is collection capped?: " +result);
            });
            collection.count(function(error, result){
                console.log("Document count in the collection: "+result);
            });
        }
    });
    db.createCollection('mongo', function(error, collection){
        if (error)
            console.log(error);
        else{
            console.log("Collection created. Collection name: "+collection.collectionName);
        }
    });
    db.renameCollection('mongodb', 'mongocoll', function(error, collection){
        if (error)
            console.log(error);
        else{
            console.log("Collection renamed: "+collection.collectionName);
        }
    });
    db.dropCollection('mongocoll', function(error, result){
        if (error)
            console.log(error);
        else{
            console.log("Collection mongocoll dropped: "+result);
        }
    });
});
```

9.  在运行 `collection.js` 脚本之前，请在 Mongo shell 中的本地数据库中创建一个名为 `mongodb` 的集合，如图 5-15 所示。如果 `local` 数据库已有 `mongodb` 集合，则无需创建。
    ![9781484215999_Fig05-15.jpg](img/image00562.jpeg)
    **图 5-15** 创建 mongodb 集合

10.  使用以下命令运行 `collection.js` 脚本。
    ```bash
    >node collection.js
    ```
    脚本输出如图 5-16 所示。脚本会持续运行，要停止脚本请按 `Ctrl + C`。
    ![9781484215999_Fig05-16.jpg](img/image00563.jpeg)
    **图 5-16** collection.js 脚本输出

我们从 `mongodb` 集合开始。随后我们创建了一个名为 `mongo` 的集合。我们将 `mongodb` 集合重命名为 `mongocoll`，然后删除了 `mongocoll` 集合。如图 5-17 所示，`local` 数据库中除了 `startup_log` 和 `system.indexes` 集合外，唯一的集合是 `mongo` 集合。
![9781484215999_Fig05-17.jpg](img/image00564.jpeg)
**图 5-17** 运行 collection.js 脚本后本地数据库中的集合列表

## 使用文档

在以下小节中，我们将添加文档、批量添加文档、查询文档、更新文档、删除文档以及对文档执行批量操作。

### 添加单个文档

在本节中，我们将向 MongoDB 集合添加一个文档。

1.  首先，在 Mongo shell 中的 `local` 数据库中创建一个名为 `catalog` 的集合。
    ```bash
    >db.createCollection('catalog')
    ```
    在 Mongo shell 中使用 `show collections` 命令列出已创建的 `catalog` 集合，如图 5-18 所示。
    ![9781484215999_Fig05-18.jpg](img/image00565.jpeg)
    **图 5-18** 创建 catalog 集合
2.  在 `C:\Program Files\nodejs\scripts` 目录中创建一个 `addDocument.js` 脚本。`Collection` 类提供了 `insertOne(doc, options, callback)` 方法来添加单个文档。
3.  在 `addDocument.js` 脚本中，为 `Db`、`Server` 和 `Collection` 类添加 `require` 语句。
    ```javascript
    Server = require('mongodb').Server;
    Db = require('mongodb').Db;
    Collection = require('mongodb').Collection;
    ```
    `insertOne` 文档的参数在表 5-13 中讨论。
    **表 5-13** insertOne 方法的参数



## `insertOne()` 方法参数

| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `doc` | object | 要插入的文档。 |
| `options` | object | 方法选项。支持的选项包括 `w` (写入关注)、`wtimeout` (写入关注超时时间)、`j` (布尔值，指示是否启用日志写入关注)、`serializeFunctions` (布尔值，指示是否序列化任何对象上的函数) 和 `forceServerObjectId` (布尔值，指示是否由服务器分配 `_id` 值而不是由驱动程序分配)。 |
| `callback` | `insertWriteOpCallback(error, result)` | 结果回调函数。 |

### 添加单个文档

4.  使用类构造函数创建一个 `Db` 实例。

    ```
    var db = new Db('local', new Server('localhost', 27017));
    ```

5.  使用 `open()` 方法打开数据库。

    ```
    db.open(function(error, db) {
        //获取集合目录
    });
    ```

6.  在结果回调函数中，调用 `Db` 实例的 `collection()` 方法以获取 `catalog` 集合。

    ```
    db.collection('catalog', function(error, collection){
        if (error)
            console.log(error);
        else{
            //使用 insertOne 添加文档
        }
    });
    ```

7.  创建要添加文档的 JSON 数据。

    ```
    doc1 = {"catalogId" : 'catalog1', "journal" : 'Oracle Magazine', "publisher" : 'Oracle Publishing', "edition" : 'November December 2013',"title" : 'Engineering as a Service',"author" : 'David A. Kelly'};
    ```

8.  使用 `insertOne()` 方法添加文档。

    ```
    collection.insertOne(doc1, function(error, result){
        if (error)
            console.log(error);
        else{
            console.log("文档已添加: "+result);
        }
    });
    ```

    `addDocument.js` 的完整代码如下。

    ```
    Server = require('mongodb').Server;
    Db = require('mongodb').Db;
    Collection = require('mongodb').Collection;
    var db = new Db('local', new Server('localhost', 27017));
    db.open(function(error, db) {
        if (error)
            console.log(error);
        else{
            db.collection('catalog', function(error, collection){
                if (error)
                    console.log(error);
                else{
                    doc1 = {"catalogId" : 'catalog1', "journal" : 'Oracle Magazine', "publisher" : 'Oracle Publishing', "edition" : 'November December 2013',"title" : 'Engineering as a Service',"author" : 'David A. Kelly'};
                    collection.insertOne(doc1, function(error, result){
                        if (error)
                            console.log(error);
                        else{
                            console.log("文档已添加: "+result);
                        }
                    });
                }
            });
        }
    });
    ```

9.  使用以下命令运行 `addDocument.js` 脚本。

    ```
    >node addDocument.js
    ```

    一个文档会被添加。脚本的输出如 图 5-19 所示。

    ![9781484215999_Fig05-19.jpg](img/image00566.jpeg)

    图 5-19。`addDocument.js` 的输出

10. 在 mongo shell 中运行以下命令来列出已添加的文档。

    ```
    >use local
    >db.catalog.find()
    ```

    如 图 5-20 所示，添加的文档被列出。

    ![9781484215999_Fig05-20.jpg](img/image00567.jpeg)

    图 5-20。列出已添加的文档

### 添加多个文档

在本节中，我们将向 MongoDB 集合添加多个文档。

1.  在 `local` 数据库中删除 `catalog` 集合并重新创建它，因为我们将使用同一个集合来添加多个文档。

    ```
    >use local
    >db.catalog.drop()
    >db.createCollection('catalog')
    ```

    `Collection` 类提供了 `insertMany(docs, options, callback)` 方法来添加多个文档。

    该方法的参数在 表 5-14 中讨论。

## `insertMany` 方法参数

| 参数 | 类型 | 描述 |
| --- | --- | --- |
| `docs` | Array <object> | 要插入的文档数组。 |
| `options` | object | 方法选项。支持的选项包括 `w` (写入关注)、`wtimeout` (写入关注超时时间)、`j` (布尔值，指示是否启用日志写入关注)、`serializeFunctions` (布尔值，指示是否序列化任何对象上的函数) 和 `forceServerObjectId` (布尔值，指示是否由服务器分配 `_id` 值而不是由驱动程序分配)。 |
| `callback` | `insertWriteOpCallback(error, result)` | 结果回调函数。 |

2.  创建一个脚本 `addDocuments.js` 来添加文档。导入 `Server`、`Db` 和 `Collection` 类。

    ```
    Server = require('mongodb').Server;
    Db = require('mongodb').Db;
    Collection = require('mongodb').Collection;
    ```

3.  如前一节关于添加单个文档所述，获取 `catalog` 集合的 `Collection` 实例。创建要添加的两个文档的 JSON 数据。

    ```
    doc1 = {"catalogId" : 'catalog1', "journal" : 'Oracle Magazine', "publisher" : 'Oracle Publishing', "edition" : 'November December 2013',"title" : 'Engineering as a Service',"author" : 'David A. Kelly'};
    doc2 = {"catalogId" : 'catalog2', "journal" : 'Oracle Magazine', "publisher" : 'Oracle Publishing', "edition" : 'November December 2013',"title" : 'Quintessential and Collaborative',"author" : 'Tom Haunert'};
    ```

4.  使用 `Collection` 类的 `insertMany()` 实例方法，添加一个文档数组。

    ```
    collection.insertMany([doc1,doc2], function(error, result){
        if (error)
            console.log(error);
        else{
            console.log("文档已添加: "+result);
        }
    });
    ```

    `addDocuments.js` 脚本的完整代码如下。

    ```
    Server = require('mongodb').Server;
    Db = require('mongodb').Db;
    Collection = require('mongodb').Collection;
    var db = new Db('local', new Server('localhost', 27017));
    db.open(function(error, db) {
        if (error)
            console.log(error);
        else{
            db.collection('catalog', function(error, collection){
                if (error)
                    console.log(error);
                else{
                    doc1 = {"catalogId" : 'catalog1', "journal" : 'Oracle Magazine', "publisher" : 'Oracle Publishing', "edition" : 'November December 2013',"title" : 'Engineering as a Service',"author" : 'David A. Kelly'};
                    doc2 = {"catalogId" : 'catalog2', "journal" : 'Oracle Magazine', "publisher" : 'Oracle Publishing', "edition" : 'November December 2013',"title" : 'Quintessential and Collaborative',"author" : 'Tom Haunert'};
                    collection.insertMany([doc1,doc2], function(error, result){
                        if (error)
                            console.log(error);
                        else{
                            console.log("文档已添加: "+result);
                        }
                    });
                }
            });
        }
    });
    ```

5.  删除并重新创建 `catalog` 集合，然后运行脚本。

    ```
    > db.catalog.drop()
    >db.createCollection('catalog')
    >node addDocuments.js
    ```

    图 5-21 中的输出显示有两个文档被添加。

    ![9781484215999_Fig05-21.jpg](img/image00568.jpeg)

    图 5-21。`addDocuments.js` 的输出

    在 Mongo shell 中运行 `db.catalog.find()` 命令以列出已添加的文档，如 图 5-22 所示。

    ![9781484215999_Fig05-22.jpg](img/image00569.jpeg)

    图 5-22。



