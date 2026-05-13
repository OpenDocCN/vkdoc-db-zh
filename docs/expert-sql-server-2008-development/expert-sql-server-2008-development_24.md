# CHAPTER „ 面向数据库世界的软件开发方法论

这本身不算大问题，但请设想一下该方法缺失了多少基础架构（例如错误处理代码）。可以合理推测，在这个银行应用程序的其他部分，很可能也存在名为 `Withdraw` 和 `Deposit` 的方法，它们执行完全相同的操作，同样需要这些基础架构代码。`TransferFunds` 方法之所以呈现**弱内聚性**，是因为在执行转账操作时，它需要的功能与单独的 `Withdraw` 和 `Deposit` 方法所提供的功能相同，却使用了完全不同的代码。

一个内聚性更强的同一方法可能类似于以下形式：

```
bool TransferFunds(
Account AccountFrom,
Account AccountTo,
decimal Amount)
{
bool success = false;
success = Withdraw(AccountFrom, Amount);
if (!success)
return(false);
success = Deposit(AccountTo, Amount);
if (!success)
return(false);
else
return(true);
}
```

尽管我已经指出这类生产环境代码会缺少基础异常处理和其他结构，但必须强调，主要缺失的是某种形式的**事务**处理。如果取款成功，但随后的存款失败，当前代码将导致资金实际上凭空消失。务必仔细测试你的关键任务代码是否具有**原子性**；要么全部成功，要么全部不成功。没有中间地带——尤其是在处理人们的资金时！

### 封装

在本节讨论的三个主题中，对于数据库开发人员来说，**封装**可能是最重要的一个。回顾一下那个内聚性更强的 `TransferFunds` 方法，并思考一下与之关联的 `Withdraw` 方法可能是什么样子——或许是这样的：

```
bool Withdraw(Account AccountFrom, decimal Amount)
{
if (AccountFrom.Balance >= Amount)
{
AccountFrom.Balance -= Amount;
return(true);
}
else
return(false);
}
```

在这个例子中，`Account` 类公开了一个名为 `Balance` 的属性，`Withdraw` 方法可以操作它。但是，如果 `Withdraw` 方法中存在错误，并且某条代码路径允许在未首先检查资金是否充足的情况下就操纵 `Balance`，该怎么办？为了避免这种情况，就不应该允许从 `Withdraw` 方法直接设置 `Balance` 的值。相反，`Account` 类应该定义自己的 `Withdraw` 方法。通过这样做，该类将在内部控制其数据和规则——而不必依赖任何外部使用者来正确执行。这里的关键目标是**将逻辑精确实现一次，并根据需要重复使用多次**，而不是在每次需要使用该逻辑的地方都重新编码。

### 接口

应用程序中模块的唯一目的，就是根据使用者（即另一个模块或系统）的请求执行某些操作。例如，如果无法存储或检索数据，数据库系统将毫无价值。因此，一个系统必须公开**接口**——其他模块可用于提出请求的、广为人知的方法和属性。模块的接口是通往其功能的网关，也是决定什么信息进入或离开模块的仲裁者。

接口设计是耦合和封装概念真正获得意义的地方。如果一个接口未能封装足够多的模块内部设计，使用者可能不得不依赖一些关于该模块的内部知识，从而与该模块**紧密耦合**。在这种情况下，模块内部实现的任何更改都可能需要修改使用者的实现。

