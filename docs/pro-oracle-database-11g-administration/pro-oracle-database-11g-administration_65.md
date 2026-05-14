# 检查归档目录是否已满

```
38 * * * * /u01/oracle/bin/arch_check.bsh DWREP
1>/u01/oracle/bin/log/arch_check.log 2>&1
```
> （注意：代码本应位于一行。本书中将其置于两行是为了适应页面排版。）

