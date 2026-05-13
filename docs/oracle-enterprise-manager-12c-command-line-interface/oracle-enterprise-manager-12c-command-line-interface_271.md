# 将当前定义的目标列表更新到当前定义的属性。
myinst.setprops(show=True)
```

现在，可以使用命令 `emcli @changeTargProps.py` 在脚本模式下调用这个完整的脚本。该脚本会将 Enterprise Manager 中 `em12cr3.example.com` 主机目标的生命周期状态属性更新为 `Development`。将示例中的命令复制到 EM CLI 交互式会话中会产生相同的效果。

`updateProps()` 的功能可以根据情况需要简单或复杂。下面的示例展示了使用 `updateProps()` 的其他方法。每个示例的功能通过注释进行了解释。将注释与命令一起包含，允许它们与命令一起复制到脚本中：

```
