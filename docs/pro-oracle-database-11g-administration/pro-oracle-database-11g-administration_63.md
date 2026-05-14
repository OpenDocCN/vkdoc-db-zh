# 文件备份，dk：2010 年 10 月 20 日，已插入。

`5 23 * * * /home/oracle/bin/backup.bsh`

4. 编辑完成后，如图所示加载回 crontab：`$ crontab mycron.txt`

如果你的文件不符合 cron 语法，你会收到类似以下的错误信息：

```
"mycron.txt":6: bad day-of-week
errors in crontab file, can't install.
```

遇到这种情况，要么修正语法错误，要么重新加载 cron 表的原始副本。

## 重定向 cron 输出

每当运行 Linux Shell 命令时，默认情况下命令的标准输出会显示在屏幕上。同样，如果产生任何错误信息，默认也会显示在你的终端上。你可以使用 `>` 或 `1>`（它们是同义的）将任何标准输出重定向到操作系统文件。此外，你可以使用 `2>` 将任何错误信息重定向到文件。`2>&1` 这种记法指示 Shell 将任何错误信息发送到与标准输出相同的位置。

当你创建一个 cron 任务时，可以使用这些重定向功能将 Shell 脚本的输出发送到日志文件。例如，在以下的 cron 表条目中，由 `backup.bsh` Shell 脚本生成的任何标准输出和错误信息都被捕获到名为 `bck.log` 的文件中：`11 12 * * * /home/oracle/bin/backup.bsh 1>/home/oracle/bin/log/bck.log 2>&1`

如果你不重定向 cron 任务的输出，那么任何输出都会以电子邮件形式发送给拥有该 cron 任务的用户。你可以通过直接在 cron 表中指定 `MAILTO` 变量来覆盖此行为。

在下一个示例中，你想"烦扰"系统管理员，将 cron 输出发送给 root 用户：

```
MAILTO=root
11 12 * * * /home/oracle/bin/backup.bsh
```

如果你不希望输出发送到任何地方，那么将其重定向到众所周知的"位桶"。以下条目将标准输出和标准错误发送到 `/dev/null` 设备：`11 12 * * * /home/oracle/bin/backup.bsh 1>/dev/null 2>&1`

## 疑难解答 cron

如果你的某个 cron 任务运行不正确，请按照以下步骤排查问题：

1.  复制你的 `cron` 条目，粘贴到操作系统命令行，并手动运行该命令。通常，目录或文件名中的轻微拼写错误可能是问题的根源。手动运行命令会突出显示此类错误。

2.  如果脚本运行 Oracle 实用程序，请确保你在脚本内部 `source`（设置）了所需的操作系统变量，例如 `ORACLE_HOME` 和 `ORACLE_SID`。通常，这些变量在你登录时由启动脚本（如 `HOME/.bashrc`）设置。由于 cron 不运行用户的启动脚本，任何必需的变量都必须在脚本内显式设置。

3.  确保任何从 cron 调用的 Shell 脚本的第一行指定了用于解释脚本内命令的程序名称。例如，`#!/bin/bash` 应该是 Bash Shell 脚本的第一行。

    由于 cron 不运行用户的启动脚本（如 `HOME/.bashrc`），你不能假设你的操作系统用户的默认 Shell 将用于运行从 cron 调用的命令或脚本。

4.  确保 cron 后台进程正在运行。在操作系统中执行以下命令进行验证：

    ```
    $ ps -ef | grep cron
    ```

5.  如果 cron 守护进程（后台进程）正在运行，你应该看到类似以下内容：

    ```
    root 2969 1 0 Mar23 ? 00:00:00 crond
    ```

6.  检查服务器上的电子邮件。当有行为异常的 cron 任务出现问题时，cron 实用程序通常会向操作系统账户发送电子邮件。

7.  检查 `/var/log/cron` 文件的内容是否有任何错误。有时这个文件包含关于未能运行的 cron 任务的相关信息。

## 自动化 DBA 任务示例

在当今通常混乱的商业环境中，自动化任务几乎是强制性的。如果不进行自动化，你可能会忘记执行某项任务，或者在手动执行工作时引入错误。如果不进行自动化，你可能会发现自己被一组更高效或更便宜的 DBA 取代或外包。

当我自动化任务时，脚本通常只在失败的情况下才发送电子邮件。

在成功时生成电子邮件通常会导致邮箱爆满。有些 DBA 喜欢看到成功消息。我通常不喜欢。

DBA 自动化各种各样的任务和工作。几乎任何类型的环境都要求你创建某种操作系统脚本，该脚本封装了操作系统命令、SQL 语句和 PL/SQL 块的组合。

本章中的以下脚本是 DBA 将自动化的各种不同类型任务的示例。这套脚本绝非完整。其中许多脚本在你的环境中可能不需要。关键是要给你提供一个良好的采样，展示自动化的任务类型以及用于完成特定任务的技术。

**注意** 第 3 章包含了一些 DBA 所需的核心脚本的基础示例。本节提供了 DBA 常自动化的任务和脚本的高级示例。

## 启动和停止数据库与监听器

在许多环境中，希望 Oracle 数据库和监听器在服务器重启时自动关闭和启动。如果你有此要求，那么请按照以下步骤自动化你的数据库和监听器的关闭和启动：

1.  编辑 `/etc/oratab` 文件，并在你希望系统重启时自动重启的数据库条目末尾放置一个 `Y`。你可能需要 `root` 权限来编辑该文件：

    ```
    # vi /etc/oratab
    ```

2.  在文件中为你的环境添加一行类似以下的内容：`O11R2:/oracle/app/oracle/product/11.2.0/db_1:Y`

3.  在上一行中，`O11R2` 是数据库名称，`/oracle/app/oracle/product/11.2.0/db_1` 指定了 `ORACLE_HOME` 目录。

    末尾的 `Y` 表示该数据库可以由 `ORACLE_HOME/bin/dbstart` 和 `ORACLE_HOME/bin/dbshut` 脚本启动和停止。

    如果你不希望数据库自动停止和重启，可以将 `Y` 替换为 `N`。

    **注意** 在某些 Unix 系统（如 Solaris）中，oratab 文件通常位于 `/var/opt/oracle` 目录。

4.  以 `root` 身份导航到 `/etc/init.d` 目录，并创建一个名为 `dbora` 的文件：

    ```
    # cd /etc/init.d
    # vi dbora
    ```

5.  将以下行放入 `dbora` 文件中。确保将变量 `ORA_HOME` 和 `ORA_OWNER` 的值更改为匹配你的环境。这是一个用于停止和启动数据库及监听器的最简骨架脚本：

    ```
    #!/bin/bash
    # chkconfig: 35 99 10
    # description: Starts and stops Oracle processes
    ORA_HOME=/oracle/app/oracle/product/11.2.0/db_1
    ORA_OWNER=oracle
    case "$1" in
    'start')
        su - $ORA_OWNER -c "$ORA_HOME/bin/lsnrctl start"
        su - $ORA_OWNER -c $ORA_HOME/bin/dbstart
        ;;
    'stop')
        su - $ORA_OWNER -c "$ORA_HOME/bin/lsnrctl stop"
        su - $ORA_OWNER -c $ORA_HOME/bin/dbshut
        ;;
    esac
    ```

这些行在 `dbora` 文件中看起来像注释，但实际上是必需的行：

```
# chkconfig: 35 99 10
```



