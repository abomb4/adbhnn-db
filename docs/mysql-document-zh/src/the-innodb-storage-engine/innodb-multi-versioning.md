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
