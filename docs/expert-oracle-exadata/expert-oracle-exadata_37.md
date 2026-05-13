# 扩展 Exadata 存储

向您现有的存储网格添加新单元是一个相当简单的过程。执行上述步骤将完成大部分工作。我们将看看剩余的过程，以便您了解所涉及的命令和文件。流程如下：

1.  执行`install.sh`中与配置单元警报相关的步骤。如果您希望手动完成，可以使用`cellcli`实现：
    ```
    ALTER CELL smtpServer='mail.example.com',
    smtpFromAddr='Exadata@example.com',
    smtpFrom='Exadata',
    smtpToAddr='all.dba@example.com,all.sa@example.com',
    notificationPolicy='critical,warning,clear',
    notificationMethod='mail'
    ```
    您可以使用`LIST CELL DETAIL`命令显示当前的单元配置。完成单元配置后，停止并重新启动单元服务以确保新设置生效。
2.  参照您的一台旧存储单元，使用`CREATE GRIDDISK`命令创建您的网格磁盘。我们在第 14 章中讨论了此命令的使用。请确保按正确的顺序创建网格磁盘，因为这会影响磁盘性能。您可以使用`LIST GRIDDISK DETAIL`命令的大小和偏移属性来确定网格磁盘的适当大小和创建顺序。通常，您应按此顺序创建网格磁盘：`DATA`、`RECO`，然后是`DBFS_DG`。
3.  更新所有计算节点（新旧）上的`/etc/oracle/cell/network-config/cellip.ora`文件，以反映所有单元的 InfiniBand IP 地址。
4.  将新的网格磁盘添加到您现有的 ASM 磁盘组中。这可以通过 ASM 中的 SQL*Plus 完成。以下示例显示了在半机架升级中将磁盘添加到`DATA`磁盘组（机架名为“dm01”）——对剩余的磁盘组重复此过程。
    ```
    SQL> ALTER DISKGROUP DATA ADD DISK
      2> 'o/*/DATA*dm01cel04*',
      3> 'o/*/DATA*dm01cel05*',
      4> 'o/*/DATA*dm01cel06*',
      5> 'o/*/DATA*dm01cel07*'
      6> rebalance power 32;
    ```

