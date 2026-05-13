# ERP 设置——实用指南

本章旨在为读者提供设置 Oracle ERP（11i 或 R12）环境的指南。这里找到的大部分内容不一定特定于 Exadata，而是适用于任何 Linux RAC 环境。然而，Exadata 被描述为“盒子里的 RAC”，因此，挑战将始终存在于 Exadata 中。因此，这里似乎是解决这些问题的好地方。

Oracle 的 ERP 标准是每个数据库从自己的 Oracle Home 运行，并且该数据库的侦听器具有唯一的、专用的 SQLNet 端口。因此，我们的第一个挑战是为每个 ERP 数据库创建一个新的 Oracle Home。Exadata 通常附带一个准备就绪的 Oracle Home 和数据库。最简单的做法是简单地在每个节点上克隆该 Oracle Home。但是，如果您安装的数据库版本与 Exadata 机器附带的版本完全不同，您可以暂时跳过本节。但是，如果您发现将来必须克隆该 Home，不要感到惊讶。无论如何，克隆 Oracle Home 是 ERP DBA 的常见做法。

## 克隆 Oracle Home

除了 Grid Infrastructure Oracle Home（即您的 ASM 主目录）之外，您的 Exadata 集群将附带一个标准数据库 Oracle Home（在每个节点上）以及一个使用该 Home 创建的数据库。实际上，您应该将此 Home 和数据库保持原样——至少可以作为参考点。特别是在开发/测试集群上，您可能需要为各种目的（开发、测试、UAT 等）创建多个 Oracle Home。使用这个“种子” Oracle Home 作为您的第一个 ERP Oracle Home 的源，然后使用 ERP 肯定需要的额外数据库补丁对其进行修补。然后，您可以继续为可能需要的任何其他 ERP 环境克隆此 Home。

除非您已向 Oracle 支持另外指定，否则“种子”数据库很可能名为 DBM。在我们的示例中，其 Oracle Home 是 `/u01/app/oracle/product/11.2.0.3/dbhome_1`。我们的新数据库将称为 ERPDEV。以 `root` 身份，在集群中的所有节点上执行以下操作：

```
$ mkdir /u01/app/oracle/product/11.2.0.3/erpdevdb
$ cd /u01/app/oracle/product/11.2.0.3/dbhome_1
$ cp -r * /u01/app/oracle/product/11.2.0.3/erpdevdb/.
$ chown -R oracle:oinstall /u01/app/oracle/product/11.2.0.3/erpdevdb
```

以 Oracle 软件所有者身份登录，并在两个节点上执行以下操作——也在两个节点上。在每个节点上重复该过程时，请确保相应地设置 `LOCAL_NODE`，如下所示：

```
$ cd /u01/app/oracle/product/11.2.0.3/erpdevdb/clone/bin

$ perl clone.pl ORACLE_BASE=$ORACLE_BASE \
ORACLE_HOME=/u01/app/oracle/product/11.2.0.3/erpdevdb \
ORACLE_HOME_NAME=OraDb11g_ERPDEV \
'-O"CLUSTER_NODES={exadb01,exadb02}"' \
'-O"LOCAL_NODE=exadb01"'
```



