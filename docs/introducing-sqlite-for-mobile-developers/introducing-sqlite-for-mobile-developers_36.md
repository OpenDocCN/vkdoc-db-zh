# Core Data 基础设置

以下代码展示了应用程序如何设置 Core Data 的核心组件。

## 准备 Core Data 存储目录

这是应用程序用于存储 Core Data 数据库文件的目录。
```swift
// 应用程序用于存储 Core Data 存储文件的目录。
// 此代码使用应用程序文档 Application Support 目录中名为 "com.champlainarts.SingleViewCoreDataSwift" 的目录。
let urls = NSFileManager.defaultManager().URLsForDirectory(.DocumentDirectory, inDomains: .UserDomainMask)
return urls[urls.count-1]
}()
```

## 在 iOS 中创建 managedObjectModel

这里的 `managedObjectModel` 代码按需创建。加粗显示的行是在使用模板时生成的。如果更改了项目名称，请相应地修改此行代码。另外请注意，默认情况下，Core Data 模型存储在应用包中。你使用 Xcode Core Data 模型编辑器构建的 `.xcdatamodeld` 文件在编译过程中会被编译成 `.momd` 文件。
```swift
lazy var managedObjectModel: NSManagedObjectModel = {
    // 应用程序的托管对象模型。此属性不是可选的。
    // 如果应用程序无法找到并加载其模型，将是一个致命错误。
    let modelURL = NSBundle.mainBundle().URLForResource("SingleViewCoreDataSwift", withExtension: "momd")!
    return NSManagedObjectModel(contentsOfURL: modelURL)!
}()
```

## 在 iOS 中创建 persistentStoreCoordinator

这段代码基于你的托管对象模型 (`managedObjectModel`) 创建一个持久化存储协调器。
加粗的第二行是根据你的项目名称生成的，它包含了 SQLite 数据库文件的路径。你通常不会移动此文件，但如果你更改了项目名称，这里可能是另一个需要修改文件名的地方。
请注意，在使用数据模型时，如果 `self.managedObjectModel` 尚不存在，它将在此处被引用后创建。
另外，需要特别指出的是选择 `SQLiteStoreType` 作为新持久化存储的类型，这在第三个加粗的行中显示。
```swift
lazy var persistentStoreCoordinator: NSPersistentStoreCoordinator = {
    // 应用程序的持久化存储协调器。此实现
    // 创建并返回一个协调器，同时为该应用程序添加了存储。
    // 此属性是可选的，因为存在可能导致存储创建失败的
    // 合法错误条件。
    // 创建协调器和存储
    let coordinator = NSPersistentStoreCoordinator(managedObjectModel: self.managedObjectModel)
    let url = self.applicationDocumentsDirectory.URLByAppendingPathComponent("SingleViewCoreData.sqlite")
    var failureReason = "创建或加载应用程序保存的数据时出错。"
    do {
        try coordinator.addPersistentStoreWithType(NSSQLiteStoreType, configuration: nil, URL: url, options: nil)
    } catch {
        // 报告我们获得的任何错误。
        var dict = [String: AnyObject]()
        dict[NSLocalizedDescriptionKey] = "初始化应用程序保存的数据失败"
        dict[NSLocalizedFailureReasonErrorKey] = failureReason
        dict[NSUnderlyingErrorKey] = error as NSError
        let wrappedError = NSError(domain: "YOUR_ERROR_DOMAIN", code: 9999, userInfo: dict)
        // 请替换此部分，以适当的方式处理错误。
        // abort() 会导致应用程序生成崩溃日志
        // 并终止。你不应在正式发布的应用程序中使用此函数，
        // 尽管它在开发过程中可能很有用。
        NSLog("未解决的错误 \(wrappedError), \(wrappedError.userInfo)")
        abort()
    }
    return coordinator
}()
```

## 解释：do/try/catch 代码块

如果你不熟悉 Swift 2，可能没见过 `do/try/catch` 代码块。这在现代 Swift 代码中越来越常用，因此这段代码可以作为你将来编写其他 `do/try/catch` 代码的模板。其核心在于 `catch` 部分。该代码的重要部分如下：

1.  捕获错误。
2.  创建一个空字典（这里是 `var dict`）。
3.  填充字典。这段代码将根据你的目的进行定制。
4.  使用你在第 2 和第 3 步创建的 domain、code 和 dictionary 创建一个新的 `NSError` 实例。
5.  使用 `NSLog` 报告错误和字典。



