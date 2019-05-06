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
