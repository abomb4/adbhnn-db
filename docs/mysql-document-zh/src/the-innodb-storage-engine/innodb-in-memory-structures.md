## InnoDB 内存数据结构

本节描述 `InnoDB` 存储引擎的内存数据结构。

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

默认情况下，被一些查询读取的 `页` 会立刻移动到新列表，意味着该 `页` 会在 `缓冲池` 留存更长的时间。
但当发生全表查询（例如 `mysqldump` 指令，与 `SELECT` 语句不加 `WHERE`）时，
大量的数据会放入 `缓冲池` 中，并有等量数据被过期，即使新数据再也没被使用过也一样。
类似地，由 `预读取` 后台线程加载并只访问一次的 `页` 会移动到新列表的头部。
这些行为会导致一些常用的 `页` 被挪到“旧列表”，成为过期候选数据。
若要了解如何优化这些行为，可以参考
[Section 15.8.3.3, “Making the Buffer Pool Scan Resistant”](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-midpoint_insertion.html)
和
[Section 15.8.3.4, “Configuring InnoDB Buffer Pool Prefetching (Read-Ahead)”](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-read_ahead.html)

`InnoDB` 标准监控输出包含一系列有关 `缓冲池` `LRU` 算法的字段，在 `BUFFER POOL AND MEMORY` 节有说明。
详情参阅
[Monitoring the Buffer Pool Using the InnoDB Standard Monitor](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html#innodb-buffer-pool-monitoring)

#### 缓冲池的配置
用户可以从一些方面对 `缓冲池` 进行配置，提高整体性能：
-


### 修改缓冲区 Change Buffer


### 自适应哈希索引 Adaptive Hash Index


### 日志缓冲区 Log Buffer
