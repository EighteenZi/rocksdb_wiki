In this page, we summarize some unknown issues and limitations

* rocksdb::DB instances need to be destroyed before your main function exits. RocksDB instances usually depends on some internal static variables. Users need to make sure rocksdb::DB instances are destroyed before those static variables.

* Universal Compaction Style has a limitation on total data size. See [Universal Compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction).