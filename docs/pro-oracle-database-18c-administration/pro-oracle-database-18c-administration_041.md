# 14. 配置归档日志删除策略

在大多数情况下，我让 RMAN 根据数据库备份的保留策略删除归档日志。这是默认行为。你可以使用 `SHOW` 命令查看数据库保留策略：

```
RMAN> show retention policy;
CONFIGURE RETENTION POLICY TO REDUNDANCY 1; # default
```

要基于数据库保留策略删除归档日志（和备份片），请运行以下命令：

```
RMAN> delete obsolete;
```

从 Oracle Database 11g 开始，你可以指定一个独立于数据库备份的归档日志删除策略。此删除策略适用于 FRA（快速恢复区）之外和之内的归档日志。

> 注意
> 在 Oracle Database 11g 之前，归档删除策略仅适用于与备用数据库关联的归档日志。

要配置归档日志删除策略，请使用 `CONFIGURE ARCHIVELOG DELETION` 命令。以下命令配置归档重做日志删除策略，使得归档日志在被备份到磁盘两次之前不会被删除：

```
RMAN> configure archivelog deletion policy to backed up 2 times to device type disk;
```

要让 RMAN 根据归档日志删除策略删除过时的归档日志，请发出以下命令：

```
RMAN> delete archivelog all;
```

> 提示
> 在运行 `DELETE` 命令之前先运行 `CROSSCHECK` 命令。这样做可以确保 RMAN 知晓文件是否在磁盘上。

要查看是否为归档日志文件设置了特定的保留策略，请使用以下命令：

```
RMAN> show archivelog deletion policy ;
```

要清除归档删除策略，请执行以下操作：

```
RMAN> configure archivelog deletion policy clear;
```

