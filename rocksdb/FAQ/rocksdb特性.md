- [写特性](#写特性)
  - [pipeline写入](#pipeline写入)
  - [写延缓](#写延缓)
  - [Merge](#merge)
- [调优](#调优)
  - [BlockCache](#blockcache)
    - [LRU Cache](#lru-cache)
    - [Clock Cache](#clock-cache)
- [系统特性](#系统特性)
  - [MANIFEST](#manifest)

# 写特性
## pipeline写入
rocksdb 5.5版本之后引入了pipeline写入，经过facebook测试发现大概可以提高30%的写入速度。

以下代码块可以开启这个特性：
```cpp
  rocksdb::Options opt;
  opt.create_missing_column_families = true;
```
>详细Wiki：https://github.com/facebook/rocksdb/wiki/Pipelined-Write

## 写延缓
上面的开关可能会导致写入过快使压缩和刷盘flush的速度跟不上，这会导致两个问题：
* 空间放大
* 写放大

为了解决这个问题，rocksdb引入了写延缓这个特性，这个特性是自动触发的，每当触发写延缓的时候，rocksdb就会将写入速度降低到`delayed_write_rate`，主要的触发场景如下：
* memtable过多：当等待flush的memtable的数量大于等于设置的`max_write_buffer_number`的时候，就会导致写延缓。这个时候会得到如下的日志信息
<font color=grey>

    Stopping writes because we have 5 immutable memtables (waiting for flush), max_write_buffer_number is set to 5

    Stalling writes because we have 4 immutable memtables (waiting for flush), max_write_buffer_number is set to 5
</font>

* 0层的SST文件过多：当0层的文件达到`level0_slowdown_writes_trigger`的时候也会触发。另外，当level-0的SST文件数达到`level0_stop_writes_trigger`，也会写延缓直到level-0和level-1层的SST文件进行Compaction操作减少level-0层的文件。这个过程会触发如下的日志信息：
<font color=grey>

    Stalling writes because we have 4 level-0 files

    Stopping writes because we have 20 level-0 files
</font>

* 等待压缩的字节过多：当等待压缩的字节达到
`soft_pending_compaction_byte`，就会触发写延缓。这个时候会触发如下的日志信息：
<font color=grey>

    Stalling writes because of estimated pending compaction bytes 500000000

    Stopping writes because of estimated pending compaction bytes 1000000000
</font>

>详细wiki：https://github.com/facebook/rocksdb/wiki/Write-Stalls

## Merge
> 大多的服务存在读后写(read-modify-write)这样的需求，在rocksdb原有实现中采用了Get+Put的方式来实现，但是这样无疑在rocksdb的短板随机读上面插上了一刀，所以rocksdb增加了Merge这个操作来优化这一过程。

Merge操作将这个操作写入到SST文件之中，并不立即执行。而是惰性的等待用户调用Get接口或者rocksdb执行Compation操作的时候处理Merge操作，从而将短板随机读改成顺序写。

<font color=grey>
注：这意味者当执行Merge的时候，可能会涉及到多个算子的运作。
</font>

我们假设当前存在于SST之中的操作按照时序是如下情况。处理逻辑就是从Merge3开始遍历，直道遇到了Put/Delete操作，这其中会合并每一步的操作。
```cpp
Merge3(key)->Merge2(key)->Merge1(key)->Put1(Key)->Get1(key)
```
这些时序操作满足如下的结合律：
`Addx * Puty = Putx+y`
，那么当用户调用Get的时候，就会遍历这些操作形成如下的调用链：

```cpp
Get2(key) = Merge3(key)->Merge2(key)->Merge1(key)->Put1(Key)
          = Merge3(key)->Merge2(key)->Put2(key)
          = Merge3(key)->Put4(key)
          = Put7(key)        
```
这个过程就被称之为`FullMerge`，但是单纯的使用FullMerge是不现实的，这个过程在遇到Put/Delete之前调用链之中的所有Merge都是无法被消除的。

对此，rocksdb提出了改良办法`PartialMerge`，这个过程会对Merge操作进行合并，从而使整个调用链满足交换律：`Addx * Addy = Addx+y`，那么之前的过程就会被这样执行：

```cpp
Get2(key) = Merge3(key)->Merge2(key)->Merge1(key)->Put1(Key)
          = Merge5(key)->Put2(key)
          = Put7(key)        
```
> API见rocksdb_API.md
> 
> 详细wiki：<https://github.com/facebook/rocksdb/wiki/Merge-Operator>

# 调优

```cpp
  //此选项关乎列族的写缓冲大
  //最好设置为2倍最坏内存使用大小
  rocksdb::ColumnFamilyOptions cf_opt;
  cf_opt.write_buffer_size = 64 << 20;  //默认64MB  
```

```cpp
  //所有列族的写缓冲大小，默认是禁用的
  rocksdb::Options opt;
  opt.write_buffer_size = 64 << 20;
```
> 详细wiki：<https://github.com/facebook/rocksdb/wiki/Setup-Options-and-Basic-Tuning>

## BlockCache
Block Cache是rocksdb把数据缓存缓存在内存中以提高读性能的方法。Cache对象可以在同一进程中供多个DB实例使用。

在rocksdb中有两种Cache的实现方式：
* LRU Cache
* Clock Cache

以上两种cache都会被分片来降低锁的压力。
### LRU Cache
LRU Cache是rocksdb的默认Cache，默认大小8M。除了容量之外，还有以下几个参数可以在Cache构造的时候进行配置：
* num_shard_bits: cache的分片数，cache将被分片为2^num_shard_bits
* strict_capacity_limit: 是否严格限制写入，如果为否，cache服务将不会遵循容量限制，直到主机没有足够的内存产生OOM错误。
* high_pri_pool_ratio: 为高优先级block预留的capacity比例

```cpp
  rocksdb::Options opt;
  //设置16M的缓存空间大小
  int capacity = 16 << 20;
  int num_shard_bits = 5;
  bool strict_capacity_limit = false;
  double high_pri_pool_ratio = 0.5;

  shared_ptr<rocksdb::Cache> cache = rocksdb::NewLRUCache(
      capacity, num_shard_bits, strict_capacity_limit, high_pri_pool_ratio);
  rocksdb::BlockBasedTableOptions table_options;
  table_options.block_cache = cache;

  opt.table_factory.reset(rocksdb::NewBlockBasedTableFactory(table_options));
```
### Clock Cache
ClockCache实现了CLOCK算法。CLOCK CACHE的每个shard都有一个cache entry的圆环list。算法会遍历圆环的所有entry寻找unspined entry来回收，但是如果上次scan操作这个entry被使用的话，也会有继续留在cache中的机会。寻找并回收entry使用tbb::concurrent_hash_map。
  
使用LRUCache的一个好处是有一把细粒度的锁。在LRUCache中，即使是查找操作也需要获取分片锁，因为有可能会更改LRU-list。在CLock cache中查找并不需要获取分片锁，只需要查找当前hash_map就可以了，只有在insert时需要获取分片锁。使用clock cache，相比于LRU cache，写吞吐有一定提升。

其也有一些需要注意的配置信息：
* capacity
* num_shard_bits
* strict_capacity_limit

```cpp
  shared_ptr<rocksdb::Cache> cache =
      rocksdb::NewClockCache(capacity, num_shard_bits, strict_capacity_limit);
```

# 系统特性

## MANIFEST
> 为了弥补POSIX文件系统的原子性和一致性缺陷，rocksdb引入了MANIFEST机制。

MANIFEST是rocksdb状态变更的transction log记录。整个机制包含`MANIIFEST-Sequence`滚动日志文件以及最新的`CURRENT`文件指针 文件。

当系统启动或者重启的时候，CURRENT指针文件指向的MANIFEST日志文件记录了rocksdb的一致性状态。当这个日志文件超过了阈值的时候，新的日志文件就会被创建，这个新文件是当前DB的快照，CURRENT也会同时更新。