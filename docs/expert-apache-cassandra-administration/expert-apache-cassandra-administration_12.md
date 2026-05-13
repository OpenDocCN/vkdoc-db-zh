# 设置资源限制

通过设置 shell 限制来限制用户可以利用的集群资源。这可以通过编辑 `/etc/security/limits.conf` 文件来实现，该文件规定了用户如何使用资源的限制。您使用 `limits.conf` 文件来配置重要操作系统属性（如文件大小、堆栈大小和进程优先级（niceness））的“软”限制和“硬”限制。

在您的 `/etc/security/limits.conf` 文件中添加以下行：
```
soft nofile    32768
hard nofile    32768
hard nproc     32768
soft nproc     32768
```

`nofile` 属性限制每个用户进程的打开描述符数量，`nproc` 指定最大进程数。软限制设置意味着警告，而硬限制设置是实际的资源限制。

