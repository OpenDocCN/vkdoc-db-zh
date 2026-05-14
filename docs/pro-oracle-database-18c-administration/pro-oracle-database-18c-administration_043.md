# 压缩算法

基本压缩算法不需要 Oracle 的额外许可。如果您拥有高级压缩选项的许可，那么您还可以使用三个额外的可配置二进制压缩级别；例如：

```
RMAN> configure compression algorithm 'HIGH';
RMAN> configure compression algorithm 'MEDIUM';
RMAN> configure compression algorithm 'LOW';
```

根据我的经验，之前的压缩算法在压缩比和创建备份所需时间方面都非常高效。

您可以查询 `V$RMAN_COMPRESSION_ALGORITHM` 来查看您数据库版本可用的压缩算法详情。要将当前压缩算法重置为默认的 `BASIC`，请使用 `CLEAR` 命令：

```
RMAN> configure compression algorithm clear;
```

