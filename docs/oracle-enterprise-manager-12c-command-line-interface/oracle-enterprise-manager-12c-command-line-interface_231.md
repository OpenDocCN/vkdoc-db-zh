# --------------------------------------------------------------------------------------------------

DIRNAME=sysman/emd
   PrepareBackupDir

FILENAME=lastupld.xml
   BackupConfigFile

FILENAME=targets.xml
   BackupConfigFile
```



# 备份目录与配置文件

## 路径 `sysman/j2ee/applications/Oc4jDcmServlet/META-INF`

使用 `PrepareBackupDir` 命令准备目录。

| 文件名 |
| :--- |
| `application.xml` |
| `orion-application.xml` |
| `jazn-data.xml` |
| `principals.xml` |

使用 `BackupConfigFile` 命令备份上述文件。

## 路径 `sysman/j2ee/applications/Oc4jDcmServlet/Oc4jDcmServlet/WEB-INF`

使用 `PrepareBackupDir` 命令准备目录。

| 文件名 |
| :--- |
| `orion-web.xml` |
| `web.xml` |

使用 `BackupConfigFile` 命令备份上述文件。

## 路径 `sysman/j2ee/application-deployments/Oc4jDcmServlet`

使用 `PrepareBackupDir` 命令准备目录。

| 文件名 |
| :--- |
| `orion-application.xml` |
| `orion-web.xml` |

使用 `BackupConfigFile` 命令备份上述文件。

## 路径 `sysman/webapps/emd/online_help`

使用 `PrepareBackupDir` 命令准备目录。

| 文件名 |
| :--- |
| `ohwconfig.xml` |

使用 `BackupConfigFile` 命令备份该文件。

## 路径 `uix`

使用 `PrepareBackupDir` 命令准备目录。

| 文件名 |
| :--- |
| `uix.conf` |

使用 `BackupConfigFile` 命令备份该文件。

## 路径 `wcs/config`

使用 `PrepareBackupDir` 命令准备目录。

| 文件名 |
| :--- |
| `wcs_httpd.conf` |

使用 `BackupConfigFile` 命令备份该文件。

## 路径 `webcache`

使用 `PrepareBackupDir` 命令准备目录。

| 文件名 |
| :--- |
| `internal.xml` |

使用 `BackupConfigFile` 命令备份该文件。

---

执行 `DashedBreak` 命令。

```bash
echo "\n 处理后备份目录及其内容\n\n"
ls -lARp ${BACKUP_DIR} | grep -v "^total" 
```

## 临时文件清理脚本

Oracle Management Server 为其每个组件生成日志，从 WebLogic 服务器到主机上的 EM 代理。预设的轮转和文件清理机制在存在时非常有用，但并非所有文件都受此管理。保持服务器整洁需要由你负责。

此脚本遍历 OMS 服务器上的日志目录。评估每个文件的空闲时间，然后删除较旧的文件：

```ksh
#!/bin/ksh
#===============================================================================
