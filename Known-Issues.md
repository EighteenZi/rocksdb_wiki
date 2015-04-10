In this page, we summarize some unknown issues and limitations that RocksDB users need to know:

* rocksdb::DB instances need to be destroyed before your main function exits. RocksDB instances usually depend on some internal static variables. Users need to make sure rocksdb::DB instances are destroyed before those static variables.

* Universal Compaction Style has a limitation on total data size. See [Universal Compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction).

* Some features are not supported in RocksJava, See [RocksJava Basics](https://github.com/facebook/rocksdb/wiki/RocksJava-Basics)

* If you use prefix iterating and iterates out of the prefix range, by running Prev() will not recover from the previous key and the results are undefined. 