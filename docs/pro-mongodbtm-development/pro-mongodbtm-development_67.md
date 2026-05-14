# 运行 `CREATE EXTERNAL TABLE` 命令

运行 `CREATE EXTERNAL TABLE` 命令以创建 Hive 表 `wlslog`，包含以下内容：
*   将列设置为 `TIME_STAMP`、`CATEGORY`、`TYPE`、`SERVERNAME`、`CODE` 和 `MSG`。
*   将 `STORED BY` 子句设置为 `'org.yong3.hive.mongo.MongoStorageHandler'`。
*   在 `WITH SERDEPROPERTIES` 子句中，将 `mongo.column.mapping` 属性设置为 MongoDB 文档集合中的列名。
*   在 `TBLPROPERTIES` 子句中，设置表 11-2 中所示的属性。

表 11-2. Hive 表的 `TBLPROPERTIES` 属性

| 属性 | 值 |
| --- | --- |
| `mongo.host` | `10.0.2.15` |
| `mongo.port` | `27017` |
| `mongo.db` | `test` |
| `mongo.user` | `hive` |
| `mongo.passwd` | `hive` |
| `mongo.collection` | `wlslog` |

在 Hive shell 中用于创建 Hive 外部表的命令是 `wlslog`。

```sql
hive>CREATE EXTERNAL TABLE wlslog (TIME_STAMP string, CATEGORY string, TYPE string, SERVERNAME string, CODE string, MSG string)
      STORED BY 'org.yong3.hive.mongo.MongoStorageHandler'
      WITH SERDEPROPERTIES ("mongo.column.mapping"="TIME_STAMP,CATEGORY,TYPE,SERVERNAME,CODE,MSG")
      TBLPROPERTIES ( "mongo.host" = "10.0.2.15", "mongo.port" = "27017",
      "mongo.db" = "test", "mongo.user" = "hive", "mongo.passwd" = "hive", "mongo.collection" = "wlslog" );
```

该命令的输出表明已创建一个 Hive 外部表。

