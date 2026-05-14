# 从附加到 VM 的虚拟硬盘检索 URI
$VHDs = $VM.StorageProfile.DataDisks.VirtualHardDisk.Uri
