# 监听器配置与状态

你可以通过以下命令验证监听器正在监听的服务：

```bash
$ lsnrctl services
```

你可以使用以下查询来检查监听器的状态：

```bash
$ lsnrctl status
```

要获取监听器命令的完整列表，请执行此命令：

```bash
$ lsnrctl help
```

> **提示**
> 使用 Linux/Unix 的 `ps -ef | grep tns` 命令可以查看服务器上运行的任何监听器进程。

