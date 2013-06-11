Multifeed is a system which serves the news feed at Facebook. This page contains benchmarks with workload similar to the one in Multifeed.

##Workload
Each database is around 60GB in size. The benchmark assumes a value size of 800.

##Method
Ran fillrandom, overwrite, updaterandom, readrandom by varying the L1 Sizes 256MB, 512MB, 1024MB
##Command
```bash
./db_bench --benchmarks=fillrandom --disable_seek_compaction=1 --mmap_read=0 --statistics=1 --histogram=1 
\--readahead=0 --num=80530636 --threads=1 --value_size=800 --block_size=16384 --cache_size=17179869184 
\--bloom_bits=10 --cache_numshardbits=4 --open_files=50000000 --verify_checksum=1 
\--db=/data/mysql/1024_32_read_while_writing --disable_data_sync=0 --disable_wal=0 
\--compression_type=none --stats_interval=10000 --compression_ratio=0.50 --write_buffer_size=67108864 
\--target_file_size_base=33554432 --max_write_buffer_number=3 --max_background_compactions=8 
\--level0_file_num_compaction_trigger=8 --level0_slowdown_writes_trigger=17 --level0_stop_writes_trigger=24 
\--num_levels=6 --delete_obsolete_files_period_micros=5000000 --min_level_to_compress=3 --stats_per_interval=1 
\--disable_auto_compactions=0 --source_compaction_factor=1 --use_existing_db=0 --seed=1365556909 
\--rate_limit=2 --mmap_write=0 --max_bytes_for_level_base=1073741824 --max_bytes_for_level_multiplier=8 
\--writes_per_second=0
```

##Results

Metric | L1 = 256mb | L1 = 512mb | L2 = 1024mb
--- | --- | --- | ---
DB Raw Size (GB) | 61.2 | 61.2 | 61..2
Largest Level | L3 ~16GB | L3 31.98 (L4 has ~16GB) | L3 41.5 GB
