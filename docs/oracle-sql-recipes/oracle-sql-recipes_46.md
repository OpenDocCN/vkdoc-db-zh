# 第 17 章 ■ 数据库管理

执行以下 SQL 语句查看数据库中`RESOURCE_LIMIT`的当前设置：
```sql
select name, value from v$parameter where name='resource_limit';
```

使用`CREATE PROFILE`命令创建资源限制配置文件，然后可将该配置文件分配给任意现有数据库用户。以下 SQL 语句创建了一个限制资源的配置文件（例如限制单个会话可消耗的 CPU 量）：
```sql
create profile user_profile_limit
limit
sessions_per_user 20
cpu_per_session 240000
logical_reads_per_session 1000000
connect_time 480
idle_time 120;
```

创建配置文件后，可将其分配给用户。在以下示例中，用户`HEERA`被分配了`USER_PROFILE_LIMIT`：
```sql
alter user heera profile user_profile_limit;
```

■ **注意** Oracle 建议您使用数据库资源管理器来管理数据库资源限制。但我们发现通过 SQL 实现的数据库配置文件是一种有效且简便的资源限制机制。

## 工作原理

Oracle 数据库配置文件主要用于两大目的：

*   设置资源限制
*   强制执行密码安全策略

创建用户时若未指定配置文件，系统将自动为新建用户分配`DEFAULT`配置文件。可通过`ALTER PROFILE`语句修改配置文件。以下示例将`DEFAULT`配置文件的`CPU_PER_SESSION`限制修改为 240000（单位：百分之一秒）：
```sql
alter profile default limit cpu_per_session 240000;
```

这将使用`DEFAULT`配置文件的用户 CPU 使用时间限制为 2400 秒。配置文件中可设置多种限制参数，表 17-1 描述了可通过配置文件限制的数据库资源设置。

**表 17-1. 数据库资源配置文件设置**

| 配置文件资源 | 含义 |
| :--- | :--- |
| `COMPOSITE_LIMIT` | 基于加权和算法的综合限制，涉及资源包括：`CPU_PER_SESSION`、`CONNECT_TIME`、`LOGICAL_READS_PER_SESSION`和`PRIVATE_SGA`。 |
| `CONNECT_TIME` | 连接时长（分钟）。 |
| `CPU_PER_CALL` | 每次调用的 CPU 时间限制（百分之一秒）。 |
| `CPU_PER_SESSION` | 每个会话的 CPU 时间限制（百分之一秒）。 |
| `IDLE_TIME` | 空闲时间（分钟）。 |
| `LOGICAL_READS_PER_CALL` | 每次调用读取的数据块数。 |
| `LOGICAL_READS_PER_SESSION` | 每个会话读取的数据块数。 |
| `PRIVATE_SGA` | 在共享池中消耗的空间量。 |
| `SESSIONS_PER_USER` | 并发会话数量。 |

在`CREATE USER`语句中，可指定除`DEFAULT`外的配置文件：
```sql
create user heera identified by foo profile user_profile_limit;
```

配置文件也用于强制执行密码安全策略（密码配置文件设置说明参见表 17-2）。例如，若希望修改`DEFAULT`配置文件，设置密码最长使用天数上限，以下代码将`DEFAULT`配置文件的`PASSWORD_LIFE_TIME`设为 300 天：
```sql
alter profile default limit password_life_time 300;
```

**表 17-2. 密码安全设置**

| 密码设置 | 描述 | 11g 默认值 | 10g 默认值 |
| :--- | :--- | :--- | :--- |
| `FAILED_LOGIN_ATTEMPTS` | 锁定模式前允许的失败登录尝试次数。 | 10 次 | 10 次 |
| `PASSWORD_GRACE_TIME` | 密码过期后所有者仍可使用旧密码登录的天数。 | 7 天 | 无限制 |
| `PASSWORD_LIFE_TIME` | 密码有效天数。 | 180 天 | 无限制 |
| `PASSWORD_LOCK_TIME` | 达到`FAILED_LOGIN_ATTEMPTS`限制后账户锁定的天数。 | 1 天 | 无限制 |
| `PASSWORD_REUSE_MAX` | 密码可重用前需要更换的次数。 | 无限制 | 无限制 |
| `PASSWORD_REUSE_TIME` | 密码可重用前必须经过的天数。 | 无限制 | 无限制 |
| `PASSWORD_VERIFY_FUNCTION` | 用于验证密码的数据库函数。 | 空 | 空 |



