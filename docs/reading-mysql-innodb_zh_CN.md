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
// TODO

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

## 术语表

### ACID
> Atomic, Consistency, Isolation and Durability.

原子性、一致性、隔离性和持久性。

### MVCC
`MVCC (multiversion concurrency control)` 多版本并发控制。
该技术使 `InnoDB` 事务可以在指定的 `隔离级别` 下施行一致读操作。
就是说，当其他事务正在修改某行数据时，可以查询查询事务更新数据之前的值。
这种吊炸天的技术可以使查询语句不需要等待其他事务的锁，大大提高并发效率。
