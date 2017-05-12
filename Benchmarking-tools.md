# db_bench
`db_bench` is the main tool that is used to benchmark RocksDB's performance. RocksDB inherited db_bench from LevelDB, and enhanced it to support many additional options. db_bench supports many benchmarks to generate different types of workloads, and its various options can be used to control the tests. 

If you are just getting started with db_bench, here are a few things you can try:
1. Start with a simple benchmark like fillseq (or fillrandom) to create a database and fill it with some data
```bash
./db_bench --benchmarks="fillseq"
```
If you want more stats, add the meta operator "stats" and --statistics flag. 
```bash
./db_bench --benchmarks="fillseq,stats" --statistics
```
2. Read the data back
```bash
./db_bench --benchmarks="readrandom" --use_existing_db
```

You can also combine multiple benchmarks to the string that is passed to `--benchmarks` so that they run sequentially. Example:
```bash
./db_bench --benchmarks="fillseq,readrandom,readseq"
```
More in-depth example of db_bench usage can be found [here](https://github.com/facebook/rocksdb/wiki/performance-benchmarks) and [here](https://github.com/facebook/rocksdb/wiki/RocksDB-In-Memory-Workload-Performance-Benchmarks).

Benchmarks List:
```
      	fillseq       -- write N values in sequential key order in async mode
      	fillseqdeterministic       -- write N values in the specified key order and keep the shape of the LSM tree
      	fillrandom    -- write N values in random key order in async mode
      	filluniquerandomdeterministic       -- write N values in a random key order and keep the shape of the LSM tree
      	overwrite     -- overwrite N values in random key order in async mode
      	fillsync      -- write N/100 values in random key order in sync mode
      	fill100K      -- write N/1000 100K values in random order in async mode
      	deleteseq     -- delete N keys in sequential order
      	deleterandom  -- delete N keys in random order
      	readseq       -- read N times sequentially
      	readtocache   -- 1 thread reading database sequentially
      	readreverse   -- read N times in reverse order
      	readrandom    -- read N times in random order
      	readmissing   -- read N missing keys in random order
      	readwhilewriting      -- 1 writer, N threads doing random reads
      	readwhilemerging      -- 1 merger, N threads doing random reads
      	readrandomwriterandom -- N threads doing random-read, random-write
      	prefixscanrandom      -- prefix scan N times in random order
      	updaterandom  -- N threads doing read-modify-write for random keys
      	appendrandom  -- N threads doing read-modify-write with growing values
      	mergerandom   -- same as updaterandom/appendrandom using merge operator. Must be used with merge_operator
      	readrandommergerandom -- perform N random read-or-merge operations. Must be used with merge_operator
      	newiterator   -- repeated iterator creation
      	seekrandom    -- N random seeks, call Next seek_nexts times per seek
      	seekrandomwhilewriting -- seekrandom and 1 thread doing overwrite
      	seekrandomwhilemerging -- seekrandom and 1 thread doing merge
      	crc32c        -- repeated crc32c of 4K of data
      	xxhash        -- repeated xxHash of 4K of data
      	acquireload   -- load N*1000 times
      	fillseekseq   -- write N values in sequential key, then read them by seeking to each key
      	randomtransaction     -- execute N random transactions and verify correctness
      	randomreplacekeys     -- randomly replaces N keys by deleting the old version and putting the new version
        timeseries            -- 1 writer generates time series data and multiple readers doing random reads on id
```

For a list of all options:
```
$ ./db_bench -help
db_bench:
USAGE:
./db_bench [OPTIONS]...

  Flags from tools/db_bench_tool.cc:
    -advise_random_on_open (Advise random access on table file open) type: bool
      default: true
    -allow_concurrent_memtable_write (Allow multi-writers to update mem tables
      in parallel.) type: bool default: false
    -base_background_compactions (The base number of concurrent background
      compactions to occur in parallel.) type: int32 default: 1
    -batch_size (Batch size) type: int64 default: 1
    -benchmark_read_rate_limit (If non-zero, db_bench will rate-limit the reads
      from RocksDB. This is the global rate in ops/second.) type: uint64
      default: 0
    -benchmark_write_rate_limit (If non-zero, db_bench will rate-limit the
      writes going into RocksDB. This is the global rate in bytes/second.)
      type: uint64 default: 0
    -benchmarks (Comma-separated list of operations to run in the specified
      order. Available benchmarks:
      	fillseq       -- write N values in sequential key order in async mode
      	fillseqdeterministic       -- write N values in the specified key order
      and keep the shape of the LSM tree
      	fillrandom    -- write N values in random key order in async mode
      	filluniquerandomdeterministic       -- write N values in a random key
      order and keep the shape of the LSM tree
      	overwrite     -- overwrite N values in random key order in async mode
      	fillsync      -- write N/100 values in random key order in sync mode
      	fill100K      -- write N/1000 100K values in random order in async mode
      	deleteseq     -- delete N keys in sequential order
      	deleterandom  -- delete N keys in random order
      	readseq       -- read N times sequentially
      	readtocache   -- 1 thread reading database sequentially
      	readreverse   -- read N times in reverse order
      	readrandom    -- read N times in random order
      	readmissing   -- read N missing keys in random order
      	readwhilewriting      -- 1 writer, N threads doing random reads
      	readwhilemerging      -- 1 merger, N threads doing random reads
      	readrandomwriterandom -- N threads doing random-read, random-write
      	prefixscanrandom      -- prefix scan N times in random order
      	updaterandom  -- N threads doing read-modify-write for random keys
      	appendrandom  -- N threads doing read-modify-write with growing values
      	mergerandom   -- same as updaterandom/appendrandom using merge operator.
      Must be used with merge_operator
      	readrandommergerandom -- perform N random read-or-merge operations. Must
      be used with merge_operator
      	newiterator   -- repeated iterator creation
      	seekrandom    -- N random seeks, call Next seek_nexts times per seek
      	seekrandomwhilewriting -- seekrandom and 1 thread doing overwrite
      	seekrandomwhilemerging -- seekrandom and 1 thread doing merge
      	crc32c        -- repeated crc32c of 4K of data
      	xxhash        -- repeated xxHash of 4K of data
      	acquireload   -- load N*1000 times
      	fillseekseq   -- write N values in sequential key, then read them by
      seeking to each key
      	randomtransaction     -- execute N random transactions and verify
      correctness
      	randomreplacekeys     -- randomly replaces N keys by deleting the old
      version and putting the new version

      	timeseries            -- 1 writer generates time series data and
      multiple readers doing random reads on id

      Meta operations:
      	compact     -- Compact the entire DB
      	stats       -- Print DB stats
      	resetstats  -- Reset DB stats
      	levelstats  -- Print the number of files and bytes per level
      	sstables    -- Print sstable info
      	heapprofile -- Dump a heap profile (if supported by this port)
      ) type: string
      default: "fillseq,fillseqdeterministic,fillsync,fillrandom,filluniquerandomdeterministic,overwrite,readrandom,newiterator,newiteratorwhilewriting,seekrandom,seekrandomwhilewriting,seekrandomwhilemerging,readseq,readreverse,compact,readrandom,multireadrandom,readseq,readtocache,readreverse,readwhilewriting,readwhilemerging,readrandomwriterandom,updaterandom,randomwithverify,fill100K,crc32c,xxhash,compress,uncompress,acquireload,fillseekseq,randomtransaction,randomreplacekeys,timeseries"
    -block_restart_interval (Number of keys between restart points for delta
      encoding of keys in data block.) type: int32 default: 16
    -block_size (Number of bytes in a block.) type: int32 default: 4096
    -bloom_bits (Bloom filter bits per key. Negative means use default
      settings.) type: int32 default: -1
    -bloom_locality (Control bloom filter probes locality) type: int32
      default: 0
    -bytes_per_sync (Allows OS to incrementally sync SST files to disk while
      they are being written, in the background. Issue one request for every
      bytes_per_sync written. 0 turns it off.) type: uint64 default: 0
    -cache_high_pri_pool_ratio (Ratio of block cache reserve for high pri
      blocks. If > 0.0, we also enable
      cache_index_and_filter_blocks_with_high_priority.) type: double
      default: 0
    -cache_index_and_filter_blocks (Cache index/filter blocks in block cache.)
      type: bool default: false
    -cache_numshardbits (Number of shards for the block cache is 2 **
      cache_numshardbits. Negative means use default settings. This is applied
      only if FLAGS_cache_size is non-negative.) type: int32 default: 6
    -cache_size (Number of bytes to use as a cache of uncompressed data)
      type: int64 default: 8388608
    -compaction_fadvice (Access pattern advice when a file is compacted)
      type: string default: "NORMAL"
    -compaction_pri (priority of files to compaction: by size or by data age)
      type: int32 default: 0
    -compaction_readahead_size (Compaction readahead size) type: int32
      default: 0
    -compaction_style (style of compaction: level-based, universal and fifo)
      type: int32 default: 0
    -compressed_cache_size (Number of bytes to use as a cache of compressed
      data.) type: int64 default: -1
    -compression_level (Compression level. For zlib this should be -1 for the
      default level, or between 0 and 9.) type: int32 default: -1
    -compression_max_dict_bytes (Maximum size of dictionary used to prime the
      compression library.) type: int32 default: 0
    -compression_ratio (Arrange to generate values that shrink to this fraction
      of their original size after compression) type: double default: 0.5
    -compression_type (Algorithm to use to compress the database) type: string
      default: "snappy"
    -cuckoo_hash_ratio (Hash ratio for Cuckoo SST table.) type: double
      default: 0.90000000000000002
    -db (Use the db with the following name.) type: string default: ""
    -db_write_buffer_size (Number of bytes to buffer in all memtables before
      compacting) type: int64 default: 0
    -delayed_write_rate (Limited bytes allowed to DB when soft_rate_limit or
      level0_slowdown_writes_trigger triggers) type: uint64 default: 8388608
    -delete_obsolete_files_period_micros (Ignored. Left here for backward
      compatibility) type: uint64 default: 0
    -deletepercent (Percentage of deletes out of reads/writes/deletes (used in
      RandomWithVerify only). RandomWithVerify calculates writepercent as (100
      - FLAGS_readwritepercent - deletepercent), so deletepercent must be
      smaller than (100 - FLAGS_readwritepercent)) type: int32 default: 2
    -deletes (Number of delete operations to do.  If negative, do FLAGS_num
      deletions.) type: int64 default: -1
    -disable_auto_compactions (Do not auto trigger compactions) type: bool
      default: false
    -disable_seek_compaction (Not used, left here for backwards compatibility)
      type: int32 default: 0
    -disable_wal (If true, do not write WAL for write.) type: bool
      default: false
    -dump_malloc_stats (Dump malloc stats in LOG ) type: bool default: true
    -duration (Time in seconds for the random-ops tests to run. When 0 then num
      & reads determine the test duration) type: int32 default: 0
    -enable_io_prio (Lower the background flush/compaction threads' IO
      priority) type: bool default: false
    -enable_numa (Make operations aware of NUMA architecture and bind memory
      and cpus corresponding to nodes together. In NUMA, memory in same node as
      CPUs are closer when compared to memory in other nodes. Reads can be
      faster when the process is bound to CPU and memory of same node. Use
      "$numactl --hardware" command to see NUMA memory architecture.)
      type: bool default: false
    -enable_write_thread_adaptive_yield (Use a yielding spin loop for brief
      writer thread waits.) type: bool default: false
    -env_uri (URI for registry Env lookup. Mutually exclusive with --hdfs.)
      type: string default: ""
    -expand_range_tombstones (Expand range tombstone into sequential regular
      tombstones.) type: bool default: false
    -expire_style (Style to remove expired time entries. Can be one of the
      options below: none (do not expired data), compaction_filter (use a
      compaction filter to remove expired data), delete (seek IDs and remove
      expired data) (used in TimeSeries only).) type: string default: "none"
    -fifo_compaction_allow_compaction (Allow compaction in FIFO compaction.)
      type: bool default: true
    -fifo_compaction_max_table_files_size_mb (The limit of total table file
      sizes to trigger FIFO compaction) type: uint64 default: 0
    -file_opening_threads (If open_files is set to -1, this option set the
      number of threads that will be used to open files during DB::Open())
      type: int32 default: 16
    -finish_after_writes (Write thread terminates after all writes are
      finished) type: bool default: false
    -hard_pending_compaction_bytes_limit (Stop writes if pending compaction
      bytes exceed this number) type: uint64 default: 137438953472
    -hard_rate_limit (DEPRECATED) type: double default: 0
    -hash_bucket_count (hash bucket count) type: int64 default: 1048576
    -hdfs (Name of hdfs environment. Mutually exclusive with --env_uri.)
      type: string default: ""
    -histogram (Print histogram of operation timings) type: bool default: false
    -identity_as_first_hash (the first hash function of cuckoo table becomes an
      identity function. This is only valid when key is 8 bytes) type: bool
      default: false
    -index_block_restart_interval (Number of keys between restart points for
      delta encoding of keys in index block.) type: int32 default: 1
    -key_id_range (Range of possible value of key id (used in TimeSeries
      only).) type: int32 default: 100000
    -key_size (size of each key) type: int32 default: 16
    -keys_per_prefix (control average number of keys generated per prefix, 0
      means no special handling of the prefix, i.e. use the prefix comes with
      the generated random number.) type: int64 default: 0
    -level0_file_num_compaction_trigger (Number of files in level-0 when
      compactions start) type: int32 default: 4
    -level0_slowdown_writes_trigger (Number of files in level-0 that will slow
      down writes.) type: int32 default: 20
    -level0_stop_writes_trigger (Number of files in level-0 that will trigger
      put stop.) type: int32 default: 36
    -level_compaction_dynamic_level_bytes (Whether level size base is dynamic)
      type: bool default: false
    -max_background_compactions (The maximum number of concurrent background
      compactions that can occur in parallel.) type: int32 default: 1
    -max_background_flushes (The maximum number of concurrent background
      flushes that can occur in parallel.) type: int32 default: 1
    -max_bytes_for_level_base (Max bytes for level-1) type: uint64
      default: 268435456
    -max_bytes_for_level_multiplier (A multiplier to compute max bytes for
      level-N (N >= 2)) type: double default: 10
    -max_bytes_for_level_multiplier_additional (A vector that specifies
      additional fanout per level) type: string default: ""
    -max_compaction_bytes (Max bytes allowed in one compaction) type: uint64
      default: 0
    -max_num_range_tombstones (Maximum number of range tombstones to insert.)
      type: int64 default: 0
    -max_successive_merges (Maximum number of successive merge operations on a
      key in the memtable) type: int32 default: 0
    -max_total_wal_size (Set total max WAL size) type: uint64 default: 0
    -max_write_buffer_number (The number of in-memory memtables. Each memtable
      is of sizewrite_buffer_size.) type: int32 default: 2
    -max_write_buffer_number_to_maintain (The total maximum number of write
      buffers to maintain in memory including copies of buffers that have
      already been flushed. Unlike max_write_buffer_number, this parameter does
      not affect flushing. This controls the minimum amount of write history
      that will be available in memory for conflict checking when Transactions
      are used. If this value is too low, some transactions may fail at commit
      time due to not being able to determine whether there were any write
      conflicts. Setting this value to 0 will cause write buffers to be freed
      immediately after they are flushed.  If this value is set to -1,
      'max_write_buffer_number' will be used.) type: int32 default: 0
    -memtable_bloom_size_ratio (Ratio of memtable size used for bloom filter. 0
      means no bloom filter.) type: double default: 0
    -memtable_insert_with_hint_prefix_size (If non-zero, enable memtable insert
      with hint with the given prefix size.) type: int32 default: 0
    -memtable_use_huge_page (Try to use huge page in memtables.) type: bool
      default: false
    -memtablerep () type: string default: "skip_list"
    -merge_keys (Number of distinct keys to use for MergeRandom and
      ReadRandomMergeRandom. If negative, there will be FLAGS_num keys.)
      type: int64 default: -1
    -merge_operator (The merge operator to use with the database.If a new merge
      operator is specified, be sure to use fresh database The possible merge
      operators are defined in utilities/merge_operators.h) type: string
      default: ""
    -mergereadpercent (Ratio of merges to merges&reads (expressed as
      percentage) for the ReadRandomMergeRandom workload. The default value 70
      means 70% out of all read and merge operations are merges. In other
      words, 7 merges for every 3 gets.) type: int32 default: 70
    -min_level_to_compress (If non-negative, compression starts from this
      level. Levels with number < min_level_to_compress are not compressed.
      Otherwise, apply compression_type to all levels.) type: int32 default: -1
    -min_write_buffer_number_to_merge (The minimum number of write buffers that
      will be merged togetherbefore writing to storage. This is cheap because
      it is anin-memory merge. If this feature is not enabled, then all
      thesewrite buffers are flushed to L0 as separate files and this increases
      read amplification because a get request has to check in all of these
      files. Also, an in-memory merge may result in writing less data to
      storage if there are duplicate records  in each of these individual write
      buffers.) type: int32 default: 1
    -mmap_read (Allow reads to occur via mmap-ing files) type: bool
      default: false
    -mmap_write (Allow writes to occur via mmap-ing files) type: bool
      default: false
    -new_table_reader_for_compaction_inputs (If true, uses a separate file
      handle for compaction inputs) type: int32 default: 1
    -num (Number of key/values to place in database) type: int64
      default: 1000000
    -num_column_families (Number of Column Families to use.) type: int32
      default: 1
    -num_deletion_threads (Number of threads to do deletion (used in TimeSeries
      and delete expire_style only).) type: int32 default: 1
    -num_hot_column_families (Number of Hot Column Families. If more than 0,
      only write to this number of column families. After finishing all the
      writes to them, create new set of column families and insert to them.
      Only used when num_column_families > 1.) type: int32 default: 0
    -num_levels (The total number of levels) type: int32 default: 7
    -num_multi_db (Number of DBs used in the benchmark. 0 means single DB.)
      type: int32 default: 0
    -numdistinct (Number of distinct keys to use. Used in RandomWithVerify to
      read/write on fewer keys so that gets are more likely to find the key and
      puts are more likely to update the same key) type: int64 default: 1000
    -open_files (Maximum number of files to keep open at the same time (use
      default if == 0)) type: int32 default: -1
    -optimistic_transaction_db (Open a OptimisticTransactionDB instance.
      Required for randomtransaction benchmark.) type: bool default: false
    -optimize_filters_for_hits (Optimizes bloom filters for workloads for most
      lookups return a value. For now this doesn't create bloom filters for the
      max level of the LSM to reduce metadata that should fit in RAM. )
      type: bool default: false
    -options_file (The path to a RocksDB options file.  If specified, then
      db_bench will run with the RocksDB options in the default column family
      of the specified options file. Note that with this setting, db_bench will
      ONLY accept the following RocksDB options related command-line arguments,
      all other arguments that are related to RocksDB options will be ignored:
      	--use_existing_db
      	--statistics
      	--row_cache_size
      	--row_cache_numshardbits
      	--enable_io_prio
      	--dump_malloc_stats
      	--num_multi_db
      ) type: string default: ""
    -perf_level (Level of perf collection) type: int32 default: 1
    -pin_l0_filter_and_index_blocks_in_cache (Pin index/filter blocks of L0
      files in block cache.) type: bool default: false
    -pin_slice (use pinnable slice for point lookup) type: bool default: true
    -prefix_size (control the prefix size for HashSkipList and plain table)
      type: int32 default: 0
    -random_access_max_buffer_size (Maximum windows randomaccess buffer size)
      type: int32 default: 1048576
    -range_tombstone_width (Number of keys in tombstone's range) type: int64
      default: 100
    -rate_limit_delay_max_milliseconds (When hard_rate_limit is set then this
      is the max time a put will be stalled.) type: int32 default: 1000
    -rate_limiter_bytes_per_sec (Set options.rate_limiter value.) type: uint64
      default: 0
    -read_amp_bytes_per_bit (Number of bytes per bit to be used in block
      read-amp bitmap) type: int32 default: 0
    -read_cache_direct_read (Whether to use Direct IO for reading from read
      cache) type: bool default: true
    -read_cache_direct_write (Whether to use Direct IO for writing to the read
      cache) type: bool default: true
    -read_cache_path (If not empty string, a read cache will be used in this
      path) type: string default: ""
    -read_cache_size (Maximum size of the read cache) type: int64
      default: 4294967296
    -read_random_exp_range (Read random's key will be generated using
      distribution of num * exp(-r) where r is uniform number from 0 to this
      value. The larger the number is, the more skewed the reads are. Only used
      in readrandom and multireadrandom benchmarks.) type: double default: 0
    -readonly (Run read only benchmarks.) type: bool default: false
    -reads (Number of read operations to do.  If negative, do FLAGS_num reads.)
      type: int64 default: -1
    -readwritepercent (Ratio of reads to reads/writes (expressed as percentage)
      for the ReadRandomWriteRandom workload. The default value 90 means 90%
      operations out of all reads and writes operations are reads. In other
      words, 9 gets for every 1 put.) type: int32 default: 90
    -report_bg_io_stats (Measure times spents on I/Os while in compactions. )
      type: bool default: false
    -report_file (Filename where some simple stats are reported to (if
      --report_interval_seconds is bigger than 0)) type: string
      default: "report.csv"
    -report_file_operations (if report number of file operations) type: bool
      default: false
    -report_interval_seconds (If greater than zero, it will write simple stats
      in CVS format to --report_file every N seconds) type: int64 default: 0
    -reverse_iterator (When true use Prev rather than Next for iterators that
      do Seek and then Next) type: bool default: false
    -row_cache_size (Number of bytes to use as a cache of individual rows (0 =
      disabled).) type: int64 default: 0
    -seed (Seed base for random number generators. When 0 it is deterministic.)
      type: int64 default: 0
    -seek_nexts (How many times to call Next() after Seek() in fillseekseq,
      seekrandom, seekrandomwhilewriting and seekrandomwhilemerging)
      type: int32 default: 0
    -show_table_properties (If true, then per-level table properties will be
      printed on every stats-interval when stats_interval is set and
      stats_per_interval is on.) type: bool default: false
    -simcache_size (Number of bytes to use as a simcache of uncompressed data.
      Nagative value disables simcache.) type: int64 default: -1
    -skip_list_lookahead (Used with skip_list memtablerep; try linear search
      first for this many steps from the previous position) type: int32
      default: 0
    -soft_pending_compaction_bytes_limit (Slowdown writes if pending compaction
      bytes exceed this number) type: uint64 default: 68719476736
    -soft_rate_limit (DEPRECATED) type: double default: 0
    -statistics (Database statistics) type: bool default: false
    -statistics_string (Serialized statistics string) type: string default: ""
    -stats_interval (Stats are reported every N operations when this is greater
      than zero. When 0 the interval grows over time.) type: int64 default: 0
    -stats_interval_seconds (Report stats every N seconds. This overrides
      stats_interval when both are > 0.) type: int64 default: 0
    -stats_per_interval (Reports additional stats per interval when this is
      greater than 0.) type: int32 default: 0
    -stddev (Standard deviation of normal distribution used for picking keys
      (used in RandomReplaceKeys only).) type: double default: 2000
    -subcompactions (Maximum number of subcompactions to divide L0-L1
      compactions into.) type: uint64 default: 1
    -sync (Sync all writes to disk) type: bool default: false
    -table_cache_numshardbits () type: int32 default: 4
    -target_file_size_base (Target file size at level-1) type: int64
      default: 67108864
    -target_file_size_multiplier (A multiplier to compute target level-N file
      size (N >= 2)) type: int32 default: 1
    -thread_status_per_interval (Takes and report a snapshot of the current
      status of each thread when this is greater than 0.) type: int32
      default: 0
    -threads (Number of concurrent threads to run.) type: int32 default: 1
    -time_range (Range of timestamp that store in the database (used in
      TimeSeries only).) type: uint64 default: 100000
    -transaction_db (Open a TransactionDB instance. Required for
      randomtransaction benchmark.) type: bool default: false
    -transaction_lock_timeout (If using a transaction_db, specifies the lock
      wait timeout in milliseconds before failing a transaction waiting on a
      lock) type: uint64 default: 100
    -transaction_set_snapshot (Setting to true will have each transaction call
      SetSnapshot() upon creation.) type: bool default: false
    -transaction_sets (Number of keys each transaction will modify (use in
      RandomTransaction only).  Max: 9999) type: uint64 default: 2
    -transaction_sleep (Max microseconds to sleep in between reading and
      writing a value (used in RandomTransaction only). ) type: int32
      default: 0
    -truth_db (Truth key/values used when using verify) type: string
      default: "/dev/shm/truth_db/dbbench"
    -universal_allow_trivial_move (Allow trivial move in universal compaction.)
      type: bool default: false
    -universal_compression_size_percent (The percentage of the database to
      compress for universal compaction. -1 means compress everything.)
      type: int32 default: -1
    -universal_max_merge_width (The max number of files to compact in universal
      style compaction) type: int32 default: 0
    -universal_max_size_amplification_percent (The max size amplification for
      universal style compaction) type: int32 default: 0
    -universal_min_merge_width (The minimum number of files in a single
      compaction run (for universal compaction only).) type: int32 default: 0
    -universal_size_ratio (Percentage flexibility while comparing file size
      (for universal compaction only).) type: int32 default: 0
    -use_adaptive_mutex (Use adaptive mutex) type: bool default: false
    -use_blob_db (Open a BlobDB instance. Required for largevalue benchmark.)
      type: bool default: false
    -use_block_based_filter (if use kBlockBasedFilter instead of kFullFilter
      for filter block. This is valid if only we use BlockTable) type: bool
      default: false
    -use_clock_cache (Replace default LRU block cache with clock cache.)
      type: bool default: false
    -use_cuckoo_table (if use cuckoo table format) type: bool default: false
    -use_direct_io_for_flush_and_compaction (Use O_DIRECT for background flush
      and compaction I/O) type: bool default: false
    -use_direct_reads (Use O_DIRECT for reading data) type: bool default: false
    -use_existing_db (If true, do not destroy the existing database.  If you
      set this flag and also specify a benchmark that wants a fresh database,
      that benchmark will fail.) type: bool default: false
    -use_fsync (If true, issue fsync instead of fdatasync) type: bool
      default: false
    -use_hash_search (if use kHashSearch instead of kBinarySearch. This is
      valid if only we use BlockTable) type: bool default: false
    -use_plain_table (if use plain table instead of block-based table format)
      type: bool default: false
    -use_single_deletes (Use single deletes (used in RandomReplaceKeys only).)
      type: bool default: true
    -use_stderr_info_logger (Write info logs to stderr instead of to LOG file.
      ) type: bool default: false
    -use_tailing_iterator (Use tailing iterator to access a series of keys
      instead of get) type: bool default: false
    -use_uint64_comparator (use Uint64 user comparator) type: bool
      default: false
    -value_size (Size of each value) type: int32 default: 100
    -verify_checksum (Verify checksum for every block read from storage)
      type: bool default: false
    -wal_bytes_per_sync (Allows OS to incrementally sync WAL files to disk
      while they are being written, in the background. Issue one request for
      every wal_bytes_per_sync written. 0 turns it off.) type: uint64
      default: 0
    -wal_dir (If not empty, use the given dir for WAL) type: string default: ""
    -wal_size_limit_MB (Set the size limit for the WAL Files in MB.)
      type: uint64 default: 0
    -wal_ttl_seconds (Set the TTL for the WAL Files in seconds.) type: uint64
      default: 0
    -writable_file_max_buffer_size (Maximum write buffer for Writable File)
      type: int32 default: 1048576
    -write_buffer_size (Number of bytes to buffer in memtable before
      compacting) type: int64 default: 67108864
    -write_thread_max_yield_usec (Maximum microseconds for
      enable_write_thread_adaptive_yield operation.) type: uint64 default: 100
    -write_thread_slow_yield_usec (The threshold at which a slow yield is
      considered a signal that other processes or threads want the core.)
      type: uint64 default: 3
    -writes (Number of write operations to do. If negative, do --num reads.)
      type: int64 default: -1
    -writes_per_range_tombstone (Number of writes between range tombstones)
      type: int64 default: 0
```

# cache_bench
```
./cache_bench --help
cache_bench: Warning: SetUsageMessage() never called
...
  Flags from cache/cache_bench.cc:
    -cache_size (Number of bytes to use as a cache of uncompressed data.)
      type: int64 default: 8388608
    -erase_percent (Ratio of erase to total workload (expressed as a
      percentage)) type: int32 default: 10
    -insert_percent (Ratio of insert to total workload (expressed as a
      percentage)) type: int32 default: 40
    -lookup_percent (Ratio of lookup to total workload (expressed as a
      percentage)) type: int32 default: 50
    -max_key (Max number of key to place in cache) type: int64
      default: 1073741824
    -num_shard_bits (shard_bits.) type: int32 default: 4
    -ops_per_thread (Number of operations per thread.) type: uint64
      default: 1200000
    -populate_cache (Populate cache before operations) type: bool
      default: false
    -threads (Number of concurrent threads to run.) type: int32 default: 16
    -use_clock_cache () type: bool default: false
```

# persistent_cache_bench
```
$ ./persistent_cache_bench -help
persistent_cache_bench:
USAGE:
./persistent_cache_bench [OPTIONS]...
...
  Flags from utilities/persistent_cache/persistent_cache_bench.cc:
    -benchmark (Benchmark mode) type: bool default: false
    -cache_size (Cache size) type: uint64 default: 18446744073709551615
    -cache_type (Cache type. (block_cache, volatile, tiered)) type: string
      default: "block_cache"
    -enable_pipelined_writes (Enable async writes) type: bool default: false
    -iosize (Read IO size) type: int32 default: 4096
    -log_path (Path for the log file) type: string default: "/tmp/log"
    -nsec (nsec) type: int32 default: 10
    -nthread_read (Lookup threads) type: int32 default: 1
    -nthread_write (Insert threads) type: int32 default: 1
    -path (Path for cachefile) type: string default: "/tmp/microbench/blkcache"
    -volatile_cache_pct (Percentage of cache in memory tier.) type: int32
      default: 10
    -writer_iosize (File writer IO size) type: int32 default: 4096
    -writer_qdepth (File writer qdepth) type: int32 default: 1