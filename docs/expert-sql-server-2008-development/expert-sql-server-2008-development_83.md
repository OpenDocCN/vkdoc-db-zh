# SQLCLR：架构与设计注意事项

## 安全异常示例

`Culture=neutral, PublicKeyToken=b77a5c561934e089` 失败。

```
System.Security.SecurityException:
   at System.Security.CodeAccessSecurityEngine.Check(Object demand,
   StackCrawlMark& stackMark, Boolean isPermSet)
   at System.Security.CodeAccessPermission.Demand()
   at System.IO.FileStream.Init(String path, FileMode mode, FileAccess access, Int32 rights, Boolean useRights, FileShare share, Int32 bufferSize, FileOptions options, SECURITY_ATTRIBUTES secAttrs, String msgPath, Boolean bFromProxy)
   at System.IO.FileStream..ctor(String path, FileMode mode)
   at udf_part2.CAS_Exception()
```

此情况下引发的异常是 `SecurityException`，表明这是一个 CAS 违规（`FileIOPermission` 类型）。但发生的不仅是异常；注意输出的第一行是字符串 `"Starting..."`，这是由存储过程第一行使用的 `SqlPipe.Send` 方法输出的。因此在遇到异常之前，方法已被进入，代码执行成功，直到尝试实际的权限违规。

`注意` 文件 I/O 是访问资源（本地或其他）的一个好例子，在上下文连接中是不允许的。要使用 SQLCLR 安全桶避免此特定违规，需要使用 `EXTERNAL_ACCESS` 权限对程序集进行编录。

## 主机保护异常

为了了解 HPA 异常的行为，让我们重复上一节中描述的相同实验，但使用以下存储过程（同样编录为 `SAFE`）：

```csharp
[SqlProcedure]
public static void HPA_Exception()
{
    SqlContext.Pipe.Send("Starting...");
    // 下一行将抛出 HPA 异常...
    Monitor.Enter(SqlContext.Pipe);
    // 释放锁（如果代码真的能执行到这里）...
    Monitor.Exit(SqlContext.Pipe);
    SqlContext.Pipe.Send("Finished...");
    return;
}
```

就像之前一样，发生了一个异常。但这次，输出有点不同：

```
Msg 6522, Level 16, State 1, Procedure HPA_Exception, Line 0
A .NET Framework error occurred during execution of user-defined routine or aggregate "HPA_Exception":
System.Security.HostProtectionException: Attempted to perform an operation that was forbidden by the CLR host.
The protected resources (only available with full trust) were: All
The demanded resources were: Synchronization, ExternalThreading
System.Security.HostProtectionException:
   at System.Security.CodeAccessSecurityEngine.ThrowSecurityException(Assembly asm, PermissionSet granted, PermissionSet refused, RuntimeMethodHandle rmh, SecurityAction action, Object demand, IPermission permThatFailed)
   at System.Security.CodeAccessSecurityEngine.ThrowSecurityException(Object assemblyOrString, PermissionSet granted, PermissionSet refused, RuntimeMethodHandle rmh, SecurityAction action, Object demand, IPermission permThatFailed)
   at System.Security.CodeAccessSecurityEngine.CheckSetHelper(PermissionSet grants, PermissionSet refused, PermissionSet demands, RuntimeMethodHandle rmh, Object assemblyOrString, SecurityAction action, Boolean throwException)
   at System.Security.CodeAccessSecurityEngine.CheckSetHelper(CompressedStack cs, PermissionSet grants, PermissionSet refused, PermissionSet demands, RuntimeMethodHandle rmh, Assembly asm, SecurityAction action)
   at udf_part2.HPA_Exception()
```

与执行 `CAS_Exception` 存储过程时不同，这次我们没有看到 `"Starting..."` 消息，表明在遇到异常之前 `SqlPipe.Send` 方法未被调用。事实上，在代码执行阶段 `HPA_Exception` 方法根本没有被进入（你可以通过尝试在函数内设置断点并在 Visual Studio 中启动调试会话来验证这一点）。断点无法命中的原因是权限检查已执行，并且在即时编译后立即抛出了异常。

你还应注意，异常的措辞与之前的情况不同。CAS 异常的措辞相当温和：“请求权限 ... 失败。”另一方面，HPA 异常带有更严厉的警告：“尝试执行被禁止的操作。”这种措辞上的差异并非偶然。CAS 授权关注的是安全性——防止代码访问某些受保护的内容，因为它不应该拥有访问权限。而 HPA 权限关注的是服务器可靠性，并保持 CLR 主机平稳高效地运行。线程和同步被认为可能威胁可靠性，因此仅限于标记为 `UNSAFE` 的程序集。

`注意` 使用 .NET 反编译器（例如 Red Gate Reflector, www.red-gate.com/products/reflector/），可以探索基类库以查看各种类和方法分配了哪些 HPA 属性。例如，`Monitor` 类被装饰了以下控制主机访问的属性：
`[ComVisible(true), HostProtection(SecurityAction.LinkDemand, Synchronization=true, ExternalThreading=true)]`。

关于根据 CAS 和 HPA 模型允许什么和不允许什么的完整列表超出了本章的范围，但 Microsoft 有很好的文档记录。请参阅以下 MSDN 主题：
- `主机保护属性与 CLR 集成编程` (http://msdn2.microsoft.com/en-us/library/ms403276.aspx)
- `CLR 集成代码访问安全性` (http://msdn2.microsoft.com/en-us/library/ms345101.aspx)

### 对代码安全性的追求

你可能想知道，既然修复异常如此简单——只需将程序集的权限级别提高到 `EXTERNAL_ACCESS` 或 `UNSAFE`，并让代码访问它需要做的事情——为什么我还要介绍 SQLCLR 权限集的内部结构以及它们的异常有何不同。事实是，提高权限级别肯定会起作用，但这样做你可能是在规避安全策略，而不是与之合作以使你的系统更安全。

如前一节所述，代码访问权限是在程序集级别而不是方法或行级别授予的。因此，为了使某个模块工作而提高给定程序集的权限，实际上会影响该程序集中包含的许多不同模块，赋予它们所有增强的访问权限。在程序集内的多个模块上授予额外权限反过来会造成维护负担：如果你希望确保没有安全问题，你必须审查程序集中每个模块的每一行代码，以确保它没有做任何不应该做的事情——你不能再信任引擎为你进行检查。

你现在可能在想解决方案很简单：拆分你的方法，使每个方法都位于一个独立的程序集中，然后按那种方式授予权限。这样每个方法确实将拥有自己的权限集。但即使在那种情况下，权限可能仍然不够细化以避免代码审查的噩梦。考虑一个复杂的 5000 行模块，它需要一个单一的文件 I/O 操作来从文本文件中读取一些行。通过给予整个模块 `EXTERNAL_ACCESS` 权限，它现在可以从该文件读取行。但当然，你仍然必须检查剩余的 4999 行代码，以确保它们没有做任何未经授权的事情。


