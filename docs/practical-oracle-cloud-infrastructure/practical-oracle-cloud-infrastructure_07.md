#  shape = "VM.Standard2.1"
shape = "VM.Standard2.2"
...
}
```

运行`terraform plan`可以解释如果我选择应用此基础设施代码修改会发生什么：
```
$ terraform plan
...
Terraform will perform the following actions:
-/+ oci_core_instance.vm (new resource required)
...
boot_volume_id: "ocid1.bootvolume.oc1..........w37sra" => 
...
image: "ocid1.image.oc1..........ai3kha" => 
...
shape: "VM.Standard2.1" => "VM.Standard2.2" (forces new resource)
...
Plan: 1 to add, 0 to change, 1 to destroy.
```

**注意**
如果您看到*0 to add, 1 to change, 0 to destroy*，则意味着便捷、全自动且 API 驱动的垂直扩容功能不仅已在 OCI API 中启用，而且在 Terraform Provider 插件中也已启用。在这种情况下，下面描述的问题将不再适用。但是，您仍然可以按照以下描述执行半手动的垂直扩容方法。事实上，它仍然会教您如何分离和重新附加引导卷。

`terraform plan`命令的输出显示*1 to add, 0 to change, 1 to destroy*。嗯，乍一看，您可能认为我们无论如何都期望这个计算实例被终止，并启动一个更强大的新实例。正确。这里的问题是，新实例也将获得一个全新的引导卷，该卷是再次从最新的`CentOS 7`基础操作系统映像构建的。此行为在基础设施代码的`source_details`对象中定义：
```
