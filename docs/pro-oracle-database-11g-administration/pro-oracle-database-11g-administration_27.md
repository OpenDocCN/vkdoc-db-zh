# 第一章 ■ 安装 Oracle 二进制文件

在这种情况下，Oracle 认为软件已经安装，原因有几个：响应文件中指定了 `ORACLE_HOME` 目录中的文件。

您 `oraInventory/ContentsXML/inventory.xml` 文件中的现有 Oracle 主目录和位置与您在响应文件中指定的相匹配。

Oracle 不允许您将一套新的二进制文件安装到现有的 Oracle 主目录之上。如果您确定不需要 `ORACLE_HOME` 目录中的任何文件，您可以删除它们（务必非常小心——确保您绝对想要这样做）。此示例导航到 `ORACLE_HOME`，然后删除 `dev_1` 目录及其内容：

```
$ cd $ORACLE_HOME
$ cd ..
$ rm -rf dev_1
```

此外，即使 `ORACLE_HOME` 目录中没有文件，安装程序也会检查 `inventory.xml` 文件以查找以前的 Oracle 主目录名称和位置。在 `inventory.xml` 文件中，您必须删除与您尝试安装到的 Oracle 主目录位置相对应的条目。

要删除该条目，首先定位您的 `oraInst.loc` 文件，该文件包含您的 `oraInventory` 的目录。导航到 `oraInventory/ContentsXML` 目录。在修改 `inventory.xml` 之前先制作一个副本：

`$ cp inventory.xml inventory.xml.old`

使用操作系统实用程序（例如 `vi`）编辑 `inventory.xml` 文件，并删除包含您先前失败安装的 Oracle 主目录信息的行。现在，您可以尝试再次执行 `runInstaller` 实用程序。

## 应用临时补丁

有时您需要应用补丁来解决数据库问题或修复错误。您通常从 MOS 网站获取补丁，并使用 `opatch` 实用程序进行安装。以下是应用补丁的基本步骤：

从 MOS 获取补丁（需要有效的支持合同）。

解压缩补丁文件。



