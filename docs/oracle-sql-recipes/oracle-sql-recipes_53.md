# 第 18 章 ■ 对象管理

##### 18-9. 创建位图索引

### 问题

你已经在数据仓库中构建了一个星型模式。你希望确保连接事实表和维度表的 SQL 查询能高效地返回数据。

### 解决方案

在用于连接事实表和维度表的列上创建位图索引。位图索引通过使用 `CREATE INDEX` 语句的关键字 `BITMAP` 来创建。此示例在 `F_DOWNLOADS` 表的 `D_DATE_ID` 列上创建了一个位图索引：

```
create bitmap index f_down_date_fk1 on f_downloads(d_date_id);
```

如果你的事实表是分区的，你必须将位图索引创建为本地分区索引。通过指定 `LOCAL` 关键字来实现：

```
create bitmap index f_down_date_fk1 on f_downloads(d_date_id) local;
```

### 工作原理

位图索引在数据仓库环境中被广泛使用，通常用于事实表的外键列。位图索引不太适合具有多会话 DML 活动的 OLTP 环境。这是因为对于位图索引列，位图索引的内部结构通常要求在更新时锁定许多行。因此，如果有多个会话执行 `UPDATES`、`INSERTS` 和 `DELETES`，你将遇到锁和死锁问题。

数据仓库环境中的位图索引通常根据需要删除并重新创建。

通常，在加载大量数据之前，会删除位图索引，然后在表填充后重新创建。为了提高创建位图索引的性能，你可以通过 `NOLOGGING` 选项关闭重做日志的生成：

```
create bitmap index f_down_date_fk1 on f_downloads(d_date_id) local nologging;
```

由于索引可以轻松重建，因此通常没有理由为 `CREATE INDEX` 操作生成重做日志。

[www.it-ebooks.info](http://www.it-ebooks.info/)

