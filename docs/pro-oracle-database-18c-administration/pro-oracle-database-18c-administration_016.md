# login.sql

使用此脚本来定制 SQL*Plus 环境的各个方面。在 Linux/Unix 中登录 SQL*Plus 时，如果`login.sql`脚本存在于`SQLPATH`变量所包含的目录中，则会自动执行该脚本。如果未定义`SQLPATH`变量，则 SQL*Plus 会在调用 SQL*Plus 的当前工作目录中寻找`login.sql`。例如，以下是在我的环境中定义`SQLPATH`变量的方式：

```
$ echo $SQLPATH
/home/oracle/scripts
```

我在`/home/oracle/scripts`目录中创建了`login.sql`脚本。它包含以下内容：

```
-- set SQL prompt
SET SQLPROMPT '&_USER.@&_CONNECT_IDENTIFIER.> '
```

现在，当我登录到 SQL*Plus 时，我的提示符会被自动设置：

```
$ sqlplus / as sysdba
SYS@o12c>
```

