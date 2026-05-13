# 第五章 与 CockroachDB 交互

让我们创建一个包含若干模式的数据库，看看如何利用模式设计来构建一个灵活的系统。首先，我们创建一个数据库：

```
CREATE DATABASE IF NOT EXISTS acme;

USE acme;
```

接下来，我们将创建一些用户来代表这三个业务领域中的人员：

```
CREATE USER IF NOT EXISTS retail_user;

CREATE USER IF NOT EXISTS manufacturing_user;

CREATE USER IF NOT EXISTS finance_user;
```

最后，我们将创建一些模式。以下语句创建了三个模式，每个业务领域一个。对于零售和制造模式，我们将授予财务用户访问权限，因为他们需要查看这两个模式中表的数据：

```
CREATE SCHEMA retail AUTHORIZATION retail_user;

GRANT USAGE ON SCHEMA retail TO finance_user;

CREATE SCHEMA manufacturing AUTHORIZATION manufacturing_user;

GRANT USAGE ON SCHEMA manufacturing TO finance_user;

CREATE SCHEMA finance AUTHORIZATION finance_user;
```

用户、模式和授权创建完成后，让我们使用 `SHOW GRANTS` 命令再次确认授权情况：

```
SELECT schema_name, grantee, privilege_type FROM [SHOW GRANTS]
WHERE schema_name IN ('retail', 'manufacturing', 'finance')
ORDER BY 1, 2, 3;
```

```
schema_name | grantee | privilege_type
----------------+--------------+-----------------
finance | admin | ALL
finance | root | ALL
manufacturing | admin | ALL
manufacturing | finance_user | USAGE
manufacturing | root | ALL
retail | admin | ALL
retail | finance_user | USAGE
retail | root | ALL
```

## 第五章 与 CockroachDB 交互

让我们以 `retail_user` 身份连接到数据库，并在 `retail` 模式中创建一张订单表。请注意，当我们以 `retail_user` 身份连接时，这是唯一一个我们能够执行此操作的模式：

```
$ cockroach sql \
--url "postgres://retail_user@localhost:26257/acme?sqlmode=disable" \
--insecure
```

```
CREATE TABLE retail.orders(
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
reference TEXT NOT NULL
);
```

然后以 `manufacturing_user` 身份做同样的操作：

```
$ cockroach sql \
--url "postgres://manufacturing_user@localhost:26257/acme?sqlmode=disable" \
--insecure
```

```
CREATE TABLE manufacturing.orders(
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
reference TEXT NOT NULL
);
```

最后，再次以 `finance_user` 身份执行相同操作：

```
$ cockroach sql \
--url "postgres://finance_user@localhost:26257/acme?sqlmode=disable" \
--insecure
```

```
CREATE TABLE finance.orders(
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
reference TEXT NOT NULL
);
```

## 第五章 与 CockroachDB 交互

让我们以 `root` 用户身份重新连接到数据库，并查看我们已创建的表：

```
$ cockroach sql \
--url "postgres://root@localhost:26257/acme?sqlmode=disable" \
--insecure
```

```
SELECT schema_name, table_name, owner
FROM [SHOW TABLES FROM acme];
```

```
schema_name | table_name | owner
----------------+------------+---------------------
finance | orders | finance_user
manufacturing | orders | manufacturing_user
retail | orders | retail_user
```

### 表设计

在任何数据库中创建表时，都有很多需要考虑的因素，CockroachDB 也不例外。其中，列名不当、数据类型使用错误、索引缺失（或冗余）以及主键配置不佳等问题，都会损害可读性和性能。

在本节中，我们将探讨一些表设计的最佳实践。

#### 命名

了解 CockroachDB 将在何处创建表非常重要，因此提供一个能明确指示 CockroachDB 将其放置在何处的名称至关重要。

你认为 CockroachDB 会将以下表放置在哪里？

```
CREATE TABLE table_name();
```

如果你回答任何类似 "¯\_(ツ)_/¯" 的变体，那也不算离谱。可能会产生以下结果：
• 该表存在于你当前所在的数据库的 public 模式中。



当前已选定。

- 该表位于 `defaultdb` 数据库的 `public` 模式中。
- 你会收到一个要求你选择数据库的错误提示。

