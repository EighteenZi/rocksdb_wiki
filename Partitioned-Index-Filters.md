As DB/mem ratio gets larger, the memory footprint of filter/index blocks becomes non-trivial. Although `cache_index_and_filter_blocks` allows storing only a subset of them in block cache, their relatively large size negatively affects performance by i) occupying the block cache space that could otherwise be used for caching data, ii) increasing load on the disk storage for loading them into the cache after a miss. Here we illustrate these problems in more detail and explain how partitioning index/filters alleviates the overhead.

## How large are the index/filter blocks?

RocksDB has by default one index/filter block per SST file. The size of index/filter varies based on the configuration but for a SST of size 256MB the index/filter block of 0.5/5MB is typical, which is much larger than the typical data block size of 4-32KB. That is fine when all index/filters fit into memory and hence are read once per SST lifetime, not so much when they compete with data blocks for the block cache space and are also likely to be re-read many times from the disk.

## What is the big deal with large index/filter blocks?

When index/filter blocks are stored in block cache they are effectively competing with data blocks (as well as with each other) on this scarce resource. A filter of size 5MB is occupying the space that could otherwise be used to cache 1000s of data blocks (of size 4KB). This would result in more cache misses for data blocks. The large index/filters also kick each other out of the block cache more often and increase their own cache miss too. This is while only a small part of the index/filter block might have been actually used during its lifetime in the cache.

After a index/filter cache miss, it has to be reloaded from the disk and its large size is not helping in reducing the IO cost. While a simple point lookup might need at most a couple of data block reads (of size 4KB) one from each layer of LSM, it might end up also loading multiple megabytes of index/filter blocks. If that happens often then the disk is spending more time serving index/filters rather than the actual data blocks.

## What is partitioned index/filters?

With partitioning, the index/filter of a SST file is partitioned into smaller blocks with an additional top-level index on them. When reading an index/filter, only top-level index is loaded into memory. The partitioned index/filter then uses the top-level index to lazily load into the block cache the partitions that are required to perform the index/filter query. The top-level index, which has much smaller memory footprint, can be stored in heap or block cache depending on the `cache_index_and_filter_blocks` setting.

### Pros:
- Higher cache hit rate: Instead of polluting the cache space with large index/blocks, partitioning allows loading index/filters with much finer granularity and hence making effective use of the cache space.
- Less IO util: Upon a cache miss for an index/filter partition, only one partition requires to be loaded from the disk, which results into much lighter load on disk compared to reading the entire index/filter of the SST file.
- No compromise on index/filters: Without partitioning the alternative approach to reduce memory footprint of index/filters is to sacrifice their accuracy by for example larger data blocks or lower bloom bits to have smaller index and filters respectively.

### Cons:
- Additional space for the top-level index: its quite small 0.1-1% of index/filter size.
- More disk IO: if the top-level index is not already in cache it would result to one additional IO. To avoid that they can be either stored in heap or stored in cache with hi priority (TODO work)
- Losing spatial locality: if a workload requires frequent, yet random reads from the same SST file it would result into loading a separate index/filter partition upon each read, which is less efficient than reading the index/filter at once. Although we did not observe this pattern in our benchmarks, it is only likely to happen for L0/L1 layers of LSM, for which partitioning can be disabled (TODO work)
