# SQLCLR 架构与设计考量：通过程序集引用实现选择性权限提升

## 手动代码审查的有效性疑问
首先是对**手动代码审查有效性**的疑问。对于任何代码变更，每次都进行严格的审查是否足以确保代码不会引发问题？如果代码被标记为`SAFE`，引擎本应能检测到这些问题。此外，你真的*希望*每次任何变更部署前都必须进行严格审查吗？在接下来的部分，我将展示如何通过利用`SQLCLR`环境中的程序集依赖关系来消除许多此类问题。

## 通过程序集引用实现选择性权限提升
在理想情况下，`SQLCLR`模块的权限可以像第 5 章描述的`T-SQL`模块权限那样工作：外部模块被授予尽可能少的权限，但能够为了执行某些需要更多访问权限的操作而选择性地、临时地提升其权限。这将显著减少特权暴露区域，这意味着对于构成给定系统大部分代码的外层（权限较低）模块层，进行严格安全审查的必要性会降低——引擎将确保它们的行为符合预期。

解决此问题的通用方案是根据权限需求将代码拆分到不同的程序集中，但不能忽视维护开销和重用性。

例如，考虑上一节提到的需要从文本文件中读取几行的 5000 行模块。可以授予整个模块足够高的权限来读取文件，也可以将读取文件的代码提取出来放到它自己的程序集中。这个外部程序集将暴露一个方法，该方法以文件名作为输入并返回一个行的集合。正如我将在后续章节中展示的，此解决方案可以让你将大部分代码编目为`SAFE`，同时仍然执行文件 I/O 操作。此外，未来需要从文本文件读取行的模块可以引用同一个程序集，因此不必重新实现此逻辑。

可惜的是，封装过程并不像创建一个包含必要逻辑的新程序集并引用它那样简单。由于`CAS`（代码访问安全）和`HPA`（宿主保护属性）异常行为的差异，你可能需要进行一些代码分析，以便正确封装内部模块的权限。在接下来的部分中，我将分别介绍每种权限类型，以说明如何设计解决方案。

## 使用宿主保护权限
一个相当常见的`SQLCLR`模式是创建可以被调用者共享的静态集合。然而，与任何共享数据集一样，如果需要在初始加载后更新部分数据，适当的同步是至关重要的。从`SQLCLR`的角度来看，这变得棘手，因为线程和同步需要`UNSAFE`访问权限——授予如此开放级别的权限并非小事。

考虑一个可能使用静态集合的场景示例：一个用于基于汇率计算货币转换的`SQLCLR`用户定义函数：

```csharp
[SqlFunction]
public static SqlDecimal GetConvertedAmount(
    SqlDecimal InputAmount,
    SqlString InCurrency,
    SqlString OutCurrency)
{
    //Convert the input amount to the base
    decimal BaseAmount =
        GetRate(InCurrency.Value) *
        InputAmount.Value;
    //Return the converted base amount
    return (new SqlDecimal(
        GetRate(OutCurrency.Value) * BaseAmount));
}
```

`GetConvertedAmount`方法内部使用了另一个方法`GetRate`：

```csharp
private static decimal GetRate(string Currency)
{
    decimal theRate;
    rwl.AcquireReaderLock(100);
    try
    {
        theRate = rates[Currency];
    }
    finally
    {
        rwl.ReleaseLock();
    }
    return (theRate);
}
```

`GetRate`在`rates`（一个`Dictionary<string, decimal>`的静态泛型实例）中执行查找。此集合包含系统中给定货币的汇率。为了防止另一个线程恰好正在更新`rates`时会发生的问题，同步是使用`rwl`（一个`ReaderWriterLock`的静态实例）来处理的。字典和`ReaderWriterLock`都在类的方法首次被调用时实例化，并且都被标记为`readonly`，以避免在实例化后被覆盖：

```csharp
static readonly Dictionary<string, decimal>
    rates = new Dictionary<string, decimal>();
static readonly ReaderWriterLock
    rwl = new ReaderWriterLock();
```

如果使用`SAFE`或`EXTERNAL_ACCESS`权限集进行编目，此代码会因使用同步机制（`ReaderWriterLock`）而失败，运行它会产生`HostProtectionException`。解决方案是将受影响的代码移动到它自己的程序集中，并编目为`UNSAFE`。因为宿主保护检查是在程序集中方法即时编译时评估的，而不是在方法运行动态进行的，检查是在跨越程序集边界时完成的。这意味着外部方法可以被标记为`SAFE`，并通过调用`UNSAFE`核心来临时提升其权限。

> **注意**：你可能会对这个示例的有效性产生疑问，因为使用纯`T-SQL`可以轻松实现此系统，从而彻底消除权限问题。我确实认为这是一个现实的例子，特别是如果系统需要在任何给定日期进行大量货币转换。`SQLCLR`代码即使对于简单的数学工作通常也优于`T-SQL`，并且将数据缓存在共享集合中而不是每次调用都从数据库读取，是一个巨大的效率提升。我确信这个解决方案会轻松优于任何纯`T-SQL`等效方案。

在设计`UNSAFE`程序集时，从重用的角度仔细分析应该公开哪些功能是很重要的。在此情况下，导致问题的并不是字典的使用——抛出实际异常的是通过`ReaderWriterLock`的同步。然而，仅仅围绕`ReaderWriterLock`包装一个方法可能不会促进太多重用。我认为更好的策略是将`Dictionary`和`ReaderWriterLock`包装在一起，创建一个新的`ThreadSafeDictionary`类。这个类可以在任何需要共享数据缓存的场景中使用。

以下是我的`ThreadSafeDictionary`实现；我没有实现泛型`Dictionary`类公开的所有方法，而只实现了我常用的那些——即`Add`、`Remove`和`ContainsKey`：

```csharp
using System;
using System.Collections.Generic;
using System.Text;
using System.Threading;

namespace SafeDictionary
{
    public class ThreadSafeDictionary<K, V>
    {
        private readonly Dictionary<K, V> dict = new Dictionary<K,V>();
        private readonly ReaderWriterLock theLock = new ReaderWriterLock();

        public void Add(K key, V value)
        {
            theLock.AcquireWriterLock(2000);
            try
            {
                dict.Add(key, value);
            }
            finally
            {
                theLock.ReleaseLock();
            }
        }

        public V this[K key]
        {
            get
            {
                theLock.AcquireReaderLock(2000);
                try
                {
                    return (this.dict[key]);
                }
                finally
                {
                    theLock.ReleaseLock();
                }
            }
            set
            {
                theLock.AcquireWriterLock(2000);
                try
                {
                    dict[key] = value;
                }
                finally
                {
                    theLock.ReleaseLock();
                }
            }
        }

        public bool Remove(K key)
        {
            theLock.AcquireWriterLock(2000);
            try
            {
                return (dict.Remove(key));
            }
            finally
            {
                theLock.ReleaseLock();
            }
        }

        public bool ContainsKey(K key)
        {
            theLock.AcquireReaderLock(2000);
            try
            {
                return (dict.ContainsKey(key));
            }
            finally
            {
                theLock.ReleaseLock();
            }
        }
    }
}
```


