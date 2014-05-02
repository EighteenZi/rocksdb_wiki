RocksJava is a project that we initiated in April 2014 to build a high performance Java driver for RocksDB.  Here we would like to release its very first performance numbers on flash storage.  This page will first present the result summary.  Details about experimental setup and commands to run the benchmark will be covered in the later sections.

# Result Summary
We repeated the benchmarks on flash storage described in [3] and compare the performance between RocksJava and the RocksDB C++.  The database used in the benchmarks has one billion key-values, and each key / value has 16 / 800 bytes respectively.  Below are the performance comparison between RocksJava and RocksDB c++ on different workloads:

### Table 1.  Performance comparison over 1TB database on flash storage.
| Benchmark                          | RocksJava|RocksDB C++| Diff(%) |     Detail Setting      |
|------------------------------------|---------:|----------:|:-------:|:-----------------------:|
|1 Sequential Writer                 | 369k wps | 371K wps  |   < 1%  | [link](#fillseq)        |
|32 Random Readers	             | 270K rps | 303K rps  | -10.8%  | [link](#readrandom)     |
|32 Random Readers <br/>w/ 1 Random Writer| 206K rps |  336K rps | -38.5% | [link](#readwhilewriting) |
|32 Sequential Readers               |2.12M rps      | 6.84M rps | -69.0% | [link](#readseq)|
* wps: writes per second.
* rps: reads per second.

Here we further discuss the 32 sequential readers benchmark where RocksJava is 70% slower than RocksDB C++.

Sequential-read is a relative high qps operation compared to the operations in other benchmarks. As a result, the overhead on the Java side will become more noticeable. In addition, in the current implementation, each read in RocksJava's sequential reader involves in four JNI calls: which are `next()`, `isValid()`, `key()`, and `value()`, while the operations used in other benchmarks only involve one JNI call:

```java
    @Override public void runTask() throws RocksDBException {
      org.rocksdb.Iterator iter = db_.newIterator();
      long i;
      for (iter.seekToFirst(), i = 0;
           iter.isValid() && i < numEntries_;
           iter.next(), ++i) {
        stats_.found_++;
        stats_.finishedSingleOp(iter.key().length + iter.value().length);
        if (isFinished()) {
          return;
        }
      }
    }
```

This can be improved by introducing a better api that combines some of these functions together (such as `boolean nextValid()`).

# Setup
We tried to reuse the settings used in [].  Here are some of important settings / difference used in this benchmark:

* Intel(R) Xeon(R) CPU E5-2660 v2 @ 2.20GHz, 40 cores.
* 25 MB CPU cache, 144 GB Ram
* CentOS release 5.2 (Final).
* Experiments were run under Funtoo chroot environment.
* g++ (Funtoo 4.8.1-r2) 4.8.1
* Java(TM) SE Runtime Environment (build 1.7.0_55-b13)
* Java HotSpot(TM) 64-Bit Server VM (build 24.55-b03, mixed mode)
* Commit [61955a0](https://github.com/facebook/rocksdb/commit/61955a0dda8222673196cd81a0afe92bbc61575a) was used in the experiment.
* 1G rocksdb block cache
* Snappy 1.1.1 is used as the compression algorithm.
* Test with 1 billion key / value pairs. Each key is 16 bytes, and each value is 800 bytes. Total database size is 800GB
* For 32 readers w/ 1 writer benchmark, the writer performs 10k writes per second.
* JEMALLOC is not used.

<a name="fillseq"/>
## Bulk Load of keys in Sequential Order
This benchmark measures the performance of loading 1B keys into the database. The keys are inserted in sequential order. The database is empty at the beginning of this benchmark run and gradually fills up. No data is being read when the data load is in progress.  In this benchmark, each key / value is 16 / 800 bytes respectively.

Here're the commands we used to run the benchmark:

RocksJava:

```c++
    java -server -d64 -XX:NewSize=4m -XX:+AggressiveOpts -Djava.library.path=.:../ -cp rocksdbjni.jar:.:./* \
        org.rocksdb.benchmark.DbBenchmark --benchmarks=fillseq --disable_seek_compaction=true \
        --mmap_read=false --statistics=true --histogram=true --threads=1 --value_size=800 \
        --block_size=65536 --cache_size=1048576 --bloom_bits=10 --cache_numshardbits=4 \
        --open_files=500000 --verify_checksum=true --db=/rocksdb-bench/java/b2 --sync=false \
        --disable_wal=true --stats_interval=1000000 --compression_ratio=0.50 --disable_data_sync=0 \
        --write_buffer_size=134217728 --target_file_size_base=67108864 --max_write_buffer_number=3 \
        --max_background_compactions=20 --level0_file_num_compaction_trigger=4 \
        --level0_slowdown_writes_trigger=8 --level0_stop_writes_trigger=12 --num_levels=6 \
        --delete_obsolete_files_period_micros=300000000 --min_level_to_compress=0
        --max_grandparent_overlap_factor=10 --stats_per_interval=1 --max_bytes_for_level_base=10485760 \
        --use_existing_db=false --cache_remove_scan_count_limit=16 --num=1000000000
```
RocksDB C++:
```c++
    ./db_bench --benchmarks=fillseq --disable_seek_compaction=1 \
        --mmap_read=0 --statistics=0 --histogram=0 --threads=1 --value_size=800 \
        --block_size=65536 --cache_size=1048576 --bloom_bits=10 --cache_numshardbits=4 \
        --open_files=500000 --verify_checksum=1 --db=/rocksdb-bench/cc/b2 --sync=0 \
        --disable_wal=1 --compression_ratio=50 --disable_data_sync=0 \
        --write_buffer_size=134217728 --target_file_size_base=67108864 --max_write_buffer_number=3 \
        --max_background_compactions=20 --level0_file_num_compaction_trigger=4 \
        --level0_slowdown_writes_trigger=8 --level0_stop_writes_trigger=12 --num_levels=6 \
        --delete_obsolete_files_period_micros=300000000 --min_level_to_compress=0 \
        --max_grandparent_overlap_factor=10 --max_bytes_for_level_base=10485760 \
        --use_existing_db=0 --num=1000000000
```

<a name="readrandom"/>
## Random Read
In this benchmark, 32 threads together issue 1 billion random reads in total from the database created in the [sequential bulk load benchmark](#fillseq).

Below are the commands used to run the benchmark.  Note that the argument `--num` has slightly different meanings in the Java db_bench and C++ db_bench:  `--num` in Java db_bench presents the total number of operations among all threads while `--num` in C++ db_bench means the number of operations each thread will perform.

RocksJava:
```C++
    java -server -d64 -XX:NewSize=4m -XX:+AggressiveOpts -Djava.library.path=.:../ -cp rocksdbjni.jar:.:./*
        org.rocksdb.benchmark.DbBenchmark --benchmarks=readrandom --disable_seek_compaction=true \
        --mmap_read=false --statistics=1 --histogram=1 --threads=32 --value_size=800 --block_size=4096 \
        --cache_size=1048576 --bloom_bits=10 --cache_numshardbits=6 --open_files=500000 \
        --verify_checksum=true --db=/rocksdb-bench/java/b2 --sync=false --disable_wal=true \
        --stats_interval=1000000 --compression_ratio=0.5 --disable_data_sync=false \
        --write_buffer_size=134217728 --target_file_size_base=67108864 --max_write_buffer_number=3 \
        --max_background_compactions=20 --level0_file_num_compaction_trigger=4 \
        --level0_slowdown_writes_trigger=8 --level0_stop_writes_trigger=12 --num_levels=6 \
        --delete_obsolete_files_period_micros=300000000 --min_level_to_compress=2 \
        --max_grandparent_overlap_factor=10 --max_bytes_for_level_base=10485760 \
        --use_existing_db=true --num=1000000000
```
RocksDB C++:
```C++
    ./db_bench --benchmarks=readrandom --disable_seek_compaction=1 \
        --mmap_read=0 --threads=32 --value_size=800 --block_size=4096 \
        --cache_size=1048576 --bloom_bits=10 --cache_numshardbits=6 --open_files=500000 \
        --verify_checksum=1 --db=/rocksdb-bench/cc/b2 --sync=0 --disable_wal=1 \
        --compression_type=none --compression_ratio=50 --disable_data_sync=0 \
        --write_buffer_size=134217728 --target_file_size_base=67108864 --max_write_buffer_number=3 \
        --max_background_compactions=20 --level0_file_num_compaction_trigger=4 \
        --level0_slowdown_writes_trigger=8 --level0_stop_writes_trigger=12 --num_levels=6 \
        --delete_obsolete_files_period_micros=300000000 --min_level_to_compress=2 \
        --max_grandparent_overlap_factor=10 --max_bytes_for_level_base=10485760 \
        --use_existing_db=1 --num=31250000
```

<a name="readwhilewriting"/>
## Multi-Threaded Read and Single-Threaded Write

In this benchmark, there are 32 threads issuing random reads while a single writer thread issues 10k writes per second, and we only measure the performance of the reader threads.  The database is again created by the [sequential bulk-load benchmark](#fillseq).  Below are the commands we used to run this benchmark:

RocksJava:
```C++
    java -server -d64 -XX:NewSize=4m -XX:+AggressiveOpts -Djava.library.path=.:../ -cp rocksdbjni.jar:.:./* \
        org.rocksdb.benchmark.DbBenchmark --benchmarks=readwhilewriting --disable_seek_compaction=true \
        --mmap_read=false --statistics=true --histogram=true --threads=32 --value_size=800 \
        --block_size=4096 --cache_size=1048576 --bloom_bits=10 --cache_numshardbits=6 \
        --open_files=500000 --verify_checksum=true --db=/rocksdb-bench/java/b2 \
        --sync=false --disable_wal=true --compression_ratio=0.50 \
        --disable_data_sync=false --write_buffer_size=134217728 --target_file_size_base=67108864 \
        --max_write_buffer_number=3 --max_background_compactions=20 --level0_file_num_compaction_trigger=4 \
        --level0_slowdown_writes_trigger=8 --level0_stop_writes_trigger=12 --num_levels=6 \
        --delete_obsolete_files_period_micros=300000000 --min_level_to_compress=2 \
        --max_grandparent_overlap_factor=10 --max_bytes_for_level_base=10485760 \
        --writes_per_second=10000 --use_existing_db=true --num=1000000000
```
RocksDB C++:
```C++
    ./db_bench --benchmarks=readwhilewriting --disable_seek_compaction=1 \
        --mmap_read=0 --threads=32 --value_size=800 \
        --block_size=4096 --cache_size=1048576 --bloom_bits=10 --cache_numshardbits=6 \
        --open_files=500000 --verify_checksum=1 --db=/rocksdb-bench/cc/b2 \
        --sync=0 --disable_wal=1 --compression_ratio=50 \
        --disable_data_sync=0 --write_buffer_size=134217728 --target_file_size_base=67108864 \
        --max_write_buffer_number=3 --max_background_compactions=20 --level0_file_num_compaction_trigger=4 \
        --level0_slowdown_writes_trigger=8 --level0_stop_writes_trigger=12 --num_levels=6 \
        --delete_obsolete_files_period_micros=300000000 --min_level_to_compress=2 \
        --max_grandparent_overlap_factor=10 --max_bytes_for_level_base=10485760 \
        --writes_per_second=10000 --use_existing_db=1 --num=31250000
```

<a name="readseq"/>
## Sequential Read
In this benchmark, 32 threads concurrently read the whole database sequentially.  The database is again created using the sequential bulk-load benchmark.  Here are the commands we used to run this benchmark:

RocksJava:
```c++
   java -server -d64 -XX:NewSize=4m -XX:+AggressiveOpts -Djava.library.path=.:../ -cp rocksdbjni.jar:.:./*
       org.rocksdb.benchmark.DbBenchmark --benchmarks=readseq --disable_seek_compaction=true \
       --mmap_read=false --statistics=1 --histogram=1 --threads=32 --value_size=800 --block_size=4096 \
       --cache_size=1048576 --bloom_bits=10 --cache_numshardbits=6 --open_files=500000 \
       --verify_checksum=true --db=/rocksdb-bench/java/b2 --sync=false --disable_wal=true \
       --stats_interval=1000000 --compression_ratio=0.5 --disable_data_sync=false \
       --write_buffer_size=134217728 --target_file_size_base=67108864 --max_write_buffer_number=3 \
       --max_background_compactions=20 --level0_file_num_compaction_trigger=4 \
       --level0_slowdown_writes_trigger=8 --level0_stop_writes_trigger=12 --num_levels=6 \
       --delete_obsolete_files_period_micros=300000000 --min_level_to_compress=2 \
       --max_grandparent_overlap_factor=10 --max_bytes_for_level_base=10485760 \
       --use_existing_db=true --num=32000000000
```
RocksDB C++:
```c++
    ./db_bench --benchmarks=readseq --disable_seek_compaction=1 --mmap_read=0 \
        --threads=32 --value_size=800 --block_size=4096 --cache_size=1048576 --bloom_bits=10 \
        --cache_numshardbits=6 --open_files=500000 --verify_checksum=1 --db=/rocksdb-bench/cc/b2 \
        --sync=0 --disable_wal=1 --compression_type=none --compression_ratio=50 --disable_data_sync=0 \
        --write_buffer_size=134217728 --target_file_size_base=67108864 --max_write_buffer_number=3 \
        --max_background_compactions=20 --level0_file_num_compaction_trigger=4 \
        --level0_slowdown_writes_trigger=8 --level0_stop_writes_trigger=12 --num_levels=6 \
        --delete_obsolete_files_period_micros=300000000 --min_level_to_compress=2 \
        --max_grandparent_overlap_factor=10 --max_bytes_for_level_base=10485760 --use_existing_db=1 \
        --threads=32 --num=1000000000
```