# 关于自主事务与数据库设计的说明

在本书中，我将使用自主事务来演示锁、阻塞和并发问题。我坚信，自主事务是 Oracle 不应该暴露给开发者的功能——原因很简单，大多数开发者并不知道何时以及如何正确地使用它们。不当使用自主事务可能导致逻辑数据完整性损坏问题。除了用作演示工具，自主事务恰好还有另一种用途——作为一种错误日志记录机制。如果你希望在异常块中记录错误，你需要将该错误记录到表中并提交——而不提交其他任何内容。那将是自主事务的一个有效用例。如果你发现自己在记录错误或演示概念之外使用自主事务，那你几乎肯定做错了什么。

由于我使用了自主事务并创建了一个子事务，我收到了一个死锁——这意味着我的第二次插入被第一次插入阻塞了。如果我使用的是两个独立的会话，则不会发生死锁。相反，第二次插入会被阻塞，并等待第一个事务提交或回滚。这正是相关项目所面临的问题——阻塞、串行化问题。

因此，我们遇到了一个问题：由于不理解数据库特性（位图索引）及其工作原理，导致数据库从一开始可扩展性就很差。更糟糕的是，那段队列代码根本没有必要编写。Oracle 数据库具有内置的队列功能。这个内置队列功能使你能够拥有多个生产者（插入未处理记录 'N' 的会话）并发地将消息放入入站队列，并拥有多个消费者（查找待处理记录 'N' 的会话）并发地接收这些消息。也就是说，不需要编写特殊代码在数据库中实现队列。开发者本应使用这个内置特性。他们可能确实想用，只是完全不知道它的存在。

幸运的是，一旦发现这个问题，纠正起来就很容易了。我们确实需要在 `processed_flag` 列上创建索引，但不是位图索引。我们需要一个常规的 `B*树` 索引。要说服人们创建这样一个索引费了一点功夫。没有人愿意相信对一个只有两个不同值的列创建常规索引是个好主意。但在建立了一个模拟环境后（我非常热衷于模拟、测试和实验），我们证明了这不仅是正确的方法，而且效果会非常好。

## 关于索引选择的说明

我们创建任何类型的索引，通常是为了在大量数据中找到少量的行。在这个案例中，我们希望通过索引找到的行数是 *一行*。我们需要找到一条未处理的记录。一行是一个非常小的行数；因此，索引是合适的。任何类型的索引都是合适的。`B*树` 索引在从大量记录中查找单个记录时非常有用。

当我们创建索引时，必须在以下方法之间做出选择：

*   只在 `processed_flag` 列上创建一个索引。
*   仅当处理标志为 `N` 时，才在 `processed_flag` 列上创建索引，即只索引感兴趣的值。我们通常不希望在处理标志为 `Y` 时使用索引，因为表中的绝大多数记录的值都是 `Y`。注意，我并没有说“我们从不希望使用……”。你可能出于某种原因需要非常频繁地统计已处理记录的数量，那么在已处理记录上建立索引可能就非常方便。

在第 11 章关于索引的内容中，我们将更详细地讨论这两种类型。最终，我们在 `processed_flag` 为 `N` 的记录上创建了一个非常小的索引。访问这些记录的速度极快，而绝大多数 `Y` 记录根本没有贡献到这个索引中。我们使用了一个基于函数的索引，该函数为 `decode( processed_flag, 'N', 'N' )`，返回 `N` 或 `NULL`——因为完全为 `NULL` 的键不会被放入常规的 `B*树` 索引中，所以我们最终只索引了 `N` 记录。

**注意**
关于 `NULL` 值和索引的更多信息见第 11 章。

## 实现队列

故事到此结束了吗？不，完全没有。我的客户手上仍然有一个不够优化的解决方案。他们仍然需要在对未处理记录进行“出队”时进行串行化。我们可以轻松地找到第一条未处理记录——使用 `SELECT * FROM queue_table WHERE decode( processed_flag, 'N', 'N' ) = 'N' FOR UPDATE` 快速定位——但一次只能有一个会话执行该操作。该项目使用的是 Oracle 10g，因此还无法利用 Oracle 11g 中新增的相对较新的 `SKIP LOCKED` 特性。`SKIP LOCKED` 将允许许多会话并发地找到第一条未锁定、未处理的记录，锁定该记录并进行处理。相反，我们必须实现代码来手动查找第一条未锁定记录并锁定它。这样的代码在 Oracle 10g 及更早版本中通常如下所示。我们首先创建一个具有前述所需索引的表，并用一些数据填充它，如下所示：

```sql
SQL> drop table t purge;
SQL> create table t ( id       number primary key,
    processed_flag varchar2(1),
    payload  varchar2(20));
表已创建。
SQL> create index  t_idx on t( decode( processed_flag, 'N', 'N' ) );
索引已创建。
SQL> insert into t
    select r,
           case when mod(r,2) = 0 then 'N' else 'Y' end,
           'payload ' || r
      from (select level r
              from dual
           connect by level < 6);
SQL> select * from t;
        ID P PAYLOAD
---------- - --------------------
         1 Y payload 1
         2 N payload 2
         3 Y payload 3
         4 N payload 4
         5 Y payload 5
```

然后，我们基本上需要找到所有未处理的记录。我们逐行询问数据库：“这一行是否已被锁定？如果没有，那就锁定它并交给我。” 代码如下所示：

```sql
SQL> create or replace
    function get_first_unlocked_row
    return t%rowtype
    as
        resource_busy exception;
        pragma exception_init( resource_busy, -54 );
        l_rec t%rowtype;
    begin
        for x in ( select rowid rid
                     from t
                    where decode(processed_flag,'N','N') = 'N')
        loop
            begin
                select * into l_rec
                  from t
                 where rowid = x.rid and processed_flag='N'
                   for update nowait;
                return l_rec;
            exception
                when resource_busy then null;
                when no_data_found then null;
            end;
        end loop;
        return null;
    end;
/
函数已创建。
```

**注意**
在前面的代码中，我运行了一些 DDL——`CREATE OR REPLACE FUNCTION`。在 DDL 运行之前，它会自动提交，所以这里有一个隐式的 `COMMIT`。我们插入的行已经在数据库中提交了——这个事实是后续示例正确工作所必需的。总的来说，我将在本书的剩余部分利用这个事实。如果你运行这些示例而没有执行 `CREATE OR REPLACE`，请确保先 `COMMIT`！

现在，如果我们使用两个不同的事务，可以看到它们获取到了不同的记录。我们还看到这两个事务并发地获取到了不同的记录（再次使用自主事务来演示并发问题）。



