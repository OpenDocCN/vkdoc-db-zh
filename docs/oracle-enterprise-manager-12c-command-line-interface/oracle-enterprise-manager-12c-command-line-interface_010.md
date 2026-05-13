# 第 2 章

![image](img/frontdot.jpg)

## 安全框架、安装与 EM12c 第 4 版

既然您已经了解了 Enterprise Manager 架构，您可能想了解更多关于 Oracle Management Service (OMS) 主机的高级安装方法或远程安装。您还需要了解与 12.1.0.4（也称为 EM12c 第 4 版）相关的新动词（verbs），这些内容都将在本章中涵盖。

## EM CLI 与 WebLogic 安装

Enterprise Manager 作为 WebLogic 服务器 (WLS) 上的一个域运行。没有 WebLogic 提供的中层架构，云生命周期解决方案就无法存在。`WLS` 处理业务逻辑，同时与 Web 服务和其他远程处理通信，以确保前端事务从开始到结束都能完成。

曾经，`OMS` 需要单独安装 `WLS`。尽管它目前是 EM12c 安装过程中的一个自动化步骤，但理解如何手动执行此操作对于管理员来说很有价值，尤其是在向前发展过程中管理和维护环境时。


