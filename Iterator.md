## Consistent View
If `ReadOptions.snapshot` is given, the iterator will return data as of the snapshot. If it is `nullptr`, the iterator will read from an implicit snapshot as of the time the iterator is created. The implicit snapshot is preserved by [[pinning resource|Iterator#resource-pinned-by-iterators-and-iterator-refreshing]]. There is not way to convert this implicit snapshot to explicit snapshot.

## Error Handling
`Iterator::status()` returns the error of the iterating. The errors include I/O errors, checksum mismatch, unsupported operations, internal errors, or other errors.

If there is no error, the status is `Status::OK()`. If the status is not OK, the iterator will be invalidated too. In another word, if `Iterator::Valid()` is true, `status()` is guaranteed to be `OK()` so it's safe to proceed other operations without checking status().

On the other hand, if `Iterator::Valid()` is false, there are two possibilities: (1) We reached the end of the data. In this case, `status()` is `OK()`; (2) there is an error. In this case `status()` is not `OK()`. It is always a good practice to check `status()` if the iterator is invalidated.

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

Coming soon...

