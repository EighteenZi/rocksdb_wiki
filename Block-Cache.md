Block cache is where RocksDB caches data in memory for reads. User can set a `Cache` object to a RocksDB instance with desire capacity (size). A `Cache` object can be shared by multiple RocksDB instance in the same process, allowing users to control the overall cache capacity. The block cache stores uncompressed blocks. Optionally user can set a second block cache storing compressed blocks. Reads will fetch data blocks first from uncompressed block cache, then compressed block cache. The compressed block cache can be a replacement of OS page cache, if direct IO is used.

There are two cache implementations in RocksDB, namely `LRUCache` and `ClockCache`. Both types of the cache are sharded to mitigate lock contention. Capacity is divided evenly to each shard and shards don't share capacity. By default each cache will be sharded into at most 64 shards, with each shard has no less than 512k bytes of capacity.

### Usage

Out of box, RocksDB will use LRU-based block cache implementation with 8MB capacity. To set a customized block cache, call `NewLRUCache()` or `NewClockCache()` to create a cache object, and set it to block based table options. Users can also has their own cache implementation by implementing the `Cache` interface.

    std::shared_ptr<Cache> cache = NewLRUCache(capacity);
    BlockBasedTableOptions table_options;
    table_options.block_cache = cache;
    Options options;
    options.table_factory.reset(new BlockBasedTableFactory(table_options));

To set compressed block cache:

    table_options.block_cache_compressed = another_cache;

RocksDB will create the default block cache if `block_cache` is set to `nullptr`. To disable block cache completly:

    table_options.no_block_cache = true;

### LRU Cache

Out of box, RocksDB will use LRU-based block cache implementation with 8MB capacity. Each shard of the cache maintain its own LRU list and its own hash table for lookup. Synchronization is done via a per-shard mutex. Both reading and writing to the cache would require locking mutex of the shard. User can create a LRU cache by calling `NewLRUCache()`. The function provide several useful options to set to the cache:

* `capacity`: Total size of the cache.
* `num_shard_bits`: The number of bits from cache keys to be use as shard id. The cache will be sharded into `2^num_shard_bits` shards.
* `strict_capacity_limit`: In rare case, block cache size can go larger than its capacity. This is when ongoing reads or iterations over DB pin blocks in block cache, and the total size of pinned blocks exceed the cpacity. If there are further reads which try to insert blocks into block cache, if `strict_capacity_limit=false`(default), the cache will fail to respect its capacity limit and allow the insertion. This can create undesired OOM error that crashes the DB if the host don't have enough memory. Setting the option to `true` will reject further insertion to the cache and fail the read or iteration. The option works on per-shard basis, means it is possible one shard is rejecting insert when it is full, while another shard still have extra unpinned space.
* `high_pri_pool_ratio`: The ratio of capacity reserve for high priority blocks. See [[Caching index and filter blocks|Block-Cache#caching-index-and-filter-blocks]] section below.

### Clock Cache

`ClockCache` implements the [CLOCK algorithm](https://en.wikipedia.org/wiki/Page_replacement_algorithm#Clock).

### Caching Index and Filter Blocks

By default index and filter blocks is cached outside of block cache, but optionally they can be managed by block cache. 

### Statistics

A list of block cache counters can be accessed through `Options.statistics` if it is non-null.   
    
    // total block cache misses
    // REQUIRES: BLOCK_CACHE_MISS == BLOCK_CACHE_INDEX_MISS +
    //                               BLOCK_CACHE_FILTER_MISS +
    //                               BLOCK_CACHE_DATA_MISS;
    BLOCK_CACHE_MISS = 0,
    // total block cache hit
    // REQUIRES: BLOCK_CACHE_HIT == BLOCK_CACHE_INDEX_HIT +
    //                              BLOCK_CACHE_FILTER_HIT +
    //                              BLOCK_CACHE_DATA_HIT;
    BLOCK_CACHE_HIT,
    // # of blocks added to block cache.
    BLOCK_CACHE_ADD,
    // # of failures when adding blocks to block cache.
    BLOCK_CACHE_ADD_FAILURES,
    // # of times cache miss when accessing index block from block cache.
    BLOCK_CACHE_INDEX_MISS,
    // # of times cache hit when accessing index block from block cache.
    BLOCK_CACHE_INDEX_HIT,
    // # of times cache miss when accessing filter block from block cache.
    BLOCK_CACHE_FILTER_MISS,
    // # of times cache hit when accessing filter block from block cache.
    BLOCK_CACHE_FILTER_HIT,
    // # of times cache miss when accessing data block from block cache.
    BLOCK_CACHE_DATA_MISS,
    // # of times cache hit when accessing data block from block cache.
    BLOCK_CACHE_DATA_HIT,
    // # of bytes read from cache.
    BLOCK_CACHE_BYTES_READ,
    // # of bytes written into cache.
    BLOCK_CACHE_BYTES_WRITE,

See also: [[Memory-usage-in-RocksDB#block-cache]]