# compute.tf
resource "oci_core_instance" "vm" {
...
  source_details {
    source_id = data.oci_core_images.centos_image.images[0].id
    source_type = "image"
  }
...
}
```


在 `source_details` 对象内，`source_type` 仍然表明需要基于 `source_id` 属性指定的镜像来创建一个新的启动卷。

我希望你现在能理解，为什么基础设施代码文件中的简单形状替换不会起作用，甚至可能是有害的，因为任何保留在根文件系统中的应用状态都将丢失。谈到根文件系统中的应用状态，`cloud-config` 文件不仅添加了一个新的基于 `stress-ng` 的 `systemd` 服务，还创建了一个包含初始启动时间戳的小型文本文件。我们将使用这个时间戳来验证在垂直扩展实例后，我们是否仍在使用同一个启动卷。首先，连接到该实例并记录这个小型文件的内容：

```
$ ssh -i ~/.ssh/oci_id_rsa opc@$INSTANCE_PUBLIC_IP
[opc@vm-1-ocpu ~]$ cat datemarker
Sat Apr 27 17:01:34 GMT 2019
[opc@vm-1-ocpu ~]$ exit
```

## 要垂直扩展实例，你需要执行以下步骤

1.  停止旧实例，同时保留启动卷。

2.  分离启动卷。

3.  使用分离出来的卷创建新实例。

### 调整 Terraform 基础设施代码

我们将调整 Terraform 基础设施代码，以将实例转移到停止状态。如果你使用的是 Linux 或 Windows Subsystem for Linux，请使用以下 `sed` 命令来取消 `chapter06/3-instance-scale-up` 目录中 `compute.tf` 文件内当前被注释掉的代码块：

```
$ pwd
/Users/michal/git/oci-book/chapter06/3-instance-scale-up/infrastructure
$ sed -i 's/\/\*//; s/\*\///' compute.tf
```

如果你使用的是 macOS，请使用以下 `sed` 命令：

```
$ sed -i '.bak' -e 's/\/\*//; s/\*\///' compute.tf
```

这将通过移除 `compute.tf` 文件中所有的块注释标记（`/*` 和 `*/`）来取消注释代码，如清单 6-13 所示。

```
resource "oci_core_instance" "vm" {
...
...
