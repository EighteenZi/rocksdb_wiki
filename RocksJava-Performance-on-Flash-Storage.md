In April'14, we started building a [Java extension](https://github.com/facebook/rocksdb/wiki/RocksJava-Basics) for RocksDB.  This page shows the benchmark results of RocksJava on flash storage.  The benchmark results of RocksDB C++ on flash storage can be found [here](https://github.com/facebook/rocksdb/wiki/Performance-Benchmarks).

# Setup
All of the benchmarks are run on the same machine. Here are the details of the test setup:

* 1 billion keys; each key / value has 16 / 800 bytes; ~1TB DB.
* Intel(R) Xeon(R) CPU E5-2660 v2 @ 2.20GHz, 40 cores.
* 25 MB CPU cache, 144 GB RAM.
* Fusion-io ioScale 3.20TB.
* Java 7 --- Java(TM) SE Runtime Environment (build 1.7.0_55-b13)
* Java HotSpot(TM) 64-Bit Server VM (build 24.55-b03, mixed mode)
* Snappy compression is used.
* JEMALLOC is not used.

# Bulk Load of keys in Sequential Order (Test 2)
Measure performance to load 1B keys into the database. The keys are inserted in sequential order. The database is empty at the beginning of this benchmark run and gradually fills up. No data is being read when the data load is in progress.

    fillseq          :     2.48233 micros/op;  311.2 MB/s; 1000000000 ops done;  1 / 1 task(s) finished.

Rocksdb was configured to use multi-threaded compactions so that multiple threads could be simultaneously compacting (via file-renames) non-overlapping key ranges in multiple levels. 

Here are the command(s) for loading the database into rocksdb.

    bpl=10485760;overlap=10;mcz=0;del=300000000;levels=6;ctrig=4; delay=8; stop=12; wbn=3; mbc=20; mb=67108864;wbs=134217728; dds=0; sync=false; t=1; vs=800; bs=65536; cs=1048576; of=500000; si=1000000;
    ./jdb_bench.sh  --benchmarks=fillseq  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --threads=$t  --key_size=10  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --compression_type=snappy  --cache_numshardbits=4  --open_files=$of  --verify_checksum=true  --db=/rocksdb-bench/java/b2  --sync=$sync  --disable_wal=true  --stats_interval=$si  --compression_ratio=0.50  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=false  --cache_remove_scan_count_limit=16 --num=1000000000

# Random Read (Test 4)
Measure random read performance of a database with 1 Billion keys, each key is 16 bytes and value is 800 bytes. RocksDB is configured with a block size of 4 KB.  Snappy compression is enabled. There were 32 threads in the benchmark application issuing random reads to the database.  In addition, RocksDB is configured to verify checksums on every read.

    readrandom       :     7.67180 micros/op;  101.4 MB/s; 1000000000 / 1000000000 found;  32 / 32 task(s) finished.

Data was first loaded into the database by sequentially writing all the 1B keys to the database. Once the load is complete, the benchmark randomly picks a key and issues a read request. The above measure measurement does not include the data loading part, it measures only the part that issues the random reads to database.

Here are the commands used to run the benchmark with rocksdb:

    echo "Load 1B keys sequentially into database....."
    n=1000000000; r=1000000000; bpl=10485760;overlap=10;mcz=2;del=300000000;levels=6;ctrig=4; delay=8; stop=12; wbn=3; mbc=20; mb=67108864;wbs=134217728; dds=1; sync=false; t=1; vs=800; bs=4096; cs=1048576; of=500000; si=1000000;
    ./jdb_bench.sh  --benchmarks=fillseq  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --num=$n  --threads=$t  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --cache_numshardbits=6  --open_files=$of  --verify_checksum=true  --db=/rocksdb-bench/java/b4  --sync=$sync  --disable_wal=true  --compression_type=snappy  --stats_interval=$si  --compression_ratio=0.50  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --min_level_to_compress=$mcz  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=false

    echo "Reading 1B keys in database in random order...."
    bpl=10485760;overlap=10;mcz=2;del=300000000;levels=6;ctrig=4; delay=8; stop=12; wbn=3; mbc=20; mb=67108864; wbs=134217728; dds=0; sync=false; t=32; vs=800; bs=4096; cs=1048576; of=500000; si=1000000;
    ./jdb_bench.sh  --benchmarks=readrandom  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --num=$n  --reads=$r  --threads=$t  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --cache_numshardbits=6  --open_files=$of  --verify_checksum=true  --db=/rocksdb-bench/java/b4  --sync=$sync  --disable_wal=true  --compression_type=none  --stats_interval=$si  --compression_ratio=0.50  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --min_level_to_compress=$mcz  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=true

# Multi-Threaded Read and Single-Threaded Write (Test 5)
Measure performance to randomly read 100M keys out of database with 1B keys and ongoing updates to existing keys. Number of read keys is configured to 100M to shorten the experiment time.  Each key is 16 bytes and value is 800 bytes. RocksDB is both configured with a block size of 4 KB.  Snappy compression is enabled. There are 32 dedicated read threads in the benchmark issuing random reads to the database.  A separate thread issues writes to the database at 10k writes per second.

    readwhilewriting :     9.55882 micros/op;   81.4 MB/s; 100000000 / 100000000 found;  32 / 32 task(s) finished.

Here are the commands used to run the benchmark with RocksJava:

    echo "Load 1B keys sequentially into database....."
    dir="/rocksdb-bench/java/b5"
    num=1000000000; r=100000000;  bpl=536870912;  mb=67108864;  overlap=10;  mcz=2;  del=300000000;  levels=6;  ctrig=4; delay=8;  stop=12;  wbn=3;  mbc=20;  wbs=134217728;  dds=false;  sync=false;  vs=800;  bs=4096;  cs=17179869184; of=500000;  wps=0;  si=10000000;
    ./jdb_bench.sh  --benchmarks=fillseq  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --num=$num  --threads=1  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --cache_numshardbits=6  --open_files=$of  --verify_checksum=true  --db=$dir  --sync=$sync  --disable_wal=true  --compression_type=snappy  --stats_interval=$si  --compression_ratio=0.5  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --min_level_to_compress=$mcz  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=false
    echo "Reading while writing 100M keys in database in random order...."
    bpl=536870912;mb=67108864;overlap=10;mcz=2;del=300000000;levels=6;ctrig=4;delay=8;stop=12;wbn=3;mbc=20;wbs=134217728;dds=false;sync=false;t=32;vs=800;bs=4096;cs=17179869184;of=500000;wps=10000;si=10000000;
    ./jdb_bench.sh  --benchmarks=readwhilewriting  --disable_seek_compaction=true  --mmap_read=false  --statistics=true  --histogram=true  --num=$num  --reads=$r  --writes_per_second=10000  --threads=$t  --value_size=$vs  --block_size=$bs  --cache_size=$cs  --bloom_bits=10  --cache_numshardbits=6  --open_files=$of  --verify_checksum=true  --db=$dir  --sync=$sync  --disable_wal=false  --compression_type=snappy  --stats_interval=$si  --compression_ratio=0.5  --disable_data_sync=$dds  --write_buffer_size=$wbs  --target_file_size_base=$mb  --max_write_buffer_number=$wbn  --max_background_compactions=$mbc  --level0_file_num_compaction_trigger=$ctrig  --level0_slowdown_writes_trigger=$delay  --level0_stop_writes_trigger=$stop  --num_levels=$levels  --delete_obsolete_files_period_micros=$del  --min_level_to_compress=$mcz  --max_grandparent_overlap_factor=$overlap  --stats_per_interval=1  --max_bytes_for_level_base=$bpl  --use_existing_db=true  --writes_per_second=$wps
