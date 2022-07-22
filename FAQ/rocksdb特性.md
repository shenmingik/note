

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