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

**Q: Can I write to RocksDB using multiple processes?**

A: No. However, it can be opened in read-only mode from multiple processes.

**Q: Does RocksDB support multi-process read access?**

A: RocksDB support multi-process read only process without writing the database.  This can be done by opening the database with DB::OpenForReadOnly() call.

**Q: Is it safe to close RocksDB while another thread is issuing read, write or manual compaction requests?**

A: No. The users of RocksDB need to make sure all functions have finished before they close RocksDB.

**Q: What's the maximum key and value sizes supported?**

A: In general, RocksDB is not designed for large keys.  The maximum recommended sizes for key and value are 8MB and 3GB respectively.

**Q: Can I run RocksDB and store the data on HDFS?**

A: Yes, by using the Env returned by NewHdfsEnv(), RocksDB will store data on HDFS.  However, the file lock is currently not supported in HDFS Env.

**Q: Can I preserve a “snapshot” of RocksDB and later roll back the DB state to it?**

A: Yes, via the BackupEngine or Checkpoint.

**Q: What's the fastest way to load data into RocksDB?**

A: A fast way to direct insert data to the DB:

1. using single writer thread and insert in sorted order
2. batch hundreds of keys into one write batch
3. use vector memtable
4. make sure options.max_background_flushes is at least 4
5. before inserting the data, disable automatic compaction, set options.level0_file_num_compaction_trigger, options.level0_slowdown_writes_trigger and options.level0_stop_writes_trigger to very large. After inserting all the data, issue a manual compaction.

3-5 will be automatically done if you call Options::PrepareForBulkLoad() to your option

If you can pre-process the data offline before inserting. There is a faster way: you can sort the data, generate SST files with non-overlapping ranges in parallel and bulkload the SST files. See https://github.com/facebook/rocksdb/wiki/Creating-and-Ingesting-SST-files

**Q: Does RocksJava support all the features?**

A: We are working toward to make RocksJava feature compatible.  However, there're missing features in RocksJava.

