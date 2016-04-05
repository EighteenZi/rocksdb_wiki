**Q: If my process crashes, can it corrupt the database?**

A: No, but data in the un-flushed mem-tables might be lost if Write-Ahead-Log (WAL) is disabled.


**Q: If my machine crashes and rebooted, will RocksDB preserve the data?**

A: Data is synced when you issue a sync write (write with WriteOptions.sync=true), call DB::SyncWAL(), or when memtables are flushed.

**Q: Does RocksDB throw exceptions?**

A: No,  RocksDB returns rocksdb::Status to indicate any error. However, RocksDB does not catch exceptions thrown by STL or other dependencies.  For instance, so it's possible that you will see std::bad_malloc when memory allocation fails, or similar exceptions in other situations.  


**Q: How to know the number of keys stored in a RocksDB database?**

A: Use GetIntProperty(cf_handle, “rocksdb.estimate-num-keys") to obtain an estimated number of keys stored in a column family, or use GetAggregatedIntProperty(“rocksdb.estimate-num-keys", &num_keys) to obtain an estimated number of keys stored in the whole RocksDB database.


**Q: Why GetIntProperty can only returns an estimated number of keys in a RocksDB database?**

A: Obtaining an accurate number of keys in any LSM databases like RocksDB is a challenging problem as they have duplicate keys and deletion entries (i.e., tombstones) that will require a full compaction in order to get an accurate number of keys.  In addition, if the RocksDB database contains merge operators, it will also make the estimated number of keys less accurate.

**Q: Is basic operations Put(), Write(), Get() and NewIterator() thread safe?**
A: Yes.

**Q: Can I write to RocksDB using multiple threads?**

A: Yes. However, they must be in the same process if at least one of them does not open rocksdb in read-only mode.

