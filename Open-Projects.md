* Pluggable WAL
* Data encryption
* Warm block cache after flush and compactions in a smart way
* Queryable Backup
* Improve sub-compaction by making data partition more evenly
* Tools to collect operations to a database and replay them
* Customized bloom filter for data blocks
* Build a new compaction style optimized for time series data
* Benchmark FIFOCompaction
* Implement YCSB benchmark scenarios in db_bench
* Improve DB recovery speed when WAL files are large (parallel replay WAL)
* use thread pools to do readahead + decompress and compress + write-behind. Igor started on this. When manual compaction is multi-threaded then we can use RocksDB as a fast external sorter -- load keys in random order with compaction disabled, then do manual compaction.
* expose merge operator in MongoDB + RocksDB
* SQLite + RocksDB
* Snappy compression for WAL writes. Maybe this is only done for large writes and maybe we add a field to WriteOptions so a user can request it.