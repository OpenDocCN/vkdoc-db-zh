# 第 5 章 与 CockroachDB 交互

我们已经涵盖了很多概念性内容，现在是时候开始作为最终用户使用 CockroachDB 了。我们已经创建了集群和表；现在，让我们连接到它们并开始使用。

### 连接到 CockroachDB

面对一个新的数据库时，你可能首先想做的就是连接到它并开始查询！在本节中，我们将使用工具和代码连接到自托管和基于云的 CockroachDB 集群。

#### 使用工具连接

有很多现成的工具可用于连接到 CockroachDB 集群。有些是免费的，有些则需要付费使用。

以下是一些用于与 CockroachDB 交互的流行现成工具。我将使用 DBeaver Community，但 DataGrip 和 TablePlus 对 CockroachDB 都有极好的支持：

*   **DBeaver** – [`dbeaver.com`](https://dbeaver.com)
*   **DataGrip** – [www.jetbrains.com/datagrip](http://www.jetbrains.com/datagrip)
*   **TablePlus** – [`tableplus.com`](https://tableplus.com)

让我们使用以下命令创建一个 CockroachDB 集群，并使用 DBeaver 连接到它。请注意，没有 `--insecure` 标志，CockroachDB 会生成用户名和密码。我将在 DBeaver 中使用这些。请注意，为简洁起见，我已省略了大部分命令输出：

```bash
$ cockroach demo --no-example-database
```

[²]: 意味着小数点后的位数是固定的。
[³]: [www.cockroachlabs.com/docs/stable/serial.html](http://www.cockroachlabs.com/docs/stable/serial.html)
[⁴]: 2D, 3D 等。
[⁵]: [www.cockroachlabs.com/docs/stable/functions-and-operators.html#built-in-functions](http://www.cockroachlabs.com/docs/stable/functions-and-operators.html#built-in-functions)
[¹]: [www.tpc.org/tpcc](http://www.tpc.org/tpcc)
[²]: [`github.com/brianfrankcooper/YCSB`](https://github.com/brianfrankcooper/YCSB)


