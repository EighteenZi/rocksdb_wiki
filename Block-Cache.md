Blocks of block-based table are cached in memory in a sharded LRU cache. By default the highest `kNumShardBits` (which is 4) of hash value is used as shard id, so there will be `2^kNumShardBits` shards. Capacity is split evenly to each of the shards. Each shard maintains its own hash table for lookup and a linked-list of cached blocks for eviction. Operations on a shard needs to lock the whole shard. As a result, shards with hot blocks (such as index blocks and filter blocks) can have lock contention.

### Usage

To use block cache, call `NewLRUCache()` and set the result to `BlockBasedTableOptions.block_cache`. Example:

    BlockBasedTableOptions table_options;
    table_options.block_cache = NewLRUCache(capacity, num_shard_bits);
    Options options;
    options.table_factory.reset(new BlockBasedTableFactory(table_options));

If `BlockBasedTableOptions.block_cache` is null, a default 8MB cache will be used. To disable block cache, set `BlockBasedTableOptions.no_block_cache` to true.

Block cache stores uncompressed block contents. Optionally, you can have a second layer block cache which stores compressed block content. It can be enable via `BlockBasedTableOptions.compressed_block_cache`.

There are two more options to control whether to cache index blocks and filter blocks:

`BlockBasedTableOptions.cache_index_and_filter_blocks`: Cache index blocks and filter blocks in block cache. Defaults to false, where index and filter blocks will be pre-loaded by `BlockBasedTableReader` and stored separately from block cache.

`BlockBasedTableOptions.pin_l0_filter_and_index_blocks_in_cache`: For L0 block based tables, avoid index blocks and filter blocks being swap out of block cache. Instead, `BlockBasedTableReader` holds a reference of these blocks, and further access will not hit block cache. This option can help reduce lock contention on block cache.