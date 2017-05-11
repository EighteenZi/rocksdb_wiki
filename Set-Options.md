Besides writing code to operate on RocksDB to achieve your functionality, using [Basic Operations](https://github.com/facebook/rocksdb/wiki/Basic-Operations), you may also be interested in how to tune RocksDB to achieve desired performance. In this page, we introduce how to get an initial set-up, which should work good enough for many use cases.

RocksDB have tons of options, but most users should ignore most of them, because they are for very specific workloads. Try to leave most RocksDB options as defaults, except what is mentioned below.

First, you need to think about options related resource limitation (which are also mentioned in [Basic Options](https://github.com/facebook/rocksdb/wiki/Basic-Operations)):

**cf_options.write_buffer_size**: this is the maximum write buffer size used for this column family. You need to budget the worst case double that amount of memory usage. If you don't have enough memory for this, you need to reduce this value. Otherwise, it is not recommended to change.

**Set block cache size**. Create a block cache with the size you want for caching the uncompressed data. This is recommended to be about 1/3 of your memory budget. The rest of free memory can be left for OS page cache. Leaving large chunk of memory to OS page cache has the benefit of avoiding tight memory budgeting (If you are interested, you can see [Memory Usage in RocksDB](https://github.com/facebook/rocksdb/wiki/Memory-usage-in-RocksDB)). Setting block cache size requires we set table related options. Here is the way to set it:
```
BlockBasedTableOptions table_options;
... \\ set options in table_options
options.table_factory.reset(new BlockBasedTableFactory(table_options));
```
Here is the way to set the block size.
```
std::shared_ptr<Cache> cache = NewLRUCache(<your_cache_size>);
table_options.block_cache = cache;
```
Remember to pass the same cache object to all the table_options for all the column families of all DBs managed by the process. If you want, you can achieve the same effect by passing the same table_factory or table_options to all the column families of all DBs. To know more about block cache, see [Block Cache](https://github.com/facebook/rocksdb/wiki/Block-Cache).


**cf_options.compression, cf_options.bottonmost_compression**.  These are the options you need to pick based on the availability of the compression types in your production host, CPU budget and space constraints. Our recommendation is to use kLZ4Compression for **cf_options.compression** and kZSTD for **cf_options.bottonmost_compression**. Please make sure you check whether the compression you picked is available in your environment. The second choices of the two options are Snappy and Zlib, respectively.


**Maybe enable bloom filter**. This is the only option I suggest you start to tune initially based on your query pattern. If most of your queries are executed using iterators, you shouldn't set bloom filter. If Get() is your common operations, then set bloom filter:
```
table_options.filter_policy.reset(NewBloomFilterPolicy(10, false));
```
To learn more about bloom filters, see [Bloom Filter](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter).

Then set some options to be the values specified below. We've set defaults for most options to achieve reasonable out-of-box performance. We didn't change these options because of the concern of incompatibility or regression when users upgrade their existing RocksDB instance to a newer version. But we suggest users to start their new DB with those setting:
```
cf_options.level_compaction_dynamic_level_bytes = true;
options.max_background_compactions = 4;
options.max_background_flushes = 2;
options.bytes_per_sync = 1048576;
table_options.block_size = 16 * 1024;
table_options.cache_index_and_filter_blocks = true;
table_options.pin_l0_filter_and_index_blocks_in_cache = true;
```
Don't feel sad if you have existing services running default of these options. Although we think these are better than default options, none of them is likely bring significant improvements.

Now you are ready to see the initial RocksDB performance. Hope it is good enough!

If the performance of RocksDB after the basic set-up above is good for you, we don't recommend you to further tune it. It is common that workload changes through time. If you budget for the performance achieved using a highly customized setting for current workload, some modest workload change may push the performance off the cliff. On the other hand, if the performance is not good enough to you, you can further tune RocksDB following more detailed [Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide). 
