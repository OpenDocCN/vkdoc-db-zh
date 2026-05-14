# 第五章 使用 Node.js 操作 MongoDB

传统的基于脚本语言的 Web 应用程序构建在客户端/服务器模型上，需要客户端脚本语言（如 JavaScript）和服务器端脚本语言（如 PHP），并利用 Web 服务器。Node.js 的不同之处在于，它是基于 V8 JavaScript 引擎构建的服务器端脚本。V8 是 Google 用于 Google Chrome 的开源 JavaScript 引擎。Node.js 基于事件驱动模型，用于开发快速、可扩展、数据密集型、实时网络应用程序。在本章中，我们将讨论使用 Node.js（或简称 Node）访问 MongoDB，以及使用 MongoDB 的 Node.js 驱动在数据库中进行数据修改。本章涵盖以下主题：

*   入门
*   使用连接
*   使用文档

## 入门

在以下小节中，我们将介绍 MongoDB 的 Node.js 驱动并设置环境。

### Node.js 驱动概述

MongoDB 的 Node.js 驱动提供了一个 API，用于连接到 MongoDB 服务器并在服务器上执行不同的操作，例如添加文档或查找文档。Node.js 驱动中的主要类包括...



