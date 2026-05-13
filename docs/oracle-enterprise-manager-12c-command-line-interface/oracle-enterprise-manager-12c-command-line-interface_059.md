# =====================================================
ARGFILE01=/tmp/argfile_create_roles.txt
ARGFILE02=/tmp/argfile_create_users.txt
ARGFILE03=/tmp/argfile_grant_roles.txt

sqlplus -S ${SYSMAN_CONNECT} <<EOF 1>/dev/null
SET ECHO OFF
SET FEEDBACK OFF HEADING OFF LINES 250 PAGES 999

SPOOL ${ARGFILE01}

SELECT 'create_role -name="' || role_name || '" -description="' || description || '"'
FROM     sysman.gc_roles
WHERE  role_type= '1';

SPOOL OFF

SPOOL ${ARGFILE02}

SELECT   'create_user -name="' || user_name || '" -description="' || user_description || '" -password='||oracle||
"  -expired="||true||'"'
FROM    sysman.gc_users
WHERE  user_name NOT IN ('SYSMAN')
ORDER BY user_name;

SPOOL OFF

SPOOL ${ARGFILE03}

SELECT 'grant_roles -name="' || user_name || '" -roles="' || role_name || '"'
FROM     sysman.gc_user_roles;

SPOOL OFF
exit
EOF

emcli login -user=sysman -pass=${CONSOLE_PWD}
emcli -argfile ${ARGFILE01}
emcli -argfile ${ARGFILE02}
emcli -argfile ${ARGFILE03}
```

## 更改 EM 管理员引用

正如我们在上一节中提到的，您通过 OEM 创建的所有信息都存储在存储库数据库中。OEM 帐户的用户数据存储在 `sysman.mgmt_created_users` 表中。当然，您绝不应直接更改存储库中的数据，但您可以将其用作参考数据。

通过 EM 控制台更新单个管理员是简单直接的。但是，如果部门名称更改或一组管理员更换了地点怎么办？

您可以创建一个简短的脚本，通过将部门 200 中的所有员工列表收集到一个假脱机文件中，然后遍历假脱机文件中的每一行，为每个员工创建 `modify_user` 语句，从而在部门 200 中的所有人都被分配到部门 500 时更改引用。该脚本由 SQL 和 EM CLI 命令组合而成。目前没有 CLI 动词可以直接收集此信息，因此我们将从存储库中提取它。

为了清晰起见，已删除了提示用户输入时典型的许多错误检查例程：用户提供答案了吗？输入的部门编号是大写还是小写？它们是有效数字吗？可扩展且可靠的脚本必须始终包含这些验证。

```
#!/bin/bash
