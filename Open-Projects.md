* Benchmark FIFOCompaction
* Implement YCSB benchmark scenarios in db_bench
* Improve DB recovery speed when WAL files are large (parallel replay WAL)
* Column-Aware Block Compression for MySQL+RocksDB
* use thread pools to do readahead + decompress and compress + write-behind. Igor started on this. When manual compaction is multi-threaded then we can use RocksDB as a fast external sorter -- load keys in random order with compaction disabled, then do manual compaction.
* expose merge operator in MongoDB + RocksDB
* SQLite + RocksDB
* Optimize RocksDB on 10+ hard drives. In this case RocksDB needs to split the LSM files over many filesystems.
* User-land log-structured read-cache for RocksDB. When a server has a large amount of disk space and smaller amount of flash then the flash device can be used as a persistent read cache managed by RocksDB. A log structured approach can reduce the write-amp on the flash device.
* Snappy compression for WAL writes. Maybe this is only done for large writes and maybe we add a field to WriteOptions so a user can request it.
* Row cache for RocksDB
* Parallel L0->L1 compaction