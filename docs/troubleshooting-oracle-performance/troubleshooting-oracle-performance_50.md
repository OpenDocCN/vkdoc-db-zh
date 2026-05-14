# OCI

OCI 是一个低级的应用程序编程接口。因此，它提供了对游标生命周期的完全控制。例如，在下面实现测试用例 2 的代码片段中，请注意每一步都是显式编码的：

```
for (i=1 ; i<=10000 ; i++)
{
  OCIStmtPrepare2(svc, (OCIStmt **)&stm, err, sql, strlen(sql), NULL, 0,
                  OCI_NTV_SYNTAX, OCI_DEFAULT);
  OCIDefineByPos(stm, &def, err, 1, val, sizeof(val), SQLT_STR, 0, 0, 0,
                 OCI_DEFAULT);
  OCIBindByPos(stm, &bnd, err, 1, &i, sizeof(i), SQLT_INT, 0, 0, 0, 0, 0,
               OCI_DEFAULT);
  OCIStmtExecute(svc, stm, err, 0, 0, 0, 0, OCI_DEFAULT);
  if (r = OCIStmtFetch2(stm, err, 1, OCI_FETCH_NEXT, 0, OCI_DEFAULT) == OCI_SUCCESS)
  {
    // do something with data...
  }
  OCIStmtRelease(stm, err, NULL, 0, OCI_DEFAULT);
}
```

由于可以完全控制游标，因此也可以实现测试用例 3。以下代码片段是一个示例。请注意，准备游标的函数 (`OCIStmtPrepare2` 和 `OCIDefineByPos`) 以及关闭游标的函数 (`OCIStmtRelease`) 被放置在循环外部，以避免不必要的软解析。

```
OCIStmtPrepare2(svc, (OCIStmt **)&stm, err, sql, strlen(sql), NULL, 0,
                OCI_NTV_SYNTAX, OCI_DEFAULT);
OCIDefineByPos(stm, &def, err, 1, val, sizeof(val), SQLT_STR, 0, 0, 0, OCI_DEFAULT);
for (i=1 ; i<=10000 ; i++) {
  OCIBindByPos(stm, &bnd, err, 1, &i, sizeof(i), SQLT_INT, 0, 0, 0, 0, 0,
               OCI_DEFAULT);
  OCIStmtExecute(svc, stm, err, 0, 0, 0, 0, OCI_DEFAULT);
  if (r = OCIStmtFetch2(stm, err, 1, OCI_FETCH_NEXT, 0, OCI_DEFAULT) == OCI_SUCCESS)
  {
    // do something with data...
  }
}
OCIStmtRelease(stm, err, NULL, 0, OCI_DEFAULT);
```

OCI 不仅能够完全控制游标，还支持客户端语句缓存。要使用它，只需启用语句缓存并使用函数 `OCIStmtPrepare2` 和 `OCIStmtRelease`（如前面的示例所示）。当调用函数 `OCIStmtRelease` 时，游标会被添加到缓存中。然后，当通过函数 `OCIStmtPrepare2` 创建新游标时，会查询缓存以查找是否存在具有相同文本的 SQL 语句。存在不同的方法来启用语句缓存。但基本上，只需在会话打开或从池中检索时指定即可。例如，如果通过函数 `OCILogon2` 打开非池化会话，则需要指定值 `OCI_LOGON2_STMTCACHE` 作为模式。

```
OCILogon2(env, err, &svc, username, strlen(username), password, strlen(password),
dbname, strlen(dbname), OCI_LOGON2_STMTCACHE)
```

默认情况下，缓存的大小为 20。下面的代码片段展示了如何通过在服务上下文上设置属性 `OCI_ATTR_STMTCACHESIZE` 将其更改为 50。请注意，将此属性设置为 0 会禁用语句缓存。

```
ub4 size = 50;
OCIAttrSet(svc, OCI_HTYPE_SVCCTX, &size, 0, OCI_ATTR_STMTCACHESIZE, err);
```

本节提供的 C 代码示例摘自文件 `ParsingTest1.c`、`ParsingTest2.c` 和 `ParsingTest3.c`，它们分别实现了测试用例 1、2 和 3。

