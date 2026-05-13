# 使复制更加安全

本节讨论几个项目，以使您的 GoldenGate 复制环境更加安全。首先探讨如何加密 GoldenGate 密码，然后研究如何加密 GoldenGate 轨迹文件。当然，启用这些功能是可选的，取决于您的具体安全要求。

#### 加密密码

在任何安全环境中，您都应该加密参数文件中的 GoldenGate 数据库用户密码。以下示例展示了如何加密密码 `abc` 并在参数文件中指定它。该示例使用了默认加密密钥，但如果需要，您可以使用特定密钥：

```
GGSCI (sourceserver) 1> encrypt password abc
未指定密钥，使用默认密钥...

加密后的密码：  AACAAAAAAAAAAADAVHTDKHHCSCPIKAFB
```

获得加密密码的值后，您可以用该值更新参数文件：

```
USERID 'GGER', PASSWORD "AACAAAAAAAAAAADAVHTDKHHCSCPIKAFB", ENCRYPTKEY default
```

除了加密密码，您还可以加密轨迹文件，接下来将看到这一点。

## 加密轨迹文件

在某些情况下，您可能需要加密存储在轨迹文件中的数据。这可以通过在参数文件中添加 `ENCRYPTTRAIL` 和 `DECRYPTTRAIL` 命令来实现。让我们看一个如何对轨迹文件使用加密的示例。

您可以加密本地 Extract 创建的源轨迹文件，如下所示。请注意，`ENCRYPTTRAIL` 参数必须位于要加密的轨迹文件之前——在本例中是 `l1` 轨迹：

```
ENCRYPTTRAIL
ExtTrail dirdat/l1
```

现在 `l1` 源轨迹文件已被加密。下一步是向数据泵 Extract 添加加密参数。首先，必须解密轨迹文件以便数据泵可以处理它，然后您可以将其重新加密为 `l2` 远程轨迹文件：

```
PassThru
DECRYPTTRAIL
ENCRYPTTRAIL
RmtHost targetserver, MgrPort 7809
RmtTrail dirdat/l2
```

请注意，即使使用了加密轨迹文件，您仍然可以为数据泵使用 passthru 模式。数据泵 Extract 通过网络传递加密的轨迹数据，并写入加密的远程轨迹。

最后一步是让 Replicat 解密轨迹文件以便处理，如以下示例所示：

```
DECRYPTTRAIL
Map HR.*, Target HR.* ;
```

让我们结合本节介绍的新增强安全更改，再看一下您的 Extract 和 Replicat 参数文件。首先是增强的 Extract 参数文件：

```
GGSCI (sourceserver) 1> edit params LHREMD1
Extract LHREMD1
-------------------------------------------------------------------
-- 针对 HR 模式的本地 Extract
-------------------------------------------------------------------
SETENV (NLS_LANG = AMERICAN_AMERICA.AL32UTF8)
USERID 'GGER', PASSWORD "AACAAAAAAAAAAADAVHTDKHHCSCPIKAFB", ENCRYPTKEY default
ReportCount Every 30 Minutes, Rate
Report at 01:00
ReportRollover at 01:15
DiscardFile dirrpt/LHREMD1.dsc, Append
DiscardRollover at 02:00 ON SUNDAY
ENCRYPTTRAIL
ExtTrail dirdat/l1
Table HR.*;
```

接下来是增强的 Replicat 参数文件：

```
GGSCI (targetserver) 1> edit params RHREMD1
Replicat RHREMD1
-------------------------------------------------------------------
-- 针对 HR 模式的 Replicat
-------------------------------------------------------------------
SETENV (NLS_LANG = AMERICAN_AMERICA.AL32UTF8)
USERID 'GGER', PASSWORD "AACAAAAAAAAAAADAVHTDKHHCSCPIKAFB", ENCRYPTKEY default
AssumeTargetDefs
ReportCount Every 30 Minutes, Rate
Report at 01:00
ReportRollover at 01:15
DiscardFile dirrpt/RHREMD1.dsc, Append
DiscardRollover at 02:00 ON SUNDAY
DECRYPTTRAIL
Map HR.*, Target HR.* ;
```

在下一节中，您将向复制环境添加专门的过滤和映射。

