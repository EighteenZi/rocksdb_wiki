# RocksDB Features that are not in LevelDB

## Performance

* Multithread compaction
* Multithread memtable inserts
* Reduced DB mutex holding
* Optimized level-based compaction style and universal compaction style
* Prefix bloom filter
* Memtable bloom filter
* Single bloom filter covering the whole SST file
* Write lock optimization
* Avoid data copying when possible
* Compression dictionary
* Improved Iter::Prev() performance
* Fewer comparator calls during SkipList searches
* Allocate memtable memory using huge page.

## Features

* Column Families
* Transactions and WriteBatchWithIndex
* Backup and Checkpoints
* Merge Operators
* Compaction Filters
* RocksDB Java
* Manual Compactions Run in Parallel with Automatic Compactions
* Persistent Cache
* Bulk loading
* Forward Iterators/ Tailing iterator
* Single delete and range delete
* Pin iterator key/value 

## Alternative Data Structures And Formats

* Plain Table format for memory-only use cases
* vector-based and hash-based memtable format
* Clock-based cache
* Pluggable information log
* Annotate transaction log write with blob (for replication)

## Tunability

* Rate limiting
* Tunable Slowdown and Stop threshold
* Option to keep all files open
* Option to keep all index and bloom filter blocks in block cache
* Multiple WAL recovery modes
* fadvise hints for readahead and to avoid caching in OS page cache
* option to pin L0 and L1 files in memory
* compression : zstd, zlib, dictionary encoding

## Manageability

* Statistics
* Thread-local profiling
* More commands in command-line tools
* User-defined table properties
* Event listeners
* More DB Properties
