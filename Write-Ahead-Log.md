## Overview

Every update to RocksDB is written to two places: 1) an in-memory data structure called memtable (to be flushed to SST files later) and 2) write ahead log(WAL) on disk. In the event of a failure, write ahead logs can be used to completely recover the data in the memtable, which is necessary to restore the database to the original state. In the default configuration, RocksDB guarantees process crash consistency by fflush()ing the WAL after every user write.

## Life Cycle of a WAL


## WAL Configurations

#### DBOptions::wal_dir

`DBOptions::wal_dir` determines the directory where RocksDB stores write-ahead log files. By default WAL is stored under the same directory as data, this config allows WALs to be stored in a separate directory for operational flexibility.

#### DBOptions::WAL_ttl_seconds, DBOptions::WAL_size_limit_MB

These two fields affect how quickly archived WALs will be deleted. Nonzero values indicate the time and disk space threshold to trigger archived WAL deletion. See [options.h](https://github.com/facebook/rocksdb/blob/5.10.fb/include/rocksdb/options.h#L554-L565) for detailed explanation.

#### DBOptions::max_total_wal_size

In order to limit the size of WALs, RocksDB uses `DBOptions::max_total_wal_size` as the trigger of column family flush. Once WAL exceed this size, RocksDB will start forcing the flush of column families whose memtables are backed by the oldest lived WAL file. This config can be useful when column families are updated at non-uniform frequencies. If there's no size limit, users may need to keep really old WALs when the infrequently-updated column families hasn't flushed for a while. 

The default value of 0 means a dynamic limit of [sum of all write_buffer_size * max_write_buffer_number] * 4. 

#### DBOptions::avoid_flush_during_recovery

By default RocksDB replay WAL logs and flush them on DB open, which may create very small SST files. If this option is enabled, RocksDB will try to avoid (but not guaranteed not to) flush during recovery. Also, existing WAL logs will be kept, so that if crash happened before flush, we still have logs to recover from. The default value is false.

#### DBOptions::manual_wal_flush

`DBOptions::manual_wal_flush` determines whether WAL flush will be automatic after every write or purely manual (user must invoke `FlushWAL` to trigger a WAL flush).

#### DBOptions::wal_filter

Through `DBOptions::wal_filter`, users can provide a filter object to be invoked while processing WALs during recovery. This allows skipping some particular records during recovery. Currently the filter is invoked at startup in a single threaded fashion.
_Note: Not supported in ROCKSDB_LITE mode_

#### WriteOptions::disableWAL

If `WriteOptions::disableWAL` is set to true, writes will not go to the WAL at all, and the write may get lost after a crash. Useful when users rely on other logging or don't care about data loss.

## WAL Filter


## Related Pages

#### WAL Recovery Modes
https://github.com/facebook/rocksdb/wiki/WAL-Recovery-Modes

#### WAL Log Format
https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format