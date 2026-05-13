# 5. `Exadata 智能闪存缓存`

Oracle 营销部门一定非常喜欢“智能”这个词。他们已将其应用于 `Exadata` 平台上的众多不同功能。他们似乎也喜欢“闪存”这个词，这个词也与至少六项功能相关。更令人困惑的是，`Oracle 数据库 12c 第 1 版` 中有两个功能的名称几乎完全相同——`数据库智能闪存缓存 (DBFC)` 和 `Exadata 智能闪存缓存 (ESFC)`。虽然这两种功能都利用了基于闪存的内存设备，但它们非常不同。在本章中，我们将重点讨论 `ESFC` 和 `OLTP` 优化，并仅简单提及 `DBFC`。

`Exadata V2`（第一个配备 `ESFC` 的版本）的初始目标之一是扩展 `Exadata` 的能力，以提高其在 `OLTP` 工作负载下的性能。为了实现这一目标，`ESFC` 是添加到 `V2` 配置中的关键组件。基于 `ESFC` 的功能——例如 `Exadata 智能闪存日志 (ESFL)` 以及后来的 `回写闪存缓存 (WBFC)`，还有闪存缓存压缩等——都在后续版本中被引入。除了从 `V2` 升级到 `X2` 的那一步，每一代新的 `Exadata` 硬件都有更多的闪存可用。在撰写本文时，当前的硬件代次是 `Exadata X5-2`，在一个全机架、配备高容量存储服务器的情况下，您可以拥有大约 `90TB` 的缓存。在全闪存、`X5-2` 高性能（`HP`）存储节点上，闪存缓存的处理方式略有不同。您可以稍后在本章中详细了解。重要的是要理解，该缓存由 `Oracle` 软件管理，并且该软件知晓存储层所支持的数据库如何使用数据。`Oracle` 在有效管理数据库缓存的软件方面已经研发了超过 30 年。由于存储软件知道数据库正在请求什么，因此它比传统存储阵列更清楚哪些数据应该或不应该被缓存。

## `DBFC` 与 `ESFC`

`DBFC` 和 `ESFC` 是两种完全不同的东西。`DBFC` 是数据库服务器上缓冲区缓存（`buffer cache`）的扩展。它是 `11g` 的标准组件，作为实例的二级（`tier 2`）缓冲区缓存实现。它仅在 `Solaris` 和 `Oracle Linux` 上受支持。它通过将闪存磁盘（`SSD`）添加到数据库服务器并告诉单个实例使用它来启用。如果用户需要一个不在缓冲区缓存中的数据块，它会在请求 `I/O` 之前先查看 `DBFC` 中是否存在该块。当数据块从缓冲池中老化时，它们会被移至 `DBFC`，而不是简单地被刷新。`ESFC` 当然是 `Exadata` 存储服务器上的磁盘缓存。它为访问存储单元（`storage cell`）的所有实例缓存数据。

### 硬件

在当前的 `Exadata` 代次发布之前，描述硬件一直更容易。从 `V2` 开始直到（但不包括）`X5-2`，每台 `Exadata` 存储服务器都可以用相同的方式描述。每个单元（`cell`）都有四块用于智能闪存缓存的 `PCI Express` 卡，加上 12 块硬盘，无论它是高性能（`HP`）还是高容量（`HC`）型号。对于 `X5` 系列，关于闪存卡的数量，您需要区分 `HC` 和 `HP` 单元。后者（`HP`）在 `Exadata` 产品历史上首次不再配备任何硬盘，而是配备了 8 块下一代 `PCIe` 卡，每个单元的原始容量为 `12.8TB`。由于 `X5-2` 高性能存储服务器没有任何用于数据库存储的硬盘，闪存设备除了作为 `RDBMS` 层 `I/O` 请求的存储层缓存外，还将被用于持久化存储数据库文件。

每台 `Exadata X5-2` 高容量存储服务器有四块 `Flash Accelerator PCIe` 卡，每块原始容量为 `1.6TB`，每个单元总原始空间为 `6.4TB`。除了所用闪存设备容量更大外，`HC` 型号遵循与早期硬件代次相同的设计原则。

查看每块卡的原始存储空间数字，您需要知道这些是规格中引用的数字。实际可用的容量略少。卡硬件设计随时间而变化，篇幅允许我们本章只讨论 `X4` 和 `X5` 代次。在适用的情况下，会提及早期的硬件代次。对于本章，重点将放在 `HC` 单元上，因为这可能是两种硬件选择中更常见的一种。

### Exadata X4-2 存储服务器中的闪存

您刚刚读到，X4-2 存储服务器的高容量和高性能型号中的闪存是相同的：每个型号都有四块卡。它们作为 PCI Express (版本 2) 扩展卡连接。这些卡所在的 PCIe 插槽每个有八条通道。Oracle 将这些闪卡称为 `F80`，或者更准确地说，是 Sun Flash Accelerator F80 PCIe 卡。其可用容量很容易记住，为 800GB——产品名称中已有所暗示。在内部，该卡由四个各为 200 GB 的所谓闪存模块或 `FMODs` 组成。因此，单个 `F80` 卡在操作系统中会呈现为四个独立的块设备。您可以在 X4-2 cell 上执行的 `lsscsi` 命令输出中看到这一点：

```
[root@enkx4cel01 ∼]# lsscsi
[0:0:0:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sda
[0:0:1:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdb
[0:0:2:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdc
[0:0:3:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdd
[1:0:0:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sde
[1:0:1:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdf
[1:0:2:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdg
[1:0:3:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdh
[2:0:0:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdi
[2:0:1:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdj
[2:0:2:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdk
[2:0:3:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdl
[3:0:0:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdm
[3:0:1:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdn
[3:0:2:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdo
[3:0:3:0]    disk    ATA      2E256-TU2-510B00 UIO3  /dev/sdp
[4:0:20:0]   enclosu ORACLE   CONCORD14        0d03  -
[4:2:0:0]    disk    LSI      MR9261-8i        2.13  /dev/sdq
[4:2:1:0]    disk    LSI      MR9261-8i        2.13  /dev/sdr
[4:2:2:0]    disk    LSI      MR9261-8i        2.13  /dev/sds
[4:2:3:0]    disk    LSI      MR9261-8i        2.13  /dev/sdt
[4:2:4:0]    disk    LSI      MR9261-8i        2.13  /dev/sdu
[4:2:5:0]    disk    LSI      MR9261-8i        2.13  /dev/sdv
[4:2:6:0]    disk    LSI      MR9261-8i        2.13  /dev/sdw
[4:2:7:0]    disk    LSI      MR9261-8i        2.13  /dev/sdx
[4:2:8:0]    disk    LSI      MR9261-8i        2.13  /dev/sdy
[4:2:9:0]    disk    LSI      MR9261-8i        2.13  /dev/sdz
[4:2:10:0]   disk    LSI      MR9261-8i        2.13  /dev/sdaa
[4:2:11:0]   disk    LSI      MR9261-8i        2.13  /dev/sdab
[11:0:0:0]   disk    ORACLE   UNIGEN-UFD       PMAP  /dev/sdac
[root@enkx4cel01 ∼]#
```

前面的设备就是 `FMODs`，每卡四个。例如，第一张卡的模块地址是从 `0:0:0:0` 到 `0:0:0:3`。如果您想了解所有细节，需要查看 `dmesg` 的输出。使用 `egrep` 进行不区分大小写的匹配，查找 "`sas|scsi`"，将向您展示这些设备如何在引导阶段在 `mpt2sas` 内核模块（"Fusion MPT SAS Host"）下出现。这与 `cellcli` 的输出相吻合：

```
CellCLI>  list physicaldisk attributes name,deviceName,diskType,makeModel -
> where deviceName like '/dev/sd[a-p]'
FLASH_1_0          /dev/sdi          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_1_1          /dev/sdj          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_1_2          /dev/sdk          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_1_3          /dev/sdl          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_2_0          /dev/sdm          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_2_1          /dev/sdn          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_2_2          /dev/sdo          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_2_3          /dev/sdp          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_4_0          /dev/sde          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_4_1          /dev/sdf          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_4_2          /dev/sdg          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_4_3          /dev/sdh          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_5_0          /dev/sda          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_5_1          /dev/sdb          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_5_2          /dev/sdc          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
FLASH_5_3          /dev/sdd          FlashDisk          "Sun Flash Accelerator F80 PCIe Card"
```

内核将这些卡作为常规块设备进行寻址，就像自 IDE 磁盘时代以来的旋转硬盘一样。在 Linux 中，它们与任何其他块设备一样可见。实际上，以 `/dev/sdq` 开头的设备是常规的旋转硬盘。这完全没有问题；在引入非易失性内存主机控制器接口规范 (`NVMHCI`) 或简称 `NVMe` 之前，这在许多设备中一直是直接连接的 NAND 闪存的标准寻址方式。您可以在下一节中阅读到的 X5-2 存储服务器上找到这些（NVMe）设备。


