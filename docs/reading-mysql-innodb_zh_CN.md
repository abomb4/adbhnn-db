# MySQL InnoDB 文档阅读记录
所有数据库技术中，我第一个想要了解的是存储引擎。

所以打算从 `MySQL` `InnoDB` 文档入手，开始学习。

这篇文档大部分都是 `MySQL` 官方文档的中文翻译，加上一些个人理解。

原文档地址：[https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)

## InnoDB 简介
`InnoDB` 是一种一般用途存储引擎，它在高可用和高性能之间做了平衡，是当前 `MySQL` 的默认存储引擎。

`InnoDB` 的优势：
- 所有 `DML` (`Data Manipulation language`, `insert`, `update`, `delete`, `etc`)
语句都遵循 `ACID` 模型，利用事务提交、回滚和异常修复能力特性来保护用户数据。
- 行粒度的锁，踢桃多用户并发能力与性能
- 使用主键来管理磁盘上的数据，每个表都有一个称为“聚簇索引”的主键，加速主键查询速度。
- 为了保持数据完整性，`InnoDB` 支持外键约束。

## 术语表

### ACID
> Atomic, Consistency, Isolation and Durability.

原子性、一致性、隔离性和持久性。

### MVCC
`MVCC (multiversion concurrency control)` 多版本并发控制。
该技术使 `InnoDB` 事务可以在指定的 `隔离级别` 下施行一致读操作。
就是说，当其他事务正在修改某行数据时，可以查询查询事务更新数据之前的值。
这种吊炸天的技术可以使查询语句不需要等待其他事务的锁，大大提高并发效率。
