#About
Find RocksDB performance with readwhilewriting. 

One thread is writing and other threads are reading from the DB.

Vary reader thread count as 1,16,32.

Used a L1 Size of 512 MB and file size of 64mb.

## Method
Create a 800G database by using fill random. Value size is 800 bytes. The experiments run on machine with 144G ram and a 24 core Intel(R) Xeon(R) CPU 2.67GHz, and on Flash hard disk drives, XFS file system.

##Command
The db_bench command is
```bash
    ./db_bench --benchmarks=readwhilewriting --disable_seek_compaction=1 \
    --mmap_read=0 --statistics=1 --histogram=1 --readahead=0 --num=10000 \
    --threads=2 --value_size=800 --block_size=16384 --cache_size=17179869184 \
    --bloom_bits=10 --cache_numshardbits=4 --open_files=50000000 --verify_checksum=1 \
    --db=/data/mysql/512_64_read_while_writing --disable_data_sync=0 --disable_wal=0 \
    --compression_type=none --stats_interval=10000 --compression_ratio=0.50 \
    --write_buffer_size=67108864 --target_file_size_base=67108864 --max_write_buffer_number=3 \
    --max_background_compactions=8 --level0_file_num_compaction_trigger=8 \
    --level0_slowdown_writes_trigger=17 --level0_stop_writes_trigger=24 --num_levels=6 \
    --delete_obsolete_files_period_micros=5000000 --min_level_to_compress=3 \
    --stats_per_interval=1 --disable_auto_compactions=0 --source_compaction_factor=1 \
    --use_existing_db=1 --seed=1365556909 --rate_limit=2 --mmap_write=0 \
    --max_bytes_for_level_base=536870912 --max_bytes_for_level_multiplier=8 \
    --writes_per_second=10000 --duration=1800
```

## Results
l1 = 512 MB
f = 64 MB
No Compression
Writes limited to 10,000 /s.

num = 10000

Metric | #reader_thread=1 | #reader_thread=8 | #reader_thread=16 | #reader_thread=32
--- | --- | --- | --- | ---
readwhilewriting (ops/s) | 437230 | 523286 | 437207 | 444108
micros/op | 4.574 | 17.198 | 38.882 | 74.306
DB_GET p99 (µs) | 7.28 | 31.65 | 75.32 | 328.51
DB_WRITE p99 (µs) | 17.36 | 47.24 | 98.15 | 533.74
L0 Stalls (s) | 10.381 | 0 | 0 | 0
Ln stall (s) | 0 | 0 | 0 | 0
Largest Level | 300 GB L5 | 300 GB L5 | 300 GB L5 | 300 GB L5


Raw files and pmp files available at /home/abhishekk/perf_rocks/readwhilewriting 

## Observations
From PMP.
* Most time is spent waiting for Mutex in Get.
* Key Compare in SkipList could be a costly place 3% time spent according to pmp. method to look at => KeyIsAfterNode