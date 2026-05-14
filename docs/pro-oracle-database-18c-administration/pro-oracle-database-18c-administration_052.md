# 设置 ECHO

另一个应在 RMAN 脚本中设置的值是 `ECHO` 命令，如下所示：

```
RMAN> set echo on;
```

这指示 RMAN 在其输出中显示正在运行的命令，因此你可以看到正在运行的 RMAN 命令，以及与该命令相关的任何错误或输出消息。当你在脚本中运行 RMAN 命令时，这一点尤其重要，因为你不是直接输入命令（并且可能不知道在 shell 脚本中发出了什么命令）。例如，未使用 `SET ECHO ON` 时，命令的输出显示如下：

```
Starting backup at...
```

使用 `SET ECHO ON` 后，此输出会显示实际运行的命令：

```
backup datafile 4;
Starting backup at...
```

从之前的输出中，你可以看到正在运行哪个命令、它何时开始等等。

