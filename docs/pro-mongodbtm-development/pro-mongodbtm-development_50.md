# 插入数据

接下来，我们将使用 `INSERT` 语句向 `catalog` 表添加数据。使用 `IF NOT EXISTS` 关键字有条件地添加行。当使用复合主键时，必须指定所有组件主键列，包括复合键列的值。

向 `CreateCassandraDatabase` 类添加一个方法 `insert()`，并在 `main()` 方法中调用该方法。

向 `catalog` 表添加两行，行 ID 分别为 `catalog1` 和 `catalog2`。

例如，两行数据被添加到 `catalog` 表中，如下所示。

```
session.execute("INSERT INTO datastax.catalog (catalog_id, journal, publisher, edition,title,author) VALUES ('catalog1','Oracle Magazine', 'Oracle Publishing', 'November-December 2013', 'Engineering as a Service','David A.  Kelly') IF NOT EXISTS");
session.execute("INSERT INTO datastax.catalog (catalog_id, journal, publisher, edition,title,author) VALUES ('catalog2','Oracle Magazine', 'Oracle Publishing', 'November-December 2013', 'Quintessential and Collaborative','Tom Haunert') IF NOT EXISTS");
```

