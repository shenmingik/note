- [写特性](#写特性)
  - [pipeline写入](#pipeline写入)
  - [写延缓](#写延缓)
  - [Merge](#merge)
- [调优](#调优)
  - [BlockCache](#blockcache)
    - [LRU Cache](#lru-cache)
    - [Clock Cache](#clock-cache)
  - [MemTable](#memtable)
  - [WAL](#wal)
- [系统特性](#系统特性)
  - [MANIFEST](#manifest)
  - [全内存数据库](#全内存数据库)
  - [index/fliter](#indexfliter)
  - [Version](#version)

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
## MemTable
Memtable的重要配置如下：
- `AdvancedColumnFamilyOptions::memtable_factory`:memtable工厂对象，用户可以改变memtable的底层实现并提供个性化的实现配置
- `ColumnFamilyOptions::write_buffer_size`:单个内存表的大小限制
- `DBOptions::db_write_buffer_size`:所有列族的内存表的总大小
- `DBOptions::write_buffer_manager`:用户自定义的内存管理
- `AdvancedColumnFamilyOptions::max_write_buffer_number`:内存表的最大个数

而在以下三种情况，MemTable会被Flush：
1. MemTable大小超过write_buffer_size
2. 全部列族的大小超过了db_write_buffer_size
3. 全部的WAL文件超过max_total_wal_size,这种情况下，内存中数据最老的内存表会被选择执行flush操作，然后内存表对应的WAL file会被回收

## WAL
对RocksDB的每一次update都会写入两个位置：1） 内存表（内存数据结构，后续会flush到SST file） 2）磁盘中的write ahead log（WAL）。在故障发生时，WAL可以用来恢复内存表中的数据。默认情况下，RocksDB通过在每次用户写时调用fflush WAL文件来保证一致性。

其配置如下：

- `DBOptions::wal_dir`: RocksDB保存WAL file的目录，可以将目录与数据目录配置在不同的路径下。
- `DBOptions::WAL_ttl_seconds`: 影响归档WAL被删除的快慢，指归档文件多久会被删除。
- `DBOptions::WAL_size_limit_MB`：影响归档WAL被删除的快慢。
- `DBOptions::max_total_wal_size`: 一旦，WALs超过了这个大小，RocksDB会强制将所有列族的数据flush到SST file，之后就可以删除最老的WALs。
- `DBOptions::manual_wal_flush`: 这个参数决定了WAL flush是否在每次写之后自动执行，或者是纯手动执行
- `DBOptions::wal_filter`: 在恢复数据时可以过滤掉WAL中某些记录


```cpp
  rocksdb::Options opt;
  opt.wal_dir = "./test/wal";
  opt.WAL_ttl_seconds = 60;
  opt.WAL_size_limit_MB = 60;
  opt.max_total_wal_size = 100;
  opt.manual_wal_flush = true;
  opt.wal_filter = new rocksdb::WalFilter();
```
# 系统特性

## MANIFEST
> 为了弥补POSIX文件系统的原子性和一致性缺陷，rocksdb引入了MANIFEST机制。

MANIFEST是rocksdb状态变更的transction log记录。整个机制包含`MANIIFEST-Sequence`滚动日志文件以及最新的`CURRENT`文件指针 文件。

当系统启动或者重启的时候，CURRENT指针文件指向的MANIFEST日志文件记录了rocksdb的一致性状态。当这个日志文件超过了阈值的时候，新的日志文件就会被创建，这个新文件是当前DB的快照，CURRENT也会同时更新。

> MANIFEST日志文件本身是一串version edit记录。rocksdb在特定时间的特定状态会关联一个version。针对version的任何改动都被视为一个version edit。其基本格式格式如下：
> 
>+-------------+------ ......... ----------+
> 
>| Record ID   | Variable size record data |
>
>+-------------+------ .......... ---------+
>
> <-- Var32 --->|<-- varies by type       -->

> 详细wiki：<https://github.com/facebook/rocksdb/wiki/MANIFEST>

## 全内存数据库
* 将DB目录设置为mount 到tmpfs或者ramfs的位置
* 设置Options::wal_log为持久化存储上的目录
* 设置Options::WAL_ttl_seconds为T second
* 每隔T/2 second，backup RocksDB ，产生snapshot文件，使用的配置需要设置:BackupableDBOptions::backup_log_files = false
* 当丢失数据时，使用配置 RestoreOptions::keep_log_file = true来从backup中恢复数据。

## index/fliter
>随着DB/mem使用越来越多，filter/index block的内存空间变得不可忽视。虽然cache_index_and_filter_blocks 配置只允许filter/index block数据的一部分cache在block cache中，但是还是会因为数据量的庞大影响RocksDB的性能。

如果要分片的话，SST file的index/filter会被分片为多个小 block，并会配备一个索引。当需要读取index/filter时，只有top-level index会load到内存。然后，通过top-level index找出具体需要查询的那个分片，然后加载那个分片数据到block cache。top-level index占用的内存空间很小，可以存储在heap 也可以存储在block cache中，这取决于cache_index_and_filter_blocks配置。

有好也必有坏，优劣如下：
* 优势：
  * 更高的cache 命中率:分片后，避免了超大的index/filter blocks 占用稀缺的cache空间，以更小的block形式加载需要的分片数据到block cache，这提高的内存空间的有效使用。
  * 节省IO: 当index/filter分片数据 cache miss后，只有一个分片需要从disk load到内存，与load SSTfile的全部index/filter相比，这会大大减轻disk的负载。
  * No compromise on index/filters：如果没有采取分片策略的话，要想减缓index/filter内存空间占用的问题可以采取以下方法：设置更大的block或者减少bloom bits来使index/filter变得更小。前者会导致刚才所述的cache 浪费问题，后者会影响bloom filter功能的正确性。

* 劣势：
  * top-level index会占用额外空间： index/filter大小的0.1~1%
  * 更高的disk IO：如果top-level index不在cache的话，会增加一次额外的IO。为了避免这种问题，可以将index 以更高的优先级存储在heap或者 cache中。（todo）
  * 损失了空间局部性: 如果是这样的场景，频繁且随机地读取相同SST文件的数据，这样就会在每次读取时都会load 不同的分片数据到内存，和一次性读取所有的index/filter相比，显然会更加低效。在RocksDB的benchmark中很少出现这种情况，但是这确实会发生在LSM tree的L0/L1数据访问中。因此，这两层的SST file 的index/filter可以不分片。(to do)

配置如下：
* index_type = IndexType::kTwoLevelIndexSearch
  * 这个配置是启用index分片
* NewBloomFilterPolicy(BITS, false)
  * 使用full filters
partition_filters = true
  * 这个配置是启用filter分片
* metadata_block_size = 4096
index 
  * 分片的大小设置
* cache_index_and_filter_blocks = false
  * 分片数据存储在cache中。控制top-level索引的存储位置，但是这种情况，在benchmark中实验数据不多。
* cache_index_and_filter_blocks = true and pin_top_level_index_and_filter = true
  * 将所有的index/filter数据和top-level index都存储在block cache。
* cache_index_and_filter_blocks_with_high_priority = true
* pin_l0_filter_and_index_blocks_in_cache = true
  1. 建议设置，因为这个配置会应用到index/filter 分片
  2. 只在compation style 是level-based时使用
  3. 需要注意：当把block 数据cache到block cache后，可能会导致超过内存设置的容量（如果strict_capacity_limit 没有设置）。
* block cache size: 如果之前都是将filter/index 存储在heap，现在设置filter/index 数据cache到block cache的话，不要忘了增加block cache size，大小与从heap中减少的量大概一致。

## Version
> 详细wiki：<https://www.jianshu.com/p/b95db752178f>
