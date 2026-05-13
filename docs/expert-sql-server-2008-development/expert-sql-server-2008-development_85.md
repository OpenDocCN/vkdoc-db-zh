# SQLCLR：架构与设计考量

## 注意

根据你的数据库是否启用了 `TRUSTWORTHY` 选项，以及你的程序集是否进行了强命名，情况可能并不像我在此暗示的那么简单。如果核心程序集没有正确的权限，本节及下一节中的示例可能会在部署时失败；或者如果你选择使用强命名程序集，也可能在运行时失败。有关更多信息，请参阅本章后面的“授予跨程序集权限”一节。与此同时，如果你正在跟着操作，请在一个启用了 `TRUSTWORTHY` 选项的数据库中工作，并暂时不要进行强命名。

## 使用代码访问安全性权限

受 HPA 保护的资源非常容易封装，因为给定方法的权限是在方法即时编译时检查的。唉，在处理受 CAS 保护的资源时，事情就没那么简单了，因为授权是在运行时通过栈遍历动态检查的。这意味着仅仅引用第二个程序集是不够的——每次都会遍历整个栈，而不考虑程序集边界。

为了说明这个问题，请创建一个包含以下方法的新程序集，该方法读取文本文件的所有行并将其作为字符串集合返回：

```csharp
public static string[] ReadFileLines(string FilePath)
{
    List<string> theLines = new List<string>();
    using (System.IO.StreamReader sr =
        new System.IO.StreamReader(FilePath))
    {
        string line;
        while ((line = sr.ReadLine()) != null)
            theLines.Add(line);
    }
    return (theLines.ToArray());
}
```

将此程序集在 SQL Server 中编目，权限集设为 `EXTERNAL_ACCESS`。现在让我们重新审视本章前面创建的 `CAS_Exception` 存储过程，它包含在一个 `SAFE` 程序集中，当用于访问本地文件资源时会抛出异常。编辑 `CAS_Exception` 程序集以包含对包含 `ReadFileLines` 方法的程序集的引用，并按如下方式修改存储过程：

```csharp
[SqlProcedure]
public static void CAS_Exception()
{
    SqlContext.Pipe.Send("Starting...");
    string[] theLines =
        FileLines.ReadFileLines(@"C:\b.txt");
    SqlContext.Pipe.Send("Finished...");
    return;
}
```

请注意，我将 `ReadFileLines` 方法创建在名为 `FileLines` 的类中；请根据你使用的类名适当引用你的方法。修改完成后，重新部署外部程序集，并确保它被编目为 `SAFE`。

运行此存储过程的修改版本，你会发现即使跨越了程序集边界，你仍然会收到与之前相同的异常。CAS 授权并没有因为引用了更高权限的程序集而改变，这是因为栈遍历不考虑被引用程序集所持有的权限。

要解决此问题，需要在被引用的程序集中控制栈遍历。

由于该程序集有足够的权限执行文件操作，它可以在内部要求栈遍历停止检查文件 I/O 权限，即使从另一个没有相应权限的程序集中调用也是如此。这是通过使用 .NET 的 `System.Security` 命名空间中公开的 `IStackWalk` 接口的 `Assert` 方法来完成的。

再次查看前面显示的 CAS 违规，注意所需的权限是 `FileIOPermission`，它位于 `System.Security.Permissions` 命名空间中。`FileIOPermission` 类——连同该命名空间中的其他“权限”类——实现了 `IStackWalk` 接口。为避免 CAS 异常，只需实例化 `FileIOPermission` 类的一个实例并调用 `Assert` 方法。以下代码是使用此技术的 `ReadFileLines` 方法的修改版本：

```csharp
public static string[] ReadFileLines(string FilePath)
{
    //Assert that anything File IO-related that this
    //assembly has permission to do, callers can do
    FileIOPermission fp = new FileIOPermission(
        PermissionState.Unrestricted);
    fp.Assert();

    List<string> theLines = new List<string>();
    using (System.IO.StreamReader sr =
        new System.IO.StreamReader(FilePath))
    {
        string line;
        while ((line = sr.ReadLine()) != null)
            theLines.Add(line);
    }
    return (theLines.ToArray());
}
```

此版本的方法使用 `PermissionState.Unrestricted` 枚举实例化了 `FileIOPermission` 类，从而使所有调用者能够执行该程序集有权执行的任何文件 I/O 相关活动。在此上下文中使用术语“无限制”并不像听起来那么危险；访问是无限制的，其含义是权限只允许调用者访问该程序集本身已拥有的文件系统访问级别。在此处进行修改并重新部署两个程序集后，CAS 异常将不再是问题。

为了让你能在更细的粒度上控制，`FileIOPermission` 类公开了其他具有不同选项的构造函数重载。对于本示例最有用的一个是使用名为 `FileIOPermissionAccess` 的枚举结合文件路径，允许你将授予调用者的权限限制为仅对指定文件的特定操作。例如，要限制访问，使调用者只能读取本示例中指定的文件，请使用以下构造函数：

```csharp
FileIOPermission fp = new FileIOPermission(
    FileIOPermissionAccess.Read,
    "C:\b.txt");
```

文件 I/O 只是你可能遇到 CAS 异常的多种权限之一。重要的是能够识别这个模式。在所有情况下，违规都会抛出 `SecurityException`，并引用 `System.Security.Permissions` 命名空间中的一个权限类。

每个类都遵循这里概述的相同基本模式，因此你应该能够轻松使用此技术来设计任意数量的权限提升解决方案。

### 授予跨程序集权限

前面章节中的示例进行了一些简化，以便一次将重点集中在一个问题上。在处理跨程序集调用时，还有另外两个问题需要关注：数据库可信度和强命名。

### 数据库可信度

“可信”数据库的概念是微软近年来对安全问题认识提高的直接产物。将数据库标记为可信只需使用 `ALTER DATABASE` 设置一个选项即可：


