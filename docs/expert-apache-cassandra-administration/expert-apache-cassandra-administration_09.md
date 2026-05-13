# PAM 安全设置

对于许多 Linux 版本，您必须启用 `pam_limits.so` 模块。这通过取消 `/etc/pam.d/su` 文件中以下行的注释来实现：
```
session  required  pam_limits.so
```

