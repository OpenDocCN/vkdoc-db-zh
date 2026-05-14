# 创建表（列族）

接下来，我们将创建一个列族，在 CQL 3 中也称为表。向 `CreateCassandraDatabase` 应用程序添加一个静态方法 `createTable()`。

如前所述，`CREATE TABLE` 命令也支持 `IF NOT EXISTS` 来有条件地创建表。CQL 3 增加了创建复合主键的功能，即由多个组件主键列创建的主键。在复合主键中，第一列称为分区键。

创建一个名为 `catalog` 的表，包含 `catalog_id`、`journal`、`publisher`、`edition`、`title` 和 `author` 列。在 `catalog` 表中，复合主键由 `catalog_id` 和 `journal` 列组成，其中 `catalog_id` 是分区键。

调用 `execute(String)` 方法来创建 `catalog` 表，如下所示。

```
session.execute("CREATE TABLE IF NOT EXISTS datastax.catalog (catalog_id text,journal text,publisher text, edition text,title text,author text,PRIMARY KEY (catalog_id, journal))");
```

在表名前加上键空间名。在 `main` 方法中调用 `createTable()` 方法。当 `CreateCassandraDatabase` 应用程序运行时，`catalog` 表将被创建。

