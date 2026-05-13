# 用户资源限制

您必须在 `/etc/security/limits.conf` 文件中为各类用户资源设置以下限制：
```
 - memlock unlimited
 - nofile 100000
 - nproc 32768
 - as unlimited
```

此外，您还需要包含以下设置：
```
vm.max_map_count = 1048575
```

如果您使用的是基于 Red Hat 的 Linux 服务器，还必须在 `/etc/security/limits.d/90-nproc.conf` 文件中设置以下 `nproc` 限制：
```
cassandra_user  - nproc 32768
```

