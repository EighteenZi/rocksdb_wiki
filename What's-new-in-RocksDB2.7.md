### Public API changes
* Removed PrefixHashRepFactory. Please use NewHashSkipListRepFactory instead.

### New features
* Put with SliceParts - Variant of Put() that gathers output like writev(2)
* Cache index and filter block in block cache - turned off by default

### Performance improvements
* Huge benchmark performance improvements. For example, increase in readonly QPS from about 300k in 2.6 release to 1.1 million in 2.7 [1]
* Speeding up a way RocksDB deleted obsolete files - no longer listing the whole directory under a lock


[1] You can see exact parameters in rocksdb/build_tools/regression_build_test.sh