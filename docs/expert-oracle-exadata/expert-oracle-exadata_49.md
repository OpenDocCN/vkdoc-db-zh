# 等待事件详解

## enq: KO - fast object checkpoint

### 事件含义

此事件用于会话在启动`direct path read`或`cell smart table scan`或`cell smart index scan`之前，等待所有脏块从缓冲区缓存中为某个段刷新完毕。此事件很重要，因为执行检查点所需的时间可能超过直接路径读取带来的好处。不过，在 Exadata 存储上，这种情况不太可能发生，因为只有通过`direct path read`机制才能启用额外的智能扫描功能。尽管如此，执行引擎在决定将查询卸载到单元之前，会尝试考虑检查点成本。第 2 章包含了关于此机制的更多信息。

### 参数

此事件的参数帮助不大，但它确实显示了正在被扫描的对象。参数定义如下：

*   `P1` - 名称/模式
*   `P2` - 未使用
*   `P3` - 未使用
*   `obj#` - 正在被检查点的对象的对象号

## reliable message

`reliable message`事件用于记录与后台进程（如检查点进程 `CKPT`）通信所花费的时间。我们在此包含它是因为它与`enq: KO – fast object checkpoint`事件密切相关。

### 事件含义

在 Oracle 11.2 及更高版本中，此事件是`enq: KO – fast object checkpoint`事件（以及其他事件）的前导事件。通信是通过进程间通信通道完成的，而不是更常规的发布机制。这种通信方式允许发送方在继续之前请求确认（ACK），因此它被称为可靠消息。它通常是一个持续时间非常短的事件，因为它只记录进程间通信的时间。用户的前台进程和 CKPT 进程在相互通信时都会等待此事件。以下是来自 11.2.0.4 数据库的 10046 跟踪文件摘录，显示了一个完整的`reliable message`事件：

```
=====================
PARSING IN CURSOR #140457550697520 len=27 dep=0 uid=44 oct=3 lid=44 tim=1404832965816981
  hv=521453784 ad=’25d376810’ sqlid=’cdjur50gj9h6s’
select count(*) from bigtab
END OF STMT
PARSE #140457550697520:c=1000,e=1072,p=0,cr=0,cu=0,mis=1,r=0,dep=0,og=1,plh=2140185107,
  tim=1404832965816980
EXEC #1...0:c=0,e=35,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=2140185107,tim=1404832965817073
WAIT #1...0: nam=’SQL*Net message to client’ ela= 1 driver id=1650815232 #bytes=1 p3=0
  obj#=18532 tim=1404832965817131
WAIT #1...0: nam=’reliable message’ ela= 1230 channel context=10035861184 channel
  handle=10055026416 broadcast message=10199413976 obj#=18532 tim=1404832965818567
WAIT #1...0: nam=’enq: KO - fast object checkpoint’ ela= 189 name|mode=1263468550 2=65571 0=1
  obj#=18532 tim=1404832965818827
WAIT #1...0: nam=’enq: KO - fast object checkpoint’ ela= 107 name|mode=1263468545 2=65571 0=2
  obj#=18532 tim=1404832965819004
WAIT #1...0: nam=’cell smart table scan’ ela= 199 cellhash#=2133459483 p2=0 p3=0 obj#=18532
  tim=1404832965820790
WAIT #1...0: nam=’cell smart table scan’ ela= 182 cellhash#=3176594409 p2=0 p3=0 obj#=18532
  tim=1404832965821746
WAIT #1...0: nam=’cell smart table scan’ ela= 163 cellhash#=379339958 p2=0 p3=0 obj#=18532
  tim=1404832965822672
```

### 参数

以下是`reliable message`事件的参数：

*   `P1` - 通道上下文
*   `P2` - 通道句柄
*   `P3` - 广播消息
*   `obj#` - 相关对象的对象号（不总是设置）

### 资源管理器事件

在结束本章之前，我们将讨论几个您应该了解的资源管理器事件。虽然这些事件并非 Exadata 专有，但资源管理器为在 Exadata 上整合混合工作负载提供了关键功能。截至 12.1.0.2 版本，实际上有八个独立的事件。以下对`V$EVENT_NAME`的查询显示了这些事件及其参数：

```
SQL> select name,parameter1,parameter2,parameter3,wait_class
  2  from v$event_name where name like ’resmgr%’ order by name;
NAME                           PARAMETER1 PARAMETER2           PARAMETER3  WAIT_CLASS
------------------------------ --------- -------------------- ---------- ------------
resmgr:become active           location                                    Scheduler
resmgr:cpu quantum             location   consumer group id                Scheduler
resmgr:internal state change   location                                    Concurrency
resmgr:internal state cleanup  location                                    Other
resmgr:large I/O queued        location                                    Scheduler
resmgr:pq queued               location                                    Scheduler
resmgr:sessions to exit        location                                    Concurrency
resmgr:small I/O queued        location                                    Scheduler
8 rows selected.
```

其中只有三个事件值得关注。

## resmgr:become active

当会话等待开始执行时，您会看到此事件。例如，假设您定义了一个使用者组，要求在给定时间点具有最大并发会话数。任何额外的会话在开始工作前都会等待“`resmgr:become active`”。

### 事件含义

该事件表明资源管理器正在阻止会话开始执行。如果您创建了一个计划指令来限制使用者组中的并发会话数，您将看到类似以下的输出：

```
SQL> select sid,serial#,username,seq#,event,resource_consumer_group rsrc_cons_grp
  2  from v$session where username = ’LMTD’;
       SID    SERIAL# USERNAME         SEQ# EVENT                          RSRC_CONS_GRP
---------- ---------- ---------- ---------- ------------------------------ -----------------
       916       1413 LMTD            31124 cell smart table scan          LMTD_GROUP
      1045       2667 LMTD            29480 cell smart table scan          LMTD_GROUP
      1112      20825 LMTD               42 resmgr:become active           LMTD_GROUP
      1178         91 LMTD               43 resmgr:become active           LMTD_GROUP
      1239      22465 LMTD               46 resmgr:become active           LMTD_GROUP
      1311      19703 LMTD               47 resmgr:become active           LMTD_GROUP
      1374      32129 LMTD               44 resmgr:become active           LMTD_GROUP
      1432      29189 LMTD               40 resmgr:become active           LMTD_GROUP
8 rows selected.
```

更准确地说，使用者组`LMTD_GROUP`的并发会话数被设置为 2。以下是相应的配置示例：

```
BEGIN
dbms_resource_manager.clear_pending_area();
dbms_resource_manager.create_pending_area();
dbms_resource_manager.UPDATE_PLAN_DIRECTIVE(
  plan => ’ENKITEC_DBRM’,
  group_or_subplan => ’LMTD_GROUP’,
  new_active_sess_pool_p1 => 2);
dbms_resource_manager.validate_pending_area();
dbms_resource_manager.submit_pending_area();
end;
/
```

使用此设置，您永远不会让超过定义数量的会话同时执行工作。

### 参数

以下是此事件的参数。注意`obj#`参数存在但未使用：

*   `P1` - 位置
*   `P2` - 未使用
*   `P3` - 未使用
*   `obj#` - 不适用

`location`参数是一个数值，很可能指的是 Oracle 代码中的一个位置（函数）。我们至少观察到了五个不同的位置。遗憾的是，Oracle 没有公开记录这些检查在 Oracle 内核中的具体执行位置。


### resmgr:cpu quantum

此事件用于记录因与高优先级任务竞争而被数据库资源管理器强制实施的空闲时间。换句话说，这是一个进程等待数据库资源管理器分配其时间片所花费的时间。有趣的是，在撰写本文时，尚无数据库事件能够反映 I/O 资源管理层的节流情况。

### 事件含义

数据库资源管理器的行为类似于 CPU 调度算法，因为它将时间划分为单位（量子），并根据系统上的其他工作负载来决定是允许进程运行还是不允许。然而，与 CPU 调度算法不同，数据库资源管理器节流被注入到 Oracle 代码的关键位置，以消除进程在持有共享资源（如闩锁）时被踢出 CPU 的可能性。这可以防止在高负载系统上可能发生的一些棘手行为，例如优先级反转问题。实际上，进程会在不再持有这些共享资源时自愿休眠。这些检查在代码中有多个实现位置。以下是显示`resmgr:cpu quantum`事件的 10046 跟踪文件摘录：

```
PARSING IN CURSOR #140473587700424 len=30 dep=1 uid=208 oct=3 lid=208 tim=4695862695328
  hv=2336211758 ad='22465dc70' sqlid='gq01wgy5mzhtf'
SELECT COUNT(*) FROM MARTIN.T1
END OF STMT
EXEC #1...4:c=0,e=97,p=0,cr=0,cu=0,mis=0,r=0,dep=1,og=1,plh=3724264953,tim=4695862695326
WAIT #1...4: nam='enq: KO - fast object checkpoint' ela= 216 name|mode=1263468550 2=65683 0=2
  obj#=79208 tim=4695862695834
WAIT #1...4: nam='reliable message' ela= 1126 channel context=10126085744 channel
  handle=10164815328 broadcast message=10179052064 obj#=79208 tim=4695862697054
WAIT #1...4: nam='enq: KO - fast object checkpoint' ela= 160 name|mode=1263468550
  2=65683 0=1 obj#=79208 tim=4695862709511
WAIT #1...4: nam='enq: KO - fast object checkpoint' ela= 177 name|mode=1263468545
  2=65683 0=2 obj#=79208 tim=4695862709811
WAIT #1...4: nam='cell smart table scan' ela= 1830 cellhash#=3249924569 p2=0 p3=0
  obj#=79208 tim=4695862713420
WAIT #1...4: nam='cell smart table scan' ela= 326 cellhash#=674246789 p2=0 p3=0 obj#=79208
  tim=4695862714610
WAIT #1...4: nam='cell smart table scan' ela= 362 cellhash#=822451848 p2=0 p3=0 obj#=79208
  tim=4695862715784
WAIT #1...4: nam='resmgr:cpu quantum' ela= 278337 location=2 consumer group id=80408  =0
  obj#=79208 tim=4695863561098
WAIT #1...4: nam='cell smart table scan' ela= 2713 cellhash#=822451848 p2=0 p3=0 obj#=79208
  tim=4695863564184
```

### 参数

以下是此事件的参数。请注意，`obj#` 参数存在但未被使用：

*   `P1` - 位置
*   `P2` - 消费者组 ID
*   `P3` - 未使用
*   `obj#` - 不适用

位置参数是一个数值，很可能指向上文`resmgr:become active`事件中描述的 Oracle 代码中的位置（函数）。

`P2` 参数中的消费者组编号是不言自明的。它映射到`DBA_RSRC_CONSUMER_GROUPS`视图中的`CONSUMER_GROUP_ID`列。此参数允许您查明进程在 CPU 使用受到限制时被分配到了哪个消费者组。

### resmgr:pq queued

此事件用于记录在并行查询队列中等待的时间。并行语句排队是 11g Release 2 引入的一项新功能。通过将`parallel_degree_policy`设置为 Automatic 启用的新功能之一是语句排队。在 Oracle 12c 中，您也可以将其值设置为`ADAPTIVE`以达到相同的目的。语句排队允许您在所需并行度无法满足时，为可用的并行服务器从进程排队，而不是像旧的并行自动调整选项那样被降级。有关 Exadata 上并行执行的更多信息，请参阅第 6 章。

### 事件含义

并行语句排队功能有其自己的等待事件。由于并行服务器进程不足或其他指令而排队的语句会将时间记录到此事件。以下是显示`resmgr:pq queued`事件的 10046 跟踪文件摘录：

```
PARSING IN CURSOR #140583609334256 len=68 dep=0 uid=205 oct=3 lid=205 tim=4755825333777
  hv=2381215807 ad='220a20bc0' sqlid='944dxty6ywy1z'
select /*+ PARALLEL(32) STATEMENT_QUEUING */ count(*) from martin.t3
END OF STMT
PARSE #140583609334256:c=0,e=188,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=3978228158,
  tim=4755825333776
WAIT #140583609334256: nam='resmgr:pq queued' ela= 25193140 location=1  =0  =0 obj#=-1
  tim=4755850527127
WAIT #140583609334256: nam='reliable message' ela= 889 channel context=10126085744 channel
  handle=10164818472 broadcast message=10179043672 obj#=-1 tim=4755850529508
WAIT #140583609334256: nam='enq: KO - fast object checkpoint' ela= 197 name|mode=1263468550
  2=65690 0=1 obj#=-1 tim=4755850529777
WAIT #140583609334256: nam='enq: KO - fast object checkpoint' ela= 124 name|mode=1263468545
  2=65690 0=2 obj#=-1 tim=4755850529959
WAIT #140583609334256: nam='PX Deq: Join ACK' ela= 39 sleeptime/senderid=268500992 passes=1
  p3=9216157728 obj#=-1 tim=4755850531333
WAIT #140583609334256: nam='PX Deq: Join ACK' ela= 42 sleeptime/senderid=268500993 passes=1
  p3=9216246432 obj#=-1 tim=4755850531420
WAIT #140583609334256: nam='PX Deq: Join ACK' ela= 39 sleeptime/senderid=268500994 passes=1
  p3=9216266144 obj#=-1 tim=4755850531507
```

在测试此用例时，发现该事件仅在开始时写入跟踪文件一次。此外，关于排队语句的跟踪信息直到会话实际开始工作才会发出。

### 参数

以下是`resmgr:pq queued`事件的参数：

*   `P1` - 位置
*   `P2` - 未使用
*   `P3` - 未使用
*   `obj#` - 不适用

位置参数是一个数值，很可能指向上文`resmgr:become active`事件中描述的 Oracle 代码中的位置（函数）。

## 总结

等待接口已扩展到涵盖多个 Exadata 特定功能。在本章中，您了解了应该知道的新等待事件。迄今为止，最有趣的新事件是`cell smart table scan`和`cell smart index scan`。这些事件涵盖了等待到存储单元的可卸载读 I/O 请求所花费的时间。在存储层发生的许多处理被归并在这些事件下。重要的是要理解，这些事件取代了`direct path read`事件，并且 Smart Scan 事件将数据直接返回到进程 PGA 的机制类似于`direct path read`的处理方式。


