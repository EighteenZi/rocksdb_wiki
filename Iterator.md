## Consistent View
If `ReadOptions.snapshot` is given, the iterator will return data as of the snapshot. If it is `nullptr`, the iterator will read from an implicit snapshot as of the time the iterator is created. The implicit snapshot is preserved by [[pinning resource|Iterator#resource-pinned-by-iterators-and-iterator-refreshing]]. There is no way to convert this implicit snapshot to an explicit snapshot.

## Error Handling
`Iterator::status()` returns the error of the iterating. The iterator may pass the blocks or files it had difficulties in reading (because of IO error, data corruption or other issues) and continue with the next available keys. `status()` may not OK after every iterator operation: `Seek()`, `Next()`, `SeekToFirst()`, `SeekToLast()`, `SeekForPrev()`, and `Prev()`, even if `Valid()=true`.

Note that previously, we state following in this wiki, which is incorrect:

> `Iterator::status()` returns the error of the iterating. The errors include I/O errors, checksum mismatch, unsupported operations, internal errors, or other errors.

> If there is no error, the status is `Status::OK()`. If the status is not OK, the iterator will be invalidated too. In another word, if `Iterator::Valid()` is true, `status()` is guaranteed to be `OK()` so it's safe to proceed other operations without checking status().

> On the other hand, if `Iterator::Valid()` is false, there are two possibilities: (1) We reached the end of the data. In this case, `status()` is `OK()`; (2) there is an error. In this case `status()` is not `OK()`. It is always a good practice to check `status()` if the iterator is invalidated.

## Iterating upper bound
You can specify an upper bound of your range query by setting `ReadOptions.iterate_upper_bound` for the read option passed to `NewIterator()`. By setting this option, RocksDB doesn't have to find the next key after the upper bound. In some cases, some I/Os or computation can be avoided. In some specific workloads, the improvement can be significant. Note it applies to both of forward and backward iterating.

See the comment of the option for more information.

## Resource pinned by iterators and iterator refreshing
Iterators by themselves don't use much memory, but it can prevent some resource from being released. This includes:
1. memtables and SST files as of the creation time of the iterators. Even if some memtables and SST files are removed after flush or compaction, they are still preserved if an iterator pinned them.
2. data blocks for the current iterating position. These blocks will be kept in memory, either pinned in block cache, or in the heap if block cache is not set. Please note that although normally blocks are small, in some extreme cases, a single block can be quite large, if the value size is very large.

So the best use of iterator is to keep it short-lived, so that these resource is freed timely.

An iterator has some creation costs. In some use cases (especially memory-only cases), people want to avoid the creation costs of iterators by reusing iterators. When you are doing it, be aware that in case an iterator getting stale, it can block resource from being released. So make sure you destroy or refresh them if they are not used after some time, e.g. one second. When you need to treat this stale iterator, before release 5.7, you'll need to destroy the iterator and recreate it if needed. Since release 5.7, you can call an API `Iterator::Refresh()` to refresh it. By calling this function, the iterator is refreshed to represent the recent states, and the stale resource pinned previously is released. 

## Prefix Iterating
Prefix iterator allows users to use bloom filter or hash index in iterator, in order to improve the performance. However, the feature has limitation and may return wrong results without reporting an error if misused. So we recommend you to use this feature carefully. For how to use the feature, see [[Prefix Seek|Prefix Seek API Changes]]. Options `total_order_seek` and `prefix_same_as_start` are only applicable in prefix iterating.

## Read-ahead
Currently RocksDB doesn't support any general read-ahead of data. If your use case is dominated by iterating and you are relying on OS page cache, you can choose to turn on readahead by setting `DBOptions.advise_random_on_open = false`. It is more helpful if you run on hard drives or remote storage, but may not have much actual effects on directly attached SSD devices.

`ReadOptions.readahead_size` provides read-ahead support in RocksDB for very limited use cases. The limitation of this feature is that, if turned on, the constant cost of the iterator will be much higher. So you should only use it if you iterate a very large range of data, and can't work it around using other approaches. A typical use case will be that the storage is remote storage with very long latency, OS page cache is not available and a large amount of data will be scanned. By enabling this feature, every read of SST files will read-ahead data according to this setting. Note that one iterator can open each file per level, as well as all L0 files at the same time. You need to budget your read-ahead memory for them. And the memory used by the read-ahead buffer can't be tracked automatically.

We are looking for improving read-ahead in RocksDB.

