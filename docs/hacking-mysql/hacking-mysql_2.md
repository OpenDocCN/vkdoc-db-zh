# 2. 个别的存储引擎

对许多人来说，使用 MySQL 只是达成目的的手段——我们中的一些人用它创建网站，而对另一些人来说，应用程序可能是重中之重。无论你的具体用例是什么，你都需要在其基础设施中善用存储引擎。

回顾第 1 章，并再次仔细查看图 1-1 中的 MySQL 架构图。最后一层提供了一个不完整的存储引擎列表，该层被称为“存储”。存储层由三个对象组成——`InnoDB`、`MyISAM` 和 `NDB`。这些是 MySQL 中可用的存储引擎，虽然其中一个可能被视为“王者”，但其他引擎也同样重要——在规划你下一步旅程之前，所有引擎都应被考虑。

### 存储引擎的现代王者

#### MySQL 和 MariaDB 中的 InnoDB

就 MySQL 中的存储引擎而言，其中一个——`InnoDB`——毫无疑问地拔得头筹。你们中的许多人可能都听说过 `InnoDB`：该存储引擎于 2010 年 7 月在 MySQL 5.5.5 版本中引入，并以支持许多用于调整 MySQL 服务器以实现高性能和可靠性的功能而闻名。

自诞生以来，`InnoDB` 已取代 `MyISAM` 成为主要和默认的存储引擎——过去，`MyISAM` 对许多开发者来说仍然是一个可行的选择，但随着时间的推移，工程师们将原本仅在 `MyISAM` 中可用的功能引入了 `InnoDB`，`MyISAM` 引擎已逐渐成为历史。`MyISAM` 在 MySQL 中仍然可见，但该引擎可以被合理地认为已经过时。话虽如此，`MyISAM` 有一个值得注意的功能，但现在请先别分心——`InnoDB` 也是一头需要妥善应对的猛兽。

一切都始于配置——在 Unix 系统上是 `my.cnf`，在 Windows 架构上是 `my.ini`。该文件可以在多个位置找到。Windows 用户可以在二进制文件夹中找到该文件：`/bin/mysql/mysql*.*.**`，其中 `*.*.**` 是你的 MySQL Server 版本；Unix 基础设施的用户可以在以下位置之一找到该文件：

```
/etc/my.cnf
/etc/mysql/my.cnf
/var/lib/mysql/my.cnf
~/my.cnf
```

在指定位置之一找到该文件后，打开它，你会看到一堆与 `InnoDB` 相关的设置。在 Windows 架构上，可用的设置将如下所示（示例基于 MySQL 8.0.31。旧版本可能有较少的可用选项——最重要的选项已加粗）：

```
innodb_adaptive_hash_index=on
innodb_buffer_pool_dump_now=off
innodb_buffer_pool_dump_at_shutdown=off
innodb_buffer_pool_instances=2
innodb_buffer_pool_load_at_startup=off
innodb_buffer_pool_size=1G
innodb_data_file_path=ibdata1:12M
innodb_default_row_format=dynamic
innodb_doublewrite=on
;skip_innodb_doublewrite
innodb_file_per_table=1
innodb_flush_log_at_trx_commit=1
innodb_flush_method=normal
;innodb_force_recovery=1
innodb_ft_enable_stopword=off
innodb_ft_max_token_size=10
innodb_ft_min_token_size=0
innodb_io_capacity=2000
innodb_max_dirty_pages_pct=90
innodb_lock_wait_timeout=600
innodb_log_buffer_size=16M
innodb_log_file_size=20M
innodb_log_files_in_group=2
innodb_optimize_fulltext_only=1
innodb_page_size=16K
innodb_purge_threads=10
innodb_read_io_threads=10
innodb_stats_on_metadata=0
innodb_strict_mode=on
innodb_thread_concurrency=16
innodb_undo_log_truncate=on
innodb_write_io_threads=4
```

以下是这些选项的含义，逐一解释：



| 设置 | 说明 |
| --- | --- |
| `innodb_adaptive_hash_index` | 使 InnoDB 能够作为内存数据库运行。一旦启用，如果基于 InnoDB 的表能完全放入服务器的内存中，MySQL 将通过自动构建哈希索引来实现对元素的直接（因而更快）查找。可能不适用于所有项目——在生产环境中使用前请进行测试。 |
| `innodb_buffer_pool_dump_now` | 立即转储缓冲池中最近缓存的页面。 |
| `innodb_buffer_pool_dump_at_shutdown` | 在 InnoDB 引擎关闭时转储缓冲池中最近缓存的页面。 |
| `innodb_buffer_pool_instances` | 定义 InnoDB 中缓冲池被划分成的实例数量。如果 InnoDB 缓冲池非常大，调整此值并增加实例数量可能会提升性能。 |
| `innodb_buffer_pool_load_at_startup` | 每当 MySQL 启动时，缓冲池会预加载与关闭时相同数量的页面。 |
| `innodb_buffer_pool_size` | 可以说是 InnoDB 中最重要的参数。指定 InnoDB 中缓冲池的大小——由于 InnoDB 缓冲池保存着与 InnoDB 表相关的数据和索引，建议将其值增大到足以容纳你的数据。将此参数的值设置为可用 RAM 的 60% 左右是一个良好的起点。根据需要进行调整。 |
| `innodb_data_file_path` | 指向 `ibdata1` 的路径——该文件保存着源自基于 InnoDB 的表的数据、元数据和索引。 |
| `innodb_default_row_format` | 指定使用 InnoDB 的表的默认行格式。可用值：1. `DYNAMIC`。从 MySQL 5.0.3 开始的默认值。使用此行格式的 InnoDB 表将以一种能够使数据库在溢出的数据库页面上存储更多数据的方式来存储行。否则，与 `COMPACT` 非常相似。2. `REDUNDANT`。MySQL 5 版本之前可用的原始行格式。3. `COMPACT`。与 `REDUNDANT` 非常相似。此格式在存储数据时需要的存储空间更少，因此得名。4. `COMPRESSED`。尽可能压缩数据，可以在溢出的页面上存储更多数据。 |
| `innodb_doublewrite` | 启用或禁用双写缓冲。可用值——`ON` 和 `OFF`。启用此参数将为 InnoDB 提供一个在将数据库页面写入 `ibdata1` 之前可以“操作”的存储区域，从而可能防止只写入一半的页面被写入 `ibdata1`。 |
| `skip_innodb_doublewrite` | 如果未被注释掉，则使 InnoDB 跳过双写缓冲。 |
| `innodb_file_per_table` | 通过将数据存储在单独的文件中，使 InnoDB 表能够与 `ibdata1` 文件分开存储，从而防止混乱。除非有充分的理由关闭它，否则请将此参数的值保持为 `ON`。 |
| `innodb_flush_log_at_trx_commit` | 控制严格的 ACID 合规性与以牺牲 ACID 为代价的更高性能之间的平衡。值 `"0"` 将使 InnoDB 每秒仅将重做日志刷新到磁盘一次；默认值 `"1"` 将使 InnoDB 符合 ACID 标准；值 `"2"` 将在提交时写入日志并每秒将日志刷新到磁盘一次；而 `"3"` 将在日志准备好和提交时分别写入磁盘一次，从而使该过程变慢。 |
| `innodb_flush_method` | 描述 InnoDB 在事务提交阶段刷新日志所使用的方法。可用值包括：1. `normal` 2. `unbuffered` 3. `async_unbuffered` 4. `fsync` 5. `O_DSYNC` 6. `nosync` 7. `littlesync` 8. `O_DIRECT` 9. `O_DIRECT_NO_FSYNC` 我稍后会解释这些值的含义，但就目前而言，对于 Windows 架构请将此设置保留为默认值，对于 Linux 则选择 `O_DIRECT`。 |
| `innodb_force_recovery` | 启用或禁用 InnoDB 的崩溃恢复模式。可用值包括从 0 到 6 的整数。 |
| `innodb_ft_enable_stopword` | 如果保持为 `ON`（默认值），将使 InnoDB 全文索引包含停用词。使用内置的停用词列表，或从 `innodb_ft_user_stopword_table` 或 `innodb_ft_server_stopword_table` 中设置的值获取。 |
| `innodb_ft_max_token_size` | 指定可以使用 `FULLTEXT` 索引的词的最大字符长度。 |
| `innodb_ft_min_token_size` | 指定可以使用 `FULLTEXT` 索引的词的最小字符长度。 |
| `innodb_io_capacity` | 定义 InnoDB 可用于后台完成任务的每秒 I/O 操作数。 |
| `innodb_max_dirty_pages_pct` | 一个基于百分比的值，定义了 InnoDB 中可接受的脏页百分比。 |
| `innodb_lock_wait_timeout` | InnoDB 事务在回滚操作之前等待行锁的时间（以秒为单位）。 |
| `innodb_log_buffer_size` | InnoDB 将数据写入日志文件时所使用的缓冲区大小。默认值 – 16MB，增大此值可能会提升 I/O 性能。 |
| `innodb_log_file_size` | InnoDB 日志文件的大小。推荐值 – 约为缓冲池大小的四分之一（25%）。从 MySQL 8.0.30 开始已弃用 – 请改为调整 `innodb_redo_log_capacity` 变量。 |
| `innodb_log_files_in_group` | 指定一个组中 InnoDB 日志文件的数量。从 MySQL 8.0.30 开始已弃用 – 请改为调整 `innodb_redo_log_capacity` 变量。 |
| `innodb_optimize_fulltext_only` | 使 `OPTIMIZE TABLE` 仅优化 `FULLTEXT` 索引。旨在用于维护操作。 |
| `innodb_page_size` | 指定运行 InnoDB 存储引擎的表的数据页大小。 |
| `innodb_purge_threads` | 定义 InnoDB 清理操作开始时要使用的线程数。 |
| `innodb_read_io_threads` | 指定为 InnoDB 中的读操作（`SELECT` 查询）分配的 I/O 线程数。 |
| `innodb_stats_on_metadata` | 当优化器统计信息非持久化时，此参数适用于 MySQL 的优化器。可用值包括 `OFF`（默认）和 `ON`。如果你处理较大数据集并涉及 `SELECT` 查询，请考虑启用此参数。 |
| `innodb_strict_mode` | 使 InnoDB 的行为更加严格，如果启用此变量，警告将被视为错误。 |
| `innodb_thread_concurrency` | InnoDB 中的最大线程数。 |
| `innodb_undo_log_truncate` | 如果撤消表空间超过了 `innodb_max_undo_log_size` 中定义的某个阈值，则日志将被截断。 |
| `innodb_write_io_threads` | 定义 InnoDB 内部用于写入操作的 I/O 线程数。 |



有很多参数可以优化，对吧？然而，虽然了解上述参数的作用可以成为我们在搞坏 MySQL 实例之前的一个良好起点，但你们大多数人可能并不需要修改甚至查看所有参数——优化其中几个就绰绰有余了，其余的参数只有在你日后想要将 InnoDB 推向极限时才需要触及。

### InnoDB 参数

在最初优化 InnoDB 时，请注意这七个加粗的值：

1.  `innodb_buffer_pool_size`：将缓冲池大小设置为系统可用 RAM 的 60–80%。缓冲池越大，InnoDB 能够缓存的数据和索引就越多，从而你的 SQL 查询完成得更快。
2.  `innodb_data_file_path`：这个参数乍一看可能不太重要，但请记住，数据文件（下文解释）存储了与你的表相关的一系列数据。因此，如果你正在进行一个基于大数据的项目，将数据文件路径设置在拥有大量空间的驱动器上至关重要。
3.  `innodb_file_per_table`：将此参数保留为其默认值 1。这样做，通过让 InnoDB 将表存储在单独的文件中，可以从数据文件中清除不必要的混乱，同时也为未来的维护操作提供更大的灵活性。
4.  `innodb_flush_log_at_trx_commit`：通常最好将此参数保留为默认值 1，以确保你的数据库保持符合 ACID 原则。虽然可以像上面指定的那样指定 0、2、3 等其他值，但那会为了速度而牺牲 ACID。
5.  `innodb_flush_method`：如上所述，此参数定义了 InnoDB 在事务提交后用于刷新日志的方法。以下是所有可用选项：
    1.  `normal`：模拟异步 I/O，缓冲 I/O。
    2.  `unbuffered`：模拟异步 I/O，非缓冲 I/O。
    3.  `async_unbuffered`：Windows 异步 I/O 和非缓冲 I/O。Windows 用户的默认设置。
    4.  `fsync`：让 InnoDB 使用 `fsync()` 函数来同步数据库和服务器的状态。Linux 用户的默认设置。
    5.  `O_DSYNC`：让 InnoDB 使用 `O_DSYNC` 函数，以保证数据和元数据在刷新日志文件之前都已写入磁盘。通常比 `O_DIRECT` 更快，但以数据一致性为代价。
    6.  `nosync`：用于内部测试，不支持生产用例。
    7.  `littlesync`：用于内部测试，不支持生产用例。
    8.  `O_DIRECT`：向 InnoDB 施加一个提示，表示你希望数据绕过 Linux 内核的缓存。通常比 `O_DSYNC` 慢，但更稳定且数据一致。
    9.  `O_DIRECT_NO_FSYNC`：向 InnoDB 施加一个提示，表示希望使用 `O_DIRECT` 同时跳过 `fsync()` 函数。
6.  `innodb_log_buffer_size`：日志缓冲区大小对于快速写入磁盘上的日志文件至关重要。增大日志缓冲区将使较大的事务能够更快完成，从而节省磁盘 I/O。
7.  `innodb_redo_log_capacity`：这是现已弃用的 `innodb_log_file_size` 和 `innodb_log_files_in_group` 变量的替代项。如果未定义此变量，而是定义了日志文件大小和组中的日志文件数，则此参数的值将使用公式计算（`innodb_log_files_in_group * innodb_log_file_size`）。否则，InnoDB 将使用此参数中指定的值。

理解上述参数对于实现你的性能目标至关重要，但也要记住，如果没有 InnoDB 系统表空间——`ibdata1`，这些参数将是不完整的。

如上所述，InnoDB 还有一个数据文件，也称为“系统表空间”。该文件就是前面提到的 `ibdata1`——它位于你 MySQL Server 安装目录的 `bin` 目录中，并充当与 InnoDB 存储引擎相关的一切的支柱。`ibdata1` 存储了所有基于 InnoDB 的表的数据和索引、回滚空间和回滚段、双写缓冲区、辅助索引或插入缓冲区的更改，以及多版本并发控制 (MVCC) 数据。启用 `innodb_file_per_table` 将有助于减轻损害，但如果 `ibdata1` 被删除或损坏，你的数据就丢失了。

除此之外，关于 `ibdata1` 及相关文件还有很多可以说的——当你“拆分和优化”我们的数据库时，你将学习到具体细节。在此之前，你必须了解可用的其余存储引擎选择。

你必须记住的一件事是，不同版本的 MySQL 也带有不同的搜索引擎选项——Percona Server 以其 XtraDB 产品而闻名，而 MariaDB 可以让你将数据存储在 Amazon S3 服务器中，或者提供 MyISAM 的崩溃安全替代方案。但是，让我们从头开始说起……

### Percona Server

#### Percona Server 中的存储引擎

Percona Server 可以看作是 MySQL Server 的增强版本。该服务器是 MySQL 的一个免费开源替代品，专注于提供极快的性能。

Percona Server 的工作方式与其同类产品类似——它确实支持完整功能的 InnoDB，但它也有自己的独门绝技：Percona 也以其 XtraDB 分支而闻名。

XtraDB 是 InnoDB 的增强版本，具有定制的内置选项以增强性能。它并非什么魔法，但如果你正在捣鼓 InnoDB，你也肯定会听说它的兄弟 XtraDB。

与普遍看法相反，XtraDB 也可以在 MariaDB 中使用，甚至在 MariaDB 10.1 之前一直是其默认存储引擎——但现在情况不同了，因为从 MariaDB 10.2 开始，团队切换回了 InnoDB，因为 XtraDB 在旧版本的数据库管理系统中比 InnoDB 有许多改进，但随着时间的推移，InnoDB 已经迅速赶了上来。

有人认为，在优化选项得当的情况下，XtraDB 只比 InnoDB 略好一点，但无论如何，有几件事你应该知道。

XtraDB 建立在 InnoDB 的基础之上，利用其稳健的设计，同时在其基础上添加了更多功能和指标。Percona Server 还支持另外两个 MySQL 中没有的存储引擎——MyRocks 和 TokuDB。

#### Percona MyRocks 和 TokuDB


### MyRocks

MyRocks 是一个基于 RocksDB 的存储引擎——一个用于更快存储的键值存储。它以占用更少的磁盘空间和更好的 I/O 能力而闻名。需要注意的是，要使用该存储引擎，必须在你的服务器上安装并启用 Percona MyRocks。一个简单的 `sudo` 命令即可完成：
```bash
sudo apt install percona-server-rocksdb-OS
```
这里的 `OS` 代表你需要根据所使用的操作系统指定的选项。如果是 Debian 或 Ubuntu，指定 `5.7` 即可。对于 RHEL 或 CentOS，则需要指定 `57.x86_64`：
```bash
sudo apt install percona-server-rocksdb-57.x86_64
```
安装好 RocksDB 后，使用 Percona Server 管理脚本 `ps-admin` 与 Percona Server 交互，像这样操作，之后就可以正常使用了：
```bash
sudo ps-admin --enable-rocksdb --u [用户名] --p[密码]
```
完成后，你就可以开始熟悉 RocksDB 了。

你首先需要了解的是，虽然 MySQL 中的大多数存储引擎都基于 B+ 树，但与 InnoDB 不同，RocksDB 是一个基于 LSM 的存储引擎。这意味着它可能不太适合读密集型工作负载，但与此同时，如果你的应用场景是写密集型工作负载，并且需要高效存储能力来处理更大的数据集，那么它也是一个相当不错的选择。MyRocks 中的数据所需的存储空间也更少，但这并非没有缺点：
- MyRocks 不支持在线 DDL 命令。
- 不支持全文索引、空间索引或外键。
- 不支持间隙锁、可移植表空间或组复制。
- MyRocks 也没有任何 `SAVEPOINT` 功能。

如果这些缺点没有吓退你，并且你的应用场景是写密集型工作负载和更大的数据集，那就熟悉一下可用的选项吧。MyRocks 提供的一些更有趣的选项包括：
| MyRocks 选项 | 说明 |
| --- | --- |
| `rocksdb_bulk_load` | 通过启用 `rocksdb_bulk_load` 变量，可以告诉 MyRocks 忽略某些检查或跳过锁的获取。MyRocks 还有几个与此相关的选项，允许你指定在批量加载期间何时提交事务等。 |
| `rocksdb_create_checkpoint` | 将创建检查点文件的目录（默认值 – 无）。 |
| `rocksdb_db_write_buffer_size` | 当内存中的表达到此参数指定的值时，数据将刷新到磁盘。 |
| `rocksdb_error_if_exists` | 在构建新数据库时使用。值为“1”时，如果数据库已存在，则显示错误。默认关闭（0）。 |
| `rocksdb_column_default_value_as_expression` | 允许将函数用作列的默认值。 |

批量加载功能、函数作为默认值……是不是很有趣？仅仅浏览一下可用的配置选项，就足以让你明白这个存储引擎是为写入而设计的。只要记住几条关键规则，向存储引擎提供数据并不复杂：
1. **批量加载**：启用 `rocksdb_bulk_load` 选项，加载数据，然后禁用该选项。启用此选项可以使数据加载更快，因为它让 MyRocks 跳过某些操作，但也意味着在批量加载操作完成之前，你的数据可能不可见。
2. **二级索引**：如果你的表有带二级索引的列，请在向表提供数据之前删除这些索引。之后，再恢复它们。
3. **未排序数据**：如果你的数据未按主键排序，请在加载数据之前启用 `rocksdb_bulk_load_allow_unsorted` 选项，并在之后禁用它。
4. **内存**：由于 MyRocks 会“记忆”正在进行的事务细节（将它们保存在内存中），因此明智的做法是保持事务规模相对较小，以避免问题。

之后，通过熟悉文档中提供的信息来保持更新，这样你就应该可以顺利使用了！

### TokuDB

直到 Percona Server 8 和 MariaDB 10.5 之前，MySQL 还有一个存储引擎可用——TokuDB。TokuDB 的理念是便于处理数据同时插入和查询的更大数据集；然而，随着时间的推移，Percona 意识到 MyRocks 提供了与 TokuDB 类似的益处，最终在 2021 年 5 月弃用了这个存储引擎。

TokuDB 并不是唯一被认为已弃用的存储引擎——MySQL 的老派狂热者会记得 MyISAM，它在 MySQL Server 5.5 推出之前一直是 MySQL 的默认存储引擎。

### InnoDB 的主要竞争对手

过去，MyISAM 是 MySQL 提供的唯一存储引擎，自身有许多优点，甚至在很长一段时间内被认为是 InnoDB 的有力竞争者——至今仍能看到一些开发者建议使用这个存储引擎，但数据库管理员会很快告诉你，InnoDB 迅速赶超并将 MyISAM 比了下去，这意味着按今天的标准，MyISAM 被广泛认为是一个过时的存储引擎，因为它不支持 ACID，并且容易崩溃。

话虽如此，MyISAM 相对于其 counterparts InnoDB 确实有两个显著优势——它不会将数据存储在任何无法收缩的文件中（InnoDB 的表空间 `ibdata1` 则不然），并且它确实能在运行该存储引擎的所有表中提供精确的行数。

#### MyISAM 的早期

过去，MyISAM 是一个了不起的存储引擎；它在 MySQL 5.5 之前是 MySQL 提供的唯一存储引擎，按当时的标准来看，它非常出色地完成了其功能。

MyISAM 基于 ISAM——一种索引顺序访问方法——由 IBM 开发，用于促进按顺序访问数据。这种访问方法允许轻松读取从第一条记录到最后一条记录的数据库文件，索引指向文件中的正确记录，从而使该方法易于理解。

MyISAM 很经济——数据在磁盘上占用的空间不大，由于存储引擎的内部设计，数据库中的表提供了精确的行数，并且没有“链接”到其他文件的文件（如果像 InnoDB 的 `ibdata1` 那样处理不当可能会使数据库混乱）。除此之外，MyISAM 支持一些 InnoDB 不支持的功能，使其成为许多开发者的极具吸引力的选择。即使 MyISAM 是一个非事务性存储引擎，考虑到这些因素，难怪开发者们长期以来都认为 MyISAM 是 InnoDB 的一个可行替代品：
| MyISAM 中的功能 | InnoDB 中的功能 |
| --- | --- |
| 支持全文索引。 | 从 MySQL 5.6 开始支持全文索引。 |
| 支持可移植表空间。 | 从 MySQL 5.6 开始支持可移植表空间。 |
| 支持空间索引。 | 从 MySQL 5.7 开始，InnoDB 中可用空间索引。 |
| 支持关于运行该存储引擎的表的最后更新时间。 | 该功能从 MySQL 5.7 开始在 InnoDB 中实现。 |

总的来说，功能相当不错，大家都接受了。但像所有事物一样，MyISAM 也并非没有缺点——随着时间的推移，这些缺点变得越来越明显。


#### MyISAM 的现状

如今，MyISAM 的主要缺点与数据不一致性和不符合 ACID 规范有关。由于其设计，MyISAM 是一个非事务性存储引擎，这意味着它不支持回滚或原子操作，因此成为许多开发者的“阿喀琉斯之踵”。

另一方面，MyISAM 的设计非常简单，这使得 InnoDB 中无法使用的功能在 MyISAM 中很容易实现——MyISAM 将其元数据中存储了运行该存储引擎的表的精确行数，并显示运行该存储引擎的任何表的行数；并且删除与该存储引擎关联的文件也能完整删除数据而没有任何问题。InnoDB 则无法做到这一点，因为它有表空间文件和日志文件，在我们启动恢复过程时需要扫描这些文件。

讽刺的是，MyISAM 的优点也使其缺点更加突出。MyISAM 在磁盘上占用的空间要小得多，但由于 MyISAM 不维护日志文件，它在从崩溃中恢复数据时根本无法查看这些日志；而且由于 MyISAM 不支持事务，它无法确保事务的完整性，因此在向数据库写入数据时，数据丢失的风险非常高。听起来像噩梦？因为它确实是。

这些问题一直延续至今，使 MyISAM 成为 InnoDB 的一个极好补充——但不再是一个可行的替代方案，这就是为什么 MyISAM 现在已被弃用。

如果你发现自己在使用 MyISAM，考虑将所有基于 MyISAM 的表转换为 InnoDB 以满足 ACID 规范：

1.  首先，运行下面的 SQL 查询获取使用 MyISAM 的表列表——将 *database* 替换为你的数据库名称：
    `SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_SCHEMA = 'database' AND ENGINE = 'MyISAM';`
2.  将所有这些表改为运行 InnoDB：
    `ALTER TABLE 'table_name' ENGINE = InnoDB;`
3.  如果需要，为你的数据库管理系统中的每个数据库重复这些步骤。

哇哦！现在你所有的表都运行 InnoDB 了——就这么简单。是时候学习我们心爱的 DBMS 中其他可用存储引擎的使用场景了！

#### 存储引擎使用场景

为什么有这么多存储引擎供我们选择——其中并无秘密：有些存储引擎将数据保存在内存中，有些支持类似 Excel 的格式，有些充当归档角色，有些旨在作为黑盒来测试我们的代码功能，还有一些是作为如何在 MySQL 中构建新存储引擎的示例而构建的。

尽管 InnoDB 作为许多通用场景、甚至是大数据（如果经过适当优化）的首选存储引擎当之无愧，但某些使用场景可能需要你熟悉 MySQL 中可用的不同存储引擎。我们现在就来做这件事。

#### MEMORY 和 TempTable 存储引擎

在 MySQL 4.1 之前，MySQL 有一个被称为“HEAP”的存储引擎——HEAP 存储引擎的目的是将数据存储在内存中，因此从 MySQL 4.1 开始，该存储引擎被称为 MEMORY。

MEMORY 存储引擎的奥秘就隐藏在名称本身中——该存储引擎将数据存储在内存中，这意味着一旦我们重启服务器或数据库因任何原因崩溃，我们的数据就会丢失。在现代世界，这种存储引擎对许多应用来说并不是一个非常实用的选择，但它有时可用于在内存中充当只读缓存的数据。

在这方面，较新版本的 MySQL 确实还有另一手准备——从 MySQL 8.3 开始，数据库管理系统基于 *TempTable* 存储引擎构建其所有内部临时表。通过调整 `internal_tmp_mem_storage_engine` 变量，可以将用于临时表的存储引擎切换为 MEMORY 存储引擎。

TempTable 和 MEMORY 存储引擎的主要区别在于：TempTable 存储引擎将所有与表相关的数据和信息存储在保存于磁盘上的一个文件中，而基于 MEMORY 存储引擎的表将数据存储在内存中，磁盘只保存 .frm（表定义）文件。TempTable 存储引擎是存储较大对象类型的一个好选择。

TempTable 存储引擎确实有三个主要参数可能对调整临时表的人有用：它们是 `tmp_table_size` 变量、`temptable_max_ram` 变量和 `temptable_max_mmap` 变量。第一个变量定义了临时表的最大大小，第二个变量定义了临时表在开始将数据存储到磁盘之前的最大大小（默认值为 1GB），最后一个变量定义了基于 TempTable 的表在开始将数据存储到磁盘之前可以占用的最大内存量。如果 TempTable 开始将数据存储到磁盘，数据将使用 InnoDB 存储引擎进行存储。

#### CSV 存储引擎

CSV 存储引擎正如其名——它将所有数据存储在具有类似 Excel 格式的文本文件中。值之间用逗号分隔，表总是带有一个 “.CSV” 扩展名。向表中写入数据意味着向 Excel 文件中写入数据。

基于 CSV 的表可以像普通表一样创建；你只需要在最后指定合适的存储引擎：

```sql
CREATE TABLE `demo_table` (
  `id` INT NOT NULL,
  `message` VARCHAR(225) NOT NULL,
  `message_from` VARCHAR(225) NOT NULL,
  `message_to` VARCHAR(225) NOT NULL,
  `time_sent` VARCHAR(225) DEFAULT CURRENT_TIMESTAMP,
  `time_received` VARCHAR(225) DEFAULT CURRENT_TIMESTAMP,
  ...
) ENGINE = CSV;
```

CSV 存储引擎确实有其优缺点——由于数据除了基于 Excel 的文件外不存储在任何其他文件中，因此可以通过修改表内或 Excel 文件本身的数据来修改文件内的数据——但该存储引擎也不支持索引或分区。



#### ARCHIVE 与 BLACKHOLE 存储引擎

其他的存储引擎也并非高深莫测。`ARCHIVE` 存储引擎旨在充当数据的归档库——其行压缩能力使得基于该引擎的数据在磁盘上占用空间极小，并且利用其行级锁功能，可以轻松地对数据进行备份。同时，使用此引擎存储的表不具备任何 `MVCC`、`B-Tree`、外键、全文索引、哈希或地理空间索引功能。

`BLACKHOLE` 存储引擎旨在充当一个“黑洞”，对于进行内部测试的人员可能非常有用——写入该引擎的数据只能被写入，但无法被检索，因为数据从一开始就没有被真正存储，因此得名：所有写入基于此引擎的表的数据都会瞬间消失。任何时候磁盘上都不会存储任何文件。此引擎与 `ARCHIVE` 一样，不支持分区，但它支持各种索引，最大可用键长为 3072 字节。

`BLACKHOLE` 存储引擎的内在功能使其具有许多有趣的用例：虽然该引擎不在磁盘上存储任何文件，但如果启用了基于语句的二进制日志记录，它可以将 SQL 查询复制到副本服务器，这使其成为复制设置中的好伙伴。假设你有一个基于 `BLACKHOLE` 存储引擎的表：

```sql
CREATE TABLE `demo_table` (
`id` INT NOT NULL,
`message` VARCHAR(225) NOT NULL,
`message_from` VARCHAR(225) NOT NULL,
`message_to` VARCHAR(225) NOT NULL,
`time_sent` VARCHAR(225) DEFAULT CURRENT_TIMESTAMP,
`time_received` VARCHAR(225) DEFAULT CURRENT_TIMESTAMP,
...
) ENGINE = BLACKHOLE;
```

一旦你对上述表执行任何（`CRUD`）操作，`MySQL` 会将这些操作写入二进制日志，然后该日志将被复制到从服务器。这意味着，如果你在从服务器上创建一个具有相同结构的表，并将其设置为使用 `InnoDB` 存储引擎，`MySQL` 会将二进制日志中的更改复制到从服务器，包括插入的数据。

换句话说，如果你希望避免在主服务器上存储数据，但同时又想在从服务器上看到这些数据，`BLACKHOLE` 就是适合你的存储引擎——只需确保从服务器上的表具有相同的结构，并且基于一个可靠、能保存数据而非将其排出的存储引擎。

`BLACKHOLE` 存储引擎的另一个流行用例是验证备份文件的语法——如果你的逻辑备份被“吞入深渊”，则说明它们的语法是正确的。

#### MERGE 存储引擎

`MySQL` 还有一个专门用于合并源自 `MyISAM` 的数据的存储引擎——过去，这个存储引擎被称为 `MRG_MyISAM`，但如今，它有了一个更简洁的名称来描述其用途——该引擎被命名为 `MERGE`。

`MERGE` 存储引擎很容易解释：它是一组具有相同结构的 `MyISAM` 表的集合。一旦数据库中出现一个基于 `MERGE` 存储引擎的表，你将不会在磁盘上看到具有 `.MYD` 或 `.MYI` 扩展名的常规文件——取而代之的是会创建一个扩展名为 `.MRG`（merge）的文件。这些表的表格式将存储在数据字典中，而任何 `.MRG` 文件的内容仅包含运行 `MyISAM` 存储引擎的表名，并且这些表将被视为一个基于 `MyISAM` 的表。

##### 优点

在 `MySQL` 中，你可能不会经常用到 `MERGE` 存储引擎提供的优势，但其中一些可能比你想象的要更常遇到：

*   由于 `MyISAM` 是一个非事务型存储引擎，其表的修复是不可避免的。如果你的应用场景需要使用大型 `MyISAM` 表，那么将它们拆分成多个 `MERGE` 表，并改为修复多个独立的表，可能会比修复单个基于 `MyISAM` 的表快得多。
*   `MERGE` 可以突破操作系统设定的文件大小上限。对于 `MyISAM` 而言，遵守此限制是必须的——而对于 `MERGE`，则不是。

##### 缺点

话虽如此，`MERGE` 确实存在一些问题。`ALTER` 查询会破坏与关联表的合并结构，`REPLACE` 查询的可靠功能可能会丢失，并且像 `ALTER TABLE`、`DROP TABLE`、`OPTIMIZE TABLE`、`REPAIR TABLE`、`ANALYZE TABLE` 或不带 `WHERE` 子句的 `DELETE` 这样的查询可能会由于 `MySQL` 引用的是主表而非合并表本身，而导致结果不正确。

所有这些因素使得 `MERGE` 存储引擎成为一个有缺陷但仍有价值的补充，即使在 `MyISAM` 已经过时的情况下也是如此。

#### 高可用性存储引擎

除了数据需要合并，你的数据还需要高可用性。`MySQL` 确实提供了一个专门用于此目的的存储引擎——它叫做 `NDB`。开头的“N”并非无缘无故：`NDB` 存储引擎代表网络数据库，并采用无共享架构。`NDB` 是一个高可用性存储引擎，始终连接到一个数据集群，因此它也被称为 `NDBCluster`。

每个 `NDB` 实例至少需要三个节点，分别是数据节点、SQL 节点和管理节点：

1.  数据节点，顾名思义，存储与集群相关的数据。在典型的 `NDB` 设置中，建议拥有两个或更多数据节点以为数据库提供冗余。
2.  SQL 节点用于在数据节点上运行 SQL 查询。换句话说，我们使用 SQL 节点通过 SQL 来访问数据节点上的数据。
3.  管理节点用于管理 `NDB` 中的所有节点。


### 安装与使用 NDB

值得一提的是，你无法使用许多人习惯的常规引擎定义方式来创建基于 NDB 的表。用户运行类似下面这样的 SQL 查询并遇到 `error #1286` 错误（MySQL 报错 “Unknown storage engine ‘NDBCLUSTER’”）的情况并不少见：

```
CREATE TABLE `demo_table` (
`id` INT NOT NULL,
`message` VARCHAR(225) NOT NULL,
`message_from` VARCHAR(225) NOT NULL,
`message_to` VARCHAR(225) NOT NULL,
`time_sent` VARCHAR(225) DEFAULT CURRENT_TIMESTAMP,
`time_received` VARCHAR(225) DEFAULT CURRENT_TIMESTAMP,
...
) ENGINE = NDBCluster;
```

你遇到此错误的原因是标准版本的 MySQL 不支持 NDB 存储引擎——请确保改用 MySQL NDB Cluster。

成功下载 NDB 并安装下载的文件后，你通常需要为运行 MySQL 的数据节点创建数据目录，并确保该目录的所有者是 `mysql`，如下所示：

```
mkdir –p /var/lib/mysql/
chown [–R] mysql:mysql /var/lib/mysql/
```

这里，`-p` 选项表示“父级”（`mkdir` 将创建目录以及任何其他尚不存在的目录），而关于 `chown` 的 `-R` 选项将使其不跟随符号链接。

之后，你需要通过运行类似以下的命令为 NDB 数据节点创建一个服务：

```
vi /etc/systemd/system/ndbd.service
```

拥有服务后，你需要在所有适用的节点上编辑你的 `my.cnf` 文件来配置 `NDBCluster`。在 `[mysqld]` 下面添加如下内容：

```
ndbcluster
ndb-connectstring=#
```

这里的 `#` 是管理节点的 IP 地址。

你还需要有一个 `[mysql_cluster]` 部分，并在那里指定相同的 `ndb-connectstring`，因此所有内容看起来应该是这样的：

```
[mysqld]
ndbcluster
ndb-connectstring=#
[mysql_cluster]
ndb-connectstring=#
```

别忘了将井号替换为你的管理节点 IP。

完成上述步骤后，运行 `systemctl enable ndbd` 以启用 NDB 服务。

接下来，你需要创建管理节点。在该节点上，你需要下载并安装 NDB 的管理服务器，然后为 NDB 集群创建一个目录，最后创建配置文件。你的配置文件应该类似于这样：

```
[ndbd default]
#### 影响所有数据节点上 ndbd 进程的选项
NoOfReplicas=2 # 副本数量
[ndb_mgmd]
#### NDB 管理节点
hostname=127.0.0.1 # 管理节点的 IP
datadir=/usr/ndb-cluster/data  # 数据目录
[ndbd]
#### 第一个数据节点
hostname=127.0.0.2 # 第一个数据节点的 IP
NodeId=2 # 此数据节点的 ID
datadir=/var/lib/mysql/ # 数据目录
[ndbd]
#### 第二个数据节点
hostname=127.0.0.3 # 第二个数据节点的主机名/IP
NodeId=3 # 此数据节点的 ID
datadir=/var/lib/mysql/ # 数据文件的远程目录
[mysqld]
NodeId=4
hostname=127.0.0.2 # NDBCluster 节点 1
[mysqld]
NodeId=5
hostname=127.0.0.3 # NDBCluster 节点 2
```

现在，在管理节点上创建 NDB 管理服务。创建一个名为 `/etc/system/system/ndb_mgmd.service` 的文件，并确保它包含三个部分——`[Unit]`、`[Service]` 和 `[Install]`。你可以在 GitHub 仓库中轻松找到此文件的结构。最后，通过运行 `systemctl enable ndb_mgmd` 来启用管理服务。

完美！继续最后一步，通过在管理节点上运行 `ndb-mgmd –-initial –-config-file=[location/of/config/file]` 来向你的管理节点提供配置文件的位置，然后在每个数据节点上运行 `ndbd`，或者进入配置目录并使用 `ndbd` 启动数据节点。

最后，在每个数据和 SQL 节点上运行 `service mysql start` 来启动 MySQL，这样就大功告成了。

一旦设置好 NDB，你就开启了一个全新的世界。现在，你可以将表设置为使用 `NDBCLUSTER` 或 `NDB` 存储引擎，并行运行 SQL 查询，享受多主复制功能等等！

通过 CLI 调用 `ndb_mgm` 来使用管理客户端，登录后调用 `SHOW` 来检查你的集群配置。

### FEDERATED 和 EXAMPLE 存储引擎

除了 NDBCluster，还有另外两种存储引擎我需要你了解。你们大多数人可能都听说过它们，但从未使用过——这没关系。这些存储引擎并非日常工作的必需品，但它们潜伏在阴影中，等待着下一次机会。虽然这个机会确实很少出现——这些存储引擎就是 `FEDERATED` 和 `EXAMPLE`。

第一个允许我们从远程数据库访问数据，第二个则作为如何构建新存储引擎的示例。

为了充分利用 `FEDERATED` 存储引擎，请利用 `CONNECTION` 选项来定义你在另一台服务器上创建的表，如下所示——这里 `federated_user` 是 `remote_server` 中的用户，`9306` 定义了 TCP 协议，`database` 定义了你的数据库名称，而 `demo_table` 指的是该数据库上的表：

```
CREATE TABLE `federated_table` (
`message_id` INT(15) NOT NULL AUTO_INCREMENT PRIMARY KEY,
`message_title` VARCHAR(120) NOT NULL,
`message_user` VARCHAR(120) NOT NULL,
`message_contents` VARCHAR(255) DEFAULT NULL,
...
) ENGINE = FEDERATED
DEFAULT CHARSET = ...
CONNECTION = 'mysql://federated_user@remote_server:9306/database/demo_table';
```

很好！现在，利用你刚刚建立的这个连接，创建一个具有相同结构的基于 InnoDB 的表：

```
CREATE TABLE `demo_table` (
`message_id` INT(15) NOT NULL AUTO_INCREMENT PRIMARY KEY,
`message_title` VARCHAR(120) NOT NULL,
`message_user` VARCHAR(120) NOT NULL,
`message_contents` VARCHAR(255) DEFAULT NULL,
...
) ENGINE = InnoDB;
```

好的！现在，一旦你向 `demo_table` 添加数据，相同的数据也会出现在 `federated_table` 中。很棒，对吧？

还有一种方法可以使用 `CREATE SERVER` 查询来创建相同的联邦表。示例如下：

```
CREATE SERVER demo_server FOREIGN DATA WRAPPER mysql OPTIONS (USER 'federated_user', HOST 'federated_host', PORT 9306, DATABASE 'demo_database');
```

这里，`demo_server` 是你的数据库服务器名称，`mysql` 定义了我们的 DBMS，`OPTIONS` 定义了用户名和主机，而 `database` 定义了你的数据库标题。

对于 `EXAMPLE` 存储引擎，一切都要简单得多——它作为如何构建新存储引擎的示例。该存储引擎不支持任何必要的 CRUD 操作、分区或索引，但它仍然在那里等待未来催生新存储引擎的机会：在 MySQL 的 `storage` 目录中找到其目录，并查看 `example` 子目录。


#### MariaDB 专属存储引擎

好了——现在你了解了 MySQL 中所有关于存储引擎的知识！嗯，几乎是全部。

你之所以知道*几乎所有*，是因为 MySQL 并非唯一参与其中的——我已经向你介绍了它的兄弟 Percona Server，而且 MySQL 也有一个名为 Maria 的姐妹。MySQL 的姐妹 MariaDB 以其存储引擎而闻名。MariaDB 支持 MySQL 中所有可用的存储引擎，甚至更多——包括 MariaDB 专属的存储引擎。

这些存储引擎包括 Aria、流行的 MariaDB ColumnStore 存储引擎、CONNECT、FederatedX、Mroonga、OQGraph、S3 和 Sequence 存储引擎，以及 SphinxSE 和 Spider。哦，还有 MyRocks 和 TokuDB——你已经对它们很熟悉了。

别担心——MariaDB 中许多可用的存储引擎其功能是自解释的：Aria 是 MyISAM 的崩溃安全版本，从 MariaDB 10.4 开始被用于系统表；ColumnStore 通过列式存储扩展了 MariaDB；CONNECT 使 MariaDB 能够以不同格式“连接”数据；FederatedX 是`FEDERATED`存储引擎的一个变体，旨在保持`FEDERATED`的功能更新，避免其彻底被弃用；Mroonga 适用于通过全文搜索功能搜索韩语、日语和中文数据；OQGraph 适用于存储图及相关数据的用户；S3 对于将数据存储在 Amazon S3 服务器中的用户来说是完美的；而 Sequence 存储引擎允许存储数字序列。序列可以是升序或降序的。

还有 SphinxSE 或 Sphinx 以及 Spider——Sphinx 充当 MariaDB 中全文搜索的替代方案，而 Spider 则将不同的 MariaDB 实例“连接”成一个，就好像它们在同一个数据库实例上运行一样。

有趣吧？MariaDB 中的存储引擎也有其独特的秘密——你听说过目前没有已知方法创建一个运行 Sequence 存储引擎的表吗？这类表也是只读的；你可以像这样使用这个存储引擎：

```
MariaDB [database_name]> SELECT * FROM seq_1_to_10;
+-----+
| seq |
+-----+
|   1 |
|   2 |
|   3 |
|   4 |
|   5 |
|   6 |
|   7 |
|   8 |
|   9 |
|  10 |
+-----+
10 rows in set (0.002 sec)
```

你也可以指定步长以及希望 MariaDB 执行的步数——如果你这样做，MariaDB 会“跳过”某些数字——例如，这里我们显示从 10 到 1 每隔第三个数字，因此跳过了每 3 个数字：

```
MariaDB [tests]> SELECT * FROM seq_10_to_1_step_3;
+-----+
| seq |
+-----+
|  10 |
|   7 |
|   4 |
|   1 |
+-----+
4 rows in set (0.002 sec)
```

这还不是全部——S3 存储引擎也拥有强大的能力。要使用 S3，你必须知道你的 S3 访问密钥和私有密钥，以及数据应存储的存储桶和区域。一旦已知，通过`my.cnf`为 S3 存储引擎配置对你的存储桶的访问权限，然后就万事俱备了。别忘了通过`my.cnf`指定`s3-host-name`、`s3-bucket`、`s3-access-key`和`s3-secret-key`，并将`s3`变量设置为`ON`，其余部分应该就能正常工作了。现在，你可以通过简单地切换引擎，将数据从 MariaDB 中已存在的表移动到 S3——如果你愿意，还可以指定压缩算法：

```
ALTER TABLE data_table ENGINE=S3 [COMPRESSION_ALGORITHM=zlib]
```

剩下的就留给你自己了——探索 MySQL、MariaDB 和 Percona Server 中的存储引擎选项，并准备好“破坏”你自己的 MySQL 实例吧。

### 总结

存储引擎是 MySQL 的关键部分，虽然有些引擎可能不如其他常用，但它们都有各自独特的优缺点。许多开发人员和工程师使用的主要存储引擎是`InnoDB`，但还有很多其他选项供你选择。无论好坏，它们都有各自的用处，一旦你发现自己正在使用任何存储引擎的 MySQL，至关重要的一点是：利用它们的优势，并避免陷入性能陷阱。

如果不了解数据库被“破坏”和“撕裂”的方式，就不可能避免陷入性能陷阱。既然你已经重温了关于存储引擎的知识，也是时候去“破坏”它们了。当然，是安全地进行。

