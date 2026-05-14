# 文件系统检查
7 * * * * /orahome/bin/filesp.bsh 1>/orahome/bin/log/filesp.log 2>&1

### login.sql

使用此脚本来定制您的 SQL*Plus 环境的某些方面。在 Linux/Unix 中登录 SQL*Plus 时，如果 `login.sql` 脚本存在于 `SQLPATH` 变量包含的目录中，则会自动执行该脚本。如果 `SQLPATH` 变量未定义，则 SQL*Plus 会在调用 SQL*Plus 时所在的工作目录中查找 `login.sql`。例如，以下是我环境中 `SQLPATH` 变量的定义方式：

$ echo $SQLPATH
/home/oracle/scripts

在 `/home/oracle/scripts` 目录中创建 `login.sql` 脚本。其内容如下：

-- 设置 SQL 提示符
SET SQLPROMPT '&_USER.@&_CONNECT_IDENTIFIER.> '

现在，当我登录 SQL*Plus 时，提示符会自动设置：

$ sqlplus / as sysdba
SYS@O11R2>

### top.sql

以下脚本列出消耗 CPU 最多的 SQL 进程。这对于识别问题 SQL 语句非常有用。将此脚本放置在 `HOME/scripts` 等目录中：

```sql
select * from(
select
    sql_text
   ,buffer_gets
   ,disk_reads
   ,sorts
   ,cpu_time/1000000 cpu_sec
   ,executions
   ,rows_processed
from v$sqlstats
order by cpu_time DESC)
where rownum < 11;
```

执行此脚本的方法如下：

SQL> @top

以下是一小段输出，显示了一条消耗大量数据库资源的 SQL 语句：

SQL_TEXT
insert into reg_queue (registration_urn,registration_data,client_ip_addr, relay_
BUFFER_GETS DISK_READS SORTS CPU_SEC EXECUTIONS ROWS_PROCESSED
----------- ---------- ---------- ---------- ---------- --------------
6079221 2482 28309 986.704467 697494 997467

### lock.sql

此脚本显示对表持有锁并阻止其他会话完成工作的会话。该脚本显示有关阻塞和等待会话的详细信息。您应将此脚本放置在 `HOME/scripts` 等目录中。`lock.sql` 的内容如下：

```sql
select s1.username blkg_user, s1.machine blkg_ws, s1.sid blkg_sid,
       s2.username wait_user, s2.machine wait_ws, s2.sid wait_sid,
       lo.object_id blkd_obj_id, do.owner, do.object_name
from v$lock l1, v$session s1, v$lock l2, v$session s2,
     v$locked_object lo, dba_objects do
where s1.sid = l1.sid
  and s2.sid = l2.sid
  and l1.id1 = l2.id1
  and s1.sid = lo.session_id
  and lo.object_id = do.object_id
  and l1.block = 1
  and l2.request > 0;
```

`lock.sql` 脚本对于确定哪个会话持有对象的锁以及显示被阻塞的会话非常有用。您可以从 SQL*Plus 运行此脚本，如下所示：

SQL> @lock.sql

以下是部分输出列表（已截断以适应单页）：

BLKG_USER BLKG_WS BLKG_SID WAIT_USER WAIT_WS
---------- -------------------- ---------- ---------- --------------------
MV ora03.regis.local 88 INV_APP ora03.regis.local

### users.sql

此脚本显示有关用户创建时间及其账户是否被锁定的信息。当您对连接问题进行故障排除时，此脚本非常有用。将其放置在 `HOME/scripts` 等目录中。以下是一个典型的用于显示用户账户信息的 `users.sql` 脚本：

```sql
SELECT
    username
   ,account_status
   ,lock_date
   ,created
FROM dba_users
ORDER BY username;
```

您可以从 SQL*Plus 执行此脚本，如下所示：

SQL> @users.sql

以下是一些示例输出：

USERNAME ACCOUNT_STATUS LOCK_DATE CREATED
-------------------- -------------------------------- --------- ---------
CLUSTERUSER OPEN 09-MAY-10
DIP EXPIRED & LOCKED 09-MAY-10 09-MAY-10
DP122764 LOCKED 07-JAN-10 09-JUL-10

## 组织脚本

当您拥有一组脚本和实用程序时，应对其进行组织，以便在每个数据库服务器上都能一致地实施。请按照以下步骤为环境中的每个数据库服务器实施前述 DBA 实用程序：

1.  创建用于存储脚本的操作系统目录。
2.  将您的脚本和实用程序复制到步骤 1 创建的目录中。
3.  配置您的启动文件以初始化环境。

前面的步骤将在以下小节中详细说明。

### 步骤 1：创建目录

在每个数据库服务器上创建一套标准目录来存储您的自定义脚本。`oracle` 用户的 `HOME` 目录下的目录通常是一个不错的位置。我通常创建以下三个目录：

*   `HOME/bin`。以自动方式（如从 `cron`）运行的 shell 脚本的标准位置。
*   `HOME/bin/log`。由计划的 shell 脚本生成的日志文件的标准位置。
*   `HOME/scripts`。存储 SQL 脚本的标准位置。

您可以使用 `mkdir` 命令创建上述目录，如下所示：

$ mkdir -p $HOME/bin/log
$ mkdir $HOME/scripts

脚本放置在何处或目录如何命名并不重要，只要您有一个标准位置，这样当您在服务器间导航时，总能在相同位置找到相同的文件。换句话说，标准是什么并不重要，重要的是您有一个标准。

### 步骤 2：将文件复制到目录

将您的实用程序和脚本放入相应的目录。将以下文件复制到 `HOME/bin` 目录：

*   `dba_setup`
*   `dba_fcns`
*   `tbsp_chk.bsh`
*   `conn.bsh`
*   `filesp.bsh`

将以下 SQL 脚本放入 `HOME/scripts` 目录：

*   `login.sql`
*   `top.sql`
*   `lock.sql`
*   `users.sql`

### 步骤 3：配置启动文件

将以下代码放入 `.bashrc` 文件或您使用的 shell 的等效启动文件中（Korn shell 为 `.profile`）。以下是如何配置 `.bashrc` 文件的示例：

```bash
# 引入全局定义
if [ -f /etc/bashrc ]; then
    . /etc/bashrc
fi
#
# 引入 Oracle 操作系统变量
. /var/opt/oracle/oraset <default_database>
#
```

