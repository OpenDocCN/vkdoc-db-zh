# 第 5 步：创建数据字典

在数据库成功创建后，您可以通过运行两个脚本来实例化数据字典。这些脚本在您安装 Oracle 二进制文件时创建。您必须以 `SYS` 方案用户身份运行这些脚本：

```
SQL> show user
USER is "SYS"
```

在创建数据字典之前，我喜欢先生成一个输出文件，以便在出现意外错误时进行检查：

```
SQL> spool create_dd.lis
```

现在，创建数据字典：

```
SQL> @?/rdbms/admin/catalog.sql
SQL> @?/rdbms/admin/catproc.sql
```

成功创建数据字典后，以 `SYSTEM` 方案用户身份创建产品用户配置文件表：

```
SQL> connect system/
SQL> @?/sqlplus/admin/pupbld
```

这些表允许 SQL*Plus 按用户禁用命令。如果未运行 `pupbld.sql` 脚本，则所有非 `sys` 用户在登录 SQL*Plus 时会看到以下警告：

```
Error accessing PRODUCT_USER_PROFILE
Warning: Product user profile information not loaded!
You may need to run PUPBLD.SQL as SYSTEM
```

这些错误可以忽略。如果您不希望在登录 SQL*Plus 时看到它们，请确保运行了 `pupbld.sql` 脚本。
至此，您应该拥有一个功能齐全的数据库。接下来，您需要配置并实现监听器以支持远程连接，并可选择设置密码文件。这些任务将在接下来的两节中描述。

