# 使用 dbnodeupdate.sh 应用补丁

`dbnodeupdate.sh` 脚本可用于协助完成计算节点操作系统和固件的升级过程。如前所述，Oracle 支持部门会定期发布新版本的 `dbnodeupdate.sh` 脚本。该脚本可通过补丁号 #16486998 获取。虽然 `dbnodeupdate.sh` 主要用于升级操作系统，但它还执行若干附加任务。由于“最佳实践”会不断演进，新的实践优于旧版，`dbnodeupdate.sh` 会例行检查以确保您的计算节点遵循 MOS 笔记 #757552.1 中的指导原则。此类检查的一个例子是 Oracle 更改了分配给 Linux 操作系统的内存量（内核参数 `vm.min_free_kbytes`）的最低建议值。在许多早期部署中，仅分配了约 50MB。在客户因操作系统内存不足而开始看到节点被逐出集群后，建议值提高到了约 512MB。由于许多客户可能在篇幅很长的笔记中看不到该建议，`dbnodeupdate.sh` 脚本会检查并在下次运行时更改此设置。这只是 `dbnodeupdate.sh` 在需要时可以实施的修复措施之一。其他版本将包含针对该版本发布后发现的各种漏洞的安全修复（特别是 BASH 的“Shellshock”漏洞）。

`dbnodeupdate.sh` 脚本的另一个关键特性是它利用 Linux 原生的逻辑卷管理 (LVM) 功能。由于 Exadata 上的文件系统布局——`/` 是一个 30GB 的逻辑卷，而 `/u01` 是一个 100GB 的逻辑卷——在进行任何更改之前，可以轻松地对根文件系统进行备份。虽然此选项自 X2-2 发布以来就已可用，但许多客户在升级操作系统前并未进行快照备份。使用 `dbnodeupdate.sh` 时，会自动创建备份。尽管方法与 Exadata 存储服务器所用的不同，但概念相似。Exadata 存储服务器采用就地补丁机制。在 Exadata 计算节点上，非活动卷仅在需要回退时使用——除非发生故障，否则补丁是直接应用在原位置的。默认的根文件系统位于 `VGExaDb-LVDbSys1` 逻辑卷上。首次运行 `dbnodeupdate.sh` 时，将创建一个新的逻辑卷 `VGExaDb-LVDbSys2`。如果之前运行过 `dbnodeupdate.sh`，则该卷将在下次运行时被覆盖。此卷将用于在脚本运行时创建原始活动卷的完整备份。首先，获取根文件系统的快照，并将其挂载到 `/mnt_snap`。在非活动根卷（通常为 `VGExaDb-LVDbSys2`）创建后，将其挂载到 `/mnt_spare`，并使用 `tar` 工具将快照内容复制到 `/mnt_spare`。将所有内容复制到 `/mnt_spare` 后，卸载该卷并删除快照卷。现在可能是个好时机提醒一下，如果增大根文件系统的大小，备份肯定会花费更长时间。此外，需要在卷组中预留额外空间以容纳更大的根卷。请记住，只有根文件系统会被 `dbnodeupdate.sh` 备份，因为该脚本不会修改 `/u01` 上的任何文件。同样也会创建 `/boot` 文件系统的副本，以防需要备份。表 1 描述了可与 `dbnodeupdate.sh` 一起使用的可用标志。

表 16-1. 与 dbnodeupdate.sh 一起使用的标志

| 标志 | 描述 |
| --- | --- |
| -u | 将计算节点更新到新版本 |
| -r | 将计算节点回退到先前版本 |
| -c | 执行补丁后或回退后操作 |
| -l -s -q -n -p -v -t -V | 包含 ISO 文件的 zip 文件的 URL 或位置 运行前关闭 Clusterware 栈 安静模式—与 -t 一起使用 禁用文件系统备份 引导阶段，当从版本 11.2.2.4.2 更新时使用 仅运行—仅验证先决条件 与 –q 一起使用以指定要更新到的版本 打印版本号 |

运行 `dbnodeupdate.sh` 之前，请下载包含所需补丁级别对应 ISO 的补丁。将该文件以及包含 `dbnodeupdate.sh` 脚本的补丁放置到每个计算节点上。虽然 `dbnodeupdate.sh` 可以通过 HTTP 从 yum 仓库拉取，但 ISO 方法已被证明更为直接。`dbnodeupdate.sh` 的每次运行分为两个阶段——升级/回退阶段和收尾阶段。就像 `OPatch` 的 `auto` 功能一样，`dbnodeupdate.sh` 执行许多原本需要单独执行的不同任务。首先，`dbnodeupdate.sh` 将关闭并解锁 grid infrastructure 主目录，然后备份根文件系统。接下来，`dbnodeupdate.sh` 将包含补丁仓库的 zip 文件解压到临时位置，挂载该 ISO，并更新 `/etc/yum.repos.d/Exadata-computenode.repo` 文件以使用 RPM 软件包的位置。执行 yum 更新，系统随后重启。在主机重启期间，固件组件会进行升级。通常，固件更新包括 InfiniBand HCA、RAID 控制器、BIOS 和 ILOM。升级过程采用与存储服务器相同的方法——运行 `/opt/oracle.cellos/CheckHWnFWProfile` 脚本，将固件版本与支持的版本注册表进行比较。如果有组件不匹配，则将其刷新到预期版本。所有固件更新应用完毕后，主机将重新启动。

主机重启后，当 `imageinfo` 命令显示升级成功，必须再次运行 `dbnodeupdate.sh` 脚本来“完成”升级。此模式将验证 yum 更新是否成功，清理 yum 缓存，为 RDS 协议重新链接 Oracle 主目录，并启动 Clusterware 及其在启动时启用。此步骤完成后，主机即可重新投入使用。

