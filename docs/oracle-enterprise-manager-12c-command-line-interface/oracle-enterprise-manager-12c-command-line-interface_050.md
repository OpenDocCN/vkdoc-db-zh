# Source the OEM function library
. /script/oem/emcli_functions.lib

export BO_NAME=database_refresh_${ORASID}

CreateBlackout

< Existing your in-house database restoration commands >

EndBlackout
```

只需要一个 cron 条目（或 OEM 计划作业），因为你已将所有三个脚本的工作包装在一个包装器中。

此示例中引用的函数库包含函数 `CreateBlackout` 和 `EndBlackout`。函数库只是一个不可执行的文本文件，存储在每台服务器的中心位置，最好通过 NFS 挂载或类似的共享文件系统。

此代码段仅包含与此示例相关的两个函数。第 8 章 包含一个完整函数库的示例。你的生产 CLI 函数库可以包含许多其他函数，除了这两个。请注意，函数的内容与我们在单独的 Shell 脚本中使用的代码相同。它们只是被记录为函数。

```shell
