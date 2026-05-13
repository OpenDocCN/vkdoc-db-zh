# SQLCLR：架构与设计考量

## SQLCLR 安全与可靠性特性

与存储过程、触发器、UDF（用户定义函数）以及其他可暴露在 SQL Server 中的代码模块不同，特定的 SQLCLR 例程并非直接与数据库关联，而是与数据库内**编录**的程序集相关联。程序集的编录是使用 SQL Server 的 `CREATE ASSEMBLY` 语句完成的，并且与它们的 T-SQL 等价物不同，SQLCLR 模块获得其第一个安全限制并非通过授权，而是在其程序集被编录的同时进行。`CREATE ASSEMBLY` 语句允许数据库管理员或开发人员指定三种安全与可靠性**权限集**之一，这些权限集决定了程序集中的代码被允许做什么。

允许的权限集是 `SAFE`、`EXTERNAL_ACCESS` 和 `UNSAFE`。每个越来越宽松的级别都包含并扩展了较低权限集授予的权限。为 `SAFE` 程序集允许的受限权限集包括对数学和字符串函数的有限访问，以及通过上下文连接对宿主数据库的数据访问。`EXTERNAL_ACCESS` 权限集增加了与 SQL Server 实例外部通信的能力，可连接到其他数据库服务器、文件服务器、Web 服务器等。而 `UNSAFE` 权限集赋予程序集几乎可以做任何事情的能力——包括运行非托管代码。

尽管表面上只是一个用户可控的设置，但在内部，每个权限集的实际权利实际上由两种不同的方法强制执行：

-   分配给每个权限集的程序集通过 .NET 的**代码访问安全**（`Code Access Security`，`CAS`）技术被*授予*执行某些操作的访问权限。
-   同时，根据对一个名为 `HostProtectionAttribute`（`HPA`）的 .NET 3.5 属性的检查，某些操作的访问被*拒绝*。

表面上看，`HPA` 和 `CAS` 的区别在于它们是对立面：`CAS` 权限规定了一个程序集可以做什么，而 `HPA` 权限规定了一个程序集不能做什么。`CAS` 授予的所有内容与 `HPA` 拒绝的所有内容的组合构成了三个权限集中的每一个。

在这个基本区别之外，是这两种访问控制方法之间更重要的区分。尽管违反由任一方法强制执行的权限都会导致运行时异常，但实际的检查是在非常不同的时间进行的。`CAS` 授权在代码执行期间通过堆栈遍历在运行时动态检查。另一方面，`HPA` 权限在即时编译时检查——就在调用被引用的方法*之前*。

要观察这些差异如何影响代码运行方式，需要一些测试用例，这些将在以下部分描述。

## 安全异常

首先，让我们看看 `CAS` 异常是如何工作的。创建一个包含以下 CLR 存储过程的新程序集：

```
[SqlProcedure]
public static void CAS_Exception()
{
    SqlContext.Pipe.Send("Starting...");
    using (FileStream fs = new FileStream(@"c:\b.txt", FileMode.Open))
    {
        //Do nothing...
    }
    SqlContext.Pipe.Send("Finished...");
    return;
}
```

将程序集编录为 `SAFE` 并执行该存储过程。这将导致以下输出：

```
Starting...
Msg 6522, Level 16, State 1, Procedure CAS_Exception, Line 0
A .NET Framework error occurred during execution of user-defined routine or aggregate "CAS_Exception":
System.Security.SecurityException: Request for the permission of type 'System.Security.Permissions.FileIOPermission, mscorlib, Version=2.0.0.0, ...
```


