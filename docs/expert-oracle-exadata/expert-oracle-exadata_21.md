# Exadata X5-2 存储服务器中的闪存

直至 X5 代，每台 Exadata 存储服务器都配备了类似于上节所示的、采用多 FMOD 设计的闪存卡。容量当然有所不同，但卡的设计一直保持不变。X5-2 是第一代改变了其中部分特性的存储服务器：

*   新的闪存卡——F160——不再由多个 FMOD 组成。
*   操作系统对这些闪存卡的寻址方式不同。
*   您可以选择使用全闪存型号的存储服务器。

当您检查 X5-2 高容量（High Capacity）单元上 `cellcli` 命令的输出时，您会很快发现确实只有一张“卡”，没有更多的模块：

```
CellCLI> list physicaldisk attributes name,deviceName,diskType,makeModel

   8:0             /dev/sda        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:1             /dev/sdb        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:2             /dev/sdc        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:3             /dev/sdd        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:4             /dev/sde        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:5             /dev/sdf        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:6             /dev/sdg        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:7             /dev/sdh        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:8             /dev/sdi        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:9             /dev/sdj        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:10            /dev/sdk        HardDisk        "HGST    H7240AS60SUN4.0T"
   8:11            /dev/sdl        HardDisk        "HGST    H7240AS60SUN4.0T"
   FLASH_1_1       /dev/nvme3n1    FlashDisk       "Oracle Flash Accelerator F160 PCIe Card"
   FLASH_2_1       /dev/nvme2n1    FlashDisk       "Oracle Flash Accelerator F160 PCIe Card"
   FLASH_4_1       /dev/nvme0n1    FlashDisk       "Oracle Flash Accelerator F160 PCIe Card"
   FLASH_5_1       /dev/nvme1n1    FlashDisk       "Oracle Flash Accelerator F160 PCIe Card"
```

如果您仔细查看输出，无疑会注意到闪存磁盘不仅数量变少了，而且它们的设备名称也大不相同。这与这些设备可以使用的新 `NVMHCI` 接口（简称 `NVMe`）有关。长期以来，高性能闪存的寻址方式或多或少与机械磁盘相同。随着近几代处理器的进步，从性能角度来看，这种闪存寻址模式变得日益过时，特别是当存储设备连接到与处理器本身具有极快链路的 `PCIe` 总线时。最显著的区别是通过 `NVMe` 设备可以处理的队列深度大幅增加，以及可处理的中断数量增加。仅为了证明这一点，您不会在 `lsscsi` 命令输出中看到 `NVMe` 设备被列出。因此，它们真正代表了 x86-64 架构中的下一代闪存。

此外，F160 卡是真正的 `PCIe` v3 卡，而 F80 是 `PCIe` v2 卡。主要区别在于 `PCIe` v3 提供的带宽增加，与上一代相比几乎翻了一番。

根据文档，存储服务器理论上允许在系统运行时更换 `PCIe` 卡。但是，《Oracle Exadata 存储软件用户指南》和 My Oracle Support 笔记 1993842.1 建议在更换这些卡之前关闭存储服务器电源。幸运的是，您可以在不经历停机的情况下完成此操作，因为 `ASM` 冗余允许整个存储单元离线而不影响它们所支持的数据库。请注意，更换其中一张闪存卡应该不需要任何重新配置。

## 闪存缓存 vs. 闪存磁盘

除 X5-2 高性能（High Performance）型号外，所有 Exadata 存储服务器都允许您定义如何使用服务器中的闪存卡。本节描述非 X5-2 高性能单元——关于全闪存或高性能型号的更多信息，您可以稍后阅读。在所有其他（非 X5-2 高性能）型号上，存储服务器上的闪存可以以两种方式使用。它可以配置为磁盘缓存 (`ESFC`)，也可以被划分并作为固态（网格）磁盘呈现，供 `ASM` 用于数据库存储。这两种模式并非互斥。闪存可以按所需任意百分比分配给任一格式。推荐的配置是将所有闪存用作闪存缓存（Flash Cache）。此配置显著加快了随机访问读取速度，并且自 Exadata 版本 11.2.3.3 起，此功能也扩展到多块读取。闪存缓存可用于两种配置：直写（write-back）和透写（write-through），这两种都将在本章后面描述。目前，只需说明 `ESFC` 的默认操作模式是透写（write-through）。

在许多系统中，将所有可用闪存分配为透写缓存的方法效果很好。然而，对于写操作非常密集的系统，从 Exadata 存储软件版本 11.2.3.2.1 开始，从透写更改为回写闪存缓存 (`WBFC`) 可能是有益的。在回写模式下运行，除了缓存读 `I/Os` 外，还能缓存写 `I/Os` 到闪存设备。启用此功能将为您提供远超本地硬盘组合所能提供的写 `IOPS`。

您也可以将部分闪存用作网格磁盘。请记住，根据所使用的 `ASM` 冗余级别（正常或高），选择此选项将消耗物体实际占用闪存容量的两倍或三倍。仅这一事实就使得该选项不那么吸引人，尤其是在闪存容量不像 X4 及更高代际那样丰富的 Exadata 硬件上。另外请注意，对数据文件的写操作是由 `DBWR` 进程在后台完成的。因此，选择使用部分基于闪存的存储作为网格磁盘，可能不会像您希望的那样带来多大益处，特别是在引入 `WBFC` 之后。幸运的是，重新分配闪存相当容易，因此除非您没有专用的测试环境，否则测试您的具体情况应该不会太困难。然而，由于本章重点介绍 Exadata 智能闪存缓存功能，我们将仅简要介绍在 X5-2 高性能型号之外将 F160 用作网格磁盘的情况。

注意

一个常见的误解是，将在线重做日志放在闪存存储上将显著加快重做日志的写入速度，从而提高高事务系统的吞吐量。我们在 Exadata 系统以及非 Exadata 硬件上都遇到过这种情况。虽然确实基于 SSD 的存储比传统磁盘能更快地处理小型随机写入，但在高事务系统上对重做日志的写入通常不属于此类，并且实际上从存储在 SSD 存储中获益并不大。事实上，这些写入应该直接进入阵列的缓存并快速完成。此外，SSD 写入机制可能导致单个写入时间存在很大差异。可能存在个别写入比平均值长几个数量级的情况，这一事实被称为写入悬崖（write cliff）。这在非常繁忙的系统上也可能导致问题。为了缓解此问题，Oracle 引入了智能闪存日志（Smart Flash Logging），您将在后面读到相关内容。



