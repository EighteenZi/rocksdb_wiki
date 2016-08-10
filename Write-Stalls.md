RocksDB has extensive system to slow down writes when flush or compaction can't keep up with the incoming write rate. Without such a system, if users keep writing more than the hardware can handle, the database will:

* Increase space amplification, which could lead to running out of disk space;
* Increase read amplification, significantly degrading read performance.

The idea is to slowdown incoming write to the speed that the database can handle. However, sometimes the database can be too sensitive to a temporary write burst, or underestimate what the hardware can handle, so that you may get unexpected slowness or query timeout.

To find out whether your DB is suffer from write stalls, you can look at:

* LOG file, which will contain info log when write stalls are triggered;
* [Compaction stats](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide#compaction-stats) found in LOG file.

Stalls may be triggered for the following reasons:

* **Too many memtables.** When the number of memtables waiting to flush is greater or equal to `max_write_buffer_number`, writes are fully stopped to wait for flush finishes. In addition, if `max_write_buffer_number` is greater than 3, and the number of memtables waiting for flush is greater or equal to `max_write_buffer_number - 1`, writes are stalled. In these cases, you will get info logs in LOG file similar to:

    > Stopping writes because we have 5 immutable memtables (waiting for flush), max_write_buffer_number is set to 5

    > Stalling writes because we have 4 immutable memtables (waiting for flush), max_write_buffer_number is set to 5

* **Too many level-0 SST files.** When the number of level-0 SST files reaches `level0_slowdown_writes_trigger`, writes are stalled. When the number of level-0 SST files reaches `level0_stop_writes_trigger`, writes are fully stopped to wait for level-0 to level-1 compaction reduce the number of level-0 files. In these cases, you will get info logs in LOG file similar to

    > Stalling writes because we have 4 level-0 files

    > Stopping writes because we have 20 level-0 files

* **Too many pending compaction bytes.** When estimated bytes pending for compaction reaches `soft_pending_compaction_bytes`, writes are stalled. When estimated bytes pending for compaction reaches `hard_pending_compaction_bytes`, write are fully stopped to wait for compaction. In these cases, you will get info logs in LOG file similar to

    > Stalling writes because of estimated pending compaction bytes 500000000

    > Stopping writes because of estimated pending compaction bytes 1000000000

Whenever stall conditions are triggered, RocksDB will reduce write rate to `delayed_write_rate`, and could possiblely reduce write rate to even lower than `delayed_write_rate` if estimated pending compaction bytes accumulates. One thing worth to note is that slowdown/stop triggers and pending compaction bytes limit are per-column family, and write stalls apply to the whole DB, which means if one column family triggers write stall, the whole DB will be stalled.

There are multiple options you can tune to mitigate write stalls. If write stalls are triggered by pending flushes, you can try:

* Increase `max_background_flushes` to have more flush threads.
* Increase `max_write_buffer_number` to have smaller memtable to flush.

If write stalls are triggered by too many level-0 files or too many pending compaction bytes, compaction is not fast enough to catch up with writes. Note that anything reduce write amplification will reduce the bytes need to write by compactions, thus speeds up compaction.Options to try:

* Increase `max_background_compactions` to have more compaction threads.
* Increase `write_buffer_size` to have large memtable, to reduce write amplification.
* Increase `min_write_buffer_number_to_merge`.

You can also set stop/slowdown triggers and pending compaction bytes limits to huge number to avoid hitting write stall. Also take a look at "What's the fastest way to load data into RocksDB?" in our [FAQ](https://github.com/facebook/rocksdb/wiki/RocksDB-FAQ) if you are bulk loading data to RocksDB.


