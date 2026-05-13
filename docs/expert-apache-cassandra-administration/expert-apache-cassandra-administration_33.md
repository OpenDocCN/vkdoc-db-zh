# 一些 JVM 通过 JMX 访问时可能会填满其堆，参见 CASSANDRA-6541
-XX:+CMSClassUnloadingEnabled
```

如果你想将垃圾收集器从默认的 CMS 收集器切换到更新的 G1 收集器，首先必须注释掉`CMS Settings`部分下的所有内容。接着，编辑 G1 设置并取消注释`-XX:+UseG1GC`属性，如下所示：

## G1 垃圾收集器设置（实验性功能，注释上一节并取消注释下节以启用）

```
## 使用 Hotspot 的垃圾优先收集器。
-XX:+UseG1GC                        => 默认情况下，此属性被注释掉
#
```

以下是可以配置的 G1 设置：

*   `-XX:MaxGCPauseMillis`：
    这是关键的 G1 垃圾收集器可调优属性。暂停目标越低，吞吐量就越低。`MaxGCPauseMillis`属性的最低设置是 200 毫秒，这也是该属性的默认值。通过将值从 200 毫秒提高到 1000 毫秒，可以增加吞吐量。你必须尝试将此参数的值保持小于`cassandra.yaml`文件中的超时时间。

    ```
    ## 让 JVM 在 STW（停止世界）暂停期间做更少的记忆集工作，
    ## 而是倾向于并发 GC。降低了 p99.9 延迟。
    -XX:G1RSetUpdatingPauseTimePercent=5
    #
    ```

*   `-XX:InitiatingHeapOccupancyPercent`：
    设置此参数可以节省大型 Java 堆（大于 16GB）中的 CPU 时间。JVM 会暂停扫描内存区域以检查它们是否已满，直到堆达到设定的百分比满为止。例如，如果你将堆占用百分比设置为 70%，这意味着当堆达到 70%满时，JVM 开始扫描内存区域。此参数的默认值为 40%。

我没有展示任何 CMS 收集器的配置属性，因为它即将被淘汰。

## 设置堆大小

你可以通过设置`jvm.options`文件中`HEAP SETTINGS`部分下列出的属性来配置 Java 堆大小。
当你使用 G1 垃圾收集器时，你需要用`-Xmx`属性配置`MAX_HEAP_SIZE`属性。
你用`-Xms`属性设置最小堆大小，用`-Xmx`属性设置最大堆大小。

默认情况下，`-Xms`和`-Xmx`属性被注释，Cassandra 使用以下简单的启发式方法自动计算`MAX_HEAP_SIZE`属性的值：

```
max(min(1/2 ram, 1024MB), min(1/4 ram, 8GB))
```

也就是说，它估算一半的 RAM 并将其上限设为 1024MB。它还计算 RAM 的四分之一值并将其上限设为 8GB。然后它取两者中较高的值。

在生产数据库中，最佳实践是为`-Xmx`和`-Xms`属性设置自定义值。设置堆的最大值时，`MAX_HEAP_SIZE`属性的推荐值是尽可能设置得高，最高可达 64GB。

另一个（可能是必要的）最佳实践是将最小和最大堆大小设置为相同的值，如下所示：

```
-Xms16G
-Xmx16G
```

将最小和最大堆大小设置为相同的值有助于数据库避免在 JVM 通过移除陈旧对象清理堆区域空间时发生长时间的垃圾收集暂停（也称为“世界停止”暂停）。相反，JVM 在服务器启动时锁定你为堆指定的内存，以防止部分内存被交换到磁盘。

## 配置垃圾收集日志记录

最佳实践是启用 GC 日志记录，默认情况下它是启用的。你可以查看 GC 日志以了解节点堆使用情况的大小，这有助于你根据情况调整它。以下是`java.options`文件中的相关部分：

```
### GC 日志记录选项 -- 取消注释以启用
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintHeapAtGC
-XX:+PrintTenuringDistribution
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintPromotionFailure
#-XX:PrintFLSStatistics=1
#-Xloggc:/var/log/cassandra/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=10M
```

如你所见，这里显示的属性帮助你配置垃圾收集日志记录的各个方面，例如日志位置、日志文件轮换、日志文件大小以及要存储的日志文件数量。

## 使用 nodetool proxyhistograms 和 tablehistograms 命令

有两个很好的`nodetool`命令可以帮助你识别集群中的性能问题：`nodetool proxyhistograms`和`nodetool tablehistograms`。这两个命令都以直方图的形式呈现数据库捕获的性能统计信息，因此得名。

`nodetool proxyhistograms`命令有助于识别性能问题。此命令的输出显示集群中的读、写和范围请求延迟，其中节点充当协调者。此命令帮助你识别集群中的一般性能问题。

`nodetool tablehistograms`命令让你专注于特定表的性能。通过此命令，你可以查看表的写和读延迟以及分区大小。

## 总结

本章介绍了几个关键的 Cassandra 性能调优策略，例如压缩和压紧。性能调优通常涉及权衡取舍；你可以通过压缩节省存储，但必须为此付出更高的 I/O 要求。同样，增加 Java 堆大小将减少查询延迟，但会减少系统中可用于其他用途的内存。

`cassandra-stress`工具简单易用，在测试潜在的配置更改（如压紧策略、压缩等）的影响时非常有用。压力测试有助于你评估集群处理不同类型工作负载的能力，并帮助你判断是否需要向集群添加更多节点。

# Cassandra 安全

保护 Cassandra 集群涉及一系列不同的任务，包括认证、授权和加密。本章描述如何管理以下与安全相关的任务：

*   **认证**：如何允许应用程序和用户登录集群。
*   **授权**：处理授予访问数据库或数据库对象（如表和物化视图）的权限。
*   **加密**：指使用安全套接字层（SSL）来保护客户端与 Cassandra 数据库之间以及集群节点之间的通信。
*   **防火墙**：管理防火墙端口访问涉及了解你必须保持打开的端口。

### 配置认证

在大多数数据库中，通常使用“用户”和“角色”这两个术语来指代不同的实体。用户是登录账户，角色封装了你分配给用户的各种对象的权限集合。

Cassandra 将所有认证和授权都基于角色。你执行`CREATE ROLE`命令来创建数据库角色。

> **注意**
>
> 所有授权和认证都通过数据库角色进行。尽管 Cassandra 仍然提供`CREATE USER`、`ALTER USER`、`DROP USER`和`LIST USERS`命令，但你不应使用它们，因为它们在当前版本中已弃用。请改用`CREATE ROLE`、`ALTER ROLE`、`DROP ROLE`、`LIST ROLES`和`LIST_PERMISSIONS`命令。

Cassandra 仍然提供`CREATE USER`命令来创建新的数据库用户账户，但此命令已弃用，仅用于向后兼容目的。

Cassandra 不再正式使用“用户”一词来指代登录账户。因此，一个（登录）角色是用户的同义词。



# Cassandra 认证与角色管理

默认情况下，Cassandra 并不要求用户进行身份验证即可登录集群。这意味着，你可以直接输入 `cqlsh` 而无需任何凭据，即可访问集群。

### 创建角色

Cassandra 内置了一个名为 `cassandra` 的角色，其密码同样是 `cassandra`。这个默认角色 `cassandra` 拥有管理员权限。你也可以创建额外的登录角色并授予它们管理员权限。最佳实践是创建替代的管理员角色，更改默认角色 `cassandra` 的密码，并且之后不再使用这个默认角色进行任何操作。

只有通过拥有管理权限的角色（可以是默认角色 `cassandra` 或你创建的任何其他角色）登录，才能创建新角色。

当你首次以默认角色 `cassandra` 登录并尝试创建角色时，可能会遇到以下错误：

```
$ cqlsh 192.168.159.129 -u cassandra -p cassandra
Connected to Test Cluster at 192.168.159.129:9042.
[cqlsh 5.0.1 | Cassandra 3.10 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cassandra@cqlsh> create role newrole with password 'newrole123';
SyntaxException: line 1:20 mismatched input 'withpassword' expecting EOF (create user newuser [withpassword]...)
cassandra@cqlsh> create role newrole with password 'newrole123';
InvalidRequest: Error from server: code=2200 [Invalid query] message="org.apache.cassandra.auth.CassandraRoleManager doesn't support PASSWORD"
cassandra@cqlsh> list users;
Unauthorized: Error from server: code=2100 [Unauthorized] message="You have to be logged in and not anonymous to perform this request"
cassandra@cqlsh>
```

出现此错误的原因是你尚未在此数据库中配置身份验证。默认情况下，`cassandra.yaml` 文件中的身份验证选项被设置为：

```
authenticator: AllowAllAuthenticator
```

`AllowAllAuthenticator` 是默认的身份验证后端。如果你将 `AllowAllAuthenticator` 设置为 `authenticator` 属性的值，就会在数据库中禁用身份验证。Cassandra 不会执行任何检查，并允许任何人无需验证即可登录。

`AllowAllAuthenticator` 的替代方案是 `PasswordAuthenticator`。此身份验证后端将使用存储在数据库 `system_auth.credentials` 表中的凭据来验证用户。

因此，即使你以默认角色 `cassandra`（拥有管理员权限）登录数据库，当你尝试创建角色时，数据库会提示“你必须已登录且不能是匿名用户才能执行此请求”。`authenticator: AllowAllAuthenticator` 选项允许所有用户，并且不检查你是否已登录。解决方案是为数据库配置身份验证，如下一节所述。

## 配置身份验证

为了能够以管理员身份登录并创建角色或执行其他与角色相关的任务，请按照以下步骤配置身份验证。

1.  将 `cassandra.yaml` 文件中的身份验证选项更改为 `PasswordAuthenticator`，如下所示：
    ```
    authenticator: PasswordAuthenticator
    ```
    此更改会强制 Cassandra 在客户端连接到集群时要求提供角色名和密码。

2.  重启数据库。

3.  使用默认超级用户 `cassandra` 的凭据登录 `cqlsh`。
    ```
    $ cqlsh -u cassandra -p cassandra
    ```

4.  系统键空间 `system_auth` 存储角色凭据。如果此键空间只有默认的单个副本，一旦该副本变得不可用，你可能会失去对集群的访问权限。因此，为了增强可用性，将 `system_auth` 表空间的复制因子提高到一个更高的值，例如每个数据中心 3 或 5，如下所示：
    ```
    cqlsh> ALTER KEYSPACE "system_auth"
    WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'dc1' : 5,
    'dc2' : 3};
    ```

5.  为确保复制因子的更改在整个集群中生效，请对 `system_auth` 键空间运行 `nodetool repair` 命令。
    ```
    $ nodetool repair system_auth
    ```

6.  重启数据库。

至此，你已经为数据库配置了身份验证，现在可以以 `cassandra/cassandra` 登录来执行与角色和权限相关的管理任务。

一旦按照上述步骤配置了身份验证，你必须始终指定超级用户的凭据。如果尝试像之前那样不带凭据登录，将会收到错误。

```
$ cqlsh 192.168.159.129
Connection error: ('Unable to connect to any servers', {'192.168.159.129': AuthenticationFailed('Remote end requires authentication.',)})
$
```

## 加速凭据认证过程

在繁忙的数据库中，你可以通过配置 `cassandra.yaml` 文件中的以下两个选项来加速凭据认证过程：

*   `credentials_validity_in_ms`
*   `credentials_update_interval_in_ms`

这两个参数的默认值均为 2000 毫秒。你可以提高其中一个或两个参数的值，以减少频繁认证凭据请求的开销。

### 创建角色

配置好身份验证后，就可以使用超级用户帐户创建角色了。这里我们使用默认的超级用户帐户 `cassandra` 来创建用户。

通过运行 `CREATE ROLE` 命令来创建角色。`CREATE ROLE` 命令的通用语法如下：

```
(CREATE | ALTER | DROP ) role_name
[WITH (LOGIN = true | SUPERUSER = true | password = 'password')];
```

以下是对关键角色属性的简要说明。默认情况下，数据库将 `SUPERUSER` 和 `LOGIN` 属性设置为 `false`，而 `password` 属性默认为 `null`。

*   `SUPERUSER`：可以执行所有 CQL 命令。如果将此属性设置为 `true`，则该角色将被授予对所有角色的 `AUTHORIZE`、`CREATE` 和 `DROP` 权限。
*   `LOGIN`：如果设置，数据库允许此角色使用密码登录以运行 CQL 语句。你需要设置此属性以为 `PasswordAuthenticator` 后端或内部身份验证创建登录帐户。
*   `PASSWORD`：表示你设置的密码。Cassandra 的内部身份验证需要密码。你必须用单引号将密码括起来。

你可以创建角色来定义一组权限，然后将这些权限分配给其他角色并映射到外部用户。角色还使你能够为内部身份验证创建登录帐户。使用内部身份验证时的最佳实践是为分配权限和登录帐户创建单独的角色。

以下是创建角色的步骤。

1.  以超级用户 `cassandra` 身份登录 `cqlsh`。
    ```
    $ cqlsh 192.168.159.129 -u cassandra -p cassandra
    Connected to Test Cluster at 192.168.159.129:9042.
    [cqlsh 5.0.1 | Cassandra 3.10 | CQL spec 3.4.4 | Native protocol v4]
    Use HELP for help.
    ```

2.  执行 `CREATE ROLE <rolename> WITH PASSWORD <password>` 命令，如下所示：
    ```
    cassandra@cqlsh> create user 'test' with password 'test123';
    ```
    `list roles` 命令显示数据库中的所有角色。它还显示哪些角色拥有超级用户权限。例如：
    ```
    cassandra@cqlsh> list roles;
    name      | super
    -----------+-------
    cassandra |  True
    test | False
    (2 rows)
    cassandra@cqlsh>
    ```
    在此情况下，默认角色 `cassandra` 的超级用户列值为 `True`，而新角色 `test` 的超级用户列设置为 `False`。因此，一切正常。

默认情况下，`CREATE ROLE` 语句中的 `LOGIN` 属性值为 `False`。当你创建登录角色时，必须将此属性设置为 `True`。

你可以通过查询 `system_auth.roles` 表来查看数据库中的角色，如下所示：


# Cassandra 角色与权限管理

```
cassandra@cqlsh> select * from system_auth.roles;
role          | can_login | is_superuser | member_of         | salted_hash
---------------+-----------+--------------+-------------------+
cassadnra1 |      True |         True |              null | $2a$10$.b52eNZqZdouserevzYgpuecaxpdmc/QRdIoUIeG73a6UDEsBIdae
manager |      True |        False | {'cycling_admin'} | $2a$10$vkujDzBblNqLUSGLdhW26OIW/ias9Vh6JA3sU4pq9uXES30cK735.
test |      True |        False |              null | $2a$10$ukMpmnzdHn8xy/7krucrdeYtblF8XHmVTvv1qKldVrGLxvLTWBxaC
test1 |      True |        False |              null | $2a$10$3CQTIi09zOOT5v2SaCxH9ekE8ZTaAGZSG6owkziZIT3ylzMDMCrhW
cassandra |      True |         True |              null | $2a$10$nRxFfMOAqxVQYkFQmEMIZ.B1.9.opcND/8.LtwyqZLgJY91eiS3Zm
cycling_admin |     False |        False |              null |   null
(6 rows)
cassandra@cqlsh>
```

`system_auth.roles` 表存储与角色相关的信息，例如角色名称、该角色是否是超级用户，以及是否可以使用该角色登录数据库。除了此表，Cassandra 还将角色相关信息存储在以下表中：

*   `system_auth.role_members`：存储角色和角色成员。
*   `system_auth.role_permissions`：存储角色、资源以及角色访问该资源的权限。
*   `system_auth.resource_role_permissons_index`：存储角色以及授予这些角色的权限。

## 修改密码

您可以使用 `ALTER ROLE` 命令修改角色的密码：

```
cassandra@cqlsh> alter role 'newsuper' with password 'newsuper2';
```

## 删除角色

您可以使用 `DROP ROLE` 命令删除角色：

```
cassandra@cqlsh> drop role test;
```

### 处理超级用户账户

除了默认的超级用户账户 `cassandra`，您还可以在创建用户时通过指定 `superuser` 选项来创建额外的超级用户账户，如下所示：

```
cassandra@cqlsh> create role 'newsuper' with password 'newsuper1'
... superuser;
cassandra@cqlsh> list roles;
name      | super
-----------+-------
cassandra |  True
newsuper |  True
test | False
(3 rows)
cassandra@cqlsh>
```

处理默认超级用户账户 `cassandra` 的最佳实践是创建一个自定义管理员账户，之后不再使用默认的 `cassandra` 账户。创建一个具有 `superuser` 选项的用户。由于默认情况下超级用户只能管理角色，因此请使用以下命令授予该用户对所有密钥空间的访问权限：

```
cqlsh> grant all permissions on all keyspaces to newsuper;
```

### 配置授权：授予资源权限

配置认证将限制对集群的访问。配置授权是您控制对各种数据库对象（如密钥空间和表）访问的方式。您可以执行 `GRANT` 命令，将角色在数据库资源（如密钥空间、表或函数）上的权限授予该角色。

让我们学习如何向登录角色授予权限，首先授予对象（如密钥空间和表）上的特定权限。之后，我将展示如何向角色授予权限，这使您能够实现基于角色的访问控制（RBAC）系统。

> 注意
>
> 默认情况下，Cassandra 不会强制执行任何限制用户执行数据库操作的能力。

在以下示例中，您首先尝试将密钥空间上的 `SELECT` 权限授予一个角色：

```
cassandra@cqlsh> grant select permission on keyspace "cycling" to 'test';
ServerError: java.lang.UnsupportedOperationException: GRANT operation is not supported by AllowAllAuthorizer
cassandra@cqlsh>
```

这个 `GRANT` 命令失败是因为默认的授权器（恰好是 `AllowAllAuthorizer`）不支持 `GRANT` 操作。在 `cassandra.yaml` 文件中，您会看到以下内容：

```
authorizer: AllowAllAuthorizer
```

`AllowAllAuthorizer` 是 Cassandra 的默认授权后端，它控制访问并提供权限。使用此默认授权后端将禁用授权，这意味着 Cassandra 允许任何用户在数据库中执行任何操作。

`CassandraAuthorizer` 是替代的授权后端。它将所有权限存储在 `system_auth.permissions` 表中。要配置授权，请编辑 `cassandra.yaml` 文件并指定 `CassandraAuthorizer` 作为授权器，如下所示：

```
authorizer: org.apache.cassandra.auth.CassandraAuthorizer
```

如上所示设置授权器将在数据库中强制执行授权。数据库现在将根据登录数据库的角色来限制访问。一旦您正确配置了 `authorizer` 属性，就可以成功发出 `GRANT` 命令。

```
cassandra@cqlsh> grant select permission on keyspace "cycling" to 'test';
```

由于丢失对 `system_auth.permissions` 表的访问权限将拒绝所有对集群的访问，请确保您有系统密钥空间的多个副本。如果您有多个数据中心，请将复制类设置为 `NetworkTopologyStrategy`。

您可以向角色授予以下广泛的权限类型：

*   `SELECT`：允许角色使用 CQL 命令 `SELECT` 读取数据。
*   `MODIFY`：允许角色使用 CQL 命令 `INSERT`、`UPDATE`、`DELETE` 和 `TRUNCATE` 添加、修改和删除数据。
*   `CREATE`：授予使用命令 `CREATE KEYSPACE` 和 `CREATE TABLE` 创建密钥空间和表的能力。
*   `ALTER`：授予使用以下命令修改密钥空间和表的能力：
    *   `ALTER KEYSPACE`
    *   `ALTER TABLE`
*   `DESCRIBE`：提供有关对象的信息。
*   `DROP`：使您能够从数据库中删除对象。
*   `EXECUTE`：使用任何函数，以及在 `CREATE AGGREGATE` 语句中的任何函数中执行 `SELECT`、`INSERT` 和 `UPDATE` 权限。
*   `AUTHORIZE`：启用对密钥空间、表、函数和角色的权限授予和撤销。
*   `ALL PERMISSIONS`：启用对表的所有类型的查询。

以下是您可以发出 `GRANT` 命令将各种权限直接分配给角色的示例。

您可以授予角色 `sam` 在所有密钥空间的所有表上执行 `SELECT` 操作的权限，如下所示：

```
cqlsh> grant select on all keyspaces to 'sam';
```

您可以授予角色 `sam` 在 `CYCLING` 密钥空间的所有表上执行修改（`INSERT`、`UPDATE`、`TRUNCATE` 和 `DELETE`）的权限，如下所示：

```
cqlsh> grant modify on keyspace cycling to 'sam';
```

您可以授予角色 `sam` `ALTER KEYSPACE` 权限，如下所示：

```
cqlsh> grant alter keyspace on cycling to 'sam';
```

`ALTER KEYSPACE` 权限使角色 `sam` 能够在 `CYCLING` 密钥空间及其表和索引上执行以下操作：

*   `ALTER KEYSPACE`
*   `ALTER TABLE`
*   `CREATE INDEX`
*   `DROP INDEX`

`ALL PERMISSIONS` 权限使用户能够在表上运行所有类型的查询：

```
cqlsh> grant all permissions on cycling.cyclists to sam;
```

### Cassandra 的访问控制矩阵

Cassandra 为访问控制采用分层继承系统，您授予层次结构中较高资源的权限会自动级联到层次结构中较低的资源。

例如，

*   如果您在 `ALL KEYSPACES` 上授予权限，它会级联到这些密钥空间中的所有表。
*   如果您在 `ALL FUNCTIONS` 上授予权限，它会级联到所有用户定义函数和聚合。


### 配置基于角色的访问控制

数据库角色是你授予访问其他数据库资源权限的载体。基于角色的访问控制是指将角色分配给用户，而不是直接分配访问各种数据库资源的权限。

通常，在其他数据库中，要使用角色来管理对数据库对象的访问，你必须按以下顺序操作：

*   创建用户。
*   创建角色。
*   向新角色授予对各种数据库资源的权限。
*   将该角色授予新用户。

你在 Cassandra 数据库中实现 RBAC 也遵循相同的策略，不同之处在于，你不是将角色授予用户，而是将它们授予（登录）角色（正如我之前解释的，这些角色在 Cassandra 中代表用户）。

在接下来的部分中，我将展示如何配置 RBAC。

### 为登录账户创建角色

当你想授予用户或应用程序访问权限时，你需要创建一个角色。以下示例展示了如何创建名为 `manager` 的登录角色：

```
cqlsh> create role manager
... with password = 'Password123'
... and LOGIN = true;
```

默认情况下，`LOGIN` 属性的值为 `false`，意味着该角色无法登录数据库。在此场景中，你希望 `manager` 角色能够登录数据库并作为登录角色使用。因此，你将 `LOGIN` 属性设置为 `true`。

`list roles` 命令显示新角色已成功创建。

```
cqlsh> list roles;
role      | super | login | options
-----------+-------+-------+---------
cassandra |  True |  True |        {}
    manager | False |  True |        {}
      test | False |  True |        {}
     test1 | False |  True |        {}
(4 rows)
cqlsh>
```

你可以测试新账户的登录能力。

```
$ cqlsh 192.168.159.129 -u cassandra -p cassandra
Connected to Test Cluster at 192.168.159.129:9042.
[cqlsh 5.0.1 | Cassandra 3.10 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cassandra@cqlsh> login manager
Password:
manager@cqlsh>
```

`cqlsh` 提示符（`manager@cqlsh`）显示了角色的名称，本例中为 `manager`。

你可以通过运行 `ALTER ROLE` 命令来更改已启用登录的角色的密码，如下所示：

```
cqlsh>ALTER ROLE manager WITH PASSWORD = 'manager';
cqlsh>
```

### 向角色授予权限

你也可以创建一个角色，并向该角色授予对一个或多个数据库对象的各种权限。然后，你可以将此角色分配给一个登录角色。

### Cassandra 中的对象权限

你可以向经过身份验证的角色授予以下数据库对象的对象权限：

*   键空间
*   表
*   函数
*   聚合
*   角色
*   MBeans

以下是你可以在数据库对象上配置的权限类型：

*   `CREATE`
*   `ALTER`
*   `DROP`
*   `SELECT`
*   `MODIFY`
*   `DESCRIBE`

### 向角色授予对象权限

以下示例展示了如何创建一个不用于登录的角色；而是使用该角色来授予对数据库对象的权限。你首先创建角色 `cycling_admin`，然后向该角色授予对键空间 `cycling` 的权限。接着，你将角色 `cycling_admin` 授予登录角色 `manager`。

1.  使用 `CREATE ROLE` 命令创建角色。

    ```
    cqlsh> create role cycling_admin;
    ```

    由于此角色用于分配对象权限而非登录账户，你没有设置 `SUPERUSER` 或 `LOGIN` 属性，这两者默认均为 `False`。

    ```
    cassandra@cqlsh> list roles;
    role         | super | login | options
    --------------+-------+-------+---------
    cassandra |  True |  True |        {}
    cycling_admin | False | False |        {}
    manager | False |  True |        {}
      test | False |  True |        {}
     test1 | False |  True |        {}
    (5 rows)
    cassandra@cqlsh>
    ```

2.  创建新角色后，向该角色授予对表空间（`cycling`）的权限。

    ```
    cqlsh> grant all permissions on keyspace cycling to cycling_admin;
    ```

3.  将新角色 `cycling_admin` 授予你在上一节中创建的登录角色 `manager`。

    ```
    cqlsh> grant cycling_admin to manager;
    ```

## 列出权限

你可以使用 `list all permissions` 命令列出授予角色的所有权限。

```
cassandra@cqlsh> list all permissions;
role          | username      | resource             | permission
---------------+---------------+----------------------+------------
    cassandra |     cassandra |                |      ALTER
    cassandra |     cassandra |                |       DROP
    cassandra |     cassandra |                |  AUTHORIZE
    cassandra |     cassandra |                    |      ALTER
    cassandra |     cassandra |                    |       DROP
    cassandra |     cassandra |                    |  AUTHORIZE
    cassandra |     cassandra |                       |      ALTER
    cassandra |     cassandra |                       |       DROP
    cassandra |     cassandra |                       |  AUTHORIZE
    cassandra |     cassandra |                      |      ALTER
    cassandra |     cassandra |                      |       DROP
    cassandra |     cassandra |                      |  AUTHORIZE
 cycling_admin | cycling_admin |                  |     CREATE
 cycling_admin | cycling_admin |                  |      ALTER
 cycling_admin | cycling_admin |                  |       DROP
 cycling_admin | cycling_admin |                  |     SELECT
 cycling_admin | cycling_admin |                  |     MODIFY
 cycling_admin | cycling_admin |                  |  AUTHORIZE
         test |          test |               |     SELECT
         test |          test |            |      ALTER
         test |          test |            |     SELECT
         test |          test |            |     MODIFY
(22 rows)
cassandra@cqlsh>
```

查看查询输出中的 `Resource` 列，了解 Cassandra 如何为数据库中的三个角色分配不同的权限：默认的超级用户角色 `cassandra`、登录角色 `test` 和你创建的新角色 `cycling_admin`。

### 查看授予角色的权限

要查看授予特定角色的权限，请运行 `list all permissions of <role_name>` 命令：

```
cassandra@cqlsh> list all permissions of manager;
role          | username      | resource           | permission
---------------+---------------+--------------------+------------
 cycling_admin | cycling_admin |                |     CREATE
 cycling_admin | cycling_admin |                |      ALTER
 cycling_admin | cycling_admin |                |       DROP
 cycling_admin | cycling_admin |                |     SELECT
 cycling_admin | cycling_admin |                |     MODIFY
 cycling_admin | cycling_admin |                |  AUTHORIZE
(6 rows)
cassandra@cqlsh>
```

### 配置访问的防火墙端口

Cassandra 节点需要开放多个防火墙端口才能进行通信。其中一些需要开放的端口是公共的，另一些则是节点间或 Cassandra 特定的端口。

确保以下端口开放：

| 端口类型             | 端口号 | 端口功能                                                             |
| -------------------- | ------ | -------------------------------------------------------------------- |
| 公共                 |        | SSH 端口                                                             |
| Cassandra（节点间）  |        | 用于节点间集群通信                                                   |
| Cassandra（节点间）  |        | 用于 SSL 节点间通信                                                  |
| Cassandra（节点间）  |        | JMX 监控端口                                                         |
| Cassandra（客户端）  |        | 客户端端口                                                           |
| Cassandra（客户端）  |        | 原生传输协议的默认端口（当你需要加密和非加密通信时）                 |

### 使用 SSL 加密 Cassandra



# SSL 加密配置指南

SSL 是一种安全协议，可在 Cassandra 客户端与节点之间以及节点之间的通信过程中对数据进行加密。每个参与通信的实体都必须拥有一个由该实体保管的私钥，以及一个与其他实体交换的公钥。服务器会向希望建立安全连接到 Cassandra 集群的客户端发送一个包含服务器公钥的证书。

> **注意**
>
> 除了连接数极高的情况，数据加密对性能的影响微乎其微。

SSL 加密可以通过加密节点间流动的数据来保护节点内部通信。您也可以设置客户端-节点 SSL 加密，以保护`cqlsh`或`nodetool`等客户端程序与集群节点之间传递的数据。

客户端通过让服务器使用其私钥验证安全证书来验证该证书。密钥库存储私钥和证书，而信任库存储公钥。系统可以使用证书颁发机构（CA），在这种情况下，信任库也会存储 CA 签名的证书。密钥库的密码称为`keypass`，信任库的密码称为`storepass`。

### 安装 Java 加密扩展文件

默认的`cassandra.yaml`文件引用了多个加密套件，其中一些仅在您使用 Oracle Java 并通过 SSL 连接时，安装了 Java 加密扩展（JCE）无限制权限策略文件后才可用。安装 JCE 文件可确保在使用 Oracle Java 时支持所有加密算法。您必须在集群的所有节点上安装这些文件。

> **注意**
>
> 当通过 SSL 使用 Oracle Java 时，安装 JCE 文件是一项最佳实践。

请按照以下步骤安装 JCE 文件。此处的步骤展示了如何在 Red Hat Linux 系统上安装 JCE。

1.  安装 EPEL 目录。
    ```
    $ sudo yum install epel-release
    ```
2.  从 Oracle Java SE 下载页面（[`www.oracle.com/technetwork/java/javase/downloads/index.html`](http://www.oracle.com/technetwork/java/javase/downloads/index.html)）下载 JCE 文件。
3.  解压下载的 JCE 文件，并将`local_policy.jar`和`US_export_policy.jar`文件复制到`$JAVA_HOME/jre/lib/security`目录。

在基于 Debian 的系统上安装 JCE 更为简单。
```
$ sudo apt-get install oracle-java8-unlimited-jce-policy
Reading package lists... Done
Building dependency tree
...
Unlimited JCE Policy for Oracle Java 8 installed
$
```

在学习如何通过 SSL 处理节点间和客户端-节点加密之前，您必须先通过准备 SSL 服务器证书来设置 SSL 加密。

### 准备服务器证书

对于节点间和客户端-节点加密，您都必须先生成 SSL 证书并验证这些证书。以下是开始使用 SSL 证书需要做的事项。

*   使用`openssl`和`keytool`实用工具生成 SSL 证书。
*   生成一个自签名的 CA 来验证 SSL 证书，如下文所示。您也可以让证书由受信任的公共证书颁发机构（如 Verisign）签名。

> **注意**
>
> Nate McCall 展示了如何使用基于 ccm 的 Cassandra 集群设置 SSL 节点间加密。文章位于[`http://thelastpickle.com/blog/2015/09/30/hardening-cassandra-step-by-step-part-1-server-to-server.html`](http://thelastpickle.com/blog/2015/09/30/hardening-cassandra-step-by-step-part-1-server-to-server.html)。

在本节中，我将展示如何为生产环境准备 SSL 证书。

### 创建证书颁发机构

使用 SSL 加密数据库的第一步是创建自己的 CA，用于签署所有特定于服务器的证书。这将有助于创建一个信任链，从而简化证书管理。请按照以下步骤创建 CA。

1.  第一步是创建根 CA 证书和密钥。执行`openssl req`命令并向其传递证书配置文件。这要求您首先创建一个证书配置文件，您将其命名为`my_rootCa_cert.conf`文件。打开 vi 或 nano 编辑器，并在`my_rootCa_cert.conf`文件中输入以下信息：
    ```
    $ vi my_rootCa_cert.conf
    # my_rootCa_cert.conf
    [ req ]
    distinguished_name    = req_distinguished_name
    prompt                = no
    output_password       = myPass
    default_bits          = 2048
    [ req_distinguished_name ]
    C                     = US
    O                     = MyCompany
    OU                    = TestCluster
    CN                    = rootCa
    $
    ```
2.  创建证书配置文件（`my_rootCa_cert.conf`）后，运行`openssl req`命令，如下所示，确保通过`config`属性传递配置文件名。
    ```
    $ sudo openssl req -config my_rootCa_cert.conf  -new -x509 -nodes -subj /CN=rootCa/OU=TestCluster/O=YourCompany/C=US/ -keyout rootCa.key -out rootCa.crt -days 365
    Generating a 2048 bit RSA private key
    ...................................+++
    ..............................................................+++
    writing new private key to 'rootCa.key'
    $
    ```
    此命令将创建`rootCa.key`和`rootCa.crt`文件，您将在后续步骤中使用它们。`rootCa.key`文件是您将写入密钥的文件，而`rootCa.cert`文件是您将写入证书的文件。
3.  使用`openssl x509`命令验证 rootCa 证书（`rootCa.crt`文件）。
    ```
    $ sudo openssl x509 -in rootCa.crt -text -noout
    Certificate:
    Data:
    Version: 1 (0x0)
    Serial Number: 12633404970202873071 (0xaf52dee2c098bcef)
    Signature Algorithm: sha256WithRSAEncryption
    Issuer: C=US, O=YourCompany, OU=TestCluster, CN=rootCa
    Validity
    Not Before: Aug 25 19:00:47 2017 GMT
    Not After : Aug 25 19:00:47 2018 GMT
    ...
    $
    ```

### 为所有节点创建证书

下一步是使用`keytool`实用工具生成公钥和私钥对。您必须在所有节点上执行此操作。您可以先在一个节点上生成证书，然后将它们复制到所有节点。
```
$ sudo keytool -genkeypair
-keyalg RSA -alias 192.168.159.129
-keystore 192.168.159.129.jks
-storepass myKeyPass
-keypass myKeyPass
-validity 365
-keysize 2048
-dname "CN=192.168.1159.129, OU=TestCluster, O=YourCompany, C=US"
$
```
在此示例中，测试集群中有两个节点，因此您需要在两个节点上都运行此命令。此处显示的`keytool`命令适用于 IP 地址为`192.168.159.129`的节点。对于集群中的第二个节点（IP 地址为`192.168.159.130`），您运行相同的命令，但将 IP 地址`192.168.159.129`替换为`192.168.159.130`。

在此命令中，
*   `-genkeypair`生成一个公钥/私钥对。
*   `-keystore`指定密钥库文件名（您使用 IP 地址以便将文件映射到 Cassandra 节点）。
*   `-storepass`指定密钥库密码。
*   `-keypass`指定私钥密码（必须与`-storepass`属性的值相同）。
*   `-dname`指定与别名值关联的可分辨名称（DN）。CN 值设置为节点的 IP 地址或 FQDN。

检查您生成的证书，以确保密钥库可访问且包含正确的密钥对。
```
$sudo keytool -list -keystore 192.168.159.129.jks -storepass myKeyPass
Keystore type: JKS
Keystore provider: SUN
Your keystore contains 1 entry
192.168.159.129, Aug 25, 2017, PrivateKeyEntry,
Certificate fingerprint (SHA1): C2:C1:6C:F5:DF:F7:EC:D6:09:66:BB:67:17:38:58:87:38:E1:AE:DD
$
```

## 导出证书签名请求



# 生成节点证书和密钥

生成节点证书和密钥后，必须为每个节点导出一个证书签名请求（CSR）。该 CSR 使用`rootCa`证书进行签名，以验证其是否受信任。

```bash
$ sudo  keytool -certreq \
-keystore 192.168.159.129.jks \
-alias 192.168.159.129 \
-file 192.168.159.129.csr \
-keypass myKeyPass -storepass myKeyPass \
-dname "CN=192.168.1159.129, OU=TestCluster, O=YourCompany, C=US"
$
```

`keytool -certreq`命令会导出一个用于 CA 签名的证书。

必须为集群中的每个节点重复此命令，将此处显示的 IP 地址（`192.168.159.129`）替换为其他节点的 IP 地址。

## 使用 CA 的公钥签署证书

接下来，为 Cassandra 集群中的每个节点，使用`rootCa`签署节点证书，通过 OpenSSL 完成。

```bash
$ sudo openssl x509 \
-req \
-CA rootCa.crt \
-CAkey rootCa.key \
-in 192.168.159.129.csr \
-out 192.168.159.129.crt_signed \
-days 365 \
-CAcreateserial -passin pass:myPass
Signature ok
subject=/C=US/O=YourCompany/OU=TestCluster/CN=192.168.1159.129
Getting CA Private Key
$
```

同样，必须为您的每个集群节点运行此处显示的`openssl`命令。

在此命令中：

*   `-CA`标识`rootCa`证书。
*   `-Cakey`标识`rootCa`密钥。
*   `-in`指定要读取证书的文件名。
*   `-out`指定签名证书的输出文件名。

使用`rootCa`证书和已签名的证书来验证签名证书。

```bash
$ openssl verify -CAfile rootCa.crt 192.168.159.129.crt_signed
192.168.159.129.crt_signed: OK
$
```

## 将 CA 添加到密钥库

将`rootCa`证书导入每个节点的密钥库。

```bash
$ sudo keytool -importcert \
-keystore 192.168.159.129.jks \
-alias rootCa \
-file rootCa.crt \
-noprompt \
-keypass myKeyPass \
-storepass myKeyPass
Certificate was added to keystore
$
```

此时密钥库文件将包含`rootCa`证书和节点证书的条目。

## 将签名的证书导入密钥库

节点上的每个密钥库现在都包含了 CA。下一步是执行与上一步相同的操作，但这次是为每个节点将节点的签名证书导入节点密钥库。

```bash
$ sudo keytool -importcert \
-keystore 192.168.159.129.jks \
-alias 192.168.159.129 \
-file 192.168.159.129.crt_signed \
-noprompt \
-keypass myKeyPass \
-storepass myKeyPass
Certificate reply was installed in keystore
$
```

之前创建的节点证书将被签名的节点证书替换。

## 构建信任库

接下来必须创建一个服务器信任库，通过验证来自其他节点的连接请求，在集群节点之间建立信任链。运行`keytool -importcert`命令来构建密钥库。

```bash
$ keytool -importcert \
-keystore generic-server-truststore.jks \
-alias rootCa \
-file rootCa.crt \
-noprompt \
-keypass myPass \
-storepass truststorePass
Certificate was added to keystore
$
```

创建信任库后，就可以在集群中共享它。该信任库表示，它将信任所有客户端证书由该 CA 签名的节点的连接。

您可以这样验证信任库文件：

```bash
$ sudo keytool -list -keystore generic-server-truststore.jks -storepass truststorePass
Keystore type: JKS
Keystore provider: SUN

Your keystore contains 1 entry

rootca, Aug 28, 2017, trustedCertEntry,
Certificate fingerprint (SHA1): BB:BF:D7:8F:15:4E:41:91:37:70:EB:4C:67:AB:2C:25:37:A4:18:B0
$
```

## 使用密钥库配置集群

之前，我展示了如何创建特定于节点的密钥库和一个通用信任库。现在必须将它们移动到 Cassandra 可以找到的位置。默认情况下，Cassandra 会在`$CASSANDRA_HOME/conf`目录中查找密钥库和信任库。

首先，为集群中的两个节点分别复制特定于节点的密钥库。

```bash
cp 192.168.192.159.129.jks  $CASSANDRA_HOME/conf/server-keystore.jks
cp 192.168.192.159.130.jks  $CASSANDRA_HOME/conf/server-keystore.jks
```

接下来，将通用信任库复制到两个节点。

```bash
cp generic-server-truststore.jks  $CASSANDRA_HOME/conf/server-truststore.jks
cp generic-server-truststore.jks  $CASSANDRA_HOME/conf/server-truststore.jks
```

至此，SSL 证书的配置完成。现在必须启用加密。

### 启用节点间加密

加密节点间通信的最后一步是通过修改`cassandra.yaml`文件来启用加密。在`cassandra.yaml`文件中，按如下方式修改`server_encryption_options`部分：

```yaml
server_encryption_options:
internode_encryption: all                /default value: none
keystore: /cassandra/conf/server-keystore.jks
keystore_password: cassandra
truststore: /cassandra/conf/server-truststore.jks
truststore_password: cassandra
protocol: TLS
algorithm: SunX509
store_type: JKS

