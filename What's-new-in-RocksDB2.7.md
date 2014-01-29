### Public API changes
* Removed PrefixHashRepFactory memtable factory. Please use NewHashSkipListRepFactory instead.
* Renamed StackableDB::GetRawDB() to StackableDB::GetBaseDB(). The call returns the underlying DB* for a given StackableDB. Before, GetRawDB() returned the raw DB* by stripping away all StackableDB layers.
* Support multi-threaded EnableFileDeletions() and DisableFileDeletions() - [b60c14f](https://github.com/facebook/rocksdb/commit/b60c14f6ee00dc179d400573d4b172d228a8c5a8)
* Added DB::GetOptions() - [3ce3658](https://github.com/facebook/rocksdb/commit/3ce36584111e236e6d7170f0c0dc9adc4b1f949e)
* Added DB::GetDbIdentity() - [1880268](https://github.com/facebook/rocksdb/commit/18802689b8081c4eaa9b477cf8fe1abddc887a3e)

### New features
* Tailing iterator - [wiki page](https://github.com/facebook/rocksdb/wiki/Tailing-Iterator)
* Implemented BackupableDB - [wiki page](https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F)
* Cache index and filter block in block cache - turned off by default
* Put with SliceParts - Variant of Put() that gathers output like writev(2) -- [8a46ecd](https://github.com/facebook/rocksdb/commit/8a46ecd3579a2f6578a260c61ad89b24660bc81f)

### Performance improvements
* Huge benchmark performance improvements by multiple efforts. For example, increase in readonly QPS from about 530k in 2.6 release to 1.1 million in 2.7 [1]
* Speeding up a way RocksDB deleted obsolete files - no longer listing the whole directory under a lock -- decrease in p99
* Use raw pointer instead of shared pointer for statistics: [5b825d](https://github.com/facebook/rocksdb/commit/5b825d6964e26ec3b4bb6faa708ebb1787f1d7bd) -- huge increase in performance -- shared pointers are slow
* Optimized locking for get -- [1fdb3f](https://github.com/facebook/rocksdb/commit/1fdb3f7dc60e96394e3e5b69a46ede5d67fb976c) -- 1.5x QPS increase for some workloads
* Cache speedup - [e8d40c3](https://github.com/facebook/rocksdb/commit/e8d40c31b3cca0c3e1ae9abe9b9003b1288026a9)
* Implemented autovector, which allocates first N elements on stack. Most of vectors in RocksDB are small. Also, we never want to allocate heap objects while holding a mutex. -- [c01676e4](https://github.com/facebook/rocksdb/commit/c01676e46d3be08c3c140361ef1f5884f47d3b3c)
* Lots of efforts to move malloc, memcpy and IO outside of locks

[1] You can see the exact parameters in rocksdb/build_tools/regression_build_test.sh