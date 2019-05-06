# MySQL InnoDB 文档阅读记录
所有数据库技术中，我第一个想要了解的是存储引擎。

所以打算从 `MySQL` `InnoDB` 文档入手，开始学习。

这篇文档大部分都是 `MySQL` 官方文档的中文翻译，加上小部分的个人理解。

原文档地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)

## InnoDB 简介
`InnoDB` 是一种一般用途存储引擎，它在高可用和高性能之间做了平衡，是当前 `MySQL` 的默认存储引擎。

`InnoDB` 的主要优点：
- 所有 [`DML`](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_dml)
  （就是 `insert`, `update`, `delete` 这种）
  语句都遵循 `ACID` 模型，利用了事务中的提交、回滚和异常修复功能来保护用户数据。
- 行粒度的锁，提高并发能力与性能
- 使用主键来管理磁盘上的数据，每个表都有一个称为“聚簇索引”的主键，加速主键查询速度。
- 为了保持数据完整性，`InnoDB` 支持外键约束。

### 使用 `InnoDB` 带来的好处
// TODO 好处很多就先不翻译了

### `InnoDB` 最佳实践（Best Practice）
本节描述使用 `InnoDB 表` 的最佳实践：

- 每个表指定一个主键，可以使用最频繁作为查询条件的一列或多列。如果没有明显可作为主键的列，
  可以使用 `auto_increment` 作为主键。
- 在多表关联场景，可以在几个表中定义相同数据类型的列，
  并使用外键 [`Foreign Keys`](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_foreign_key)
  可以提高 `join` 效率。
  使用外键可以确保关联列都有索引，可以提高性能。
  外键还可以使 `update` `delete` 语句对所有关联的表产生作用，防止插入无效数据。
- 关掉 (`autocommit`)[https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_autocommit]。
  频繁进行提交（`COMMIT` 操作）会降低性能。
- 将一大堆相互关联的 `DML` 语句组合成一个事务，也就是用 `START TRANSACTION` 和 `COMMIT` 括起来。
  我们不希望每次都提交，但也不希望存在一大堆没提交的 `INSERT`, `UPDATE`, `DELETE` 语句。
- 避免使用 [`LOCK TABLES`](https://dev.mysql.com/doc/refman/8.0/en/lock-tables.html) 语句。
  `InnoDB` 可以在不牺牲可靠性与性能的情况下，处理多个客户端同时读写一个表的场景。
  如果需要独占写，可以使用 (`SELECT ... FOR UPDATE`)[https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html]
  语法来锁住需要独占的行。
- 开启 `innodb_file_per_table` 选项，或者使用“一般表空间”来将数据和索引拆成不同的文件。
  避免使用系统表空间（[system tablespace](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_system_tablespace)）。
  （`innodb_file_per_table` 选项默认是开启的。）
- 评估数据是否可以受益于 `InnoDB` [压缩](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_compression) 功能，
  `InnoDB` 表在合适的情况可以被压缩，而不影响读写功能。
- 如果建表语句存在非 `InnoDB` 引擎，可以开启 `--sql_mode=NO_ENGINE_SUBSTITUTION` 选项保证使用的是 `InnoDB` 。

### `InnoDB` 性能评估
可以使用 `SHOW ENGINES;` 语句，或 `SELECT * FROM INFORMATION_SCHEMA.ENGINES;` 来确认表的存储引擎。
[https://dev.mysql.com/doc/refman/8.0/en/innodb-benchmarking.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-benchmarking.html)


## InnoDB 与 ACID 模型
`ACID` 模型是在数据库设计中强调数据可靠性的一系列设计原则，
数据可靠性在业务数据和一些关键任务中非常的重要。
`MySQL` 中有一些组件，例如 `InnoDB` 存储引擎，设计上与 `ACID` 紧密关联，
当出现异常状况（例如软件崩溃、硬件故障）时数据不会损坏，也不会变得混乱。
依赖于 `ACID` 兼容的特性，用户无需为了一致性检查与崩溃数据恢复机制重新造轮子。
当用户拥有自己额外的软件安全机制、超稳定的硬件，或者允许数据出现一些不一致或丢失的场景下，
可以调整 `MySQL` 的 `ACID` 相关配置，以获得更好的性能或吞吐量。

以下讨论 `InnoDB` 存储引擎的功能与 `ACID` 模型的关联。

### Atomicity 原子性
涉及 `ACID` 模型中的 `原子性` 概念的主要是 `InnoDB` 的 `事务` 特性，相关功能包括：

- [`Autocommit`](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_autocommit) 的配置
- `COMMIT` 语句
- `ROLLBACK` 语句
- `INFORMATION_SCHEMA` 表中的运行数据

### Consistency 一致性
涉及 `一致性` 概念的功能主要是 `InnoDB` 当发生故障时保护数据的处理机制。相关功能包括：

- `InnoDB` 双写缓冲区（[doublewrite buffer](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_doublewrite_buffer)）
- `InnoDB` 故障恢复（[crash recovery](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_crash_recovery)）

### Isolation 隔离性
涉及 `隔离性` 的主要是 `InnoDB` 事务功能，
特别是事务中的 [隔离级别](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_isolation_level) 机制。
相关功能：

- `Autocmimit` 配置
- `SET ISOLATION LEVEL` 语句
- `InnoDB` [锁](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_locking) 的底层实现机制。
  在性能调整期间，可以在 `INFORMATION_SCHEMA` 中查看到关于锁的细节。（？）

### Durability 持久性
`持久性` 对应 `MySQL` 软件功能与特定硬件配置的交互行为。由于 CPU、网络、存储设备等的各种兼容问题，
很难在持久性方面给出一个具体的准则。

`持久性` 涉及的各种特性与配置包括：
- `InnoDB` 的 `doublewrite buffer`，可以通过 `innodb_doublewrite` 配置来开关功能。
- [`innodb_flush_log_at_trx_commit`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit) 配置项。
- [`sync_binlog`](https://dev.mysql.com/doc/refman/8.0/en/replication-options-binary-log.html#sysvar_sync_binlog) 配置项。
- [`innodb_file_per_table`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_file_per_table) 配置项。
- 缓冲区写入外存，如写入到硬盘、 SSD 或 RAID 阵列。
- 支持电池的存储设备。
- 运行 `MySQL` 的操作系统，特别是支持 `fsync()` 系统调用的系统。
- 使用 UPS 电源，保证断电后有机会持久化数据。
- 用户的备份策略。
- 分布式环境中，`MySQL` 数据中心的服务器硬件位置，与各个数据中心的网络连接。


## InnoDB 多版本控制
`InnoDB` 是一个多版本存储引擎 （[multi-versioned storage engine](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_mvcc)）：
它存储被修改的行的旧版本，用来支持事务特性，例如并发和回滚。
旧版本信息使用一种称为 `回滚段`（[rollback segment](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_rollback_segment)）
的数据结构存储在 `表空间` 中。`InnoDB` 使用 `回滚段` 中的信息来实现事务回滚功能中的回退操作。
`InnoDB` 还可以根据 `回滚段` 信息来构造旧版本的行信息，实现 `一致读`（[consistent read](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_consistent_read)）。

实现方面，`InnoDB` 在数据库中每一行都添加三个域：
- 6 字节的 `DB_TRX_ID` ，存储最后修改这一行的事务的唯一标识符。
  这里将删除操作视为修改操作，删除实际上是对这行中的一个特殊的标志进行修改，来标记这行“已删除”。
- 7 字节的 `DB_ROLL_PTR` ，称为滚动指针 `roll pointer` 。滚动指针指向一个 `回滚段` 中的 `回滚日志` 记录。
  如果某行被更新，`回滚日志` 记录可以构造出某行更新前的内容。
- 6 字节的 `DB_ROW_ID` ，存储一个仅仅在插入新行时自动增加的 `Row ID` 。
  如果 `InnoDB` 自动生成一个 `聚簇索引` ，那这个索引中就包含了 `row ID` 信息。
  否则 `DB_ROW_ID` 不会出现在任何其他索引中。

`回滚段` 中的 `回滚日志`分为插入类和修改类日志。
插入类日志尽在事务回滚时使用，而且事务提交后就可以丢弃了。
修改类日志还应用于 `一致读` 场景，但是只有在 `InnoDB` 没有分配快照的事务之后才能丢弃它们，
在一致读取中可能需要更新撤消日志中的信息来构建数据库行的早期版本。（？太长了不知道怎么翻译）

用户需要有计划地提交事务，那些只涉及一致读的事务也需要注意。否则， `InnoDB` 就无法丢弃 `更新重做日志` 中的数据，
`回滚段` 会变得越来越大，占满表空间。

`回滚段` 中的 `回滚日志` ，实际占用空间一般小于相应的插入或修改的行。用户可以据此计算 `回滚段` 的所需物理空间。

在 `InnoDB` 的多版本控制方案中，一个 `delete` 语句执行过后，行并没有立刻从数据库中物理删除。
`InnoDB` 只有在删除语句产生的 `更新重做日志` 被删除时（基本可以理解为，针对相关行的所有事务都提交了），
才会对相应的行进行物理删除。
这种物理删除操作称为 `清除` （[purge](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_purge)）。
`清除` 操作是比较快的，基本与 `delete` SQL 语句的速度以一致的。

如果对一张表，用近乎相同的频率进行小批量的 `insert` 与 `delete` 操作，
那么此时清除线程（Purge Thread）可能会开始变得落后，表也会变得越来越大，
因为在不停地增删增删的过程中产生的“死行”越来越多，使得很多操作都直接与磁盘绑定，变得缓慢。
这种情况下，可以限制新行的操作频率（？），并且通过修改
[`innodb_max_purge_lag`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_max_purge_lag)
配置以分配更多的资源给清除线程。详情可以查看
[15.13, "InnoDB Startup Options and System Variables"](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html)
节。

### 多版本控制与二级索引
`InnoDB` 多版本并发控制机制处理 `二级索引` （[secondary index](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_secondary_index)）
的方式与 `聚簇索引` 是不一样的。根据 `聚簇索引` 可以直接对记录进行就地更新，
并且记录中的系统隐藏列（大概就是 `DB_ROLL_PTR`）指向了 `回滚日志` ，可以据此构建一条记录的历史版本。
而 `二级索引` 中不包含系统隐藏列，也无法进行就地更新。

当 `二级索引` 中的列进行了更新时，原来的二级索引记录被标记为删除，并插入一条新纪录，
最后被标记删除的原始记录被 `清除`。
当一个 `二级索引` 记录被标记为删除，或 `二级索引` 所在的 `page` 被新的事务更新时，
`InnoDB` 会尝试在聚簇索引中检索数据库记录（大概就是不用索引了直接扫描？）。
在 `聚簇索引` 中，若一条记录在一个读事务生成后被更新，则会检查该记录的 `DB_TRX_ID` ，然后通过 `回滚日志` 检索这条记录的正确版本。

如果 `二级索引` 记录被标记为要删除，或者 `二级索引` 的 `page` 被 `update` 事务更新，
则不会使用 `覆盖索引`（[covering index](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_covering_index)）
技术（当查询列和搜索条件列全部位于某个 `二级索引` 中而使用的索引，
会直接从索引列中返回记录而不是再根据主键去查 `聚簇索引`）。
此时 `InnoDB` 没有从索引结构返回记录值，而是在 `聚簇索引` 中查找记录。

然而，若启用了 `ICP`
（[index condition pushdown](https://dev.mysql.com/doc/refman/8.0/en/index-condition-pushdown-optimization.html)）
优化，并且 `WHERE` 条件的某些部分仅使用索引列即可计算求值，
`MySQL` 服务则会继续将这部分可以使用索引的 `WHERE` 条件下推（下推意味着提前执行）到存储引擎中进行计算。
若没有符合条件的记录，则避免了进行 `聚簇索引` 的再查找。
若查到了记录，即时是标记为删除的记录，`InnoDB` 也会进行 `聚簇索引` 查找。


## InnoDB 架构
此图摘自官网，显示 `InnoDB` 内存与磁盘存储架构。下两节内容会详细讲解 `InnoDB` 内存架构和 `InnoDB` 磁盘存储架构。

![MySQL 官网架构图](https://dev.mysql.com/doc/refman/8.0/en/images/innodb-architecture.png "MySQL 官网架构图")


## InnoDB 内存架构
### 缓冲池 Buffer Pool
`缓冲池` 是主内存中的一片内存区域，在访问表和索引数据时对她们进行缓存。
`缓冲池` 允许经常使用的数据可以直接从内存中访问，以加快处理速度。
在专门的数据库服务器上，经常会将高达 80% 的物理内存分配给 `缓冲池`。

为支持大量的读取操作，`缓冲池` 拆分成一堆可能包含很多行的 `页`
（[page](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_page)）
来提高读取效率。
为了提高缓存管理效率，`缓冲池` 以链表的形式实现 `页` ；
不常用的缓存数据的过期方式采用了 `LRU` 算法的一种变体。

#### 缓冲池 LRU 算法
`缓冲池` 被一种（LRU）算法的一种变体作为列表来管理。当需要添加一个新的 `页` 到 `缓冲池` 时，
最近最少使用的 `页` 就过期掉，新的页会加到列表中间。
插入到列表中间的策略将整个列表视为两个子列表：

- 在在顶部，是最近访问的新（年轻）页面的子列表
- 在尾部，是最近访问较少的旧页面的子列表

![Buffer Pool List](https://dev.mysql.com/doc/refman/8.0/en/images/innodb-buffer-pool-list.png "Buffer Pool List")

该算法将经常在语句中用到的 `页` 放在“新列表”。“旧列表”包含一些不常用的 `页` ；这些页是被过期的候选页面。

默认情况下，算法拥有下列行为：
- 整个缓冲池的 3/8 用于“旧列表”
- 整个列表的中点是“新列表”的尾部指针与“旧列表”的头部指针接触的地方
- 当 `InnoDB` 读取一个页放到 `缓冲池` 时，该页一开始会放在中点位置（“旧列表”的头部）。
  页面是可以读取的，因为它是用户指定的操作（如SQL查询）或 `InnoDB` 自动执行的 `预读取`
  （[read-ahead](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_read_ahead)）
  操作的一部分所必需的。
- 访问“旧列表”中的 `页` 会将该 `页` 标记为“年轻”，并将其放入整个 `缓冲池` 的头部（即“新列表”的头）。
  若一个 `页` 被用户读取，那么 `first access` 会立刻生效， `页` 会被标记为“年轻”；
  若一个 `页` 被 `预读取` 操作所读取，那么 `first access` 不会立刻生效（可能一直到回收都没有生效）；
- 随着数据库的运行，缓冲池中未被访问的 `页` 会通过移向列表末尾而“老化”。
  新旧子列表中的 `页` 都会随着其他 `页` 的更新而“老化”。
  旧列表中的页面还会在其他页面插入到 `缓冲池` 中点时发生老化。
  最终，未使用的页面会到达旧列表的底部，然后被过期掉。




### 修改缓冲区 Change Buffer


### 自适应哈希索引 Adaptive Hash Index


### 日志缓冲区 Log Buffer



## InnoDB 磁盘架构


# 术语表

### ACID
> Atomic, Consistency, Isolation and Durability.

原子性、一致性、隔离性和持久性。

### MVCC
`MVCC (multiversion concurrency control)` 多版本并发控制。
该技术使 `InnoDB` 事务可以在指定的 `隔离级别` 下施行一致读操作。
就是说，当其他事务正在修改某行数据时，可以查询查询事务更新数据之前的值。
这种吊炸天的技术可以使查询语句不需要等待其他事务的锁，大大提高并发效率。
