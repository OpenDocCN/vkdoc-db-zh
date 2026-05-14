# 在 RMAN 中运行 SQL

从 Oracle 数据库 12c 开始，您可以直接在 RMAN 中运行 SQL 语句（并查看结果）：

```
RMAN> select * from v$rman_encryption_algorithms;
```

在 12c 之前，您可以使用 RMAN 的 `sql` 命令运行之前的 SQL 语句，但不会显示结果：

```
RMAN> sql 'select * from v$rman_encryption_algorithms';
```

RMAN 的 `sql` 命令更常用于运行诸如 `ALTER SYSTEM` 之类的命令：

```
RMAN> sql 'alter system switch logfile';
```

现在，在 12c 中，您可以直接运行 SQL：

```
RMAN> alter system switch logfile;
```

能够在 RMAN 中运行 SQL 是一个非常实用的增强；它允许您查看 SQL 查询的结果，并且不再需要指定 `sql` 关键字以及在 SQL 命令本身周围加上引号。

