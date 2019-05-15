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
- 随着数据库的运行，缓冲池中所有未被访问的 `页` 会通过移向列表末尾而“老化”。
  新旧子列表中的 `页` 都会随着其他 `页` 的更新而“老化”。
  旧列表中的 `页` 还会在其他 `页` 插入到 `缓冲池` 中点时发生老化。
  最终，未使用的 `页` 会到达旧列表的底部，然后被过期掉。

默认情况下，被一些查询读取的 `页` 会立刻移动到新列表，意味着该 `页` 会在 `缓冲池` 留存更长的时间。
但当发生全表查询（例如 `mysqldump` 指令，与 `SELECT` 语句不加 `WHERE`）时，
大量的数据会放入 `缓冲池` 中，并有等量数据被过期，即使新数据再也没被使用过也一样。
类似地，由 `预读取` 后台线程加载并只访问一次的 `页` 会移动到新列表的头部。
这些行为会导致一些常用的 `页` 被挪到“旧列表”，成为过期候选数据。
若要了解如何优化这些行为，可以参考
[Section 15.8.3.3, “Making the Buffer Pool Scan Resistant”](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-midpoint_insertion.html)
和
[Section 15.8.3.4, “Configuring InnoDB Buffer Pool Prefetching (Read-Ahead)”](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-read_ahead.html)

`InnoDB` 标准监控输出的 `BUFFER POOL AND MEMORY` 小节中，包含一系列与 `缓冲池` `LRU` 算法有关的字段。
详情参阅
[使用 InnoDB 标准监视器监控缓冲池](#使用-innodb-标准监视器监控缓冲池)

#### 缓冲池的配置
用户可以从一些方面对 `缓冲池` 进行配置，以提高整体性能：
- 理想情况下，用户会把缓冲池设置地尽可能大，为服务器上的其他进程留下足够的内存来运行，而无需过多的分页（？）。
  `缓冲池` 越大， `InnoDB` 越像一个内存数据库，从磁盘读取一次数据后，之后的读取都从内存直接读取。
  详情参阅 [Section 15.8.3.1, “Configuring InnoDB Buffer Pool Size”](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool-resize.html)

- 在拥有足够内存的 64 位系统上，可以将 `缓冲池` 拆分成多个部分，以减少并发情况下对内存结构的竞争。
  详情参阅 [Section 15.8.3.2, “Configuring Multiple Buffer Pool Instances”](https://dev.mysql.com/doc/refman/8.0/en/innodb-multiple-buffer-pools.html)

- 可以通过配置，避免突然读取大量数据的高峰操作把大量不常用数据放到 `缓冲池` 中，从而保持真正常用的数据在内存中。
  详情参阅 [Section 15.8.3.3, “Making the Buffer Pool Scan Resistant”](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-midpoint_insertion.html)

- 可以控制何时以及如何执行 `预读取` 请求，以便将预计不久将需要的页面异步预取到缓冲池中。
  详情参阅 [Section 15.8.3.4, “Configuring InnoDB Buffer Pool Prefetching (Read-Ahead)”](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-read_ahead.html)

- 可以控制后台刷新合适执行，以及是否根据工作负载动态调整刷新频率。
  详情参阅 [Section 15.8.3.5, “Configuring InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-adaptive_flushing.html)

- 可以微调缓冲池刷新机制的各个方面，以提高性能，
  详情参阅 [Section 15.8.3.6, “Fine-tuning InnoDB Buffer Pool Flushing”](https://dev.mysql.com/doc/refman/8.0/en/innodb-lru-background-flushing.html)

- 可以配置 `InnoDB` 如何保存 `缓冲池` ，避免服务重启后的启动预热阶段时间过长。
  详情参阅 [Section 15.8.3.7, “Saving and Restoring the Buffer Pool State”](https://dev.mysql.com/doc/refman/8.0/en/innodb-preload-buffer-pool.html)

#### 使用 InnoDB 标准监视器监控缓冲池
`InnoDB` 标准监视器的输出，提供了关于 `缓冲池` 操作的各项数据指标。
可以通过 `SHOW ENGINE INNODB STATUS` 语句来获取标准监视器输出结果。
`缓冲池` 相关的指标位于 `BUFFER POOL AND MEMORY` 一节，它的输出类似下面的形式：
```SQL
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 2198863872
Dictionary memory allocated 776332
Buffer pool size   131072
Free buffers       124908
Database pages     5720
Old database pages 2071
Modified db pages  910
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 4, not young 0
0.10 youngs/s, 0.00 non-youngs/s
Pages read 197, created 5523, written 5060
0.00 reads/s, 190.89 creates/s, 244.94 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not
0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read
ahead 0.00/s
LRU len: 5720, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```

下面的表格描述 `InnoDB` 标准输出中与 `缓冲池` 相关的指标的含义。

> **注意：**
>
> `InnoDB` 标准监视器输出中提供的每秒平均值，基于自上次打印 `InnoDB` 标准监视器输出以来经过的时间。

|指标名|描述|
|---|---|
|Total memory allocated|整个 `缓冲池` 在内存中分配的所有空间，以字节为单位|
|Dictionary memory allocated|分配给 `InnoDB` 数据字典的总内存，以字节为单位|
|Buffer pool size|分配给 `缓冲池` 所有 `页` 的总大小|
|Free buffers|`缓冲池` 可用列表中的 `页` 的总大小|
|Database pages|`缓冲池` `LRU` 机制的“新列表”中 `页` 的总大小|
|Old database pages|`缓冲池` `LRU` 机制的“旧列表”中 `页` 的总大小|
|Modified db pages|当前 `缓冲池` 中被修改的 `页` 的总数|
|Pending reads|等待读入 `缓冲池` 的缓冲池 `页` 的数目（？）|
|Pending writes LRU|`缓冲池` 中要从 `LRU` 新列表底部写入的旧脏页的数量。|
|Pending writes flush list|在 `检查点期间`（？） 要刷新的 `缓冲池` `页` 的数量。|
|Pending writes single page|`缓冲池` 中挂起的独立 `页` 写的数目（？）|
|Pages made young|`缓冲池` LRU 列表中被标记为年轻的 `页` 的总数 （向“新列表”头部移动过的总数）|
|Pages made not young|`缓冲池` LRU 列表中被标记为“不年轻”的 `页` 的总数 （没有被编辑为“年轻”而向“旧列表”移动过总数）|
|youngs/s|`LRU` 列表中 `页` 被标记年轻的频率，取每秒平均值。表格下方有对此的更多说明|
|non-youngs/s|`LRU` 列表中 `页` 被标记“不年轻”的频率，取每秒平均值。表格下方有对此的更多说明|
|Pages read|`缓冲池` 中被读取的 `页` 总数.|
|Pages created|在 `缓冲池` 中创建的 `页` 的总数|
|Pages written|在 `缓冲池` 中小or是的 `页` 的总数（？）|
|reads/s|平均每秒在 `缓冲池` 中读取的 `页` 的数量|
|creates/s|平均每秒在 `缓冲池` 中创建的 `页` 的数量|
|writes/s|平均每秒在 `缓冲池` 中写入的 `页` 的数量|
|Buffer pool hit rate|直接从 `缓冲池` 命中的 `页` 与从磁盘直接读取的 `页` 的数目比例|
|young-making rate|将某 `页` 记为“年轻”的读取的命中率。表格下方有详情说明|
|not (young-making rate)|没有将某 `页` 记为“年轻”的读取的命中率。表格下方有详情说明|
|Pages read ahead|`预读取` 操作的每秒平均发生频率|
|Pages evicted without access|从未在 `缓冲池` 中被读取的页的每秒平均过期频率|
|Random read ahead|随机 `预读取` 曹锁的每秒发生频率|
|LRU len|`LRU 列表` 中所有 `页` 的总大小|
|unzip_LRU len|`缓冲池` 的 `unzip_LRU 列表` 的总大小|
|I/O sum|最近 50 秒内 `LRU 列表` 被访问的次数|
|I/O cur|`LRU 列表` 被访问的总次数|
|I/O unzip sum|`unzip_LRU 列表` 被访问的总次数|
|I/O unzip cur|`unzip_LRU 列表` 被访问的总次数|

**说明**

- `yongs/s` 指标只针对“旧列表”中的 `页`。这个指标与 `页` 被访问的数量有关，与总数量无关。
  有可能一个 `页` 会被访问好几次，这几次都会被计数。
  若在没有大量表扫描操作时，`yongs/s` 数值比较低，此时可能需要降低延迟时间（？）或提高 `缓冲池` 中“旧列表”的尺寸占比，
  使"旧列表“头部的数据更慢地走向尾部，这些 `页` 则更可能被访问，并移动到”新列表“。
- `non-yongs/s` 指标只针对“旧列表”中的 `页`。这个指标与 `页` 被访问的数量有关，与总数量无关。
  一个 `页` 可能被访问多次，每次访问都会计数。
  若在发生全表扫描时，本指标的数值不高（同时 `yongs/s` 指标很高），增大延迟时间。
- `young-making` 频率描述了整个 `缓冲池` 中所有 `页` 被访问时的情况，不仅仅是“旧列表”中的页。
  `young-making` 频率与其反向指标的频率，其总和加起来一般小于缓冲池总命中率。
  “旧列表”中的 `页` 并未访问是会移动到“新列表”，但新列表中的 `页` 被访问时，
  只有距离链表头部有一定距离的 `页` 才会移动到 `缓冲池` 头部。
- `not (young-making rate)`

### 修改缓冲区 Change Buffer


### 自适应哈希索引 Adaptive Hash Index


### 日志缓冲区 Log Buffer
