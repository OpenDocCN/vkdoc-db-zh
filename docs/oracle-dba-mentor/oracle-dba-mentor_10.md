# 8. 安全性

在上一章中，我们迈出了确保自己能成为一名良好数据守护者的第一步。我们学习了恢复与备份。数据守护者需要保护数据，并能够在灾难后提供对数据的访问。

本章将完善数据守护者的职责，将其扩展到保护数据免受未授权访问。本章重点讨论数据库安全。鉴于当今的数据泄露事件，所有数据库管理员确保其数据库免受未授权访问至关重要。

## 访问安全指南

Oracle 提供了许多文档，帮助你了解 Oracle 数据库安全性的方方面面。将你的网络浏览器指向 `https://docs.oracle.com` 并点击 `Database` 按钮。然后点击 `Oracle Database` 按钮。在下拉菜单中，选择 `12.2` 数据库版本。到现在为止，你应该已经熟悉了如何获取你所用版本的 Oracle 文档，因为我们已经访问过 Oracle 文档集中的许多指南。

在 `Topics` 部分，点击 `Security` 链接。你应该能看到两本最重要的书：`Oracle Database 2 Day + Security Guide`，紧随其下的是 `Database Security Guide`。

### 提示

在阅读本书时，请确保你正逐渐熟悉 Oracle 文档。

Oracle 提供了多个此类“2 天”指南，涵盖各种主题，旨在让你快速上手。在阅读完 `2 Day + Security Guide` 后，常规的 `Security Guide` 会让你对相关主题有更深入的了解。

请记住，阅读 Oracle 文档时，记住每个细节并不重要。相反，试着记住文档包含的内容。如有需要，可以浏览文档，并务必查阅目录。在你未来的职业生涯中，你将会知道去哪里寻找你想要的答案。

需要提醒的是，不必立即阅读本书每章中提到的 Oracle 文档。你完全可以先读完本书，然后再去查阅 Oracle 文档。但请务必在某个时间点花时间阅读 Oracle 文档。

## 默认用户

在创建 Oracle 数据库时，Oracle 会提供许多默认用户账户。本书前面已经向你介绍了 `SYS` 和 `SYSTEM` 用户。Oracle 为你创建的某些用户取决于你是否安装了某些选件。`DBA_USERS` 视图包含一个名为 `ORACLE_MAINTAINED` 的列，显示用户是否由 Oracle 提供。我们可以在清单 8-1 的输出中看到 Oracle 维护的用户。根据你的版本和已安装的选件，你的用户数量可能与清单 8-1 所示的不同。

```
SQL> select username,account_status
2  from dba_users where oracle_maintained='Y'
3  order by username;
USERNAME                  ACCOUNT_STATUS
------------------------- --------------------------------
ANONYMOUS                 EXPIRED & LOCKED
APPQOSSYS                 EXPIRED & LOCKED
AUDSYS                    EXPIRED & LOCKED
DBSFWUSER                 EXPIRED & LOCKED
DBSNMP                    EXPIRED & LOCKED
DIP                       EXPIRED & LOCKED
GGSYS                     EXPIRED & LOCKED
GSMADMIN_INTERNAL         EXPIRED & LOCKED
GSMCATUSER                EXPIRED & LOCKED
GSMUSER                   EXPIRED & LOCKED
OJVMSYS                   EXPIRED & LOCKED
ORACLE_OCM                EXPIRED & LOCKED
OUTLN                     EXPIRED & LOCKED
REMOTE_SCHEDULER_AGENT    EXPIRED & LOCKED
SYS                       OPEN
SYS$UMF                   EXPIRED & LOCKED
SYSBACKUP                 EXPIRED & LOCKED
SYSDG                     EXPIRED & LOCKED
SYSKM                     EXPIRED & LOCKED
SYSRAC                    EXPIRED & LOCKED
SYSTEM                    OPEN
WMSYS                     EXPIRED & LOCKED
XDB                       EXPIRED & LOCKED
XS$NULL                   EXPIRED & LOCKED
清单 8-1
Oracle 维护的用户
```

在当今的 Oracle 版本中，Oracle 维护的用户是被锁定的，因此没有人可以使用这些用户中的任何一个访问数据库。开箱即用，只有两个用户是解锁的：`SYS` 和 `SYSTEM`。除非你有特定需要，否则不要解锁任何其他用户。在大多数情况下，你永远不需要直接连接到那些用户，保持它们的锁定状态将使你的系统更安全。大多数 Oracle 维护的用户是作为数据库选项的模式存在的。例如，`XDB` 用于支持数据库中的 XML。既然你不是一个数据库选项，你就不需要连接到那个用户。你是一名数据库管理员。因此，在本章后面，你将创建自己的 DBA 账户并锁定 `SYSTEM`。


## 密码配置文件

今天的 IT 从业者都知道，一个好密码对于确保系统安全至关重要。像 password、123456 和 qwerty 这样的密码简直是在邀请黑客使用你的账户。你肯定有过登录某个计算机系统并尝试为账户设置密码时，收到反馈说密码必须超过八个字符、包含大小写字母混合、并包含数字和特殊字符的经历。在本节中，你将学习如何创建一个密码验证函数来强制执行类似的规则。

在开始之前，我们需要将 Oracle 提供的一个文件复制到我们的主目录中。我们不想直接修改这个文件，而只对文件副本进行操作。在清单 8-2 中，我们将 `catpvf.sql` 脚本复制到测试服务器上的 Linux 主目录。

```bash
[oracle@dbamentor ~]$ cp $ORACLE_HOME/rdbms/admin/catpvf.sql $HOME/.
```
清单 8-2
复制安全文件

`catpvf.sql` 脚本包含一个密码验证函数。如果你像第 6 章描述的那样，使用数据库配置助手（`DBCA`）创建了数据库，那么 Oracle 已经针对数据库运行过这些脚本。

在文本编辑器中打开 `catpvf.sql` 脚本。如果你滚动浏览这个文件，会看到它首先创建了一个名为 `ora_complexity_check` 的函数。该函数检查密码是否包含大小写字母、数字和特殊字符。它还检查密码是否足够长。如果其中任何一项检查失败，该函数就会引发错误。

在同一脚本的后面部分，是用于创建函数 `ora12_verify_function` 的代码。这是实际的密码验证函数，在创建用户或更改其密码时会被调用。检查这段代码可以发现，该函数确保密码不包含用户名或反转的用户名。它确保密码不包含服务器名。我们还可以在清单 8-3 中看到对 `ora_complexity_check` 的调用。

```sql
IF NOT ora_complexity_check(password, chars => 8, letter => 1, digit => 1,
special => 1) THEN
  RETURN(FALSE);
```
清单 8-3
调用 `ora_complexity_check`

开箱即用，Oracle 会检查密码是否至少有八个字符，且至少包含一个字母、一个数字和一个特殊字符。许多安全专家会说，好密码的长度应至少为 12 个字符。在这个文件中，让我们将 `8` 改为 `12`。我们还需要通过更改清单 8-3 中的那些参数，来要求密码至少包含两个字母、两个数字和两个特殊字符。然后保存文件。

密码验证函数必须由 `SYS` 用户拥有。启动 `SQL*Plus`，以 `SYS` 用户身份连接，然后执行我们刚刚修改的脚本。我们可以在清单 8-4 中看到一个例子。

```sql
sqlplus sys as sysdba
SQL*Plus: Release 12.2.0.1.0 Production on Thu Jul 5 13:34:05 2018
Copyright (c) 1982, 2016, Oracle.  All rights reserved.
Enter password:
Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production
SQL> @catpvf.sql
```
清单 8-4
修改密码验证函数

输出信息将快速滚动，如果我们正确修改了文件，则不会引发错误。

你不仅限于使用一个密码验证函数。你可以为不同类别的用户设置不同的验证规则。例如，你可能希望管理员用户的密码比普通用户的密码具有更严格的限制。我们将在下一节中看到如何为用户强制使用的密码验证函数进行定义。

现在密码验证函数已经被修改，我们可以通过将 `SYSTEM` 的密码更改为一些我们知道无法通过测试的密码来测试它。在清单 8-5 中，我们首先尝试将密码更改为 "password"，然后更改为一个不包含特殊字符的密码。

```sql
SQL> alter user system identified by password;
alter user system identified by password
*
ERROR at line 1:
ORA-28003: password verification for the specified password failed
ORA-20001: Password length less than 12

SQL> alter user system identified by thisis12char;
alter user system identified by thisis12char
*
ERROR at line 1:
ORA-28003: password verification for the specified password failed
ORA-20026: Password must contain at least 2 special character(s)
```
清单 8-5
尝试使用弱密码

每次，密码都未能通过验证函数，并且给出了有意义的错误信息。

## 安全配置文件

Oracle 允许我们通过一个*配置文件*来控制用户的安全。一个 Oracle 数据库可以使用许多配置文件来支持不同类别的用户。在我管理的数据库中，我通常定义两个配置文件：`DEFAULT` 配置文件和一个用于管理员用户的配置文件。最终用户将获得 `DEFAULT` 配置文件，管理员则获得另一个配置文件。当然，你也可以根据需要添加其他配置文件。当配置文件首次出现时，我从未使用过 `DEFAULT` 配置文件。然而，我多次因为创建用户账户时忘记指定正确的配置文件而吃亏，导致用户账户的安全设置不当。随着时间的推移，我转而向最终用户提供 `DEFAULT` 配置文件。无论你做什么，只需确保在整个企业中保持一致即可。

`DEFAULT` 配置文件是开箱即用创建的，你无法删除它。如果你创建一个新用户而没有指定要使用的配置文件，该用户将获得 `DEFAULT` 配置文件。在文本编辑器中，打开文件 `$ORACLE_HOME/rdbms/admin/utlpwdmg.sql` 并检查其内容。我们不会修改这个文件，只是查看它以了解 Oracle 在使用 `DBCA` 创建数据库时是如何设置的。


### 提示

如果您手动创建数据库，那么您也必须手动设置安全控制。

在 `utlpwdmg.sql` 脚本中，您可以看到 `DEFAULT` 配置文件已被定义，其限制如清单 8-6 所示。

```sql
ALTER PROFILE DEFAULT LIMIT
PASSWORD_LIFE_TIME 180
PASSWORD_GRACE_TIME 7
PASSWORD_REUSE_TIME UNLIMITED
PASSWORD_REUSE_MAX  UNLIMITED
FAILED_LOGIN_ATTEMPTS 10
PASSWORD_LOCK_TIME 1
INACTIVE_ACCOUNT_TIME UNLIMITED
PASSWORD_VERIFY_FUNCTION ora12c_verify_function;
清单 8-6
DEFAULT 配置文件定义
```

配置文件是对账户的一组限制。从清单 8-6 中我们可以看到，该配置文件的 `PASSWORD_LIFE_TIME` 限制为 180 天，`PASSWORD_GRACE_TIME` 限制为 7 天。那么，所有这些限制都意味着什么呢？表 8-1 展示了用于保护 Oracle 数据库的配置文件限制定义。

表 8-1
配置文件参数描述

| 配置文件参数 | 定义 |
| --- | --- |
| `FAILED_LOGIN_ATTEMPTS` | 指定在 Oracle 自动锁定账户之前允许的连续登录失败次数。 |
| `PASSWORD_LIFE_TIME` | 指定密码可以使用的天数。超过这些天数后，必须更改密码。 |
| `PASSWORD_REUSE_TIME` | 指定在多少天后可以重用密码。例如，您可以设置密码在 365 天后才能被重用。 |
| `PASSWORD_REUSE_MAX` | 指定在重用密码之前必须更改密码的次数。例如，您可能需要在更改密码四次后才能重用它。 |
| `PASSWORD_LOCK_TIME` | 指定如果 Oracle 因 `FAILED_LOGIN_ATTEMPTS` 自动锁定账户后，账户将保持锁定的天数。通常此值小于 1。 |
| `PASSWORD_GRACE_TIME` | 指定密码过期后仍可使用的宽限期天数。例如，如果您将 `PASSWORD_LIFE_TIME` 设置为 90 天，`PASSWORD_GRACE_TIME` 设置为 7 天，那么用户实际上有 97 天的时间来更改密码。宽限期过后，账户将被锁定。 |
| `INACTIVE_ACCOUNT_TIME` | 指定账户因不活动而被自动锁定前的天数。例如，当您希望在用户 365 天不登录时锁定账户，此参数很有用。 |
| `PASSWORD_VERIFY_FUNCTION` | 定义配置文件使用的密码验证函数。通过此参数，我们可以根据用户分类来利用不同的密码验证函数。 |

如果您指定了 `PASSWORD_REUSE_TIME`，则也必须指定 `PASSWORD_REUSE_MAX`，反之亦然。

如表 8-1 所述，`PASSWORD_LOCK_TIME` 是以天为单位指定的。实际上我们很少将账户锁定一整天，因此此值通常小于 1。常见做法是将账户锁定 10 分钟。一天有 24 小时，每小时 60 分钟。要计算 10 分钟折合为多少天的值，我们会进行一个简单的计算：`10/(24*60)`，大约为 `0.00694`。我不清楚您如何，但要我记住 `0.00694` 天等于 10 分钟是很困难的。因此，在清单 8-7 中，我将使用公式 `10/(24*60)` 而不是 `0.00694` 来表示 10 分钟，这使得限制条件更容易理解。

我在生产数据库中锁定安全性的第一步就是修改 `DEFAULT` 配置文件。我的公司政策规定，用户必须每 90 天更改一次密码，并且不能重用过去两年内使用过的任何密码。我们希望 Oracle 在五次无效登录尝试后锁定账户，并保持锁定 10 分钟。我们还会锁定过去一年内未使用的账户。基于这些定义，我现在可以发出清单 8-7 中的命令来修改 `DEFAULT` 配置文件的设置。

```sql
ALTER PROFILE DEFAULT LIMIT
PASSWORD_LIFE_TIME 90
PASSWORD_GRACE_TIME 7
PASSWORD_REUSE_TIME 730
PASSWORD_REUSE_MAX  8
FAILED_LOGIN_ATTEMPTS 5
PASSWORD_LOCK_TIME 10/(24*60)
INACTIVE_ACCOUNT_TIME 365
PASSWORD_VERIFY_FUNCTION ora12c_verify_function;
清单 8-7
改进 DEFAULT 配置文件
```

如前所述，我使用公式为锁时间参数 `PASSWORD_LOCK_TIME` 定义了 10 分钟。再次强调，我发现通过看到公式而不是 `0.00694` 来判断我意指 10 分钟往往更容易。但是，如果我们查询数据字典以获取配置文件的密码限制，我们会看到锁时间表示为数字，如清单 8-8 所示。

```sql
SQL> select resource_name,limit from dba_profiles
2  where profile='DEFAULT'
3  and resource_type='PASSWORD';
RESOURCE_NAME                    LIMIT
-------------------------------- ------------------------------
FAILED_LOGIN_ATTEMPTS            5
PASSWORD_LIFE_TIME               90
PASSWORD_REUSE_TIME              730
PASSWORD_REUSE_MAX               8
PASSWORD_VERIFY_FUNCTION         ORA12C_VERIFY_FUNCTION
PASSWORD_LOCK_TIME               .0069
PASSWORD_GRACE_TIME              7
INACTIVE_ACCOUNT_TIME            365
清单 8-8
查询配置文件限制
```

接下来，我将为我们的管理员用户创建一个配置文件。由于账户角色的重要性增加，此配置文件比最终用户的配置文件限制更严格。对于管理员，我们希望他们每 60 天更改一次密码。他们的账户应在三次无效登录尝试后被锁定，并保持锁定 20 分钟。创建配置文件后，我将修改 `SYS` 和 `SYSTEM` 用户以使用该配置文件。清单 8-9 显示了配置文件创建和用户修改过程。

```sql
SQL> create profile admin_profile
2  limit failed_login_attempts 3
3  password_life_time 60
4  password_verify_function ora12c_verify_function
5  password_lock_time 20/(24*60)
6  inactive_account_time 90;
Profile created.
SQL> alter user sys profile admin_profile;
User altered.
SQL> alter user system profile admin_profile;
User altered.
SQL> select username,profile from dba_users
2  where username in ('SYS','SYSTEM');
USERNAME                  PROFILE
------------------------- ---------------
SYS                       ADMIN_PROFILE
SYSTEM                    ADMIN_PROFILE
清单 8-9
管理员配置文件
```

配置文件已创建。用户已修改。最后，已验证用户具有正确的配置文件。

现在我们有了管理员配置文件，就可以创建我们的 DBA 用户了。多人共享同一个账户是一种不良的安全实践，尤其是对于管理账户。不幸的是，我发现有太多站点过于频繁地与他们的 DBA 团队共享 `SYSTEM` 账户。每个 DBA 都应该拥有自己唯一的账户。在清单 8-10 中，我将使用 `ADMIN_PROFILE` 创建我的 DBA 账户，授予其在数据库中创建会话的能力，并赋予其 `DBA` 角色。最后，我将锁定 `SYSTEM` 账户。

```sql
SQL> create user peasland identified by MyPassword31$$ profile admin_profile;
User created.
SQL> grant create session to peasland;
Grant succeeded.
SQL> grant dba to peasland;
Grant succeeded.
SQL> alter user system account lock;
User altered.
SQL> connect peasland/MyPassword31$$
Connected.
清单 8-10
创建 DBA 用户
```

由于这些命令出现在一本书中，您可以放心，我在清单 8-10 中使用的密码绝不会用于我的任何真实账户。

### 提示

创建您自己的 DBA 账户并锁定 `SYSTEM`。


## 审计

到目前为止，本章已经稍微加强了数据库的安全性。如果有人确实入侵了数据库会怎样？如果获得数据库访问权限的人决定滥用该权限会怎么办？我们需要采取一些措施，以便进行取证分析，帮助追踪安全问题。我们将利用 Oracle 的审计机制来实现这些目的。

在 Oracle Database 12c 之前，DBA 必须先开启审计，然后告诉 Oracle 具体将哪些事件写入审计跟踪。Oracle 12c 引入了统一审计来简化这一过程。在 12c 中，你仍然可以执行传统审计。你甚至可以同时执行传统审计和统一审计，Oracle 称之为**混合模式审计**。

要开始审计，必须执行两个步骤。首先，开启生成审计跟踪记录的功能。其次，告诉 Oracle 要审计什么。在我的职业生涯中，我见过人们忘记其中任何一个步骤。DBA 开启审计后，会抱怨 Oracle 没有向审计跟踪写入任何内容，却不知道需要执行第二个步骤。我见过的另一个常见抱怨是，DBA 告诉了 Oracle 要审计什么，但审计跟踪中仍然没有写入任何内容，因为他们忘记开启审计了。第一步只需要执行一次。第二步可以执行多次。

### 提示

审计是一个两步过程。开启审计。告诉 Oracle 要审计什么。

如果你使用 Oracle 12c 的 DBCA 实用程序创建数据库，默认情况下审计是开启的。如果审计未开启，你可以通过执行清单 8-11 中的命令来开启。

```
SQL> connect / as sysdba
Connected.
SQL> alter system set audit_trail=db scope=spfile;
System altered.
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup
ORACLE instance started.
Total System Global Area 4294967296 bytes
Fixed Size                  8628936 bytes
Variable Size             939525432 bytes
Database Buffers         3338665984 bytes
Redo Buffers                8146944 bytes
Database mounted.
Database opened.
```

清单 8-11
开启审计

因为实例被重启了，清单 8-11 中的步骤需要停机时间，并且可能需要在维护窗口内执行。

现在审计已经开启，我们将使用两个统一审计策略。第一个策略名为`ORA_LOGON_FAILURES`，每当有人无法正确向 Oracle 数据库认证时，它将在审计跟踪中生成记录。第二个策略名为`ORA_ACCOUNT_MGMT`，每当管理员创建、修改和删除用户时，它将在审计跟踪中生成记录。Oracle 的统一审计开箱即提供了这些策略，因为它们是典型的审计活动。在清单 8-12 中，我告诉 Oracle 开始审计这两个策略。我必须重新连接后，统一审计才会捕获我的活动。然后我创建了一个测试用户。最后，我查询审计跟踪以显示该活动已被捕获。

```
SQL> audit policy ora_logon_failures;
Audit succeeded.
SQL> audit policy ora_account_mgmt;
Audit succeeded.
SQL> connect peasland
Enter password:
SQL> create user test_user identified by h3r$passw0rd$;
User created.
SQL> select dbusername,event_timestamp,sql_text from unified_audit_trail
2  where sql_text like 'create user%';
DBUSERNAME EVENT_TIMESTAMP                SQL_TEXT
---------- ---------------------------- ------------------------------------
SYS        05-JUL-18 03.14.05.874100 PM   create user test_user identified by *
```

清单 8-12
审计操作

审计还有很多内容。我在清单 8-12 中开启的两个统一审计策略是我最常用的。然而，建议你通读*《Oracle 安全指南》*，以了解更多内容，并查看是否有其他策略对你的具体情况有益。本节旨在向你展示如何开始审计重要操作，并让你了解审计是什么以及它的作用。

## 其他安全主题

一整本书都可以用来写 Oracle 数据库安全。这个主题如此广阔。本章实在没有足够的篇幅来讨论与数据库安全相关的所有内容。本章的其余部分将高层讨论一些安全主题，你可能需要在以后的职业生涯中进一步探索。希望当你遇到数据库的安全考虑时，可以参考本节，从而知道去哪里探索以完成工作。

### 系统权限

Oracle 开箱即提供了许多系统权限。创建会话和连接到数据库的能力是我们在本章中已经看到的一个例子。其他系统权限的例子包括创建表的能力以及创建或删除用户的能力。

### 对象权限

一旦用户在数据库中创建了对象，例如表或视图，他们可能希望授予其他用户与这些对象交互的能力。一个常见的例子是让另一个用户查看表的内容但不修改数据。表的所有者将授予从该表`SELECT`的权限。执行另一个用户的存储过程需要另一个对象权限。

### 角色

由于存在这些不同类型的权限，通常希望给予同类用户相同的权限。在本章前面，我创建了一个 DBA 用户。完成后，在我的 Oracle 12.2 数据库中，该用户被授予了 237 个不同的系统权限。然而，本章中的示例只显示了一个授权被发出。该用户被授予了`DBA`角色。你可以将多个权限授予一个角色，然后当你将该角色授予另一个用户时，该用户将继承该角色的所有权限。角色存在的唯一原因就是为了让管理员更容易地授予多个权限。如果管理员必须记住在 Oracle 12.2 中构成`DBA`角色的所有 237 个系统权限，那肯定会漏掉一些。通过将权限授予一个角色，管理员只需要记住角色名称并授予该角色即可。

Oracle 数据库的一个奇特之处是，角色在存储过程、函数和包内不起作用。如果用户创建了这些对象之一并遇到权限问题，DBA 应验证对象所有者已被直接授予对象的权限，而不是通过角色。

### 透明数据加密

透明数据加密（TDE）是 Oracle 的一个选项，用于加密处于静态的 Oracle 数据库内容。数据在磁盘上的数据文件中被加密。好处在于，这种加密对最终用户和应用程序是透明的，因此得名。

太多人错误地认为 TDE 意味着他们的数据库对窥探者是加密的，而事实并非如此。任何能够访问数据库并拥有适当权限的用户都可以以未加密的形式查看该对象中的数据。

透明数据加密的最大好处是加密你的数据库备份。借助 TDE，你可以放心，如果有人获取了你的备份（可能来自异地位置），只要备份不包含数据库的钱包（存储加密密钥），数据就是加密的。请记住，一旦数据库开放供业务使用，任何有权访问数据库的人都可以透明地以未加密形式查看数据。

透明数据加密确实要求你的组织获得 Oracle 高级安全选项的许可。此选项不包含在企业版（EE）许可中。即使你想要许可它，也不能在标准版中使用 TDE。你必须使用 EE。

### DBMS_CRYPTO

如果你希望即使在数据库开放的情况下，数据也能免受窥探，最好的办法是利用 `DBMS_CRYPTO` 供应包。这不需要额外的许可证，但它对应用程序并不透明。应用程序必须在想要查看加密数据时调用此包，并且必须调用此包来以加密形式存储数据。

### 虚拟专用数据库

虚拟专用数据库 (`VPD`) 是企业版功能，它允许每个用户访问同一个表，但只允许每个用户看到他们有权看到的数据。例如，你可能有一个 `EMPLOYEES` 表。员工只能看到自己在此表中的记录。经理可以看到自己的记录以及向他们汇报的任何员工的记录。人力资源部门的某个人可能可以看到此表中的所有员工。`VPD` 使用应用程序上下文来定义某人在启用了 `VPD` 的表中可以看到的记录。一旦定义了应用程序上下文，用户就只能看到他们被允许看到的数据。通过这种方式，表中的数据根据其分类对用户是私有的。

### 网络加密

Oracle 会通过网络以明文形式传输数据。这可能引起一些组织的担忧。Oracle 确实提供了在数据流经网络时对其进行加密的能力。与透明数据加密类似，此功能需要额外成本，包含在高级安全选项中，这是 Oracle 企业版的一个额外付费选项。

### 更强的身份验证

对于某些组织来说，简单的密码是不够的。Oracle 确实提供了利用 `Kerberos` 或 `RADIUS` 进行身份验证的能力。组织也可以选择利用 `LDAP` 身份验证。

## 继续前行

本章通过提高数据库的安全性来帮助我们加固数据库。我们配置了一个密码验证函数，然后利用配置文件来实施安全控制。本章以及第七章通过概述如何通过备份和更好的安全性来保护数据，使我们能够成为公司数据库更好的数据守护者。

请记住，数据库安全这个主题非常广泛，无法在本书的一章中充分涵盖。至关重要的是，你要阅读更多关于这个主题的资料，特别是本章引用的 Oracle 指南，以学习如何保护数据库中的数据。

到目前为止，我们已经建立了我们的测试系统，然后确保我们为未来做好了准备。接下来的几章将有一点不同，因为我们将更多地转向如何使用 Oracle 数据库。在下一章中，我们将讨论如何连接到 Oracle 数据库。用户很少从数据库服务器本身连接到数据库。正如我们将要看到的，连接到一个运行的 Oracle 实例有多种方式。

