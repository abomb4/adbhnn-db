# 术语表

### ACID
> Atomic, Consistency, Isolation and Durability.

原子性、一致性、隔离性和持久性。

### MVCC
`MVCC (multiversion concurrency control)` 多版本并发控制。
该技术使 `InnoDB` 事务可以在指定的 `隔离级别` 下施行一致读操作。
就是说，当其他事务正在修改某行数据时，可以查询查询事务更新数据之前的值。
这种吊炸天的技术可以使查询语句不需要等待其他事务的锁，大大提高并发效率。
