
# Apache Cassandra 高级管理指南

## Sam R. Alapati 精通 Apache Cassandra 管理

### 第一部分：简介、安装与配置

### 第二部分：数据模型、集群架构与 Cassandra 查询语言

### 第三部分：维护、监控、调优与保障 Apache Cassandra 安全

## 第一部分

### 1. `Apache Cassandra` 简介

### 最终一致性与 Cassandra

### 2. 安装 Cassandra 并开始使用 CQL Shell

#### 用户资源限制

#### PAM 安全设置

#### 设置 Java 堆大小

#### 禁用交换空间

#### 设置资源限制

#### Java 大页面设置

## 第二部分

### 4. Cassandra 数据建模以及数据的读写

#### Cassandra 一致性级别与配置

##### 数据中心一

##### 数据中心二

##### 指示此节点的机架和数据中心

```bash
docker run --name cassandra2 -m 2g -d -e CASSANDRA_SEEDS="172.17.0.3" cassandra:3.0.4
```

```bash
docker run --name cassandra3 -m 2g -d -e CASSANDRA_SEEDS="172.17.0.3" cassandra:3.0.4
```

```bash
sudo docker ps
```

```bash
sudo docker exec -i -t cassandra1 sh -c 'nodetool status'
```

```bash
docker run -it --link cassandra1 --rm cassandra:3.0.4 \
```

```bash
curl -L https://github.com/docker/compose/releases/download/1.14.0/docker-compose-'uname -s'-'uname -m' > /usr/local/bin/docker-compose
```

```bash
chmod +x /usr/local/bin/docker-compose
```

```bash
docker-compose
```

```bash
cucumber features/cassandra.feature
```

```bash
ls -altr
```

```bash
cp /cassandra/apache-cassandra-3.10/data/data/cycling/cyclist_name-39cd6de060de11e7805be14006afbdda/snapshots/truncated-1499189595147-cyclist_name/* .
```

```bash
./nodetool refresh cycling cyclist_name
```

```bash
/usr/lib/jvm/java-8-oracle/bin/jconsole
```

一些 JVM 通过 JMX 访问时可能会填满其堆，参见 `CASSANDRA-6541`

```yaml
cipher_suites: [TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_256_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA]
```

`enable or disable client/server encryption.`

`If enabled and optional is set to true encrypted and unencrypted connections are handled.`

`Set trustore and truststore_password if require_client_auth is true`
