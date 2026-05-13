# cipher_suites: [TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA]
require_client_auth: true
require_endpoint_verification: false
```

之前，我介绍了如何在 JDK 中安装强加密策略文件（JCE 策略文件）。这支持使用强密码，例如 256 位 AES，正如`cassandra.yaml`文件中的`cipher_suites`属性所示。

现在所有加密设置都已完成。重启集群。

```bash
$ kill -9 cassandra-pid
$ $CASSANDRA_HOME/bin/cassandra
INFO  [main] 2017-08-28 12:33:50,572 MessagingService.java:687 - Starting Encrypted Messaging Service on SSL port 7001
```

消息“`Starting Encrypted Messaging Service on SSL port 7001`”表明节点间通信的 SSL 加密已生效。

### 启用客户端加密

在上一节中，我展示了如何配置节点到节点加密。然而，诸如`cqlsh`之类的客户端与集群之间的通信仍然是未加密的。在本节中，我将展示如何配置客户端到节点的加密。您使用为节点间通信创建的相同 SSL 证书。只需要启用客户端加密并将 CA 证书添加到`cqlshrc`文件中。

要启用客户端加密，请按如下方式修改`cassandra.yaml`文件：

```yaml
