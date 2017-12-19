Write buffer manager helps users control the total memory use by memtables across multiple column families and/or DB instances. By doing that, users can achieve:

1. Try to limit to total memtable usage across multiple column families and DBs under a threshold.
2. Cost the memtable memory usage to block cache

The usage of write buffer manager is similar to rate_limiter and sst_file_manager. Users create one write buffer manager object and pass it to all the option of column families or DBs whose memtable size you want to be controlled by this object. See code comment of write_buffer_manager.h for how to use it.

## Limit total memory of memtables
A memory limit is given when creating the write buffer manager object. RocksDB will try to limit the total memory to under this limit.

In version 5.6 or higher, a flush will be triggered on one column family of the DB you are inserting to, if total mutable memtable size exceeds 90% of the limit. If the actual memory is over the limit, more aggressive flush may also be triggered even if total mutable memtable size is below 90%. Before version 5.6, a flush will be triggered if total mutable memtable size exceeds the limit.

In version 5.6 or higher, the memory is counted as total memory allocated in arena, even if some of them may not yet be used by memtable.  In earlier versions, the memory is counted as memory actually used by memtables.

## Cost memory used in memtable to block cache

Since version 5.6, users can set up RocksDB to cost memory used by memtable to block cache. This can happen no matter whether enable memtable memory limit or not.

In most cases, actually used blocks in block cache are just a small percentage than data cached in block cache, so when users enable this feature, the block cache capacity will cover the memory usage for both of block cache and memtable. If users also enable `cache_index_and_filter_blocks`, then the three major uses of memory of RocksDB will be capped by the single cap.

Here is how it is implemented. For every 1MB memory allocated memtable, WriteBufferManager will put dummy 1MB entries to block cache so that the block cache can track the size correctly and evict blocks to make room if needed. In case the memory used by the memtable shrinks, WriteBufferManager will not immediately remove the dummy blocks, but slowly release them when the memory usage drop significantly. This is because memtable memory is naturally up and down and we try to be less intrusive to block cache.

To enable this feature,
* pass the block cache you are using to the WriteBufferManager you are going to use.
* still pass the parameter of WriteBufferManager as the maximum memory you want RocksDB to use for memtables.
* set the capacity of your block cache to be the sum of the memory used for cached data blocks and memtables.