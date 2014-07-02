The purpose of this guide is to provide you with enough information so you can tune RocksDB for your workload and your system configuration. RocksDB is very flexible, which is both good and bad. It is good because you can tune it for variety of workloads and storage technologies. Inside of Facebook we use the same code base for in-memory workload, flash devices and spinning disks. However, flexibility is not always user-friendly. We introduced a huge number of tuning options that are sometimes confusing and hard to understand. We hope that by reading this guide you will be able to squeeze the last drop of performance out of your system and fully utilize your resources.

We assume you have basic knowledge of how Log-structured merge-tree (LSM) works. There are already plenty of great resources on LSM, no need to write one more.

## Amplification factors
When we talk about tuning RocksDB, we often mean that we're trading off three amplification factors: write amplification, read amplification and space amplification. Let's define them here.

**Write amplification** is the ratio of bytes written to database and bytes written to storage. For example, if you are writing 10 MB/s to the database and you observe 30 MB/s disk write rate, your write amplification is 3. If write amplification is high the workload might be bottlenecked on disk throughput. For example, if write amplification is 50 and max disk throughput is 500 MB/s, your database can sustain 10 MB/s write rate. Decreasing write amplification will directly increase max write rate. High write amplification will also decrease flash lifetime.

**Read amplification** is the number of disk reads per query. If you need to read 5 pages to answer a query, we say that read amplification is 5.

**Space amplification** is the ratio of the size of database files on disk and data size. If you store 10MB in the database and it stores 100MB on disk, this is space amplification of 10. Usually you will want to set hard limit on space amplification since you don't want to run out of disk space or memory.

If you want to learn more about the three amplification factors in context of different database algorithms we strongly recommend Mark Callaghan's talk at Highload[1].

## Parallelism options
In LSM architecture, there are two background processes: flush and compaction. Both of them can execute concurrently to take full advantage of storage technology concurrency. Flush threads are submitted to HIGH priority pool, while compaction threads are submitted to LOW priority pool. To increase number of threads in respective thread pools call:

     options.env->SetBackgroundThreads(num_threads, Env::Priority::HIGH);
     options.env->SetBackgroundThreads(num_threads, Env::Priority::LOW);
 
Once you increase number of threads in thread pool, you can also  increase max number of parallel compactions and flushes:

**max_background_compactions** is the maximum number of concurrent background compactions. Default is 1, but to fully utilize your CPU and storage you might want to increase this to approximately number of cores in the system.

**max_background_flushes** is the maximum number of concurrent flush operations. It is usually good enough to set this to 1.

## Flushing options
All writes to RocksDB are first inserted into an in-memory data structure called memtable. Once the **active memtable** is full, we create a new one and mark the old one read-only. We call read-only memtable **immutable**. At any point in time there is exactly one active memtable and zero or more immutable memtables. Immutable memtables are waiting to be flushed to storage. There are three options that control flushing behavior.

**write_buffer_size** is the size of a single memtable. Once memtable exceeds this size, we mark it immutable and create a new one.

**max_write_buffer_number** is the maximum number of memtables, both active and immutable. If the active memtable fills up and the total number of memtables is bigger than max_write_buffer_number we stall further writes. This can happen if flush process is slower than the write rate.

**min_write_buffer_number_to_merge** is the minimum number of memtables that will be merged together before flushing to storage. For example, if this option is set to 2, we will not flush a single immutable memtable. We will only start flushing immutable memtables once there are two of them. If we merge multiple memtables together, we might also write less data to storage since we will merge two updates to a single key. However, every Get() needs to traverse all immutable memtables linearly to check if the key is there. If this option is set too high it might hurt read performance.

Let's assume your options are:
  
    write_buffer_size = 512MB;
    max_write_buffer_number = 5;
    min_write_buffer_number_to_merge = 2;

and your write rate is 16MB/s. New memtable will be created every 32 seconds and two memtables will be merged together and flushed every 64 seconds. Depending on the working set size, your flush size will be somewhere from 512MB to 1GB. In case flushing can't keep up with write rate, the memory used by memtables will be capped at 5*512MB = 2.5GB. Once we reach that, we will block any further writes until the flush finishes and frees memory used by the memtables. 

## Level Style Compaction
In level style compaction, database files are organized into levels. Memtables are flushed to files in level 0, which contains the newest data. Higher levels contain older data with higher level containing oldest data. Files in level 0 can be overlapping, but files in level 1 and higher are non-overlapping. That means that Get() usually needs to check each file from level 0, but for each next level there can be no more than one file containing the key. Each level is 10 times bigger than the previous one (this multiplier is configurable). A compaction will take few files from level N and compact them with overlapping files from level N+1. Two compactions operating at different levels or at different key ranges are independent and can be executed concurrently. Speed of compaction is directly proportional to max write rate. If compaction can't keep up with the write rate, space used by the database will just keep increasing. It is important to configure RocksDB in such a way that compactions can be executed with high concurrency and fully utilize storage.

Compactions at levels 0 and 1 are tricky. Files at level 0 usually span the entire key space. When compacting L0->L1 (from level 0 to level 1), compaction includes all files from level 1. With all files from L1 getting compacted with L0, compaction L1->L2 can not proceed -- it has to wait for the L0->L1 compaction to finish. If L0->L1 compaction is slow, it will be the only compaction running in the system most of the time, since other compactions will have to wait for L0->L1 to finish. You can check if that's the issue by checking disk utilization. If disk is not fully utilized, there might be an issue with compaction configuration. The advice we often give is to make L0->L1 as fast as possible by making size of level 0 similar to size of level 1.

Once you determine the size of level 1, you will need to decide the level multiplier. Let's assume your level 1 size is 512 MB, level multiplier is 10 and size of the database is 500GB. Level 2 size will then be 5GB, level 3 51GB and level 4 512GB. Since your database size is 500GB, levels 5 and higher will be empty. Size amplification is easy to calculate. It is `(512 MB + 512 MB + 5GB + 51GB + 512GB) / (500GB) = 1.14`. Here is how we would calculate write amplification: every byte first gets written out to level 0. Next, it needs to be compacted into level 1. Since level 1 size is the same as level 0, write amplification of L0->L1 compaction is 2. However, when a byte from level 1 needs to get compacted into level 2, we need to compact it with 10 bytes from level 2 (because level 2 is 10x bigger). The same is also true for L2->L3 and L3->L4 compactions. Total write amplification is then approximately `1 + 2 + 10 + 10 + 10 = 33`. Point lookups need to consult all files in level 0 and at most one file from each other levels. However, bloom filters help and greatly reduce read amplification. Short-lived range scans are a bit more expensive, however. Bloom filters are not useful for range scans, so the read amplification is `number_of_level0_files + number_of_non_empty_levels`.

Let's dive into options that control level compaction. We will start with more important ones and follow with less important ones.

**level0_file_num_compaction_trigger** -- Once level 0 reaches that many files, L0->L1 compaction will be triggered. This means that we can estimate level 0 size in stable state as `write_buffer_size * min_write_buffer_number_to_merge * level0_file_num_compaction_trigger`.

**max_bytes_for_level_base** and **max_bytes_for_level_multiplier** -- max_bytes_for_level_base is total size of level 1. As recommended, this should be similar to size of level 0. Each next level is max_bytes_for_level_multiplier bigger than previous one. The default is 10 and we don't recommend changing that.

**target_file_size_base** and **target_file_size_multiplier** -- Files in level 1 will have target_file_size_base bytes. Each next level's file size will be target_file_size_multiplier bigger than previous one. However, by default target_file_size_multiplier is 1, so files in all L1..Lmax levels are equal. Increasing target_file_size_base will reduce total number of database files, which is generally a good thing. We recommend setting target_file_size_base to be `max_bytes_for_level_base / 10`, so that there are 10 files in level 1.

**compression_per_level** -- You can use this option to set different compressions for different levels. It usually makes sense to avoid compressing levels 0 and 1 and only compress data in higher levels. You can even set slower compression in highest level and faster compression in lower levels (by highest we mean Lmax).

**num_levels** -- It is safe for num_levels to be bigger than expected number of levels in the database. Some higher levels might be empty, which will not impact performance in any way. You need to change this option only if you expect your number of levels will be bigger than 7 (default).

## Universal Compaction
TODO

## Write stalls
RocksDB has extensive system to slow down writes when compaction can't keep up with incoming write rate. Without such system, short-lived write bursts would: 1) increase space amplification, which could lead to running out of disk space, 2) increase read amplification, which would significantly degrade read performance. The idea is to smooth out write bursts by slowing down writes. Options that control write stalls are:

**level0_slowdown_writes_trigger** and **level0_stop_writes_trigger** -- When number of level0 files is bigger than slowdown limit, we stall writes. When the number is bigger than stop limit, we fully stop writes and wait until compaction is done.

**soft_rate_limit** and **hard_rate_limit** -- In level style compaction, each level has its compaction score. Once compaction score is bigger than 1, we trigger the compaction. In case the score for any level is bigger than soft_rate_limit, we will slow down writes. If score is bigger than hard_rate_limit, writes will be stopped until the compaction for that level reduces its score.

## Other options
**block_cache** -- Usually you just want to set this to the result of the call `rocksdb::NewLRUCache(cache_capacity, shard_bits)`. We need to lock accesses to block cache and sometimes we see RocksDB bottlenecked on block cache's mutex, especially when DB size is smaller than RAM. In that case, it makes sense to shard block cache by setting shard_bits to a bigger number. If shard_bits is 4, total number of shards will be 16.

**max_open_files** -- RocksDB keeps all file descriptors in a table cache. If number of file descriptors exceeds max_open_files, some files are evicted from table cache and their file descriptors closed. This means that every read has to go through the table cache to lookup the file needed. If you set max_open_files to -1 we will always keep all files opened which avoids expensive table cache calls.

**table_cache_numshardbits** -- This option controls table cache sharding and it makes sense to increase it if you see that table cache mutex is contended.

## Sharing Env and cache
TODO describe how thread pool and cache can be shared between different RocksDB instances.

## Prefix databases
TODO Explain what happens when prefix_extractor is set and how to take full advantage of prefix API.

## Example configurations
TODO. Here we can also share some interesting configurations, for example when inserting monotonically increasing keys, doing only prefix lookups, etc.

[1] http://vimeo.com/album/2920922/video/98428203