# 1. Setup

All of the benchmarks are run on the same machine. Here are the details of the test setup:

* 12 CPUs, HT enabled -> 24 vCPUs reported, 2 sockets X 6 cores/socket with X5650 @ 2.67GHz
* 2 FusionIO devices in SW RAID 0 that can do ~200k 4kb read/second at peak
* Fusion IO devices were about 50% to 70% full for each of the benchmark runs
* Machine has 144 GB of RAM
* Operating System Linux 2.6.38.4
* 1G rocksdb block cache
* snappy compression enabled
* 1 Billion keys; each key is of size 10 bytes, each value is of size 800 bytes

# 2. Bulk Load of keys in Random Order

Measure performance to load 1B keys into the database. The keys are inserted in random order. The database is empty at the beginning of this benchmark run and gradually fills up. No data is being read when the data load is in progress. 

    rocksdb:   103 minutes, 80 MBytes/sec (total data size 481 GB, 1 billion key-values)
    leveldb:   many many days (in 24 hours it inserted only 2 million keys) 

Rocksdb was configured to first load all the data in L0 with compactions switched off and nd then it made a second pass over the data to merge-sort all the files in L0 into sorted files in L1. Leveldb is very slow because of high write amplification. Here are the command(s) for loading the data into rocksdb

    echo "Bulk load database into L0...."
    bpl=10485760;overlap=10;mcz=2;del=300000000;levels=2;ctrig=10000000; delay=10000000; stop=10000000; wbn=30; mbc=20; mb=1073741824;wbs=268435456; dds=1; sync=0; r=1000000000; t=1; vs=800; bs=65536; cs=1048576; of=500000; si=1000000; ./db_bench --benchmarks=fillrandom --disable_seek_compaction=1 --mmap_read=0 --statistics=1 --histogram=1 --num=$r --threads=$t --value_size=$vs --block_size=$bs --cache_size=$cs --bloom_bits=10 --cache_numshardbits=4 --open_files=$of --verify_checksum=1 --db=/data/mysql/leveldb/test --sync=$sync --disable_wal=1 --compression_type=zlib --stats_interval=$si --compression_ratio=50 --disable_data_sync=$dds --write_buffer_size=$wbs --target_file_size_base=$mb --max_write_buffer_number=$wbn --max_background_compactions=$mbc --level0_file_num_compaction_trigger=$ctrig --level0_slowdown_writes_trigger=$delay --level0_stop_writes_trigger=$stop --num_levels=$levels --delete_obsolete_files_period_micros=$del --min_level_to_compress=$mcz --max_grandparent_overlap_factor=$overlap --stats_per_interval=1 --max_bytes_for_level_base=$bpl --memtablerep=vector --use_existing_db=0 --disable_auto_compactions=1 --source_compaction_factor=10000000
    echo "Running manual compaction to do a global sort map-reduce style...."
    bpl=10485760;overlap=10;mcz=2;del=300000000;levels=2;ctrig=10000000; delay=10000000; stop=10000000; wbn=30; mbc=20; mb=1073741824;wbs=268435456; dds=1; sync=0; r=1000000000; t=1; vs=800; bs=65536; cs=1048576; of=500000; si=1000000; ./db_bench --benchmarks=compact --disable_seek_compaction=1 --mmap_read=0 --statistics=1 --histogram=1 --num=$r --threads=$t --value_size=$vs --block_size=$bs --cache_size=$cs --bloom_bits=10 --cache_numshardbits=4 --open_files=$of --verify_checksum=1 --db=/data/mysql/leveldb/test --sync=$sync --disable_wal=1 --compression_type=zlib --stats_interval=$si --compression_ratio=50 --disable_data_sync=$dds --write_buffer_size=$wbs --target_file_size_base=$mb --max_write_buffer_number=$wbn --max_background_compactions=$mbc --level0_file_num_compaction_trigger=$ctrig --level0_slowdown_writes_trigger=$delay --level0_stop_writes_trigger=$stop --num_levels=$levels --delete_obsolete_files_period_micros=$del --min_level_to_compress=$mcz --max_grandparent_overlap_factor=$overlap --stats_per_interval=1 --max_bytes_for_level_base=$bpl --memtablerep=vector --use_existing_db=1 --disable_auto_compactions=1 --source_compaction_factor=10000000
    du -s -k test
    504730832	test


Here are the command(s) for loading the data into leveldb:

    echo "Bulk load database ...."
    wbs=268435456; r=1000000000; t=1; vs=800; cs=1048576; of=500000; ./db_bench --benchmarks=fillrandom --num=$r --threads=$t --value_size=$vs --cache_size=$cs --bloom_bits=10 --open_files=$of --db=/data/mysql/leveldb/test --compression_ratio=50 --write_buffer_size=$wbs --use_existing_db=0

# 3. Bulk Load of keys in Sequential Order

Measure performance to load 1B keys into the database. The keys are inserted in sequential order. The database is empty at the beginning of this benchmark run and gradually fills up. No data is being read when the data load is in progress.

    rocksdb:   36 minutes, 370 MBytes/sec (total data size 760 GB)
    leveldb:   91 minutes, 146 MB/sec (total data size is 760 GB)

Rocksdb was configured to use multi-threaded compactions so that multiple threads could be simultaneously compacting (via file-renames) non-overlapping key ranges in multiple levels. This was the primary reason why rocksdb is much much faster than leveldb for this workload. Here are the command(s) for loading the database into rocksdb.

    echo "Load 1B keys sequentially into database....."
    bpl=10485760;overlap=10;mcz=2;del=300000000;levels=6;ctrig=4; delay=8; stop=12; wbn=3; mbc=20; mb=67108864;wbs=134217728; dds=0; sync=0; r=1000000000; t=1; vs=800; bs=65536; cs=1048576; of=500000; si=1000000; ./db_bench --benchmarks=fillseq --disable_seek_compaction=1 --mmap_read=0 --statistics=1 --histogram=1 --num=$r --threads=$t --value_size=$vs --block_size=$bs --cache_size=$cs --bloom_bits=10 --cache_numshardbits=4 --open_files=$of --verify_checksum=1 --db=/data/mysql/leveldb/test --sync=$sync --disable_wal=1 --compression_type=zlib --stats_interval=$si --compression_ratio=50 --disable_data_sync=$dds --write_buffer_size=$wbs --target_file_size_base=$mb --max_write_buffer_number=$wbn --max_background_compactions=$mbc --level0_file_num_compaction_trigger=$ctrig --level0_slowdown_writes_trigger=$delay --level0_stop_writes_trigger=$stop --num_levels=$levels --delete_obsolete_files_period_micros=$del --min_level_to_compress=$mcz --max_grandparent_overlap_factor=$overlap --stats_per_interval=1 --max_bytes_for_level_base=$bpl --use_existing_db=0

Here are the command(s) for loading the data into leveldb:

    echo "Load 1B keys sequentially into database....."
    wbs=134217728; r=1000000000; t=1; vs=800; cs=1048576; of=500000; ./db_bench --benchmarks=fillseq --num=$r --threads=$t --value_size=$vs --cache_size=$cs --bloom_bits=10 --open_files=$of --db=/data/mysql/leveldb/test --compression_ratio=50 --write_buffer_size=$wbs --use_existing_db=0

# 4. Write Performance

Measure performance to overwrite 1B keys into the database. The database was first created by sequentially inserting all the 1 B keys. The results here do not measure the sequential-insertion phase, it measures only second part of the test that overwrites 1 B keys in random order.

    rocksdb: 15 hours 38 minutes; 56.295 micros/op 17763 ops/sec 13.8 MB/s; P99.99: 11636.26 micros
    leveldb: 
    
Here are the commands to overwrite 1 B keys in rocksdb:

    echo "Overwriting the 1B keys in database in random order...."
    bpl=10485760;overlap=10;mcz=2;del=300000000;levels=6;ctrig=4; delay=8; stop=12; wbn=3; mbc=20; mb=67108864;wbs=134217728; dds=0; sync=0; r=1000000000; t=1; vs=800; bs=65536; cs=1048576; of=500000; si=1000000; ./db_bench --benchmarks=overwrite --disable_seek_compaction=1 --mmap_read=0 --statistics=1 --histogram=1 --num=$r --threads=$t --value_size=$vs --block_size=$bs --cache_size=$cs --bloom_bits=10 --cache_numshardbits=4 --open_files=$of --verify_checksum=1 --db=/data/mysql/leveldb/test --sync=$sync --disable_wal=1 --compression_type=zlib --stats_interval=$si --compression_ratio=50 --disable_data_sync=$dds --write_buffer_size=$wbs --target_file_size_base=$mb --max_write_buffer_number=$wbn --max_background_compactions=$mbc --level0_file_num_compaction_trigger=$ctrig --level0_slowdown_writes_trigger=$delay --level0_stop_writes_trigger=$stop --num_levels=$levels --delete_obsolete_files_period_micros=$del --min_level_to_compress=$mcz --max_grandparent_overlap_factor=$overlap --stats_per_interval=1 --max_bytes_for_level_base=$bpl --use_existing_db=1

Here are the commands to overwrite 1 B keys in leveldb:

    echo "Overwriting the 1B keys in database in random order...."
    wbs=268435456; r=1000000000; t=1; vs=800; cs=1048576; of=500000; ./db_bench --benchmarks=overwrite --num=$r --threads=$t --value_size=$vs --cache_size=$cs --bloom_bits=10 --open_files=$of --db=/data/mysql/leveldb/test --compression_ratio=50 --write_buffer_size=$wbs --use_existing_db=1


  