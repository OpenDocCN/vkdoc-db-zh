# 第 8 章：生成的数据访问层

通过**部分类**可以开放类的可扩展性。当大部分代码是生成的时候，此功能非常有用。生成的文件可以与手动编辑的文件保持分离，这样在重新生成代码更新时，你的修改就不会丢失。

`SubSonic`生成器创建的每个类都被标记为部分类。如果你想为生成的类添加功能，可以创建一个新的部分类来添加额外的方法和属性。假设你有一个名为`Person`的表，其中包含`FirstName`和`LastName`字段。当使用`SubSonic`生成`Person`类时，你将获得这些表列的属性。你可以创建一个部分类来添加一个名为`FullName`的属性，如**代码清单 8-17**所示。

**代码清单 8-17.** `FullName` 属性

```csharp
namespace Chapter08.SubSonicDAL
{
    public partial class Person : ActiveRecord<Person>
    {
        /// <summary>
        /// FirstName 和 LastName 组合
        /// </summary>
        public string FullName {
            get {
                return this.FirstName + " " + this.LastName;
            }
        }
    }
}
```

由于`FirstName`和`LastName`属性已经定义，尽管它们定义在另一个源文件中，你仍可以将它们作为同一类的一部分来访问。你还可以添加一个方法来提供`SubSonic`原生未提供的行为。一个标准的方法是`FetchByID`方法，它接受目标表的主键。该方法只是简单地查询数据库中的单条记录。`FetchByID`方法定义在`PersonController`类中。

你可以做的是，使用几乎相同的方法签名向该类添加缓存行为。你不是只传入`ID`参数，而是添加一个名为`cached`的布尔值。当它为`true`时，将使用缓存，如**代码清单 8-18**所示。

**代码清单 8-18.** 带缓存的 `FetchByID`

```csharp
namespace Chapter08.SubSonicDAL
{
    public partial class PersonController
    {
        public PersonCollection FetchByID(object ID, bool cached)
        {
            if (!cached)
            {
                return FetchByID(ID);
            }
            string cacheKey = (typeof(Person)).ToString() + "-" + ID;
            Cache cache = HttpRuntime.Cache;
            if (cache[cacheKey] != null) {
                return cache[cacheKey] as PersonCollection;
            }
            PersonCollection coll = new PersonCollection().Where(Person.Columns.ID, ID).Load();
            cache.Insert(cacheKey, coll, null,
                DateTime.Now.AddMinutes(5), TimeSpan.Zero);
            return coll;
        }
    }
}
```

以这种方式缓存`Person`记录可能会消除应用程序中的一个瓶颈，但你也需要确保当`Person`记录更新时，它能从缓存中被移除。**代码清单 8-19**中的`ClearCachedItem`方法负责从缓存中移除该项目。

**代码清单 8-19.** `ClearCachedItem` 方法

```csharp
public void ClearCachedItem(object ID)
{
    string cacheKey = (typeof(Person)).ToString() + "-" + ID;
    Cache cache = HttpRuntime.Cache;
    cache.Remove(cacheKey);
}
```

每次你调用`Insert`和`Update`方法时，都可以确保调用`ClearCachedItem`方法。以这种方式添加功能是优化使用此数据访问层的应用程序的绝佳方式。

#### 查询工具

`SubSonic`使用一个内置的查询工具，该工具生成用于在数据库中获取、保存和删除数据的 SQL。这个查询工具在`LINQ`技术作为社区技术预览版发布之前很久就开发出来了，但它有相当多的共通之处。

该查询工具使用一个名为`Query`的类。它通过一系列来自许多方法的调用来构建，这些调用逐步完善查询。其目标是使`C#`代码看起来很像用 SQL 编写的查询。**代码清单 8-20**显示了一个示例方法，它通过`LocationID`加载所有`Person`记录。

**代码清单 8-20.** `FetchPersonsByLocationID` 方法

```csharp
public PersonCollection FetchPersonsByLocationID(object LocationID)
{
    return new PersonCollection().
        Where(Person.Columns.LocationID, LocationID).
        OrderByAsc(Person.Columns.LastName).Load();
}
```

关于`FetchPersonsByLocationID`方法有几个重要的细节。第一个细节是该方法是完全类型安全的。使用类型化`DataSet`的一个强烈理由是为了类型安全，但是由于`SubSonic`生成的代码直接基于数据库，它会自动创建类型感知的代码。在此方法中，对`Where`方法的调用可以使用字面字符串，但这里使用的是为`LocationID`列定义的常量。

第二个细节是查询完全由查询工具使用带有`LocationID`列的`Person`表透明地构建。在`C#`中编写此查询非常容易，并可以根据需要轻松添加其他细化条件。最终生成的查询构建了一个参数化查询，当查询被重复使用时，`SQL Server`能够很好地优化它。

#### 脚手架

生成的数据访问层最后一个主要优点是能够将*脚手架*系统自动连接到生成的代码。*脚手架*是一组接口，你无需编写一行代码就可以使用它们向数据库添加示例数据。许多开发人员发现这非常有用，特别是当脚手架能智能处理表之间的关系时。`SubSonic`包含一个自动脚手架系统，其灵感来源于在`Ruby on Rails`（一个基于`Ruby`脚本语言的流行 Web 框架）上完成的工作。`SubSonic`实际上借鉴了`Ruby on Rails`的几个概念。

对于`SubSonic`，你可以使用`SubSonicCentral`网站中的`AutoScaffold`页面，这是可下载源代码的一部分。你可以将此页面及其依赖项复制到你自己网站的管理部分，它将为你提供数据库的界面。图 8-5 显示了编辑`Person`表的`AutoScaffold`页面。

**图 8-5.** `AutoScaffold` 页面

虽然`AutoScaffold`页面不会完美，但作为你应用程序的唯一编辑器，它确实以极小的工作量提供了对数据的自动访问。并且当你更新数据库并重新生成数据访问层时，`AutoScaffold`页面将自动调整以匹配更改。你无需付出任何努力来保持其最新。

除了简单地将编辑器封装在表周围，`AutoScaffold`系统还能感知关系。`Person`表有一个指向`Location`表的外键引用。这个简单的引用足以让`AutoScaffold`页面在编辑`Person`记录时提供一个下拉列表来选择位置。图 8-6 显示了处于编辑模式的`Person`记录。

**图 8-6.** 编辑带位置引用的 `Person` 记录

凭借`SubSonic`提供的所有功能，你应该能够快速启动并运行。`SubSonic`实现的生成的数据访问层和开发体验与类型化`DataSet`提供的截然不同，后者虽然也生成大量代码，但极其僵化且难以随数据库变更而更新。当你开始构建应用程序并发生变更时，你应该能够轻松适应。当你向数据库添加新列时，只需运行生成命令即可更新你的数据访问层。并且借助部分类和代码生成模板，你可以轻松修改代码生成过程以满足你的独特需求。

**Blinq**

