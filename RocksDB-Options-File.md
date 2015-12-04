In RocksDB 4.3, we added a set of features that makes managing RocksDB options easier.

1. Each RocksDB database will now automatically persist its current set of options into a file on every successful call of DB::Open(), SetOptions(), and CreateColumnFamily() / DropColumnFamily().

2. [LoadLatestOptions() / LoadOptionsFromFile()](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/utilities/options_util.h#L20-L58): Construct RocksDB options object from an options file.

3. [CheckOptionsCompatibility](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/utilities/options_util.h#L64-L77): Compatibility check of two sets of RocksDB options.

With the above options file support, developers no longer need to maintain the full set of options of a previously-created RocksDB instance.  In addition, when changing options is needed, CheckOptionsCompatibility() can further make sure the resulting set of Options can successfully open the same RocksDB database without corrupting the underlying data.