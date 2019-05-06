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
